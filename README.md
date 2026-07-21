# trader-skills

A community library of **ready-to-run trading skills** for
[tradecli](https://github.com/TradingSandbox/TradingSandbox).

Each skill is a concrete, fully-resolved **mandate** — a goal-directed loop
(thesis → execute → monitor → review → close) that reuses TradeCLI's current
tools to run a scenario, hypothesis backtest, routine, or review. A manual
invocation runs one tick. Trigger frontmatter is policy metadata; it does not
silently install a scheduler or keep a skill alive while TradeCLI is offline.

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

## Use in TradeCLI

Current TradeCLI releases bootstrap a compatible Pi package on first run and
keep it in Pi's normal package settings, so install, update, remove, and reload
remain dynamic. A matching embedded snapshot is used when that first install is
offline or unavailable, and by packaged binaries as the recovery fallback.
TradeCLI validates the package's compatibility contract and safety manifests
before allowing its same-name workflows to become authoritative.

TradeCLI exposes only the skills compatible with the active India Hedge Fund
role (`ceo`, `investor`, or `trader`). All package skills are present in the
release; role filtering prevents a research pane from receiving an execution
mandate or a trader pane from impersonating the CEO.

Invoke a compatible skill by name:

```text
/skill:backtest-strategy-lab
```

Pass the task after the command when useful:

```text
/skill:backtest-strategy-lab Backtest time-series momentum on NSE:RELIANCE using the daily timeframe.
```

The same command path is used by the interactive TUI, RPC/Operator sessions,
and delegated employee tasks. TradeCLI preserves the mandate manifest during
expansion and intersects its `allowed_tools` and entry limits with the active
persona's already-restricted tool surface.

### Standalone Pi installation

For a Pi environment that does not ship this TradeCLI release, install the
package manually:

Run the installation from inside the tradecli app using Pi shell mode:

```text
!!pi install git:github.com/TradingSandbox/trader-skills
```

Reload Pi resources so the newly installed skills become available:

```text
/reload
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

Pi installs the package globally by default. Standalone Pi does not provide
TradeCLI's persona filter, AITradingOffice context, or manifest dispatch guard;
the skill still cannot use tools the host has not exposed.

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
| `routine` | policy metadata / manual tick | habitual | optional | Setup → Tick → persist next state |
| `monitor` | time / condition | until trigger or stop | none | Start → Tick → Handoff |
| `review` | schedule / manual | single or recurring | none | Read → Assess → Record |

## Hedge fund automation suite

The hedge-fund office skills are designed to run together from AITradingOffice:

- `/skill:hedge-fund-ceo-daily-office` is the entry point. It reads fund cash,
  P&L, positions, employees, and recent records; allocates daily paper budgets;
  delegates or consumes `/skill:screener-conviction-workflow` as the shared
  candidate-generation layer; routes individual desks; and records the daily
  office plan.
- Desk routines handle the specialized work: Short MIS, Long MIS, Long MTF,
  Investor, Index Options, Stock Options, Commodity, and Stock Futures. Equity
  and F&O desks start from the shared screener output, then use their own tools
  for validation, sizing, execution, and journaling.
- `/skill:hedge-fund-skill-improvement-review` reads the day's office records,
  trades, rejected ideas, and P&L to produce concrete rule and skill-improvement
  actions for the next run.

All hedge-fund automation skills are paper-only. Trading desks may enter only
through `strategy` → exact returned payload → `enter_trade`, and settle through
`exit_trade`; `aitradingoffice_hf` is read/roster/research-record state, not a
transaction-write bypass. CEO and Investor skills coordinate and research but
do not place trades. Current option execution is restricted to single-leg long
calls/puts, commodity execution to supported futures, and MTF-style research is
booked as unlevered CNC paper equity because no margin model exists.

The suite lives under `skills/hedge-fund/`. Skill names stay unchanged, so
existing commands such as `/skill:hedge-fund-commodity-desk` still work.

## Contributing a skill

1. Pick a loop shape from [`templates/`](templates/) and fill in every
   `{{SLOT.*}}` to produce a complete, resolved skill (no placeholders left).
   Add it at `skills/<your-skill>/SKILL.md`. The frontmatter must declare
   `name`, `description`, and a `mandate` block with `kind`, `persona`, `roles`,
   `allowed_tools`, `allowed_ledgers`, and `limits`. Select the smallest useful
   tool set from the generated `.tradecli-tools/TOOLS.md`; do not copy the whole
   catalog into a mandate.
2. Append an entry to `index.json` (`name`, `kind`, `summary`, `author`, `path`).
3. Run `npm run tools:check` to verify that every concrete skill names tools
   present in the generated TradeCLI catalog.
4. Open a PR.

Keep skills **paper-first** (`paper_only: true`) and conservatively limited
unless there's a clear reason otherwise.
