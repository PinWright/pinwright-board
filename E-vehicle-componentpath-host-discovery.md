---
id: E-vehicle-componentpath-host-discovery
title: "vehicle.* wheel-setup/suspension RPCs give no steer for finding the wheeled-vehicle Blueprint host — agent whole-project-dumps 364 BPs instead of one asset.search parentClassPath"
status: OPEN
severity: Low
category: ergonomic
tags: [vehicle, chaos-vehicle, componentPath, discovery, asset-search, docs]
encounters: 5
lastSeen: 2026-06-24T06:10:13Z
---

# The `vehicle.*` component-targeting RPCs document `componentPath` but offer no affordance for *finding* the host Blueprint — agents fall back to a whole-project Blueprint dump

`vehicle.set_wheel_setup`, `vehicle.remove_wheel_setup`, and
`vehicle.set_suspension` all operate on a `componentPath` pointing at a
`UChaosWheeledVehicleMovementComponent` living on a wheeled-vehicle
Blueprint CDO (`/Game/.../BP_Car.BP_Car_C:VehicleMovement`-style path). The
task that surfaced this required, as step 4, "find a
`UChaosWheeledVehicleMovementComponent` on a
ChaosWheeledVehicle/WheeledVehiclePawn Blueprint CDO; if none exists, note
that you could not locate one." There is no documented path from "I have a
vehicle namespace and need its component host" to "here is the
`componentPath`" — so the agent reverse-engineered the discovery and chose
the most expensive route.

The right tool already exists and is shipped+verified:
[`F-asset-search-native-subclass`](F-asset-search-native-subclass.md) (DONE)
added `asset.search parentClassPath="/Script/<Module>.<NativeClass>"`, which
recursively matches Blueprint assets whose `ParentClass`/`GeneratedClass`
descends from a native class. A single
`asset.search parentClassPath="/Script/ChaosVehicles.WheeledVehiclePawn"`
would have answered "does any Blueprint host a wheeled-vehicle movement
component?" in one call (returning either the host BP or an empty list).

Instead, with no steer toward it, the agent:

- `asset.list` of **all** `/Game` Blueprints → **364 results** → an
  over-limit **494 KB** payload spilled to a file under
  `Saved/EditorAutomation/HttpResponses/` that had to be grepped.
- `system.inspect.search_classes query=ChaosWheeledVehicle` (native catalog
  only — finds the component class, not its asset hosts).
- `asset.list filter.class=WheeledVehiclePawn` (short name) → 0.
- `system.inspect.search_classes parentClass=Pawn` (native catalog).
- `asset.list filter=/Script/ChaosVehicles.WheeledVehiclePawn` (full path) → 0.

That is **5 discovery calls** (one of them a 494 KB spill-to-file) to reach a
"component host not found" conclusion that one
`asset.search parentClassPath=...` call expresses directly — and the
multiple confirmation queries were driven by the agent's own uncertainty,
because nothing told it which query is authoritative for "find the BP that
hosts this component."

## What it should do

Capability is present; the gap is discovery/usage guidance. Cheapest fix
ships in docs. Amend `docs/wiki-src/vehicle.md` (currently a single sentence)
to add a short "finding the vehicle component host" note on the
`componentPath`-taking methods:

- To locate the wheeled-vehicle Blueprint, run
  `asset.search parentClassPath="/Script/ChaosVehicles.WheeledVehiclePawn"`
  (recursive native-subclass match — one call, no whole-project Blueprint
  dump). An empty result is the authoritative "no host exists" answer for the
  documented component-not-found path.
- The `componentPath` is then
  `<BlueprintPath>.<BlueprintName>_C:<ComponentName>` (read the component
  name from `asset.dump` / `actor.get_components` on the BP CDO if it is not
  the conventional `VehicleMovement`).
- Steer away from `asset.list` with no/short class filter for this purpose —
  it both fires the short-name ensure path and returns an over-limit payload
  on content-heavy projects.

Optionally name the same `asset.search parentClassPath` recipe in the
`vehicle.set_wheel_setup` / `vehicle.set_suspension` /
`vehicle.remove_wheel_setup` per-method wiki pages where `componentPath` is
described.

This is the orthogonal layer to
[`E-property-route-no-component-path-discovery`](E-property-route-no-component-path-discovery.md)
(OPEN): that ticket is about discovering a component **subobject name on an
already-known actor** via the `property.*` route; this one is about
discovering **which asset hosts the component at all**, before any
`componentPath` can be built. Distinct from `F-vehicle-chaos-typed` (DONE),
which implemented the verbs but did not document how to find their host.

**Workaround:** Use
`asset.search parentClassPath="/Script/ChaosVehicles.WheeledVehiclePawn"`
(or the relevant native vehicle base) to find the host Blueprint in one call
instead of `asset.list`-dumping every `/Game` Blueprint.

## History
- `#1-initial-audit` `OPEN` reporter — Process-friction audit of a Chaos-vehicle wheel/suspension authoring task (namespace `vehicle`, outcome clean). Steps 1-3 (wheel-asset CRUD + fast-path property bumps) were smooth; the friction was entirely in step 4's host discovery. Friction note (verbatim): *"Minor friction only in locating the (nonexistent) vehicle Blueprint: asset.list returned an over-limit 494KB payload spilled to a file that I had to grep, and I ran extra confirmation queries (short-name filter, full-path filter, and a parent-restricted class search) to be sure no wheeled-vehicle component existed before concluding step 4 was impossible."* Call-log shows 5 discovery calls (`asset.list` all /Game BPs → 364 results / 494 KB spill, `search_classes ChaosWheeledVehicle`, `asset.list class=WheeledVehiclePawn`→0, `search_classes parent=Pawn`, `asset.list /Script/ChaosVehicles.WheeledVehiclePawn`→0) to reach "host not found." The shipped+verified `asset.search parentClassPath` (`F-asset-search-native-subclass`, DONE) answers this in one call, but `vehicle.md` (single sentence) never points to it. Proposed `docs`-tagged wiki steer in `docs/wiki-src/vehicle.md` (and optionally the per-method `componentPath` pages). Dedup: ripgrep across OPEN/closed — `E-property-route-no-component-path-discovery` is the subobject-name-on-known-actor layer (distinct); `F-asset-search-native-subclass`/`F-search-api-native-uclasses` are the capabilities this proposes documenting (not the vehicle discovery steer); `F-vehicle-chaos-typed` implemented the verbs but not the host-finding guidance; no vehicle/componentPath-discovery doc ticket exists.
- `#2-additional-cdo-path-misuse` `OPEN` reporter — Additional evidence (distinct manifestation of the same `componentPath`/no-steer gap; seed `vehicle.create_wheel_asset`, finding about `vehicle.set_suspension`). A Chaos rally-car authoring task whose step 3 said *"set SuspensionMaxRaise/MaxDrop/SpringRate/SuspensionDampingRatio ... Use the vehicle suspension tooling for this on the rear wheel asset."* The agent reasonably pointed `vehicle.set_suspension`'s `componentPath` directly at the **standalone wheel-asset CDO** (there is no vehicle Blueprint host in play at all — just two loose wheel assets), trying both `/Game/Vehicles/Rally/BP_RallyWheel_Rear.BP_RallyWheel_Rear_C` and `...Default__BP_RallyWheel_Rear_C`, and got (replay-confirmed verbatim via `mcp__editor-automation__call`): `[COMPONENT_NOT_FOUND] Failed to resolve component: /Game/Vehicles/Rally/BP_RallyWheel_Rear.BP_RallyWheel_Rear_C`. `set_suspension` strictly resolves `componentPath` → `UChaosWheeledVehicleMovementComponent` and walks `WheelSetups[idx].WheelClass` (`ResolveWheeledComp` + the loop in `ChaosVehicleHandler.cpp`), so a wheel-asset CDO can never satisfy it — correct behavior, but the error gives zero steer that, for suspension on a *standalone* wheel asset, the right tool is the sibling `vehicle.set_wheel_asset_property` (which the agent fell back to and succeeded with for all four fields). Two doc facets to add alongside the existing host-find steer: (1) `vehicle.set_suspension`/`set_wheel_setup`/`remove_wheel_setup` `componentPath` is a **movement-component host** path, NOT a wheel-asset CDO path; (2) to edit suspension fields on a standalone wheel asset (no host vehicle), use `vehicle.set_wheel_asset_property` (SuspensionMaxRaise/SuspensionMaxDrop/SpringRate/SuspensionDampingRatio are all valid wheel CDO props) or pass them at `vehicle.create_wheel_asset` time. Dedup: ripgrep OPEN+closed — same `vehicle.*` `componentPath` no-steer root cause as `#1`; not the `property.*` subobject layer (`E-property-route-no-component-path-discovery`); `F-vehicle-chaos-typed` (DONE) defined this exact `componentPath` contract for `set_suspension` by design, so no regression — appended here rather than a new file.
- `#4-asset-dump-emits-non-roundtrippable-componentpath` `OPEN` reporter — Additional evidence: same `scs.get`→`vehicle.*` round-trip mismatch as `#3`, reproduced through the **`asset.dump` emitter** (vs `#3`'s `blueprint.scs.get`) and with the seed being `vehicle.set_wheel_setup` itself this time (task outcome `done`, so ERGONOMIC). A 4-wheel off-road-buggy authoring task: agent built `/Game/Vehicles/BP_Buggy` (`WheeledVehiclePawn`, native `VehicleMovementComp` of class `ChaosWheeledVehicleMovementComponent`), then ran `asset.dump` to find the movement-component path. The dump's `scs.json`/`scs.txt` reports the component object-ref fields in the CDO-instance shape, verbatim: `"UpdatedComponent": "/Game/Vehicles/BP_Buggy.Default__BP_Buggy_C:VehicleMesh"` (and `UpdatedPrimitive` the same) — the `.Default__<Name>_C:<Sub>` shape. The agent copied that shape onto the movement comp and called `vehicle.set_wheel_setup {componentPath:"/Game/Vehicles/BP_Buggy.Default__BP_Buggy_C:VehicleMovementComp", wheelIndex:0}` → `[COMPONENT_NOT_FOUND] Failed to resolve component: /Game/Vehicles/BP_Buggy.Default__BP_Buggy_C:VehicleMovementComp`, then recovered with the plain package path `"/Game/Vehicles/BP_Buggy:VehicleMovementComp"` (after reading `ComponentPathUtils.cpp` as a last resort — the same "real discoverability gap" friction `#3` recorded). Replay-confirmed verbatim via `mcp__editor-automation__call`: (a) `asset.dump` `scs.json` line emits `"UpdatedComponent": "/Game/Vehicles/BP_Buggy.Default__BP_Buggy_C:VehicleMesh"` (the misleading shape, verbatim, on disk under `Saved/EditorAutomation/asset-dumps/Game/Vehicles/BP_Buggy/scs.json`); (b) `vehicle.set_wheel_setup {componentPath:"/Game/Vehicles/BP_Buggy.Default__BP_Buggy_C:VehicleMovementComp", wheelIndex:4}` → `[COMPONENT_NOT_FOUND] Failed to resolve component: /Game/Vehicles/BP_Buggy.Default__BP_Buggy_C:VehicleMovementComp`. Same root cause as `#3` (`ComponentPath::Resolve` loads the `Default__BP_C` left-half to a CDO instance whose cast to `UBlueprint`/`UClass` fails → null `OwnerClass`). This adds a second confirmed read-emitter (`asset.dump`, not just `blueprint.scs.get`) producing the rejected shape, strengthening fix-facet (2) from `#3` (make `ResolveBlueprintCdoSubobject` redirect a loaded CDO instance via `GetClass()` so the `Default__BP_C` form round-trips) — both BP-introspection dumps hand back the failing shape. No new file: same `vehicle.*` `componentPath` non-round-trippable root cause; appended here.

- `#5-no-concrete-cdo-path-example-host-absent` `OPEN` reporter — Additional evidence (same `componentPath` no-steer/no-example gap, viewed from the *host-absent* angle; seed `vehicle.create_wheel_asset`, finding about `vehicle.set_wheel_setup`/`vehicle.set_suspension`; task outcome `clean`, so ERGONOMIC). A 4-wheel off-road-buggy authoring task whose step 4 said "I have a wheeled-vehicle Blueprint at /Game/Vehicles/BP_OffroadBuggy with a movement component. Mount all four wheels … " and step 5 routed suspension through that same component. The Blueprint did not exist (`asset.exists`=false, `/Game/Vehicles` absent from the registry), so the agent reasonably guessed the conventional path form `/Game/Vehicles/BP_OffroadBuggy.BP_OffroadBuggy_C:MovementComponent` and both `vehicle.set_wheel_setup {wheelIndex:0,...}` and `vehicle.set_suspension {all wheels,...}` returned `[COMPONENT_NOT_FOUND] Failed to resolve component: /Game/Vehicles/BP_OffroadBuggy.BP_OffroadBuggy_C:MovementComponent` (correct — there is no host; the agent then cleanly reported the BP/movement-comp must be created to proceed). Friction note (verbatim): *"the wiki for set_suspension/set_wheel_setup gives no concrete componentPath example for a BP CDO movement component (only \"BP CDO subobject path or live actor component path\"), so I had to guess the path form — though since the BP doesn't exist it would have failed regardless, and the resulting error was clear."* This is the same docs gap as `#3`/`#4` (the `componentPath` param doc gives no concrete accepted-shape example), distinct manifestation: the agent had to *invent* the path form from a one-line param description with the host absent — strengthening fix-facet (1) docs (give a concrete `<BlueprintPath>:<ComponentName>` example on the `componentPath` param). Note the conventional component name the agent guessed (`MovementComponent`) also differs from the real Chaos default (`VehicleMovement`/`VehicleMovementComp`), a second reason a concrete example helps. Also exercised step 5's suspension on **standalone wheel assets** via `vehicle.set_wheel_asset_property` (8 single-field calls, 4 per asset) — see `F-vehicle-wheel-asset-batch-properties` for that batch facet. Dedup: ripgrep OPEN+closed — same `vehicle.*` `componentPath` no-steer/no-example root cause as `#1`–`#4`; appended here rather than a new file.
- `#3-scs-get-emits-non-roundtrippable-componentpath` `OPEN` reporter — Additional evidence: a third, sharper facet of the same `componentPath` contract gap — the **read RPC emits a path shape the write RPC rejects** (seed `vehicle.remove_wheel_setup`, finding about `vehicle.set_wheel_setup`; task outcome was `done` so this is an ERGONOMIC round-trip mismatch, not a tool bug). A six-wheel-hauler build task. The agent inspected the freshly-created `WheeledVehiclePawn` BP with `blueprint.scs.get`, whose `VehicleMovementComp` `properties` block reports an object-ref field as the literal CDO-instance path `"/Game/.../BP.Default__BP_C:VehicleMesh"` (the `AssetName.Default__AssetName_C:Subobject` shape). The agent copied that exact shape into `vehicle.set_wheel_setup`'s `componentPath` (param doc: *"BP CDO subobject path or live actor component path"* — so the shape looks like precisely what is asked for) and got `[COMPONENT_NOT_FOUND]`; only after reading `ComponentPathUtils.cpp` did it find the working plain package path `"/Game/.../BP:VehicleMovementComp"`. Replay-confirmed verbatim via `mcp__editor-automation__call` on a fresh `/Game/FuzzVeh/BP_ReproHauler` (`WheeledVehiclePawn`): (a) `blueprint.scs.get` emits `"UpdatedComponent":"/Game/FuzzVeh/BP_ReproHauler.Default__BP_ReproHauler_C:VehicleMesh"` (the misleading shape, verbatim); (b) `vehicle.set_wheel_setup {componentPath:"/Game/FuzzVeh/BP_ReproHauler.Default__BP_ReproHauler_C:VehicleMovementComp", wheelIndex:0}` → `[COMPONENT_NOT_FOUND] Failed to resolve component: /Game/FuzzVeh/BP_ReproHauler.Default__BP_ReproHauler_C:VehicleMovementComp`; (c) the `_C`-without-`Default__` form `"/Game/FuzzVeh/BP_ReproHauler.BP_ReproHauler_C:VehicleMovementComp"` → SUCCESS (`wheelSetupCount:1`); (d) the plain package path `"/Game/FuzzVeh/BP_ReproHauler:VehicleMovementComp"` → SUCCESS (`wheelSetupCount:2`). Root cause: `ComponentPath::Resolve` (`ComponentPathUtils.cpp`) splits on the last `:`, then `LoadObject`s the LEFT half and accepts it only if it casts to `UBlueprint` (uses `GeneratedClass`) or `UClass`; the `Default__BP_C` left-half loads to the **CDO instance** (a plain `UObject`), so `OwnerClass` stays null → `COMPONENT_NOT_FOUND`. So three of the four shapes a reasonable agent would copy work/fail inconsistently, and the one shape the read RPC literally hands back is the failing one. Two additive fixes alongside the host-find + wheel-asset-CDO steers already proposed here: (1) **docs** — on `set_wheel_setup`/`set_suspension`/`remove_wheel_setup` `componentPath`, state the accepted shapes explicitly (`<BlueprintPath>:<ComponentName>` plain package path, or `<BlueprintPath>.<BlueprintName>_C:<ComponentName>`) and warn that the `.Default__<Name>_C:` form that `blueprint.scs.get`/`inspect` emit is NOT accepted; (2) **resolver (cheap, robust)** — in `ResolveBlueprintCdoSubobject`, when the loaded left-half is itself a `UObject` CDO (or any object), redirect via `Loaded->GetClass()`/`Loaded->IsDefaultSubobject` to the owning class so the `Default__BP_C` form round-trips, mirroring the `E-property-blueprint-cdo` `_C`-path redirect that was already accepted for the `property.*` resolver. Dedup: ripgrep OPEN+closed — no ticket mentions `UpdatedComponent` or the `scs.get`→`vehicle.*` round-trip; `E-spawn-returns-actor-not-component-path` (OPEN) is typed-spawn verbs *withholding* a componentPath (different — here a componentPath IS emitted but is non-round-trippable); `E-property-blueprint-cdo` (DONE) is the analogous `_C`/CDO redirect in the separate `UtilityPropertyHandler` resolver, not `ComponentPathUtils`; same `vehicle.*` `componentPath` no-contract root cause as `#1`/`#2` → appended here rather than a new file.
