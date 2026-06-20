---
name: "backtest-strategy-lab"
description: "Generate, run, compare, and critically review a curated catalog of systematic Pine strategies using TradeCLI's TradingView tools. Use for backtesting time-series momentum, Donchian breakout, short-term mean reversion, volatility-contraction breakout, or opening-range breakout strategies across symbols, timeframes, nearby parameters, and execution-cost assumptions; collecting Strategy Tester metrics, trades, and equity curves; and recording a research verdict. This is a single-symbol-per-run research workflow, not a portfolio simulator or industrial optimizer."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [tv_health_check, tv_chart_get_state, tv_chart_get_visible_range, tv_chart_set_symbol, tv_chart_set_timeframe, tv_ui_open_panel, tv_data_get_strategy_results, tv_data_get_trades, tv_data_get_equity, tv_capture_screenshot, pine_get_source, pine_set_source, pine_analyze, pine_check, pine_smart_compile, pine_get_errors, pine_get_console, aitradingoffice_hf, aitradingoffice_workflows]
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

# Backtest Strategy Lab

## Intent

Evaluate systematic Pine strategies on TradingView with a reproducible test
matrix, realistic assumptions, trade-level evidence, and a skeptical research
verdict. Compare like with like and preserve enough context to reproduce every
run.

## Scope

- Generate and run one or more catalog Pine `strategy(...)` scripts.
- Accept an existing or user-supplied Pine strategy when explicitly requested.
- Vary strategy, symbol, timeframe, nearby parameters, or execution costs.
- Execute each matrix cell as an independent TradingView Strategy Tester run.
- Collect metrics, trades, equity curve, configuration, and visible history.
- Compare runs and recommend `reject`, `revise_and_retest`, `promising`, or
  `approve_for_paper_validation`.

This skill does not perform multi-asset portfolio simulation, survivorship-safe
universe testing, exhaustive optimization, or unattended live trading.

## Strategy catalog

These are transparent research archetypes used in professional systematic
trading. Their inclusion does not imply that they possess an edge on a given
market. Generate Pine v6 from the exact rules below; do not substitute a
similarly named public TradingView indicator.

### `time-series-momentum`

Purpose: capture persistent directional trends.

- Best starting scope: liquid indices, futures, ETFs, and large-cap equities;
  `60`, `D`, or `W`.
- Long entry: fast EMA crosses above slow EMA.
- Short entry: fast EMA crosses below slow EMA, only when `allow_short = true`.
- Exit: opposite crossover or ATR protective stop.
- Defaults: fast EMA `50`, slow EMA `200`, ATR `14`, stop `3 ATR`.
- Neighbor test: fast `[40, 50, 60]`; slow `[150, 200, 250]`.
- Known weakness: repeated whipsaws in range-bound markets.

### `donchian-trend-breakout`

Purpose: capture medium-term price breakouts in the style of managed-futures
trend systems.

- Best starting scope: liquid futures, indices, commodities, and large-cap
  equities; `60`, `D`, or `W`.
- Long entry: close exceeds the prior `entry_length`-bar high.
- Short entry: close falls below the prior `entry_length`-bar low, only when
  `allow_short = true`.
- Long exit: close falls below the prior `exit_length`-bar low.
- Short exit: close exceeds the prior `exit_length`-bar high.
- Use prior-channel values with `[1]`; never include the current bar in its own
  breakout threshold.
- Defaults: entry length `55`, exit length `20`, ATR `20`, emergency stop `2 ATR`.
- Neighbor test: entry `[40, 55, 70]`; exit `[15, 20, 25]`.
- Known weakness: false breakouts and large giveback after trend exhaustion.

### `short-term-mean-reversion`

Purpose: test temporary oversold/overbought dislocations inside a liquid,
non-collapsing market.

- Best starting scope: liquid index ETFs and large-cap equities; `60` or `D`.
- Long regime: close above the `200`-bar SMA.
- Long entry: RSI falls below `25` and close is below the lower Bollinger Band.
- Long exit: close reaches the Bollinger midline, RSI exceeds `50`, the time
  stop fires, or the ATR stop fires.
- Optional short regime: close below the `200`-bar SMA, RSI above `75`, and
  close above the upper band.
- Defaults: RSI `2`, Bollinger length `20`, width `2`, time stop `10` bars,
  emergency stop `2 ATR`.
- Neighbor test: RSI entry `[20, 25, 30]`; band width `[1.5, 2.0, 2.5]`.
- Known weakness: averaging into structural breaks; the regime and hard stop
  must not be removed.

### `volatility-contraction-breakout`

Purpose: test expansion after compressed volatility.

- Best starting scope: liquid equities, indices, and futures; `15`, `60`, or
  `D`.
- Compression: Bollinger Band width is below its own rolling percentile proxy
  or below `compression_factor ×` its moving average.
- Long entry: after compression, close breaks above the prior `breakout_length`
  high with volume above its moving average.
- Optional short entry: symmetric downside break.
- Exit: ATR trailing stop, opposite breakout, or maximum holding period.
- Defaults: Bollinger `20/2`, width average `100`, compression factor `0.75`,
  breakout length `20`, volume average `20`, ATR trail `3`, max hold `40` bars.
- Neighbor test: compression `[0.65, 0.75, 0.85]`; breakout `[15, 20, 30]`.
- Known weakness: failed expansion in low-participation or news-distorted moves.

### `opening-range-breakout`

Purpose: test intraday continuation after the market establishes an opening
range.

- Best starting scope: highly liquid Indian indices, index futures, and
  large-cap equities; `5` or `15`.
- Session timezone: `Asia/Kolkata`.
- Opening range: high/low from `09:15` through `09:30` by default.
- Long entry: after the range closes, price closes above opening-range high and
  volume exceeds its `20`-bar average.
- Short entry: symmetric close below opening-range low.
- Stop: opposite side of the opening range, capped by a configurable maximum
  risk distance.
- Exit: fixed R target, trailing stop, or mandatory square-off at `15:15`.
- Defaults: range end `09:30`, volume multiplier `1.2`, target `2R`, one entry
  per direction per session.
- Neighbor test: range end `[09:30, 09:45]`; volume multiplier
  `[1.0, 1.2, 1.5]`.
- Known weakness: gap days, opening noise, and severe slippage around events.

## Common Pine contract

Every generated catalog strategy must:

- start with `//@version=6` and declare `strategy(...)`;
- expose catalog parameters with `input.*`;
- default to `pyramiding = 0`, `calc_on_every_tick = false`, and
  `process_orders_on_close = true`;
- expose `allow_short` and the date filter as inputs;
- encode commission, slippage, and default position size as explicit constants
  in `strategy(...)` for each generated test cell;
- use `strategy.percent_of_equity` with default size `10%`;
- include at least one hard protective exit and close all intraday positions at
  the declared square-off time;
- avoid lookahead, future indexing, and current-bar channel leakage;
- plot the decision levels needed to visually audit entries and exits;
- use stable order ids so `tv_data_get_trades` is interpretable.

Default execution assumptions when the user supplies none:

```yaml
commission_type: strategy.commission.percent
commission_value: 0.03
slippage_ticks: 2
position_size_pct: 10
pyramiding: 0
process_orders_on_close: true
calc_on_every_tick: false
```

Label these as generic research defaults, not a complete broker charge model.
For Indian-market conclusions, run at least one higher-cost cell.

## Hard limits

- Run at most **12 matrix cells** per invocation.
- Change one experimental axis at a time unless the user explicitly requests a
  compact strategy comparison.
- Never place or modify a trade. `allowed_ledgers` is empty.
- Never call `pine_save` or `pine_publish`.
- Never describe TradingView's loaded history as a user-selected date range
  unless it was verified.

## Intake

Resolve:

```yaml
objective: baseline | compare_strategies | parameter_sensitivity | cost_stress
strategies:
  - id: time-series-momentum
    source: catalog | current_chart | supplied_pine
symbols: [NSE:RELIANCE]
timeframes: [D]
variants:
  - id: baseline
    parameters: {}
    commission: strategy_default
    slippage: strategy_default
benchmark: optional descriptive benchmark
max_cells: 12
```

If the request is underspecified, show the five catalog choices. If the user
asks to "backtest strategies" without choosing, run a three-family comparison
using `time-series-momentum`, `donchian-trend-breakout`, and
`short-term-mean-reversion` on the current symbol and timeframe when compatible.

Before running, state the exact matrix and total cell count. Reduce a larger
request to the smallest decision-useful subset.

## Workflow

### 1. Rehydrate and preflight

1. If invoked as an AITradingOffice run, call `get_run` first and preserve its
   identifiers, params, and existing log.
2. Call `tv_health_check`. Stop cleanly if TradingView Desktop is unavailable.
3. Read `tv_chart_get_state` and `tv_chart_get_visible_range`.
4. Open the Pine Editor and Strategy Tester with `tv_ui_open_panel`.
5. Record symbol, timeframe, visible history, strategy identity, timestamp, and
   TradingView target id.

### 2. Resolve and audit each strategy

For each distinct strategy:

1. Generate Pine v6 from the selected catalog contract, obtain explicit
   existing source with `pine_get_source`, or use supplied Pine.
2. Confirm it declares `strategy(...)`, not `indicator(...)`.
3. Check entry, exit, sizing, pyramiding, order timing, commission, and slippage.
4. Reject or repair lookahead, repainting, missing exits, silent zero costs,
   ambiguous fills, and changes outside the stated experimental axis.
5. Run `pine_analyze` and `pine_check`.
6. Inject with `pine_set_source`, compile with `pine_smart_compile`, and inspect
   `pine_get_errors` / `pine_get_console` when compilation is unclear.

Do not proceed with a strategy whose implementation cannot be reconciled with
its stated rules.

### 3. Execute the matrix

For every approved cell:

1. Set symbol and timeframe.
2. Apply only that cell's strategy, parameter, or cost change.
3. Compile and confirm the strategy is active.
4. Open Strategy Tester.
5. Collect `tv_data_get_strategy_results`, `tv_data_get_trades` (up to 200),
   `tv_data_get_equity`, and `tv_chart_get_visible_range`.
6. Optionally capture the Strategy Tester for audit evidence.
7. Store raw values before interpreting them.

If a cell returns no results, check compilation, strategy type, loaded history,
and whether any trades fired. Mark it `no_result`; do not silently drop it.

### 4. Normalize each result

```yaml
cell_id:
strategy_id:
symbol:
timeframe:
history:
parameters:
execution_assumptions:
metrics:
  net_profit:
  net_profit_pct:
  total_trades:
  win_rate:
  profit_factor:
  max_drawdown:
  average_trade:
  largest_win:
  largest_loss:
  sharpe:
  sortino:
trade_evidence:
  retrieved:
  profitable:
  losing:
equity_evidence:
  points_retrieved:
warnings: []
status: completed | no_result | audit_failed
```

Use `not_returned` for unavailable fields.

### 5. Apply research checks

Flag:

- fewer than 30 trades;
- zero losing trades;
- win rate above 75% with fewer than 50 trades;
- negligible drawdown with strong returns;
- inconsistency between headline metrics and the trade list;
- one or a few trades dominating total profit;
- material deterioration under nearby parameters or higher costs;
- a small timing change destroying the result;
- inconsistent history across comparison cells;
- results that cannot be reproduced after recompilation.

Separate implementation validity, simulation validity, and evidence of edge.

### 6. Compare and decide

Compare only cells sharing compatible history, sizing, and execution
assumptions. Lead with robustness and drawdown, not highest return.

Use exactly one verdict:

- `reject`
- `revise_and_retest`
- `promising`
- `approve_for_paper_validation`

Approval means paper validation only, never readiness for live capital.

### 7. Persist and finish

1. Create an employee-owned AITradingOffice research record containing the
   hypothesis, strategy ids, matrix, assumptions, comparison, warnings, verdict,
   and next experiment.
2. If invoked as a workflow run, write the normalized result packet and
   research-record id into the run log, then mark it `done`.
3. If persistence fails, return the report marked `not_persisted`.

## Output

Return the matrix and reproducibility context, implementation audit, comparison
table, trade/equity findings, limitations, verdict, one next experiment, and
AITradingOffice identifiers when persisted.

Always end with the tested scope: TradingView, loaded history, one symbol per
cell, and no portfolio-level inference.
