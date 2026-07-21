---
name: "backtest-strategy-lab"
description: "Author or select one parameterized Pine strategy, compile it through TradeCLI's high-level Pine loader when available, verify its exposed inputs, and run an approval-gated durable backtest. Use for trend, breakout, mean-reversion, volatility-expansion, and opening-range hypotheses without raw Pine or optimizer calls."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  roles: [ceo, trader]
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [load_pine_source, backtest]
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

## Goal

Turn one falsifiable strategy hypothesis into one compiled, parameterized Pine
strategy and one durable `backtest` experiment. This skill never places trades,
directly writes AITradingOffice state, drives raw TradingView UI/Pine tools, or
runs model-authored cells. The high-level tools own those boundaries.

## Choose one strategy family

- trend: moving-average or channel regime plus pullback/continuation entry;
- breakout: price/volume breakout with an explicit invalidation;
- mean reversion: deviation plus regime filter and bounded exit;
- volatility expansion: compression-to-expansion trigger;
- opening range: session-defined range, breakout, and same-session exit.

State before testing: market intuition, entry, invalidation, exit, tunable
inputs, fixed assumptions, expected failure regime, and minimum useful sample.
Do not browse results and then retrofit the hypothesis.

## Pine boundary

When `load_pine_source` is present (Hedge Fund CEO and experienced-trader
panes), author one complete Pine strategy with:

- `strategy(...)`, not an indicator;
- stable, uniquely named inputs for every intended sweep/fixed parameter;
- explicit long/short rules and realistic exits;
- no lookahead, future leaks, repaint-dependent entries, or hidden manual
  values;
- a title that identifies the hypothesis/version.

Load it only through `load_pine_source`. If that tool is unavailable, require
the user to compile the strategy on the chart first; do not fall back to
raw Pine tools, keyboard automation, or another pane's tool.

## Build, approve, and run

1. Call `backtest` with `action: build`, the script title when authored here,
   and every intended `expected_inputs` name. The build gate fingerprints the
   compiled strategy, lists its inputs, and probes that the expected inputs
   affect the report. A failed build blocks the experiment.
2. Define a bounded grid. Hold risk/execution assumptions fixed unless they
   are the hypothesis. Prefer holdout or walk-forward validation; include
   realistic adverse costs and named regime windows only as finalist checks.
3. Call `backtest` with `action: plan`. Planning persists the immutable
   specification and returns `plan_id`/`spec_hash`.
4. TUI and Operator/RPC approval surfaces show the native confirmation and
   start only when the human approves. Without a confirmation surface, keep
   the plan pending. A later approved start must use the exact `plan_id` and
   `spec_hash`; never simulate approval in chat.
5. Let the durable workflow heartbeat continue one claimed chunk per tick.
   Recover with `status`; call `continue` only for explicit/manual recovery
   when no heartbeat owner is advancing the job. Read final evidence with
   `results`; cancel only on request. Do not create a new job to resume an
   existing one.
6. Promote only on explicit request after a completed `PROMISING` verdict.
   Promotion proposes a paper deployment; it is not live-trading authority.

## Review standard

Reject results with broken build provenance, insufficient trades, unstable
neighboring parameters, unacceptable drawdown, concentrated profits, cost
fragility, or inconsistent unseen windows. Report the hypothesis, Pine title
and fingerprint, exact inputs/grid, history and bar size, validation split,
stress evidence, caveats, and the next falsification step. Never describe an
in-sample winner as validated.
