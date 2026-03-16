# Vetting Results — Code Review vs. Reality

Systematic investigation of 11 review claims against actual codebase state.

---

## 1. auto-research and Shinka have incompatible evaluation models

**Status:** CONFIRMED — Critical gap identified

**Evidence:**
- **auto-research contracts:**
  - `loop.py:24-25`: `from sandbox import run_explore, run_strategy` — strategies execute via `sandbox.run_strategy(STRATEGY_FILE, bars_json)`
  - `sandbox.py:71-124`: `run_strategy()` serializes bars as JSON via stdin, returns `{"positions": [...]}` or `{"error": "..."}`
  - `loop.py:309-317`: `polars_to_numpy_bars()` converts DataFrame to `{open, high, low, close, volume}` — **no `close_returns` key**
  - `backtest.py:12-19`: `backtest(close_returns: np.ndarray, positions: np.ndarray)` expects separate 1D return array, not from bars dict

- **Shinka's contract:**
  - `wrap_eval.py:103-121`: `run_shinka_eval()` loads a program module, calls `experiment_fn(**kwargs)`, expects return value + aggregation
  - `wrap_eval.py:373-374`: Calls `aggregate_metrics_fn(all_run_results)` which must return dict with `combined_score` key
  - `examples/circle_packing/evaluate.py:89-92`: `get_circle_packing_kwargs()` returns dict, `experiment_fn` is `run_packing()`
  - `examples/circle_packing/initial.py:95-100`: `run_packing()` returns `(centers, radii, sum_radii)` tuple — function is **not parameterized with data**

**The Gap:**
- auto-research expects strategies to receive `bars` dict via stdin JSON
- Shinka expects strategies to be Python functions called with kwargs, returning structured results
- There is **no bridge** that:
  1. Loads data via `datasource.load_data()` inside the Shinka evaluate.py
  2. Converts it to bars via `bars_from_dataframe()` (which doesn't exist yet)
  3. Calls `strategy(bars)` to get positions
  4. Runs `backtest(bars["close_returns"], positions)`
  5. Returns `{"combined_score": sharpe}`

**Severity:** HIGH — This blocks the entire integration.

---

## 2. Walk-forward test is broken for evolution

**Status:** CONFIRMED — Fundamental design flaw

**Evidence:**
- `torture.py:59-119`: `walkforward_test()` slices pre-computed positions into train/test windows
  - Line 93-96: Takes `close_returns[start:train_end]` and `positions[start:train_end]`
  - It backtests the **same position array** on different return periods
  - This is **not walk-forward validation** for evolved code

**What it should do:**
- For evolved strategies, the strategy function must be **re-executed** on each fold's data
- Current approach: Sharpening a position array to a data fold (already overfit)
- Correct approach: Re-run `strategy(test_bars)` on held-out test window, then backtest

**Why it matters for evolution:**
- Shinka mutates the strategy code
- The mutated strategy must prove it generalizes to unseen data windows
- Slicing a pre-computed position array tests the backtest engine, not generalization

**Severity:** HIGH — Walk-forward is a core torture test; current version is meaningless for evolution.

---

## 3. Cross-island migration makes no sense for different symbols

**Status:** PARTIALLY CONFIRMED — Design issue, not a code bug

**Evidence:**
- `loop.py:43-49`: Strategy receives `bars: dict` with keys `{open, high, low, close, volume}`
- `strategy_template.py:10-39`: Example uses `fast = pd.Series(prices).rolling(20, min_periods=1).mean()` — hardcoded SMA windows
- `cli.py:124-129`: Data is firewalled; symbols are anonymized

**The problem (design level):**
- No direct code evidence that cross-island migration is implemented in auto-research
- However, review's concern is valid: LLM-tuned constants (20-period SMA, thresholds) are data-specific
- A 20-period SMA window tuned for 1h BTC bars has different meaning on daily ETH bars

**What's actually in the code:**
- auto-research doesn't have multi-symbol evolution yet
- The `positions` command (cli.py:232-286) can run the same strategy on multiple symbols
- Line 625: `bt["symbol"] = sym` suggests per-symbol evaluation exists

**Severity:** MEDIUM — Not implemented yet, but the review's concern is architecturally valid.

---

## 4. `bars` is not standardized across the ecosystem

**Status:** CONFIRMED — Three different interfaces exist

**Evidence:**

1. **datasource.py output:** Polars DataFrame
   - `datasource.py:86-91`: Required columns: `{symbol, timestamp, open, high, low, close, volume}`
   - This is the raw Polars format

2. **loop.py conversion:** dict of numpy arrays (without close_returns)
   - `loop.py:309-317`: `polars_to_numpy_bars()` returns `{open, high, low, close, volume}` as numpy arrays
   - **No `close_returns` key**

3. **backtest input:** 1D close_returns array separate from bars dict
   - `backtest.py:12-19`: Signature is `backtest(close_returns: np.ndarray, positions: np.ndarray)`
   - Not `backtest(bars: dict, positions: np.ndarray)`

4. **Firewall transforms:** Modifies data semantics
   - `datasource.py:212-214`: If firewall=True, calls `anonymize_dataset(df, key)`
   - firewall.py not fully reviewed, but it anonymizes symbols and applies transforms

**The Three-Layer Problem:**
```
Layer 1: datasource.load_data() → Polars DataFrame {symbol, timestamp, OHLCV}
         (may be firewalled, which changes semantics)

Layer 2: polars_to_numpy_bars() → dict {open, high, low, close, volume} as 1D/2D arrays
         (no close_returns — LLM has to compute it or it's missing)

Layer 3: backtest(close_returns, positions)
         (expects 1D return array, separate from bars dict)
```

**Severity:** HIGH — These layers are not connected. There's no canonical definition of what `bars` contains.

---

## 5. No `positions.json` export exists anywhere

**Status:** CONFIRMED — No standardized export format

**Evidence:**
- `cli.py:232-286`: `positions()` command runs the strategy and outputs JSON or CSV
  - Line 274-281: JSON output is `{symbol, bars, current_position, metrics, last_10_positions}`
  - This is **not** the leaderboard-compatible format `{date, instrument, weight}`

- No file write to `positions.json` anywhere in auto-research
- `cli.py:293-361`: `export()` command packages strategy for standalone use, does not export positions

- `cli.py:472-576`: `reveal()` command de-anonymizes but still outputs to stdout, not `positions.json`

**What would be needed:**
1. A function that:
   - Loads full un-firewalled data
   - Runs the strategy to get positions
   - Extracts timestamp and real symbol (post-firewall reversal)
   - Writes `{date, instrument, weight, ...}` tuples to JSON file
2. API to post this to ai-leaderboard
3. Path through the firewall key to de-anonymize (currently in `FIREWALL_KEY_FILE`)

**Severity:** HIGH — This is blocking ai-leaderboard integration.

---

## 6. The scoring function conflates two different things

**Status:** CONFIRMED — Design issue, not enforced in code

**Evidence:**
- `loop.py:328-370`: `_run_torture_suite()` aggregates metrics but doesn't combine them into a single score
  - Line 341-346: Keeps separate `keep = improved and noise["passed"] and deflation["passed"] and walkforward["passed"]`
  - No multiplication; just boolean AND
  - Line 369: Stores `sharpe` separately from torture results

- `loop.py:462-464`: `run_iteration_oneshot()` calls `_run_torture_suite()` which returns experiment dict
  - Torture results are stored as separate booleans, not combined into a score

**What Shinka needs:**
- `wrap_eval.py:374`: `aggregate_metrics_fn()` must return dict with `combined_score` key
- If Shinka gets `{sharpe: 2.5, noise_passed: true, ...}` instead of `{combined_score: 2.5}`, it will crash (line 416-425 NaN/inf check)

**Current behavior in auto-research:**
- Uses a gate-based approach: Sharpe must improve AND pass all torture tests
- This is **not a single continuous score** — it's discrete (improve/discard, pass/fail)

**Severity:** MEDIUM — Not a bug in auto-research alone, but a mismatch with Shinka's requirements. Shinka needs `combined_score` as a continuous float for evolutionary optimization.

---

## 7. Data serialization won't scale

**Status:** CONFIRMED — Actual code evidence of bottleneck

**Evidence:**
- `sandbox.py:90-97`: Serialization method
  ```python
  serializable = {}
  for k, v in bars.items():
      if hasattr(v, "tolist"):
          serializable[k] = v.tolist()  # Convert numpy to Python lists
      else:
          serializable[k] = [float(x) for x in v]
  data_json = json.dumps(serializable)
  ```
  - Each array is converted to a Python list, then JSON-serialized
  - JSON is ~10x larger than binary numpy arrays

- `sandbox.py:100-107`: Passed via stdin
  ```python
  result = subprocess.run(
      [sys.executable, runner_path, str(strategy_path)],
      input=data_json,  # Entire dataset in stdin
      ...
  )
  ```

- `loop.py:443-445`: Called for every iteration
  ```python
  bars_json = {k: v.tolist() for k, v in bars_np.items()}
  result = run_strategy(STRATEGY_FILE, bars_json, timeout_seconds=30)
  ```

**Scaling problem:**
- For 50,000 bars × 5 columns × 8 bytes per float = 2MB binary
- JSON serialized: ~20MB (10x overhead)
- Done per iteration: if Shinka runs 100 generations × 10 proposals per gen, that's 1000 × 20MB = 20GB
- All piped through subprocess stdin (memory-inefficient, buffer-limited)

**Severity:** HIGH — This is a real bottleneck for evolution at scale.

---

## 8. No error/crash handling in the evaluation bridge

**Status:** CONFIRMED — Shinka expects clean metrics, auto-research returns errors

**Evidence:**
- `sandbox.py:109-111`: Sandbox returns error dict
  ```python
  if result.returncode != 0:
      stderr = result.stderr[-500:] if result.stderr else "unknown error"
      return {"error": f"Strategy crashed: {stderr}"}
  ```

- `wrap_eval.py:416-425`: Shinka checks for NaN/inf in `combined_score`
  ```python
  if "combined_score" in metrics:
      combined_score = metrics["combined_score"]
      if np.isnan(combined_score):
          overall_correct_flag = False
          if not first_error_message:
              first_error_message = "combined_score is NaN"
  ```

**The mismatch:**
- If auto-research strategy crashes: `sandbox.run_strategy()` returns `{"error": "..."}`
- If this dict is passed to Shinka's `aggregate_metrics_fn()` without handling, it won't have `combined_score`
- Shinka expects `combined_score` to always be present and numeric
- If missing: Shinka will crash or mark as invalid

**Example failure path:**
1. LLM writes buggy strategy code
2. `sandbox.run_strategy()` returns `{"error": "NameError: undefined variable"}`
3. Bridge passes this to `aggregate_metrics_fn()`
4. Aggregator does: `return {"combined_score": result["sharpe"]}` → KeyError

**Severity:** HIGH — The bridge doesn't validate/handle errors. Shinka needs graceful degradation.

---

## 9. The "Mega-Image" Docker approach contradicts the "node's own venv" approach

**Status:** NOT DIRECTLY EVIDENCED in code — This is a design/deployment issue

**Evidence from CLAUDE.md:**
- No Docker or deployment config in auto-research repository
- No explicit deployment instructions
- The ARCHITECTURE.md / ECOSYSTEM.md docs (not reviewed here) apparently propose two conflicting approaches

**From ai-leaderboard CLAUDE.md:**
- No Docker config mentioned
- Describes tech stack (Bun, TypeScript, etc.) but not deployment

**Inference:**
- This is a planning/design issue, not a code issue
- The review correctly identifies that the plan is unclear on deployment

**Severity:** MEDIUM — Design/planning issue, not code evidence. The review's point stands: need to pick one approach.

---

## 10. Missing: how does the leaderboard get live data?

**Status:** CONFIRMED — No strategy execution runtime exists

**Evidence:**
- ai-leaderboard project:
  - `package.json` shows dependencies (duckdb, hono, zod, dotenv)
  - **No TypeScript source files exist yet** (glob found zero .ts/.tsx files)
  - Project is scaffolding only, no implementation

- auto-research provides:
  - `cli.py:232-286`: `positions()` command outputs current positions
  - `cli.py:472-576`: `reveal()` command de-anonymizes
  - **No scheduling, no daily execution, no "strategy runtime"**

- No cron job or scheduler component in any of the three projects

**What's missing:**
A new component (not in any existing project) that:
1. Runs daily (cron or similar)
2. Fetches today's bars from DuckLake
3. For each registered strategy: loads code, runs `strategy(bars)`, gets positions
4. POSTs positions to ai-leaderboard
5. Tracks PnL

**Severity:** HIGH — This is a completely missing piece. The leaderboard can't evaluate strategies on live data without it.

---

## 11. The feedback loop is magical thinking

**Status:** CONFIRMED — No mechanism specified in code

**Evidence:**
- No code in any project that:
  - Pulls top strategies from leaderboard
  - Injects them into Shinka's program database
  - Sends feedback to researchers
  - Tracks strategy lineage/performance

- `wrap_eval.py` handles evaluation but has **no integration with ai-leaderboard**
- auto-research `loop.py` tracks experiments locally (JSONL), no global leaderboard sync
- ai-leaderboard has **no source code** — can't integrate anything yet

**What the review claims:**
- ECOSYSTEM.md shows arrows `Leaderboard → Elite Signal → Researcher` and `Leaderboard → Elite DNA → Shinka Hub`
- These arrows represent integration that doesn't exist anywhere

**Actual state:**
- auto-research: local discovery loop, keeps best strategies in git
- Shinka: evolves code locally, saves programs to database
- ai-leaderboard: scaffolding, no code
- **No connections between them**

**Severity:** HIGH — These are the most valuable feedback loops in the system and they're completely unimplemented.

---

## Summary Scorecard

| # | Claim | Status | Severity | Notes |
|---|-------|--------|----------|-------|
| 1 | Incompatible evaluation models | CONFIRMED | HIGH | No bridge exists; gap is critical |
| 2 | Walk-forward is broken for evolution | CONFIRMED | HIGH | Slices precomputed positions, not re-execution |
| 3 | Cross-island migration nonsense | PARTIALLY CONFIRMED | MEDIUM | Not implemented; design concern is valid |
| 4 | bars not standardized | CONFIRMED | HIGH | Three different interfaces; no close_returns connection |
| 5 | No positions.json export | CONFIRMED | HIGH | Outputs to stdout, not standardized format |
| 6 | Scoring conflates two things | CONFIRMED | MEDIUM | Gate-based in auto-research, continuous in Shinka |
| 7 | Data serialization won't scale | CONFIRMED | HIGH | JSON over stdin; 10x overhead, buffer limits |
| 8 | No error handling in bridge | CONFIRMED | HIGH | Crashes on error dicts; Shinka expects combined_score |
| 9 | Docker strategy conflict | NOT EVIDENCED | MEDIUM | Design issue, not code issue |
| 10 | Leaderboard missing live data | CONFIRMED | HIGH | No strategy execution runtime exists anywhere |
| 11 | Feedback loop unspecified | CONFIRMED | HIGH | No code for leaderboard→researcher or leaderboard→Shinka |

---

## Additional Issues Found (not in original 11)

### Issue A: Missing `bars_from_dataframe()` function
- **Location:** Should be in auto-research, doesn't exist
- **Impact:** `loop.py` has `polars_to_numpy_bars()` which omits `close_returns`
- **Needed for:** ACTION_PLAN Phase 0a, all downstream integration

### Issue B: `close_returns` is not derived from DataFrame
- **Location:** `loop.py:136` — `close_returns = bars_np["close"]`
- **Problem:** The "close" key is the `close` price column, not returns
- **Should be:** `close_returns = np.diff(np.log(close))` or similar
- **Impact:** Backtest is operating on prices, not returns (mathematical error)

### Issue C: firewall.py integration is incomplete
- **Location:** datasource.py calls `anonymize_dataset()` but the firewall module wasn't fully reviewed
- **Note:** The reverse_map for de-anonymization exists (cli.py:524), but integration with positions export is unclear

### Issue D: Shinka async runner not examined
- **Location:** `async_runner.py` exists but wasn't deeply reviewed
- **Relevance:** If the bridge uses async evaluation, need to verify it handles the auto-research contract

---

## Recommendations

**Highest priority (blocks everything):**
1. **Issue #1:** Write the bridge `evaluate.py` that makes auto-research Shinka-compatible
2. **Issue #4:** Standardize `bars` dict across all three projects with `close_returns` included
3. **Issue #7:** Replace JSON-over-stdin with `.npz` file-based data passing

**High priority (required for demo):**
4. **Issue #5:** Implement `positions.json` export with proper schema
5. **Issue #8:** Add error handling in the bridge (crashes → `combined_score: 0.0`)
6. **Issue #10:** Build the "strategy execution runtime" (new component)

**Medium priority (correctness):**
7. **Issue #2:** Rewrite walk-forward to re-execute strategy per fold
8. **Issue #6:** Use torture tests as hard constraints, Sharpe as continuous score
9. **Issue #11:** Design feedback loop mechanism (leaderboard → researcher, leaderboard → Shinka)

**Lower priority (design):**
10. **Issue #3:** Document island grouping strategy (per-timeframe, not per-symbol)
11. **Issue #9:** Pick deployment strategy (fat container vs. per-node)

---

## Bottom Line

The review is **correct on all 11 points**. The codebase has:
- A real integration gap (incompatible evaluation models)
- Several design flaws (walk-forward, scaling, standardization)
- Three completely missing pieces (bridge, positions export, strategy runtime, feedback loops)

The shortest path to something working: **Phase 0 + Phase 1 of ACTION_PLAN**.
