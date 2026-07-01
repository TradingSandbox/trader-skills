---
name: "screener-conviction-workflow"
description: "Run a decision-ready Indian equity and F&O screener workflow using TradeCLI's TradingView, Groww, news, derivative, and AITradingOffice research tools. Use when the user wants named views such as high momentum, 52-week breakout, RVOL spike, pullback, oversold bounce, bearish breakdown, intraday movers, or sector rotation, then adapt candidates to trade kinds such as CNC, MIS, MTF, futures, options buying, or options selling. Analysis-only: never places, modifies, or exits orders."
mandate:
  kind: "oneshot"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, groww_curate_symbols, groww_fetch_technical_screener, groww_fetch_fundamentals_screener, groww_fetch_stocks_fundamental_data, groww_fetch_historical_candle_data, groww_get_historical_technical_indicators, groww_get_historical_candlestick_patterns, groww_fetch_curated_fno, groww_fno_mcx_contracts_search_tool, groww_get_open_interest_analysis, groww_get_greeks_for_fno_contract, groww_get_greeks_for_fno_symbol, groww_calculate_equity_margin, groww_calculate_fno_margin, groww_get_quotes_and_depth, news_fetch, browser, stock_trend_snapshot, tv_health_check, tv_screener_fields, tv_screener_ops, tv_screener_query, tv_symbol_search, tv_symbol_info, tv_quote_get, tv_chart_set_symbol, tv_chart_set_timeframe, tv_chart_manage_indicator, tv_data_get_ohlcv, tv_data_get_study_values, tv_data_get_indicator, tv_capture_screenshot, tv_draw_shape, aitradingoffice_hf, aitradingoffice_workflows]
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

## Intent

Convert named screener views into a short, evidence-ranked watchlist or paper
trade plan, then adapt each candidate to the intended trade kind. A view is the
market idea, such as breakout stocks, high momentum leaders, pullbacks, oversold
bounces, or bearish breakdowns. A trade kind is the execution wrapper, such as
CNC, MIS, MTF, futures, option buying, or option selling.

## Scope

- Market: Indian listed equities unless the user explicitly asks otherwise.
- Views: high momentum, 52-week breakout, RVOL spike, pullback in uptrend,
  oversold bounce, bearish breakdown, intraday bearish momentum, sector
  rotation, or custom combinations built from TradingView attributes.
- Trade kinds: CNC/swing equity, MIS intraday equity, MTF/positional equity,
  futures, options buying, options selling, or index trading.
- Output: ranked candidates, rejected candidates, and at most four decision-ready
  setups.
- This is analysis-only. Do not place, modify, cancel, or exit any order.

## Intake

Resolve these fields without unnecessary questions:

```yaml
market: india
view: auto | high_momentum | breakout_52w | rvol_spike | pullback_uptrend | oversold_bounce | bearish_breakdown | intraday_bearish | sector_rotation | custom
trade_kind: auto | CNC | MIS | MTF | futures | options_buying | options_selling | index
direction: long | short | both | auto
universe: all_nse | large_caps | fno | user_list
timeframe: intraday | swing_1_10d | positional_1_4w
max_candidates: 25
risk_style: conservative | balanced | aggressive
need_oi: false
record_to_office: true
```

Defaults: `view: auto`, `trade_kind: CNC`, `direction: auto`, `universe:
large_caps`, `timeframe: swing_1_10d`, `max_candidates: 25`, `risk_style:
balanced`, `need_oi: false`.

Always call `groww_resolve_market_time_and_calendar` before using current,
today, yesterday, tomorrow, weekly, monthly, expiry, or relative-date language.

## Core workflow

### 1. Establish the data boundary

1. State the market date/time, whether the market is open, and whether live or
   previous-close data is being used.
2. If running inside an AITradingOffice workflow, hydrate the run with
   `aitradingoffice_workflows` and preserve identifiers for the final record.
3. Confirm the screener fields/operators with `tv_screener_fields` and
   `tv_screener_ops` when using a field not already known from this skill.
4. Prefer `tv_screener_query` for custom TradingView columns and ranking. Use
   Groww screeners as fallback or cross-check.

### 2. Know the available screener surface

TradingView has screeners across stocks, forex, crypto, and ETFs. This skill
defaults to the stock screener for Indian equities, which is the richest surface:

- Price and volume: price, change %, volume, relative volume, average volume.
- Technicals: RSI, MACD, Stochastic, ADX, ATR, CCI, Momentum, Williams %R.
- Moving averages: SMA/EMA 5/10/20/50/100/200 and VWAP where exposed.
- Candle patterns: doji, hammer, engulfing, harami, morning/evening star, and
  related pattern fields when exposed by the screener.
- Performance: 1W, 1M, 3M, 6M, YTD, 1Y, 5Y returns.
- Fundamentals and valuation: market cap, P/E, P/B, P/S, EPS, dividend yield,
  revenue, net income, debt/equity, ROE, ROA, current ratio, EV/EBITDA, PEG,
  free cash flow, and book value.
- Description: exchange, sector, industry, country, index membership.

Treat this as an attribute map, not an exact schema. Before filtering on any
non-core field, call `tv_screener_fields` to get the exact accepted field name.
For Indian-market fundamentals, prefer Groww fundamentals or a browser check of
Screener.in/company filings when precision matters.

### 3. Build named views from attributes

Use the requested view as the primary plan. If the user asks for "best stocks
today" or `view: auto`, run a compact view stack: `high_momentum`,
`breakout_52w`, `pullback_uptrend`, `oversold_bounce`, and
`bearish_breakdown`, then rank the best setups across views.

Each view has:

```yaml
view:
intent:
attributes:
query:
sort:
post_filters:
reject_if:
deep_check_focus:
```

#### View: high_momentum (trend continuation)

Intent: find stocks already trending up with strength and liquidity.

Attributes: `ADX`, `RSI`, `Perf.W`, `Perf.1M`, `change`,
`average_volume_30d_calc`, `relative_volume_10d_calc`, `market_cap_basic`,
`close`, `EMA20`, `EMA50`, `sector`.

```json
{
  "market": "india",
  "filter": [
    { "left": "ADX", "operation": "greater", "right": 25 },
    { "left": "RSI", "operation": "in_range", "right": [55, 70] },
    { "left": "Perf.W", "operation": "greater", "right": 1 },
    { "left": "change", "operation": "greater", "right": 0.5 },
    { "left": "average_volume_30d_calc", "operation": "greater", "right": 300000 },
    { "left": "market_cap_basic", "operation": "greater", "right": 5000000000 }
  ],
  "sort": { "sortBy": "ADX", "sortOrder": "desc" },
  "columns": [
    "name", "close", "change", "ADX", "RSI", "EMA20", "EMA50",
    "Perf.W", "Perf.1M", "relative_volume_10d_calc", "sector"
  ],
  "range": [0, 25]
}
```

Post-filter: keep only rows where `close > EMA20 > EMA50`.
Reject if RSI is above 75, price is extremely extended from EMA20, or the latest
candle is a high-volume rejection.

#### View: pullback_uptrend

Intent: find strong stocks cooling off without breaking trend.

Attributes: `RSI`, `Perf.W`, `Perf.1M`, `Perf.3M`,
`average_volume_30d_calc`, `market_cap_basic`, `close`, `EMA20`, `EMA50`,
`EMA200`, `sector`.

```json
{
  "market": "india",
  "filter": [
    { "left": "RSI", "operation": "in_range", "right": [38, 52] },
    { "left": "Perf.3M", "operation": "greater", "right": 8 },
    { "left": "Perf.W", "operation": "less", "right": 0 },
    { "left": "average_volume_30d_calc", "operation": "greater", "right": 300000 },
    { "left": "market_cap_basic", "operation": "greater", "right": 15000000000 }
  ],
  "sort": { "sortBy": "Perf.3M", "sortOrder": "desc" }
}
```

Post-filter: `close > EMA50`; best names hold `close > EMA50 > EMA200`.
Reject if price closes below EMA50 on high volume or the weekly structure has
already made a lower-low breakdown.

#### View: breakout_52w

Intent: find fresh leadership near or above 52-week highs.

Attributes: `relative_volume_10d_calc`, `Perf.1M`, `Perf.W`, `RSI`,
`average_volume_30d_calc`, `market_cap_basic`, `close`,
`price_52_week_high`, `ADX`, `sector`.

```json
{
  "market": "india",
  "filter": [
    { "left": "relative_volume_10d_calc", "operation": "greater", "right": 1.5 },
    { "left": "Perf.1M", "operation": "greater", "right": 5 },
    { "left": "Perf.W", "operation": "greater", "right": 1 },
    { "left": "RSI", "operation": "in_range", "right": [55, 80] },
    { "left": "average_volume_30d_calc", "operation": "greater", "right": 300000 },
    { "left": "market_cap_basic", "operation": "greater", "right": 5000000000 },
    { "left": "close", "operation": "greater", "right": 50 }
  ],
  "sort": { "sortBy": "Perf.1M", "sortOrder": "desc" },
  "columns": [
    "name", "close", "change", "price_52_week_high",
    "relative_volume_10d_calc", "RSI", "ADX", "Perf.W", "Perf.1M", "sector"
  ],
  "range": [0, 25]
}
```

Post-filter: keep only rows where `close / price_52_week_high >= 0.97`, then
confirm volume expansion and no obvious failed-breakout candle.
Reject if the breakout candle is too extended for the user's timeframe; label it
`wait_for_pullback` instead of `trade_now`.

#### View: rvol_spike

Intent: find unusual participation that may mark fresh institutional interest.

Attributes: `relative_volume_10d_calc`, `change`, `volume`, `close`,
`market_cap_basic`, `RSI`, `Perf.W`, `Perf.1M`, `sector`.

```json
{
  "market": "india",
  "filter": [
    { "left": "relative_volume_10d_calc", "operation": "greater", "right": 2 },
    { "left": "change", "operation": "greater", "right": 1.5 },
    { "left": "market_cap_basic", "operation": "greater", "right": 5000000000 },
    { "left": "close", "operation": "greater", "right": 50 }
  ],
  "sort": { "sortBy": "relative_volume_10d_calc", "sortOrder": "desc" }
}
```

Post-filter: require constructive price location: above EMA20/EMA50 for longs or
breaking a clear resistance zone. Reject one-day news spikes with no structure.

#### View: oversold_bounce

Intent: find contrarian mean-reversion candidates where long-term trend is not
broken.

Attributes: `RSI`, `Perf.1M`, `Perf.3M`, `market_cap_basic`,
`average_volume_30d_calc`, `close`, `EMA50`, `EMA200`, `BB.lower`,
`BB.basis`, `sector`.

```json
{
  "market": "india",
  "filter": [
    { "left": "RSI", "operation": "in_range", "right": [28, 38] },
    { "left": "Perf.3M", "operation": "greater", "right": 5 },
    { "left": "market_cap_basic", "operation": "greater", "right": 10000000000 },
    { "left": "average_volume_30d_calc", "operation": "greater", "right": 300000 }
  ],
  "sort": { "sortBy": "RSI", "sortOrder": "asc" }
}
```

Post-filter: `close > EMA200`; reject if price is breaking EMA200 on heavy
volume.
Prefer large/liquid names. Require a nearby support/invalidation level.

#### View: bearish_breakdown

Intent: find short candidates and weak-sector follow-through.

Attributes: `RSI`, `Perf.W`, `Perf.1M`, `volume`, `average_volume_30d_calc`,
`market_cap_basic`, `close`, `EMA20`, `EMA50`, `EMA200`, `MACD.macd`,
`MACD.signal`, `MACD.hist`, `ADX`, `sector`.

```json
{
  "market": "india",
  "filter": [
    { "left": "RSI", "operation": "less", "right": 45 },
    { "left": "Perf.W", "operation": "less", "right": -1 },
    { "left": "Perf.1M", "operation": "less", "right": 0 },
    { "left": "volume", "operation": "greater", "right": 500000 },
    { "left": "market_cap_basic", "operation": "greater", "right": 5000000000 }
  ],
  "sort": { "sortBy": "Perf.W", "sortOrder": "asc" }
}
```

For intraday bearish momentum, use `change < -1.5`, `RSI < 45`, `volume >
500000`, `market_cap_basic > 5000000000`, sorted by `change` ascending.

Post-filter: `close < EMA20 < EMA50`; higher conviction when `close < EMA200`
and MACD histogram is negative. Remove penny stocks, illiquid names, and very
small caps unless the user explicitly wants them.

#### View: intraday_bearish

Intent: find liquid stocks selling off today with enough participation for an
intraday or very short swing setup.

Attributes: `change`, `RSI`, `volume`, `market_cap_basic`, `close`,
`MACD.macd`, `MACD.signal`, `BB.lower`, `EMA20`, `EMA50`, `sector`.

```json
{
  "market": "india",
  "filter": [
    { "left": "change", "operation": "less", "right": -1.5 },
    { "left": "RSI", "operation": "less", "right": 45 },
    { "left": "volume", "operation": "greater", "right": 500000 },
    { "left": "market_cap_basic", "operation": "greater", "right": 5000000000 }
  ],
  "sort": { "sortBy": "change", "sortOrder": "asc" }
}
```

Post-filter: avoid names already stretched far below lower Bollinger Band unless
the setup is explicitly a continuation short. Pull extended columns for the
shortlist before final ranking.

#### View: sector_rotation

Intent: find where money is rotating by grouping candidates by sector.

Attributes: `sector`, `Perf.W`, `Perf.1M`, `Perf.3M`, `relative_volume_10d_calc`,
`market_cap_basic`, `RSI`, `ADX`, `change`.

Run `high_momentum`, `rvol_spike`, and `bearish_breakdown` views, then group
results by sector. Prefer sectors with multiple confirming names. Penalize lone
outliers unless news explains the divergence.

#### View: custom

Intent: build a query from the user's attribute idea.

1. Translate the user's plain-English view into TradingView attributes.
2. Call `tv_screener_fields` and `tv_screener_ops` for exact names/operators.
3. Create the minimal query that expresses the view.
4. Pull columns needed for post-filtering and interpretation.
5. State every manual post-filter explicitly.

### 4. Respect TradingView screener limits

TradingView REST screener filters may not reliably support cross-field
comparisons such as `close > EMA20`, `EMA20 > EMA50`, or `MACD.macd <
MACD.signal`. Pull all needed columns and do these comparisons in the analysis.
When the query accepts a cross-field filter, still verify it manually on the
returned rows.

Default columns for candidate screens:

```text
name, close, change, open, high, low, volume,
average_volume_30d_calc, relative_volume_10d_calc,
RSI, ADX, MACD.macd, MACD.signal, MACD.hist,
EMA20, EMA50, EMA200,
BB.upper, BB.lower, BB.basis,
Stoch.K, Stoch.D, ATR,
Perf.W, Perf.1M, Perf.3M,
price_52_week_high, price_52_week_low,
sector, market_cap_basic
```

### 5. Rank the candidates

Score each candidate from 0-5 across:

- Setup fit: how cleanly it matches the preset.
- Trend confirmation: EMA stack, swing structure, ADX, higher highs/lows or
  lower highs/lows.
- Participation: current volume, relative volume, and whether high volume
  confirms the intended direction.
- Sector/peer context: sector moving with or against the stock.
- Catalyst and event risk: recent news, results, guidance, corporate actions.
- Reward-to-risk: distance to invalidation versus target.

Prefer names appearing across multiple compatible screens. Penalize extended
moves, low liquidity, unclear stops, and event risk that cannot be sized.

### 6. Apply the trade-kind overlay

After ranking candidates by view, reinterpret the shortlist through
`trade_kind`. The same view can lead to different conclusions for different
trade kinds.

#### Trade kind: CNC

Use for cash equity swing or positional trades.

- Best views: `high_momentum`, `breakout_52w`, `pullback_uptrend`,
  `oversold_bounce`.
- Required checks: daily and weekly trend, EMA20/50/200, support/resistance,
  news, sector, basic fundamentals, liquidity.
- Reject if the setup depends on same-day momentum only, has major event risk
  before the intended holding period, or lacks a daily invalidation level.
- Output should include entry zone, daily-close invalidation, targets, time
  horizon, and whether to enter now or wait.

#### Trade kind: MIS

Use for intraday equity trades.

- Best views: `intraday_bearish`, `rvol_spike`, `bearish_breakdown`,
  intraday `high_momentum`.
- Required checks: 5m/15m OHLCV, opening range, VWAP or intraday moving
  averages when available, current volume/RVOL, quote/depth, nearby day high/low.
- Reject if spread/depth is poor, the move is already too extended from VWAP or
  lower/upper Bollinger Band, or the stop must be wider than the intraday plan.
- Output should include trigger, hard stop, target, no-trade zone, and square-off
  assumption. Do not use daily investment logic for MIS.

#### Trade kind: MTF

Use for margin-funded positional equity ideas.

- Best views: `pullback_uptrend`, `high_momentum`, selective `oversold_bounce`.
- Required checks: large-cap/liquid universe, lower gap risk, daily/weekly trend,
  margin estimate with `groww_calculate_equity_margin` where useful, results/news
  calendar risk, and deeper drawdown tolerance.
- Reject if the stock is event-heavy, illiquid, highly gapped, or below EMA200
  unless the user explicitly wants a contrarian MTF idea.
- Output must call out leverage risk and a tighter invalidation than CNC.

#### Trade kind: futures

Use when the underlying is F&O eligible and the user wants leveraged directional
exposure.

- Best views: `high_momentum`, `bearish_breakdown`, `sector_rotation`,
  `rvol_spike`.
- Required checks: F&O eligibility/contract discovery, futures quote/volume/OI,
  margin with `groww_calculate_fno_margin`, daily plus intraday structure, and
  futures chart if available in TradingView.
- Treat futures Volume Profile as useful only on the futures symbol, not the
  cash chart.
- Reject if OI/volume does not confirm, contract liquidity is poor, or the stop
  implies unacceptable lot-size risk.

#### Trade kind: options_buying

Use when the user wants convex exposure to a directional move.

- Best views: `breakout_52w`, `rvol_spike`, `bearish_breakdown`,
  `intraday_bearish`.
- Required checks: underlying direction first, OI by strike, option liquidity,
  bid/ask spread, IV/expiry risk, Greeks for candidate contracts, and time to
  expiry.
- Prefer options only when expecting momentum expansion or event-driven movement.
- Reject if the underlying setup is slow, IV is already too rich, theta risk
  dominates, or the option spread is wide.
- Output should include underlying trigger, candidate strike/expiry, premium
  risk, invalidation on underlying, and max loss assumption.

#### Trade kind: options_selling

Use when the user wants defined range, mean reversion, or premium decay logic.

- Best views: `oversold_bounce`, range-bound custom views, sector/index levels.
- Required checks: OI support/resistance, max pain or high-OI clusters when
  available, IV context, margin with `groww_calculate_fno_margin`, gap/event
  risk, and payoff-risk explanation.
- Reject naked selling when trend is breaking strongly, event risk is near, or
  loss is undefined without an explicit hedge.
- Prefer defined-risk spreads in the written plan when risk is not clearly
  bounded.

#### Trade kind: index

Use for NIFTY/BANKNIFTY/index futures or options.

- Best views: `sector_rotation`, `intraday_bearish`, `rvol_spike`, custom
  breadth and OI views.
- Required checks: index chart, futures chart when relevant, options OI by
  strike, high CE/PE OI zones, global cues if market-moving, sector leadership,
  and intraday breadth.
- Output support/resistance should distinguish price levels, futures levels, and
  options OI strike levels.

When `trade_kind: auto`, infer it from the user's language:

- "delivery", "swing", "CNC", "positional" -> `CNC`
- "intraday", "MIS", "today only", "scalp" -> `MIS`
- "margin", "MTF", "hold with leverage" -> `MTF`
- "futures", "fut" -> `futures`
- "calls", "puts", "option buy" -> `options_buying`
- "sell options", "write", "credit spread", "theta" -> `options_selling`
- "NIFTY", "BANKNIFTY", "index expiry" -> `index`

### 7. Deep-check only the shortlist

For the top candidates, switch TradingView to the symbol and inspect daily and
weekly:

1. Use `tv_chart_set_symbol`, `tv_chart_set_timeframe`, `tv_data_get_ohlcv`,
   and `tv_data_get_study_values`.
2. Add or verify RSI, MACD, Bollinger Bands, EMA20, EMA50, EMA200, and Volume
   Profile if available via the UI. If Volume Profile cannot be read, infer
   only approximate high-volume/low-volume zones from visible OHLCV and label
   them as inferred, not native VP.
3. Use `groww_get_historical_candlestick_patterns` when formal candle-pattern
   detection is useful.
4. Check:
   - daily and weekly trend structure;
   - candlestick patterns;
   - gaps and unfilled gap zones;
   - support/resistance from swing highs/lows, 52-week levels, EMAs, and round
     numbers;
   - volume profile concepts: POC, VAH, VAL, HVN, LVN when available;
   - volume behavior: high-volume breakdown/breakout, low-volume bounce,
     accumulation or distribution.
5. Capture a chart screenshot when the chart view materially supports the
   decision.

### 8. Add sector, news, fundamentals, and OI only where useful

- Use `news_fetch` for recent company, sector, and macro catalysts. Use
  `browser` for primary company/exchange/regulator pages when precision matters.
- Use `groww_fetch_stocks_fundamental_data` for valuation, margins, leverage,
  shareholding, and dividend context.
- Use `groww_get_open_interest_analysis` only for index/F&O names or when the
  user asks for options/futures context. Treat high CE OI as potential
  resistance and high PE OI as potential support, but never as a prediction.
- For F&O or index-futures chart context, futures volume profile can be checked
  on the futures symbol in TradingView. Options OI cannot be natively overlaid
  by TradingView; draw manual OI strike lines only if the user asks.

### 9. Final artifact

Return a concise decision table plus notes:

```yaml
as_of:
data_sources:
candidates_screened:
shortlist:
  - symbol:
    setup:
    view:
    trade_kind:
    score:
    entry_zone:
    stop_or_invalidation:
    targets:
    reward_to_risk:
    timeframe:
    execution_checks:
    confidence:
    trade_now_or_wait:
    reason:
    key_risks:
    derivative_contract:
rejected:
  - symbol:
    reason:
data_gaps:
```

Use these final labels:

- `trade_now`: entry trigger already present and R:R is acceptable.
- `wait_for_pullback`: setup is strong but extended.
- `wait_for_breakout`: needs confirmation above a level.
- `watchlist_only`: useful but incomplete.
- `reject`: poor R:R, broken structure, weak liquidity, or unsupported thesis.

If `record_to_office` is true and an office context exists, save the final
artifact with `aitradingoffice_hf` `action: create_record` or
`update_record`.

## Guardrails

- Do not present screen results as live unless the tool returned current data.
- Do not call a level support/resistance without naming the evidence behind it.
- Do not treat Volume Profile, OI, RSI, MACD, or any single indicator as
  predictive. They are context and probability tools.
- Do not average down or suggest averaging down in a failed trade plan.
- Do not recommend a trade without a clear invalidation level.
- Do not recommend futures or options exposure without checking contract
  liquidity, margin or premium risk, and expiry/event risk.
- Do not convert a cash-equity signal into an option or futures idea unless the
  trade-kind overlay still passes after derivative checks.
- Keep current trades separate from investment/DCA ideas.
- Always say when a cross-field filter was manually post-filtered.
