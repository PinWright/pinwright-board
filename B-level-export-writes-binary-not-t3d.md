---
id: B-level-export-writes-binary-not-t3d
title: "level.export writes a binary .umap package, not the documented text-format .t3d — breaks diff/copy-across-projects use case"
status: IN-REVIEW
severity: High
category: bug
tags: [level, export, t3d, contract-violation, silent-wrong-result]
---

# level.export writes binary Unreal package bytes, not text-format .t3d

`level.export` documents (wiki + handler description verbatim):
*"**Export a level to a text-format .t3d file on disk.** Useful for diffing or
copy-pasting actors across projects."* The implementation does not produce a
.t3d text file at all. It writes a **binary Unreal `.umap` package** to the path
the caller supplied (with a `.t3d` extension), so the on-disk bytes are opaque
serialized package data — not the human-readable `Begin Map`/`Begin Actor` T3D
text the doc promises. The call returns `{"success":true}` regardless, so the
caller has no signal that the artifact is the wrong format.

This defeats the method's entire stated purpose: a binary `.umap` cannot be
diffed line-by-line, cannot be hand-edited, and cannot be copy-pasted as actors
into another project the way a real T3D export can. It is a silent
success-with-a-wrong-result that contradicts the method's own documented
contract.

## Root cause (handler source)

`Source/PinWright/Private/Handlers/Level/LevelHandler.cpp`,
`level.export` handler (registered line 650, body around line 701-702):

```cpp
IFileManager::Get().MakeDirectory(*FPaths::GetPath(ExportPath), true);
FEditorFileUtils::SaveMap(WorldToExport, ExportPath);   // writes a BINARY .umap package
```

`FEditorFileUtils::SaveMap` performs a normal binary map save (UPackage
serialization). It never invokes a T3D text exporter
(`UExporter::ExportToFile` / `World->ExportText` / the `EditorExportT3D` path),
so the output is a binary `.umap`, not text `.t3d`. The handler also does not
echo back the written path; it returns only `{"success":true}`.

## Verbatim repro (replayed via mcp__editor-automation__call)

1. A level is active (`level.get_info` `{}` ->
   `{"levelPath":"/Temp/EditorAutomation/LightingStudy","actorCount":14}`).
2. `level.export` `{"exportPath":"X:/.../Saved/oracle_export_test.t3d"}`
   -> `{"success":true}`
3. Inspect the written file's first bytes:
   - First 4 bytes on disk: `C1 83 2A 9E` == little-endian `0x9E2A83C1`, the
     **Unreal package magic** (`PACKAGE_FILE_TAG`). A genuine T3D export would
     begin with ASCII `Begin Map` / `Begin Level`.
   - Only 197 of the first 512 bytes are printable ASCII (the rest is binary
     serialization) — confirmed not text.
   - File contains no `Begin Map` marker.

So the `.t3d`-named file is byte-for-byte a binary Unreal map package.

## What it should do

Either:
- Actually export T3D text — e.g. select the level's actors and run the T3D
  exporter (`UExporter::ExportToFile` with a `ULevelExporterT3D` / world
  `ExportText`), writing the human-readable `Begin Map { Begin Level ... }`
  format the doc and `level.import` (which advertises reading a `.t3d` file)
  both assume; or
- If a binary map copy is the real intent, fix the documentation and parameter
  text to say "binary .umap copy" (not "text-format .t3d", not "diffing"),
  rename the param away from `.t3d`, and echo back the written path so callers
  can locate the artifact.

The doc-vs-behavior split currently misleads any agent that picks `level.export`
specifically to get a diffable/portable text dump.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed against mcp__editor-automation__call: `level.export {"exportPath":".../oracle_export_test.t3d"}` -> `{"success":true}`; the written file's first 4 bytes are `C1 83 2A 9E` (Unreal package magic 0x9E2A83C1), only 197/512 leading bytes are printable, and it contains no `Begin Map` text marker — i.e. a binary .umap package, not the documented "text-format .t3d". Root cause: handler calls `FEditorFileUtils::SaveMap(WorldToExport, ExportPath)` (LevelHandler.cpp ~line 702), a binary package save, never a T3D text exporter. Handler also returns only `{"success":true}` with no echoed path.
- `#2-export-real-t3d-text` `IN-REVIEW` developer — Option A (doc-honoring root-cause fix). Replaced the binary `FEditorFileUtils::SaveMap(WorldToExport, ExportPath)` in the `level.export` handler with `GEditor->ExportMap(WorldToExport, *ExportPath, /*bExportSelectedActorsOnly=*/false)` — the engine's own map-export path (drives `UExporter::RunAssetExportTask` with Exporter=NULL, which picks the text `ULevelExporterT3D` by the `.t3d` extension and writes the "Begin Map { Begin Level ... }" text format). Since `ExportMap` returns void, the handler now verifies the file landed on disk and sends `EXPORT_FAILED` if nothing was written (instead of fake-success); it also echoes the written `exportPath` in the response (the prior secondary "no path signal" symptom). Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Level/LevelHandler.cpp` (handler body), `Source/EditorAutomationRpcGateway/Private/Tests/World/TestLevelHandlers.cpp` (new regression test + existing no-crash test switched to a `.t3d` path). Regression test `EditorAutomationRpcGateway.level.export.WritesT3dTextNotBinary` exports the live editor world via the production handler and asserts the on-disk bytes are T3D TEXT: leading bytes are NOT the package magic `C1 83 2A 9E`, the file contains a `Begin Map` marker, and the response echoes `exportPath` — all of which fail if the fix is reverted to `SaveMap`. Not compiled/run yet (later phase).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
