---
name: "hedge-fund-skill-improvement-review"
description: "Review the hedge fund's AITradingOffice records, trades, P&L, desk reports, rejected ideas, and workflow logs to improve the automated hedge-fund skill system. Use after a trading day or week to identify what worked, what failed, which desk rules should change, and what concrete edits or playbook updates should be made. This is records-only and never trades."
mandate:
  kind: "review"
  persona: "hedgefund"
  risk_profile: "conservative"
  office_tool: "aitradingoffice_hf"
  allowed_tools: [groww_resolve_market_time_and_calendar, aitradingoffice_hf, aitradingoffice_workflows, hedgefund_report, watch_schedule]
  allowed_ledgers: []
  limits:
    max_trades_per_tick: 0
    max_notional: 0
    paper_only: true
  trigger: { kind: cron, expr: "35 16 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# Hedge Fund Skill Improvement Review

## Intent

Turn daily office activity into better rules. This review reads the fund book,
desk records, workflow logs, and transaction outcomes, then writes a precise
improvement plan for the CEO and desks.

## What it reviews

- CEO allocation records and the daily `screener-conviction-workflow` output.
- Desk reports from Short MIS, Long MIS, Long MTF, Investor, Index Options,
  Stock Options, Commodity, and Stock Futures.
- Open and closed equity, option, and futures transactions.
- Transaction records: thesis, stop, target, exit reason, data gaps, and rule
  breaches.
- Staged execution quality: initial tranche size, add decisions, reductions,
  premature full-size entries, and whether active monitoring improved or hurt
  the outcome.
- Candidate provenance: whether trades came from the shared screener, a local
  desk fallback, or a fresh post-screener catalyst.
- Fund P&L, drawdown, gross exposure, and unused allocation.

## Run

1. Call `groww_resolve_market_time_and_calendar`.
2. Rehydrate the workflow run when available.
3. Read fund summary, fund P&L, fund book, account ledger, positions,
   transactions, transaction records, and recent desk records.
4. Classify each desk:

```yaml
desk:
activity:
pnl_or_open_risk:
followed_rules: yes | no | partial
best_decision:
worst_decision:
missed_opportunity:
data_gap:
rule_change_needed:
```

5. Identify cross-desk issues:
   - over-allocation or idle cash;
   - repeated late exits;
   - bad symbol selection;
   - shared screener candidates that were ignored, stale, or missing;
   - stops too wide/tight;
   - full-size entries that should have been staged;
   - adds that increased risk without confirmation;
   - reductions or exits that were too slow after price movement changed;
   - option structures harmed by theta/IV;
   - futures notional/exposure crowding;
   - news or event risk missed;
   - tools unavailable or data stale.
6. Write one AITradingOffice `skill_improvement_review` record:

```yaml
review_date:
fund_state:
desk_scores:
keep_rules:
change_rules:
new_watch_items:
skill_edit_requests:
tomorrow_ceo_instructions:
```

7. Send `hedgefund_report` to the CEO pane with the top five changes.

## Guardrails

- Never trade or modify transactions.
- Do not mark a rule as failed unless the office record shows the rule existed
  before the decision.
- Separate bad process from unlucky outcome.
- Keep recommendations editable: name the exact skill and section that should
  change when a skill edit is needed.
