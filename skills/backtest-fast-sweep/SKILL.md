---
name: "backtest-fast-sweep"
description: "Rapidly sweep a Pine strategy across parameter grids, symbols, and timeframes using TradeCLI's TradingView tools. Compiles the strategy once, then drives every sweep cell through instant input changes (tv_indicator_set_inputs) and batched symbol runs (tv_batch_run) — no Strategy Tester UI, no per-cell recompile. Use for fast parameter-sensitivity screens, watchlist-wide strategy screens, and quick robustness triage. This is a screening tool that ranks cells by headline metrics; promote finalists to backtest-strategy-lab for a full audit."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [tv_health_check, tv_chart_get_state, tv_chart_set_symbol, tv_chart_set_timeframe, tv_pine_set_source, tv_pine_smart_compile, tv_pine_get_errors, tv_indicator_set_inputs, tv_batch_run, tv_data_get_strategy_results, tv_data_get_trades, tv_data_get_equity, aitradingoffice_hf, aitradingoffice_workflows]
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

Screen a Pine strategy across many parameter, symbol, and timeframe cells as
fast as TradingView can recalculate. Rank cells by headline metrics, pull trade
evidence only for the leaders, and hand finalists to `backtest-strategy-lab`
for the slow, audited pass.

## Relationship to backtest-strategy-lab

- **This skill**: many cheap cells, headline metrics, ranked shortlist. No
  per-cell screenshots, no per-cell history verification, no implementation
  audit beyond one compile check.
- **backtest-strategy-lab**: few expensive cells, full audit, research
  verdict, reproducibility context.

A fast-sweep ranking is never itself evidence of edge. Its only valid output
is a shortlist plus the instruction to lab-test the survivors.

## The speed contract

Every run must obey these rules — they are what makes the sweep fast:

1. **Compile exactly once per strategy source.** `tv_pine_set_source` +
   `tv_pine_smart_compile` is the only slow step. Every parameter you intend
   to sweep must be exposed as an `input.*` in the generated Pine so it can
   be changed without recompiling.
2. **Never open the Strategy Tester or any panel.**
   `tv_data_get_strategy_results`, `tv_data_get_trades`, and
   `tv_data_get_equity` read the strategy's report data directly from the
   chart model; no UI is required. `tv_pine_set_source` opens the Pine
   Editor on its own.
3. **Parameter cells**: call `tv_indicator_set_inputs` with the strategy's
   `entity_id` (from `tv_chart_get_state`, fetched once after compiling),
   then read `tv_data_get_strategy_results`. TradingView recalculates the
   whole backtest on an input change.
4. **Symbol / timeframe cells**: call `tv_batch_run` with
   `action: "get_strategy_results"` and the symbol/timeframe lists — one
   call covers the whole grid. Set `delay_ms` to at least `1500` so each
   combo finishes recalculating before its metrics are read.
5. **Read only headline metrics per cell.** Pull `tv_data_get_trades` and
   `tv_data_get_equity` only for the top-ranked cells (at most 3).
6. **Guard against stale reads.** Recalculation is asynchronous. After every
   `tv_indicator_set_inputs` or `tv_chart_set_symbol`, wait briefly, then
   confirm the metrics actually changed versus the previous cell (total
   trades or net profit will almost always differ). If two consecutive cells
   return byte-identical metrics, re-read once before accepting; if still
   identical, mark the later cell `suspect_stale` rather than trusting it.

## What cannot be swept without recompiling

Arguments baked into the `strategy(...)` declaration — commission, slippage,
initial capital, position sizing basis, pyramiding — are fixed at compile
time. A cost-stress or sizing axis therefore needs **one recompile per
level**, then the cheap parameter/symbol sweep repeats inside each level.
Budget recompiles accordingly and say so in the plan.

## Pine contract

Generated or supplied strategies must:

- start with `//@version=6` and declare `strategy(...)`;
- expose **every sweep axis** via `input.*` (lengths, thresholds,
  multipliers, `allow_short`, date filter);
- default to `pyramiding = 0`, `calc_on_every_tick = false`,
  `process_orders_on_close = true`;
- declare explicit commission and slippage in `strategy(...)` (defaults:
  `strategy.commission.percent` at `0.03`, `slippage = 2`), labeled as
  generic research defaults;
- use `strategy.percent_of_equity` with default size `10%`;
- include at least one hard protective exit;
- avoid lookahead, future indexing, and current-bar channel leakage.

The five catalog archetypes in `backtest-strategy-lab` (time-series-momentum,
donchian-trend-breakout, short-term-mean-reversion,
volatility-contraction-breakout, opening-range-breakout) are valid sources
here; generate them from that skill's rules with all named parameters as
inputs.

## Hard limits

- At most **36 cells** per invocation (cells are cheap; reading them is not
  free — each result still costs context).
- At most **3 recompiles** per invocation (initial + 2 cost/sizing levels).
- Never place or modify a trade. `allowed_ledgers` is empty.
- Never call `pine_save` or `pine_publish`.
- Rank by robustness-aware criteria, never by net profit alone.

## Intake

Resolve:

```yaml
objective: parameter_sensitivity | symbol_screen | timeframe_screen | cost_stress
strategy:
  id: time-series-momentum
  source: catalog | current_chart | supplied_pine
parameter_grid:            # swept via tv_indicator_set_inputs
  fast_len: [40, 50, 60]
  slow_len: [150, 200, 250]
symbols: [NSE:RELIANCE]    # swept via tv_batch_run
timeframes: [D]
cost_levels: []            # each extra level = one recompile
rank_by: profit_factor_with_drawdown_penalty
max_cells: 36
```

If both a parameter grid and multiple symbols are requested, sweep parameters
on the first symbol, pick the best surviving parameter set, then batch-run
that one set across the remaining symbols. State this reduction before
running. If the request would exceed `max_cells`, propose the smallest
decision-useful subset.

## Workflow

### 1. Preflight (once)

1. If invoked as an AITradingOffice run, call `get_run` first and preserve
   its identifiers, params, and existing log.
2. `tv_health_check` — stop cleanly if TradingView Desktop is unavailable.
3. `tv_chart_set_symbol` / `tv_chart_set_timeframe` to the base cell.

### 2. Compile (once per strategy / cost level)

1. Generate or accept the Pine source per the contract above.
2. `tv_pine_set_source`, then `tv_pine_smart_compile`; on failure read
   `tv_pine_get_errors`, repair, and retry (at most twice).
3. `tv_chart_get_state` — record the strategy's `entity_id` and confirm no
   other strategy is on the chart (two strategies make results ambiguous;
   remove or stop).
4. Read baseline `tv_data_get_strategy_results` and record it as cell 0.

### 3. Sweep

For a **parameter grid**: iterate cells in a fixed documented order; per
cell call `tv_indicator_set_inputs` with only that cell's changed inputs,
apply the stale-read guard, then store the headline metrics.

For a **symbol or timeframe grid**: one `tv_batch_run` with
`action: "get_strategy_results"`, `delay_ms >= 1500`. Cells returning
errors or empty metrics are recorded as `no_result`, never dropped.

For **cost levels**: repeat step 2 with the changed `strategy(...)`
constants, then rerun the winning sweep cells inside that level.

### 4. Record each cell

```yaml
cell_id:
axis_values: {}            # exactly what changed vs baseline
metrics:
  net_profit_pct:
  total_trades:
  win_rate:
  profit_factor:
  max_drawdown:
status: completed | no_result | suspect_stale
```

Use `not_returned` for unavailable fields.

### 5. Rank and spot-check

1. Discard cells with fewer than 20 trades (report them as `thin_sample`).
2. Rank the rest by the declared criterion; a sane default is profit factor
   penalized by max drawdown, with ties broken toward more trades.
3. For at most the top 3 cells, pull `tv_data_get_trades` (up to 50) and
   `tv_data_get_equity`; flag any cell where a few trades dominate profit,
   losses are implausibly absent, or the trade list contradicts the
   headline metrics.
4. Flag parameter cliffs: a top cell whose neighboring cells are materially
   worse is fragile — say so explicitly.

### 6. Persist and finish

1. Create one employee-owned AITradingOffice research record with the
   sweep definition, the full ranked cell table, flags, and the shortlist.
2. If invoked as a workflow run, write the result packet and record id into
   the run log, then mark it `done`.
3. If persistence fails, return the report marked `not_persisted`.

## Output

Return: the sweep definition (strategy, grid, history caveat), the ranked
cell table, spot-check findings on the leaders, fragility flags, the
shortlist (0–3 cells), and the explicit next step — run
`backtest-strategy-lab` on the shortlist for an audited verdict.

Always end with the tested scope: TradingView loaded history only, headline
metrics per cell, spot-checked leaders only, screening — not validation.
