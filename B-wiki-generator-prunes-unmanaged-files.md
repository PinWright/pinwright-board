---
id: B-wiki-generator-prunes-unmanaged-files
title: "WikiDiskGenerator deletes every unregistered Markdown file in a configurable output directory without proving PinWright owns it"
status: IN-REVIEW
severity: High
category: bug
tags: [wiki, catalog, prune, unmanaged-files, data-loss, output-directory]
---

# A Wiki output setting can turn startup regeneration into a broad Markdown delete

## What's wrong

`WikiDiskGenerator::OutputDirectory` accepts an arbitrary absolute path, or any project-relative
path, from `WikiOutputDirectory` (`Catalog/WikiDiskGenerator.cpp:377-391`). During generation,
`WikiDiskGenerator_PruneOrphans` enumerates every top-level `*.md` in that directory and deletes
any basename absent from the current page map (`:357-373`). It does not check the generated-file
header, a manifest, or a dedicated-root marker.

Concrete failure: configure the output as `Docs` (or another existing Markdown directory), launch
the editor, and every unrelated top-level document whose basename is not a wiki slug is deleted.
The generator writes pages first, ignores `SaveStringToFile` results (`:285-305`), then prunes
(`:439-441`), so an I/O failure can also delete old generated pages before a valid replacement set
exists.

High reflects durable broad document loss with a deterministic path, reduced from Critical because
the unsafe non-default output setting is required.

## What it should do

Require a dedicated owned output root and prune only files listed in the previous generated
manifest or carrying the exact PinWright-generated header. Stage and validate the whole generation
before replacing/pruning anything; abort pruning on any write failure.

## Workaround

Leave `WikiOutputDirectory` empty so output stays under the dedicated Saved/PinWright/wiki folder.

## Fix

Root cause: generation treated every top-level Markdown file in the configured output directory as
PinWright-owned and ignored page-write failures before pruning. `WikiDiskGenerator` now publishes
changed pages, `registry.json`, and `.pinwright-wiki-manifest.json` through the shared
`AtomicFileWriter`. Before any final publication it writes and byte-verifies the complete desired
set, including a conservative manifest, in one unique sibling staging directory; a staging failure
cleans that directory, leaves the prior output tree unchanged, and does not create an absent final
output root. After staging succeeds, changed pages and `registry.json` are published one at a time;
any failure stops before pruning. Authorized stale files are then pruned while failed deletions are
collected. The final manifest is rebuilt from desired filenames plus only those failures, revalidated,
and atomically published last. Successfully deleted names are therefore absent from a successfully
published manifest, while failed deletions remain owned for retry.

Per-file publication was chosen over a whole-directory swap because open Windows readers may block
directory renames. A mid-publication or final-manifest failure can leave earlier successful
publications or prunes; the implementation reports failure and does not claim whole-tree rollback.

Pruning requires both the canonical `Project/Saved/PinWright/wiki` output root and a valid previous
manifest with the exact owner/schema and strictly sorted, unique, safe top-level generated filenames.
A configured output override still receives generated files but reports pruning skipped and performs
zero deletion even when it contains a valid manifest. Missing, unreadable, malformed, oversized,
over-entry, wrong-owner, wrong-schema, unsorted, duplicate, absolute, path-containing, or
traversal-bearing manifests also authorize zero deletion. Pruning considers only previously listed
files absent from the desired set, and a failed delete remains listed for retry. Manifest input and
output are bounded to 4 MiB and 8,192 entries before deletion or manifest replacement.

Files changed: `Source/PinWright/Private/Catalog/WikiDiskGenerator.h`,
`Source/PinWright/Private/Catalog/WikiDiskGenerator.cpp`,
`Source/PinWright/Private/Tests/Infra/TestWikiDiskGenerator.cpp`,
`Source/PinWright/Public/PinWrightProjectSettings.h`, `docs/arch.md`, `docs/wiki-src/wiki.md`, and
`X:/src/unreal/.pinwright-board/B-wiki-generator-prunes-unmanaged-files.md`.

Regression tests: `PinWright.infra.wiki_disk_generator.PruneOnlyManifestOwnedFiles` and
`PinWright.infra.wiki_disk_generator.WriteFailureSkipsPruneAndManifestCommit`. They use an isolated
explicit-directory production reconciliation seam with per-call failure injection. Coverage includes
manifest-only ownership, invalid and 8,193-entry manifests, canonical-root enforcement, deterministic
8,192-entry bounds, canonical UTF-8 bytes, failed-delete retry ownership, and a late staging failure
after an earlier staged file. They also assert prune-before-final-manifest ordering and that a late
staging failure neither creates an initially absent final root nor leaves its sibling staging
directory. No test mutates the configured live wiki directory. Build and automation execution were
not performed in this pass.

Deliberately unchanged: public `OutputDirectory()` / `PagePath()` behavior, subsystem startup order,
the shared atomic-writer design, and unrelated file writers.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the regeneration-clobber catalog; no editor, build, test, or RPC run was performed.
- `#2-manifest-owned-pruning` `IN-REVIEW` developer — Replaced wildcard pruning with exact manifest-owned reconciliation, atomically published every generated output, retained failed deletes for retry, and added isolated ownership/failure regressions; no build, test, Unreal, or MCP run was performed.
