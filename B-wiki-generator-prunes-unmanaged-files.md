---
id: B-wiki-generator-prunes-unmanaged-files
title: "WikiDiskGenerator deletes every unregistered Markdown file in a configurable output directory without proving PinWright owns it"
status: OPEN
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

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the regeneration-clobber catalog; no editor, build, test, or RPC run was performed.
