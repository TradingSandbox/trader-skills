---
name: "backtest-fast-sweep"
description: "Run durable, resumable TradingView-only backtests through TradeCLI's high-level backtest tool. Use when a compiled current-chart strategy needs parameter screening, holdout or walk-forward validation, regime and execution-cost stress tests, finalist trade/equity diagnostics, replay checks, or a paper-candidate readiness verdict. The tool keeps selection inside training windows, evaluates finalists afterward, never rewrites Pine or places trades, and persists the job in AITradingOffice."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [backtest]
  allowed_ledgers: []
  limits:
    max_trades_per_tick: 0
    max_notional: 0
    paper_only: true
  trigger: null
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Backtest Fast Sweep

## Intent

Turn a user's backtest idea into one deterministic `backtest` job. The Pine
strategy must already be compiled on the current TradingView chart. This skill
does not write, repair, compile, save, or publish Pine and does not call raw
TradingView or AITradingOffice tools. The `backtest` tool owns those mechanics
and persists the complete job.

This is fast screening, not proof of edge. TradingView is the only simulator.
The default test horizon is the most recent 365 days through Deep Backtesting.

## Tool boundary

`backtest` is the sole entry point. Its actions are:

- `start`: inspect the current strategy, create the AITradingOffice research
  record and workflow run, plan all cells, and execute the first bounded chunk;
- `continue`: execute exactly the next persisted chunk;
- `status`: read progress without touching TradingView;
- `results`: return the persisted specification, cells, and leaderboard;
- `cancel`: stop a running job and invalidate its workflow run.

Never replace this with a model-authored per-cell loop. Never call
`strategy_fast_sweep`, `tv_strategy_optimize`, Pine tools, or office tools from
this skill.

## Intake

Resolve the smallest decision-useful specification:

```yaml
hypothesis: "What should improve, and why?"  # required before seeing results
fixed_parameters:                            # optional; held constant
  "RR target": 2.0
  "ATR stop multiplier": 2.0
sweep_parameters:                            # optional; arrays form a Cartesian grid
  "Entry mode": [breakout, pullback, hybrid]
  "Swing length": [10, 20, 30]
symbols: [NSE:RELIANCE]                      # optional; current symbol by default
timeframes: [D]                              # candle resolution; current chart by default
horizon: last_365d                           # historical range; one year by default
# date_from: 2024-01-01                      # optional custom range (requires date_to)
# date_to: 2025-12-31
rank_by: drawdown_adjusted_profit_factor
min_trades: 20
finalists: 3
validation_mode: holdout                       # none | holdout | walk_forward
out_of_sample_pct: 20                          # holdout default
# walk_forward_folds: 3                        # walk-forward default
# initial_train_pct: 50                        # expanding first training window
regime_windows:
  - {name: crash, from: 2020-03-01, to: 2020-06-30}
cost_scenarios:
  - {name: high_cost, commission_value: 0.10, slippage: 4}
finalist_diagnostics: true
replay_bars: 100
create_paper_candidate: true                  # verdict only; never trades
chunk_size: 5
max_cells: 1000
employee_id: 7                                # omit when pane context provides it
book_id: 3                                    # omit when pane context provides it
```

Parameter keys may be the visible TradingView input names or actual input IDs.
Do not guess strategy-specific inputs. If the user has not named an axis, ask
for it or run a one-cell baseline with the current settings. Values in
`fixed_parameters` are still applied through TradingView but do not multiply
the cell count.

Use these defaults unless the user overrides them:

- current chart symbol and candle timeframe;
- `horizon: last_365d` (one year of Deep Backtesting history);
- `rank_by: drawdown_adjusted_profit_factor`;
- `min_trades: 20`, `finalists: 3`;
- `chunk_size: 5` for historical-range jobs (`27` only for `standard_chart`);
- `max_cells: 1000`.

Choose validation deliberately:

- `none`: exploratory screening only; do not call its winner validated.
- `holdout`: select parameters on the earlier 80% and test only the finalists
  on the untouched final 20%. Prefer this as the minimum serious validation.
- `walk_forward`: use an expanding training window followed by the next unseen
  test window, repeated across 3 folds by default. Prefer this when the horizon
  contains enough history and trades.

Never use out-of-sample results to alter parameters inside the same job. If a
test result inspires a new grid, treat that as a new hypothesis and new job.

Run post-selection checks only on finalists:

- use named `regime_windows` for crashes, trends, and sideways periods;
- use realistic and adverse `cost_scenarios` without ranking low costs as an
  optimizable parameter;
- enable `finalist_diagnostics` to inspect trade concentration and equity;
- enable `replay_bars` as an additional forward-style chart check;
- use `create_paper_candidate` only for an auditable readiness verdict. It
  never authorizes or places a paper trade.

Keep candle resolution and test horizon distinct. `timeframes: [60, D, W]`
tests different bar sizes. `horizon` controls historical coverage and accepts
`last_7d`, `last_30d`, `last_90d`, `last_365d`, `entire_history`, or
`standard_chart`. Use `date_from` plus `date_to` for a custom regime; they
override the preset. Use `standard_chart` only when the user explicitly wants
whatever history is currently loaded on the chart.
Calculate the conceptual cell count as:

`product(sweep value counts) × symbols × timeframes`.

Fixed parameters do not multiply it. Do not silently shrink the experiment;
the tool will split it into runtime-safe chunks. If it exceeds `max_cells`,
help the user narrow the hypothesis before starting.
Validation repeats the training grid per fold and adds finalist-only test
cells, so use the tool's reported upper bound when checking the budget.

## Workflow

### Start a new job

Call `backtest` once with `action: start` and the complete specification. Do
not create office records first; the tool does that atomically and returns the
canonical `job_id`.

```yaml
action: start
hypothesis: "Entry mode changes robustness; RR and ATR stop stay fixed."
fixed_parameters:
  "RR target": 2.0
  "ATR stop multiplier": 2.0
sweep_parameters:
  "Entry mode": [breakout, pullback, hybrid]
  "Swing length": [10, 20, 30]
horizon: last_365d
validation_mode: holdout
out_of_sample_pct: 20
finalist_diagnostics: true
create_paper_candidate: true
rank_by: drawdown_adjusted_profit_factor
```

If no compiled strategy is found, stop and tell the user to add/compile one on
the current chart. Do not generate a replacement.

### Finish a running job

When `status` is `running`, preserve the returned `job_id` and call:

```yaml
action: continue
job_id: <returned job id>
```

Each call completes one chunk and writes progress back to the same record/run.
Repeat only while the returned status is `running`. Never call `start` again to
resume. A later invocation can use `status` first and then continue the same
job, so tool or model timeouts do not lose completed work.

### Read or cancel

Use `status` for compact progress. Use `results` after completion, or when the
user explicitly asks for all persisted cells. Use `cancel` only when asked to
stop the job.

## Interpretation

Report:

1. canonical `job_id`, research `record_id`, and workflow run ID;
2. tested strategy, symbols, candle timeframes, historical horizon, fixed
   values, and swept axes;
3. completed/total chunks and cell classifications;
4. the deterministic leaderboard and ranking rule;
5. for validation jobs, each train/test window, selected training finalists,
   and their untouched out-of-sample results;
6. whether the winner is stable across nearby parameters, folds, and symbols;
7. any thin samples or missing results;
8. regime, cost, trade/equity, and replay findings when requested;
9. paper-candidate status and every failed readiness criterion;
10. the next validation step.

Do not claim that the top cell is a deployable strategy. A useful winner has
enough trades, tolerable drawdown, and a neighborhood of similar results—not
one isolated peak. For walk-forward jobs, emphasize consistency across unseen
folds rather than the single best fold.

Always end with the tested scope: current compiled Pine strategy, TradingView
Deep Backtesting range (or explicit standard-chart opt-out), screening or
chronological validation as applicable.
