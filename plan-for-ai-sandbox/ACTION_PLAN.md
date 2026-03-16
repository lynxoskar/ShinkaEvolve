# Action Plan — Fixing the Integration

Addresses every issue raised in REVIEW.md. Ordered by dependency — each phase unlocks the next.

## Phase 0: Align the Data Surface (prerequisite for everything)

The root problem: three projects speak different data dialects. Fix this first.

### 0a. Define the canonical `bars` dict

One definition, used everywhere:

```python
# bars: dict[str, np.ndarray]
# Keys: "close_returns", "open", "high", "low", "close", "volume", "timestamp"
# Values: 1D arrays (single symbol) or 2D arrays (n_bars x n_symbols)
# "close_returns" is ALWAYS present — it's what backtest() and torture use.
```

**Where this lives:** A shared `contracts.py` or a markdown spec in this plan directory.
Both auto-research and the Shinka bridge import/reference it.

### 0b. Fix the DataFrame → bars → backtest pipeline

Currently:
```
datasource.py → Polars DataFrame (symbol, timestamp, OHLCV)
    → ??? (manual, ad-hoc in loop.py)
        → sandbox gets bars dict via JSON
            → backtest() gets close_returns (separate 1D array, disconnected)
```

Should be:
```
datasource.py → Polars DataFrame
    → bars_from_dataframe(df) → canonical bars dict (includes close_returns)
        → strategy(bars) → positions
            → backtest(bars["close_returns"], positions)
            → torture tests use same bars["close_returns"]
```

**Action:** Add `bars_from_dataframe(df: pl.DataFrame) -> dict[str, np.ndarray]` to auto-research. This is the single conversion point. Everyone else uses the output.

### 0c. Fix data serialization for scale

Replace JSON-over-stdin with a temp `.npz` file path:

```python
# sandbox.py — new approach
np.savez(tmp_path, **bars)
# subprocess receives path, does: bars = dict(np.load(path))
```

Binary, fast, no pipe buffer limits. Works for millions of bars.

---

## Phase 1: The Bridge evaluate.py (REVIEW issues #1, #8)

Write a Shinka-compatible `evaluate.py` + `initial.py` for one auto-research node.

### 1a. `evaluate.py` (Shinka protocol)

```python
from shinka.core import run_shinka_eval

def get_kwargs(run_idx):
    return {"data_source": "synthetic", "data_args": {}}

def aggregate(results):
    sharpe, torture_passed = results[0]
    if not torture_passed:
        return {"combined_score": 0.0}  # hard constraint
    return {"combined_score": max(0.0, sharpe)}

def validate(result):
    sharpe, torture_passed = result
    return isinstance(sharpe, float), None

metrics, correct, err = run_shinka_eval(
    program_path=program_path,
    results_dir=results_dir,
    experiment_fn_name="run_strategy_eval",
    num_runs=3,
    get_experiment_kwargs=get_kwargs,
    aggregate_metrics_fn=aggregate,
    validate_fn=validate,
)
```

### 1b. `initial.py` (the seed strategy)

```python
import numpy as np

# EVOLVE-BLOCK-START
def strategy(bars):
    close = bars["close"]
    fast = np.convolve(close, np.ones(20)/20, mode='full')[:len(close)]
    slow = np.convolve(close, np.ones(50)/50, mode='full')[:len(close)]
    return np.where(fast > slow, 1.0, 0.0)
# EVOLVE-BLOCK-END

def run_strategy_eval(data_source="synthetic", data_args=None):
    """Called by Shinka's evaluate.py via run_shinka_eval."""
    from datasource import load_data
    from backtest import backtest
    from torture import noise_test, deflation_test, walkforward_test

    df = load_data(data_source, **(data_args or {}))
    bars = bars_from_dataframe(df)

    positions = strategy(bars)
    bt = backtest(bars["close_returns"], positions)

    noise = noise_test(bars["close_returns"], positions)
    deflation = deflation_test(bars["close_returns"], positions)
    # TODO: walk-forward needs re-execution (Phase 2)

    torture_passed = noise["passed"] and deflation["passed"]
    return bt["sharpe"], torture_passed
```

### 1c. Error handling

The bridge wraps strategy execution in try/except. Crashes → `combined_score: 0.0`. Shinka treats this as a failed mutation and moves on.

**Milestone:** One strategy evolving locally via `shinka_run --task-dir`.

---

## Phase 2: Fix Walk-Forward for Evolution (REVIEW issue #2)

Current walk-forward slices pre-computed positions. For evolution, we need to **re-run the strategy on each fold's data**.

```python
def walkforward_evolution_test(strategy_fn, full_bars, n_folds=5):
    """Re-execute strategy on each fold — true out-of-sample test."""
    for fold in folds:
        train_bars = slice_bars(full_bars, train_start, train_end)
        test_bars = slice_bars(full_bars, test_start, test_end)

        # Strategy sees only test data — no lookahead
        test_positions = strategy_fn(test_bars)
        test_bt = backtest(test_bars["close_returns"], test_positions)
        fold_results.append(test_bt["sharpe"])

    pass_rate = sum(1 for s in fold_results if s > 0) / len(fold_results)
    return pass_rate > 0.5
```

This requires the bridge evaluate.py to have access to the strategy function, not just its output. Since Shinka already `exec()`s the program, this is achievable — the evaluate.py imports the evolved code and calls `strategy()` directly per fold.

---

## Phase 3: Fix the Scoring Function (REVIEW issue #6)

**Decision:** Sharpe is the score. Torture tests are hard constraints.

```python
def combined_score(sharpe, torture_results):
    # Any torture failure → score 0 (Shinka discards this candidate)
    if not all(t["passed"] for t in torture_results):
        return 0.0
    # Positive Sharpe only (negative strategies are useless)
    return max(0.0, sharpe)
```

This gives Shinka a smooth optimization surface (higher Sharpe = better) while maintaining the torture tests as non-negotiable gates.

---

## Phase 4: Fix Cross-Island Migration (REVIEW issue #3)

**Decision:** Islands are per-timeframe, not per-symbol.

Same-timeframe strategies can meaningfully cross-pollinate because the code patterns (window sizes, threshold logic) operate on compatible data distributions. Different-timeframe strategies cannot.

**Island grouping:**
- Island group A: all 1h strategies (BTC-1h, ETH-1h, SOL-1h)
- Island group B: all daily strategies (BTC-daily, ETH-daily)

Migration happens **within** groups. Cross-group migration is disabled or limited to structural patterns only (a future enhancement, not V1).

**Shinka config:**
```python
DatabaseConfig(
    num_islands=len(same_timeframe_nodes),
    migration_interval=10,
    migration_rate=0.1,
    enforce_island_separation=False,  # allow cross-pollination within group
)
```

---

## Phase 5: positions.json Export (REVIEW issues #5, #10)

### 5a. auto-research → leaderboard export

Add `export_positions()` to auto-research that:
1. Loads the kept strategy
2. Runs it on full (un-firewalled) data
3. Writes `{date, instrument, weight}` tuples to JSON
4. POSTs to ai-leaderboard API

This requires de-anonymization (firewall key) to map anonymized symbols back to real instruments.

### 5b. Strategy execution runtime (the missing piece)

For the leaderboard to paper-trade going forward, *someone* must run strategies on new daily data. This is a **new component** — a lightweight cron job:

```
daily-runner (new):
    1. Fetch today's bars from DuckLake
    2. For each registered strategy: load code, run strategy(bars), get positions
    3. POST positions to ai-leaderboard
```

This is small (~100 lines) but it's a real service that needs to exist. The plan completely omits it.

---

## Phase 6: Close the Feedback Loop (REVIEW issue #11)

### Leaderboard → Researcher
Concrete mechanism: leaderboard CLI command that shows ranked strategies with links to their git repos/commits. The researcher reads the leaderboard, sees what's winning, and adjusts their approach manually. No magic — just visibility.

### Leaderboard → Shinka Hub
Concrete mechanism: top-N strategies from the leaderboard can be injected into Shinka's program database as "archive inspirations." A CLI command or cron job:
1. Query leaderboard for top strategies
2. Fetch their source code (from git repo metadata)
3. Insert into Shinka's `ProgramDatabase` as archive entries

This is Phase 6 because it requires all previous phases to be working.

---

## Phase 7: Deployment Decisions (REVIEW issue #9)

**Decision:** Single fat container for V1.

Rationale:
- Researchers share the same core deps (numpy, polars, anthropic)
- Strategy code uses only standard scientific Python — no exotic deps
- Per-node containers add build pipeline complexity for zero benefit in V1
- If a researcher needs a custom dep, they add it to the shared image

Revisit if dep conflicts actually occur. Don't solve problems we don't have.

---

## Execution Order

```
Phase 0 (data surface)     ← Do this week. Foundation for everything.
Phase 1 (bridge evaluate)  ← Immediately after. Proves the integration.
Phase 2 (walk-forward fix) ← During Phase 1, natural extension.
Phase 3 (scoring fix)      ← Trivial, do with Phase 1.
Phase 4 (island grouping)  ← Config decision, apply when multi-node.
Phase 5 (positions export) ← When ai-leaderboard is ready to receive.
Phase 6 (feedback loop)    ← After leaderboard is populated.
Phase 7 (deployment)       ← When local single-node works end-to-end.
```

**The only thing that matters right now is Phase 0 + Phase 1.** Everything else is sequenced behind a working single-node integration.
