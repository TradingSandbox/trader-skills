---
name: "{{SLOT.name}}"
description: "{{SLOT.description}}"
# --- mandate manifest (read by the runner/guard; the SKILL loader ignores these keys) ---
mandate:
  kind: "review"
  persona: "{{SLOT.persona}}"
  risk_profile: "{{SLOT.risk_profile}}"
  office_tool: "{{SLOT.office_tool}}"
  allowed_tools: {{SLOT.allowed_tools}}
  allowed_ledgers: []                   # review is records-only; it assesses, it does not trade
  limits:
    max_trades_per_tick: 0
    max_notional: 0
    paper_only: true
  trigger: {{SLOT.trigger}}             # null for one-off, or a schedule for EOD/weekly reviews
  lifecycle:
    success: null
    invalidation: null
    max_days: null
---

# {{SLOT.title}}

> **Review mandate — do not run this template directly.** Fill every `{{SLOT.*}}`
> and flatten into `.pi/skills/<name>/SKILL.md`.

## Intent

{{SLOT.intent}}

## What this skill is

A **records-only assessment**: read existing state — the book, a transaction's
lifecycle, or a performance window — judge it against objectives and risk, and
write a structured review. It surfaces action items but does **not** act; acting
is a separate `scenario`/`routine` handoff.

## What it reviews

{{SLOT.review_target}}   <!-- e.g. "open positions in the fund book", "transaction 42's lifecycle", "this week's realized P&L" -->

## Modes

### Run  (each invocation)
1. Read the target state via `{{SLOT.office_tool}}` — positions, ledger rows,
   transaction records, realized/unrealized P&L as relevant.
2. Assess against: {{SLOT.criteria}}  <!-- e.g. mandate fit, risk limits, thesis drift, win/loss patterns -->
3. Write a structured review record: what stands out, what's off-track, concrete
   action items (each tagged with the skill that should act on it).
4. If the manifest `trigger` is set, reschedule the next review; otherwise mark
   the run `done`.

## Guardrails
- Records-only: never place or modify an order; `allowed_ledgers` is empty by design.
- Ground every claim in the state you read — do not infer P&L or positions from memory.
- Separate observation from recommendation; recommendations are handoffs, not actions.
- Keep it decision-oriented: lead with what to change.
