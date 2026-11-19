# 05: Building blocks: what the documentation says, and what to do about it

Research behind a multi-machine, multi-AI-tool working folder: the sync tool, each AI tool's config layout, MCP servers, dotfile managers, secrets, remote handoff, onboarding and inventories. Findings come from vendor documentation read in October 2026; verify them before relying on them, because tools change. Machines are `laptop-a` (macOS) and `laptop-b` (Linux, always on). `~/machine-flows` is the synced folder.

**Bottom line:** keep the architecture "real files in one synced folder, a thin apply layer per machine". Five adjustments: (1) tools load synced files through imports and links, instead of syncing the tools' home folders; (2) one canonical MCP manifest rendered into each tool's format, with secrets only as environment-variable *names*; (3) an inbox for hand-offs where each file has exactly one writer; (4) a doctor script that checks conflicts, secrets and version skew; (5) a desired-state file next to every generated inventory.

---

## 1. The sync tool (Syncthing) for config and many devices

| Topic | Finding |
| --- | --- |
| Ignore file | `.stignore` is never synced, but it can `#include` a synced file (the "global ignore" pattern). An `#include`d file that does not exist is an **error**, so every machine needs it before the ignore applies. |
| Matching | First match wins, so put `!` exceptions *before* broader patterns. `/pattern` anchors to the root; `**` crosses directories; `(?d)` lets the tool delete OS junk that blocks a folder delete; `(?i)` is case-insensitive (always on for macOS). A non-rooted `!pattern` forces a walk of ignored folders; a rooted one does not. |
| Timing | Changes are batched about 10 s; deletions wait about a minute more. Keep hourly full scans on because the watcher can miss changes. |
| Conflicts | Edits on two devices produce `*.sync-conflict-<date>-<time>-<device>.*`; the file with the older mtime becomes the copy. Conflict copies are normal files and **sync everywhere**. |
| Case | Names differing only in case cannot coexist on macOS; the tool reports a case conflict instead of overwriting. |
| What travels | Contents and mtimes; permission bits where supported; **symlinks as links, never followed**; not hard links or directory mtimes. Leave `ignorePerms` off between macOS and Linux so the executable bit on scripts survives. |
| Versioning | Configured per folder **per device**, and archives only files changed by *remote* devices. Staggered versioning keeps 30 s / 1 h / 1 day / 1 week buckets up to `maxAge`; versions go to `.stversions/` by default. |
| Folder types | `sendonly` and `receiveonly` suit one-writer folders (a machine's own inventory). |
| Introducer | Mark one always-on device as introducer; never make two devices introducers of each other. |
| Linux limit | The default of 8192 inotify watches may be too low for big trees. |
| On-demand clouds | iCloud-style "evict old files" folders confuse the tool (placeholders look like changes). Keep synced folders **outside** them: in `$HOME`, not `~/Desktop` or `~/Documents` on macOS. |

**Practices.** One writer per file: per-machine data in per-machine names (`inventory/<host>/`, `inbox/*-<host>-*.md`, logs); shared files edited on one machine at a time, waiting for "Up to Date" before switching. A conflict sweep (`find ~/machine-flows -name '*.sync-conflict-*'`) in the doctor script, and agents never leave conflict copies unresolved. Staggered versioning, 30-90 days, on **every** device. Never sync absolute symlinks into tool homes. Auto-upgrade off; upgrade all devices together.

A secrets guard for the global ignore file (order matters; keep `!*.pub` above the key patterns). Check it on a staging copy with the tool's "ignored files" view, and check that no authored file matches `credentials*.json`:

```text
!*.pub
(?i)*.env
(?i).env.*
*.pem
*.p12
id_rsa*
id_ed25519*
**/credentials*.json
**/logs
*.log
```

Leave `*.key` out on purpose: presentation software uses that extension.

---

## 2. Portable configuration for AI tools

| Item | Claude Code | Codex | Cursor | Cross-tool standard |
| --- | --- | --- | --- | --- |
| Global instructions | `~/.claude/CLAUDE.md`, supports `@path` imports (user-scope imports load without approval) | `~/.codex/AGENTS.md` (or `AGENTS.override.md`) | User Rules in settings (UI, not a file) | `AGENTS.md` |
| Project instructions | `CLAUDE.md`; reads `AGENTS.md` only when no `CLAUDE.md` exists, or via an `@AGENTS.md` import | `AGENTS.md` from repo root to cwd, 32 KiB cap | `.cursor/rules/*.mdc` and nested `AGENTS.md` | agents.md |
| Skills | `~/.claude/skills/<name>/SKILL.md`; a skill folder may be a symlink | `~/.agents/skills`, follows symlinks | listed as supported | Agent Skills: `SKILL.md` with `name` and `description` |
| Settings | `~/.claude/settings.json` plus project and local files; watched and reloaded | `~/.codex/config.toml` (project file only in trusted projects) | app settings | none |
| Machine and auth state | `~/.claude.json` (sign-in, MCP servers, trust): never sync | runtime and auth files | app storage | none |
| Memory | Auto memory per project, machine-local by default; a setting can point it elsewhere | none | none | none |
| Relocate home | `CLAUDE_CONFIG_DIR` | `CODEX_HOME` | none | none |

**Pattern.** Author once in `ai-authored/` (a canonical `AGENTS.md`; a Claude file whose first line imports it; skills in Agent Skills format so one copy serves all tools; Claude path rules; Cursor `.mdc` rules). Apply per machine with an idempotent script that defaults to a dry run: write a *real* `~/.claude/CLAUDE.md` containing only the import (an atomic rewrite by the tool cannot replace a symlink); link each skill folder into `~/.claude/skills` and `~/.agents/skills`; link or copy the Codex `AGENTS.md` (test whether Codex follows a link; if not, copy and compare hashes); merge shared settings into the real settings file with `jq`, preserving local keys, because the tool writes that file from `/config` and `/model`. Verify with the tool's own context view.

---

## 3. MCP configuration portability

| Tool | Where | Variable syntax | Secrets |
| --- | --- | --- | --- |
| Claude Code | local and user scope in `~/.claude.json`; project scope in `.mcp.json` (needs approval) | `${VAR}`, `${VAR:-default}` | local scope is meant for credentials; use variables for paths and keys |
| Codex | `[mcp_servers.<name>]` in `config.toml` | `env_vars = [...]`, `bearer_token_env_var`, `codex mcp login` for OAuth | reference variable names |
| Cursor | `~/.cursor/mcp.json` or `.cursor/mcp.json` | `${env:NAME}`, `${userHome}`, `${workspaceFolder}`, `envFile` | never hard-code |

**Recommendation.** `mcp/servers.yaml` is the canonical list (name, transport, command and args or URL, platforms, tools, required env-var **names**, OAuth yes or no). An apply script renders the entries for this machine: `claude mcp add-json --scope user` (never edit `~/.claude.json` directly), a managed block in `config.toml`, and a merge into `mcp.json`; it prints a diff first. Prefer remote HTTP servers with OAuth where vendors offer them; each machine just logs in. Pin the versions of `npx` stdio servers to avoid drift.

---

## 4. Dotfile managers next to the sync tool

| Tool | Model | Fit |
| --- | --- | --- |
| chezmoi | source dir rendered into `$HOME`; Go templates on OS and hostname; built-in age encryption and password-manager functions | the best complement when templating or secrets get painful |
| yadm | git repo with `$HOME` as work tree; alternate files per OS or class | overlaps the sync tool's role |
| bare git repo | `git --git-dir=$HOME/.cfg --work-tree=$HOME` | fine for shell rc files with history |
| GNU Stow | packages of files become a symlink farm | links are made per machine, which is good |

**Judgement.** The sync tool is for live, frequently edited shared content. A bare repo stays for shell dotfiles that want history. Adopt chezmoi later, only if per-host templates or secrets in dotfiles become a problem. Never manage the same target file with two systems.

---

## 5. Secrets

| Option | Use for |
| --- | --- |
| References, not values, in the synced tree | any config that needs a key: `op://vault/item/field` in templates, `op inject` at apply time, delete the rendered file after |
| chezmoi with a password manager or age | dotfiles that need secrets |
| OS keychain (`security`, `secret-tool`) | scripts and executors |
| Tool-native stores | logins (always per machine) |
| Machine-local env files, mode 600, outside the synced folder | interim, blocked by the ignore guard |

**Rules for agents:** never write a secret value into the synced folder; refer to secrets by env-var name or reference path; when you find one, report where, not what.

---

## 6. Live hand-off with a mesh VPN, SSH and tmux

Tailscale SSH servers run on Linux and on the open-source macOS build; the App Store macOS app is a client only. Plain OpenSSH over the mesh works everywhere. Default policy lets users SSH to their own devices in check mode (periodic re-auth). `tmux-resurrect` and `tmux-continuum` persist sessions across reboots.

**Pattern.** Enable the SSH server on the always-on machine; enable Remote Login on the Mac; `ssh <host> -t 'tmux new -A -s work'` attaches the same live session or creates it; name sessions per project; keep host aliases (hostnames only, no keys) in a synced `ssh/config.d/` file.

---

## 7. Onboarding a new machine

An idempotent script where every step checks before it acts:

1. Install the sync tool from a verified release as a user service, auto-upgrade off; check the version matches the others.
2. Pair with the introducer, accept the folder, create the local `.stignore` with `#include .stglobalignore`; check "Up to Date".
3. Run the inventory script for this OS.
4. Install desired packages (`brew bundle --file desired/Brewfile`, or apt from `desired/apt-manual.txt`); diff observed against desired.
5. Run the AI apply script; check the context view lists the files.
6. Run the MCP apply script; check `mcp list` for each tool.
7. Secrets: sign in or prompt for each machine-local env file; the doctor confirms none sit inside the synced folder.
8. Bring up the VPN and an SSH alias; check `ssh <other> true`.
9. Record the result in a plan checklist and the machine's summary file.

---

## 8. Inventory as code

| Platform | Observed | Desired |
| --- | --- | --- |
| macOS Homebrew | `brew bundle dump --file desired/Brewfile` | `brew bundle --file ...`; `brew bundle cleanup` lists extras (use it **without** `--force`) |
| Debian apt | `apt-mark showmanual > desired/apt-manual.txt` | install from the list on a new laptop; `apt-mark minimize-manual` tidies metapackages |
| Language tools | `npm ls -g --depth=0`, `pipx list --short`, `cargo install --list`, `flatpak list --app` | plain lists per host |

Keep **observed** (generated) and **desired** (hand-curated, short) side by side; the doctor reports the differences. Machines of one family can share a `_common/desired/` file.

---

## 9. Folder verdicts

| Folder | Verdict and rules |
| --- | --- |
| `ai-authored/` | Canonical `AGENTS.md`, a Claude file importing it, skills, rules, Cursor rules, commands, agents. |
| `service-settings/` | Base settings per tool, scheduler templates (launchd, systemd), SSH config fragments. **Templates and references only, no values.** |
| `mcp/` | `servers.yaml` + apply script + README. |
| `service-memories/` | Pilot memory sharing per project first; one active machine at a time; transcripts later and only if size and privacy allow. |
| `inventory/` | Observed plus `desired/` per host. |
| `scripts/` | Apply, doctor (paths, conflicts, secrets guard, versions), conflict finder, onboarding. |
| `plans/` | Numbered living documents with checklists. |
| `inbox/` | **Add**: capture and hand-off folders where each filename carries the host, so each file has one writer. |

---

## 10. Prioritized checklist

**P0, before pairing:** add the secrets guard and runtime ignores and test them on a staging copy; confirm the folder is outside cloud-sync paths; pair the machines and enable staggered versioning on both.

**P1, authored files and MCP:** create the canonical instructions and the Claude import; write the apply script (dry run default) and verify with each tool's context view; write the MCP manifest and render script, preferring OAuth remote servers and pinned `npx` versions.

**P2, hand-off and onboarding:** VPN SSH on the always-on machine, Remote Login on the Mac, tmux aliases; write the onboarding script and test it on the next new laptop; add desired-state files; upgrade runtimes that current MCP servers need.

**P3, refinement:** memory pilot for one project; a doctor script run weekly by launchd or systemd; revisit chezmoi only if templating pain grows; with a third device, make the always-on one the single introducer.

---

## Sources (fetched October 2026)

Syncthing: Ignoring Files, File Versioning, Understanding Synchronization, Introducer, FAQ, Config (docs.syncthing.net). Claude Code: Settings, Memory, Skills, MCP (code.claude.com/docs). Codex: AGENTS.md, Skills, MCP (developers.openai.com/codex). Cursor: Rules, MCP (cursor.com/docs). Standards: agents.md, agentskills.io. Dotfiles: chezmoi.io (comparison, machine differences, password managers, age). Secrets: 1Password CLI secrets in config files. Remote: Tailscale SSH, tmux-resurrect. Inventory: Homebrew Bundle, apt-mark(8).
