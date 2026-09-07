---
id: E-property-cdo-path-trap-docs
title: "property wiki doesn't warn that the `..._C` (UClass/generated-class) objectPath is the trap that returns PROPERTY_NOT_FOUND — callers map \"class default object\" to `_C` and waste a call"
status: OPEN
severity: Medium
category: ergonomic
tags: [docs, property, property-get, property-list, cdo, class-resolution, c-suffix, readback, discovery]
encounters: 3
costly: 1
lastSeen: 2026-09-07T00:00:00Z
---

# `property` wiki doesn't warn against the `..._C` generated-class objectPath (the trap), only documents the two forms that work

`property.get` / `property.list` accept a Blueprint CDO three ways: the **bare
asset path** `/Game/.../BP_Foo` (auto-resolves to CDO via the `E-property-blueprint-cdo`
redirect), and the **explicit `Default__..._C` inner-object path**
`/Game/.../BP_Foo.Default__BP_Foo_C` (targets the CDO directly). The **third,
most natural-looking form — the `..._C` generated-class path**
`/Game/.../BP_Foo.BP_Foo_C` — does **not** resolve: it lands on the
`UClass`/`UBlueprintGeneratedClass` metatype (which doesn't hold member/CDO
values) and returns `[PROPERTY_NOT_FOUND] ... Property 'X' not found`.

The `docs/wiki-src/property.md` overlay documents the two forms that work but
**never warns against the form that fails**:
- Line 7: *"…resolve `UBlueprint` paths to the Class Default Object (CDO)… The
  explicit `Default__` inner-object path still works when you need to target the
  CDO yourself."* — names the bare path and the `Default__` path, both correct.
- The "Inspecting Blueprint CDO defaults — worked example" (lines 96-109) shows
  **only** the bare-asset-path form. No example, and no caution, for the `..._C`
  generated-class path.

So a caller who reads "read back from the **class default object**" (or who knows
that `ui.create_hud` / `ui.activatable_push` *require* the `_C` suffix on their
class params — see `E-create-hud-requires-c-suffix-class-path`,
`E-activatable-push-requires-c-suffix`) reasonably constructs the `..._C` path for
`property.get`, hits PROPERTY_NOT_FOUND, and only then falls back to the bare or
`Default__` form. The `_C`-suffix meaning is **inverted between namespaces** (a
class param in `ui.*` demands it; an objectPath in `property.*` is broken by it),
which is exactly the confusion that produces the wasted call. The overlay should
say so explicitly.

This is the **docs/discoverability** angle, distinct from the resolver code-fix on
`E-property-blueprint-cdo` (add a `Cast<UClass>`/`UBlueprintGeneratedClass` →
`GetDefaultObject()` redirect so the `_C` path *just works*). That fix is
"DONE pending the resolver extension" and may sit unlanded; a one-line overlay
caution is the cheap interim mitigation that would have prevented all four wasted
calls already on record (`E-property-blueprint-cdo` histories `#3`/`#4`/`#5`/`#6`).
Once the resolver redirect lands, this docs ticket can be closed as superseded.

**Fix (docs only; downstream wiki process):** In `docs/wiki-src/property.md`, add a
one-line caution beside the existing CDO-resolution note (line 7 / the worked
example): the bare asset path `/Game/.../BP_Foo` and the explicit
`/Game/.../BP_Foo.Default__BP_Foo_C` path both resolve to the CDO, but the
`/Game/.../BP_Foo.BP_Foo_C` **generated-class** path does NOT — it targets the
`UClass` and returns `PROPERTY_NOT_FOUND` for member/CDO properties. Prefer the
bare asset path. (When the `E-property-blueprint-cdo` redirect lands, drop this
note.)

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of seed `misc.set_net_update_frequency` (namespace `misc`, 23 calls, outcome ergo; per-finding judge filed/aggregated the resolver angle on `E-property-blueprint-cdo #6`). Realism task: build a networked "ammo crate" replicated Actor Blueprint end to end (`/Game/Network/Pickups/BP_AmmoCrate`, parent `Actor`) — `blueprint.create` + `misc.set_replication` + `misc.set_net_update_frequency {frequency:30,minFrequency:5}` + `misc.configure_net_cull_distance` + `misc.create_replicated_variable` + `misc.create_rpc` + `blueprint.compile` (UpToDate, clean) + `asset.save`, all `ok:true`; 8 wiki-navs up front, write phase had zero retries. The **only** friction was the final readback: the task text literally said "read back the NetUpdateFrequency property from the Blueprint's **class default object**," which the agent mapped to the `_C` path — `property.get { objectPath: "/Game/Network/Pickups/BP_AmmoCrate.BP_AmmoCrate_C", propertyName: "NetUpdateFrequency" }` → `[PROPERTY_NOT_FOUND] ... Property 'NetUpdateFrequency' not found` (one wasted call) — then recovered with `property.get { objectPath: "/Game/Network/Pickups/BP_AmmoCrate.Default__BP_AmmoCrate_C", ... }` → `value:30`. Verbatim friction note: "property.get on the task-suggested objectPath '...BP_AmmoCrate_C' (the UClass) failed with PROPERTY_NOT_FOUND; the CDO is at '...Default__BP_AmmoCrate_C' and that returned 30. The property.get wiki page says asset CDOs are valid targets but never documents the 'Default__' CDO-path convention, so I relied on UE knowledge rather than the wiki to recover." Verified against `docs/wiki-src/property.md`: the overlay DOES document the `Default__` path (line 7) and the auto-resolving bare path (worked example, lines 96-109), but never **warns against** the `..._C` generated-class path that the agent actually tried — the trap form is undocumented. This is the durable docs/discovery angle (overlay page = `docs/wiki-src/property.md`); the resolver code-fix that would erase the trap entirely is on `E-property-blueprint-cdo` (DONE pending). Cross-refs the inverse `_C`-suffix-required ergonomic gaps `E-create-hud-requires-c-suffix-class-path` / `E-activatable-push-requires-c-suffix`, which prime callers to expect `_C` and walk straight into this trap. Severity Low (pure docs/discoverability friction; one wasted call with an obvious recovery), but note `property.get`/`property.list` is an every-session readback verb and this exact trap has now bitten four independent asset families (Lyra widget, character, GameMode, networked Actor).
- `#2-liveness` `OPEN` reporter — Still observed (struggle-audit, `BP_TreasureChest` interaction build): `property.get { objectPath:"/Game/Interactables/BP_TreasureChest.BP_TreasureChest_C", propertyName:"bIsLocked" }` → `[PROPERTY_NOT_FOUND] Failed to resolve property 'bIsLocked' on object /Game/Interactables/BP_TreasureChest.BP_TreasureChest_C`, self-corrected on the next call to the `Default__BP_TreasureChest_C` CDO path (`bIsLocked` → false). One wasted call, obvious recovery. Reconfirms the `..._C` generated-class trap (now also in the `interaction` namespace, while verifying interaction CDO defaults); same docs gap (`docs/wiki-src/property.md`), no new angle.
- `#3-fifth-family-and-the-method-page-carries-nothing` `OPEN` WEAPONS-critic — Fifth independent asset family for the same `..._C` trap, plus a **precise location for the docs gap that `#1` stated approximately**. `encounters` 2 → 3, **severity raised Low → Medium**, status unchanged at `OPEN`. Measured in a WEAPONS critic review round 5: `property.get {objectPath:"/Game/FPS/Weapons/BP_Weapon_AR.BP_Weapon_AR_C", propertyName:"PenetrationThickness"}` → `PROPERTY_NOT_FOUND` — the path resolves the `UClass`, which holds no such property — while `/Game/FPS/Weapons/BP_Weapon_AR.Default__BP_Weapon_AR_C` works. Same leaf-property form as `E-property-blueprint-cdo` `#3`/`#5`/`#6`, now on a weapon subclass (vs Lyra widget, character, GameMode, replicated Actor). **New and re-checkable:** `#1` located the gap on the overlay source `docs/wiki-src/property.md`, whose line 7 does document the `Default__` form. The page a caller actually reads is the generated **method** page, and there the note is absent entirely. Verified against this checkout's generated wiki at `X:/src/unreal/EAContentExamples58/Saved/PinWright/wiki/`: `property.get.md` contains **zero** occurrences of `Default__`, and its whole `objectPath` documentation is line 11, `- objectPath (path, required): Object path or actor name`; the `Default__` sentence lives one level up, on the namespace page `property.md` line 11. So a caller who goes straight to the method page — the normal path once the method name is known — gets no CDO guidance at all, neither the working form nor a caution against the failing one. That makes the fix cheaper and more specific than `#1` framed it: the caution belongs beside the `objectPath` parameter on the `property.get` / `property.list` **method** overlays, not only on the namespace page, which already carries the positive half. **Severity raise Low → Medium** by the board's own reach modifier ("if the affected method runs in almost every session, bump up one level"): impact stays pure friction, reach is an every-session readback verb, and this is now the fifth asset family. **Why it cost more than one call this round:** together with `E-blueprint-get-defaults-always-empty` (whose `defaults` map returns `{}` on this same Blueprint) the two defects left a CDO sweep with no surface reporting the truth, and the sweep published two false claims in `Docs/fps/reports/weapons-build-06.md`. The measurement is recorded on `E-blueprint-get-defaults-always-empty #4`; noted here because the pair, not either alone, is what produced the wrong report. No plugin source was opened; the wiki line numbers above are read off the generated pages in this checkout and are re-checkable there.
