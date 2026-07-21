---
name: "backtest-fast-sweep"
description: "Plan and run a durable, resumable TradingView backtest for the strategy already compiled on the current chart. Use for bounded parameter sweeps, holdout or walk-forward validation, cost/regime stress, finalist diagnostics, replay checks, and an auditable paper-candidate verdict."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  roles: [ceo, investor, trader]
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

## Boundary

Use only the high-level `backtest` tool. The Pine strategy must already be
compiled on the current TradingView chart. Do not write or repair Pine, call
raw Pine/optimizer tools, create office records yourself, or place a
trade. `backtest` owns TradingView state, immutable planning, persistence,
chunking, validation, and recovery.

## Intake

Capture a pre-result hypothesis and the smallest decision-useful experiment:

```yaml
hypothesis: "Entry mode changes robustness while risk settings stay fixed."
expected_inputs: ["Entry mode", "Swing length", "RR target"]
fixed_parameters:
  "RR target": 2.0
sweep_parameters:
  "Entry mode": [breakout, pullback, hybrid]
  "Swing length": [10, 20, 30]
symbols: [NSE:RELIANCE]
timeframes: [D]
horizon: last_365d
rank_by: drawdown_adjusted_profit_factor
min_trades: 20
finalists: 3
validation_mode: holdout
out_of_sample_pct: 20
chunk_size: 5
max_cells: 1000
```

Use visible input names or exact input IDs; never guess strategy-specific
inputs. If the user has not chosen an axis, ask once or plan a one-cell
baseline. Keep timeframe (bar size) separate from horizon (history length).
Prefer chronological holdout as the minimum serious validation and expanding
walk-forward when history and trade count support it. Never tune on untouched
test results inside the same experiment.

Optional finalist-only checks include named `regime_windows`, adverse
`cost_scenarios`, `finalist_diagnostics`, `replay_bars`, and
`create_paper_candidate`. The latter is a readiness verdict, not authority to
trade.

## Required lifecycle

1. Call `backtest` with `action: build` and `expected_inputs`. This verifies the
   compiled strategy exposes—and, by default, actually responds to—the inputs
   the experiment will vary. If the gate fails, stop; do not invent a
   replacement script.
2. Call `backtest` with `action: plan` and the complete specification. Planning
   persists an immutable preview and returns `plan_id` plus `spec_hash`.
3. In TUI or an Operator/RPC session with an approval surface, `plan` opens the
   native confirmation and starts automatically only after approval. If the
   session has no confirmation UI, leave the plan pending; do not claim it ran.
4. A separately approved start must pass only the exact returned `plan_id` and
   `spec_hash` to `action: start`. Never reconstruct or alter the approved
   specification at start time.
5. Let the durable workflow heartbeat claim and continue one persisted chunk
   per tick. Use `status` for a read-only progress check. Call `continue` with
   the `job_id` only for explicit/manual recovery when no heartbeat owner is
   advancing it; claim serialization prevents two workers from owning a chunk.
   Use `results` for the durable specification, cells, diagnostics, and
   leaderboard. Never call `start` to resume.
6. Use `cancel` only when the user asks to stop. Use `promote` only after a
   completed job has a `PROMISING` judge verdict and the user explicitly asks
   to promote it; promotion creates a validated strategy version and a
   proposed paper deployment that still requires human approval.

## Interpretation

Report the canonical plan/job/record identities, compiled strategy fingerprint,
tested symbols/timeframes/horizon, fixed and swept inputs, completed cells,
ranking rule, chronological train/test windows, finalist unseen results,
sample-size gaps, neighborhood stability, and stress/diagnostic findings.

Do not call the top cell deployable merely because it ranks first. Prefer
adequate trade count, controlled drawdown, nearby parameter stability, and
consistent unseen windows over a single isolated peak. End by stating the
exact TradingView scope and whether the result is exploratory, chronologically
validated, merely promising, or rejected.
