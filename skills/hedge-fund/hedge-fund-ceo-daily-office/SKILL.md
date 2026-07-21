---
name: "hedge-fund-ceo-daily-office"
description: "Run one Hedge Fund CEO office tick: inspect the authoritative AITradingOffice book and roster, set paper-risk budgets, delegate bounded research/desk tasks, collect reports, and record the daily decision plan. The CEO coordinates but never books trades directly."
mandate:
  kind: "routine"
  persona: "hedgefund"
  roles: [ceo]
  risk_profile: "conservative"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, hedgefund_delegate, news_fetch, browser, watch_schedule, tv_health_check, tv_chart_get_state, tv_quote_get]
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

## Runtime semantics

One manual invocation is one office tick. The frontmatter trigger is policy
metadata; loading this skill does not install a cron job. Only when the user
explicitly requests recurring review may `watch_schedule` create an interval
schedule (minimum one minute), and it runs only while a scheduler owns the
pane. Do not promise exact weekday/08:45 execution or offline survival.
AITradingOffice workflow rows persist state and `next_run_at`; they do not
execute another skill by themselves.

## Authority

AITradingOffice is the source of truth for roster, book, records, positions,
transactions, P&L, and workflow state. `status: active` means an employee is
eligible in the office roster, not that a Pi process is running. Delegation
success/failure is the runtime fact. The CEO may create/update research records
and workflow narratives, but must never create or exit a transaction directly
or call the trade pipeline.

## Tick

1. Check `tv_health_check` and record current runtime/source time. If current
   India-session or calendar status matters, verify it from official browsed
   sources; ambiguity blocks new desk-entry mandates but not risk reduction.
2. Read, at minimum, with `aitradingoffice_hf`:
   - `get_book`/`fund_summary`, `get_fund_pnl`, and `market_terminal`;
   - `list_clients` and constraints;
   - `list_employees` (the authoritative team roster);
   - recent employee research records and transaction/position state needed
     for the plan.
3. Reconcile cash, current exposure, open stop risk, realized/unrealized P&L,
   client restrictions, stale/missing data, and unresolved prior reports.
4. Set conservative desk envelopes. A CEO allocation is only an upper bound;
   each skill mandate, active client constraint, employee mandate, server risk
   check, available cash, and current exposure may lower or block it.
5. Generate the shared candidate task by delegating a self-contained request
   whose final line is exactly `/skill:screener-conviction-workflow`. Do not
   create a workflow row and pretend it ran the skill. Consume the resulting
   research record/report only when delegation actually succeeds.
6. Delegate desk tasks to the specific roster employee with objective, book,
   allocation, risk ceiling, input record IDs, deadline, required report, and
   the exact desk `/skill:...` command as the final line. There is no
   `wake_all`: `hedgefund_delegate` is the start/route operation. A failed,
   missing, or capacity-blocked delegation remains visible and unassigned.
7. Collect completed reports and distinguish them from merely active roster
   entries. Resolve conflicts, preserve rejected ideas and reasons, and never
   infer a position from prose when the office ledger disagrees.
8. Create/update a daily CEO research record containing book snapshot,
   allocations, delegations and their runtime outcomes, report IDs, decisions,
   blocked risks, data gaps, and next review. Update an existing workflow run
   only with truthful tick state/status.

## Close

Return a compact office summary: authoritative capital/exposure, active-client
constraints, employee eligibility, which agents were actually routed, desk
budgets, open positions/risk, accepted and rejected ideas, missing reports,
and next actions. Never say “all desks are running” merely because six
employees are active in AITradingOffice.
