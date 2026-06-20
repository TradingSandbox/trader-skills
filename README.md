# trader-skills

A community library of **ready-to-run trading skills** for
[tradecli](https://github.com/TradingSandbox/TradingSandbox).

Each skill is a concrete, fully-resolved **mandate** — a goal-directed loop
(thesis → execute → monitor → review → close) that reuses tradecli's existing
tools to run a scenario, a hypothesis backtest, a routine, or a monitor. Browse
the library, add one to your tradecli, and run it — it loops on its own schedule
and you reuse it across sessions.

This repo is the **single source of truth** for both the loop **templates** and
the **concrete skills** built from them. tradecli ships nothing of its own — it
fetches everything from here.

```
index.json              # catalog of every concrete skill
templates/              # the loop shapes (authoring reference)
  <kind>/SKILL.md       # oneshot | scenario | routine | monitor | review
skills/                 # concrete, ready-to-run skills
  <skill-name>/
    SKILL.md            # the complete, runnable skill (frontmatter + loop)
```

## How a skill loads into tradecli

1. tradecli reads `index.json` to list what's available.
2. You pick one to install.
3. tradecli stores it in your AITradingOffice as a reusable skill
   (`create_skill`) — the stored row is the source of truth.
4. You kick off a **run**; the skill loops on its `trigger` schedule and you can
   re-run it any time.

## Skill types

Every skill declares a `kind` in its frontmatter — the loop shape it follows:

| Kind | Trigger | Lifecycle | Ledger | Loop |
|---|---|---|---|---|
| `oneshot` | manual | single run | optional (≤1 trade) | Run → artifact → stop |
| `scenario` | event / kickoff | episodic, self-terminating | yes | Open → Tick(days) → Close |
| `routine` | cron | habitual, indefinite | optional | Setup → Tick(repeat) → Stop-on-request |
| `monitor` | time / condition | until trigger or stop | none | Start → Tick → Handoff |
| `review` | schedule / manual | single or recurring | none | Read → Assess → Record |

## Contributing a skill

1. Pick a loop shape from [`templates/`](templates/) and fill in every
   `{{SLOT.*}}` to produce a complete, resolved skill (no placeholders left).
   Add it at `skills/<your-skill>/SKILL.md`. The frontmatter must declare
   `name`, `description`, and a `mandate` block with `kind`, `persona`,
   `allowed_ledgers`, and `limits`.
2. Append an entry to `index.json` (`name`, `kind`, `summary`, `author`, `path`).
3. Open a PR.

Keep skills **paper-first** (`paper_only: true`) and conservatively limited
unless there's a clear reason otherwise.
