---
name: "hedge-fund-investor-desk"
description: "Run one long-horizon Hedge Fund Investor research tick: inspect the office book and watchlist, source current technical/fundamental/news evidence, create or update employee-owned theses, review open positions, and report findings without placing trades."
mandate:
  kind: "routine"
  persona: "hedgefund"
  roles: [investor]
  risk_profile: "conservative"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, scan, news_fetch, browser, stock_trend_snapshot, tv_health_check, tv_symbol_search, tv_chart_set_symbol, tv_chart_set_timeframe, tv_quote_get, tv_technicals_get, tv_fundamentals_get, tv_data_get_ohlcv, tv_news_list, tv_news_read, hedgefund_report, watch_schedule]
  allowed_ledgers: []
  limits:
    max_trades_per_tick: 0
    max_notional: 0
    paper_only: true
  trigger: { kind: cron, expr: "30 9 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Investor Desk

## Boundary and scheduling

Research and reporting only. Do not call `strategy`, `enter_trade`,
`exit_trade`, or direct transaction mutations. One invocation is one tick.
The trigger above is policy metadata, not an installed schedule. Use
`watch_schedule` only on an explicit request; it is interval-based and remains
alive only while the scheduler owns the pane. A workflow row stores state but
does not execute this skill.

## Tick

1. Read the assigned workflow/CEO records when present. Through
   `aitradingoffice_hf`, inspect the selected book, fund summary/P&L,
   book-wide marked positions, this employee's research records, and relevant
   transaction records. Treat AITradingOffice as authoritative.
2. Check TradingView health and source timestamps. Resolve every company to an
   exchange-qualified symbol. Stale, ambiguous, or conflicting identity/data
   is a recorded gap, not permission to guess.
3. For new candidates, use a bounded high-level `scan` or the assigned shared
   screener record. Validate business quality, balance sheet/cash generation,
   valuation, catalysts, risks, multi-horizon price/volume trend, and current
   news with TradingView reads, `stock_trend_snapshot`, `news_fetch`, and
   primary browsed sources where useful.
4. For existing holdings, compare the live marked position and original
   thesis with current fundamentals, trend, events, and invalidation
   conditions. Recommend `hold`, `watch`, `reduce review`, or `exit review` as
   research; the trading owner must independently decide and execute.
5. Create a new employee-owned thesis with `aitradingoffice_hf {action:
   create_record}` or update the exact existing record with `update_record`.
   Include symbol identity, source timestamps, horizon, thesis, evidence,
   valuation context, catalyst path, key risks, invalidation, current position
   link, confidence, and missing data.
6. Update an existing workflow run with truthful progress/status and send
   `hedgefund_report` summarizing new/changed theses, position reviews,
   rejected ideas, source gaps, and requests for CEO/trader follow-up.

Current TradingView fundamentals/news are not guaranteed point-in-time. For a
historical as-of request, use a genuinely contemporaneous source or label the
field unavailable; never contaminate the old decision with current facts.
