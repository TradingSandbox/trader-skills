---
name: "hedge-fund-long-mis-desk"
description: "Run the Hedge Fund Long MIS desk as a daily intraday long-equity paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow long candidates, validate liquid Indian stocks for same-day long momentum, breakout, pullback, or VWAP reclaim trades; create, monitor, exit, and journal fund-book equity paper transactions; and report results back to the hedge-fund office."
mandate:
  kind: "routine"
  persona: "hedgefund"
  risk_profile: "aggressive"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_curate_symbols, groww_fetch_technical_screener, groww_fetch_market_movers_and_trending_stocks_funds, groww_fetch_historical_candle_data, groww_get_historical_technical_indicators, groww_get_historical_candlestick_patterns, groww_get_quotes_and_depth, groww_get_ltp, groww_calculate_equity_margin, news_fetch, stock_trend_snapshot, tv_symbol_search, tv_quote_get, tv_data_get_ohlcv, aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
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

Trade only high-quality intraday long equity setups for the hedge fund paper
book. The desk exists to capture same-day upside momentum without carrying risk
overnight.

## Setup

1. Read the workflow run with `aitradingoffice_workflows` when available.
2. Read CEO allocation records, fund summary, open equity positions, today's
   equity transactions, and transaction records from `aitradingoffice_hf`.
3. Read the latest `daily_shared_screener` record or
   `screener-conviction-workflow` run when available.
4. Schedule the routine tick with `watch_schedule`.

## Tick

1. Call `groww_resolve_market_time_and_calendar`. If the cash market is closed,
   manage open records only and report no new entries.
2. Confirm available desk budget from the CEO record. Effective notional is the
   lower of CEO allocation, manifest `max_notional`, and available fund cash.
3. Start with `long_mis_candidates` from the shared
   `screener-conviction-workflow` output. Add a local market mover or technical
   screener candidate only when it is newer than the shared screener and the
   record explains why it was not present upstream.
4. Require all entry candidates to pass:
   - price above VWAP or reclaiming VWAP with volume support;
   - clean intraday higher-high/higher-low or breakout structure;
   - positive relative volume and acceptable spread/depth;
   - no immediate negative news shock;
   - stop level close enough that loss at stop fits desk risk.
5. For each approved candidate, call `aitradingoffice_hf` with
   `action: check_tradeable`, then `groww_calculate_equity_margin`.
6. Enter in parts. The first paper order should normally be 30-50% of intended
   size. Add only if price confirms the thesis, volume improves, and the
   combined risk remains inside the original stop-risk budget. Reduce or cut
   immediately if price loses VWAP, momentum fails, spread widens, or the stop
   is threatened. Use the full size at once only when the record explains why
   staged execution is not useful.
7. Create at most three paper equity transactions using
   `aitradingoffice_hf` with `action: create_equity_transaction`.
8. Immediately add a transaction record containing:

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
ceo_allocation_record:
screener_record:
```

9. Keep watching live price movement after entry. Add, trim, or exit based on
   the transaction record rather than waiting passively. Exit through
   `exit_equity_transaction` when stop, target, reversal, failed add condition,
   or time exit is hit.
10. Send a `hedgefund_report` with trades opened, exits, open risk, rejected
   candidates, whether the shared screener helped or missed the setup, and one
   improvement note.

## Guardrails

- Do not carry MIS positions overnight. If still open after 15:10, exit or write
  an urgent record explaining why the exit tool failed.
- Do not average down. One add is allowed only if it reduces average risk and
  remains inside allocation.
- Do not deploy the full allocation in one order unless the transaction record
  explicitly justifies it.
- Do not trade illiquid small caps, news-halting names, or symbols with unclear
  identity.
- Every trade must have a stop before creation.
