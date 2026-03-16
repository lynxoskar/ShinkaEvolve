# Plan Review — Blunders, Gaps, and Sloppy Thinking

Critical review of ARCHITECTURE.md and ECOSYSTEM.md against the actual codebase state.

## 1. The Big One: auto-research and Shinka have incompatible evaluation models

**Shinka's contract:**
- `initial.py` has an `EVOLVE-BLOCK` containing pure algorithmic code
- `evaluate.py` is a standalone script that calls `run_shinka_eval()` — it imports and calls a function from `initial.py`, passes args, validates output, returns `combined_score`
- The LLM mutates code **inside the EVOLVE-BLOCK only**
- Data is internal to the evaluate/initial scripts — no external data pipeline

**auto-research's contract:**
- `strategy(bars)` receives a `bars` dict via JSON over stdin to a subprocess
- The sandbox runner (`sandbox.py`) serializes the entire dataset as JSON, pipes it in
- `backtest.py` takes `close_returns` (1D numpy array) + `positions` (1D numpy array)
- `torture.py` takes the same flat arrays

The plan says "the Universal Evaluator calls `auto-research/backtest.py` and `torture.py`" — but **there's no `evaluate.py` that conforms to Shinka's `run_shinka_eval()` protocol**. You'd need to write a bridge: a Shinka-compatible `evaluate.py` that loads data via `datasource.py`, runs the evolved strategy through `sandbox.py`, feeds the result through `backtest()` and torture tests, and returns a `combined_score`. This is the most important missing piece and the plan treats it as trivial ("Build `shinka_aws_eval.py`" — one line in the implementation plan).

## 2. The walk-forward test is broken for evolution

auto-research's `walkforward_test()` takes **pre-computed positions** and slices them into folds. But for evolution, the strategy code needs to be **re-run on each fold's data** to test generalization. Right now it just slices the same position array — the strategy saw all the data when it generated those positions. This isn't walk-forward validation, it's slicing an already-fitted output. Shinka would need a walk-forward that re-executes the strategy on held-out data windows.

## 3. Cross-island migration makes no sense for different symbols

The plan says: "a regime filter from an ETH island is applied to a BTC strategy." But auto-research strategies receive `bars` — a dict keyed by column names like `close`, `volume`, etc. The strategies are **data-agnostic in theory** but **data-coupled in practice** because:
- The LLM tunes constants (moving average windows, thresholds) to the specific data distribution
- A 20-period SMA crossover tuned for 1h BTC bars is meaningless on daily ETH bars
- Cross-pollinating code between nodes with different timeframes, symbols, or data sources will mostly produce garbage that fails evaluation

The plan hand-waves this as Shinka's "migration" feature, but Shinka's migration moves code between islands running the **same** evaluation. Cross-node migration needs a much more careful design — maybe only migrating structural patterns (regime detection, signal combination logic) while re-fitting parameters.

## 4. `bars` is not standardized across the ecosystem

The plan claims `bars` is "a standardized dictionary of OHLCV arrays." But:
- `datasource.py` returns a **Polars DataFrame** with columns `{symbol, timestamp, open, high, low, close, volume}`
- `sandbox.py` converts this to a **dict of numpy arrays** passed via JSON
- `backtest.py` expects a single `close_returns` 1D array — not `bars` at all
- The firewall can anonymize symbols, shift dates, normalize to returns — so firewalled `bars` has different semantics than raw `bars`
- Multi-symbol data is 2D arrays; single-symbol is 1D

There's a gap between "the Polars DataFrame from datasource" and "the numpy dict the strategy receives" and "the 1D arrays backtest/torture expect." The plan pretends these are all the same thing.

## 5. No `positions.json` export exists anywhere

The plan says strategies export `positions.json` to ai-leaderboard. Neither auto-research nor the Shinka integration writes this file. auto-research has a `positions` CLI command, but it prints to stdout — there's no structured export that conforms to ai-leaderboard's `{date, instrument, weight}` format. And for firewalled data, the instruments are anonymized — you'd need to de-anonymize before the leaderboard can join against real market data.

## 6. The scoring function conflates two different things

ARCHITECTURE.md says `Score: Sharpe * PassRate`. But:
- Sharpe comes from `backtest()` — a single number
- PassRate comes from `walkforward_test()` and `noise_test()` — binary pass/fail with a ratio
- Shinka maximizes `combined_score` — a single float
- Multiplying Sharpe by a binary pass/fail is discontinuous and will confuse the evolutionary optimizer. Better: Sharpe as the score, torture tests as hard constraints (reject if they fail, don't scale the score).

## 7. Data serialization won't scale

`sandbox.py` pipes the entire dataset as JSON through stdin. For DuckLake data (potentially millions of bars), this will:
- Blow up memory (JSON is ~10x the size of binary numpy)
- Be extremely slow to serialize/deserialize
- Hit subprocess pipe buffer limits

For Shinka running hundreds of evaluations, this becomes a bottleneck. The evaluate.py should load data once and pass a file path or shared memory reference, not serialize per evaluation.

## 8. No error/crash handling in the evaluation bridge

Shinka expects `evaluate.py` to always return metrics. auto-research's sandbox can return `{"error": "..."}`. The plan doesn't address what happens when a mutated strategy crashes, times out, or returns wrong-shaped positions. Shinka needs a `combined_score` of 0 (or negative) for failed evaluations, and the bridge needs to handle this gracefully.

## 9. The "Mega-Image" Docker approach contradicts the "node's own venv" approach

Section 5 says evaluations run in "the node's specific `uv` virtualenv or a shared 'Mega-Image'." These are opposite strategies with different tradeoffs. The plan doesn't decide which one. For AWS Batch, you need to pick: either one fat container with everything, or per-node containers (which means a container registry and build pipeline per researcher). This needs a decision, not an "or."

## 10. Missing: how does the leaderboard get live data?

The plan says ai-leaderboard paper-trades strategies against DuckLake data. But evolved strategies are static code — they don't produce positions on new, unseen data. The leaderboard needs a way to:
1. Get today's market data
2. Feed it through each registered strategy
3. Record the positions
4. Compute PnL

This "strategy execution runtime" doesn't exist in any of the three projects. The leaderboard explicitly says "it doesn't run strategies." So who runs them on new data?

## 11. The feedback loop is magical thinking

ECOSYSTEM.md shows `Leaderboard → Elite Signal → Researcher` and `Leaderboard → Elite DNA → Shinka Hub`. There's no mechanism for either. How does a leaderboard ranking turn into actionable feedback for a researcher? How does a ranked strategy's code get injected back into Shinka's program database? These arrows represent the most valuable part of the system and they're completely unspecified.

---

## Bottom Line

**The shortest path to something real:** Write the bridge `evaluate.py` that makes one auto-research strategy Shinka-compatible. Get one node evolving locally before proving multi-node. The plan jumps to cloud-hosted multi-researcher before proving the single-node integration works.
