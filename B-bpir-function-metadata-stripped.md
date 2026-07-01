---
id: B-bpir-function-metadata-stripped
title: "BPIR strips function-level metadata (Category, Tooltip, Keywords, blueprint flags)"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, metadata]
---

# BPIR strips function-level metadata

Function `entry` lines in BPIR show only signature + position; metadata that lives in the BP function's Details panel (Category, Tooltip, Keywords, `CallInEditor`, `BlueprintNativeEvent`, `BlueprintCallable`, `BlueprintPure` flags) is completely absent.

Sample line:
```
entry function `2D Clouds Shading Offset Vector`() -> struct<LinearColor> @(13708, -60) {
```

No preceding lines carrying `Category="Sky.Clouds"`, `Tooltip="..."`, or function-flag annotations. A round-trip recompile loses these annotations entirely.

## Affected

- `Game/UltraDynamicSky/Blueprints/Ultra_Dynamic_Sky/bpir.txt` — 369 functions, none with metadata
- `Game/M4/Blueprints/MapMakerInterface/bpir.txt` — 100+ functions

## Fix sketch

`BpirTextEmitter.cpp`: extend the per-function emission path to include an optional metadata block before/after the signature line:

```
metadata function `2D Clouds Shading Offset Vector` {
    Category: Sky.Clouds
    Tooltip: "Returns the offset for 2D cloud shading."
    Keywords: clouds sky
    BlueprintCallable: true
    BlueprintPure: true
}
entry function `2D Clouds Shading Offset Vector`() -> ... {
```

Parser updates needed in `BpirParser.cpp` to round-trip the new block.

## History
- `#1-fn-metadata-stripped` `OPEN` reporter — affects 369 functions in Ultra_Dynamic_Sky alone. Pure additive fix; minimal grammar impact.
- `#2-already-has-entry-decorators` `IN-REVIEW` developer — Verified current source already implements function-level BPIR metadata via `@meta(...)` / `@flags(...)` entry decorators rather than the proposed `metadata function` block. Covered paths: `EmitEntryDecoratorLines`, `ParseMetaDecorator` / `ParseFlagsDecorator`, compiler metadata application helpers, and existing metadata emit/compile regression tests. No source edits needed.
- `#3-skip-mcp-unavailable` `SKIP` tester — Cannot exercise the fix: the `mcp__editor-automation__call` tool is not present in this session (absent from the deferred-tool list; ToolSearch for it and for blueprint.decompile/asset.dump returns no matches), so no live `blueprint.decompile` or fresh `asset.dump` is possible. Cached `bpir.txt` for the affected assets exists but may predate the fix; the ticket's surface is behavioral (decompile emitting `@meta`/`@flags` decorators), not file/doc state, so source/cache inspection alone cannot PASS per protocol.
- `#4-verify-fix` `DONE` tester — Verified: live `blueprint.decompile` on `/Game/UltraDynamicSky/Blueprints/Ultra_Dynamic_Sky` (3.2 MB payload) emits both decorator forms on function entries. Found `@meta(Category=..., Tooltip=..., Keywords=...)` (e.g. `@meta(Category=\"Utility Functions\", Tooltip=\"Converts Time of Day into a standard Timecode format...\", Keywords=\"timecode time seconds minutes hours frames\")`) and `@flags(Public, Pure)` / `@flags(Protected)` annotations. Metadata is no longer stripped; the reported bug is resolved.
