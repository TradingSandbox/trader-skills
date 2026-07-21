---
name: "hedge-fund-commodity-desk"
description: "Run the Hedge Fund Commodity desk as a daily MCX/NCDEX commodity-futures paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital for crude, gold, silver, natural-gas, or other supported commodity futures; resolve the exact venue and contract with TradingView reads, size every tranche through strategy, enter and exit only through the deterministic trade pipeline, monitor the fund-book position, journal it, and report results. Commodity-option ideas are analysis-only and cannot be booked by this skill."
mandate:
  kind: "routine"
  persona: "hedgefund"
  roles: [trader]
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, strategy, enter_trade, exit_trade, news_fetch, browser, tv_health_check, tv_symbol_search, tv_symbol_info, tv_chart_set_symbol, tv_chart_set_timeframe, tv_quote_get, tv_depth_get, tv_technicals_get, tv_data_get_ohlcv, tv_futures_curve, tv_news_list, tv_news_read, hedgefund_report, watch_schedule]
  allowed_ledgers: [futures]
  limits:
    max_trades_per_tick: 2
    max_notional: 300000
    paper_only: true
  trigger: { kind: cron, expr: "0 11 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Commodity Desk

## Intent

Trade commodity futures in the hedge-fund paper book only when contract
identity, venue, session, liquidity, fund exposure, and macro/event context are
clear. Commodity-option proposals may be analyzed and reported but are never
sent to the trade pipeline. The only futures trade-ledger path is `strategy` to
`enter_trade` to `exit_trade`.

## Setup

1. Rehydrate workflow state with `aitradingoffice_workflows` when present.
2. Through `aitradingoffice_hf`, read CEO allocation, fund cash and P&L,
   commodity-futures positions, recent transactions, and their
   transaction records.
3. Call `tv_health_check` and require the pane's runtime-assigned `target_id`.
   Before a deep check, set the commodity symbol and timeframe on that target.
   Pass the same `target_id` to every stateful TradingView call that accepts it.
   If no target is assigned, stop rather than read another pane's chart. Use
   `news_fetch` or `browser` for decision-relevant
   macro, inventory, supply, currency, or event context.
4. The manifest trigger is schedule metadata, not a live cron. A manual
   invocation executes one tick. Call `watch_schedule` only when the user
   explicitly asks for repetition; it creates an in-process interval that
   fires only while the current scheduler/runtime remains alive.

## Tick

1. Establish the exact commodity session, current timestamp, source freshness,
   expiry, and time to cutoff. Do not assume equity-market hours. An ambiguous
   session or stale contract quote blocks new entries but not risk reduction.
2. Compute effective capacity as the minimum of CEO allocation, manifest
   `max_notional`, available fund cash, and remaining daily-risk budget after
   all open commodity exposure.
3. Resolve the exact exchange-qualified underlying, contract month, expiry,
   trading symbol, and lot size with `tv_symbol_search`, `tv_symbol_info`, quote,
   and OHLCV reads. Never infer a tradable contract from a continuous symbol
   alone.
4. Build a futures thesis from trend, volatility, price/participation evidence,
   current liquidity, and sourced macro/event context. If the assignment asks
   for a commodity option, provide analysis and report that the idea is
   unbookable under this futures-only mandate; do not send it to `strategy`.
5. Reject unresolved venue/contract identity, poor quote quality, unacceptable
   event risk, stale data, or notional and stop-loss exposure above the
   allocation.
6. Call `aitradingoffice_hf` with `action: list_clients`. Select a concrete
   client whose status is active and whose allowed instruments and constraints
   permit this commodity future. If none qualifies or the choice is ambiguous,
   stop. Call `action: check_tradeable` with that client's id, then reconcile
   the proposal with fund cash, existing exposure, CEO allocation, manifest
   notional, and maximum loss. A failed or unreadable gate means skip and report.
7. Size the first 30-50% tranche with `strategy` using `flavor: hedgefund`, the
   `instrument: future`, exact `underlying`, `expiry`, verified `lot_size` and
   `tv_symbol`, the decided side, explicit
   stop and targets, tranche risk, and a stable run/contract/tranche
   idempotency key. Omit an entry-price override.
8. Review the risk card and returned `enter_trade` payload. It must preserve the
   resolved commodity venue and contract consistently. If the payload omits or
   conflicts with the MCX/NCDEX venue, stop and report the contract mismatch;
   never patch around it with a guessed field. When it is venue-consistent and
   within limits, pass it unchanged to `enter_trade`.
9. Call `aitradingoffice_hf` with `action: add_transaction_record`,
   `instrument: future`, and the returned future transaction id:

```yaml
desk: commodity
commodity:
instrument: future
exchange:
contract:
expiry:
thesis:
entry:
initial_tranche_pct:
add_rules:
reduce_rules:
notional_exposure:
stop:
target:
time_or_session_exit:
macro_risk:
idempotency_key:
```

10. Add only after contract price confirms the thesis, liquidity remains sound,
    and combined risk remains inside the original budget. Re-read exposure,
    rerun `strategy` for residual risk with a distinct tranche key, and use only
    the resulting venue-consistent `enter_trade` payload.
11. Monitor contract OHLCV/quote, spread, fund exposure, expiry/session cutoff,
    and event risk. Reduce or exit on thesis failure, stop, target, failed add,
    widening spread, excessive exposure, changed macro risk, or cutoff. Call
    `exit_trade` with `dry_run: true` first; repeat with `dry_run: false` only
    after an unambiguous verdict and quantity reconciliation.
12. Update the transaction record with additions, reductions, exit evidence,
    and remaining exposure. Send `hedgefund_report` with positions, P&L,
    rejected ideas, data gaps, and contract/session lessons.

## Guardrails

- Never trade until exchange, exact contract, expiry, lot size, and quote symbol
  agree. A continuous chart alone is not a booking identifier.
- Commodity options are analysis-only and unbookable under this skill.
- Do not deploy the full allocation in one tranche unless the record explicitly
  justifies why staged execution adds no risk benefit.
- Treat overnight commodity risk separately from cash-equity risk, and document
  the session-specific exit plan.
- Do not transplant stock/index option assumptions to commodities without
  contract-specific evidence.
- Treat the AITradingOffice futures ledger as paper/audit evidence only. Its
  account-ledger amount may be zero, and it does not simulate broker margin,
  collateral, daily settlement, or full mark-to-market accounting. Size from
  notional and stop risk; do not present its cash or P&L as broker controls.
- Each successful `enter_trade` tranche counts toward `max_trades_per_tick`.
- Every entry must be sized by `strategy` and booked by `enter_trade`; every
  settlement must use `exit_trade`.
