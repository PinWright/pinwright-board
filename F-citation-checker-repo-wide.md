---
id: F-citation-checker-repo-wide
title: "Nothing validates file:line citations anywhere in the plugin — check_hazards is host-project-only and scoped to Docs/scripts/, and a line-range check alone would catch 1 of the 13 stale citations found"
status: OPEN
severity: Medium
category: feature
tags: [docs, citations, file-line, tooling, drift, check_hazards, contract-test, staleness]
encounters: 1
lastSeen: 2026-08-20T00:00:00Z
---

# The only citation checker is in the other repo, and it looks at one directory

`check_hazards.py` lives at `X:\src\unreal\EAContentExamples58\Docs\tools\check_hazards.py` — the
**host project**, not this plugin — and its input corpus is `Docs/scripts/` only: `SCRIPTS_DIR` at
`:44`, the `--root` default at `:2291-2292`, and its own note at `:2094-2096` that citations outside
that directory are "invisible to this gate". Resolution was widened to the whole tracked host repo,
but the input corpus was not: only `.py` banners under `Docs/scripts/` and the hazard table in
`Docs/scripts/README.md` are ever scanned. Nothing under `Plugins/PinWright/` is an input. `:311-313`
also records its own ceiling — for Markdown it checks only that the cited line is within the file,
and `:315-318` explicitly declines semantic checking.

The plugin has no equivalent. `Source/**` contains no test matching `citation`, `file:line` or
`LineExists` (only prose comments at `LevelHandler.cpp:254-255`,
`MetaSoundPatchPresetHandler.cpp:14`, `MSIRDecompiler.cpp:615`); `Content/Python/` holds
`check_suite_log.py`, `mcp_proxy.py`, `recorder_query.py` and three proxy tests; `scripts/` holds
packaging `.ps1` and two `PROPOSED-ci-*.patch`. No checker in any of them.

`Source/PinWrightGeometry/Private/Model/PwModelParser.cpp:94` already records the rule with nothing
enforcing it: *"a line citation into another file goes stale on its next edit"*.

## What adjacent gates do exist

- `Source/PinWright/Private/Tests/Core/TestDocsIndexCoverage.cpp` — reachability of every maintainer
  doc from `Docs/index.md` and `Docs/tags.md`. `:16-21` explicitly declines frontmatter rows, index
  descriptions and ordering. Never citations.
- `Source/PinWrightGeometry/Private/Tests/Core/TestPwModelDiagnosticCatalog.cpp:202` —
  `DocumentedCodesMatchEmittedCodes`, reconciling the `PWMODEL_*` table in `pwmodel-format.md`
  against source in both directions. **The nearest working precedent and the shape to copy.**
- ~60 `Test*Docs.cpp` contract tests asserting doc *content strings*, never line numbers.
- `Docs/plans/defect-backlog.md` `D-95` — the same *shape* of gap for the error-code catalog
  (regenerate-and-diff), different artefact and mechanism.

## Measured, and the measurement changes the design

Scanned `Docs/**/*.md`, `Examples/**` and C++ comments under `Source/**` (excluding `Binaries/`,
`Intermediate/`, `dist/`, `scratchpad/`):

```
citation tokens scanned  : 2083
resolved inside plugin   :  790
external/engine (skipped): 1293
OUT OF RANGE             :    1
cited plugin files that no longer exist : 0
```

The single out-of-range hit: `Source/PinWright/Private/Tests/Infra/TestGameFrameworkAssetReadbackDocs.cpp:6`
cites `GameFrameworkHandler.cpp:865-911`; that file is **830 lines**.

A hand-checked random sample of 24 doc→source citations found **8 more** that resolve *in range* but
land on unrelated code:

| Citing site | Cites | What is actually there | Real site |
|---|---|---|---|
| `Docs/geometry-debug-sink-sweep.md:130` | `PwModelCompiler.cpp:985` for `ClearMaterialIDs` | `const FTransform Local = ReadTransformParams(Op.Params);` | `:1034` |
| `Docs/plans/defect-backlog.md:106` | `PwModelParser.cpp:1896-1901` for `PWMODEL_MATERIAL_ON_BOOLEAN` | comment on an absent spherical-projection fn | `:1987`, `:2001-2003` |
| `Docs/plans/pwmodel-skeleton-skin-use.md:13` | `PwModelCompiler.cpp:1929-1951` for `PWMODEL_UNSUPPORTED_IN_VERSION` | boolean-op enum selection | `:2112`, `:2122` |
| `Docs/plans/pwmodel-value-system.md:552` | `PwModelParser.cpp:1888` for `bBooleanOp` | a type check | `:1993`, `:2001` |
| `Docs/plans/defect-backlog.md:126` | `PwModelParser.cpp:1005` for `cap` | the `scale_start` param | `:1007`, `:1021` |
| `Docs/rpc-hard-removal-rejected-candidates.md:18` | `RpcDispatcher.cpp:123-158` for `environment.build` | `StripInternalDispatchFields` comment | string absent from the file |
| `Docs/plans/pwanim-animation-format.md:336` | `TestSequencerHandlers.cpp:920-931` | the `sequencer.list_track_types` banner | no such test exists |
| `Docs/plans/defect-backlog.md:857` (D-8C) | `Tests/**Sequencer**/TestSequencerHandlers.cpp:920-931` | wrong directory — file is `Tests/Media/` | — |

Plus `Docs/engine-research-2026-08-plugin-pass.md`, wrong three ways: `:90` cites
`PinWrightGeometry.Build.cs:54` for "already links GeometryScriptingCore", but `:54` is an
engine-version probe tuple and the symbol is at `:75`; `:79-80` plans registering
`UNSUPPORTED_ORTHOGRAPHIC_ROTATION` "which was never registered", which **has landed**
(`Handlers/ErrorCodes.h:1098`, emitted at `PreviewViewportCaptureUtils.cpp:1066` and
`ViewProjectionUtils.cpp:103`); `:161` cites `RpcDispatcher.cpp:326` for `FScopedUnattendedRpc` when
the real sites are `:335`/`:586` — and `Docs/lessons.md:177` cites `:335` correctly, so two docs
disagree about one symbol. `:250` cites `CameraFrameHandler.cpp:291-294` as "the in-repo comment
asserting otherwise"; those lines are now an `if (Fov <= 0.0f)` validation block.

And `D-8B` is still live post-wave: `ModelCompileHandler.cpp:73` still cites
`GeometryAssetCreate.cpp:169` for the provenance-stamp comparison; `:169` is blank and the comparison
is at `:147-148`. Commit `937dbd39` ("Cut the comments and citations that had drifted from the code")
did not reach it.

**Total confirmed stale: 13. Out-of-range catches 1.**

## Design consequence

A checker built on line-range alone would catch **1 in 13**. Files grow, so a drifted citation almost
always stays in range and points at unrelated code. The gate has to resolve the *symbol or text* the
citation claims to be about — the `DocumentedCodesMatchEmittedCodes` pattern — or citations have to
carry an anchor that can be re-found (a symbol name rather than a bare line). Either is a bigger
build than the item implied, and shipping only the range check would produce a green gate over
twelve live stale citations.

**Fix:** a plugin-side automation test in the `Test*Docs.cpp` family that walks every `file.ext:NNN`
token in `Docs/**`, `Examples/**` and `Source/**` comments, resolves the path, fails on
missing-file/out-of-range, and — for citations that name a symbol in the same sentence — fails when
that symbol is not within a small window of the cited line. Report the 13 above as the seed corpus,
and fix `D-8B` and `TestGameFrameworkAssetReadbackDocs.cpp:6` as part of standing it up.

## Related

- `Docs/plans/defect-backlog.md` `D-8B` (CONFIRMED) — one specific stale citation; its own Fixed note
  proposes exactly this checker but is scoped to correcting a single line. This is the generalisation.
- `Docs/plans/defect-backlog.md` `D-95` (REFINED) — error-code catalog staleness, same shape,
  different artefact.
- `B-example-triangle-counts-stale` — a numeric duplication with the same root cause.

## History
- `#1-no-citation-gate-and-range-alone-is-not-enough` `OPEN` reporter — `check_hazards.py` lives in the host project (`Docs/tools/check_hazards.py`) and takes its input corpus from `Docs/scripts/` only (`SCRIPTS_DIR` `:44`, `--root` default `:2291-2292`, self-documented at `:2094-2096`), so nothing under `Plugins/PinWright/` is ever an input; the plugin itself has no citation checker in `Source/**`, `Content/Python/` or `scripts/`. Adjacent gates cover other things: `TestDocsIndexCoverage.cpp` checks doc reachability and `:16-21` declines everything else; `TestPwModelDiagnosticCatalog.cpp:202` reconciles the `PWMODEL_*` table against source and is the working precedent. Measured: 2,083 citation tokens across `Docs/**`, `Examples/**` and `Source/**` comments, 790 resolving inside the plugin, exactly **1** out of range (`Tests/Infra/TestGameFrameworkAssetReadbackDocs.cpp:6` cites `GameFrameworkHandler.cpp:865-911`; the file is 830 lines) and 0 pointing at deleted files. A hand-checked sample of 24 doc→source citations found **8** that resolve in range but land on unrelated code — `geometry-debug-sink-sweep.md:130` cites `PwModelCompiler.cpp:985` for `ClearMaterialIDs`, which is at `:1034`; `defect-backlog.md:857` cites `Tests/Sequencer/TestSequencerHandlers.cpp` when the file is under `Tests/Media/` — plus four in `engine-research-2026-08-plugin-pass.md` (one of which also proposes registering `UNSUPPORTED_ORTHOGRAPHIC_ROTATION`, a code that already exists at `ErrorCodes.h:1098`) and D-8B's `ModelCompileHandler.cpp:73`, still live after `937dbd39`. **13 confirmed stale against 1 out-of-range**: a range-only checker catches 1 in 13, because files grow and drifted citations stay in range — so the gate must resolve the symbol the citation names, not just the line number, and shipping the range check alone would produce a green gate over twelve live stale citations. Seed corpus for the build is the 13 listed here. `PwModelParser.cpp:94` already states the rule with nothing enforcing it. Distinct from D-8B (one line, and its Fixed note proposes this checker without building it), D-95 (error-code catalog), D-96 and D-97 (host-path citations, already FIXED).
