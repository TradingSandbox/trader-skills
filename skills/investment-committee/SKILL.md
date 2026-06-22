---
name: "investment-committee"
description: "Evaluate one Indian-listed equity through a structured multi-perspective investment committee using TradeCLI's market, fundamentals, news, browser, TradingView, delegation, and AITradingOffice record tools. Use when the user wants a deeply researched Buy, Overweight, Hold, Underweight, or Sell view; a bull-versus-bear debate; an independent risk challenge; or a decision-ready trade proposal with entry, invalidation, sizing, and time horizon. This is an analysis-only one-shot workflow: it may recommend a paper trade but never places, modifies, or exits an order."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_curate_symbols, groww_fetch_stocks_fundamental_data, groww_fetch_historical_candle_data, groww_get_historical_technical_indicators, groww_get_quotes_and_depth, news_fetch, browser, stock_trend_snapshot, tv_symbol_search, tv_symbol_info, tv_quote_get, tv_data_get_ohlcv, hedgefund_employee_list, hedgefund_delegate, aitradingoffice_hf, aitradingoffice_workflows]
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

## Intent

Produce a decision-ready equity recommendation by separating evidence gathering,
adversarial thesis review, transaction design, and portfolio-risk judgment.
Preserve disagreements instead of averaging them into bland consensus.

This workflow adapts the role structure of Tauric Research's open-source
[TradingAgents](https://github.com/TauricResearch/TradingAgents) project to
TradeCLI's native tools and office model. It is an independent workflow, not a
copy of TradingAgents' Python/LangGraph runtime.

## Scope

- Analyze exactly one Indian-listed equity per invocation.
- Support current analysis or an explicitly supplied historical as-of date.
- Produce a five-tier final rating: `Buy`, `Overweight`, `Hold`,
  `Underweight`, or `Sell`.
- Optionally use independent hedge-fund employee panes for research lanes when
  suitable employees are already available.
- Save the completed committee memo as an AITradingOffice research record when
  the invocation is attached to an office run or employee.

Do not use this skill for portfolio optimization, derivatives structuring,
backtesting, unattended monitoring, or order execution.

## Hard limits

- Never place, modify, cancel, or exit a trade. `allowed_ledgers` is empty.
- Do not turn `Buy` or `Sell` into an executed transaction.
- Do not analyze more than one symbol in a run; ask the user to invoke the skill
  separately for comparisons.
- Do not invent social-media sentiment. If no direct social source is available,
  label the sentiment lane `news-and-attention proxy`.
- Do not claim a price, indicator, filing value, date, or market status unless a
  tool returned it.
- Do not present this committee output as a backtest or a proven source of alpha.

## Intake

Resolve the following without unnecessary questions:

```yaml
symbol: required
exchange: NSE | BSE
as_of: current | YYYY-MM-DD
time_horizon: 1-4 weeks | 1-3 months | 3-12 months
current_position: none | held | unknown
risk_budget: conservative | balanced | aggressive
decision_requested: rating | trade_plan | full_committee
```

Defaults:

- `exchange: NSE`
- `as_of: current`
- `time_horizon: 1-3 months`
- `current_position: unknown`
- `risk_budget: balanced`
- `decision_requested: full_committee`

Resolve ambiguous symbols with `groww_curate_symbols`. State the resolved company,
exchange, symbol, analysis date, horizon, and data cut before interpreting data.

## Committee state

Maintain these sections throughout the run:

```yaml
identity:
market_report:
fundamentals_report:
news_report:
sentiment_proxy_report:
bull_case:
bear_case:
research_verdict:
trader_proposal:
risk_views:
  aggressive:
  conservative:
  neutral:
final_decision:
data_gaps: []
```

Keep raw observations distinct from interpretation. Carry material data gaps
forward into every downstream decision.

## Workflow

### 1. Rehydrate and establish the evidence boundary

1. If invoked as an AITradingOffice run, call `aitradingoffice_workflows` with
   `action: get_run` and preserve its identifiers, params, employee id, and log.
2. Call `groww_resolve_market_time_and_calendar` before any current or relative
   date reasoning.
3. Resolve the instrument and verify that the company identity matches the
   symbol. Never silently substitute a similarly named security.
4. For a historical `as_of`, use only observations that existed on or before
   that date. Current news must not leak into a historical decision.
5. Optionally read recent AITradingOffice research records for the same symbol.
   Treat prior views as lessons to test, not evidence that overrides fresh data.

### 2. Choose local or delegated analysis

Use `hedgefund_employee_list` when running in the hedge-fund CEO pane.

- If suitable employees are active and their reports can return during this
  invocation, delegate clearly bounded research lanes with
  `hedgefund_delegate`.
- Send fundamentals, news, and long-horizon thesis work to an `investor`.
- Send market structure, liquidity, entry, stop, and execution-feasibility work
  to a `trader`.
- Give every delegate the exact symbol, exchange, as-of date, horizon, required
  evidence, and instruction to report data gaps.
- Keep the final synthesis and rating with the lead pane.
- If delegation is unavailable, incomplete, or would stall the run, perform the
  lanes directly. Never fabricate a missing employee report.

### 3. Build four independent analyst reports

#### Market analyst

Use historical candles, technical indicators, live quote/depth, and optionally
TradingView fallback data. Select a small complementary set rather than an
indicator dump:

- trend: 20/50/200-day averages or equivalent;
- momentum: RSI and MACD;
- volatility: ATR and Bollinger width;
- participation: volume, OBV, or VWAP where available;
- structure: recent swing high/low and decision-relevant support/resistance.

Report trend regime, momentum, volatility, liquidity, key levels, and what
would falsify the technical view. Exact levels must come from tool output.

#### Fundamentals analyst

Use `groww_fetch_stocks_fundamental_data` for financial values. Assess:

- revenue and earnings direction;
- margins and cash-generation quality;
- leverage, liquidity, and dilution;
- valuation against the company's own history or a clearly named peer context;
- ownership/shareholding changes when available;
- two strengths, two red flags, and the next fundamental catalyst.

Do not infer missing statement items. Mark stale reporting periods explicitly.

#### News analyst

Use `news_fetch` for current Indian-market coverage and `browser` only for
decision-relevant company, exchange, regulator, filing, or macro sources.
Separate:

- confirmed events;
- management or media claims;
- market interpretation;
- upcoming catalysts;
- unresolved risks.

Record publication/event dates and avoid treating repeated syndication as
independent confirmation.

#### Sentiment and attention analyst

Estimate only what the available evidence supports:

- headline tone and change in narrative;
- unusual attention or volume;
- analyst/market framing visible in sourced material;
- divergence between price action and news tone.

Label the output `news-and-attention proxy` unless direct social-platform data
was actually retrieved. Give a direction (`bullish`, `mixed`, `neutral`, or
`bearish`) and confidence (`low`, `medium`, or `high`).

### 4. Run the investment debate

Write one evidence-based bull case and one evidence-based bear case.

The bull must identify:

- the strongest growth or rerating mechanism;
- competitive or financial advantages;
- supportive technical/news evidence;
- why the bear's best objection may already be priced in.

The bear must identify:

- the most credible permanent-loss path;
- valuation, balance-sheet, governance, cyclicality, or competition risks;
- adverse technical/news evidence;
- which bullish assumption is most fragile.

Each side must directly rebut the other side's strongest point. Do not add a
second debate round unless the first round reveals a factual conflict that can
be resolved with an allowed tool.

### 5. Judge the research and draft a transaction proposal

Act as Research Manager:

1. Rank the three strongest arguments on each side.
2. Identify which claims are facts, inferences, or unresolved.
3. Issue a preliminary five-tier rating.
4. Explain why the winning side carried more evidentiary weight.

Then act as Trader and convert the verdict into a proposal:

```yaml
action: Buy | Hold | Sell
entry:
  method: market_if_liquid | limit_near_level | wait_for_confirmation | no_entry
  level_or_condition:
stop_or_invalidation:
target_or_review_condition:
time_horizon:
position_size_guidance:
reward_to_risk:
execution_notes:
```

Use `Hold`/`no_entry` when the rating is not actionable, evidence is stale, or
the setup lacks a defensible invalidation point. Position sizing is guidance
only and must reflect volatility, liquidity, confidence, and the selected risk
budget.

### 6. Run the risk committee

Produce three short, distinct views:

- **Aggressive:** articulate the maximum credible upside and the cost of waiting.
- **Conservative:** articulate drawdown, gap, liquidity, governance, and thesis
  failure risks; propose the safest acceptable alternative.
- **Neutral:** test both extremes and propose the most robust risk-adjusted path.

Each view must challenge the trader proposal, not merely restate the research.

### 7. Make the final portfolio decision

Act as Portfolio Manager. Use exactly one rating:

- `Buy`: strong conviction to initiate or add;
- `Overweight`: favorable, but build exposure gradually;
- `Hold`: maintain or wait; evidence is balanced or setup is not actionable;
- `Underweight`: reduce or avoid adding;
- `Sell`: exit or avoid because downside evidence dominates.

The final decision must include:

```yaml
rating:
confidence: low | medium | high
executive_summary:
investment_thesis:
decisive_evidence:
key_counterargument:
entry_or_action:
invalidation:
price_target_or_review_condition:
time_horizon:
position_size_guidance:
data_freshness:
data_gaps:
```

Never use confidence above `medium` when a core analyst lane is missing or when
the exact instrument identity, quote, or as-of boundary remains uncertain.

### 8. Record and stop

1. Present the compact final decision first, followed by the four analyst
   summaries, bull/bear debate, trader proposal, and risk views.
2. If attached to AITradingOffice, save one research record containing the
   resolved identity, as-of timestamp, evidence summary, debate, final decision,
   and data gaps.
3. If attached to a workflow run, append the artifact to the run log and mark
   the one-shot run `done`.
4. Stop. Do not schedule monitoring or execute the proposal.

## Quality checks

Before returning:

- Confirm every exact number is traceable to a tool result.
- Confirm news respects the as-of boundary.
- Confirm the bull and bear engaged with each other's strongest argument.
- Confirm the risk views materially changed or validated the trader proposal.
- Confirm the rating, action, sizing, and invalidation are mutually consistent.
- Confirm missing evidence lowered confidence.
- Confirm no order tool was called.
