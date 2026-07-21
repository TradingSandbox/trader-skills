---
name: "hedge-fund-option-stock-desk"
description: "Run the Hedge Fund Stock Options desk as a daily paper-trading routine in AITradingOffice. Use when the CEO or user assigns capital to consume screener-conviction-workflow stock-option candidates; evaluate event-aware directional or spread ideas on Indian F&O stocks; execute only single-leg long calls or puts resolved by option_scan; size each tranche through strategy; enter and exit only through the deterministic trade pipeline; journal the fund-book transactions; and report results."
mandate:
  kind: "routine"
  persona: "hedgefund"
  roles: [trader]
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, scan, option_scan, strategy, enter_trade, exit_trade, news_fetch, browser, tv_health_check, tv_symbol_search, tv_symbol_info, tv_chart_set_symbol, tv_chart_set_timeframe, tv_quote_get, tv_depth_get, tv_technicals_get, tv_fundamentals_get, tv_data_get_ohlcv, tv_options_expirations, tv_options_chain, tv_news_list, tv_news_read, hedgefund_report, watch_schedule]
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

Trade a single-stock option only when the underlying setup, catalyst, liquidity,
and option-chain evidence agree. Executable ideas are single-leg long calls or
puts with premium risk capped at the debit paid. Multi-leg structures are
analysis-only because the pipeline cannot book them atomically. The only
trade-ledger path is `strategy` to `enter_trade` to `exit_trade`.

## Setup

1. Rehydrate workflow state with `aitradingoffice_workflows` when present.
2. Through `aitradingoffice_hf`, read CEO allocation, fund cash and P&L, open
   option positions, recent option transactions and records, stock-specific
   research records, and the latest `daily_shared_screener` or
   `screener-conviction-workflow` run.
3. Call `tv_health_check` and require the pane's runtime-assigned `target_id`.
   Before a deep check, set the underlying symbol and timeframe on that target.
   Pass the same `target_id` to every stateful TradingView call that accepts it.
   If no target is assigned, stop rather than read another pane's chart. Use
   `news_fetch` and `browser` for
   sourced event, filing, or fundamental checks.
4. The manifest trigger is schedule metadata, not a live cron. A manual
   invocation executes one tick. Call `watch_schedule` only when the user
   explicitly asks for repetition; it creates an in-process interval that
   fires only while the current scheduler/runtime remains alive.

## Tick

1. Establish the current India-market timestamp, expiry context, session
   cutoff, and data freshness. If session, event date, or contract expiry is
   ambiguous, open no new option.
2. Compute available premium and maximum-loss capacity as the minimum of CEO
   allocation, manifest `max_notional`, fund cash, and remaining daily-risk
   budget after open stock-option exposure.
3. Start with `stock_option_candidates` from the shared screener. If that output
   is stale or absent, run a high-level `scan` on eligible liquid underlyings.
   Add a fresh CEO/investor watch name or news catalyst only with a record that
   reconciles it against the shared list.
4. Validate the underlying trend or event thesis with safe TradingView reads
   and sourced news/fundamental evidence. Keep facts distinct from inference.
5. Run `option_scan` for the chosen underlying. Require an exact supported
   contract symbol, expiry, strike, call/put type, acceptable bid/ask spread,
   useful delta/IV fields, and a premium within the risk budget. `option_scan`
   does not establish the current exchange lot size: verify that value against
   a current authoritative exchange or broker source with `browser`, record the
   source and as-of date, and stop if it cannot be verified. Use only fields
   actually returned by `option_scan`; do not infer missing chain metrics.
6. Select one mode:
   - `directional_buy` for an executable single-leg long call or put;
   - `spread_analysis_only` when the thesis requires multiple legs;
   - `no_trade` when liquidity, event risk, expected move, or time decay makes
     the option inferior to cash equity or no position.
7. Call `aitradingoffice_hf` with `action: list_clients`. Select a concrete
   client whose status is active and whose allowed instruments and constraints
   permit this long stock option. If none qualifies or the choice is ambiguous,
   stop. Call `action: check_tradeable` with that client's id, then reconcile
   the long option with current exposure, fund cash, CEO allocation, manifest
   notional, premium at risk, and maximum loss. A failed or unreadable gate
   means skip and report.
8. Size the first 30-50% tranche with `strategy` using `flavor: hedgefund`,
   `instrument: option`, `side: buy`, exact `underlying`, `expiry`, `strike`,
   `option_type`, verified `lot_size` and `tv_symbol`, explicit stop and targets,
   tranche risk, and a stable run/symbol/contract/tranche idempotency key. Omit
   an entry-price override.
9. Review the risk card and pass the returned `enter_trade` payload unchanged
   to `enter_trade`. Only the selected single long contract may reach this
   step; never write a trade through `aitradingoffice_hf`. Call it only with
   `action: add_transaction_record`, `instrument: option`, and the returned
   option transaction id:

```yaml
desk: option_stock
underlying:
event_or_setup:
contract:
side: buy
delta_and_iv:
premium_exposure:
max_loss:
lot_size_source:
initial_tranche_pct:
add_rules:
reduce_rules:
stop:
target:
exit_before:
idempotency_key:
screener_record:
```

10. Add only after the underlying confirms the move, liquidity remains sound,
    and delta/IV evidence still fits the thesis. Re-read exposure and event
    state, rerun `strategy` for residual risk with a distinct tranche key, and
    use only that result's exact `enter_trade` payload.
11. Monitor underlying OHLCV, contract mark and spread, delta/IV evidence, event
    timing, time decay, and the persisted plan. Reduce or exit on underlying
    failure, stop, target, event change/passage, volatility crush, excessive
    theta, widening spread, or time exit. Call `exit_trade` with `dry_run: true`
    first; repeat with `dry_run: false` only after an unambiguous verdict and
    quantity reconciliation.
12. Update the transaction record with additions, reductions, exit evidence,
    and residual risk. Report entries, exits, rejected chains, exposure, data
    gaps, and one improvement with `hedgefund_report`.

## Guardrails

- Execute only single-leg long calls or puts. Multi-leg structures and all
  short-option legs are analysis-only in this skill.
- Do not trade a stock option with uncertain F&O eligibility, contract identity,
  lot size, expiry, or quote quality.
- Stop when the current lot size cannot be verified from an authoritative
  external source.
- Do not deploy the full allocation in one tranche unless the premium maximum
  loss is small, fully bounded, and explicitly justified in the record.
- Do not hold through results or material news unless the record states the
  event, sourced evidence, maximum loss, and exit plan.
- Prefer cash equity or no trade when it is cleaner and lower risk.
- Each successful `enter_trade` tranche counts toward `max_trades_per_tick`.
- Every entry must be sized by `strategy` and booked by `enter_trade`; every
  settlement must use `exit_trade`.
