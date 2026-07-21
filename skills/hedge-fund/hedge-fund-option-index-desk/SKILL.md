---
name: "hedge-fund-option-index-desk"
description: "Run the Hedge Fund Index Options desk as a daily paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow index bias; evaluate directional or spread ideas on NIFTY, BANKNIFTY, FINNIFTY, or other supported indices; execute only single-leg long calls or puts resolved by option_scan; size each tranche through strategy; enter and exit only through the deterministic trade pipeline; journal the fund-book transactions; and report results."
mandate:
  kind: "routine"
  persona: "hedgefund"
  roles: [trader]
  risk_profile: "aggressive"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, option_scan, strategy, enter_trade, exit_trade, news_fetch, browser, tv_health_check, tv_symbol_search, tv_symbol_info, tv_chart_set_symbol, tv_chart_set_timeframe, tv_quote_get, tv_depth_get, tv_technicals_get, tv_data_get_ohlcv, tv_options_expirations, tv_options_chain, tv_news_list, tv_news_read, hedgefund_report, watch_schedule]
  allowed_ledgers: [options]
  limits:
    max_trades_per_tick: 2
    max_notional: 250000
    paper_only: true
  trigger: { kind: cron, expr: "35 9 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Index Options Desk

## Intent

Express index views with tightly capped premium risk in the hedge-fund paper
book. Direction, volatility, expiry, contract liquidity, delta, and event risk
must be understood before entry. Executable ideas are single-leg long calls or
puts only; multi-leg structures are analysis-only because the pipeline cannot
book them atomically. The only trade-ledger path is `strategy` to `enter_trade`
to `exit_trade`.

## Setup

1. Rehydrate workflow state with `aitradingoffice_workflows` when present.
2. Through `aitradingoffice_hf`, read CEO allocation, fund cash and P&L, open
   option positions, recent option transactions and records, and the shared
   screener's current `index_bias`.
3. Call `tv_health_check` and require the pane's runtime-assigned `target_id`.
   Before a deep check, set the underlying symbol and timeframe on that target.
   Pass the same `target_id` to every stateful TradingView call that accepts it.
   If no target is assigned, stop rather than read another pane's chart.
4. The manifest trigger is schedule metadata, not a live cron. A manual
   invocation executes one tick. Call `watch_schedule` only when the user
   explicitly asks for repetition; it creates an in-process interval that
   fires only while the current scheduler/runtime remains alive.

## Tick

1. Establish the current India-market timestamp, expiry date, time to expiry,
   time to session cutoff, and data freshness. If expiry or session state is
   ambiguous, open no new structure.
2. Compute available premium and maximum-loss capacity as the minimum of CEO
   allocation, manifest `max_notional`, available fund cash, and remaining
   daily-risk budget after all open option exposure.
3. Choose one mode:
   - `directional_buy` for an executable single-leg long call or put with
     premium risk capped at the debit paid;
   - `spread_analysis_only` when the thesis would require multiple legs; record
     the analysis but send no component to `strategy` or `enter_trade`;
   - `no_trade` when direction, volatility, liquidity, or event risk conflicts.
4. Validate the underlying with the shared `index_bias`, safe TradingView
   quotes/OHLCV/indicators, and decision-relevant news. Use `option_scan` to
   resolve the exact expiry, strike, option type, contract symbol, delta/IV
   fields, and bid/ask-spread evidence. `option_scan` does not establish the
   current exchange lot size: verify that value against a current authoritative
   exchange or broker source with `browser`, record the source and as-of date,
   and stop if it cannot be verified. Never rely on a remembered/default lot.
5. Reject the proposal when liquidity is poor, spread is wide, theta dominates
   the expected move, expiry risk is unacceptable, the stop cannot be stated as
   premium/underlying/time, or aggregate maximum loss exceeds allocation.
6. Call `aitradingoffice_hf` with `action: list_clients`. Select a concrete
   client whose status is active and whose allowed instruments and constraints
   permit this long index option. If none qualifies or the choice is ambiguous,
   stop. Call `action: check_tradeable` with that client's id, then reconcile
   the proposed option with fund cash, existing exposure, CEO allocation,
   manifest notional, premium at risk, and maximum loss. A failed or unreadable
   gate means skip and report.
7. Size the first 30-50% premium tranche with `strategy` using
   `flavor: hedgefund`, `instrument: option`, `side: buy`, exact `underlying`, `expiry`,
   `strike`, `option_type`, verified `lot_size` and `tv_symbol`, explicit stop
   and targets, tranche risk, and a stable run/contract/tranche
   idempotency key. Omit an entry-price override.
8. Review the risk card and send the returned `enter_trade` payload unchanged
   to `enter_trade`. Only the selected single long contract may reach this step.
9. Call `aitradingoffice_hf` with `action: add_transaction_record`,
   `instrument: option`, and the returned option transaction id:

```yaml
desk: option_index
underlying:
expiry:
strike:
option_type:
side: buy
premium:
quantity:
lot_size_source:
initial_tranche_pct:
add_rules:
reduce_rules:
delta_and_iv:
stop:
target:
time_exit:
max_loss:
idempotency_key:
screener_bias_record:
```

10. Add only after the underlying confirms direction and volatility/liquidity
    remain consistent. Each add is a fresh long-option tranche: re-read
    exposure, rerun `strategy` with residual risk and a new key, and use only
    the resulting exact `enter_trade` payload.
11. Monitor underlying OHLCV, contract marks, spread, delta/IV evidence, time
    decay, expiry, and the persisted plan. Reduce or exit on stop, target,
    failed underlying level, volatility crush, excessive time decay, failed add,
    widening spread, or cutoff. Call `exit_trade` with `dry_run: true` first;
    repeat with `dry_run: false` only after an unambiguous verdict and quantity
    reconciliation.
12. Update transaction records with adds, reductions, exit evidence, and
    residual premium risk. Send `hedgefund_report` with actions, exposure,
    rejected structures, data gaps, and one lesson.

## Guardrails

- Execute only single-leg long calls or puts. Multi-leg structures and every
  short-option leg are analysis-only in this skill.
- Do not deploy the full allocation in one tranche unless the structure's
  maximum loss is already fully defined, small, and the record justifies it.
- Do not hold near-expiry long options without a documented reason and time
  exit.
- Stop when the current lot size cannot be verified from an authoritative
  external source.
- Do not trade when the index-direction and volatility views conflict.
- Each successful `enter_trade` tranche counts toward `max_trades_per_tick`.
- Every entry must be sized by `strategy`, booked by `enter_trade`, and settled
  by `exit_trade`; `aitradingoffice_hf` is for reads and records, not trade writes.
