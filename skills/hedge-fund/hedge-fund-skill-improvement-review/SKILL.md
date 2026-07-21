---
name: "hedge-fund-skill-improvement-review"
description: "Review a Hedge Fund book's office records, employee reports, transactions, positions, P&L, and workflow logs to propose evidence-backed process or skill improvements. Use for post-session review; it never trades or edits installed skill files."
mandate:
  kind: "review"
  persona: "hedgefund"
  roles: [ceo, investor]
  risk_profile: "conservative"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
  allowed_ledgers: []
  limits:
    max_trades_per_tick: 0
    max_notional: 0
    paper_only: true
  trigger: { kind: cron, expr: "30 16 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Skill Improvement Review

## Runtime semantics

One invocation is one review. The trigger is policy metadata only. On explicit
request, `watch_schedule` may install an interval schedule while a scheduler
owns the pane; it cannot guarantee an exact cron time or offline execution. Its
prompt must begin exactly with `/skill:hedge-fund-skill-improvement-review`.
Workflow records persist review state but do not invoke skills. This review may
recommend edits, but cannot modify package files or silently change mandates.

## Evidence collection

1. Read `fund_summary`, `get_fund_pnl`, and `market_terminal` for the whole
   selected book.
2. Read `list_employees`, then iterate relevant `employee_id` values for their
   research records, equity/option/future transaction lists, positions, and
   attached transaction records. Do not confuse roster-active with runtime-
   running.
3. Use `aitradingoffice_workflows` to inspect due/running/done/failed runs and
   logs for the review window. Treat missing or contradictory evidence as a
   finding; do not reconstruct fills or outcomes from narrative.
4. Join decisions by explicit record, transaction, employee, book, workflow,
   plan, or delegation identifiers. Separate unrealized from realized P&L and
   market outcome from process quality.

## Assessment

For each desk/process, compare:

- mandate/tool compliance and whether the deterministic pipeline was used;
- entry thesis, stop/target/time-exit plan, additions, reductions, and actual
  settlement evidence;
- risk budget versus reserved/realized exposure;
- stale data, unsupported claims, retries around blocked gates, or missing
  client checks;
- shared-screener usefulness, rejected-idea quality, delegation/runtime
  failures, and report completeness;
- repeated errors that warrant code or skill changes versus one-off market
  noise.

Propose only changes supported by at least one concrete evidence ID. Classify
each as documentation/skill instruction, deterministic code/tool contract,
office data quality, runtime/supervision, or human policy. Include expected
benefit, downside, validation test, rollback criterion, and priority. Never
relax a hard risk boundary to improve backtest or P&L optics.

## Output

Create/update a review research record through `aitradingoffice_hf` with the
review window, joined evidence IDs, findings, proposed changes, validation
plan, and unresolved gaps. Update the existing review workflow run when one
exists. In an Investor pane, send the result with `hedgefund_report`; in a CEO
pane, return/store it locally because that reporting tool is not part of the
CEO surface.
