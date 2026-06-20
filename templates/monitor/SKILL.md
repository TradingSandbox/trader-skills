---
name: "{{SLOT.name}}"
description: "{{SLOT.description}}"
# --- mandate manifest (read by the runner/guard; the SKILL loader ignores these keys) ---
mandate:
  kind: "monitor"
  persona: "{{SLOT.persona}}"
  risk_profile: "{{SLOT.risk_profile}}"
  office_tool: "{{SLOT.office_tool}}"
  allowed_tools: {{SLOT.allowed_tools}}
  allowed_ledgers: []                   # monitor is records-only by definition; it never trades
  limits:
    max_trades_per_tick: 0
    max_notional: 0
    paper_only: true
  trigger: {{SLOT.trigger}}             # how often to check, e.g. { kind: every, everyMs: 900000 }
  lifecycle:
    success: "{{SLOT.trigger_condition}}"   # the condition that fires the handoff and ends the watch
    invalidation: "{{SLOT.invalidation}}"
    max_days: {{SLOT.max_days}}
---

# {{SLOT.title}}

> **Monitor mandate — do not run this template directly.** Fill every `{{SLOT.*}}`
> and flatten into `.pi/skills/<name>/SKILL.md`.

## Intent

{{SLOT.intent}}

## What this skill is

A **records-only watch**: observe an entity/condition over time, journal only
meaningful change, and **hand off** when the trigger fires. It never trades — if a
watch should act, it hands off to a `scenario` skill instead.

## State lives in the run

Baseline and observations live in the run's `log` (and, when a trade is being
watched, in that transaction's records). **Read the baseline first** each tick
via `get_run`; compare against it, not against memory.

## What it watches for

- Trigger (fires handoff + ends the watch): {{SLOT.trigger_condition}}
- What change matters: {{SLOT.watch_for}}

## Modes

### Start  (pending → waiting)
1. Capture a baseline packet (price/levels/news/context) and write it as the
   baseline record.
2. Schedule recurring ticks (`watch_schedule`, manifest `trigger`).

### Tick  (each scheduled run)
1. Re-read the baseline.
2. Gather the current packet with allowed tools.
3. Compare. Report and journal **only meaningful change**; if nothing material
   changed, say so briefly.
4. If the trigger condition is met → **Handoff**. If `invalidation` or `max_days`
   is reached → stop with a note. Else reschedule.

### Handoff  (→ done)
1. Write a record summarizing what fired and the current packet.
2. Propose the follow-on action — typically launching the matching `scenario`
   skill with this context as its `params`. Do not place trades here.
3. Stop the schedule (`watch_schedule` stop).

## Guardrails
- Records-only: never place an order; `allowed_ledgers` is empty by design.
- Never fabricate news/prices; verify with `browser`/`indian-market-news`.
- Report only decision-relevant change — no noise every tick.
- Don't claim monitoring beyond the scheduled ticks.
