---
name: "hedge-fund-ceo-daily-office"
description: "Run the daily Hedge Fund CEO operating loop in AITradingOffice: read fund capital, cash, P&L, positions, employee state, and market context; run or consume screener-conviction-workflow for shared candidate generation; wake/delegate to hedge-fund desks; allocate paper risk budgets across Short MIS, Long MIS, Long MTF, Investor, Index Options, Stock Options, Commodity, and Stock Futures desks; collect reports; write the daily office plan and review state. Use when the user wants the hedge fund to operate as an automated office from available fund money. This routine coordinates and logs; it does not place trades directly."
mandate:
  kind: "routine"
  persona: "hedgefund"
  risk_profile: "balanced"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, news_fetch, browser, hedgefund_employee_list, hedgefund_employee_wake_all, hedgefund_delegate, aitradingoffice_hf, aitradingoffice_workflows, watch_schedule]
  allowed_ledgers: []
  limits:
    max_trades_per_tick: 0
    max_notional: 0
    paper_only: true
  trigger: { kind: cron, expr: "45 8 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund CEO Daily Office

## Intent

Operate the hedge fund as a paper-first AITradingOffice. Every market day, read
the fund's money, open risk, and employee state; assign desk budgets; delegate
work; collect reports; write a dated plan; and preserve lessons for tomorrow.

## Operating principles

- Capital comes only from `aitradingoffice_hf` fund state. Never assume cash,
  buying power, margin, positions, or P&L from memory.
- The CEO allocates risk, not trades. Desks place paper fund-book transactions
  under their own mandates.
- Unused allocation stays cash. Do not force activity because money is idle.
- Every delegation must include budget, trade type, risk cap, evidence required,
  stop rules, and the reporting deadline.
- Candidate generation should be centralized through
  `screener-conviction-workflow`; desks consume its watchlist and then perform
  desk-specific validation.
- Every day ends with a record that explains what was delegated, what happened,
  what changed, and what should improve.

## Desk map

```yaml
short_mis: hedge-fund-short-mis-desk
long_mis: hedge-fund-long-mis-desk
long_mtf: hedge-fund-long-mtf-desk
investor: hedge-fund-investor-desk
option_index: hedge-fund-option-index-desk
option_stock: hedge-fund-option-stock-desk
commodity: hedge-fund-commodity-desk
stock_futures: hedge-fund-stock-futures-desk
screener: screener-conviction-workflow
review: hedge-fund-skill-improvement-review
```

## Desk commands

When delegating, include the exact command the specialist should run as the last
line of the task packet. Use these commands:

```yaml
short_mis: "/skill:hedge-fund-short-mis-desk"
long_mis: "/skill:hedge-fund-long-mis-desk"
long_mtf: "/skill:hedge-fund-long-mtf-desk"
investor: "/skill:hedge-fund-investor-desk"
option_index: "/skill:hedge-fund-option-index-desk"
option_stock: "/skill:hedge-fund-option-stock-desk"
commodity: "/skill:hedge-fund-commodity-desk"
stock_futures: "/skill:hedge-fund-stock-futures-desk"
review: "/skill:hedge-fund-skill-improvement-review"
```

## Setup

1. Call `aitradingoffice_hf` with `action: health`, then `fund_summary`,
   `get_fund_book`, `fund_account_ledger`, and all position/transaction list
   actions needed to know current fund state.
2. Call `hedgefund_employee_list`; if desks are enabled but asleep, call
   `hedgefund_employee_wake_all`.
3. Confirm `screener-conviction-workflow` is available through
   `aitradingoffice_workflows`; register or create a run only if the workflow
   system reports it missing or stale.
4. Create an AITradingOffice record titled `hedge_fund_ceo_charter` containing
   this desk map, risk budget rules, daily schedule, shared screener usage, and
   paper-only rule.
5. Ensure the recurring CEO tick is scheduled with `watch_schedule`.

## Tick

### 1. Rehydrate the run

Call `aitradingoffice_workflows` with `action: get_run` when invoked by a
workflow. Read the prior log, last CEO plan, recent review records, and the most
recent records written by each desk.

### 2. Establish today's boundary

1. Call `groww_resolve_market_time_and_calendar`.
2. Record the market date, session state, expiry context when relevant, and any
   holiday/half-day warning.
3. Fetch only decision-relevant news or macro context. Label any incomplete data
   clearly.

### 3. Read the fund

Use `aitradingoffice_hf` to read:

- `fund_summary`;
- `get_fund_book`;
- `get_fund_pnl`;
- `fund_account_ledger`;
- open equity, option, and future positions;
- today's and recent transactions;
- transaction records that mention stops, review dates, breaches, or pending
  action.

Compute a plain-language risk state:

```yaml
fund_equity:
cash_available:
gross_exposure:
open_risk_at_stop:
realized_pnl_today:
unrealized_pnl:
drawdown_state: normal | caution | halt_new_risk
blocked_cash:
data_gaps: []
```

### 4. Run the shared screener

Before allocating desk-specific budgets, call `aitradingoffice_workflows` to
find today's latest `screener-conviction-workflow` run or create a fresh one.
Use it as the single shared discovery layer for Indian equity and F&O ideas.

Run a compact screen pack unless the CEO has a narrower instruction:

```yaml
skill: screener-conviction-workflow
params:
  market: india
  view: auto
  trade_kind: auto
  direction: both
  universe: fno
  timeframe: intraday
  max_candidates: 25
  risk_style: balanced
  need_oi: true
  record_to_office: true
```

If the market is not open yet, allow previous-close candidates but label them
`preopen_candidates`. If the screener run fails, record `screener_unavailable`
and permit desks to use local tools only as fallback.

Write or reference a `daily_shared_screener` office record containing:

```yaml
run_id:
as_of:
long_mis_candidates:
short_mis_candidates:
mtf_candidates:
stock_option_candidates:
stock_future_candidates:
index_bias:
rejected:
data_gaps:
```

### 5. Allocate daily budgets

Start from cash and open risk, then scale by drawdown state:

- `normal`: allocate up to 45% of available cash notionally across desks.
- `caution`: allocate up to 20%; only A-grade setups may open.
- `halt_new_risk`: no new trades; delegate exits/reviews only.

Default desk weights before scaling:

```yaml
long_mis: 12
short_mis: 10
long_mtf: 18
investor: 0
option_index: 18
option_stock: 10
commodity: 10
stock_futures: 12
cash_reserve: 10
```

Adjust weights using market context:

- Trend day with breadth support: favor `long_mis`, `stock_futures`, and
  `option_index`.
- Weak breadth or gap-down follow-through: favor `short_mis` and defined-risk
  index options.
- Event/expiry day: reduce option selling and illiquid stock options.
- High drawdown or stale data: raise cash reserve first.

### 6. Delegate

For each active desk, call `hedgefund_delegate` with a complete task packet:

```yaml
desk_skill:
desk_command:
budget_notional:
max_loss_today:
max_open_positions:
market_boundary:
allowed_trade_types:
preferred_universe:
shared_screener_record:
execution_style:
  initial_tranche_pct:
  add_rules:
  reduce_rules:
  full_size_allowed_only_if:
must_check:
  - fund allocation still available
  - candidate appears in or is reconciled against screener-conviction-workflow
  - tradeable status
  - margin or premium requirement
  - liquidity/depth
  - stop/invalidation
  - staged entry and active reduction plan
  - transaction record schema
report_deadline:
last_step: "Run <desk_command> and follow that specialist skill exactly."
```

Do not delegate a desk if the fund state is unreadable, the market boundary is
unclear, or the desk's risk would conflict with existing fund exposure.

Use the desk command matching the assignment:

- Short MIS tasks end with `/skill:hedge-fund-short-mis-desk`.
- Long MIS tasks end with `/skill:hedge-fund-long-mis-desk`.
- Long MTF tasks end with `/skill:hedge-fund-long-mtf-desk`.
- Investor research tasks end with `/skill:hedge-fund-investor-desk`.
- Index option tasks end with `/skill:hedge-fund-option-index-desk`.
- Stock option tasks end with `/skill:hedge-fund-option-stock-desk`.
- Commodity tasks end with `/skill:hedge-fund-commodity-desk`.
- Stock futures tasks end with `/skill:hedge-fund-stock-futures-desk`.

### 7. Collect and decide

Read employee reports and desk records. For each proposal:

- approve only if it fits allocation, limits, and current fund exposure;
- reject any proposal that spends the full desk allocation in one entry without
  a written reason, such as a tiny position, exceptional liquidity, or a
  structure whose risk is already fully defined;
- prefer ideas traceable to the shared screener record unless the desk explains
  a fresh intraday catalyst that appeared after the screener run;
- reject if the thesis lacks invalidation, liquidity, margin, or record plan;
- ask for revision only when the desk can fix the gap before the deadline.

Write one `daily_ceo_plan` record with allocation, approvals, rejections,
open-risk summary, and desk instructions.

### 8. End-of-day handoff

After market close or on the next CEO tick, read current fund P&L and desk
records. Delegate `hedge-fund-skill-improvement-review` with the day, shared
screener record, desk reports, transactions, rejected ideas, limit breaches, and
missing data. Update the workflow log with tomorrow's watch items.

## Guardrails

- Never call create/exit/cancel transaction actions from this CEO skill.
- Never allocate more than available fund cash or the manifest's paper limit
  used by each desk.
- Never ask a desk to trade outside its mandate or ledger.
- If employee delegation is unavailable, write a CEO plan with `delegation_failed`
  and stop; do not impersonate all desks in the CEO pane.
- Keep recommendations auditable: every allocation must point to fund state,
  market state, or a prior review lesson.
