---
name: "hedge-fund-stock-futures-desk"
description: "Run the Hedge Fund Stock Futures desk as a daily paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow stock futures candidates and trade Indian single-stock futures, including long or short futures based on trend, basis, liquidity, OI, and notional exposure; create, monitor, exit, and journal fund-book futures paper transactions without broker margin checks."
mandate:
  kind: "routine"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_curate_symbols, groww_fetch_curated_fno, groww_fetch_technical_screener, groww_fetch_historical_candle_data, groww_get_historical_technical_indicators, groww_get_open_interest_analysis, groww_get_quotes_and_depth, groww_get_ltp, news_fetch, stock_trend_snapshot, tv_symbol_search, tv_quote_get, tv_data_get_ohlcv, aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
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

Use single-stock futures for liquid, high-conviction directional exposure where
futures are cleaner than equity or options.

## Tick

1. Rehydrate workflow state, CEO allocation, stock futures positions, and
   recent futures transaction records, plus the latest `daily_shared_screener`
   or `screener-conviction-workflow` run.
2. Call `groww_resolve_market_time_and_calendar`.
3. Start with `stock_future_candidates` from the shared screener output. Use F&O
   movers, trend screeners, OI changes, and CEO focus names to validate or
   update the shared list.
4. Require:
   - clear long or short structure on the underlying;
   - F&O liquidity and quote/depth support;
   - OI/price behavior consistent with the thesis;
   - notional exposure inside allocation;
   - stop and target expressed in underlying/futures price.
5. Check tradeability, CEO allocation, manifest notional limit, available fund
   cash, and max-loss budget. Do not call broker margin tools.
6. Enter in parts. The first paper order should normally be 30-50% of intended
   futures size. Add only when the underlying confirms direction, OI/volume
   supports the move, and total risk remains inside budget. Reduce or exit when
   price fails, OI contradicts the thesis, fund exposure becomes too large, or
   CEO allocation is reduced.
7. Create at most two paper futures transactions through AITradingOffice.
8. Attach a record with direction, contract, notional exposure, initial tranche
   percent, add rules, reduce rules, stop, target, roll/expiry plan, risk at
   stop, and screener record id.
9. Keep watching price movement, OI, spread, and fund exposure. Add, reduce, or exit
   through `exit_future_transaction` when invalidated, target hit, failed add
   condition, exposure risk rises, or CEO reduces futures allocation.
10. Report positions, exposure, rejected trades, and one rule improvement.

## Guardrails

- Do not use futures when the same idea can be expressed with materially lower
  risk in cash equity.
- Do not deploy the full allocation in one order unless the transaction record
  explicitly justifies it.
- Do not hold through known event risk without explicit record justification.
- Do not allow one stock future to dominate total fund risk.
