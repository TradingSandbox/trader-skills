---
name: "hedge-fund-long-mis-desk"
description: "Run the Hedge Fund Long MIS desk as a daily intraday long-equity paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow long candidates, validate liquid Indian stocks for same-day momentum, breakout, pullback, or VWAP-reclaim setups, size every tranche through strategy, enter and exit only through the deterministic trade pipeline, journal the fund-book transactions, and report results back to the hedge-fund office."
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
    max_notional: 300000
    paper_only: true
  trigger: { kind: cron, expr: "20 9 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Long MIS Desk

## Intent

Trade only high-quality intraday long-equity setups in the hedge-fund paper
book. Capture same-day upside momentum without carrying cash-equity risk
overnight. The deterministic trade pipeline is the only trade-ledger path:
`strategy` sizes a plan, its exact payload goes to `enter_trade`, and
`exit_trade` settles it.

## Setup

1. Rehydrate the assigned run with `aitradingoffice_workflows` when present.
2. Use `aitradingoffice_hf` to read the CEO allocation record, fund summary,
   fund book and cash, P&L, open equity positions, today's equity transactions,
   and their transaction records.
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

1. Establish the current India-market timestamp and data freshness from
   TradingView state, quotes, and OHLCV. If the session boundary or data date
   is ambiguous, open no new position. When the cash market is closed, manage
   existing records and exits only.
2. Compute the effective desk budget as the minimum of the CEO allocation,
   manifest `max_notional`, available fund cash, and the remaining daily-risk
   allowance. Include existing desk exposure before sizing anything new.
3. Start with `long_mis_candidates` from the shared screener. If that record is
   absent or stale, run a high-level `scan`. A fresh local candidate may be
   added only when its record explains why it was not available upstream.
4. Validate each candidate with safe TradingView reads and, where relevant,
   `news_fetch`. Require:
   - price above VWAP or a confirmed VWAP reclaim with volume support;
   - a clean higher-high/higher-low, breakout, or constructive pullback;
   - positive relative participation and liquid, current quotes;
   - no immediate adverse news shock;
   - an explicit stop close enough that loss at stop fits the desk budget.
5. Call `aitradingoffice_hf` with `action: list_clients`. Select a concrete
   client whose status is active and whose allowed instruments and constraints
   permit this MIS equity trade. If none qualifies or the choice is ambiguous,
   stop. Call `action: check_tradeable` with that client's id, then reconcile
   the candidate with fund cash, current exposure, CEO allocation, manifest
   notional, and risk-at-stop. A failed or unreadable gate means skip and
   report; never retry around it.
6. For an approved first tranche, call `strategy` with `flavor: hedgefund`,
   `instrument: equity`, `product_type: MIS`, the decided stop and targets,
   the tranche risk budget, and a stable idempotency key derived from the run,
   symbol, and tranche. The first tranche should normally be 30-50% of the
   intended size. Omit an entry-price override so the pipeline obtains a live
   TradingView mark.
7. Review the returned risk card. If it fits every limit, pass the returned
   `enter_trade` payload unchanged to `enter_trade`. Do not construct a ledger
   write through `aitradingoffice_hf` and do not invent a fill.
8. After a successful entry, call `aitradingoffice_hf` with
   `action: add_transaction_record`, `instrument: equity`, and the returned
   equity transaction id:

```yaml
desk: long_mis
setup:
entry_reason:
entry_price:
quantity:
notional:
initial_tranche_pct:
add_rules:
reduce_rules:
stop:
target:
time_exit: "15:10 Asia/Kolkata"
risk_at_stop:
idempotency_key:
ceo_allocation_record:
screener_record:
```

9. Add only after price confirms the thesis and combined open risk remains
   inside the original budget. Every add is a new tranche: re-read fund state,
   rerun `strategy` with the residual risk and a new stable tranche key, then
   send only that result's exact payload to `enter_trade`. Never average down.
10. Monitor current OHLCV, quote, VWAP/structure, the persisted plan, and the
    15:10 cutoff. For a stop, target, failed add, VWAP loss, reversal, reduction,
    or time exit, call `exit_trade` with `dry_run: true` first. When the verdict
    is unambiguous and quantities match the office position, repeat with
    `dry_run: false`; use partial exits for planned trims and full exits for
    invalidation or cutoff.
11. Update the transaction record with adds, reductions, exit evidence, and
    remaining risk. Send `hedgefund_report` with entries, exits, open risk,
    rejected candidates, shared-screener hits or misses, and one improvement
    note.

## Guardrails

- Never carry an MIS position overnight. If settlement fails near the cutoff,
  stop retrying blindly and send an urgent report with transaction id, dry-run
  evidence, remaining quantity, and error.
- Never average down. An add is allowed only after confirmation and only when
  combined risk stays within the original stop-risk budget.
- Do not deploy the full allocation in one tranche unless the transaction
  record explains why staged execution adds no risk benefit.
- Do not trade illiquid names, symbols with uncertain identity, stale data, or
  news-halting conditions.
- Each successful `enter_trade` tranche counts toward `max_trades_per_tick`.
- Every entry must come from `strategy`, carry a stop before booking, and flow
  through `enter_trade`; every settlement must flow through `exit_trade`.
