---
name: "backtest-fast-sweep"
description: "Rapidly sweep a Pine strategy across parameter grids, symbols, and timeframes using TradeCLI's deterministic strategy_fast_sweep tool over TradingView. Compile once, resolve TradingView input IDs, then execute the full grid in code — no Strategy Tester UI, no per-cell AI loop, and no per-cell recompile. Every invocation is linked to an AITradingOffice research record and workflow run. Use for fast parameter-sensitivity screens, watchlist-wide strategy screens, and quick robustness triage; promote finalists to backtest-strategy-lab for a full audit."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [tv_health_check, tv_chart_get_state, tv_chart_set_symbol, tv_chart_set_timeframe, tv_pine_set_source, tv_pine_smart_compile, tv_pine_get_errors, tv_data_get_indicator, tv_indicator_set_inputs, tv_data_get_strategy_results, tv_data_get_trades, tv_data_get_equity, strategy_fast_sweep, aitradingoffice_hf, aitradingoffice_workflows]
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
fast as TradingView can recalculate. AI defines the experiment once;
`strategy_fast_sweep` executes and ranks the cells in deterministic code. Pull
trade evidence only for the leaders, persist the experiment and execution
lineage in AITradingOffice, and hand finalists to `backtest-strategy-lab` for
the slow, audited pass.

## Relationship to backtest-strategy-lab

- **This skill**: many cheap cells, headline metrics, ranked shortlist. No
  per-cell screenshots, no per-cell history verification, no implementation
  audit beyond one compile check.
- **backtest-strategy-lab**: few expensive cells, full audit, research
  verdict, reproducibility context.

A fast-sweep ranking is never itself evidence of edge. Its only valid output
is a shortlist plus the instruction to lab-test the survivors.

## The speed contract

Every run must obey these rules — they are what makes the sweep fast and
reproducible:

1. **Compile exactly once per strategy source.** `tv_pine_set_source` +
   `tv_pine_smart_compile` is the only slow step. Every parameter you intend
   to sweep must be exposed as an `input.*` in the generated Pine so it can
   be changed without recompiling.
2. **Never open the Strategy Tester or any panel.**
   `tv_data_get_strategy_results`, `tv_data_get_trades`, and
   `tv_data_get_equity` read the strategy's report data directly from the
   chart model; no UI is required. `tv_pine_set_source` opens the Pine
   Editor on its own.
3. **Resolve inputs once.** Call `tv_data_get_indicator` with the strategy's
   `entity_id` and map requested Pine inputs to the actual TradingView IDs
   (`in_0`, `in_1`, and so on). Never guess an ID. If a requested axis cannot
   be mapped, stop before running the sweep and report the available IDs.
4. **Execute the grid once.** Call `strategy_fast_sweep` exactly once per
   compiled strategy/cost level with the resolved input-ID grid plus all
   requested symbols and timeframes. The tool owns the Cartesian product,
   chart changes, recalculation polling, stale-report detection, ranking, and
   restoration. Never hand-roll a per-cell tool loop around
   `tv_indicator_set_inputs` or `tv_batch_run`.
5. **Read only headline metrics per cell.** Pull `tv_data_get_trades` and
   `tv_data_get_equity` only for the top-ranked cells (at most 3).
   `tv_indicator_set_inputs` is allowed after the sweep solely to restore one
   of those finalists for evidence collection, never to execute the grid.
6. **Trust the runner's classifications, not optimistic inference.** The
   runner accepts a changed report only after two identical reads of the new
   fingerprint. Preserve every `thin_sample`, `suspect_stale`, and
   `no_result` count in the research record; never silently reinterpret or
   drop them.

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

- At most **1,000 cells** per synchronous `strategy_fast_sweep` invocation.
  The tool returns a compact leaderboard rather than streaming every cell
  through model context. Split larger grids into separate workflow runs.
- At most **3 recompiles** per invocation (initial + 2 cost/sizing levels).
- Never place or modify a trade. `allowed_ledgers` is empty.
- Never call `pine_save` or `pine_publish`.
- Rank by robustness-aware criteria, never by net profit alone.
- Every invocation must have both an employee-owned AITradingOffice research
  record and a workflow run. If office persistence is unavailable, the sweep
  may still finish but its report must be labeled `not_persisted`.

## Intake

Resolve:

```yaml
objective: parameter_sensitivity | symbol_screen | timeframe_screen | cost_stress
strategy:
  id: time-series-momentum
  source: catalog | current_chart | supplied_pine
parameter_grid:            # passed once to strategy_fast_sweep
  in_0: [40, 50, 60]       # actual IDs resolved after compile
  in_1: [150, 200, 250]
parameter_labels: {in_0: fast_len, in_1: slow_len}
symbols: [NSE:RELIANCE]
timeframes: [D]
cost_levels: []            # each extra level = one recompile
rank_by: drawdown_adjusted_profit_factor
max_cells: 1000
```

The runner executes the full parameter × symbol × timeframe Cartesian product;
do not reduce it to "optimize one symbol, then test the winner elsewhere"
unless the user explicitly requests that cheaper approximation. Calculate the
cell count before running. If it exceeds `max_cells`, split it into explicit
workflow runs or propose the smallest decision-useful subset.

## Workflow

### 1. Establish office lineage (once)

Every sweep gets a research record plus a workflow run before TradingView work
begins.

1. Resolve the acting hedge-fund employee and book from pane context. If they
   cannot be resolved, stop and ask for `employee_id` / `book_id`.
2. If the caller supplied `skill_id` + `run_id`, call
   `aitradingoffice_workflows.get_run`, preserve its params/log, and update it
   to `running`.
3. Otherwise list workflow skills and find `backtest-fast-sweep`. Create the
   `oneshot` skill when absent, then create a run after creating the experiment
   record below.
4. Reuse the run's `params.record_id` when it points to the intended experiment;
   otherwise create one employee-owned research record titled
   `backtest-fast-sweep: <strategy id>`:
   - `setup`: compact JSON containing objective, source/source hash when known,
     requested grid, symbols, timeframes, cost levels, ranking rule, and
     history caveat;
   - `hypothesis`: the user's pre-test hypothesis plus `status: pending`.
5. For a newly-created workflow run, put the resulting `record_id` and the
   complete sweep specification in `params`, then update the run to `running`.
6. Treat `(persona, employee_id, skill_id, run_id)` as the canonical job
   identity; include the `record_id` in every subsequent run-log entry.

If record/run creation fails, continue only when the user asked for an ad-hoc
sweep, and mark the final result `not_persisted`.

### 2. TradingView preflight (once)

1. `tv_health_check` — on failure, update the run to `failed` and stop.
2. `tv_chart_set_symbol` / `tv_chart_set_timeframe` to the base cell.

### 3. Compile (once per strategy / cost level)

1. Generate or accept the Pine source per the contract above.
2. `tv_pine_set_source`, then `tv_pine_smart_compile`; on failure read
   `tv_pine_get_errors`, repair, and retry (at most twice).
3. `tv_chart_get_state` — record the strategy's `entity_id` and confirm no
   other strategy is on the chart (two strategies make results ambiguous;
   remove or stop).
4. Call `tv_data_get_indicator(entity_id)` and read the actual input IDs and
   current values. `strategy_fast_sweep` accepts IDs, not Pine variable names.
   For supplied/current-chart strategies, require an explicit
   `{input_id: values}` grid from the caller after showing the available IDs.
   For a generated strategy, preserve the author-declared label-to-ID mapping
   alongside the generated input order; if that mapping cannot be established
   unambiguously, stop rather than guessing. Store the verified mapping in the
   experiment record.
5. Read baseline `tv_data_get_strategy_results` for the experiment record.

### 4. Sweep (one deterministic call per cost level)

Call `strategy_fast_sweep` with:

```yaml
entity_id: <compiled strategy entity id>
grid: {in_0: [40, 50, 60], in_1: [150, 200, 250]}
symbols: [NSE:RELIANCE]
timeframes: [D]
rank_by: drawdown_adjusted_profit_factor
min_trades: 20
top_n: 10
max_cells: 1000
```

Do not issue per-cell TradingView calls. Preserve the runner's totals,
classifications, restoration flag, and leaderboard exactly as returned.

For **cost levels**: repeat step 3 with the changed `strategy(...)`
constants, then rerun the winning sweep cells inside that level.

### 5. Spot-check the ranked leaders

1. Do not re-rank cells in model reasoning; use the runner's deterministic
   ranking. `thin_sample`, `suspect_stale`, and `no_result` cells are not
   finalists.
2. For at most the top 3 cells, restore that cell's symbol/timeframe/inputs,
   allow recalculation, then pull `tv_data_get_trades` (up to 50) and
   `tv_data_get_equity`; flag any cell where a few trades dominate profit,
   losses are implausibly absent, or the trade list contradicts the
   headline metrics.
3. Flag parameter cliffs: a top cell whose neighboring returned cells are materially
   worse is fragile — say so explicitly.

### 6. Persist and finish

1. Update the existing experiment record — do not create a second success
   record. Keep `setup` as the reproducibility specification and write compact
   result JSON into `hypothesis`: status, canonical job identity, TradingView
   entity id, baseline, cell totals, classifications, restoration flag,
   leaderboard, spot-check flags, shortlist, and tested-scope caveat.
2. Append the same compact result packet plus `record_id` to the workflow run
   log and mark it `done`.
3. On any terminal error, update the record with `status: failed`, append the
   stage/error to the run log, and mark the run `failed`.
4. If final persistence fails after a successful sweep, return the report
   marked `not_persisted`; never claim the job is durably complete.

## Output

Return: the canonical job identity, `record_id`, sweep definition (strategy,
grid, history caveat), runner totals/classifications, compact leaderboard,
spot-check findings, fragility flags, shortlist (0–3 cells), persistence
status, and the explicit next step — run `backtest-strategy-lab` on the
shortlist for an audited verdict.

Always end with the tested scope: TradingView loaded history only, headline
metrics per cell, spot-checked leaders only, screening — not validation.
