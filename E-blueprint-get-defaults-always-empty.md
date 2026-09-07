---
id: E-blueprint-get-defaults-always-empty
title: "`blueprint.get` always returns `defaults:{}` — per-variable CDO defaults are advertised but never surfaced in the readback"
status: IN-REVIEW
severity: High
category: ergonomic
tags: [blueprint, blueprint-get, defaults, cdo, readback, discovery]
encounters: 3
lastSeen: 2026-09-07T00:00:00Z
---

# `blueprint.get` emits an empty `defaults:{}` while the variable defaults provably exist on the CDO

`blueprint.get` returns a `defaults` field — both its own wiki Notes
("parent class, `variables`, `functions`, `events` (plus `metadata`/`defaults`
where tracked)") and the live response (which always includes a `defaults` key)
advertise it as the place to read back per-variable default values. In practice
that field is **always emitted as an empty object `defaults:{}`**, even when the
Blueprint's member variables have non-zero defaults baked onto the class default
object (CDO). So the obvious "read the defaults back to confirm them" step is
unsatisfiable from `blueprint.get` alone — a caller is steered to believe the BP
has no variable defaults when it actually does.

The data is **not lost** — the defaults are present on the CDO and read back
correctly via `property.get {…, includeDefault:true}` — so this is a readback /
discovery gap, not a data-loss bug; hence Low severity. The cost is that the
agent cannot verify defaults through the summary verb that names a `defaults`
field, and must fall back to one `property.get` per variable (or trust the
write-time RPC responses). The attempt agent's friction note pointed exactly
here: *"blueprint.get's `defaults:{}` never surfaces per-variable defaults so
the 1.0/true defaults are only verifiable via the write-time RPC responses, not
the readback."*

This is the same self-contradicting-readback shape the board already corrected
for the `components` field on this very method
(`E-blueprint-get-omits-components-readback-guidance`, IN-REVIEW): that ticket's
own analysis notes the handler registry merge "only folds in
`defaults`/`metadata`/`functions`/`events`" — i.e. the code path that should
populate `defaults` exists in the merge, yet the live response is empty. This
ticket is the `defaults` analogue: a field that is advertised and emitted but
never populated.

**Repro (replayed live against `mcp__editor-automation__call`):**

1. On `BP_Light_Bulb_Basic` (`/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic.BP_Light_Bulb_Basic`)
   with member variables `BrightnessMultiplier` (float, default `1.0`) and
   `bStartLit` (bool, default `true`) — defaults already baked onto the CDO:
   `blueprint.get {path:"…/BP_Light_Bulb_Basic.BP_Light_Bulb_Basic"}` →
   returns full `variables[]` (with `name/type/editable/category/metadata`),
   `functions[]`, `events[]`, a populated top-level `metadata{}` — and
   **`"defaults":{}`** (empty) as the final field.
2. Control proving the defaults exist and the CDO read path is fine:
   `property.get {objectPath:"…/BP_Light_Bulb_Basic.BP_Light_Bulb_Basic", propertyName:"BrightnessMultiplier", includeDefault:true}` →
   `{"propertyName":"BrightnessMultiplier","value":1,…,"defaultSource":"class_cdo","defaultValue":1}`.
3. Same for the bool:
   `property.get {…, propertyName:"bStartLit", includeDefault:true}` →
   `{"propertyName":"bStartLit","value":true,…,"defaultSource":"class_cdo","defaultValue":true}`.

So both variables hold their requested defaults (`1.0` / `true`) on the CDO, yet
`blueprint.get`'s `defaults` map is empty.

**Workaround:** read per-variable defaults with `property.get {objectPath:<BP
asset path>, propertyName:<var>, includeDefault:true}` (returns `defaultValue` /
`defaultSource:"class_cdo"`), one call per variable. Treat `blueprint.get`'s
`defaults` field as unreliable for verification.

**Fix:** either (preferred) populate `blueprint.get`'s `defaults` map with each
member variable's CDO default (e.g. export the property value off the
generated-class CDO the same way `property.get`/`set_default` do) so the
advertised field is real; or, if surfacing CDO defaults in the summary is out of
scope, drop the `defaults` field from both the response and the wiki Notes and
route default readback to `property.get` (mirroring the `components` →
`blueprint.scs.get` routing fix in
`E-blueprint-get-omits-components-readback-guidance`). A regression test should
fail if `defaults` is emitted-but-always-empty while a member variable has a
non-zero CDO default.

## Notes
Distinct from `B-add-variable-default-value-ignored` (IN-REVIEW) — that ticket
is about `add_variable` *dropping* the supplied `defaultValue` so it never lands
on the CDO; here the defaults **do** land (confirmed via `property.get`) and the
defect is purely that `blueprint.get` does not surface them. Distinct from
`E-blueprint-get-omits-components-readback-guidance` (IN-REVIEW) — same method,
same "advertised-field-never-emitted" shape, but a different field
(`components` vs `defaults`) and a different routing target
(`blueprint.scs.get` vs `property.get`).

## History
- `#1-initial-repro` `OPEN` reporter — Struggle-audit of a `blueprint.get`-seeded task (configure `BP_Light_Bulb_Basic`: add `BrightnessMultiplier` float default 1.0 + `bStartLit` bool default true, set CDO defaults, compile, read back). Replayed live against `mcp__editor-automation__call`: `blueprint.get` returns full `variables`/`functions`/`events`/`metadata` but `"defaults":{}` (empty), while `property.get {propertyName:"BrightnessMultiplier", includeDefault:true}` reads `defaultValue:1` / `defaultSource:"class_cdo"` and `property.get {propertyName:"bStartLit", includeDefault:true}` reads `defaultValue:true` — so the defaults are demonstrably on the CDO yet absent from the `blueprint.get` readback. Advertised both in the wiki Notes ("plus `metadata`/`defaults` where tracked") and by emitting the empty `defaults` key. Low severity (data reachable via `property.get`; cost is one corrective call per variable). Not a dup of `B-add-variable-default-value-ignored` (that drops the default; here it lands) nor `E-blueprint-get-omits-components-readback-guidance` (same method/shape, different field `components` vs `defaults`).
- `#2-populate-defaults-from-cdo` `IN-REVIEW` developer — Implemented the ticket's **preferred** fix (populate, not drop): `blueprint.get`'s `defaults` map now carries each member variable's class-default-object value. Chose populate over the drop-and-route alternative because, unlike the components sibling (whose data genuinely lives on a different verb, `blueprint.scs.get`), per-variable defaults belong on the same per-variable summary `blueprint.get` already enumerates — the direct precedent is `E-get-material-info-no-param-defaults` (IN-REVIEW), which populated a per-parameter `defaultValue` on the summary verb rather than shunting it elsewhere; and the CDO-export machinery is already module-public, so this reuses an existing pattern rather than reinventing one. CODE: added `BuildBlueprintDefaults(UBlueprint*)` in `Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp` — for each `Blueprint->NewVariables` entry it finds the `FProperty` on `GeneratedClass` and exports the value off the CDO via the shared `PINWRIGHT_API ExportPropertyToJsonValue(Container, Property)` (`Utils/PropertyExport.h`, the same export `property.get` uses, so values match `property.get {includeDefault:true}` / `defaultSource:"class_cdo"`); a variable not yet compiled onto the class (no CDO property) is omitted rather than reported wrong. Wired it into `BuildBlueprintSnapshot` so the snapshot now always emits `defaults`; the handler's registry merge at `BlueprintInfoHandler.cpp:90` only folds in the empty registry `defaults` when the snapshot lacks one, so the live CDO values take precedence. Updated the `blueprint.get` registration summary at `BlueprintInfoHandler.cpp:52` to name the `defaults` map, and the `### blueprint.get` overlay in `Docs/wiki-src/blueprint.md` to describe it truthfully (CDO values, same shape as `property.get includeDefault`, uncompiled vars omitted). TEST: `FBlueprintGetDefaultsReflectCdoTest` (`PinWright.blueprint.get.DefaultsReflectCdo`) in `Source/PinWright/Private/Tests/Blueprint/TestBlueprintHandlers.cpp` — adds a float var `Brightness` default `1.5` via `blueprint.add_variable` (lands on the compiled CDO), then asserts `blueprint.get`'s `defaults.Brightness == 1.5`; pre-fix `defaults` was the always-empty registry object, so reverting `BuildBlueprintDefaults` fails it.
- `#3-inherited-overrides-not-covered-by-the-fix` `IN-REVIEW` WEAPONS-critic — Additional evidence, **status deliberately unchanged** — this is not a return, and `#2`'s fix is not disputed for the case it was written for. Measured in a WEAPONS critic review round 3: `blueprint.get {assetPath:"/Game/FPS/Weapons/BP/BP_Weapon_AR"}` still returns **`defaults: {}`** on a Blueprint with **19 overridden inherited properties**. That is consistent with `#2` rather than contradicting it: `BuildBlueprintDefaults` iterates `Blueprint->NewVariables`, i.e. the class's **own** declared variables, and a child Blueprint that overrides properties inherited from its parent declares none — so the map is correctly empty by the implementation and still empty in the caller's experience. The gap this ticket describes ("the obvious read-the-defaults-back step is unsatisfiable from `blueprint.get` alone") therefore survives the fix for the exact asset shape where it bites hardest: a weapon subclass whose entire configuration *is* inherited-property overrides. Ask, for whoever tests `#2`: verify against a **child** Blueprint as well as a class with own variables, and decide explicitly whether `defaults` covers overridden inherited properties. If it should, the source is the same CDO export already in use, walked over the parent chain's properties rather than `NewVariables`; if it should not, the wiki Notes need to say `defaults` is class-own-variables-only, because an empty map on a Blueprint with 19 live overrides reads as "no defaults", not as "no *own* defaults". Related, filed the same round on the same call: `B-blueprint-get-omits-inputkey-events` (High) — the same response also omits every `K2Node_InputKey` entry point from `events[]`, so a caller reading `blueprint.get` on this asset sees neither its input bindings nor its overridden defaults. No plugin source was opened for this entry; the `NewVariables` reading is quoted from `#2`'s own implementation note.
- `#4-inherited-omission-produced-two-false-claims` `IN-REVIEW` WEAPONS-critic — Additional evidence from a WEAPONS critic review round 5. **Status deliberately unchanged at `IN-REVIEW`** — this is not a return, and `#2`'s fix is not disputed for the case it was written for. **Severity raised Low → High**; `encounters` 2 → 3. `#3` predicted the gap would survive `#2`'s fix on a child Blueprint whose configuration is entirely inherited-property overrides; it did, and this round measured what it costs. `blueprint.get {path:"/Game/FPS/Weapons/BP_Weapon_AR"}` returns `"defaults": {}` **and** `"variables": []`, while that CDO carries **78 reflected values and 17 stored deltas** — `MuzzleOffset (55,0,5)`, a 20-entry `RecoilPattern`, `SpreadMin 0.22`, `SpreadMax 4.6`, `ReloadTime 2.1`, `WeaponName "M4A1 Carbine"`, `TracerVFX`, and more. On `BP_Weapon_Pistol` the same call returns 3 entries — that Blueprint's own `Slide*` variables only. That contrast is `#2`'s `NewVariables` walk behaving exactly as implemented, and it is precisely why the empty map reads as authoritative rather than as a known omission: the field is populated when the class declares its own variables, so an empty one looks like a measurement, not a scope limit. **Measured consequence, not a prediction — it produced TWO false claims in one shipped build report** (`Docs/fps/reports/weapons-build-06.md`, this project). A CDO sweep built on `blueprint.get` concluded that `BP_Weapon_Pistol` "never held a stored `PenetrationThickness` delta" — byte scans of the `.uasset` prove it did (`PenetrationThickness` FName ×1 and `double 16.0` ×1 at 124,271 B; both ×0 in the 124,069 B file written after, cross-referenced from `B-blueprint-set-default-not-persisted #4`) — and that "the AR stores no gameplay deltas", when it stores 17, present in the pre-build blob. Both conclusions were reached by reading an empty map as an empty CDO, which is exactly the failure mode this ticket describes, now with a shipped artifact behind it rather than an argument. **Working surfaces, for anyone doing a CDO sweep on this kit:** `asset.dump`'s `properties.json`, and `property.get {objectPath:"<pkg>.Default__<BP>_C", includeOverrideState:true}`, whose `isOverridden` flag answers the question directly. **Ask, sharpened from `#3`:** either emit inherited members in `defaults`, or emit an explicit marker — `omitsInherited: true`, or a separate `inheritedDefaults` map — so a sweep cannot silently read `{}` as "this CDO has no defaults". A silent empty map is the one shape that cannot be defended, because it is indistinguishable from the true-empty case. **Severity rationale for the Low → High raise:** `#1` set Low on "the data is not lost, it is one `property.get` away", which holds only for a caller who already doubts the result. This round shows the defect's normal outcome is a caller who does not doubt it and publishes wrong facts — impact class "silent wrong data on a normal path (the caller trusts a result that is a lie and builds on it)" — on `blueprint.get`, an every-session verb. **Related, filed the same round:** the CDO-addressing half of the same sweep failure is on `E-property-cdo-path-trap-docs #3`; together they are why CDO sweeps in this project keep coming out wrong — the natural class path does not resolve, and the summary verb that would have answered instead returns empty. No plugin source was opened for this entry; the `NewVariables` reading is quoted from `#2`'s own implementation note, and the byte-scan figures are cross-referenced, not re-derived.
