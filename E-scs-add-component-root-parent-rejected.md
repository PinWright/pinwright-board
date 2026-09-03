---
id: E-scs-add-component-root-parent-rejected
title: "blueprint.scs.add_component rejects parentComponentName='RootComponent' (resolver only matches SCS nodes, not the native root)"
status: OPEN
severity: Low
category: ergonomic
tags: [scs, blueprint, add_component, root, discovery, error-diagnostics]
---

# `blueprint.scs.add_component` rejects `parentComponentName: "RootComponent"`

Passing the obvious `parentComponentName: "RootComponent"` (the natural first
guess for "attach under the root") fails with `SCS_ERROR Parent component not
found: RootComponent`. `FSCSHandlers::AddSCSComponent` resolves the parent name
by walking **only** `SCS->GetAllNodes()` and matching `GetVariableName()`
(`PinWright_SCSHandlers.cpp:668-687`, error emitted at `:683`); the actor's
native/inherited `RootComponent` is a CDO template, not an SCS node, so it is
never a match. The non-success result is wrapped with the default `SCS_ERROR`
code by `SendSCSResult` (`SCSHandler.cpp:42`). The error names neither the valid
SCS node options nor the omit-to-attach-at-root path, so the natural first guess
errors with a non-diagnostic message.

Distinct from the IN-REVIEW sibling `E-scs-add-component-root-promotion-undocumented`,
which documents what UE does on the *omit* path (promote first scene component,
reparent the rest). This ticket is the inverse: explicitly *naming* the root is
rejected, not silently reparented.

**Workaround:** omit `parentComponentName` (or pass empty) — that path attaches
under the native root correctly (confirmed by `blueprint.scs.get`). The param
doc (`SCSHandler.cpp:85`) does already say "Existing SCS node name … omit … to
attach as a child of the root," so a doc-reading caller can avoid it.
**Fix:** resolve `"RootComponent"` (and the actual native root component name) to
the attach-at-root path instead of erroring; or make the error name the valid
parent options so the recovery is obvious.

## History
- `#1-initial-repro` `OPEN` reporter — Verified against source: `AddSCSComponent` resolves `parentComponentName` only over `SCS->GetAllNodes()` (`PinWright_SCSHandlers.cpp:668-687`); the native root is not an SCS node, so `parentComponentName:"RootComponent"` returns `SCS_ERROR Parent component not found: RootComponent` (wrap `SCSHandler.cpp:42`). Session evidence: across ~32 gate migrations, `add_component {…, parentComponentName:"RootComponent"}` errored; omitting it parented under the root correctly. Filed E-/Low (not B-): the omit path works and the param doc hints at it, so this is a discoverability/error-diagnostics gap, not a functional defect. Distinct from `E-scs-add-component-root-promotion-undocumented` (omit-path promotion docs).
- `#2-inherited-non-root-parent-has-no-workaround` `OPEN` WEAPONS — Same resolver, strictly worse case, and the mitigation this ticket rests on does not apply. `AddSCSComponent` walking only `SCS->GetAllNodes()` also excludes **inherited SCS nodes from a parent Blueprint**, not just the native root. Measured on `/Game/FPS/Weapons/BP_Weapon_AR` (child of `BP_WeaponBase`): `blueprint.scs.get` lists `WeaponMesh` plainly as `source: "inherited-scs", inheritedFrom: /Game/FPS/Weapons/BP_WeaponBase`, yet `add_component {componentName:"MagazineMesh", parentComponentName:"WeaponMesh"}` returns `SCS_ERROR Parent component not found: WeaponMesh`, and `reparent_component {componentName:"MagazineMesh", newParentName:"WeaponMesh"}` returns the matching `SCS_ERROR New parent not found: WeaponMesh`. So the readback advertises a parent the writers cannot address — the caller is told the node exists and then told it does not. **Why this is not merely ergonomic like the RootComponent case:** this ticket is held at Low because "omit `parentComponentName` — that path attaches under the native root correctly". For an inherited *non-root* parent there is no such escape: omitting the parameter attaches under the inherited root (`WeaponRoot`), which is a different and wrong place, and no other parameter reaches the intended node. There is no RPC path to parent a new component under an inherited non-root component at all. Consequence: the SCS cannot express the hierarchy, so the attachment has to be re-done in the graph — I shipped `call K2_AttachToComponent(Target: $MagazineMesh, Parent: $WeaponMesh, LocationRule/RotationRule/ScaleRule: EAttachmentRule::SnapToTarget)` in each child's ConstructionScript, leaving the SCS tree permanently misleading (the component reads as a child of `WeaponRoot` in every SCS readback and in `scs.txt`, but is a child of `WeaponMesh` at runtime and in the editor viewport). Two weapons now carry that divergence (`MagazineMesh` on `BP_Weapon_AR`, `SlideMesh` on `BP_Weapon_Pistol`). Suggest the fix here widen to resolve the parent name across the inherited SCS chain (the same set `blueprint.scs.get` already enumerates and labels `inherited-scs`), not only `RootComponent`, and that severity be reconsidered above Low for the inherited-non-root case since it has no workaround.
