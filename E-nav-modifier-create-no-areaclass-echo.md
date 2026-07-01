---
id: E-nav-modifier-create-no-areaclass-echo
title: "navigation.create_nav_modifier_component doesn't echo the AreaClass/extent it applied; sparse scs.get readback can't confirm NavArea_Null (the constructor default)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [navigation, nav-modifier, readback, scs, docs]
---

# navigation.create_nav_modifier_component doesn't echo the AreaClass/extent it applied; readback can't confirm NavArea_Null

`navigation.create_nav_modifier_component` resolves and applies `areaClass`
(`ModComp->AreaClass = AreaClass`, `Handlers/AI/NavigationHandler.cpp:385`)
and `failsafeExtent` (`ModComp->FailsafeExtent = FailsafeExtent`, `:381`),
but its success result echoes only `componentName` / `blueprintPath` /
`existsAfter` (`:395-397`) plus the standard asset-verification block
(`AddAssetVerification`, `:398`) — it never echoes back the resolved
`AreaClass` or `FailsafeExtent` it just set. So the create call gives the
caller no positive confirmation of *which* nav area landed.

The natural fallback verify surface — `blueprint.scs.get` /
`asset.dump` (`scs.json` / `properties.json`) — is intentionally **sparse**:
it emits only properties that differ from the diff base (see
[B-component-diff-vs-class-cdo](B-component-diff-vs-class-cdo.md),
[B-instanced-subobject-no-cdo-diff](B-instanced-subobject-no-cdo-diff.md)).
`NavArea_Null` is the value the `UNavModifierComponent` **constructor**
installs as the default `AreaClass`, so when the caller sets exactly
`NavArea_Null`, the value equals the CDO and is **correctly omitted** from
every sparse readback. The caller is then left unable to distinguish
"omitted because it equals the default (correct)" from "omitted because the
set silently failed (missing)" — the two look identical in the readback, and
nothing in the create response or the dumps disambiguates them.

This is the same author-then-verify round-trip gap as
[E-get-material-info-no-param-defaults](E-get-material-info-no-param-defaults.md)
(readback confirms the thing *exists* and its *type*, but not the value the
caller just set), surfacing here on the navigation create verb.

**What it should do:** have `create_nav_modifier_component` (mirroring the
sibling `set_nav_area_class`, which already echoes `areaClass` at `:456`, and
the just-landed `configure_nav_link` echo at `:618-631`) add a
`resolvedAreaClass` field to the success result — the actual
`ModComp->AreaClass->GetPathName()` after resolution, including the case
where it fell back to the constructor default because no `areaClass` arg was
passed — plus `failsafeExtent`. That closes the loop at the create call with
no second round-trip and no dependence on sparse-diff behavior.

**Docs angle (`docs/wiki-src/navigation.md`):** the navigation overlay has a
prerequisite/setup-order prelude but **no AreaClass-readback guidance**. It
should state explicitly that (a) `AreaClass` is omitted from `scs.get` /
`asset.dump` whenever it equals the `UNavModifierComponent` constructor
default (`NavArea_Null`), so its absence in a sparse readback is **not**
evidence the set failed; and (b) `set_nav_area_class` targets **placed actors
only** (it resolves an actor in the editor world via `TActorIterator`,
`NavigationHandler.cpp:421-428`) — to author the area class on a Blueprint
*asset*, pass `areaClass` to `create_nav_modifier_component` or use
`blueprint.scs.set_property`, not `set_nav_area_class`.

**Workaround (used in the triggering task):** read engine source
(`NavModifierComponent.cpp`) to learn that `NavArea_Null` is the component's
constructor default, then treat its omission from the sparse dump as
correct-by-default rather than missing; set the value via
`blueprint.scs.set_property` (since `set_nav_area_class` is actor-only) and
re-read.

## History
- `#2-additional-scs-set-property-equals-default` `OPEN` reporter — Same set-then-verify round-trip gap, now on the general `blueprint.scs.set_property` → `blueprint.scs.get` path (not just the navigation create verb), confirming this is the broad sparse-readback ergonomic, not nav-specific. Realism task: build /Game/Blueprints/BP_StreetLamp (Actor) with SCS tree Base(StaticMesh,root) > Pole > LampHead > Bulb(PointLightComponent), then set Bulb Intensity=5000 + AttenuationRadius=1500 via `blueprint.scs.set_property` and read back with `blueprint.scs.get` to confirm. Replay-confirmed live: `blueprint.scs.set_property {componentName:"Bulb", propertyName:"Intensity", propertyValue:5000}` → `{"success":true,"compiled":true,"saved":true}`, but the canonical readback `blueprint.scs.get {blueprintPath:"/Game/Blueprints/BP_StreetLamp", componentClass:"PointLightComponent"}` emits only `"properties":{"AttenuationRadius":1500}` — **Intensity is absent**, because 5000 equals the `UPointLightComponent` CDO default and the properties block is the intentional sparse CDO-diff (per DONE `B-component-diff-vs-class-cdo` / `B-instanced-subobject-no-cdo-diff`). The value IS correctly stored: `property.get {objectPath:"…BP_StreetLamp_C:Bulb_GEN_VARIABLE", propertyName:"Intensity", includeDefault:true}` returns `{"value":5000,"defaultValue":5000,"defaultSource":"class_cdo"}`. Proof it's the equals-default filter and not a silent set failure: re-setting Intensity=5001 (non-default) made it appear — `scs.get` then showed `"properties":{"AttenuationRadius":1500,"Intensity":5001}` — and re-setting it back to 5000 dropped it again. So a verifier doing the natural "scs.get should echo Intensity==5000" check cannot distinguish set-to-default from never-set, and must fall back to `property.get … includeDefault` (which carries `value`+`defaultValue`+`defaultSource` and disambiguates). The fix proposed here (echo applied values at the write call + a wiki readback caveat) generalizes: `blueprint.scs.set_property` likewise echoes only `propertyName`/`message` and not the resolved value, so an equals-default write is unconfirmable from either the setter or the sparse `scs.get`; the `docs/wiki-src/blueprint.md` `scs.get` overlay should state that the `properties` block is sparse-vs-CDO and that a property absent from it may simply equal the class default (use `property.get … includeDefault` to confirm). Distinct from `E-property-get-override-state-ignores-boverride-bit` (that is `property.get`/`list`'s `isOverridden` boolean reading false for `bOverride_*`-gated PPV fields; this is `scs.get` dropping the key entirely).
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean navigation.create_nav_modifier_component task (focus navigation.create_nav_modifier_component, outcome clean). Built /Game/AI/Navigation/BP_NavNoGoZone (Actor) with a BoxComponent root + NavModifierComponent set to NavArea_Null, extent 200/200/200. Friction note (quoted): "scs.get/scs.json/properties.json never surface AreaClass for NavModifierComponent, so the readback couldn't visibly confirm NavArea_Null; had to fall back to reading engine source (NavModifierComponent.cpp) via qmd to learn that NavArea_Null IS the component default, meaning it is correctly omitted from override dumps rather than missing. A re-run of create_nav_modifier_component also errored ALREADY_EXISTS, and navigation.set_nav_area_class only works on placed actors not BP assets, so I used blueprint.scs.set_property as the follow-up." Call-log evidence: of 23 calls, ~5 were spent on the verify-the-area-class loop after the create — two `blueprint.scs.get` readbacks (full + filtered) that did not surface AreaClass, one `create_nav_modifier_component` retry that errored `[ALREADY_EXISTS] Component 'NavModifier' already exists`, one `blueprint.scs.set_property` follow-up to set AreaClass explicitly, and `blueprint.inspect` + two `asset.dump` calls to try to read it back, before the source-dive resolved it. Root cause confirmed at `Handlers/AI/NavigationHandler.cpp`: the create handler assigns `ModComp->FailsafeExtent`/`AreaClass` (:381,:385) but the success result (:395-397) echoes only componentName/blueprintPath/existsAfter, so the only verification path is the intentionally-sparse scs/dump readback, which correctly omits a default-valued AreaClass. Low severity (the asset was authored correctly; the friction is verify-side only). E-/docs because it is a readback-echo ergonomic + a navigation-overlay wiki gap, not a wrong result. Distinct PROCESS angle from the per-finding judge (which saw outcome clean / filed nothing).
- `#3-retriage` `OPEN` triage — Low→Medium: create response omits resolved AreaClass and sparse readback can't confirm the default, forcing a source-dive/property.get fallback, niche nav path.
- `#4-echo-applied-area-class` `IN-REVIEW` developer — Implemented. `navigation.create_nav_modifier_component` now echoes `resolvedAreaClass` (`ModComp->AreaClass->GetPathName()` read off the component template after the write, including the constructor-default fallback when no `areaClass` arg is passed) and `failsafeExtent` (`JsonBuilders::BuildVectorJson(ModComp->FailsafeExtent)`) in its success result, mirroring the additive-echo convention of the sibling `set_nav_area_class` (areaClass) and the already-shipped `configure_nav_link` (link geometry). Closes the author-then-verify loop inline so a caller no longer has to fall back to the intentionally-sparse `scs.get`/`asset.dump` (which correctly omit a default-valued AreaClass) or `property.get … includeDefault`. File: `Source/PinWright/Private/Handlers/AI/NavigationHandler.cpp` (create handler result block). Docs: `docs/wiki-src/navigation.md` gains a "Confirming a NavModifierComponent's AreaClass" section stating (a) AreaClass is omitted from sparse readbacks when it equals the `NavArea_Null` constructor default so its absence is not a failed set (use `property.get … includeDefault` to confirm an instance), and (b) `set_nav_area_class` is placed-actor-only — to author on a BP asset pass `areaClass` to create or use `blueprint.scs.set_property`. Regression test `FNavCreateNavModifierComponentEchoesAreaClassTest` (`PinWright.navigation.create_nav_modifier_component.EchoesResolvedAreaClass`, in `Source/PinWright/Private/Tests/Gameplay/TestAIHandlers.cpp`): creates a BP asset, adds a nav modifier with explicit non-default `NavArea_Obstacle` + extent 200/150/75, asserts the result echoes `resolvedAreaClass` (== applied path) and `failsafeExtent` {x,y,z}, and cross-checks `resolvedAreaClass` against the reloaded component template's `AreaClass->GetPathName()` so it can't pass by reflecting the raw payload. Fails if the echo is reverted. No aspect-cache bump (no asset-dump serialization changed). Not yet compiled/tested — a later phase drives it green.
- `#4-fix` `IN-REVIEW` developer — Reworded (stale line numbers refreshed to current source `:381`/`:385`/`:395-397`/`:421-428`/`:456`/`:618-631`, and the "4-line stub" overstatement corrected — the navigation overlay has a prerequisite prelude but no AreaClass-readback guidance), then fixed as GO. `Handlers/AI/NavigationHandler.cpp`: `create_nav_modifier_component`'s success result now echoes `resolvedAreaClass` (= `ModComp->AreaClass->GetPathName()` read off the component template after resolution, so the no-arg fallback to the `UNavModifierComponent` constructor default `NavArea_Null` is captured exactly) plus `failsafeExtent` (`{x,y,z}` via `JsonBuilders::BuildVectorJson`) — mirroring the additive-echo convention of the siblings `set_nav_area_class` (areaClass echo) and `configure_nav_link` (resolved link-geometry echo). Response-only JSON (no asset-dump aspect), so no AspectVersion bump. Docs: `Docs/wiki-src/navigation.md` gains a "Confirming a NavModifierComponent's AreaClass after authoring" section stating (a) AreaClass is correctly omitted from the sparse `scs.get`/`asset.dump` when it equals the `NavArea_Null` default (use the create echo or `property.get … includeDefault`), and (b) `set_nav_area_class` is placed-actor-only (use create's `areaClass`/`blueprint.scs.set_property` for assets). Regression test `FNavCreateNavModifierComponentEchoesResolvedAreaClassTest` (`PinWright.navigation.create_nav_modifier_component.EchoesResolvedAreaClass` in `Tests/Gameplay/TestAIHandlers.cpp`) authors a transient Actor BP, creates the modifier (a) with an explicit non-default `NavArea_Obstacle` + a 250³ failsafeExtent and (b) with NO areaClass arg, asserting the result echoes the resolved class in both cases — crucially the no-arg fallback to `NavArea_Null` (the exact equals-default value the sparse readback can't confirm) and the failsafeExtent {x,y,z}. Fails if the echo is reverted or sourced from the raw payload instead of the stored template.
