---
id: B-niagara-authored-emitter-forces-inert
title: "An emitter assembled entirely with niagara.add_module produces no usable particles — force modules never move them, a mesh emitter renders nothing — while compile, strict validate, the UE log and every write echo report it healthy"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, add_emitter, add_module, forces, solve-forces-and-velocity, simulation, false-success, silent-noop, inert-emitter, mesh-renderer, system-graph, emitter-node, root-cause-found]
encounters: 5
lastSeen: 2026-08-27T19:50:00+05:00
---

# Emitters built module-by-module through this API do not simulate, and every published signal says they are fine

An emitter whose ParticleUpdate stack was assembled entirely through
`niagara.add_module` spawns and renders particles correctly, but they **never leave
the spawn volume**. `AccelerationForce`, `CurlNoiseForce`, `DragForce` and
`SolveForcesAndVelocity` are all present, enabled, and correctly chained, and none
of them has any effect on particle position.

A second, independently built system generalises it past forces and past sprites: a
mesh-renderer emitter built the same way renders **nothing at all**.

Nothing in the plugin's reporting distinguishes either from a working emitter.
`niagara.validate level:strict` returns `valid: true` with **zero errors and zero
warnings**; there is no Niagara script compile error in
`Saved/Logs/EAContentExamples58.log`; every `add_module` and `set_module_input`
returns success.

**The cause is not identified.** This ticket is a verified symptom with a controlled
negative, not a diagnosis. The "already ruled out" section below exists so the next
agent does not re-walk the eliminations.

## System A — forces have no effect (sprite renderer)

`/Game/Atlantis/VFX/NS_Bubbles_Stream`, emitter `Bubbles` (CPUSim, local space).

- ParticleSpawn: `InitializeParticle`, `CylinderLocation` (radius 45, height 30)
- ParticleUpdate: `ParticleState`, `AccelerationForce`, `CurlNoiseForce`,
  `DragForce`, `SolveForcesAndVelocity`, `ScaleSpriteSizeBySpeed`, `ScaleColorBySpeed`

```
effect.activate_niagara   {systemName, reset: true}
effect.advance_simulation {systemName, deltaTime: 0.0333, steps: 300}   # 10 s
render.capture_open_level {...}
```

Particles sit in a cluster the width of the spawn cylinder (~45 uu) and ~250 uu
tall, unchanged.

### Five controlled negatives, each with its own capture

| Change | Result |
|---|---|
| `AccelerationForce.Acceleration` Z 35 → **260** (7.4x) | cluster identical |
| `ParticleState` static switch `Kill Particles When Lifetime Has Elapsed` → **false** | cluster identical — so they are not dying and still do not move |
| `InitializeParticle.Mass` → **1.0** | cluster identical |
| `InitializeParticle.Write Mass` → **true** | cluster identical |
| Deleted the actor and re-spawned it fresh against the recompiled system | cluster identical |

Captures: `Saved/Screenshots/OpenLevel/bubbles_final_{D1,E1,F1}.png`,
`bubbles_test_{G1,H1,I1}.png`.

## Same-level, same-session control — the editor, the viewport and the capture path all work

`/Niagara/DefaultAssets/DefaultSystem` spawned into the **same level, same session,
600 uu away** renders a correct moving fountain (`vfx_control_lit_M1.png`), and its
`actor.get_bounding_box` grows to extent `(305,135,389)` with the origin drifted off
the spawn point.

So the editor ticks Niagara, the level viewport renders Niagara, and
`render.capture_open_level` photographs Niagara. **Only the API-authored emitter is
inert.**

## System B — generalises to a second system, a second renderer type, a second author

`/Game/Atlantis/VFX/NS_FishSchool` was built from nothing in the same session
(`niagara.create_system` + `niagara.create_emitter` + `niagara.add_emitter` +
`niagara.add_renderer` + seven `niagara.add_module` calls), with a **mesh** renderer
rather than a sprite one. It renders **nothing at all** — not even a static clump.

Stack: `EmitterState`, `SpawnRate` (30/s literal); `InitializeParticle` (Lifetime 25,
Mesh Scale 3,3,3), `CylinderLocation` (r 700, h 500), `AddVelocity` (0,300,0);
`ParticleState`, `SolveForcesAndVelocity`.

Ruled out one at a time, each with its own capture (`fish_test_{A1,B1,C1}.png`):

- **Mesh assigned** — `Meshes[0].Mesh` = `/Game/Atlantis/Meshes/SM_Fish_A`,
  `set_property` succeeded.
- **Material assigned** — on the mesh asset itself (`static_mesh.set_material` →
  `MI_Fish_Blue`), because `OverrideMaterials[0]` returns
  `PROPERTY_NOT_FOUND: Array index 0 out of range (length 0)` and `set_property`
  cannot grow the array.
- **Mesh Scale** — the default would render at zero size; set to `(3,3,3)`. No change.
- **Frustum culling** — the component's bounds were the editor `ArrowComponent`'s
  `(244,128,128)`. Set `bFixedBounds` + `FixedBounds`
  (−3000,−3000,−1500)…(3000,9000,1500); the capture response then reports
  `boundsRadius: 6873.86` at the right origin, so the bounds are real and in frame.
  No change.

The failure is therefore **not** renderer-specific, **not** force-specific, and
**not** one-asset-specific: an emitter authored through this API does not produce
usable particles, whether it renders sprites (static clump) or meshes (nothing).
Two systems, two renderer types, two authors.

## Already ruled out / still untested — do NOT re-walk these

**Ruled out (each with a capture, listed above):** force magnitude; lifetime kill;
`Mass`; `Write Mass`; a stale spawned actor; renderer type; mesh assignment;
material assignment; mesh scale; frustum culling / bounds; the editor's ability to
tick, render and capture Niagara at all.

**Explicitly disproved:** the obvious "force ÷ mass = NaN" theory. Both the `Mass`
and `Write Mass` tests were run precisely to test it and **both failed to change
anything**. That hypothesis is dead, not open.

**The wiring is correct, so the fault is in evaluation, not in the graph shape.**
`niagara.decompile_nir` shows
`Map Set`_4 → `Acceleration Force` → `Map Set`_5 → `Curl Noise Force` →
`Map Set`_6 → `Drag Force` → `Map Set`_7 → `Solve Forces and Velocity` →
`Map Set`_12 → … → `Output Particle Update`.

**Still untested, and the two most likely next probes — UNVERIFIED, hypotheses only,
recorded so they are not lost:**

1. Whether `SolveForcesAndVelocity`'s own static switches
   (`Write to Presolve Properties`, `Clamp Velocity`, `Limit Acceleration`,
   `Rotational Solver Is Enabled`) come out of `add_module` in a combination the
   module cannot run in. Note `B-niagara-static-switch-enum-display-name`: a static
   switch left at index 0 is not necessarily the option a caller would assume.
2. Whether `Particles.PhysicsForce` is actually being written at all.

Neither has been tested. Do not treat either as a finding.

**Untried workaround (the obvious next thing):** duplicate a stock engine emitter
that already has a working force stack instead of assembling one with `add_module`.
Not tested here. No workaround is currently known.

## `niagara.validate level:strict` returning `valid:true` is expected, not a second bug

`E-niagara-validate-strict-empty-system-undocumented` documents that strict escalates
exactly `NO_EMITTERS` / `DISABLED_EMITTER` / `NO_RENDERERS` from warning to error
(its `#4-docs-strict-validate-levels` pins the escalation to
`NiagaraInspectHandler.cpp` `NormalizeValidationSeverity` `:101-107`). A populated
emitter with a renderer trips none of the three, so `valid: true` with zero issues is
exactly what strict is designed to return here. **Validate is not broken; it simply
has nothing to say about whether an emitter simulates.** That is the gap this ticket
names: there is no signal anywhere in the plugin that separates a working emitter
from an inert one.

## Reusable diagnostic trap: `actor.get_bounding_box` on an `ANiagaraActor` lies

When particle bounds are empty, `actor.get_bounding_box` on an `ANiagaraActor`
returns the editor **`ArrowComponent`'s** box, not the effect's:

```
origin (x+116, y, z), extent (244,128,128), radius 303.8157
```

`radius 303.8157` is its signature. The value also goes **stale across
`bFixedBounds` toggles**. A non-zero bounding box is **not** proof of live
particles. Only the stock control system's `(305,135,389)` with a drifted origin was
a real particle bound. Any future automated "did the effect work" check must not use
this probe alone.

## Prior encounters this probably explains

`E-niagara-standard-stack-recipe-undocumented` (OPEN) records **at least four**
prior fuzz tasks that built exactly this stack shape and declared success:

- `#3-additional-order-sensitive-phrase` — `/Game/VFX/FX_CampfireEmbers`, "task
  otherwise completed clean (system+emitter+renderer+2 modules built,
  compile/validate/save all green)"
- `#5-additional-particle-state` — `/Game/FX/NS_CampfireEmbers`,
  "system+emitter+sprite renderer+EmitterState/SpawnRate/InitializeParticle/
  AddVelocity/ParticleState/Drag/SolveForcesAndVelocity built, compile/validate
  strict/save all green"
- `#6-additional-scoremodulematch-guilty-line` — `/Game/VFX/NS_CampfireEmbers`,
  "outcome clean/done"
- `#1-initial-audit` — `/Game/VFX/NS_FloatingEmbers`, the same four-module canonical
  stack

Every one of those declared success on **compile + strict validate + save alone**.
None of them simulated the system or looked at the result. If this ticket is real,
those are probably four unrecorded instances of it, and the "green" in those history
entries is the same green this ticket shows is uninformative.

## Impact

Force-driven motion is what most particle effects *are*. An emitter built
module-by-module through this API renders a static clump (or nothing), and every
published signal — strict validate, compile status, the UE log, each
`set_module_input` echo — says it is healthy. Two agents in a row lost most of a
session to this on one system. It does not crash and does not affect other agents
sharing the editor.

## Distinct from related tickets

- `E-niagara-validate-strict-empty-system-undocumented` (IN-REVIEW, Low) is the docs
  ticket for what strict escalates. Cited here to establish that `valid:true` is
  correct behaviour, not a validate defect.
- `E-niagara-standard-stack-recipe-undocumented` (OPEN, Low) is about **discovering**
  the module paths (`search_modules` phrase matching). This ticket is about the stack
  those calls build not working. Its History is cited as prior-encounter evidence.
- `B-niagara-literal-over-linked-override-pin` is a *contributing* silent-write
  defect on the same asset (`NS_Bubbles_Stream`) but cannot explain this: the force
  values were confirmed to change (`Acceleration` Z 35 → 260 was applied and made no
  difference), and it does not explain `NS_FishSchool` rendering nothing.
- `B-niagara-reset-module-input-corrupts-stack` and
  `B-niagara-set-parameter-emitter-scope-unreachable` are separate defects found on
  the same build; neither is on this path.

severity rationale: impact=silent false-success on the central authoring path — a fully-authored emitter is inert while compile, `validate level:strict`, the engine log and every write echo report it healthy, so the caller trusts a result that is a lie and ships it x reach=`add_module` stack assembly is *the* way effects are authored through this plugin, reproduced on two systems with two renderer types, with at least four probable prior unrecorded instances on the board -> High. Not Critical: nothing crashes and no asset data is corrupted — the asset is merely wrong.

---

# ROOT CAUSE FOUND (encounter 2, 2026-08-27) - `niagara.add_emitter` never wires the emitter into the system graph

**The emitter is not inert. It is never called.** `niagara.add_emitter` adds an
`FNiagaraEmitterHandle` to the system and stops there. It never creates the pair of
`UNiagaraNodeEmitter` nodes that the system's SystemSpawn / SystemUpdate graph needs in
order to *invoke* that emitter's spawn and update scripts. A system built with
`niagara.create_system` + `niagara.add_emitter` therefore has an emitter that compiles,
validates, has a renderer and is listed everywhere - and is never executed, so it never
spawns a single particle.

This supersedes the "cause not identified" framing above. It is not force-specific, not
`add_module`-specific and not renderer-specific, which is exactly what encounter 1
observed but could not explain.

## Guilty source lines

`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraHandler.cpp:87-91`:

```cpp
    const FScopedTransaction Transaction(NSLOCTEXT("PinWright", "NiagaraAddEmitter", "Add Niagara Emitter"));
    System->Modify();
    NewHandle = System->AddEmitterHandle(*Emitter, HandleName, EmitterVersion);
    System->MarkPackageDirty();
```

followed by `System->RequestCompile(true)` at `:97`. The engine's own equivalent is
`FNiagaraEditorUtilities::AddEmitterToSystem` (NiagaraEditorUtilities.cpp), which after
`AddEmitterHandle` also calls **`FNiagaraStackGraphUtilities::RebuildEmitterNodes(InSystem)`**
plus `SynchronizeOverviewGraphWithSystem`. `RebuildEmitterNodes` is what creates the
`UNiagaraNodeEmitter` for each handle and links it into the SystemSpawn/SystemUpdate
output chain. The plugin never calls it: a grep for `RebuildEmitterNodes` and for
`NiagaraNodeEmitter` across the whole of `Source/PinWright/Private` returns exactly one
hit, and it is a comment saying the branch was deliberately left out -
`Handlers/Niagara/NiagaraGraphResetUtils.cpp:106-109`:

```
// Vendored verbatim from FNiagaraStackGraphUtilities::ResetGraphForOutput (UE 5.6,
// NiagaraStackGraphUtilities.cpp). The engine version lacks NIAGARAEDITOR_API and
// cannot be linked. The SystemSpawn/SystemUpdate RebuildEmitterNodes branch is
// intentionally omitted - callers in this plugin only invoke for ParticleEvent /
// ParticleSimulationStage usages, never the system-graph cases.
```

So the plugin has no code path anywhere that can produce a `UNiagaraNodeEmitter`. The
same non-export problem applies to `RebuildEmitterNodes` itself, so a fix probably has
to be vendored the way `ResetGraphForOutput` already was.

## Evidence - the NIR system graph, working vs dead

`asset.dump` -> `nir.txt`, `graph SystemUpdate` block, same session, same editor.

**Works** - `/Game/PinWrightScratch/NS_DefProbe`, an `asset.duplicate` copy of
`/Niagara/DefaultAssets/DefaultSystem`:

```
node `System State` : NiagaraNodeFunctionCall  @(-800, 150)
node `Emitter Fountain Spawn` : NiagaraNodeEmitter  @(-400, 0)
node `Emitter Fountain Update` : NiagaraNodeEmitter  @(-400, 150)
...
link `Emitter Fountain Spawn`.OutputMap  -> `Output System Spawn`.Out
link `Emitter Fountain Update`.OutputMap -> `Output System Update`.Out
link `System State`.OutputMap            -> `Emitter Fountain Update`.InputMap
link InputMap.Input                      -> `Emitter Fountain Spawn`.InputMap
```

**Dead** - `/Game/Atlantis/VFX/NS_FishSchool`, built with `create_system` +
`add_emitter`:

```
node `System State` : NiagaraNodeFunctionCall  @(-400, 150)
...
link `System State`.OutputMap -> `Output System Update`.Out
link InputMap.Input           -> `Output System Spawn`.Out
```

No `NiagaraNodeEmitter`. `Output System Spawn` is fed straight from `InputMap`. The
emitter handle `FishA` exists with `bIsEnabled: true`, its `SpawnScript` and
`UpdateScript` are `NCS_UpToDate`, its renderer has a mesh and a material - and nothing
calls it.

`/Game/Atlantis/VFX/NS_Bubbles_Stream` (encounter 1's System A) has the **same empty
system graph**, and so does `/Game/Atlantis/VFX/NS_Plankton_Drift`, a third system by a
third author. Every system on this project built through `create_system` +
`add_emitter` has it.

## A/B inside ONE system - the tightest repro

Two emitters in the same system, one wired, one not:

```
asset.duplicate   {sourcePath: "/Niagara/DefaultAssets/DefaultSystem",
                   destinationPath: "/Game/PinWrightScratch/NS_DefProbe"}
niagara.compile   {assetPath: "/Game/PinWrightScratch/NS_DefProbe", force: true, wait: true}
niagara.spawn_actor {systemPath: "/Game/PinWrightScratch/NS_DefProbe",
                     location: {x:19500, y:-5200, z:1900}, name: "PWFISH_DefProbe"}
effect.activate_niagara {systemName: "PWFISH_DefProbe", reset: true}
actor.get_bounding_box  {actorName: "PWFISH_DefProbe"}
  -> origin [19585.86, -5187.46, 1743.10], extent [274.14, 140.54, 418.94]   # real particles

niagara.add_emitter {systemPath: "/Game/PinWrightScratch/NS_DefProbe",
                     emitterPath: "/Niagara/DefaultAssets/Templates/Emitters/UpwardMeshBurst.UpwardMeshBurst",
                     name: "Tpl2", compile: true, save: true}   -> success, emitterCount: 2, compiled: true
niagara.compile   {assetPath: "/Game/PinWrightScratch/NS_DefProbe", force: true, wait: true}
effect.activate_niagara {systemName: "PWFISH_DefProbe", reset: true}
render.capture_open_level {location:{x:17700,y:-5200,z:1960}, rotation:{pitch:-2,yaw:0,roll:0},
                           fov:60, width:768, height:768,
                           exposure:{mode:"fixed", ev100:0}, hideEditorSprites:true}
```

`Saved/Screenshots/OpenLevel/fish_diag4.png`: the `Fountain` emitter's sprites are
there, the `Tpl2` mesh particles are not - one system, one compile, one frame. The
system graph after the add has emitter nodes for `Fountain` only.

## New, cheap discriminator: the component refuses to activate

`UNiagaraComponent::IsActive()` is the signal encounter 1 was missing. It is reachable
without opening any asset editor:

```
object.call_function {objectPath: "/Game/Maps/Atlantis.Atlantis:PersistentLevel.NiagaraActor_4.NiagaraComponent0",
                      function: "IsActive", args: {}}
```

Measured on six `ANiagaraActor`s in one `python.execute` sweep, reading `IsActive()`
immediately after `comp.activate(True)` in the same script, so there is no real-time
race:

| actor | system | `IsActive()` after `activate(True)` |
|---|---|---|
| `PWFISH_DefProbe` | duplicate of stock `DefaultSystem` | **true** |
| `PWFISH_Control` | stock `/Niagara/DefaultAssets/DefaultSystem` | **true** |
| `PWFISH_Probe` | `NS_FishSchool` (create_system + add_emitter) | **false** |
| `PWFISH_TplProbe` | `NS_TplProbe` (create_system + add_emitter) | **false** |
| `VFX_BubbleVent_A` | `NS_Bubbles_Stream` | **false** |
| `PWTEST_Plankton` | `NS_Plankton_Drift` | **false** |

`effect.activate_niagara` returns `{"active": true}` on every one of those, including
the four that read false a millisecond later - it echoes the request, not the
component. That is a second, separable reporting defect on the same path.

## Two corrections to encounter 1

1. **"Particles sit in a cluster ~45 uu wide" (System A) cannot be what it was read
   as.** `NS_Bubbles_Stream`'s system graph has no emitter node either, so that emitter
   has never executed a tick. Whatever is in `bubbles_final_*.png` is not this emitter
   simulating and failing to move; it is not simulating at all.
2. **`OverrideMaterials` is not ungrowable.** Encounter 1 records `OverrideMaterials[0]`
   -> `PROPERTY_NOT_FOUND: Array index 0 out of range (length 0)` "and `set_property`
   cannot grow the array". Assigning the **whole array** works:

   ```
   niagara.set_property {assetPath: "/Game/Atlantis/VFX/NS_FishSchool",
                         target: {kind:"renderer", emitter:"FishA", index:0},
                         propertyPath: "Meshes",
                         value: [{"Mesh": "/Game/Atlantis/Meshes/SM_Fish_A.SM_Fish_A", "Scale":[1,1,1]},
                                 {"Mesh": "/Game/Atlantis/Meshes/SM_Fish_B.SM_Fish_B", "Scale":[1,1,1]},
                                 {"Mesh": "/Game/Atlantis/Meshes/SM_Fish_A.SM_Fish_A", "Scale":[1,1,1]}]}
   ```

   read back through `niagara.inspect` as 3 entries with the right meshes. Only the
   indexed path `Meshes[i]` / `OverrideMaterials[i]` is bounded by the current length.

## Answering the open question this ticket left: does duplicating a stock template help?

The host project raised a ruling permitting stock-template duplication specifically to
work around this ticket. Both halves measured:

- **Duplicating a stock template EMITTER does not help.**
  `asset.duplicate /Niagara/DefaultAssets/Templates/Emitters/UpwardMeshBurst` ->
  `/Game/Atlantis/VFX/NE_Fish_A`, then `niagara.add_emitter` into a system: still no
  emitter node, still zero particles, unedited. The emitter was never the problem.
- **Duplicating a stock SYSTEM does work**, because the copy carries the system graph
  and its emitter nodes. It needed one `niagara.compile {force: true}` after the
  duplicate before the component would activate (`IsActive()` false -> true, bbox extent
  `(244,128,128)` ArrowComponent -> `(274,141,419)` with a drifted origin).

## Workaround

Do not build a system with `niagara.create_system` + `niagara.add_emitter` until this is
fixed - the result cannot run. Instead `asset.duplicate` a complete stock system that
already has one emitter-node pair (`/Niagara/DefaultAssets/DefaultSystem`), run
`niagara.compile {force: true, wait: true}` once, and rewrite that emitter in place:
`niagara.add_renderer` / `niagara.remove_renderer`, `niagara.add_module`,
`niagara.set_module_input`, `niagara.set_static_switch` and `niagara.set_property` all
operate below the system graph and are unaffected. The cost is exactly one emitter per
system, because there is no way to add a second wired one.

Verify with `IsActive()` (table above) plus a capture, never with compile/validate/save.

## Suggested fix

After `System->AddEmitterHandle(...)` in `niagara.add_emitter` (and symmetrically after
the removal in `niagara.remove_emitter`), rebuild the system graph's emitter nodes.
`FNiagaraStackGraphUtilities::RebuildEmitterNodes` is not `NIAGARAEDITOR_API`-exported,
so it likely has to be inline-vendored the way `ResetGraphForOutput` already is in
`NiagaraGraphResetUtils.cpp` - the same "Engine-helper non-export gotcha" the
`niagara.authoring` wiki page documents. A regression test should assert that after
`add_emitter` the system's `SystemSpawnScript` graph contains a `UNiagaraNodeEmitter`
whose `EmitterHandleId` matches the new handle and whose output pin reaches the
`UNiagaraNodeOutput` for `SystemSpawnScript`; that test fails against current HEAD.

Independently: `effect.activate_niagara` should report the component's measured
`IsActive()` after the call rather than a constant `true`, and `niagara.validate` should
flag an emitter handle that has no `UNiagaraNodeEmitter` in the system graph. Either one
alone would have collapsed both encounters of this ticket into a single call.


---

# The two remaining repair routes are closed too (encounter 5, 2026-08-27)

Other encounters ruled out `niagara.compile` (with and without `force`), `save` and
`asset.reload` — all leave 0 `NiagaraNodeEmitter`. That covers the routes that go
*through* the compile pipeline. These two go around it, and both are dead ends, so a
system built with `add_emitter` really cannot be repaired by any client-side call:

- **`niagara.graph.create_node` refuses the class, and refusing it kills the editor.**
  `nodeClass: "NiagaraNodeEmitter"` returns
  `[UNSUPPORTED_NODE_CLASS] Node class 'NiagaraNodeEmitter' is not supported by
  niagara.graph.create_node v1` — the v1 allowlist in `ApplyCreateNodePayload`
  (`NiagaraGraphHandler.cpp:571`) is `{NiagaraNodeOp, NiagaraNodeInput, NiagaraNodeOutput,
  NiagaraNodeCustomHlsl, NiagaraNodeStaticSwitch, NiagaraNodeIf, NiagaraNodeReroute,
  NiagaraNodeConvert}`. Worse, that rejection path skips `NodeCreator.Finalize()` and the
  `FGraphNodeCreator` destructor asserts, so the attempt takes the whole editor down —
  `B-niagara-create-node-unfinalized-graph-node-creator-fatal`. **Do not run this probe
  again to confirm it.**

- **Python cannot finish the job either.** `python.execute` can `new_object` a
  `UNiagaraNodeEmitter` and can write its `UPROPERTY`s (`OwnerSystem`, `EmitterHandleId`,
  `ScriptType`), and `UEdGraph::Nodes` is a `UPROPERTY` so the node can even be inserted.
  But `UEdGraphPin` is not a `UObject` and is not exposed to UE Python, and
  `AllocateDefaultPins` is not a `UFUNCTION`, so the node can never get its InputMap /
  OutputMap pins or be wired to `Output System Spawn` / `Output System Update`. A pinless
  emitter node is no better than no node. `niagara.graph.connect_pins` cannot help,
  because it needs pins that exist.

So the fix has to be server-side: vendor `RebuildEmitterNodes` into `add_emitter` as the
suggested-fix section above says. Until then a system built with
`create_system` + `add_emitter` must be **discarded and rebuilt from an `asset.duplicate`
of a stock system**, not repaired.


## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at `8e76cad5` in this checkout. `/Game/Atlantis/VFX/NS_Bubbles_Stream` emitter `Bubbles`: full ParticleUpdate force stack (`AccelerationForce`, `CurlNoiseForce`, `DragForce`, `SolveForcesAndVelocity`) present, enabled and correctly chained per `decompile_nir`, and particles never leave the ~45 uu spawn cylinder after `effect.advance_simulation {deltaTime:0.0333, steps:300}`. Five controlled negatives each with its own capture (acceleration Z 35→260; lifetime-kill switch off; `Mass` 1.0; `Write Mass` true; fresh re-spawned actor) all identical. Same-level same-session control `/Niagara/DefaultAssets/DefaultSystem` 600 uu away renders a correct moving fountain with bbox growing to extent `(305,135,389)` and a drifted origin, so the editor ticks, renders and captures Niagara correctly — only the API-authored emitter is inert. `niagara.validate level:strict` → `valid:true`, zero errors AND zero warnings; no Niagara compile error in `Saved/Logs/EAContentExamples58.log`. Generalises to a second, independently built system: `/Game/Atlantis/VFX/NS_FishSchool` (mesh renderer, seven `add_module` calls) renders nothing at all, with mesh assignment, material assignment, Mesh Scale and frustum culling/bounds each individually ruled out (`fish_test_{A1,B1,C1}.png`). **Cause NOT identified — filed as a verified symptom, not a diagnosis.** The obvious force÷mass theory is explicitly DISPROVED by the `Mass`/`Write Mass` negatives; the two remaining probes (`SolveForcesAndVelocity` static-switch combination out of `add_module`; whether `Particles.PhysicsForce` is written at all) are recorded as untested hypotheses only. Reusable trap recorded: `actor.get_bounding_box` on an `ANiagaraActor` returns the editor `ArrowComponent`'s `origin (x+116,y,z), extent (244,128,128), radius 303.8157` when particle bounds are empty and goes stale across `bFixedBounds` toggles, so a non-zero box is not proof of live particles. `valid:true` from strict validate confirmed EXPECTED via `E-niagara-validate-strict-empty-system-undocumented` (strict escalates only `NO_EMITTERS`/`DISABLED_EMITTER`/`NO_RENDERERS`), so validate is not at fault. Prior-encounter argument: `E-niagara-standard-stack-recipe-undocumented` History `#1`/`#3`/`#5`/`#6` record at least four fuzz tasks that built this same stack shape and declared success on compile + strict validate + save alone without ever simulating or looking — probably four unrecorded instances of this defect. All evidence carried over from the session log; nothing in this ticket was re-verified against plugin source, because the fault is in runtime evaluation and no source line has been implicated.
- `#2-root-cause-add-emitter-no-system-graph-node` `OPEN` reporter - 2026-08-27, UE 5.8, same Atlantis build and editor as #1. Root cause identified and sourced: `niagara.add_emitter` calls only `System->AddEmitterHandle(...)` (`Handlers/Niagara/NiagaraHandler.cpp:89`) and never `FNiagaraStackGraphUtilities::RebuildEmitterNodes`, so no `UNiagaraNodeEmitter` pair is created in the system SystemSpawn/SystemUpdate graph and the emitter is never invoked - it does not simulate at all, which is why forces, renderer type and `add_module` were all red herrings in #1. Grep for `RebuildEmitterNodes`/`NiagaraNodeEmitter` over `Source/PinWright/Private` returns one hit, the comment at `NiagaraGraphResetUtils.cpp:106-109` saying the branch was deliberately omitted, so no code path in the plugin can create the node. Proven by NIR `graph SystemUpdate` diff (stock-duplicate `NS_DefProbe` has `Emitter Fountain Spawn`/`Update` `NiagaraNodeEmitter` nodes linked to both `Output System *` nodes; `NS_FishSchool`, `NS_Bubbles_Stream` and `NS_Plankton_Drift` have none) and by an A/B inside one system (added `Tpl2` to the working `NS_DefProbe`: `Fountain` still renders, `Tpl2` never spawns, emitter nodes exist for `Fountain` only - `Saved/Screenshots/OpenLevel/fish_diag4.png`). New cheap discriminator: `UNiagaraComponent::IsActive()` via `object.call_function`, false on all four create_system+add_emitter systems and true on both stock-derived ones, measured in one `python.execute` sweep straight after `comp.activate(True)`; `effect.activate_niagara` returns `active:true` on all of them regardless. Corrects two claims in #1: System A's "static clump" cannot have been this emitter simulating (its system graph is empty too), and `OverrideMaterials`/`Meshes` CAN be grown by assigning the whole array through `niagara.set_property` (only the indexed `[i]` path is length-bounded). Answers the ticket's untried workaround: duplicating a stock template EMITTER does not help (still no node, zero particles, unedited); duplicating a stock SYSTEM does, after one `niagara.compile {force:true}`. Status left OPEN - reporter, not fixer.
- `#3-no-repair-route-exists` `OPEN` reporter — 2026-08-27, UE 5.8, Atlantis build (plankton/ambient-bubbles agent). Answers the open question from #2 — whether an already-broken `add_emitter` system can be repaired in place — with three measured negatives: **nothing regenerates the missing `UNiagaraNodeEmitter`.** Probe `/Game/PinWrightScratch/NS_ZZ_D27Probe` built with `niagara.create_system` + `niagara.create_emitter` + `niagara.add_emitter {compile:true, save:true}`, then `asset.reload`; `niagara.decompile_nir` afterwards still reports **0** `NiagaraNodeEmitter` occurrences and a `graph SystemUpdate` containing only `Output System Spawn` / `InputMap` / `Output System Update` / `InputMap_2` / `System State`. Independently, the earlier `/Game/Atlantis/VFX/NS_Plankton_Drift` built the same way had been through a *real* compile (`LogNiagara: Compiling System ... took 0.886659 sec`, `Saved/Logs/EAContentExamples58.log`) plus repeated `save:true` edits and still decompiled to 0 emitter nodes. So `niagara.compile` (with and without `force:true`), `save`, and `asset.reload` are all confirmed non-repairing — the node is created only at emitter-add time and there is no second chance. Route deliberately NOT tried: `niagara.graph.create_node` to insert a `NiagaraNodeEmitter` by hand, because that verb killed this editor earlier in the same session (`Assertion failed: Created node was not finalized in a FGraphNodeCreator<EdGraphNode>`, `Handlers/Niagara/NiagaraGraphHandler.cpp:763`). Practical consequence for authors: whole-system `asset.duplicate` of a stock template is not merely the easier path, it is the ONLY path — a system already built with `add_emitter` must be discarded and rebuilt, not fixed. Confirmed working side of the A/B in this session: `asset.duplicate` of `/Niagara/DefaultAssets/DefaultSystem` yields `Emitter Fountain Spawn` / `Emitter Fountain Update` `NiagaraNodeEmitter` nodes (8 occurrences) and renders and simulates correctly after `niagara.compile {force:true}` + `effect.activate_niagara {reset:true}`.
- `#4-vectorvm-dataset-assert-and-crash-tally` `OPEN` reporter — 2026-08-27, UE 5.8, Atlantis, observed as a bystander (rubble-grounding agent, not a Niagara task). **Unattributed observation, recorded so the log is not lost to rotation, NOT a diagnosis and NOT claimed to be this ticket's root cause.** The editor died on a background worker with `Assertion failed: DataSetIdx < ExecCtx->DataSets.Num()` (`Runtime/VectorVM/Private/VectorVMRuntime.cpp:421`), i.e. a Niagara CPU-sim script indexing past its own data-set array — a *running* script whose bindings disagree with its emitter, which is a different failure from the never-invoked emitters of `#2`/`#3` and may belong to a separate defect. Log preserved by rotation as `Saved/Logs/EAContentExamples58-backup-*.log` (crash at 14:41:59 UTC). Context worth having while diagnosing: **six editor kills in this one shared session, three of them Niagara** — this VectorVM assert; the `FGraphNodeCreator` assert from `niagara.graph.create_node` that `#3` names, now filed on its own as `B-niagara-create-node-unfinalized-graph-node-creator-fatal` (root cause: three `return true` paths sit between `CreateNode` and `Finalize()`, so every declared error that verb can raise is an editor kill, and its unit tests miss it by calling `ApplyCreateNodePayload` directly instead of through the dispatcher; this entry originally cited the duplicate `B-niagara-create-node-early-return-before-finalize-crash`, since merged into that ticket and deleted); and a render-thread kill where a static-mesh rebuild left a live Niagara mesh renderer holding a stale LOD index, filed as `B-model-compile-live-niagara-mesh-renderer-raytracing-assert` (this entry originally cited the duplicate `B-static-mesh-rebuild-crashes-live-niagara-mesh-renderer`, since merged into that ticket and deleted). Whoever picks up the crash pattern should treat those three as one cluster: the Niagara surface has no guards around objects other subsystems still hold live references to.
- `#5-repair-routes-closed` `OPEN` reporter - 2026-08-27, same session. Closed the two repair routes that bypass the compile pipeline, completing the negative case. `niagara.graph.create_node {nodeClass:"NiagaraNodeEmitter"}` is refused by the v1 allowlist in `ApplyCreateNodePayload` (`NiagaraGraphHandler.cpp:571`) AND the refusal path kills the editor (`FGraphNodeCreator` `bPlaced` assert - separate ticket `B-niagara-create-node-unfinalized-graph-node-creator-fatal`); do not re-run that probe. Python is also a dead end: `unreal.new_object` can build a `UNiagaraNodeEmitter` and `UEdGraph::Nodes` is a `UPROPERTY`, but `UEdGraphPin` is not a `UObject` and is not exposed to UE Python and `AllocateDefaultPins` is not a `UFUNCTION`, so the node can never get pins or be wired to the `Output System Spawn`/`Output System Update` nodes, and `niagara.graph.connect_pins` needs pins that already exist. Conclusion: no client-side repair exists; the fix must vendor `RebuildEmitterNodes` server-side, and until then an `add_emitter` system is discarded and rebuilt from an `asset.duplicate` of a stock system.

- `#6-rebuild-emitter-nodes-and-surface-uninvoked` `IN-REVIEW` developer — Fixed as #2 diagnosed. Vendored `FNiagaraStackGraphUtilities::RebuildEmitterNodes` (not `NIAGARAEDITOR_API`) as `PinWrightNiagara::RebuildSystemEmitterNodes` in `Handlers/Niagara/NiagaraGraphResetUtils.cpp/.h`, beside the existing `ResetGraphForOutput` vendoring whose comment named the omitted system-graph branch; that comment now points at the new function. `UNiagaraNodeEmitter` is `UCLASS(MinimalAPI)`, so `StaticClass()` links but `SetOwnerSystem`/`SetEmitterHandleId`/`GetEmitterHandleId` do not — the node's identity is written and read through its `OwnerSystem` / `EmitterHandleId` UPROPERTYs by reflection, `SetUsage` is inline in the header, `AllocateDefaultPins` comes free from `FGraphNodeCreator::Finalize`, and `RefreshFromExternalChanges` is called through the `UNiagaraNode` base declaration so only the vtable entry is needed. `niagara.add_emitter` calls it inside its transaction right after `AddEmitterHandle`, and `niagara.remove_emitter` right after `RemoveEmitterHandlesById` (which otherwise leaves nodes naming a dead handle id). Both then read the graph back through `FindUninvokedEmitterHandles` BEFORE compiling or saving and refuse with the new `NIAGARA_EMITTER_NOT_IN_SYSTEM_GRAPH` rather than persisting an inert asset; both responses now publish `emittersInvokedBySystemGraph` and `emitterNodesRebuilt`, measured off the graph. Visibility half: `niagara.validate` gained `AddUninvokedEmitterIssues` (`NiagaraInspectHandler.cpp`), which raises `EMITTER_NOT_IN_SYSTEM_GRAPH` as an **error at every level** (not a strict-only escalation) for each handle the system graph never invokes, and one issue naming the unreadable graph when no verdict is possible — so an already-broken system now fails the read the caller was already making. Side effect worth knowing: because the rebuild covers every handle, running `add_emitter`/`remove_emitter` on a system broken by the old code now repairs it, contradicting `#3`/`#5`'s "no repair route exists" for post-fix builds only. Tests: `PinWright.niagara.add_emitter.WiresEmitterIntoSystemGraph` and `PinWright.niagara.validate.UninvokedEmitterIsAnError` in `Tests/Niagara/TestNiagaraSystemGraphEmitterWiring.cpp`, both on a fixture built with `UNiagaraSystemFactoryNew::InitializeSystem` (what `niagara.create_system` calls), counting emitter nodes through `niagara.graph.get` — a different subsystem from the one that writes them. Counterfactuals: 0 emitter nodes after a successful `add_emitter` pre-fix vs 2 post-fix; `valid:true` with zero issues pre-fix vs a hard `EMITTER_NOT_IN_SYSTEM_GRAPH` error post-fix. NOT compiled or run — the fix agent does not build; the orchestrator owns the build and the suite.
