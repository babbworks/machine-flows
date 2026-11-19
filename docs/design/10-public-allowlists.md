# Public exports with an allowlist

A reliable public export starts from an empty destination and selects only
reviewed files. This keeps new local files private until someone deliberately
adds them to the publication map.

## Select files explicitly

Keep the allowlist beside the source project, but outside the public export if
it contains real organization names or private paths. A generic example lives
in [`examples/publishing/`](../../examples/publishing/). For `mach publish`, a
line-based manifest can include or exclude paths, map a reviewed source to its
public destination, enable scans, and deny names that must not appear.

## Publish generic rewrites

When a useful source document contains local details, write a separate generic
version. Map that version into the public tree and retain the original in its
local tier. This makes the relationship visible without exporting the source
content. Keep the rewrite's provenance in the private classification map.

## Preview and review

Run the exporter in preview mode. Review every selected path and each finding,
then apply only when the result is clean. Treat scans as a backstop: the
allowlist and human review decide what belongs in the public export.
