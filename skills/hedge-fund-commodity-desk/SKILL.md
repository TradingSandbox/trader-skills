---
name: "hedge-fund-commodity-desk"
description: "Run the Hedge Fund Commodity desk as a daily MCX/F&O commodity paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital for commodity futures or options ideas, including crude, gold, silver, natural gas, or other supported contracts; create, monitor, exit, and journal fund-book futures or option paper transactions."
mandate:
  kind: "routine"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_fno_mcx_contracts_search_tool, groww_fetch_historical_candle_data, groww_get_historical_technical_indicators, groww_get_quotes_and_depth, groww_get_ltp, groww_get_open_interest_analysis, groww_get_greeks_for_fno_contract, groww_calculate_fno_margin, news_fetch, browser, tv_symbol_search, tv_quote_get, tv_data_get_ohlcv, aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
  allowed_ledgers: [futures, options]
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

Trade commodity contracts in the paper fund book only when contract identity,
session, liquidity, margin, and macro/event context are clear.

## Tick

1. Rehydrate workflow state, CEO allocation, commodity-related positions, and
   recent futures/options records.
2. Call `groww_resolve_market_time_and_calendar` and verify the relevant
   commodity session rather than assuming equity-market hours.
3. Use `groww_fno_mcx_contracts_search_tool` to resolve the exact contract.
4. Build a setup from trend, volatility, OI, quote/depth, and relevant macro or
   inventory/news context.
5. Reject trades with unresolved contract month, poor depth, major event risk,
   or margin above allocation.
6. Use `groww_calculate_fno_margin` before futures or short-option exposure.
7. Enter in parts. The first paper order should normally be 30-50% of intended
   futures size, premium, or margin. Add only after contract price confirms the
   thesis and liquidity remains healthy. Reduce or exit when price rejects the
   thesis, spread widens, margin risk rises, or a macro/event risk changes.
8. Create paper futures transactions for futures ideas and paper option
   transactions for option ideas.
9. Add a transaction record with commodity, contract, expiry, thesis, initial
   tranche percent, add rules, reduce rules, margin, stop, target, time/session
   exit, and macro risk.
10. Keep watching contract price, spread, margin, OI, and event risk. Add,
   reduce, or exit through the matching AITradingOffice action based on the
   transaction record.
11. Report exposures, P&L, rejected ideas, and contract/session lessons.

## Guardrails

- Never trade a commodity symbol until the exact exchange contract is resolved.
- Do not deploy the full allocation in one order unless the transaction record
  explicitly justifies it.
- Treat overnight commodity risk as separate from equity-market risk.
- Do not use stock/index option logic for commodities without checking contract
  behavior and liquidity.
