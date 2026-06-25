# trader-skills

A community library of **ready-to-run trading skills** for
[tradecli](https://github.com/TradingSandbox/TradingSandbox).

Each skill is a concrete, fully-resolved **mandate** — a goal-directed loop
(thesis → execute → monitor → review → close) that reuses tradecli's existing
tools to run a scenario, a hypothesis backtest, a routine, or a monitor. Browse
the library, add one to your tradecli, and run it — it loops on its own schedule
and you reuse it across sessions.

This repo is the **single source of truth** for both the loop **templates** and
the **concrete skills** built from them. It is distributed as a Pi package, so
tradecli can discover and load its skills directly.

```
index.json              # catalog of every concrete skill
.tradecli-tools/        # generated local tool catalog (gitignored)
  TOOLS.md              # human-readable TradeCLI tool catalog
  tradecli-tools.json   # machine-readable TradeCLI tool catalog
templates/              # the loop shapes (authoring reference)
  <kind>/SKILL.md       # oneshot | scenario | routine | monitor | review
skills/                 # concrete, ready-to-run skills
  <skill-name>/
    SKILL.md            # the complete, runnable skill (frontmatter + loop)
```

## Install and use in tradecli

Run the installation from inside the tradecli app using Pi shell mode:

```text
!!pi install git:github.com/TradingSandbox/trader-skills
```

Reload Pi resources so the newly installed skills become available:

```text
/reload
```

Invoke a skill by name:

```text
/skill:backtest-strategy-lab
```

You can pass the task directly after the skill command:

```text
/skill:backtest-strategy-lab Backtest time-series momentum on NSE:RELIANCE using the daily timeframe.
```

`!!` runs the installation command without adding its terminal output to the
conversation. The same installation command can also be run from a normal
terminal without the `!!` prefix.

### Update or remove

From inside tradecli:

```text
!!pi update git:github.com/TradingSandbox/trader-skills
/reload
```

To uninstall:

```text
!!pi remove git:github.com/TradingSandbox/trader-skills
/reload
```

Pi installs the package globally by default, making its skills available across
tradecli projects and personas.

## TradeCLI tool catalog

Skill and workflow authors can generate the complete local catalog and query it:

```text
npm run tools:refresh
npm run tools
npm run tools -- tradingview
npm run tools -- --search strategy
npm run tools:check
```

The generated files live in `.tradecli-tools/` and are ignored by Git. Use the
exact names from `.tradecli-tools/TOOLS.md` or
`.tradecli-tools/tradecli-tools.json` when filling a skill's
`mandate.allowed_tools`.

Tools such as `aitradingoffice_hf`, `aitradingoffice_pms`,
`aitradingoffice_profile`, and `aitradingoffice_workflows` are multi-action
wrappers. The catalog lists every accepted `action` beneath its wrapper. In
`allowed_tools`, grant the wrapper name; inside the workflow, call it with the
documented action, for example `aitradingoffice_hf` with
`action: create_record`.

The catalog is the union of tools loaded by the TradeCLI runtime. A particular
session may expose a smaller set based on its active persona, broker setup,
installed MCPs, authentication, and runtime configuration. Declaring a tool in
`allowed_tools` grants a mandate permission to use it; it does not install,
authenticate, or activate that tool.

To refresh the catalog against a local TradeCLI checkout:

```text
npm run tools:refresh
```

By default the script looks for the sibling checkout at `../TradingSandbox`.
Pass another location with `--tradecli /absolute/path/to/TradingSandbox` or the
`TRADECLI_REPO` environment variable. The refresh command starts TradeCLI's
tool-list runtime and may update TradeCLI's local cache.

## Skill types

Every skill declares a `kind` in its frontmatter — the loop shape it follows:

| Kind | Trigger | Lifecycle | Ledger | Loop |
|---|---|---|---|---|
| `oneshot` | manual | single run | optional (≤1 trade) | Run → artifact → stop |
| `scenario` | event / kickoff | episodic, self-terminating | yes | Open → Tick(days) → Close |
| `routine` | cron | habitual, indefinite | optional | Setup → Tick(repeat) → Stop-on-request |
| `monitor` | time / condition | until trigger or stop | none | Start → Tick → Handoff |
| `review` | schedule / manual | single or recurring | none | Read → Assess → Record |

## Hedge fund automation suite

The hedge-fund office skills are designed to run together from AITradingOffice:

- `/skill:hedge-fund-ceo-daily-office` is the entry point. It reads fund cash,
  P&L, positions, employees, and recent records; allocates daily paper budgets;
  runs or consumes `/skill:screener-conviction-workflow` as the shared
  candidate-generation layer; wakes/delegates desks; and records the daily
  office plan.
- Desk routines handle the specialized work: Short MIS, Long MIS, Long MTF,
  Investor, Index Options, Stock Options, Commodity, and Stock Futures. Equity
  and F&O desks start from the shared screener output, then use their own tools
  for validation, sizing, execution, and journaling.
- `/skill:hedge-fund-skill-improvement-review` reads the day's office records,
  trades, rejected ideas, and P&L to produce concrete rule and skill-improvement
  actions for the next run.

All hedge-fund automation skills are paper-first. Trading desks write to the
AITradingOffice hedge-fund paper book through `aitradingoffice_hf`; the CEO and
Investor skills coordinate and research but do not place trades.

## Contributing a skill

1. Pick a loop shape from [`templates/`](templates/) and fill in every
   `{{SLOT.*}}` to produce a complete, resolved skill (no placeholders left).
   Add it at `skills/<your-skill>/SKILL.md`. The frontmatter must declare
   `name`, `description`, and a `mandate` block with `kind`, `persona`,
   `allowed_tools`, `allowed_ledgers`, and `limits`. Select the smallest useful
   tool set from the generated `.tradecli-tools/TOOLS.md`; do not copy the whole
   catalog into a mandate.
2. Append an entry to `index.json` (`name`, `kind`, `summary`, `author`, `path`).
3. Run `npm run tools:check` to verify that every concrete skill names tools
   present in the generated TradeCLI catalog.
4. Open a PR.

Keep skills **paper-first** (`paper_only: true`) and conservatively limited
unless there's a clear reason otherwise.
