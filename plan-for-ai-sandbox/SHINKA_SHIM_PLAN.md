# Shinka-Shim: Implementation Plan

A thin bridge layer that lets ShinkaEvolve evolve auto-research trading strategies without modifying either project.

## Problem Summary

ShinkaEvolve and auto-research have incompatible evaluation models. Shinka evolves code within EVOLVE-BLOCK markers in `initial.py`, then calls an experiment function from that module via `evaluate.py` using `run_shinka_eval()`. auto-research runs strategies as subprocesses with JSON-over-stdin and expects separate `close_returns` + `positions` arrays for backtesting. The shim bridges these two worlds without modifying either project.

## Architecture Decision

The shim is a **standalone task directory** that Shinka consumes natively. It lives at `/home/oskar/ai-echosystem/shinka-shim/`. It imports auto-research's `backtest`, `torture`, and `datasource` modules as libraries (via `sys.path` manipulation). It does NOT use `sandbox.py` at all — the subprocess/JSON serialization layer is bypassed entirely because Shinka already `exec()`s the evolved code in-process.

## Key Design Decisions

1. **No sandbox subprocess.** Shinka already loads and executes the evolved `initial.py` in its own process. The strategy function is called directly as a Python function, not through `sandbox.run_strategy()`. This eliminates the JSON serialization bottleneck entirely.

2. **Data loaded once at evaluate.py import time.** The `.npz` file is loaded once as a module-level constant. Each evaluation call (potentially hundreds) reuses the same numpy arrays. The `.npz` file is pre-built by a `prepare_data.py` script.

3. **EVOLVE-BLOCK contains only `def strategy(bars)`.** Everything else (data loading, backtesting, torture tests, scoring) is fixed in the non-evolved parts of `initial.py` and `evaluate.py`.

4. **`close_returns` computed correctly in the shim.** The bug at `loop.py:136` (`close_returns = bars_np["close"]`) is not inherited. The shim computes `close_returns = np.diff(close) / close[:-1]` (percentage returns) during data preparation.

5. **Walk-forward re-executes the strategy.** The shim implements its own `walkforward_evolution_test()` that calls `strategy(sliced_bars)` on each fold, rather than slicing pre-computed positions.

6. **Errors produce `combined_score: 0.0`.** All strategy execution is wrapped in try/except. Any failure (crash, wrong shape, NaN positions) returns score 0.

## File Structure

```
/home/oskar/ai-echosystem/shinka-shim/
    evaluate.py          # Shinka-compatible evaluator
    initial.py           # Seed strategy + run_strategy_eval() entry point
    prepare_data.py      # One-time script: datasource → .npz file
    shinka_small.yaml    # Default Shinka config for quick runs
    shinka_medium.yaml   # Medium run config
    README.md            # How to use
    data/                # .gitignored, holds prepared .npz files
        bars.npz         # Output of prepare_data.py
    nodes/               # Per-researcher/per-symbol config examples
        btc_1h.yaml
        eth_1h.yaml
```

## File-by-file Design

### `prepare_data.py` — Data Preparation Script

Run once before evolution starts. Loads data from auto-research's `datasource.py`, converts to numpy, computes `close_returns` correctly, saves to `.npz`.

```python
"""Prepare market data as .npz for Shinka strategy evolution.

Usage:
    python prepare_data.py --source synthetic --output data/bars.npz
    python prepare_data.py --source ducklake --symbol AAPL --output data/bars.npz
    python prepare_data.py --source parquet --path local.parquet --output data/bars.npz
"""
import sys
import os
import argparse
import numpy as np

sys.path.insert(0, "/home/oskar/ai-echosystem/auto-research/auto-research")
from datasource import load_data


def bars_from_dataframe(df):
    """Polars DataFrame → canonical bars dict with correct close_returns."""
    close = df["close"].to_numpy(allow_copy=True)
    returns = np.empty_like(close)
    returns[0] = 0.0
    returns[1:] = np.diff(close) / close[:-1]

    return {
        "open": df["open"].to_numpy(allow_copy=True),
        "high": df["high"].to_numpy(allow_copy=True),
        "low": df["low"].to_numpy(allow_copy=True),
        "close": close,
        "volume": df["volume"].to_numpy(allow_copy=True),
        "close_returns": returns,
    }


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--source", required=True)
    parser.add_argument("--output", default="data/bars.npz")
    parser.add_argument("--symbol", default=None)
    parser.add_argument("--path", default=None)
    parser.add_argument("--firewall", type=bool, default=True)
    args = parser.parse_args()

    kwargs = {}
    if args.symbol:
        kwargs["symbol"] = args.symbol
    if args.path:
        kwargs["path"] = args.path
    kwargs["firewall"] = args.firewall

    df = load_data(args.source, **kwargs)
    bars = bars_from_dataframe(df)

    os.makedirs(os.path.dirname(args.output) or ".", exist_ok=True)
    np.savez(args.output, **bars)
    print(f"Saved {len(bars['close'])} bars to {args.output}")
    print(f"Keys: {list(bars.keys())}")
    print(f"close_returns range: [{bars['close_returns'].min():.6f}, {bars['close_returns'].max():.6f}]")


if __name__ == "__main__":
    main()
```

### `initial.py` — Seed Strategy + Experiment Entry Point

Two sections: the EVOLVE-BLOCK (which Shinka's LLM mutates) and the fixed harness (which Shinka never touches).

```python
import numpy as np

# EVOLVE-BLOCK-START
def strategy(bars):
    """Generate position weights from market data.

    Args:
        bars: dict with keys: open, high, low, close, volume, close_returns
              Each value is a 1D numpy array (single symbol).

    Returns:
        1D numpy array of position weights.
        -1.0 = full short, 0.0 = flat, 1.0 = full long.
    """
    close = bars["close"]
    n = len(close)

    fast_window, slow_window = 20, 50
    fast = np.convolve(close, np.ones(fast_window) / fast_window, mode="full")[:n]
    slow = np.convolve(close, np.ones(slow_window) / slow_window, mode="full")[:n]

    positions = np.where(fast > slow, 1.0, 0.0)
    positions[:slow_window] = 0.0
    return positions
# EVOLVE-BLOCK-END


import sys
import os

def run_strategy_eval(data_path="data/bars.npz"):
    """Entry point called by Shinka's evaluate.py via run_shinka_eval.

    Loads pre-computed bars from .npz, runs the evolved strategy,
    backtests, and runs torture tests.
    """
    sys.path.insert(0, "/home/oskar/ai-echosystem/auto-research/auto-research")
    from backtest import backtest
    from torture import noise_test, deflation_test

    base_dir = os.path.dirname(os.path.abspath(__file__))
    abs_data_path = os.path.join(base_dir, data_path)

    data = np.load(abs_data_path)
    bars = {k: data[k] for k in data.files}

    positions = strategy(bars)
    positions = np.asarray(positions, dtype=np.float64)

    n_bars = len(bars["close"])
    if positions.ndim != 1 or len(positions) != n_bars:
        raise ValueError(f"positions shape {positions.shape} incompatible with {n_bars} bars")

    positions = np.clip(positions, -1.0, 1.0)
    close_returns = bars["close_returns"]

    bt = backtest(close_returns, positions)
    noise = noise_test(close_returns, positions)
    deflation = deflation_test(close_returns, positions)
    wf = walkforward_evolution_test(strategy, bars, n_folds=5)

    torture_passed = noise["passed"] and deflation["passed"] and wf["passed"]

    return bt["sharpe"], torture_passed, {
        "backtest": bt,
        "noise": noise,
        "deflation": deflation,
        "walkforward": wf,
    }


def walkforward_evolution_test(strategy_fn, full_bars, n_folds=5, train_frac=0.6):
    """Walk-forward test that re-executes the strategy on each fold.

    Unlike auto-research's walkforward_test which slices pre-computed positions,
    this re-runs strategy_fn on each test window for true out-of-sample testing.
    """
    sys.path.insert(0, "/home/oskar/ai-echosystem/auto-research/auto-research")
    from backtest import backtest

    n = len(full_bars["close"])
    if n < 50:
        return {"passed": True, "folds": [], "pass_rate": 0.0, "skipped": "need >= 50 bars"}

    window_size = n // n_folds
    if window_size < 10:
        return {"passed": True, "folds": [], "pass_rate": 0.0, "skipped": "window too small"}

    train_size = int(window_size * train_frac)
    folds = []

    for i in range(n_folds):
        start = i * window_size
        train_end = start + train_size
        test_end = start + window_size

        if test_end > n or train_end >= test_end:
            break

        test_bars = {k: v[train_end:test_end] for k, v in full_bars.items()}

        if len(test_bars["close"]) == 0:
            break

        try:
            test_positions = strategy_fn(test_bars)
            test_positions = np.asarray(test_positions, dtype=np.float64)
            test_bt = backtest(test_bars["close_returns"], test_positions)
            test_sharpe = test_bt["sharpe"]
        except Exception:
            test_sharpe = -999.0

        folds.append({"fold": i, "test_sharpe": round(test_sharpe, 4)})

    if not folds:
        return {"passed": True, "folds": [], "pass_rate": 0.0, "skipped": "no valid folds"}

    pass_rate = sum(1 for f in folds if f["test_sharpe"] > 0) / len(folds)
    return {"passed": pass_rate > 0.5, "folds": folds, "pass_rate": round(pass_rate, 4)}
```

### `evaluate.py` — Shinka Protocol Adapter

```python
"""Shinka evaluator for auto-research trading strategies."""
import os
import argparse
import numpy as np
from typing import Any, Dict, List, Optional, Tuple

from shinka.core import run_shinka_eval


def get_strategy_kwargs(run_index: int) -> Dict[str, Any]:
    return {"data_path": "data/bars.npz"}


def validate_strategy_result(
    run_output: Tuple[float, bool, dict],
) -> Tuple[bool, Optional[str]]:
    try:
        sharpe, torture_passed, metrics = run_output
        if not isinstance(sharpe, (int, float)):
            return False, f"sharpe is not numeric: {type(sharpe)}"
        if np.isnan(sharpe) or np.isinf(sharpe):
            return False, f"sharpe is {sharpe}"
        return True, None
    except Exception as e:
        return False, f"Validation error: {e}"


def aggregate_strategy_metrics(
    results: List[Tuple[float, bool, dict]],
    results_dir: str,
) -> Dict[str, Any]:
    """Scoring: torture tests are hard constraints, Sharpe is continuous score."""
    if not results:
        return {"combined_score": 0.0, "error": "No results to aggregate"}

    sharpe, torture_passed, metrics = results[0]

    if not torture_passed:
        combined_score = 0.0
    else:
        combined_score = max(0.0, float(sharpe))

    output = {
        "combined_score": combined_score,
        "sharpe": float(sharpe),
        "torture_passed": torture_passed,
    }

    if metrics:
        for key in ("backtest", "noise", "deflation", "walkforward"):
            if key in metrics:
                output[key] = metrics[key]

    try:
        extra_file = os.path.join(results_dir, "extra.npz")
        np.savez(extra_file, **{"sharpe": sharpe, "torture_passed": torture_passed})
    except Exception:
        pass

    return output


def main(program_path: str, results_dir: str):
    os.makedirs(results_dir, exist_ok=True)

    def _aggregator(results):
        return aggregate_strategy_metrics(results, results_dir)

    metrics, correct, error_msg = run_shinka_eval(
        program_path=program_path,
        results_dir=results_dir,
        experiment_fn_name="run_strategy_eval",
        num_runs=1,
        get_experiment_kwargs=get_strategy_kwargs,
        validate_fn=validate_strategy_result,
        aggregate_metrics_fn=_aggregator,
    )

    if correct:
        print(f"Evaluation OK: combined_score={metrics.get('combined_score', 'N/A')}")
    else:
        print(f"Evaluation failed: {error_msg}")


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Strategy evaluator for Shinka")
    parser.add_argument("--program_path", type=str, default="initial.py")
    parser.add_argument("--results_dir", type=str, default="results")
    args = parser.parse_args()
    main(args.program_path, args.results_dir)
```

### `shinka_small.yaml` — Default Configuration

```yaml
max_evaluation_jobs: 2
max_proposal_jobs: 2
max_db_workers: 2

db_config:
  num_islands: 1
  archive_size: 40
  elite_selection_ratio: 0.3
  num_archive_inspirations: 4
  num_top_k_inspirations: 2
  migration_interval: 10
  migration_rate: 0.0
  island_elitism: true
  enforce_island_separation: true
  parent_selection_strategy: weighted
  parent_selection_lambda: 10
  archive_selection_strategy: crowding
  archive_criteria:
    combined_score: 1.0
    loc: -0.2

evo_config:
  patch_types: [diff, full, cross]
  patch_type_probs: [0.5, 0.35, 0.15]
  num_generations: 100
  max_api_costs: 1.0
  max_patch_resamples: 3
  max_patch_attempts: 3
  max_novelty_attempts: 3
  job_type: local
  language: python
  llm_models:
    - "gemini-3-flash-preview"
    - "gpt-5-mini"
  llm_kwargs:
    temperatures: [0, 0.5, 1.0]
    max_tokens: 16384
  embedding_model: text-embedding-3-small
  code_embed_sim_threshold: 0.99
  init_program_path: initial.py
  evolve_prompts: true
  prompt_evolution_interval: 5
```

## How It All Flows

```
1. Researcher runs:
   python prepare_data.py --source ducklake --symbol AAPL --output data/bars.npz

2. Researcher launches evolution:
   shinka_run --task-dir /home/oskar/ai-echosystem/shinka-shim \
              --results_dir results/aapl_v1 \
              --num_generations 100 \
              --config-fname shinka_small.yaml

3. Shinka reads evaluate.py and initial.py from --task-dir

4. For each generation, Shinka:
   a. Mutates the EVOLVE-BLOCK in initial.py (only the strategy() function)
   b. Writes the mutated code to a temp file
   c. evaluate.py calls run_shinka_eval() which:
      - Loads the mutated module via importlib
      - Calls run_strategy_eval(data_path="data/bars.npz")
      - run_strategy_eval loads bars from .npz (fast, binary, ~2MB not 20MB)
      - Calls strategy(bars) directly (no subprocess, no JSON)
      - Runs backtest + torture + walk-forward
      - Returns (sharpe, torture_passed, metrics)
   d. aggregate_strategy_metrics produces combined_score
   e. Shinka records the score and evolves

5. After evolution, best strategies are in Shinka's results directory
```

## Per-Node Configuration

A researcher configures a new "node" by:

1. Running `prepare_data.py` with different `--source`/`--symbol` args
2. Optionally creating a node-specific subdirectory with its own `data/bars.npz`

For multi-node setups: separate task directories per node, each with its own data. The `get_strategy_kwargs` function can be parameterized via environment variable or a small `node_config.yaml`.

## Issues Addressed

| REVIEW Issue | How the Shim Fixes It |
|---|---|
| #1 Incompatible evaluation models | `evaluate.py` + `initial.py` conform to Shinka protocol, call auto-research as libraries |
| #2 Walk-forward broken | `walkforward_evolution_test()` re-executes `strategy()` per fold |
| #4 bars not standardized | `bars_from_dataframe()` produces canonical dict with `close_returns` |
| #6 Scoring conflates things | Sharpe as continuous score, torture as hard constraint (0 if any fail) |
| #7 Data serialization | Binary `.npz` loaded once, no JSON, no subprocess per eval |
| #8 No error handling | `wrap_eval.py` outer try/except returns `combined_score: 0.0` |
| VET Issue B: close vs returns | `close_returns = np.diff(close) / close[:-1]` computed correctly in `prepare_data.py` |

## Potential Pitfalls

1. **`sys.path` manipulation**: Auto-research imports happen inside `run_strategy_eval()`, not at module top level. Shinka loads the module fresh each time via `importlib`, and path manipulation at the top level could interfere with the EVOLVE-BLOCK.

2. **Data path resolution**: The `.npz` file path is resolved relative to `initial.py`'s location via `os.path.dirname(os.path.abspath(__file__))`.

3. **Strategy determinism**: With `num_runs=1`, we assume the strategy is deterministic. If strategies use `np.random`, results will vary — acceptable, Shinka handles variance naturally.

4. **Firewalled data**: The `.npz` contains whatever `datasource.load_data()` returns. If firewalled, the evolved strategy sees firewalled data. Correct behavior.

## Not Addressed (Future Phases)

- **Cross-island migration** (#3) — requires multi-node setup, deferred
- **positions.json export** (#5) — separate tool, not part of the evolution shim
- **Strategy execution runtime** (#10) — new component for live paper-trading
- **Feedback loop** (#11) — leaderboard → Shinka injection, requires leaderboard to exist first
- **Docker deployment** (#9) — single fat container decision, deferred until local works
