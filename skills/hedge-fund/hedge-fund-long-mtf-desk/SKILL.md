---
name: "hedge-fund-long-mtf-desk"
description: "Run the Hedge Fund Long MTF desk as a daily positional long-equity paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow MTF candidates, validate Indian stocks for multi-day to multi-week CNC paper trades, size each tranche from fund cash and risk limits through strategy, enter and exit only through the deterministic trade pipeline, monitor fund-book positions, and journal thesis improvements. This skill does not simulate MTF financing or leverage."
mandate:
  kind: "routine"
  persona: "hedgefund"
  roles: [trader]
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, scan, strategy, enter_trade, exit_trade, news_fetch, browser, tv_health_check, tv_symbol_search, tv_symbol_info, tv_chart_set_symbol, tv_chart_set_timeframe, tv_quote_get, tv_depth_get, tv_technicals_get, tv_fundamentals_get, tv_data_get_ohlcv, tv_news_list, tv_news_read, hedgefund_report, watch_schedule]
  allowed_ledgers: [equities]
  limits:
    max_trades_per_tick: 2
    max_notional: 600000
    paper_only: true
  trigger: { kind: cron, expr: "0 10 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Long MTF Desk

## Intent

Build and manage multi-day long-equity exposure in the hedge-fund paper book.
Use technical strength plus fundamental and news sanity checks, and exit when
the thesis or risk budget breaks. Despite the desk name, execution is CNC paper
only: AITradingOffice does not simulate MTF funding, leverage, borrowing cost,
or interest. The only trade-ledger path is `strategy` to `enter_trade` to
`exit_trade`.

## Setup

1. Rehydrate workflow state with `aitradingoffice_workflows` and read the CEO
   allocation through `aitradingoffice_hf`.
2. Read fund summary, cash, P&L, open equity positions, recent equity
   transactions, current MTF records, and the latest `daily_shared_screener` or
   `screener-conviction-workflow` run.
3. Call `tv_health_check` and require the pane's runtime-assigned `target_id`.
   Before a candidate deep check, set its symbol and timeframe on that target.
   Pass the same `target_id` to every stateful TradingView call that accepts it.
   If no target is assigned, stop rather than read another pane's chart. Use
   `news_fetch` and `browser` only for
   decision-relevant news, filings, or fundamentals.
4. The manifest trigger is schedule metadata, not a live cron. A manual
   invocation executes one tick. Call `watch_schedule` only when the user
   explicitly asks for repetition; it creates an in-process interval that
   fires only while the current scheduler/runtime remains alive. Its prompt
   must begin exactly with `/skill:hedge-fund-long-mtf-desk`.

## Tick

1. Establish the India-market timestamp, source freshness, and whether current
   or previous-close data is being used. An ambiguous boundary blocks new
   entries but not risk-reducing exits.
2. Compute available desk capacity as the minimum of CEO allocation, manifest
   `max_notional`, fund cash, and the remaining risk budget after existing MTF
   exposure.
3. Start with `mtf_candidates` from the shared screener. Use a high-level
   `scan` only to verify, rank, or fill a documented gap when the shared output
   is unavailable or stale.
4. For each candidate, verify with safe TradingView reads and sourced research:
   - a daily uptrend above key moving averages or a fresh base breakout;
   - liquidity and quote quality suitable for the intended fund size;
   - no clearly deteriorating fundamental fact;
   - no news or scheduled event that invalidates the intended holding period;
   - an explicit stop, review condition, and 3-20-session horizon.
5. Call `aitradingoffice_hf` with `action: list_clients`. Select a concrete
   client whose status is active and whose allowed instruments and constraints
   permit this CNC equity trade. If none qualifies or the choice is ambiguous,
   stop. Call `action: check_tradeable` with that client's id, then reconcile
   the proposal with current positions, concentration, CEO allocation,
   manifest notional, fund cash, and risk at stop. A failed or unreadable gate
   means skip and report.
6. Size the first 30-50% tranche with `strategy` using `flavor: hedgefund`,
   `instrument: equity`, `product_type: CNC`, the decided stop and scale-out
   targets, tranche risk, and a stable run/symbol/tranche idempotency key. Omit
   entry price so the pipeline obtains the live mark. Treat the notional as
   cash-funded CNC paper exposure, never as levered MTF buying power.
7. Review the risk card. If valid, send the returned `enter_trade` payload
   unchanged to `enter_trade`. Never construct a trade write through
   `aitradingoffice_hf` or supply a guessed fill. Call `aitradingoffice_hf`
   with `action: add_transaction_record`, `instrument: equity`, and the
   returned equity transaction id:

```yaml
desk: long_mtf
horizon: "3-20 trading days"
thesis:
entry:
initial_tranche_pct:
add_rules:
reduce_rules:
stop:
target_or_trailing_rule:
review_dates:
risk_at_stop:
fund_exposure_after_entry:
idempotency_key:
screener_record:
```

8. Add only on constructive follow-through, a successful retest, or documented
   thesis confirmation. Re-read cash and exposure, move the stop only when
   market structure warrants it, and rerun `strategy` for residual risk with a
   new tranche key. Use only that result's exact `enter_trade` payload.
9. On every review, compare live OHLCV, structure, volume, sourced news/events,
   CEO allocation, and transaction record. Trim or exit on a structure break,
   confirmed distribution, contradictory news, reduced allocation, stop,
   target, review-date invalidation, or maximum holding period.
10. For each reduction or exit, call `exit_trade` with `dry_run: true` first.
    Only when the candle verdict is unambiguous and quantity matches the office
    position may it be repeated with `dry_run: false`. Update the transaction
    record with evidence and remaining thesis/risk.
11. Send `hedgefund_report` with exposure, entries, exits, watchlist, screener
    hits or misses, data gaps, and one screening improvement.

## Guardrails

- Do not pyramid unless the original position is profitable, the revised stop
  reduces total risk, and CEO allocation still permits the add.
- Do not deploy the full allocation in one tranche unless the record explicitly
  explains why staged execution adds no risk benefit.
- Do not enter ahead of material event risk unless the record states the event,
  evidence, maximum loss, and why the risk is acceptable.
- Never convert a failed MTF trade into an investment or weaken its stop to
  avoid settlement.
- Do not report paper cash, exposure, or P&L as broker MTF financing, leverage,
  collateral, interest, or margin accounting.
- Each successful `enter_trade` tranche counts toward `max_trades_per_tick`.
- Every entry must be sized by `strategy` and booked by `enter_trade`; every
  settlement must use `exit_trade`.
