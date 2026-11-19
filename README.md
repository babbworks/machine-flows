# machine-flows

machine-flows is a folder that several computers share through [Syncthing](https://syncthing.net/), and that AI tools (Claude Code, Codex, Cursor, ChatGPT and others) read and write. It holds the things that should be the same everywhere: shared AI instructions, plans, prompt templates, MCP server definitions, inventories of each machine, and a companion CLI called [`mach`](https://github.com/babbworks/mach).

This repository is the **public, generic part** of machine-flows. The working folder can also hold private inventories, settings, notes and project plans; those stay in the personal copy and are represented by safe stubs in [`examples/`](examples/). The companion CLI is [`babbworks/mach`](https://github.com/babbworks/mach).

## The idea in one minute

- **One folder, many computers, many AI tools.** The things that should be the same everywhere (instructions, plans, templates, tool-server definitions, inventories) live in one folder that file sync keeps identical on every machine.
- **Sync for sameness, git for review.** File sync moves working files between your own machines all day with no review. Git records and publishes changes that someone has looked at. `.git` is never synced; secrets and project code never travel by sync.
- **Three tiers, each a filtered projection of the one above.** A personal copy holds everything; an organization-private repository holds the flows a team shares; a public repository holds only the generic, reusable parts. Content moves downward only, through a manifest per tier and a scan, never by hand copying (`mach publish`).
- **Preview, then apply.** Anything that changes state shows what it would do first and acts only on an explicit flag.

The full technical breakdown is in [`ARCHITECTURE.md`](ARCHITECTURE.md).

## mach: the tool that works on this folder

`mach` is the companion command-line tool for setup, inventories, AI-tool wiring and approval-gated work runs. It is maintained separately at [`babbworks/mach`](https://github.com/babbworks/mach) (MIT), so it can be installed and updated independently of a synced workspace.

## Layout

| Folder | Purpose | In this repo |
| --- | --- | --- |
| `ai-authored/` | Hand-written AI instructions shared by every machine (`claude/CLAUDE.md`, one file per machine under `claude/machines/`) and `templates/` (prompt documents for `mach run`) | Only the generic templates |
| `ai-tools/` | Reference notes and comparison sheets for the AI tools in use | Local only |
| `drafts/` | Drafts-app action packs and automation notes | Local only |
| `examples/` | Placeholder versions of every local-only area | Yes |
| `files/` | Inbox, lessons knowledge base and working files | Local only (see `examples/service-memories/`) |
| `inventory/<host>/` | Generated description of each machine: system, packages, tools, services, scheduled jobs | Local only (see `examples/inventory/`) |
| `mach/` | The `mach` CLI, installed separately | Not part of this workspace; keep its checkout outside the synced folder (see `SETUP.md`) |
| `mcp/` | One manifest (`servers.yaml`) for the MCP servers every tool should have, plus helpers | Only `helpers/github-auth-headers.sh` (see `examples/mcp/`) |
| `plans/` | Living planning documents, one topic per file | Local only (convention in `examples/plans/README.md`) |
| `scripts/` | Inventory and MCP helper scripts (CLI names: `mach names`) | None |
| `service-memories/` | Lessons and per-service memory (the lessons index, reviews) | Local only (see `examples/service-memories/`) |
| `service-settings/` | Shared settings per service (for example `mach/config.toml`) | Local only (see `examples/service-settings/`) |
| `subsystems/` | One TOML manifest per company or project: units, roles, repos, service profiles | Only `_template/` (see `examples/subsystem.toml`) |
| `models.toml`, `model_router/` | Capability ladder for completions: task class, provider fallback, daily USD budget | Yes |

"Local only" folders are excluded by the whitelist in `.gitignore`: it starts with `/*` and un-ignores generic paths one by one. A new file is never committed by accident; to publish something you add a line for it.

## Ideas that hold the folder together

**One source of truth per fact.** Instructions live in `ai-authored/`; each tool's own config file is a two-line import of it. MCP servers are defined once in `mcp/servers.yaml` and applied per machine. Credentials never enter the folder: only file names, environment-variable names and helper-script paths do.

**Preview, then apply.** Scripts and `mach` commands that change anything print a preview first and need `--apply`. Changes back up what they overwrite and print a rollback line.

**Sheet sets.** Tracking tables are a folder holding an `index.csv` plus the CSV files it lists. The CSVs are the source of truth; the Excel workbook is generated from them (`mach sheets build <dir>`, `mach sheets new <dir>` to start one) and never edited by hand. `mach sheets lint` checks that every CSV is indexed, row counts are right and the workbook is current.

**Plans.** `plans/` holds one living document per topic, numbered `NN-short-name.md`. Each has a `Status:` line, short prose, then checklists. Boxes are ticked only when verified, facts carry a date and a source, and a plan is archived to `plans/archived/` only when every box is ticked and the owner agrees. Plan `00` is the master plan and is never archived. The full convention is in [`examples/plans/README.md`](examples/plans/README.md).

**Tiers.** `mach publish --tier NAME` selects the files a tier may contain from a manifest, scans them (secrets always; machine paths, ticket keys and denied names on request) and writes an export folder outside the synced tree. It refuses while a finding is open and never runs git: you review the folder, then commit and push yourself.

**Worktrees.** Feature and agent work happens in git worktrees under one root, named `<repo>--<ticket>-<slug>`, never in extra clones. Primary checkouts stay on the main branch.

**Subsystems.** A subsystem names a company or project and its units (pillars, services, products, pipelines). A TOML manifest per subsystem records who leads each unit (an AI tool, a Grok Bot or the owner) so `mach <subsystem> <unit> "prompt"` can route a prompt. Start from [`examples/subsystem.toml`](examples/subsystem.toml).

**Model router.** Completions go through `model_router` and `models.toml`. A caller passes a task class (`triage`, `extraction`, `synthesis`, `reasoning`, `chat`, `code`); the router chooses the provider. Model ids live in that file, not in code. Daily USD caps are per subsystem; local and free tiers do not count. Preview a choice with `python3 -m model_router --class triage --channel cli --subsystem demo --dry-run`.

## Where to start

- Design notes (generic write-ups of the reasoning behind this folder): [`docs/design/INDEX.csv`](docs/design/INDEX.csv) lists them in one line each. Start with `03` (sharing AI-tool config) and `09` (memory and lessons).

- Using this with an AI tool: [`SETUP.md`](SETUP.md) and [`AGENTS.md`](AGENTS.md).
- Building your own copy: create the folder, share it with Syncthing, copy `examples/` stubs to the real locations and fill them in.
