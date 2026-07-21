---
name: "hedge-fund-short-mis-desk"
description: "Run the Hedge Fund Short MIS desk as a daily intraday short-equity paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow bearish candidates, validate liquid Indian stocks for same-day breakdown, failed-breakout, VWAP-rejection, or weak-sector setups, size every tranche through strategy, enter and exit only through the deterministic trade pipeline, journal the fund-book transactions, and report results back to the hedge-fund office."
mandate:
  kind: "routine"
  persona: "hedgefund"
  roles: [trader]
  risk_profile: "aggressive"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, scan, strategy, enter_trade, exit_trade, news_fetch, browser, tv_health_check, tv_symbol_search, tv_symbol_info, tv_chart_set_symbol, tv_chart_set_timeframe, tv_chart_manage_indicator, tv_quote_get, tv_depth_get, tv_technicals_get, tv_data_get_ohlcv, tv_data_get_study_values, tv_news_list, tv_news_read, hedgefund_report, watch_schedule]
  allowed_ledgers: [equities]
  limits:
    max_trades_per_tick: 3
    max_notional: 250000
    paper_only: true
  trigger: { kind: cron, expr: "25 9 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Short MIS Desk

## Intent

Trade only same-day bearish equity setups in liquid Indian names. Protect the
fund when breadth is weak and exploit intraday downside without overnight cash
short exposure. The only trade-ledger path is `strategy` to `enter_trade` to
`exit_trade`.

## Setup

1. Rehydrate the assigned run with `aitradingoffice_workflows` when present.
2. Read the CEO allocation, fund cash and P&L, open equity positions, today's
   transactions, relevant transaction records, and recent Short MIS records
   through `aitradingoffice_hf`.
3. Read the latest `daily_shared_screener` record or
   `screener-conviction-workflow` run.
4. Call `tv_health_check` and require the pane's runtime-assigned `target_id`.
   Before a candidate deep check, set its symbol and timeframe on that target.
   Pass the same `target_id` to every stateful TradingView call that accepts it.
   If no target is assigned, stop rather than read another
   pane's chart. Do not use raw Pine or UI automation tools.
5. The manifest trigger is schedule metadata, not a live cron. A manual
   invocation executes one tick. Call `watch_schedule` only when the user
   explicitly asks for repetition; it creates an in-process interval that
   fires only while the current scheduler/runtime remains alive.

## Tick

1. Establish the current India-market timestamp and freshness from TradingView
   state, quotes, and OHLCV. If the session or data date is unclear, open no new
   short. Outside cash-market hours, manage existing exits and records only.
2. Compute effective desk capacity as the minimum of CEO allocation, manifest
   `max_notional`, available fund cash, and the remaining daily-risk budget,
   after accounting for existing desk exposure.
3. Start with `short_mis_candidates` from the shared screener. If it is stale or
   unavailable, run a high-level bearish `scan`. Label and explain any fresh
   post-screener fallback candidate in the office record.
4. Require safe TradingView and news evidence for:
   - a lower-high/lower-low structure, failed breakout, or confirmed breakdown;
   - price below VWAP or a clear rejection from VWAP/resistance;
   - weak relative strength versus the index or sector;
   - liquid, current quotes and feasible India MIS cash-short treatment;
   - an explicit stop above the rejection or breakdown level.
5. If broad market strength conflicts with the short, require a stock-specific
   catalyst or failed-breakout structure; otherwise record `no_short_edge`.
6. Call `aitradingoffice_hf` with `action: list_clients`. Select a concrete
   client whose status is active and whose allowed instruments and constraints
   permit this MIS equity short. If none qualifies or the choice is ambiguous,
   stop. Call `action: check_tradeable` with that client's id, then reconcile
   the proposal with fund cash, current exposure, CEO allocation, notional
   limits, and risk at stop. Skip and report any failed or unreadable gate;
   never retry around it.
7. Size the first tranche with `strategy` using `flavor: hedgefund`,
   `instrument: equity`, `side: sell`, `product_type: MIS`, explicit stop and
   targets, tranche risk, and a stable run/symbol/tranche idempotency key. It
   should normally be 30-50% of intended size. Omit an entry-price override so
   the pipeline uses a live TradingView mark.
8. Review the risk card, then pass the returned `enter_trade` payload unchanged
   to `enter_trade`. Never write the position through `aitradingoffice_hf` or
   invent a fill. Call `aitradingoffice_hf` with
   `action: add_transaction_record`, `instrument: equity`, and the returned
   equity transaction id:

```yaml
desk: short_mis
thesis:
entry_price:
quantity:
notional:
initial_tranche_pct:
add_rules:
reduce_rules:
stop:
target:
time_exit: "15:10 Asia/Kolkata"
max_loss:
idempotency_key:
ceo_allocation_record:
screener_record:
```

9. Add only after downside follow-through, a failed reclaim, or newly weak
   breadth confirms the trade. Every add must re-read exposure and rerun
   `strategy` for residual risk with a distinct tranche key before using its
   exact `enter_trade` payload. Never add to a losing short.
10. Monitor price, VWAP, breadth, spread, and the persisted plan. Reduce or cover
    on a VWAP reclaim, failed breakdown, adverse breadth reversal, stop, target,
    failed add, or 15:10 square-off. Run `exit_trade` with `dry_run: true` first;
    only an unambiguous, quantity-reconciled verdict may be repeated with
    `dry_run: false`.
11. Update the record with additions, covers, exit evidence, and remaining risk.
    Report entries, exits, rejected shorts, risk remaining, screener usefulness
    or misses, and one learning note with `hedgefund_report`.

## Guardrails

- Do not short against strong breadth without a stock-specific catalyst or
  failed-breakout structure.
- Never average up into a losing short, bypass the India MIS rule, or carry a
  cash-equity short overnight.
- Do not deploy the full allocation in one tranche unless the transaction
  record explicitly justifies it.
- Skip the trade when identity, session, liquidity, data freshness, or short
  feasibility cannot be confirmed.
- Each successful `enter_trade` tranche counts toward `max_trades_per_tick`.
- Every entry must be sized by `strategy` and booked by `enter_trade`; every
  cover must be settled by `exit_trade`.
