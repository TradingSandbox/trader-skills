---
name: "hedge-fund-option-stock-desk"
description: "Run the Hedge Fund Stock Options desk as a daily paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow stock option candidates and trade single-stock options on Indian F&O names, including event-aware directional option buying or spreads; create, monitor, exit, and journal fund-book option paper transactions without broker margin checks."
mandate:
  kind: "routine"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_curate_symbols, groww_fetch_curated_fno, groww_fetch_stocks_fundamental_data, groww_fetch_historical_candle_data, groww_get_historical_technical_indicators, groww_get_open_interest_analysis, groww_get_greeks_for_fno_contract, groww_get_greeks_for_fno_symbol, groww_get_quotes_and_depth, groww_get_ltp, news_fetch, browser, stock_trend_snapshot, tv_symbol_search, tv_quote_get, tv_data_get_ohlcv, aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
  allowed_ledgers: [options]
  limits:
    max_trades_per_tick: 2
    max_notional: 200000
    paper_only: true
  trigger: { kind: cron, expr: "50 9 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Stock Options Desk

## Intent

Trade single-stock options only when the underlying setup, catalyst, liquidity,
and option chain agree. This desk avoids using options as a vague substitute for
equity entries.

## Tick

1. Rehydrate workflow state and CEO allocation.
2. Read open option positions, recent option transactions, stock-specific
   records, and the latest `daily_shared_screener` or
   `screener-conviction-workflow` run from AITradingOffice.
3. Call `groww_resolve_market_time_and_calendar`.
4. Start with `stock_option_candidates` from the shared screener output. Add F&O
   movers, CEO/investor watchlist names, trend snapshots, and news catalysts
   only when they improve or update the shared list.
5. Require:
   - underlying trend or event thesis;
   - option liquidity and acceptable bid/ask;
   - strike/expiry selected from greeks, OI, and expected move;
   - premium exposure and maximum loss inside allocation;
   - explicit stop and event-risk plan.
6. Prefer debit spreads or premium-defined buys unless CEO explicitly approved a
   short-premium structure.
7. Enter in parts unless the option structure's max loss is already fully
   defined and small. The first paper order should normally be 30-50% of
   intended premium exposure. Add only after the underlying confirms the move,
   liquidity remains acceptable, and greeks still fit the thesis. Reduce or exit
   when the underlying fails, IV crush appears, theta decay dominates, or event
   risk changes.
8. Create paper option transactions through AITradingOffice and attach a record:

```yaml
desk: option_stock
underlying:
event_or_setup:
contract:
greeks:
premium_exposure:
max_loss:
initial_tranche_pct:
add_rules:
reduce_rules:
stop:
target:
exit_before:
screener_record:
```

9. Keep watching underlying price, premium, greeks, spread, and event timing.
   Manage open trades for add conditions, stop, target, liquidity deterioration,
   event passage, IV crush, or time decay.
10. Report trades, rejected chains, exposure, and what to improve.

## Guardrails

- Do not trade options on stocks with unclear F&O eligibility or poor depth.
- Do not deploy the full allocation in one order unless the transaction record
  explicitly justifies it.
- Do not hold through results/news unless the record explicitly prices that
  event risk.
- Do not open a stock option trade when the equity trade would be cleaner and
  lower risk.
