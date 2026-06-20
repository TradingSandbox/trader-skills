---
name: "{{SLOT.name}}"
description: "{{SLOT.description}}"
# --- mandate manifest (read by the runner/guard; the SKILL loader ignores these keys) ---
mandate:
  kind: "scenario"
  persona: "{{SLOT.persona}}"           # hedgefund | pms | profile
  risk_profile: "{{SLOT.risk_profile}}" # conservative | balanced | aggressive
  office_tool: "{{SLOT.office_tool}}"   # aitradingoffice_hf | aitradingoffice_pms | aitradingoffice_profile
  allowed_tools: {{SLOT.allowed_tools}}
  allowed_ledgers: {{SLOT.allowed_ledgers}}  # subset of [equities, options, futures]
  limits:
    max_trades_per_tick: {{SLOT.max_trades_per_tick}}
    max_notional: {{SLOT.max_notional}}
    paper_only: {{SLOT.paper_only}}
  trigger: {{SLOT.trigger}}             # cron for the monitor tick, e.g. { kind: cron, expr: "0 9 * * 1-5", tz: "Asia/Kolkata" }
  lifecycle:
    success: "{{SLOT.success}}"
    invalidation: "{{SLOT.invalidation}}"
    max_days: {{SLOT.max_days}}
---

# {{SLOT.title}}

> **Scenario mandate — do not run this template directly.** Fill every `{{SLOT.*}}`
> and flatten into `.pi/skills/<name>/SKILL.md`.

## Intent

{{SLOT.intent}}

## What this skill is

An **episodic** mandate: form a thesis from the Intent, trade it within fixed
limits, monitor over days, and close when the thesis resolves or breaks. It adds
no new capability — it points existing tools at this scenario.

## State lives in the run

The run's `log` is the memory across ticks — thesis, decisions, what changed;
trades it places live on the persona's central transactions (with research as
transaction records). **Re-read the run first on every tick** (`get_run`). Never
assume continuity from this prompt alone.

## Limits — hard, enforced outside this prompt

Only `{{SLOT.allowed_ledgers}}`; ≤ `{{SLOT.max_trades_per_tick}}` trades/tick;
notional ≤ `{{SLOT.max_notional}}`; `paper_only = {{SLOT.paper_only}}`. Every
trade carries the run's `employee_id`. A violating call is rejected before it
reaches a ledger.

## Persona & disposition

Act as **{{SLOT.persona}}**, **{{SLOT.risk_profile}}**: {{SLOT.personality}}

## Modes (selected by run status)

### Open  (pending → running)
1. Form the thesis from the Intent; load `trading-thesis` / `indian-market-news`
   to generate and validate. Do not act on a `contradicted`/`unconfirmed` thesis.
2. Record the thesis in the run's `log` (and, once a trade is opened, as a record
   on that transaction).
3. Plan trades that fit the validated thesis, sized for `{{SLOT.risk_profile}}`,
   within Limits.
4. Execute: if `paper_only` or within auto-approve budget, place via
   `{{SLOT.office_tool}}` with `employee_id`; else present the plan and **request
   approval** first.
5. Journal the plan and fills as records.
6. Schedule the next tick (`watch_schedule`, manifest `trigger`); persist `next_run_at`.

### Tick  (waiting → running)
1. Re-hydrate from the run (`get_run`) and its central transactions.
2. Gather fresh context (news, charts, quotes) with allowed tools.
3. Evaluate against `lifecycle.success` and `lifecycle.invalidation`.
4. Act within Limits: add / trim / exit — or hold and say why.
5. Journal a dated record.
6. On `success`, `invalidation`, or `max_days` → **Close**; else reschedule.

### Close  (→ done | invalidated)
1. Flatten or hand off remaining positions per Limits.
2. Write a retro record: outcome vs. thesis, realized P&L, lesson.
3. Stop the schedule (`watch_schedule` stop).
4. Mark the run `done` (resolved) or `invalidated` (thesis broke).

## Guardrails
- Never exceed the Limits — the guard rejects it.
- Never fabricate news/filings/prices; verify with `browser`/`indian-market-news`.
- Keep thesis validity separate from entry timing.
- Don't claim monitoring beyond the scheduled ticks.
