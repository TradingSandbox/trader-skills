---
name: "hedge-fund-stock-futures-desk"
description: "Run the Hedge Fund Stock Futures desk as a daily paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow stock-futures candidates and trade liquid Indian single-stock futures long or short from underlying trend, contract identity, liquidity, participation, and notional exposure; size every tranche through strategy, enter and exit only through the deterministic trade pipeline, journal the fund-book transactions, and report results."
mandate:
  kind: "routine"
  persona: "hedgefund"
  roles: [trader]
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, scan, strategy, enter_trade, exit_trade, news_fetch, browser, tv_health_check, tv_symbol_search, tv_symbol_info, tv_chart_set_symbol, tv_chart_set_timeframe, tv_quote_get, tv_depth_get, tv_technicals_get, tv_data_get_ohlcv, tv_futures_curve, tv_news_list, tv_news_read, hedgefund_report, watch_schedule]
  allowed_ledgers: [futures]
  limits:
    max_trades_per_tick: 2
    max_notional: 500000
    paper_only: true
  trigger: { kind: cron, expr: "10 10 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Stock Futures Desk

## Intent

Use single-stock futures for liquid, high-conviction directional exposure when
futures are cleaner than cash equity or long options. Contract, expiry,
notional, and stop risk must be explicit. The only trade-ledger path is
`strategy` to `enter_trade` to `exit_trade`.

## Setup

1. Rehydrate workflow state with `aitradingoffice_workflows` when present.
2. Through `aitradingoffice_hf`, read CEO allocation, fund cash and P&L, open
   stock-future positions, recent future transactions and records, and the
   latest `daily_shared_screener` or `screener-conviction-workflow` run.
3. Call `tv_health_check` and require the pane's runtime-assigned `target_id`.
   Before a deep check, set the underlying or exact contract symbol and
   timeframe on that target. Pass the same `target_id` to every stateful
   TradingView call that accepts it. If no target is assigned,
   stop rather than read another pane's chart.
4. The manifest trigger is schedule metadata, not a live cron. A manual
   invocation executes one tick. Call `watch_schedule` only when the user
   explicitly asks for repetition; it creates an in-process interval that
   fires only while the current scheduler/runtime remains alive.

## Tick

1. Establish the current India-market timestamp, data freshness, futures
   expiry, and session cutoff. If the data date, contract month, or session is
   ambiguous, open no new future.
2. Compute available capacity as the minimum of CEO allocation, manifest
   `max_notional`, available fund cash, and remaining daily-risk budget after
   existing future exposure and concentration.
3. Start with `stock_future_candidates` from the shared screener. If it is stale
   or missing, run a high-level `scan` over liquid F&O-eligible underlyings. A
   fresh CEO focus name may be added only with a record explaining its source
   and reconciliation against the shared list.
4. Resolve the exact futures contract, expiry, exchange-qualified underlying,
   TradingView quote symbol, and current lot size with symbol search/info and
   quote/OHLCV reads. Never infer the booking contract from a continuous symbol
   alone.
5. Require:
   - a clear long or short structure on the underlying and contract;
   - current, liquid quote and participation evidence;
   - contract/basis behavior that does not contradict the thesis;
   - notional and concentration inside allocation;
   - an explicit stop, targets, and roll/expiry plan.
6. Check decision-relevant news, then call `aitradingoffice_hf` with
   `action: list_clients`. Select a concrete client whose status is active and
   whose allowed instruments and constraints permit this stock future. If none
   qualifies or the choice is ambiguous, stop. Call `action: check_tradeable`
   with that client's id, then reconcile the proposal with current exposure,
   fund cash, CEO allocation, manifest notional, and maximum loss. A failed or
   unreadable gate means skip and report; never retry around it.
7. Size the first 30-50% tranche with `strategy` using `flavor: hedgefund`,
   `instrument: future`, the exact contract `symbol`, decided `side`,
   exchange-qualified `underlying`, `expiry`, verified `lot_size` and
   `tv_symbol`, explicit stop and targets, tranche risk, and a stable
   run/contract/tranche idempotency key. Omit an entry-price override.
8. Review the risk card and verify that the returned `enter_trade` payload
   preserves contract, venue, expiry, lot size, and exposure. If valid, pass it
   unchanged to `enter_trade`; never construct a futures trade through
   `aitradingoffice_hf` or invent a fill. Call `aitradingoffice_hf` with
   `action: add_transaction_record`, `instrument: future`, and the returned
   future transaction id:

```yaml
desk: stock_futures
direction:
underlying:
contract:
expiry:
lot_size:
notional_exposure:
initial_tranche_pct:
add_rules:
reduce_rules:
stop:
target:
roll_or_expiry_plan:
risk_at_stop:
idempotency_key:
screener_record:
```

9. Add only after the underlying and contract confirm direction, participation
   remains healthy, and combined risk remains inside the original budget.
   Re-read exposure, rerun `strategy` for residual risk with a distinct tranche
   key, and use only that exact `enter_trade` payload.
10. Monitor underlying and contract OHLCV/quotes, basis behavior, liquidity,
    expiry, and fund exposure. Reduce or exit on invalidation, stop, target,
    failed add, adverse basis/liquidity change, excessive concentration, CEO
    allocation reduction, or the roll/expiry deadline. Call `exit_trade` with
    `dry_run: true` first; repeat with `dry_run: false` only after an unambiguous
    verdict and quantity reconciliation.
11. Update the transaction record with additions, reductions, exit evidence,
    and remaining risk. Send `hedgefund_report` with positions, exposure,
    rejected trades, data gaps, and one rule improvement.

## Guardrails

- Do not use futures when the same thesis can be expressed with materially lower
  risk in cash equity or a capped-premium long option.
- Never trade until the exact contract, expiry, venue, lot size, and quote
  symbol agree; a continuous chart alone is not a booking identifier.
- Do not deploy the full allocation in one tranche unless the transaction
  record explicitly justifies it.
- Do not hold through known event or expiry risk without explicit record
  justification and an exit/roll plan.
- Do not allow one stock future to dominate total fund risk.
- Treat the AITradingOffice futures ledger as paper/audit evidence only. Its
  account-ledger amount may be zero, and it does not simulate broker margin,
  collateral, daily settlement, or full mark-to-market accounting. Size from
  notional and stop risk; do not present its cash or P&L as broker controls.
- Each successful `enter_trade` tranche counts toward `max_trades_per_tick`.
- Every entry must be sized by `strategy` and booked by `enter_trade`; every
  settlement must use `exit_trade`.
