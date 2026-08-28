---
id: B-source-recompile-state-loss
title: "Source rebuilds silently discard asset state that the incoming source cannot express"
status: IN-REVIEW
severity: Critical
category: bug
tags: [pwsource, pwskel, pwanim, pwmodel, recompile, data-loss]
encounters: 3
lastSeen: 2026-08-28T09:10:00+05:00
---

# Source rebuilds silently discard asset state outside the incoming source

PinWright's source compilers generate one asset from one source file, but a later typed
RPC mutation can add asset state that the source does not describe. Recompiling the same
source, or taking over the asset from a different stamped source, can then replace or reset
that state and report success. A same-source provenance stamp provides a generated-state
baseline; a takeover must instead compare the live asset directly with the incoming source.

The measured skeleton case included a preview mesh, non-default per-bone translation
retargeting, and animation curve metadata. A source file with none of those fields rebuilt
successfully and silently lost them. The same failure class applied to model and animation
source compilers whenever an independent mutator changed a generated asset.

**Impact:** a normal successful recompile can irreversibly destroy authored asset data.

**Fix:** record the generated semantic state in provenance, compare baseline/current/desired
state before any destructive rebuild, and reject unmanaged changes by default with
`PWSRC_RECOMPILE_UNMANAGED_STATE`. A takeover without `overwrite=true` is refused and names
the state that would be lost. An explicit overwrite permits the takeover but changes that
same diagnostic to a warning. Never carry unspecified live state forward.
The skeleton source format now owns preview mesh, per-bone translation retargeting, and
animation curve metadata. Unsupported state remains guarded and can only be discarded by
an explicit overwrite.

## Encounter 2026-08-27 — the guard's comparison is broken for the model compiler's material slot: it compares an OBJECT path to a PACKAGE path

Observed 2026-08-27, UE 5.8, PinWright at this checkout's HEAD, while building
the Atlantis level (map as forcing function; see host `CLAUDE.md` § "What this
project is for"). **This is not a new mechanism — the fix belongs inside this
ticket's own comparison code**, which is why it is recorded here rather than
filed separately.

`PWSRC_RECOMPILE_UNMANAGED_STATE` refuses a recompile and prints two material
paths that are **the same material**:

```
[PWSRC_RECOMPILE_UNMANAGED_STATE] Recompiling .pwmodel asset '/Game/Atlantis/Meshes/SM_Fish_A'
would discard state the source does not reproduce: material[Fish] is
'/Game/Atlantis/Materials/MI_Fish_Blue.MI_Fish_Blue' but the source requests
'/Game/Atlantis/Materials/MI_Fish_Blue'; ... Put the state in the source, or pass
overwrite=true to discard it deliberately.
```

Live value is the **object** path (`/Package.Object`); the source value is the
**package** path (`/Package`). **The remedy the message names — "Put the state in
the source" — is literally impossible to carry out**, because a `.pwmodel` has no
spelling of a material that matches.

### Verbatim repro

1. Compile `SM_Fish_A.pwmodel` with
   `materials { Fish = "/Game/Atlantis/Materials/MI_Fish_Silver" }`. Succeeds;
   `materialSlotList[0].material` reads back `.../MI_Fish_Silver`.
2. Another agent binds `MI_Fish_Blue` to slot `Fish` through a static-mesh
   authoring verb (`static_mesh.set_material`).
3. Edit the source to `Fish = "/Game/Atlantis/Materials/MI_Fish_Blue"` — i.e. do
   exactly what the diagnostic asks — and recompile.
4. **Same error**, now quoting `MI_Fish_Blue` on both sides.

**Control in the same batch:** `SM_Fish_B`, whose live material had only ever
been written by `model.compile`, recompiled to a *different* material
(`MI_Fish_Orange`) without complaint. So the guard is not comparing against the
**source stamp** — which agreed — but against the **live object path**, which is
spelled differently from anything a `.pwmodel` can write.

**Second datum from the same responses:** the response's own
`materialSlotList[0].material` prints the **package** form
(`/Game/Atlantis/Materials/MI_Fish_Blue`) while the diagnostic prints the
**object** form for the live side. The two halves of one response disagree on
spelling.

### Root cause (guilty source lines — all verified against this tree at HEAD)

**The comparison is a raw `FString == FString` with no normalisation anywhere in
the file.** `PwSourceRecompileGuard::Check`,
`Source/PinWright/Private/PwSource/PwSourceRecompileGuard.cpp:63-77`:

```cpp
        const FPwSourceStateEntry* const* DesiredEntry = Desired.Find(Key);
        const FString DesiredValue = DesiredEntry ? (*DesiredEntry)->Value : FString();

        if (CurrentEntry.Value == DesiredValue)
        {
            // An out-of-band edit that has since been written into the source is source-owned
            // now. This is the accepting half of the three-way rule.
            continue;
        }

        if (Request.bSameSourceRecompile && bHasBaseline)
        {
            const FString* BaselineValue = Request.Baseline->Find(Key);
            const FString PreviousValue = BaselineValue ? *BaselineValue : FString();
            if (CurrentEntry.Value == PreviousValue)
            {
```

No `FSoftObjectPath`, no `FTopLevelAssetPath`, no normalisation call appears
anywhere in that 151-line file. The diagnostic is emitted at
`PwSourceRecompileGuard.cpp:131-135`.

**The two sides genuinely differ in shape.**

- **Live side** —
  `Source/PinWrightGeometry/Private/Handlers/Geometry/GeometryAssetCreate.cpp:133`:
  ```cpp
                Material.MaterialInterface ? Material.MaterialInterface->GetPathName() : FString(),
  ```
  `GetPathName()` yields the OBJECT path `/Game/Pkg/M_Foo.M_Foo`.
- **Source side** — same file, `:92-94`, built from `Spec.MaterialBindings`, a
  `TMap<FString, FString>` (`GeometryAssetCreate.h:101`) filled **verbatim** from
  the document's `Binding.AssetPath` at
  `Source/PinWrightGeometry/Private/Model/PwModelCompiler.cpp:3685` (skeletal
  spec) and `:3743` (static spec) — no normalisation on the way in.

A source that writes the package path therefore **never string-equals** the live
object path.

**The skeletal twin has the same pattern**, so a fix must cover both:
`GeometrySkeletalAssetCreate.cpp:137` is the source side
(`Spec.MaterialBindings.Find(Slot)`, package path) and **`:162`** is the live
side (`Material.MaterialInterface->GetPathName()`, object path). *(A citation
handed to this reporter named `:460` for the skeletal live side; `:460` is
actually `Spec.MaterialBindings.Find(SlotName)` inside the creation path, which
resolves the same map but is not the comparison side. `:162` is the
`GetPathName()` line. Corrected here rather than repeated.)*

### What this narrows about the guard family's verification claim

This ticket records the guard family as complete and verified — `#3`: 621/621
tests, `#5`: 37/37. **One guarded field, the model compiler's material slot, is
comparison-broken in a way none of those tests caught.** The claim is not wrong
about coverage count; it is wrong about what the coverage proves.

Note also **which path this fires on**: the SAME-SOURCE rebuild path guarded by
`#2`, not the takeover path `#4`/`#5` added. The `SM_Fish_B` control above proves
the same-source stamp comparison is working; it is the live-object-path
comparison beside it that is not.

### Consequence

**Any asset touched once by `static_mesh.set_material` is permanently in the
"needs overwrite" state.** The signal therefore stops distinguishing real
conflicts from reconciled ones, and `overwrite: true` is mislabelled *"discard it
deliberately"* when nothing is being discarded — the two values denote one asset.
That is the habit that eventually discards work that mattered. The guard itself
is right and valuable: it is what stopped this session silently clobbering
another agent's deliberate material choice, which is exactly the concurrent-agent
case this build is full of.

### Fix

Normalise both sides before comparing — one `FSoftObjectPath` round-trip on each
side, or resolve both to `UObject*`. Separately, when the live value and the
source value resolve to the **same asset**, the guard should not fire at all:
there is no state to discard. Cover the static and skeletal paths together.

**The IN-REVIEW disposition should not pass until this is addressed** — the
diagnostic this ticket introduced currently names a remedy that cannot be carried
out on the field it fires on most often.

**Workaround used on this build:** `overwrite: true`, after checking by eye that
the two paths name the same asset. Safe in that instance and verified before use;
not safe as a habit.

## History
- `#1-silent-recompile-loss` `OPEN` reporter — Reproduced a generated skeleton carrying a preview mesh, per-bone retargeting, and animation curve metadata that its source could not express; recompiling silently reset the state and returned success. Source inspection found no overwrite, clobber, preservation, out-of-band, or unmanaged-state diagnostic across the source compiler family.
- `#2-guard-source-recompile` `IN-REVIEW` developer — Added a shared three-way provenance guard to the skeleton, animation, static-model, and skeletal-model rebuild paths. Added `preview_mesh`, per-bone `retarget`, and `curve` metadata to the skeleton source format. Added failure-direction tests that require the exact diagnostic on unmanaged live state, require a clean recompile when source describes it, and require explicit overwrite to warn before clearing unsupported state. Updated source-format and wiki documentation.
- `#3-scoped-verification` `IN-REVIEW` developer — Full editor build completed with `Result: Succeeded`. Scoped automation for Model, infra, Skeleton, Animation, and the shared source contract completed 621/621 tests successfully with zero failures or skips and both queue-drain and test-exit markers. Ticket remains `IN-REVIEW` for independent tester disposition.
- `#4-takeover-gap-reproduced` `OPEN` reporter — Reproduced that all four guarded rebuild paths ran `PWSRC_RECOMPILE_UNMANAGED_STATE` only for a matching source stamp. Skeleton, animation, static-model, and skeletal-model takeovers therefore skipped the guard entirely; `overwrite=true` could silently discard live state, while a refusal did not name what would be lost.
- `#5-guard-takeover-paths` `IN-REVIEW` developer — Changed the shared guard and all four callers to check same-source rebuilds and takeovers. Takeovers compare current state directly with desired state, no-overwrite refusals include the named losses, and overwrite-permitted takeovers emit `PWSRC_RECOMPILE_UNMANAGED_STATE` as a warning while succeeding. Added failure-direction coverage for skeleton, animation, static-model, and skeletal-model paths. Full build: `Result: Succeeded`; scoped affected groups: 37/37 succeeded, zero failed, zero skipped. Ticket remains `IN-REVIEW` for independent tester disposition.
- `#6-object-vs-package-path-compare` `OPEN` reporter — Additional evidence, not a status change (status left `IN-REVIEW` for the tester; see the dated encounter section in the body for the full readings). Observed while building the Atlantis level, UE 5.8, PinWright at this checkout's HEAD. `PWSRC_RECOMPILE_UNMANAGED_STATE` refused a `.pwmodel` recompile of `/Game/Atlantis/Meshes/SM_Fish_A` quoting live `'/Game/Atlantis/Materials/MI_Fish_Blue.MI_Fish_Blue'` against source `'/Game/Atlantis/Materials/MI_Fish_Blue'` — an OBJECT path against a PACKAGE path, the same material — so the remedy the message names ("Put the state in the source") is impossible to carry out and `overwrite: true` is the only way past. Source-confirmed at HEAD in this tree: the comparison is raw `FString == FString` in `PwSourceRecompileGuard::Check` at `PwSourceRecompileGuard.cpp:63-77` (`CurrentEntry.Value == DesiredValue`, then `CurrentEntry.Value == PreviousValue`) with no `FSoftObjectPath`, no `FTopLevelAssetPath` and no normalisation anywhere in that 151-line file; emitted at `:131-135`. The two sides genuinely differ in shape — live side `GeometryAssetCreate.cpp:133` uses `MaterialInterface->GetPathName()` (object path), source side `:92-94` reads `Spec.MaterialBindings`, a `TMap<FString,FString>` (`GeometryAssetCreate.h:101`) filled verbatim from `Binding.AssetPath` at `PwModelCompiler.cpp:3685` and `:3743` with no normalisation. Skeletal twin has the same split: source side `GeometrySkeletalAssetCreate.cpp:137`, live side `:162` (`GetPathName()`) — a fix must cover both. Controls: `SM_Fish_B`, live material only ever written by `model.compile`, recompiled to a different material (`MI_Fish_Orange`) in the same batch without complaint, proving the guard is not comparing against the source stamp (which agreed) but against the live object path; and the response's own `materialSlotList[0].material` prints the PACKAGE form while the diagnostic prints the OBJECT form, so the two halves of one response disagree on spelling. Fires on the SAME-SOURCE rebuild path (`#2`'s guard), not the takeover path `#4`/`#5` added. Narrowing recorded plainly: `#3` (621/621) and `#5` (37/37) claim the guard family is complete and verified, yet one guarded field — the model compiler's material slot — is comparison-broken and no test caught it. Consequence: any asset touched once by `static_mesh.set_material` is permanently in the "needs overwrite" state, so the signal stops distinguishing real conflicts from reconciled ones and `overwrite:true` is mislabelled "discard it deliberately" when nothing is being discarded. Fix: normalise both sides (one `FSoftObjectPath` round-trip each) or resolve both to `UObject*`, and do not fire at all when both sides resolve to the same asset. **The IN-REVIEW disposition should not pass until this is addressed.** Worked around on this build with `overwrite: true` after checking by eye that the paths name one asset; `encounters` 1 -> 2, `lastSeen` refreshed.
- `#7-refuted-object-vs-package-compare-still-live` `OPEN` verifier — 2026-08-28, rebuilt DLL at plugin HEAD `b79ba53e`, editor pid 14932. **Reopened. `#6` was never addressed and reproduces verbatim on the rebuilt binary.** `#6` itself said "The IN-REVIEW disposition should not pass until this is addressed"; no developer entry since has touched it.
  **Independent reproduction on a fresh fixture**, deliberately not `#6`'s `SM_Fish_A`, so this is a second observation rather than a re-reading of the first. Wrote `Content/PinWrightScratch/SM_PwVerifyMatPath.pwmodel` binding `Mat = "/Engine/EngineMaterials/DefaultMaterial"`; `model.compile` succeeded and `materialSlotList[0].material` read back the PACKAGE form. Bound a different material out of band, `static_mesh.set_material {materialPath:"/Engine/EngineMaterials/WorldGridMaterial"}`, whose own response echoed the OBJECT form `/Engine/EngineMaterials/WorldGridMaterial.WorldGridMaterial`. Then edited the source to name that same material — i.e. carried out exactly what the diagnostic instructs — and recompiled:
  ```
  [PWSRC_RECOMPILE_UNMANAGED_STATE] Recompiling .pwmodel asset '/Game/PinWrightScratch/SM_PwVerifyMatPath'
  would discard state the source does not reproduce: material[Mat] is
  '/Engine/EngineMaterials/WorldGridMaterial.WorldGridMaterial' but the source requests
  '/Engine/EngineMaterials/WorldGridMaterial'; ... Put the state in the source, or pass
  overwrite=true to discard it deliberately.
  ```
  One material, object path against package path, `savedToDisk: false`, and the named remedy is impossible. `#6`'s second datum reproduces in the same response: `materialSlotList[0].material` prints the PACKAGE form while the diagnostic prints the OBJECT form, so the two halves still disagree on spelling.
  **Source at this HEAD, re-verified independently of `#6`'s citations:** `PwSourceRecompileGuard::Check` still compares raw `FString == FString` (`PwSourceRecompileGuard.cpp:66-71` for the desired branch, `:77` for the baseline branch) with no `FSoftObjectPath`, no `FTopLevelAssetPath` and no normalisation anywhere in the 151-line file; the diagnostic is emitted at `:132`/`:134`. Live side is still `Material.MaterialInterface->GetPathName()` — `GeometryAssetCreate.cpp:134` (static) and `GeometrySkeletalAssetCreate.cpp:162` (skeletal) — against the verbatim `Spec.MaterialBindings` value at `GeometryAssetCreate.cpp:93-95` and `GeometrySkeletalAssetCreate.cpp:137-139`, filled unnormalised from the document's `Binding.AssetPath` at `PwModelCompiler.cpp:3959` and `:4017`. Both callers hand the raw string maps straight to `Check` (`GeometryAssetCreate.cpp:283-285`, `GeometrySkeletalAssetCreate.cpp:405-407`). Nothing anywhere resolves the two sides to a common form before firing.
  **One narrowing worth having, not in `#6`:** materials are the *only* asymmetric field. The skeleton field is immune because both its sides call `GetPathName()` on a resolved `USkeleton*` (`GeometrySkeletalAssetCreate.cpp:132` desired against `:153` current). The only normaliser in the whole `PwSource` tree, `NormalizeToObjectPath` (`PwSkeletonRef.cpp:38`), is skeleton-only and is not reached from either material path — so the fix is to apply that same treatment to the material field on both the static and skeletal sides, and to skip the diagnostic entirely when both sides resolve to one asset.
  **Fixture left on disk** at `Content/PinWrightScratch/SM_PwVerifyMatPath.pwmodel` -> `/Game/PinWrightScratch/SM_PwVerifyMatPath`, already in the failing state: edit the source's `materials` line to any other loadable material and `model.compile` reproduces in one call. `encounters` 2 -> 3.
- `#8-normalize-asset-path-comparison` `IN-REVIEW` developer — `#7` confirmed at HEAD: nothing in `#2`/`#5` ever normalized the material field, and every guarded comparison in `PwSourceRecompileGuard::Check` was still raw `FString == FString`. `#7`'s narrowing also confirmed — materials are the only asymmetric field; the skeleton (`.pwmodel` skeletal) and preview-mesh (`.pwskel`) and skeleton (`.pwanim`) fields resolve both sides through `GetPathName()` already, so nothing else needed touching. Fix: `FPwSourceStateEntry` gains `bAssetPath`, set only on the material entries of both geometry state builders (`GeometryAssetCreate.cpp` desired + current, `GeometrySkeletalAssetCreate.cpp` desired + current). `Check` routes those entries' values through one `NormalizeToObjectPath` round-trip (`Utils/PathUtils.h`, the same normalizer `PwSkeletonRef` uses) before all three comparisons — desired, baseline, and the reported text — so a package path and an object path naming one asset are one value; every non-asset value is still compared verbatim. The diagnostic now prints both sides in the same spelling. No `StateVersion` bump: stamps already record the object form (`MakeStateMap(CurrentStaticMeshState(...))`) and normalization is idempotent, so existing baselines stay valid. Regression coverage is behavioural and fails before the fix: `PinWright.Geometry.AssetCreate.ReconciledMaterialPathIsNotUnmanagedState` (`Source/PinWrightGeometry/Private/Tests/Geometry/TestGeometryAssetCreate.cpp`) compiles a slot bound by package path, rebinds it out of band with `UStaticMesh::SetMaterial` exactly as `static_mesh.set_material` does, asserts the guard still refuses while the source names the *other* material, then asserts a recompile whose source names the live material emits no `PWSRC_RECOMPILE_UNMANAGED_STATE` and succeeds without `overwrite`; `PinWright.Geometry.AssetCreate.SkeletalReconciledMaterialPathIsNotUnmanagedState` (`TestGeometrySkeletalAssetCreate.cpp`) covers the skeletal takeover path both ways — same material named by the incoming source is not reported as a loss, a source that declares no material still is. `Docs/pwmodel-format.md` provenance section records the compare-by-asset rule. NOT COMPILED and NOT RUN by this change (wave orchestrator owns build + suite); the two new ids are leaves and collide with no existing id prefix.
