---
id: B-niagara-authored-emitter-forces-inert
title: "An emitter assembled entirely with niagara.add_module produces no usable particles — force modules never move them, a mesh emitter renders nothing — while compile, strict validate, the UE log and every write echo report it healthy"
status: OPEN
severity: High
category: bug
tags: [niagara, add_module, forces, solve-forces-and-velocity, simulation, false-success, inert-emitter, mesh-renderer, cause-unknown]
encounters: 1
lastSeen: 2026-08-27T18:56:59+05:00
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

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at `8e76cad5` in this checkout. `/Game/Atlantis/VFX/NS_Bubbles_Stream` emitter `Bubbles`: full ParticleUpdate force stack (`AccelerationForce`, `CurlNoiseForce`, `DragForce`, `SolveForcesAndVelocity`) present, enabled and correctly chained per `decompile_nir`, and particles never leave the ~45 uu spawn cylinder after `effect.advance_simulation {deltaTime:0.0333, steps:300}`. Five controlled negatives each with its own capture (acceleration Z 35→260; lifetime-kill switch off; `Mass` 1.0; `Write Mass` true; fresh re-spawned actor) all identical. Same-level same-session control `/Niagara/DefaultAssets/DefaultSystem` 600 uu away renders a correct moving fountain with bbox growing to extent `(305,135,389)` and a drifted origin, so the editor ticks, renders and captures Niagara correctly — only the API-authored emitter is inert. `niagara.validate level:strict` → `valid:true`, zero errors AND zero warnings; no Niagara compile error in `Saved/Logs/EAContentExamples58.log`. Generalises to a second, independently built system: `/Game/Atlantis/VFX/NS_FishSchool` (mesh renderer, seven `add_module` calls) renders nothing at all, with mesh assignment, material assignment, Mesh Scale and frustum culling/bounds each individually ruled out (`fish_test_{A1,B1,C1}.png`). **Cause NOT identified — filed as a verified symptom, not a diagnosis.** The obvious force÷mass theory is explicitly DISPROVED by the `Mass`/`Write Mass` negatives; the two remaining probes (`SolveForcesAndVelocity` static-switch combination out of `add_module`; whether `Particles.PhysicsForce` is written at all) are recorded as untested hypotheses only. Reusable trap recorded: `actor.get_bounding_box` on an `ANiagaraActor` returns the editor `ArrowComponent`'s `origin (x+116,y,z), extent (244,128,128), radius 303.8157` when particle bounds are empty and goes stale across `bFixedBounds` toggles, so a non-zero box is not proof of live particles. `valid:true` from strict validate confirmed EXPECTED via `E-niagara-validate-strict-empty-system-undocumented` (strict escalates only `NO_EMITTERS`/`DISABLED_EMITTER`/`NO_RENDERERS`), so validate is not at fault. Prior-encounter argument: `E-niagara-standard-stack-recipe-undocumented` History `#1`/`#3`/`#5`/`#6` record at least four fuzz tasks that built this same stack shape and declared success on compile + strict validate + save alone without ever simulating or looking — probably four unrecorded instances of this defect. All evidence carried over from the session log; nothing in this ticket was re-verified against plugin source, because the fault is in runtime evaluation and no source line has been implicated.
