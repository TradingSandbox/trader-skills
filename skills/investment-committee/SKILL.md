---
name: "investment-committee"
description: "Evaluate one Indian-listed equity through independent market, fundamentals, news, bull, bear, trading, and risk stages, then write a sourced committee verdict. Use for a decision-quality research memo; this workflow never places a trade."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  roles: [ceo, investor]
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, browser, hedgefund_delegate, hedgefund_report, news_fetch, tv_health_check, tv_symbol_search, tv_chart_set_symbol, tv_chart_set_timeframe, tv_symbol_info, tv_quote_get, tv_depth_get, tv_technicals_get, tv_fundamentals_get, tv_data_get_ohlcv, tv_news_list, tv_news_read]
  allowed_ledgers: []
  limits:
    max_trades_per_tick: 0
    max_notional: 0
    paper_only: true
  trigger: null
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Investment Committee

## Scope

Produce one evidence-linked committee memo for one Indian-listed equity. This
is research only: do not call `strategy`, `enter_trade`, `exit_trade`, or any
direct transaction write. A later trading desk must independently revalidate
and size any accepted idea.

## Intake and identity

Ask only for missing essentials: company/symbol, decision horizon, objective,
and any as-of date. Resolve the exact exchange-qualified symbol with
`tv_symbol_search`/`tv_symbol_info`; reject ambiguous identities. Check
`tv_health_check` and record source timestamps. Current TradingView
fundamentals and news are not point-in-time historical datasets: for an old
as-of date, use a genuinely contemporaneous source or mark that stage
unavailable. Never leak current facts into a historical verdict.

## Independent stages

Run each stage from the raw evidence, not from another stage's conclusion:

1. **Market/technical:** quote, depth when available, multi-horizon OHLCV,
   trend/volatility/levels, liquidity, and regime.
2. **Fundamentals:** business profile, valuation, growth, profitability,
   balance sheet, cash generation, and material missing fields from
   `tv_fundamentals_get` plus primary-company sources when browsed.
3. **News/events:** current and decision-relevant filings/news from
   `tv_news_list`/`tv_news_read`, `news_fetch`, and `browser`; separate event
   fact, source time, and inferred impact.
4. **Bull case:** strongest falsifiable thesis, catalysts, evidence, horizon,
   and disconfirming conditions.
5. **Bear case:** strongest independent downside thesis, accounting/business/
   valuation/event risks, and disconfirming conditions.
6. **Trader view:** entry conditions, invalidation, liquidity, and timing as
   analysis only—no size or booking.
7. **Risk chair:** identify concentration, gap/event risk, evidence conflicts,
   data staleness, and conditions that make the idea untradeable.

In a CEO pane, `aitradingoffice_hf {action: list_employees}` is the roster
source of truth. The CEO may delegate genuinely independent stages with
`hedgefund_delegate`; each task must be self-contained and end with the exact
skill command only when invoking another skill. Delegation failure is visible
and must fall back to a local stage or an explicit gap—never a fabricated
report. In an Investor pane, run stages locally and send the finished memo with
`hedgefund_report`.

## Synthesis

Reconcile conflicts explicitly. Give a verdict of `support`, `watch`, `reject`,
or `insufficient evidence`, with confidence, time horizon, top evidence,
strongest opposing evidence, missing data, invalidation conditions, and next
research step. Do not average stage opinions into a vote.

When acting as an office employee, create or update an AITradingOffice research
record through `aitradingoffice_hf` and, if a workflow run exists, update only
that run's narrative/status through `aitradingoffice_workflows`. The record
must include source names and timestamps, symbol identity, raw stage findings,
the synthesis, and every unresolved gap. Workflow state does not execute this
skill by itself.
