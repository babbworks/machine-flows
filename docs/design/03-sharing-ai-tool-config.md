# 03: Sharing AI-tool configuration across machines

A design for keeping instructions, skills, rules, settings and memory for several AI coding tools the same on several computers, using one synced folder plus git. It is the generic version of a working design; findings about tool behavior were checked in October 2026, so verify them against current vendor documentation before relying on them.

Machines here are called `laptop-a` (macOS) and `laptop-b` (Linux). `~/machine-flows` is the synced folder; replace it with your own.

## 1. What there is to share

| Item | Share how | Why |
| --- | --- | --- |
| User instructions (`CLAUDE.md`, `AGENTS.md`) | Real file in the synced folder; the tool's home directory holds a link or a two-line import | Same rules everywhere; one place to edit |
| Skills, agents, commands, rules | Real files in the synced folder, one link per item | Tools read them from fixed home-directory locations |
| Settings (`settings.json`, `config.toml`) | **Generate** from a shared fragment, a per-machine fragment and local keys | They mix portable keys with machine-specific ones (plugins, paths, identity) |
| Memory the tool writes | Share per project, as a pilot, through the tool's memory-directory setting | Machine-local by default |
| Transcripts and sessions | Copy-only archive, never a live mirror | Retention sweeps delete them; a mirror would spread the deletions |
| Credentials, caches, SQLite databases | Never | Leak risk and corruption |

## 2. Findings that shaped the design

- **Moving the config directory moves everything** (settings, history, plugins), so a "config dir" variable is not a sharing mechanism.
- **A memory-directory setting exists** and can point a project at a shared folder. At user scope it applies to every project, so set it per project in an untracked project-local settings file. A settings file that is tracked by git, or lives under a symlinked `.claude` directory, counts as repository-supplied and the setting is ignored.
- **Memory is keyed by the git repository** (all worktrees share it), else by the project folder. The folder name encodes the absolute path, so the same project has a different memory folder on each machine unless you map them.
- **Retention is silent.** The default sweep deletes transcripts and other history after about 30 days. Copy transcripts; do not mirror them.
- **Imports:** `@~/path` works in the instructions file, a few hops deep; user-scope imports need no approval.
- **Symlinks:** personal skill folders and rules may be symlinks; the tool's Edit and Write refuse to write through a symlinked instructions file; some desktop surfaces skip a symlinked user instructions file. Point instructions at real paths where you can.
- **Codex** reads global instructions from `AGENTS.md` (32 KiB limit, no import), user skills from `~/.agents/skills`, and has no config include: merge shared settings into `config.toml` with a script, or use a profile file.
- **The sync tool syncs a symlink as a link** and does not follow it. Real files must live inside the synced folder; links live in the tool directories, outside it.

## 3. A staging test worth repeating

Before trusting links, test with a throwaway home and config directory (no login, no real credentials): make `settings.json`, the instructions file and the global state file symlinks to real files elsewhere, then let the tool write them (`mcp add`, `plugin disable`).

Result in this design's test: the symlinks survived. One write replaced the target with a new file; another updated it in place and reordered keys. The tool also added a machine identifier to its global state file, which proves that file must never be shared. Interactive writes (`/config`, `/model`) need a logged-in test.

## 4. Options compared

| | A. Sync each tool's home directory | **B. Real files in the synced folder, links from tool directories** | C. Generate or copy with an apply script |
| --- | --- | --- | --- |
| Extra synced folders | Two or more per machine | None | None |
| Leak risk | High: one wrong ignore line exposes credentials and sessions | Low: only what you put there | Low |
| Tool churn inside the synced root | High (rewrites, databases, locks) | Low | None, but needs a re-run after edits |
| Per-machine differences | Hard | Easy: a machine include plus local files | Easy |
| New laptop | Re-create ignores | A link script | Run the apply script |

Recommended: **B** for instructions, skills, agents, commands and rules; **C** for settings; the memory-directory setting for memory. Avoid **A**.

## 5. Target tree

```text
~/machine-flows/
  ai-authored/
    claude/   CLAUDE.md, machines/<host>.md, skills/, agents/, commands/, rules/
    codex/    AGENTS.md, rules/shared.rules
    shared/   skills/<skill>/SKILL.md          (usable by both tools)
  service-settings/   claude/settings.shared.json, settings.<host>.json, codex/config.shared.toml
  service-memories/   project-map.tsv, claude/projects/<id>/memory/, archive/<host>/...
  scripts/            ai-link.sh (create and verify links; dry run by default)
                      ai-settings-merge.sh (shared + host + local -> settings; dry run by default)
```

Links each machine creates, outside the synced folder:

| Tool path | Points to | How |
| --- | --- | --- |
| `~/.claude/CLAUDE.md` | `ai-authored/claude/CLAUDE.md` | link (or copy where a surface refuses links) |
| `~/.claude/CLAUDE.machine.md` | `ai-authored/claude/machines/<host>.md` | link; each machine links one |
| `~/.claude/skills/<name>` | `ai-authored/.../skills/<name>` | one link per skill; leave tool-managed folders alone |
| `~/.claude/rules/` | `ai-authored/claude/rules/` | directory link; test that rules load without an approval prompt |
| `~/.claude/settings.json` | generated | merge script writes a real file |
| `~/.codex/AGENTS.md` | `ai-authored/codex/AGENTS.md` | link |
| `~/.agents/skills/<name>` | `ai-authored/shared/skills/<name>` | link |
| `<project>/.claude/settings.local.json` | sets the memory directory to the shared folder | written by a script that merges keys and refuses if the file is tracked or `.claude` is a link |

Add a safety net to the sync ignore file so credentials never travel even if someone drops them in: `**/.credentials.json`, `**/auth.json`, `**/*.sqlite*`, and the transcript archive until you decide to sync it.

## 6. Mapping projects across machines

Because the encoded folder name comes from the absolute path, the same project has different names on each machine, and you usually cannot make the paths match. Keep a map:

```text
id        git_remote            laptop_a_path               laptop_b_path             claude_folder_a   claude_folder_b   memory_seed
my-app    owner/my-app          <laptop-a path to my-app>   <laptop-b path to my-app> <encoded path a>  <encoded path b>  laptop-a
```

Record both spellings when one path goes through a symlink (the tool may use the resolved path). Match projects by git remote, instruction-file hashes and names; treat memory folders whose project path no longer exists as archive material, not live memory.

## 7. How memory and transcripts sync

**Memory (phase 1).** One live directory per project id. Each machine points its project at it. Seed from whichever side has content; if both do, merge the index by hand and keep both topic files. Rules: wait for the sync tool to report up to date before switching machines; a memory edit reaches a new session on the other machine only; treat `*.sync-conflict-*` files as a failed check. Fallback if the setting misbehaves: link the tool's per-project memory directory to the shared one (a directory link, so files are created inside the target).

**Transcripts (phase 2, later).** They hold absolute paths and are listed only under the local folder name, so moving them does not give a working `/resume`. Use SSH and tmux for live continuity. Archive instead: each machine copies its own new transcripts to a per-host folder from a timer, without deleting, so the archive outlives the retention sweep. Decide whether it belongs in the main folder (versioning doubles it) or a separate synced folder with versioning off, and consider raising the retention period.

## 8. Staged rollout with acceptance tests

1. **Scaffold** the folders and READMEs. Accept: they appear on the other machine; no conflict files.
2. **Snapshot** authored files and memory directories on each machine (credentials excluded) to an unsynced folder. Accept: the archive listing matches the inventories.
3. **Instructions:** shared file plus per-machine includes; link on one machine, verify with a headless run that quotes both, then the other. Accept: the context view lists both files; an edit on one machine shows in a new session on the other.
4. **Skills, agents, commands, rules:** start with one test skill. Accept: both tools list it.
5. **Settings:** split into shared, host and local keys; write the merge script (dry run). Accept: the dry run reproduces each current file exactly; a `/model` change is detected as drift.
6. **Memory pilot** on one project. Accept: a new session answers from the seeded memory; a fact saved on one machine is recalled on the other, and the reverse; no conflict files; the old per-project memory is untouched.
7. **Expand memory** to more projects; archive orphaned memory with a copy first.
8. **Transcript archive** after the open decisions below.
9. **Retire** any old per-directory sync folders once the above pass.

## 9. Open decisions to settle early

- Approve option B + C and retire any whole-directory sync of tool homes.
- Which project is the memory pilot; whether clones of one repository share a memory.
- Orphaned memory: archive all, import some, or drop.
- Transcripts: inside the main folder, a separate synced folder, or not at all; retention period.
- Whether a desktop surface that refuses symlinks needs a copy instead of a link.
