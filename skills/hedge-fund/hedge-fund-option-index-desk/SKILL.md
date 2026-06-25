---
name: "hedge-fund-option-index-desk"
description: "Run the Hedge Fund Index Options desk as a daily paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow index bias and trade NIFTY, BANKNIFTY, FINNIFTY, or other index option trades, including directional option buying, spreads, and tightly risk-capped intraday option ideas; create, monitor, exit, and journal fund-book option paper transactions."
mandate:
  kind: "routine"
  persona: "hedgefund"
  risk_profile: "aggressive"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_curate_symbols, groww_fetch_curated_fno, groww_get_open_interest_analysis, groww_get_greeks_for_fno_contract, groww_get_greeks_for_fno_symbol, groww_get_atm_straddle_chart, groww_get_payoff_chart_steps, groww_get_quotes_and_depth, groww_get_ltp, groww_calculate_fno_margin, news_fetch, stock_trend_snapshot, tv_symbol_search, tv_quote_get, tv_data_get_ohlcv, aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
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

Express index views with defined or tightly capped risk in the paper fund book.
The desk must understand direction, volatility, expiry, greeks, and event risk
before opening any option transaction.

## Tick

1. Rehydrate workflow state, CEO allocation, option positions, option
   transactions, fund P&L, and the shared `screener-conviction-workflow`
   `index_bias` when available.
2. Call `groww_resolve_market_time_and_calendar`; record expiry and time to
   close.
3. Pick the trade mode:
   - `directional_buy`: strong trend/breakout with premium risk capped;
   - `spread`: directional view with theta control;
   - `no_trade`: unclear direction, poor liquidity, or event risk.
4. Use OI, greeks, straddle chart, quotes/depth, index trend, and shared
   screener `index_bias` to validate the structure.
5. Reject trades when:
   - liquidity is poor or spreads are wide;
   - theta decay dominates the expected move;
   - stop cannot be expressed as premium, underlying level, or time;
   - margin/premium exceeds allocation.
6. Check tradeability and F&O margin/premium.
7. Enter in parts unless the structure's maximum loss is already fully defined
   and small. The first paper order should normally be 30-50% of intended
   premium or margin. Add only after the underlying confirms direction or
   volatility behaves as expected. Reduce or exit quickly when delta moves
   against the thesis, theta decay accelerates, IV collapses, spread widens, or
   the underlying fails the planned level.
8. Create at most two paper option transactions through
   `aitradingoffice_hf:create_option_transaction`.
9. Add transaction records with underlying, expiry, strike, option type,
   premium, quantity, initial tranche percent, add rules, reduce rules, greeks,
   stop, target, time exit, max loss, and screener bias reference.
10. Keep watching underlying price, premium, greeks, spread, and time decay.
   Add, reduce, or exit through `exit_option_transaction` when stop, target,
   time decay, volatility crush, failed add condition, or session cutoff
   invalidates the trade.
11. Report action, exposure, rejected structures, and lesson learned.

## Guardrails

- Prefer defined-risk structures; naked short option exposure requires explicit
  CEO allocation and margin confirmation.
- Do not deploy the full allocation in one order unless the transaction record
  explicitly justifies it.
- Do not hold near-expiry long options without a stated reason.
- Do not trade index options if market direction and volatility view conflict.
