# 06: Covering several AI tools from one source

A design for giving every AI tool you use (here: Claude, Codex, Cursor, Kiro and a chat assistant with local execution) the same rules, skills and MCP servers on every computer, from one place. "Fully covered" means a tool gets all three from the shared folder, not that every tool is configured identically. Tool behaviors were checked in October 2026; verify them before relying on them.

Machines: `laptop-a` (macOS) and `laptop-b` (Linux). `~/machine-flows` is the synced folder. Legend for coverage tables: wired from the shared folder / partial or native only / nothing yet / not applicable.

## 1. One source of truth

Everything shared is written **once** under the synced folder. Each tool gets it through the lightest mechanism it supports. There are **no symlinks inside the synced folder** (the sync tool would sync the links themselves); links are made *from* a tool's own directory *into* the folder, per machine.

```text
~/machine-flows/
  ai-authored/
    AGENTS.md                    tool-neutral shared rules: canonical for all tools
    claude/CLAUDE.md             Claude extras; first line imports ../AGENTS.md
    claude/machines/<host>.md    per-machine overrides, appended by every renderer, not just Claude's
    codex/extra.md               optional Codex-only addendum
    cursor/user-rules.md         GENERATED: text to paste into Cursor's account-wide User Rules
    kiro/steering/*.md           optional Kiro-only steering
    skills/<name>/SKILL.md       shared skills (name, description, body)
    skills/<name>/agents/openai.yaml   optional Codex extras
  mcp/servers.yaml               MCP manifest with a target per tool
  service-settings/<tool>/       shared settings fragments
  scripts/mcp-apply.sh           MCP renderer (dry run by default)
  scripts/ai-apply.sh            renders rules and links skills (dry run by default)
```

**Commands are skills.** All of these tools now treat slash commands as skills. A shared "command" is a skill that the model may not invoke by itself: `disable-model-invocation: true` for Claude and Cursor, `policy.allow_implicit_invocation: false` in `agents/openai.yaml` for Codex. There is no separate commands directory.

## 2. Wiring per tool

| Aspect | Claude | Codex | Cursor | Kiro | Chat assistant (account-side) |
| --- | --- | --- | --- | --- | --- |
| **Rules** | `~/.claude/CLAUDE.md` is a real file containing an `@` import of the shared Claude file (up to four hops) | **Generated** `~/.codex/AGENTS.md` = shared rules + machine file + extras, with a header carrying the source hash ("generated, edit the shared source"); Codex has no import | No global rules file: User Rules live in the account, so paste a generated text; project rules in `.cursor/rules/*.mdc` | Steering files under `~/.kiro/steering/`, which can include other files | Account skill whose body is generated from the shared rules |
| **Skills** | one symlink per skill into `~/.claude/skills/` (leave the tool-synced folder alone) | one symlink per skill into `~/.agents/skills/` | reads `~/.agents/skills` and `~/.claude/skills`: nothing extra | link or copy into `~/.kiro/skills/` (test links first) | mirror chosen skills into the account library, recorded with source hash and date |
| **MCP** | render with `claude mcp add-json --scope user` (writes the tool's global state file; never edit it directly) | managed `[mcp_servers.*]` block in `config.toml` | JSON merge into `~/.cursor/mcp.json` (`${env:NAME}`); one file for the IDE **and** the CLI | JSON merge into `~/.kiro/settings/mcp.json` (`${NAME}`) plus approved env-var names | account connectors; nothing local |
| **Settings** | generated from a shared fragment plus a host fragment (design 03) | managed block of shared keys (model, reasoning effort, features); trust entries stay local | `jq` merge of a shared settings fragment into the user `settings.json` | same idea | n/a |
| **Memory** | optional per-project pilot (design 03) | on per machine, never live-synced; collect reviewed lessons (design 09) | none | steering is the memory | server-side |
| **Transcripts** | copy-only archive later | copy-only archive later | keep local | keep local | server-side |

## 3. Per-machine overrides

- **Text:** one per-machine rules file per host, appended by every renderer, so all tools see the same machine notes. Mind the import depth: if the Claude chain is already three of four hops, a machine file may import at most one more level.
- **MCP:** `servers.yaml` has `platforms:` and `machines:` per server (for example a Mac-only notes app).
- **Settings:** `service-settings/<tool>/<host>.json` overlays.
- **Paths:** use `~/` and `$HOME` everywhere; scripts resolve OS-specific app-settings locations.

## 4. What always stays local (never synced, never generated)

| Tool | Local only |
| --- | --- |
| Claude | credentials, the global state file (MCP state and OAuth), project transcripts (until a pilot), sessions, plugin caches, locks |
| Codex | `auth.json`, installation id, SQLite databases, live memories, sessions, the auto-appended approvals file, trust entries, plugin caches |
| Cursor | app state databases and workspace storage; per-project chats, plans, worktrees, sandbox policies, extensions and plugin caches; CLI state |
| Kiro | app state, sessions, installed powers, OAuth tokens, CLI history |
| Chat assistant | everything under its local-execution daemon directory (credentials, connection files): never read or copied |
| All | API keys and tokens live only in the shell environment or a keychain; manifests carry variable **names** |

## 5. Rollout order

**Foundation, then Claude (finish), Codex, Cursor, the chat assistant, Kiro.**

- A foundation step comes first: every tool after Claude needs the tool-neutral rules file and the `skills/` folder.
- Codex and Cursor stay back to back because they share `~/.agents/skills`: the skills step for one is the skills step for the other.
- The account-side assistant moves ahead of the least-used IDE: one skill created in a chat covers every device, its MCP is already covered by connectors, and it has no local config to break, so it is high value per minute.
- The least-used tool goes last: its config is empty, and it has the most open questions (copy versus link for skills, the approved env vars step), so it benefits from a renderer already proven on the others.

Per tool, a stage is: render rules, link skills, render MCP, merge settings, verify with the tool's own context view, tick the checklist with a date and machine.

## 6. The MCP renderer

`mcp/servers.yaml` has a `tools:` block (file, format and env syntax per tool) and a target per tool on every server. Methods: `user` (render into the tool's file), `connector` (an account connector), `chat` (ask the assistant to add it), or `none` with a reason. The renderer previews, per tool, the fragment that would be merged: `${NAME}` becomes `${env:NAME}` for Cursor and `${NAME}` for Kiro, and Kiro's approved-env-vars list is updated. It prints only variable names, never values. Add an `--apply` mode one tool at a time, Claude first, and keep a dated backup of every file it will change.

## 7. Explicit CLI names and installer behavior

Some installers recreate a bare `agent` command (Cursor's installer replaces `~/.local/bin/agent` and the named command; a Grok CLI installer links `agent` in its own bin directory and may add that directory to your shell rc). Two tools claiming the same short name is a trap for scripts and agents.

- Decide one explicit name per CLI and have scripts call only that name. Check for the bare name after any install or update with a small script that **reports** first and fixes only symlinks in user-owned bin directories, recording the old link targets so the change is reversible.
- Mind PATH order: a directory earlier on PATH can re-expose the bare name even after you remove it elsewhere.
- Where a self-updating tool keeps recreating its own bare name, you may decide to accept that one name and keep the explicit name as an alias; record that decision in the rules file.

## 8. Open questions to settle early

- Whether the Cursor CLI honors a user-level rules file as well as project rules (test with a marker word before relying on it).
- Whether shared skills should be links or copies for tools that do not document symlink support.
- Where each tool's memory fits the lessons design (design 09), and which tools need the external-context protection.
- Whether to mirror skills into an account-side assistant automatically or by hand.
