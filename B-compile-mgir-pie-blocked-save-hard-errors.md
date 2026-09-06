---
id: B-compile-mgir-pie-blocked-save-hard-errors
title: "material.compile_mgir raises MGIR_SAVE_FAILED when another session's PIE blocks the write, discarding the whole success payload for a graph write that DID land — the caller is told the compile failed when only the save did"
status: IN-REVIEW
severity: High
category: bug
tags: [material, mgir, compile_mgir, save, pie, savestate, error-payload, multi-agent, transient, false-failure]
encounters: 1
lastSeen: 2026-09-05T20:15:00Z
---

# `material.compile_mgir` turns a transient PIE save block into a hard RPC error and loses the compile result

## Symptom

In a shared editor where another stream held a PIE session (`editor.pie_status` →
`inPie: true`, `T_Player`, one standalone context), the first `material.compile_mgir` on a
freshly created master returned **only an error**:

```
[MGIR_SAVE_FAILED] Material compiled but no .uasset reached disk:
/Game/FPS/Weapons/Materials/M_WPN_OpticLens.M_WPN_OpticLens
Docs: X:/.../Saved/PinWright/wiki/material.compile_mgir.md
```

The identical call re-issued with `save: false` returned success:

```json
{"mode":"Append","blocksCompiled":1,"expressionsCreated":33,
 "assetPaths":["/Game/FPS/Weapons/Materials/M_WPN_OpticLens.M_WPN_OpticLens"],
 "consumerRefresh":{"measured":true,"consumersFound":0,"complete":true}}
```

Nothing about the document changed between the two calls. The blocker was PIE, and the error
text names neither PIE nor the fact that the graph write itself succeeded.

## Why this is worse than the sibling PIE tickets

`B-asset-save-omits-savestate-pie-block` / `B-asset-save-pie-failure-reports-pendingflush` are
about `asset.save` **dropping a field** from a response that still arrives.
`B-model-compile-pie-blocked-save-unattributed` is about `model.compile` returning
`success: true` with `saveState: "failed"` and no cause. Both of those still hand the caller a
payload.

`material.compile_mgir` hands back **nothing**. The error is raised in place of the response, so
on the failing call the caller loses `blocksCompiled`, `expressionsCreated`, `assetPaths`,
`consumerRefresh` **and** `shaderCompile` — the very block the wiki tells them to branch on
(`material.mgir` § Compile / decompile: *"branch on `shaderCompile.status`"*). The error message
says "Material compiled but no .uasset reached disk", so the caller can infer the graph landed,
but has no measurement of it and no way to tell whether the shader compiled, whether consumers
refreshed, or how many expressions were placed.

Two concrete failure modes this creates in a shared editor:

1. **A caller who treats the error as "the compile failed" retries in `Append` mode.** `Append`
   empties and rebuilds the expression collection, so a retry loop against a PIE session that is
   never going to end re-tears-down and rebuilds a live master's graph on every attempt, with
   `consumerRefresh` unread each time. The wiki warns about exactly this reallocation
   (`material.mgir` § *Extend cannot update an existing expression*), and the error steers the
   caller straight into it.
2. **A caller who treats it as fatal abandons a compile that already succeeded**, leaving the
   in-memory master ahead of disk — the divergence `B-model-compile-pie-blocked-save-unattributed`
   `#2` describes, now with no response payload to reconstruct what the in-memory revision
   contains.

## Expected

Return the normal success payload and report the save outcome inside it, the way every other
mutator on this path does. The classifier already exists — `B-editor-save-all-pie-diagnostic` is
`DONE` and landed `ClassifyFailureReason` + a `pieActive` / `editorMode` probe for
`editor.save_all`:

```json
{"mode":"Append","blocksCompiled":1,"expressionsCreated":33,
 "assetPaths":["/Game/FPS/Weapons/Materials/M_WPN_OpticLens.M_WPN_OpticLens"],
 "shaderCompile":{"status":"completed","succeeded":true},
 "consumerRefresh":{"complete":true},
 "saved":false,"saveState":"blockedByPie","saveCause":"BlockedByPie","retryable":true,
 "pieActive":true,"editorMode":"PIE",
 "pieWorlds":[{"pieInstance":0,"mapName":"T_Player",
               "worldPath":"/Game/FPS/Test/UEDPIE_0_T_Player.T_Player"}],
 "saveDetail":"The graph was written and the shader compiled; no package can be saved while a
               play session is running. Re-issue asset.save on the listed paths after PIE stops.
               Do NOT re-run this document in Append mode — the graph is already current."}
```

`MGIR_SAVE_FAILED` should stay for the causes that are genuinely fatal (read-only file, an
unmountable path, a save the package system refuses on its own terms). A PIE block is
self-clearing and belongs in the payload, not in an exception.

## Second, smaller finding on the same call

`material.compile_mgir` accepted `waitForShaderCompile: true` and the successful (`save: false`)
response carried **no `shaderCompile` block at all**. The wiki is explicit that this block is the
measurement and that `waitForShaderCompile: true` is how you get the real verdict rather than the
non-blocking probe. Absent, a caller cannot distinguish "shaders are clean" from "nothing was
measured"; the wiki's own advice — *"treat `notCompiled` as 'no compile has run' rather than as
clean"* — has nothing to read. Working around it costs a separate
`material.authoring.compile_material` call, which did return
`compileStatus: "completed"` / `compileSucceeded: true` for the same asset. Filed here rather
than separately because both were observed on the same two calls; split it out if the owner
prefers.

## Repro

1. Have another session start PIE (`editor.pie_status` → `inPie: true`).
2. `material.authoring.create_material {assetPath: "/Game/Tmp/M_Probe", blendMode: "Translucent"}`
   — note it returns a payload with `saved:false`, `pendingFlush:true`, `pieActive:true`,
   `editorMode:"PIE"`, `pieWorlds[]`. This is the shape `compile_mgir` should have.
3. `material.compile_mgir {text: "<any valid entry material block for that path>",
   mode: "Append", save: true}` → `MGIR_SAVE_FAILED`, no payload.
4. Re-issue with `save: false` → full success payload, same document.

Observed 2026-09-05 on UE 5.8, PinWright MCP port 27145, while authoring
`/Game/FPS/Weapons/Materials/M_WPN_OpticLens`.

## Workaround

Probe `editor.pie_status` first; when `inPie` is true, pass `save: false` on every
`material.compile_mgir` / `material.authoring.*` call, keep the paths, and issue `asset.save` on
each after PIE ends. That is what was done here. It requires the caller to know in advance that
this verb's save path throws rather than reports, which nothing in the wiki says.

## Fix

The MGIR finalizers called `SaveAssetToDiskReportingPresence` as a bool and converted every
non-durable result into `MGIR_SAVE_FAILED`, discarding the helper's measured
`EAssetSaveState::BlockedByPie` distinction before the handler could build its success payload.
`FMGIRCompileResult` now carries the aggregate save state; material and material-function saves
continue through a PIE block but preserve hard `MGIR_SAVE_FAILED` errors for every other
non-durable requested save. `MGIRCompileHandler.cpp` emits the shared AssetSaveState report and
adds `saveError: "PIE_ACTIVE"` on that one transient branch.

Files changed:
- `Source/PinWright/Private/MGIR/MGIRCompiler.h`
- `Source/PinWright/Private/MGIR/MGIRCompiler.cpp`
- `Source/PinWright/Private/Handlers/Material/MGIRCompileHandler.cpp`
- `Source/PinWright/Private/Tests/Material/TestCompileMgirPieBlockedSave.cpp`
- `docs/wiki-src/material.mgir.md`

Regression test: `PinWright.material.compile_mgir.ReportsPieBlockedSaveWithoutDiscardingCompile`.
It calls the real handler on a GUID-scoped asset saved to disk, simulates the engine PIE gate,
and asserts the retained compile payload, typed blocked-save report, in-memory graph mutation,
and byte-identical on-disk baseline. Per the worker brief, it was authored but not run here.

Deliberately not changed: the separate `waitForShaderCompile` observation. Current source already
routes `true` through `ProbeAndWait` and then `AddReport`; it is outside this worker's assigned
PIE-save contract and needs separate runtime evidence if it can still be reproduced.

## History
- `#1-pie-block-raised-as-error-payload-lost` `OPEN` reporter — First encounter. `material.compile_mgir {save: true}` on a fresh master raised `MGIR_SAVE_FAILED` while another stream's PIE session held the editor; the same document with `save: false` returned `blocksCompiled: 1`, `expressionsCreated: 33`, `consumerRefresh.complete: true`. Distinct from the `asset.save` and `model.compile` PIE tickets: those return a payload with a field missing or a cause missing, this one returns no payload at all, and the mode it strands the caller in (`Append`) rebuilds the target graph on every retry. Also notes `waitForShaderCompile: true` producing no `shaderCompile` block on the successful call.
- `#2-waitforshadercompile-true-REMOVES-the-shadercompile-block` `OPEN` reporter — The "second, smaller finding" above is now a controlled pair, and it inverts the documented contract. Two `material.compile_mgir` calls on the same asset, minutes apart, differing only in that flag: with `waitForShaderCompile: true` the response was `{"mode":"Append","blocksCompiled":1,"expressionsCreated":33,"assetPaths":[...],"consumerRefresh":{...}}` and **no `shaderCompile` key at all**; with the flag omitted (default `false`) the response carried `"shaderCompile":{"status":"outstanding","succeeded":false,"failed":false,"rendersDefaultMaterial":true,"hint":"A shader compile is still in flight...","materials":[{"assetPath":"/Game/FPS/Weapons/Materials/M_WPN_OpticLens.M_WPN_OpticLens","status":"outstanding"}]}`. So the parameter whose entire purpose is *"report the real verdict rather than the non-blocking probe"* removes the block that carries the verdict, while omitting it returns the probe as documented. A caller who follows `material.mgir` § Compile / decompile — *"branch on `shaderCompile.status`"* — and passes the flag has nothing to branch on, and an absent key is easy to read as "no problem" rather than "no measurement". Workaround is the same either way: a separate `material.authoring.compile_material`, which returned `compileStatus:"completed"`, `compileSucceeded:true`, `shaderCompile.status:"completed"` for this asset. Both observations 2026-09-05, UE 5.8, `/Game/FPS/Weapons/Materials/M_WPN_OpticLens`.
- `#3-report-pie-blocked-save` `IN-REVIEW` developer — Carried `EAssetSaveState::BlockedByPie` through `MGIRCompiler.cpp` instead of raising `MGIR_SAVE_FAILED`, emitted the shared save report plus typed `saveError: PIE_ACTIVE` from `MGIRCompileHandler.cpp`, documented the contract, and added `PinWright.material.compile_mgir.ReportsPieBlockedSaveWithoutDiscardingCompile` with a real saved asset and simulated PIE gate.
