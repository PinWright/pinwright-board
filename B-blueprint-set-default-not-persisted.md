---
id: B-blueprint-set-default-not-persisted
title: "blueprint.set_default only mutates the live CDO — override never reaches the .uasset, silently reverts on editor restart"
status: IN-REVIEW
severity: High
category: bug
tags: [set-default, cdo, persistence, silent-false-success, throttled-save, blueprint]
encounters: 1
---

# `blueprint.set_default` only mutates the live CDO — the override never reaches the `.uasset` and silently reverts on editor restart

**Symptom:** `blueprint.set_default` returns success and every in-session check
agrees the write landed: the readback `value` in the response is correct,
`asset.dump` / `property.get` show the new default, `blueprint.compile` reports
`UpToDate`, and `asset.save` reports saved. But the `.uasset` on disk never
changes, and after an editor restart the property silently reverts to its old
default. Real overrides were lost to this in production use — the agent (and
user) had no in-band signal anything was wrong because the readback comes from
the same live CDO the handler just wrote.

**Root cause** (in `Source/PinWright/Private/Handlers/Blueprint/BlueprintPropertyHandler.cpp`,
`blueprint.set_default` handler):

1. The handler wrote the value onto the live in-memory CDO and called only the
   NON-structural `FBlueprintEditorUtils::MarkBlueprintAsModified` — it never
   called `FKismetEditorUtilities::CompileBlueprint`, so the new default was
   never baked into the Blueprint (the sibling `add_variable` and every
   `blueprint.scs.*` mutation compile; the wiki documents set_default as
   "then compiles and saves" — the code regressed away from that).
2. The save went through `SaveLoadedAssetThrottled` (`Utils/AssetUtils.cpp`),
   which **returns true when it SKIPS the write inside the throttle window** —
   and the handler ignored the return value anyway and emitted no `saved`
   field, so a skipped save was indistinguishable from a real one.
3. The readback was taken from the same live CDO that was just written
   (comment claimed "no recompile, CDO is still valid"), so verification
   always looked correct regardless of persistence.

Contrast: `blueprint.scs.set_property` persists because
`FSCSHandlers::FinalizeBlueprintSCSChange` (`PinWright_SCSHandlers.cpp`) does
`MarkBlueprintAsStructurallyModified` + `CompileBlueprint` + `McpSafeAssetSave`.

**Workaround (pre-fix):** none reliable — every readback surface lied. A
manual full recompile + forced save of the Blueprint from the editor UI after
each set_default was the only way to make the override stick.

**Fix:** rework the set_default persistence block to mirror the proven SCS
finalize sequence, kept local to `BlueprintPropertyHandler.cpp`:
`MarkBlueprintAsStructurallyModified` + `FKismetEditorUtilities::CompileBlueprint`;
re-fetch the CDO after the compile (the compile invalidates the old
CDO/FProperty/container pointers and rebuilds defaults), re-resolve the
property against the fresh CDO and re-apply the JSON value via
`ApplyJsonValueToProperty`; mark-for-save via `McpSafeAssetSave` (the SCS
path's helper — not the throttled saver) and report its result as a new
`saved` boolean in the response; take the readback from the re-fetched
post-compile CDO. The now-wrong "no recompile, CDO is still valid" comment is
replaced. Regression test
`PinWright.blueprint.set_default.PersistsThroughCompile` in
`Tests/Blueprint/TestBlueprintHandlers.cpp` drives add_variable + set_default
on a transient Actor BP and asserts success, `saved:true` (pre-fix the response
carried no `saved` field), no dirty residue on the transient package, and the
value surviving a fresh `CompileBlueprint`. Fix commit: `0f632687` (plugin
master).

## History
- `#1-production-revert-observed` `OPEN` reporter — Confirmed in production use: `blueprint.set_default` overrides read back correctly all session (response echo, `asset.dump`, `property.get` all agree; `blueprint.compile` reports `UpToDate`; `asset.save` reports saved) yet the `.uasset` never changes and the values revert on editor restart. Root cause read from source: no structural mark, no `CompileBlueprint`, save routed through `SaveLoadedAssetThrottled` whose throttle-window skip returns true with the return value ignored and no `saved` field emitted; readback taken from the same live CDO that was just written. `blueprint.scs.set_property` persists via `FinalizeBlueprintSCSChange` (structural mark + compile + `McpSafeAssetSave`), isolating the defect to set_default's finalize block.
- `#2-compile-reapply-safe-save` `IN-REVIEW` developer — Reworked the `blueprint.set_default` finalize block in `Source/PinWright/Private/Handlers/Blueprint/BlueprintPropertyHandler.cpp` to replicate the SCS finalize sequence locally: `MarkBlueprintAsStructurallyModified` + `CompileBlueprint`, then re-fetch the CDO from `Blueprint->GeneratedClass->GetDefaultObject()` (old pointers are invalid post-compile), re-resolve the property/target container (including the nested `ResolveNestedPropertyPath` branch) and re-apply the value via `ApplyJsonValueToProperty`, then `McpSafeAssetSave(Blueprint)` with its result emitted as a new `saved` response field; readback (`ExportPropertyToJsonValue`) now comes from the post-compile CDO. Transaction/Modify calls, error paths, and the rest of the response shape are unchanged. Regression test `PinWright.blueprint.set_default.PersistsThroughCompile` added to `Tests/Blueprint/TestBlueprintHandlers.cpp` (transient Actor BP via `CompilerTestUtils::CreateTransientTestBP`, bool variable seeded through the real `blueprint.add_variable` handler, flip via `blueprint.set_default`, asserts success + `saved:true` + package not dirty + value survives a fresh `CompileBlueprint`). Wiki source `docs/wiki-src/blueprint.md` updated to document the `saved` response field. Fix commit: TBD.
- `#3-fix-landed-and-live-verified` `IN-REVIEW` developer — Fix committed as `0f632687` (rebased onto the concurrently-pushed `77856a0a`, pushed to plugin master). Verified before commit: full test run green including the new `PersistsThroughCompile` (489/493 suite; the 4 failures are pre-existing App PID-tuner tests, unrelated), then IN ANGER: seven production CDO edits (five on B_PioneerSumo incl. an enum + a float + three new props, two on B_SumoPedestal) all reported `saved:true`, binary-grep of both `.uasset` files showed every override's FName + value bytes on disk after a follow-up `asset.save`, and a full editor restart + fresh-session CDO probe confirmed every value survived. Caveat documented in the wiki: `saved:true` means queued-for-persistence (`McpSafeAssetSave` is mark-dirty-only by design to dodge the UE 5.7 recursive-FlushRenderingCommands crash); an explicit `asset.save` flushes to disk — during PIE it can defer with `saved:false, pendingFlush:true` and must be retried after PIE ends (observed live).
- `#4-saved-false-but-disk-written-no-pie` `IN-REVIEW` reporter — **The persistence fix works; the `saved` field is a false NEGATIVE outside PIE.** Two calls, both `blueprint.set_default {propertyName:"PenetrationThickness", value:"28"}`, on `/Game/FPS/Weapons/BP_Weapon_AR` (06:50:0xZ) and `/Game/FPS/Weapons/BP_Weapon_Pistol` (06:55:1xZ), 2026-09-07. Both returned `compiled:true, status:"UpToDate", value:28, markedForSave:true, saveRequested:true` **and `saved:false, pendingFlush:true, pendingSave:true` with NO `pieActive` / `editorMode` / `pieWorlds` keys**. `editor.pie_status` returned `{"inPie":false,"count":0,"contexts":[]}` immediately after the AR call and immediately before the pistol call, so no PIE session was involved in either. **The `.uasset` was nevertheless written to disk within ~10 s of each call, with no `asset.save` from me**: `BP_Weapon_AR.uasset` mtime moved to 06:50:06Z and `BP_Weapon_Pistol.uasset` 124271 -> 124069 B at 06:55:18Z, and a byte scan of both files shows the `PenetrationThickness` FName count going 1 -> **0** and the old `double 16.0` gone (the override equalled the new parent default, so UE correctly dropped it from serialization and both children now inherit the base 28). Disk state is exactly right; only the report is wrong. Contrast with a third call in the same window that hit a genuine PIE block (another stream started PIE between my calls): that one returned the documented shape **with** `pieActive:true, editorMode:"PIE", pieWorlds:[...]`, so the two cases are distinguishable in the payload and the non-PIE one is not the documented deferral. Why it matters operationally: `#3` above tells callers that `saved:false, pendingFlush:true` means "retry after PIE ends", and this project's editor-discipline rules make every extra `set_default` a synchronous compile + GC, which is an active editor-kill risk here (`B-blueprint-compile-gc-kills-editor-via-python-pre-gc-hook`). So a caller who believes this field retries a write that already landed and pays a second compile for nothing. Requested: either report `saved:true` when `McpSafeAssetSave` queued successfully and the flush is expected, or add a field distinguishing "queued, will flush" from "blocked, will not flush without a retry" — `pendingFlush:true` currently carries both meanings. Honest alternative I cannot fully exclude: some other agent in this shared editor issued a save that happened to flush both of my packages within ten seconds of each of my two calls; I judge that unlikely two-for-two but did not instrument it.
