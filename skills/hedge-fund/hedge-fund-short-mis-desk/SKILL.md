---
name: "hedge-fund-short-mis-desk"
description: "Run the Hedge Fund Short MIS desk as a daily intraday short-equity paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow bearish candidates, validate liquid Indian stocks for same-day short breakdown, failed breakout, VWAP rejection, or weak-sector trades; create, monitor, exit, and journal fund-book equity paper transactions; and report results back to the hedge-fund office."
mandate:
  kind: "routine"
  persona: "hedgefund"
  risk_profile: "aggressive"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_curate_symbols, groww_fetch_technical_screener, groww_fetch_market_movers_and_trending_stocks_funds, groww_fetch_historical_candle_data, groww_get_historical_technical_indicators, groww_get_quotes_and_depth, groww_get_ltp, news_fetch, stock_trend_snapshot, tv_symbol_search, tv_quote_get, tv_data_get_ohlcv, aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
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

Trade only same-day bearish equity setups in liquid names. The desk protects the
fund when breadth is weak and exploits intraday downside without overnight risk.

## Setup

Read the CEO allocation, fund state, open equity positions, the latest
`daily_shared_screener` or `screener-conviction-workflow` run, and recent short
MIS records from AITradingOffice. Schedule the routine tick with
`watch_schedule`.

## Tick

1. Call `groww_resolve_market_time_and_calendar`.
2. If short selling is not suitable for the session, report `no_short_edge` and
   do not force trades.
3. Start with `short_mis_candidates` from the shared
   `screener-conviction-workflow` output. Add weak-sector movers, failed
   breakouts, or stocks below VWAP only as fresh intraday fallbacks and label
   them `post_screener_candidate`.
4. Require:
   - lower-high/lower-low structure or confirmed range breakdown;
   - price below VWAP or rejected from VWAP/resistance;
   - weak relative strength versus NIFTY/sector;
   - acceptable spread, depth, borrow/intraday feasibility when exposed;
   - stop above the breakdown/rejection level.
5. Check tradeability, CEO allocation, manifest notional limit, available fund
   cash, and risk-at-stop budget before any transaction.
6. Enter in parts. The first paper order should normally be 30-50% of intended
   short size. Add only after downside follow-through, failed reclaim, or fresh
   weak breadth confirms the trade. Reduce or cover quickly if price reclaims
   VWAP, the breakdown fails, spread widens, or the short thesis weakens.
7. Create at most three paper equity transactions through
   `aitradingoffice_hf:create_equity_transaction`.
8. Add a transaction record containing thesis, initial tranche percent,
   add rules, reduce rules, stop, target, time exit, max-loss amount, CEO
   allocation id, and screener record id.
9. Keep watching live price movement after entry. Add, trim, or cover based on
   the record. Exit through `exit_equity_transaction` on stop, target, sharp
   breadth reversal, failed add condition, or 15:10 square-off.
10. Report opened trades, exits, rejected shorts, risk remaining, screener
   usefulness or misses, and learning notes through `hedgefund_report`.

## Guardrails

- Do not short against strong market-wide breadth without a stock-specific
  catalyst or failed-breakout structure.
- Never average up into a losing short.
- Do not deploy the full allocation in one order unless the transaction record
  explicitly justifies it.
- Do not hold overnight.
- If data cannot confirm short feasibility, skip the trade.
