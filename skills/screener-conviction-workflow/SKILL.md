---
name: "screener-conviction-workflow"
description: "Run a reproducible Indian-market screen, validate candidates with TradingView evidence, and convert the survivors into desk-specific research cards for cash, MIS, MTF-style, futures, options, or index analysis. This workflow creates candidates but never places trades."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  roles: [ceo, investor, trader]
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [scan, option_scan, strategy, aitradingoffice_hf, aitradingoffice_workflows, news_fetch, browser, tv_health_check, tv_screener_fields, tv_screener_ops, tv_screener_query, tv_symbol_search, tv_symbol_info, tv_chart_set_symbol, tv_chart_set_timeframe, tv_chart_manage_indicator, tv_quote_get, tv_depth_get, tv_technicals_get, tv_fundamentals_get, tv_data_get_ohlcv, tv_data_get_study_values, tv_data_get_indicator, tv_data_get_volume_profile, tv_capture_screenshot, tv_options_expirations, tv_options_chain, tv_futures_curve, tv_news_list, tv_news_read]
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

# Screener Conviction Workflow

## Boundary

Create an auditable candidate set, not a trade. `strategy` may be used only to
show deterministic risk math for a candidate; never pass its payload to
`enter_trade` from this skill. Do not claim broker margin, borrow availability,
option open interest, or IV rank: the current TradingView surface does not
provide those fields reliably. Chain volume may be used only when the current
response returns it, with its timestamp and liquidity caveats.

## Intake

Resolve:

- market/universe and maximum result count;
- named scan or inline thesis;
- horizon and direction;
- intended desk kind: CNC research, long/short MIS, MTF-style positional,
  stock/index future, long single-leg option, or analysis-only structure;
- hard exclusions and freshness requirement.

Use `tv_screener_fields` and `tv_screener_ops` before composing a raw
`tv_screener_query`. Use `query.limit` for bounds; do not invent a raw row
range. Prefer the high-level `scan` tool for saved/named or inline reproducible
queries. Record the actual normalized query, not only its display name.

## Candidate workflow

1. Check TradingView health and source time. Run the bounded screen. Preserve
   rejected rows and reasons; never silently replace an empty result with
   remembered symbols.
2. Resolve every survivor to an exchange-qualified symbol. Remove duplicates,
   ambiguous identities, stale quotes, illiquid/invalid rows, and candidates
   outside the requested market.
3. Validate survivors with quote/depth when available, OHLCV, technicals,
   fundamentals when relevant, volume/participation evidence, and current
   news. A chart screenshot is supporting evidence only, never the source of a
   numeric claim.
4. Rank on explicit, reproducible fields. Separate data facts from model
   interpretation and show missing inputs.
5. Adapt cards by desk:
   - MIS: session-valid trend/trigger/invalidation and same-day exit condition;
   - MTF-style: positional cash-equity thesis; no leverage/margin claim;
   - futures: resolve dated contract/expiry through `tv_futures_curve` and
     symbol search; keep margin and OI conclusions unavailable;
   - options: call `option_scan` or the TradingView chain for exact expiry,
     strike, call/put, delta, IV percentage, premium, and spread. Executable
     follow-up is limited to a later desk's single-leg long call/put. Short or
     multi-leg structures remain analysis-only because atomic spread booking
     and required risk fields are unavailable;
   - index: use an exchange-qualified index/underlying and label it analysis
     unless a later desk resolves a supported contract.
6. `strategy` may calculate a hypothetical stop-risk card for comparison, but
   its returned entry payload must not be fired by this workflow.

## Office output

Store or update a research record through `aitradingoffice_hf` when an office
context exists. Include normalized query, timestamps, universe, ranked
candidates, desk routing, evidence, rejection reasons, unsupported-data gaps,
and links to parent idea/workflow identity. Update
`aitradingoffice_workflows` only for an existing run; a workflow row stores
state and does not execute this skill.

Return concise buckets such as `long_mis_candidates`,
`short_mis_candidates`, `long_mtf_style_candidates`,
`stock_future_candidates`, `long_option_candidates`, and
`analysis_only_candidates`. Each card needs symbol/contract identity,
direction, trigger, invalidation, freshness, liquidity evidence, event risk,
and next owner. Empty buckets are valid and preferable to weak substitutions.
