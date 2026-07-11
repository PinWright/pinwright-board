---
id: E-property-get-cdo-hides-asset-level-props
title: "property.get/list on a Blueprint/AnimBlueprint asset ALWAYS resolves to the generated CDO, so editor-asset-level UPROPERTYs (UAnimBlueprint::TargetSkeleton, ParentClass) are unreachable — the wiki never says to use asset.get_dependencies instead"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, property, property-get, property-list, cdo, anim-blueprint, target-skeleton, asset-level-property, discovery]
encounters: 1
lastSeen: 2026-07-11T04:58:19+03:00
---

# property.get can't reach a Blueprint's asset-level UPROPERTYs — the CDO auto-resolve hides them, and nothing points you at the working alternative

`property.get` / `property.list` auto-resolve a Blueprint/AnimBlueprint **asset**
path to its generated-class **CDO** — this is the intended, DONE
`E-property-blueprint-cdo` feature (`ResolveObjectForProperty()` redirects
`IsA<UBlueprint>()` → `GeneratedClass->GetDefaultObject()`). The redirect fires
for **every** path form that lands on the `UBlueprint`, including the full
object path `/Game/.../ABP_Manny.ABP_Manny`.

The consequence, which the wiki never states, is that any UPROPERTY that lives
on the **editor asset object itself** (the `UAnimBlueprint` / `UBlueprint`)
rather than on the generated CDO becomes **unreachable through property.get** —
there is no path form that targets the `UAnimBlueprint` object, because the
redirect always diverts to the CDO. `UAnimBlueprint::TargetSkeleton`,
`ParentClass`, `BlueprintType`, etc. are exactly such asset-level members: they
are not properties of the compiled `AnimInstance` CDO, so a read resolves to
`Default__X_C` and honestly reports the property missing there.

This is **works-but-non-obvious**, NOT a defect (the per-finding Judge replay
confirmed the auto-resolve worked as documented and the error text honestly
names `Default__ABP_Manny_C`; `TargetSkeleton` simply isn't a CDO property).
It is distinct from `E-property-blueprint-cdo` (the CDO redirect feature + its
`_C` resolver gap) and from `E-property-cdo-path-trap-docs` (the `..._C`
generated-class path form that fails to resolve): here a **correct** path form
resolved successfully to the CDO — the property just doesn't live there, and the
redirect makes the object that *does* hold it unaddressable.

The working alternative for the AnimBP → skeleton case is a **different verb**:
`asset.get_dependencies` (returns the referenced Skeleton, e.g. `SK_Mannequin`).
The property wiki doesn't mention that an AnimBP's skeleton / parent-class must
be read this way, so the natural `property.get TargetSkeleton` reach costs one
wasted RPC before the caller pivots to the sibling verb.

## Workaround
Read an AnimBlueprint's skeleton via `asset.get_dependencies` (the referenced
Skeleton asset), not `property.get TargetSkeleton`. More generally, asset-level
UBlueprint/UAnimBlueprint members are not visible through the CDO-resolving
property verbs.

## Fix
Docs only (downstream wiki process, not this audit's job). On
`docs/wiki-src/property.md`, beside the existing CDO-resolution note, add a
one-line caution: property.get/list on a Blueprint/AnimBlueprint asset always
target the **generated CDO**, so **asset-level** UPROPERTYs on the
`UBlueprint`/`UAnimBlueprint` itself (e.g. `UAnimBlueprint::TargetSkeleton`,
`ParentClass`) are not readable through these verbs — use `asset.get_dependencies`
(skeleton/referenced assets) or the relevant `get_*_info` reader instead. Optionally
mirror a one-liner on `docs/wiki-src/anim.md` for the TargetSkeleton case.

severity rationale: impact=Low (pure docs/discoverability — one wasted call with an obvious sibling-verb recovery) × reach=every-session (property.get/list is an every-session readback verb, but this particular asset-level miss is a narrow path) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an AGIR-transfer task (namespace `anim`, outcome `tool_bug` on the unrelated `anim.compile_agir` cached-pose corruption filed as `B-agir-cached-pose-forward-ref-corruption`). Distinct PROCESS angle: to find `ABP_Manny`'s skeleton the agent called `property.get { objectPath: "/Game/Characters/Mannequins/Animations/ABP_Manny.ABP_Manny", propertyName: "TargetSkeleton" }` → `[PROPERTY_NOT_FOUND] Failed to resolve property 'TargetSkeleton' on object /Game/Characters/Mannequins/Animations/ABP_Manny.Default__ABP_Manny_C: Property 'TargetSkeleton' not found` (one wasted RPC — the CDO auto-resolve fired and `TargetSkeleton` is a UAnimBlueprint editor-asset member, absent from the `AnimInstance` CDO), then recovered via the working sibling `asset.get_dependencies` → `.../Meshes/SK_Mannequin`. The per-finding Judge replay ruled this NOT a defect (bare-path→CDO auto-resolve worked as documented per DONE `E-property-blueprint-cdo`; the error honestly names `Default__ABP_Manny_C`), so this is filed as the residual docs/discoverability angle only: the wiki never notes that asset-level UBlueprint/UAnimBlueprint UPROPERTYs are unreachable through the CDO-resolving property verbs and that `asset.get_dependencies` is the way to read an AnimBP's skeleton. Distinct from `E-property-cdo-path-trap-docs` (the `..._C` path-form trap — there a wrong path form fails; here a correct form resolves to the CDO but the property isn't on it). Overlay page = `docs/wiki-src/property.md`. Severity Low (docs; one wasted call, clean recovery).
