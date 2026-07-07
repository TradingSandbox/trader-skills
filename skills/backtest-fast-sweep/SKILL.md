---
name: "backtest-fast-sweep"
description: "Run durable, resumable TradingView-only backtests through TradeCLI's high-level backtest tool. Use when a compiled strategy is already on the current chart and the user wants a baseline, parameter sweep, symbol screen, or timeframe screen without generating or rewriting Pine. The tool chunks work below runtime limits, calls TradingView's native optimizer, restores chart state, ranks robust cells, and stores the job in AITradingOffice records and workflow runs. This skill gathers the experiment choices, starts or resumes the job, and explains the results."
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

This is fast screening, not proof of edge. TradingView is the only simulator;
results cover its loaded chart history and standard backtest model.

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
timeframes: [D]                              # optional; current timeframe by default
rank_by: drawdown_adjusted_profit_factor
min_trades: 20
finalists: 3
chunk_size: 27
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

- current chart symbol and timeframe;
- `rank_by: drawdown_adjusted_profit_factor`;
- `min_trades: 20`, `finalists: 3`, `chunk_size: 27`;
- `max_cells: 1000`.

Calculate the conceptual cell count as:

`product(sweep value counts) × symbols × timeframes`.

Fixed parameters do not multiply it. Do not silently shrink the experiment;
the tool will split it into runtime-safe chunks. If it exceeds `max_cells`,
help the user narrow the hypothesis before starting.

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
2. tested strategy, symbols, timeframes, fixed values, and swept axes;
3. completed/total chunks and cell classifications;
4. the deterministic leaderboard and ranking rule;
5. whether the winner is stable across nearby parameter values and symbols;
6. any thin samples or missing results;
7. the next validation step.

Do not claim that the top cell is a deployable strategy. A useful winner has
enough trades, tolerable drawdown, and a neighborhood of similar results—not
one isolated peak. Recommend an out-of-sample or walk-forward pass before any
paper deployment.

Always end with the tested scope: current compiled Pine strategy, TradingView
standard backtest and loaded history, screening rather than validation.
