# Publishing with an allowlist

Use a manifest to define the files a public export may contain. Start with an
empty export and add only generic files that have been reviewed. A new file is
private by default until the manifest includes it.

[`public.manifest.example`](public.manifest.example) shows the line-based
manifest format used by `mach publish`. Its `include` entries select files,
`exclude` entries narrow the selection, and `map` entries export a reviewed
source file at a different destination. `scan` enables additional checks, and
`deny` lists names that must not appear in the selected content.

Keep the real manifest and its organization-specific deny list private. If a
source document contains local or company details, write a generic version and
map that version into the export; do not copy the original and hope a scan
removes sensitive content. Preview with `mach publish --tier public`, review
the file list and scan results, then apply only after the preview is clean.

This is a starting example. Adjust the paths and checks for the project, and
keep the allowlist smaller than the source tree.
