# Blank tracking templates

This folder is a generic, empty sheet set for optional local tracking. Every CSV contains headers only. Copy the folder into a
workspace if these records are useful, then build its workbook with `mach sheets build PATH`.

The files separate machine identity, workspace membership, scan history, installed tools/integrations, and peer connections.
Keep the set local unless you have reviewed its fields for the workspace's sharing policy. Never put tokens, private keys, raw
configuration, or unreviewed machine inventory into these tables.

| CSV | Tracks |
| --- | --- |
| `machines.csv` | Human-assigned machine identifiers and purpose |
| `workspaces.csv` | Independent personal or group workspace roots and sync choices |
| `scans.csv` | Scan category, date, scope and review state |
| `tools.csv` | Developer tools, apps, plugins and MCP names/versions |
| `connections.csv` | Peer relationship and transport state; no device keys or IP addresses |

`index.csv` is the sheet-set index. The workbook is generated from these CSVs and is never edited directly.
