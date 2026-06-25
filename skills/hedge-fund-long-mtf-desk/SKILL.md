---
name: "hedge-fund-long-mtf-desk"
description: "Run the Hedge Fund Long MTF desk as a daily positional long-equity paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow MTF candidates, validate Indian stocks for multi-day to multi-week long MTF-style trades, size them from fund cash and risk limits, create and monitor fund-book equity paper transactions, and journal thesis improvements."
mandate:
  kind: "routine"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_curate_symbols, groww_fetch_technical_screener, groww_fetch_fundamentals_screener, groww_fetch_stocks_fundamental_data, groww_fetch_historical_candle_data, groww_get_historical_technical_indicators, groww_get_quotes_and_depth, groww_get_ltp, groww_calculate_equity_margin, news_fetch, browser, stock_trend_snapshot, tv_symbol_search, tv_quote_get, tv_data_get_ohlcv, aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
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

Build and manage multi-day long equity exposure in the paper fund book. The desk
uses technical strength plus fundamental/news sanity checks, and it exits when
the thesis or risk budget breaks.

## Tick

1. Rehydrate workflow state and CEO allocation.
2. Read fund summary, cash, open equity positions, current MTF desk records,
   latest `daily_shared_screener` or `screener-conviction-workflow` run, and
   recent fund P&L from AITradingOffice.
3. Call `groww_resolve_market_time_and_calendar`.
4. Start with `mtf_candidates` from the shared screener output. Use local
   technical/fundamental screeners only to verify, rank, or fill a gap when the
   shared screener is stale or unavailable.
5. For each candidate, verify:
   - daily trend above key moving averages or a fresh base breakout;
   - liquidity and spread fit fund size;
   - fundamentals are not obviously deteriorating;
   - news does not contradict the long thesis;
   - stop and review condition are explicit.
6. Size each paper trade by the smaller of CEO allocation, manifest notional,
   fund cash, and risk at stop. Use `groww_calculate_equity_margin` when the
   trade is MTF-style.
7. Enter in parts. The first paper order should normally be 30-50% of intended
   size. Add only on constructive follow-through, successful retest, or thesis
   confirmation, and only if total risk at the revised stop remains within the
   original budget. Trim or exit if price breaks structure, volume confirms
   distribution, news contradicts the thesis, or CEO allocation is reduced.
8. Create paper equity transactions only after `check_tradeable` succeeds.
9. Add a transaction record:

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
screener_record:
```

10. On each tick, update, add, trim, or exit positions based on price movement
   and the transaction record. Exit positions whose stop, thesis break, event
   risk, target, or max holding period has arrived.
11. Report to the office with open exposure, new trades, exits, watchlist,
   screener hit/miss notes, and what should improve in screening.

## Guardrails

- Do not pyramid unless the original trade is profitable, stop has moved to
  reduce total risk, and CEO allocation still allows it.
- Do not deploy the full allocation in one order unless the transaction record
  explicitly justifies it.
- Do not enter before major event risk unless the record states why the event is
  acceptable.
- Never convert a failed MTF trade into an investment.
