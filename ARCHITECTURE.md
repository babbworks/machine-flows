# machine-flows: how the idea works

A technical breakdown of the folder model behind machine-flows. For the command-line tool that runs it, see the separate `mach` outline. This document is about the structure: what the folder is, what travels where, and the rules that keep several computers and several AI tools from stepping on each other.

Status of each part is marked: **built** (in use), **designed** (agreed on paper, not implemented).

## 1. The problem

A person who works on several computers, with several AI coding tools, ends up with the same things scattered everywhere: the instructions each tool should follow, which servers each tool can call, notes about what is installed where, plans that span weeks, and prompt templates. Each tool keeps its own config in its own place, each machine drifts, and nobody can say which copy is right.

machine-flows is one folder that holds the things that should be the same everywhere, plus a small set of rules for how it changes.

## 2. The folder (built)

One directory, for example `~/machine-flows`, shared between computers with a file-sync tool (Syncthing). Everything the person authors by hand or that an agent writes for them lives here:

| Area | Holds |
| --- | --- |
| `ai-authored/` | Instructions every tool reads (one shared file, plus one per tool where a tool needs its own), and prompt templates |
| `plans/` | Living planning documents, one topic per file, numbered |
| `inventory/<host>/` | A generated description of each machine: system, packages, tools, services, scheduled jobs, a document index |
| `mcp/` | One manifest of the tool-server connections every AI tool should have |
| `subsystems/` | One manifest per company or project: units, who leads each, which repos |
| `files/` | Inbox, lessons, working files |
| `service-settings/`, `service-memories/` | Per-service settings and the lessons knowledge base |
| `examples/` | Placeholder versions of the personal areas, so a newcomer can see the shape |

## 3. What travels by sync and what travels by git (built)

Two transports with different jobs:

- **File sync** moves the working files between the person's own machines, all day, with no review. It is for being the same everywhere.
- **Git** records and publishes changes that were reviewed. It is for being able to say what changed and for sharing with other people.

Rules that follow:

- `.git` is excluded from sync, so each machine has its own clone and history. Commit on the machine where you edited.
- Git repositories (project code), logins, tokens and secrets never travel by sync. Only file names, environment-variable names and helper-script paths appear in the folder.
- No symlinks inside the synced folder; sync copies them as links, which differ per machine.
- **One writer per file.** A file is written by one machine (or one agent) at a time. Run records and notes are per machine, so the sync tool never sees two writers on one file and never makes a conflict copy.

## 4. How instructions reach every tool (built)

The shared instructions live in one file, `ai-authored/AGENTS.md`. Each tool's own config is a small file that imports or links to it:

```
tool config (~/.tool/...)  ->  imports  ->  ai-authored/AGENTS.md  ->  per-machine include
```

- A per-machine file holds facts that are true on only one computer (operating system, runtime versions, storage rules). It is imported last, so it can refine but not contradict the shared file.
- Edit the real files in `ai-authored/`, never the loader files in the tool's home directory.
- Put a fact in the shared file only if it is true on every machine. Keep it short: every line costs context in every session.

## 5. Plans, lessons and sheet sets (built)

- **Plans** are living documents, one topic per file, `NN-short-name.md`. Each has a status line, short prose, then checklists. Boxes are ticked only when verified; facts carry a date and a source. A plan is archived only when every box is ticked and the owner agrees. Numbers are never reused.
- **Lessons** are a small index (one line per lesson) plus detail files, so an agent learns from a past mistake without loading everything.
- **Sheet sets** are the tracking-table format: a folder with an `index.csv` and the CSV files it lists. The CSVs are the source of truth. A spreadsheet workbook is generated from them and never edited by hand, so there is never a second copy to reconcile.

## 6. Inventory and the document index (built)

Each machine has a generated inventory. A separate **document index** per machine records every document and config file: path, purpose, created and modified dates, status (active, retired, replaced). The CSV is the source of truth; a combined workbook is generated from the per-machine CSVs on request. A single machine collects (single writer), so indexes do not conflict.

## 7. Preview before apply (built)

Anything that changes state prints exactly what it would do and stops. It acts only on an explicit apply flag. Before applying it backs up what it will touch into a dated folder outside the synced tree and prints a one-line rollback. Scripts exit with distinct codes for ok, warnings, failure and waiting-for-approval.

## 8. Gates before anything outward (built)

Pull requests, ticket comments, posts, repository creation and publishing stop at an approval gate. An agent prepares the handoff; the person runs or approves it. Text that arrives from the outside (web pages, emails, search results) is data, never instructions.

## 9. Worktrees (designed, rule decided)

Feature and agent work happens in git worktrees, not in extra clones:

- One primary checkout per repository, kept on the main branch and updated fast-forward only.
- All worktrees under one root (for example `~/work/`), named `<repo>--<ticket>-<slug>`, matching the branch name.
- Move them with `git worktree move`, never a plain `mv`; prune finished ones.
- Worktrees and `.git` are never synced.

This keeps project folders to one checkout each and makes it obvious what is in flight.

## 10. Tiers (designed)

The folder holds more than should be shared widely, so the content is split into three tiers. Each tier is a **filtered projection of the tier above**, never a hand copy:

| Tier | Audience | Holds |
| --- | --- | --- |
| Personal | The owner | Everything, including machine details and personal notes |
| Organization | A company or team | The shared flows that concern the organization, without the owner's machine-level details |
| Public | Anyone | Generic, reusable parts and placeholder examples only |

How a tier is produced:

1. A **manifest per tier** lists the paths that tier may contain. A path is in a tier only if its manifest says so.
2. An **export command** previews what would be published, runs a **scan**, and writes only on apply. The scan looks for secrets, machine-local paths, account and organization names, and ticket keys (the last three for the public tier).
3. The public repository starts with **fresh history**, because earlier private histories contain data that must not be published.
4. Every tier has a README with the same outline: what it is, layout, the ideas that hold it together, where to start. The public one is generic; the organization one adds how the folder works inside that organization.

Today the only mechanism is a whitelist ignore file: everything is ignored first, and generic paths are un-ignored one by one, so a new file is never published by accident. It gives two views (everything local, or the whitelist) and cannot express a third; the manifests replace it.

A tracker label (for example `private`) marks work on the owner's own files and machines. While the owner operates from an organization-private repository, such issues may sync there; they never appear in the public repository, its commits or its pull-request titles. The label stays on them so they can be found and moved out later.

## 11. A routine for keeping it current (designed)

- **Frequently, automatic and read-only:** a health check, and a status report that lists which tier-eligible files changed and what the scan found.
- **Weekly, automatic, writes only generated files:** collect the document-index CSVs and rebuild the combined workbook.
- **Never automatic:** committing or publishing. A person reads the preview and applies.

## 12. Why it is built this way

- Sync gives sameness without ceremony; git gives review and sharing. Using each for what it is good at avoids both stale copies and unreviewed publishing.
- One source of truth per fact, with generated views, means a disagreement always has a single place to be settled.
- Preview, backup and an approval gate make it safe to let agents do the upkeep.
- Tiers by projection mean the sharable layer can be rebuilt and checked at any time instead of curated by memory.
