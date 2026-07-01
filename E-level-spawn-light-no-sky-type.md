---
id: E-level-spawn-light-no-sky-type
title: "level.spawn_light omits 'Sky' from its lightType enum, so a single 'drop in a few lights incl. a Sky light' gesture forces a hop to the lighting.* namespace"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [level, lighting, spawn_light, sky-light, namespace-split, discoverability, docs]
---

# `level.spawn_light` can't spawn a Sky light; sky lives in a different namespace

`level.spawn_light` is the convenience light spawner an agent working in the
`level.*` namespace naturally reaches for. Its `lightType` enum is documented and
implemented as **`'Point' | 'Directional' | 'Spot' | 'Rect'`** only —
`LevelHandler.cpp:508` (`RPC_PARAM_OPT("lightType", "string", "Light kind:
'Point', 'Directional', 'Spot', or 'Rect'. Defaults to 'Point'.")`) and the
`if/else` class-name map at `:518-527` (directional → `DirectionalLight`, spot →
`SpotLight`, rect → `RectLight`, else `PointLight`). There is **no `Sky` branch**,
so a Sky light cannot be spawned through `level.spawn_light`.

Sky lights instead live in a **different namespace**: `lighting.spawn_sky_light`
(`Handlers/Environment/LightingHandler.cpp:352`, namespace `"lighting"`). So the
everyday intent "drop in a few lights, including a Sky light for ambient fill" —
one conceptual gesture, one light set — cannot be expressed against one method.
The agent has to spawn Directional/Point/Spot/Rect via `level.spawn_light`, then
discover and hop to a *second* namespace's wiki page for the sky light.

Note on the sibling `lighting.spawn_light` (`Handlers/Environment/LightingHandler.cpp:75`):
its summary *advertises* `sky` as a lightType, but it does **not** actually spawn a
sky light. It resolves `'sky'` to `ASkyLight::StaticClass()` (`:120-121,148-149`)
and then hits the guard `if (!LightClass || !LightClass->IsChildOf(ALight::StaticClass()))`
(`:157`), which rejects it with `INVALID_ARGUMENT "Invalid light class: SkyLight"` —
because `ASkyLight` derives from `AInfo`, not `ALight`. So today **no** spawn verb
in either namespace will spawn a sky light through a `lightType:'sky'` path; the
only working sky spawner is the dedicated `lighting.spawn_sky_light`. The gap this
ticket addresses is real on its own merits: from a level-namespace vantage point
the sky option is simply invisible.

## Evidence (PROCESS friction, from the audited task)

Task: "Create a fresh empty level 'LightingStudy', then drop in a few lights:
a Directional, a Sky light for ambient fill, and a couple of Point lights." The
outcome was clean (all four lights landed); this is friction that showed up
*despite* success.

- The wiki-nav call-log shows the agent had to read **two** light-spawn wiki
  pages before/while executing: `level.spawn_light.md` **and**
  `lighting.spawn_sky_light.md` (bundled in the multi-page nav entry
  `"level.get_bounds.md / save.md / duplicate.md / list.md / lighting.spawn_sky_light.md"`).
  The sky light is the only one of the four that pulled in a second namespace's
  page.
- Execution then split across namespaces: three `level.spawn_light` calls
  (Directional pitch -45; Point (-400,-300,250); Point (500,350,200)) followed by
  one `lighting.spawn_sky_light` (StudySkyLight z400). One light-set, two
  namespaces.

No wasted `is_error` call and zero blocked progress — pure discoverability /
namespace-coherence overhead (an extra wiki page and a namespace hop for what
reads as one homogeneous "place some lights" step). Hence Low severity, ergonomic.

## What it should do

Make a Sky light reachable from the verb a level-building agent already holds.
Pick one (smallest first):

1. **Ergonomic (preferred):** add a `Sky` branch to `level.spawn_light`'s
   `lightType` map and add `'Sky'` to the documented enum at `LevelHandler.cpp:508`.
   `level.spawn_light` is a thin wrapper that delegates to `actor.spawn` by
   `classPath`; `actor.spawn` resolves any `AActor` subclass (no `ALight`
   restriction — unlike `lighting.spawn_light`'s guarded path, which this fix must
   NOT model itself on), so mapping `'sky'` → `classPath:"SkyLight"` lets
   `actor.spawn` spawn an `ASkyLight`. This makes one method cover the whole
   "drop in a few lights" gesture. (Sky lights take no rotation, so the wrapper
   can pass the `rotation` param through harmlessly — `actor.spawn` just applies
   whatever pose it is given.) The deliberate scope is parity with the other
   convenience light kinds: a bare `ASkyLight` spawn (which captures the scene by
   default). Sky-specific tuning — sourceType, cubemap, recapture — stays the job
   of the dedicated `lighting.spawn_sky_light`.
2. **Docs-only (minimum):** in `docs/wiki-src/level.md` (which today does **not**
   mention `spawn_light`, `lightType`, or `sky` at all — confirmed by grep), add a
   one-line steer on the `level.spawn_light` entry: "Sky lights are not a
   `lightType` here — use `lighting.spawn_sky_light`." That removes the
   second-page hunt even if the enum stays as-is.

This is a guessability/namespace-coherence gap, not a bug — every method behaves
as documented in isolation; the friction is that the *level-namespace* light
spawner silently excludes one common light kind that lives one namespace over.

## History
- `#1-initial-audit` `OPEN` reporter — STRUGGLE (process) audit of the
  "LightingStudy sandbox" task (outcome tool_bug; the `level.get_bounds`
  all-zeros tool bug was filed separately by the judge as
  `B-level-get-bounds-ignores-actors`, the OUTCOME angle). Distinct PROCESS angle:
  the "drop in a few lights incl. a Sky light" gesture could not be done with one
  spawn verb. `level.spawn_light` (`LevelHandler.cpp:485,497-504`) only maps
  Point/Directional/Spot/Rect — no Sky branch — so the agent read a second
  namespace's wiki page (`lighting.spawn_sky_light.md`, per the wiki-nav log) and
  split execution across namespaces (3× `level.spawn_light` + 1×
  `lighting.spawn_sky_light`). Made arbitrary by the sibling `lighting.spawn_light`
  (`LightingHandler.cpp:75`) already documenting `sky` as a lightType. No wasted
  error call, zero blocked progress — pure discoverability/namespace-coherence
  overhead. Dedup (ripgrep over OPEN+closed; qmd unavailable): distinct from
  `F-sky-cloud-reflection-actors` (DONE — SkyAtmosphere/VolumetricCloud/
  ReflectionCapture spawn, a different actor set, not `ASkyLight`) and from
  `E-actor-verbs-reject-actorpath-slot` (OPEN — actor-identity param drift). Fix:
  add a `Sky` branch + enum entry to `level.spawn_light`, or at minimum a
  cross-ref steer in `docs/wiki-src/level.md` (which currently omits `spawn_light`
  entirely).
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORDED then implemented. The
  body's central justifying claim — that sibling `lighting.spawn_light` "already
  accepts sky" so the capability "exists in the surface" — was factually wrong:
  `lighting.spawn_light` advertises `sky` in its summary but rejects `ASkyLight`
  via `if (!LightClass || !LightClass->IsChildOf(ALight::StaticClass()))`
  (`Handlers/Environment/LightingHandler.cpp:157`), since `ASkyLight : public AInfo`
  is not an `ALight`. Reworded the title (dropped "even though sibling … already
  accepts sky"), corrected the stale `LevelHandler.cpp` line numbers (485→508,
  497-504→518-527), fixed the `LightingHandler.cpp` dir to `Handlers/Environment/`,
  and re-grounded the preferred fix on the real mechanism (delegate to `actor.spawn`,
  which has no `ALight` guard — NOT the broken `lighting.spawn_light` path).
  IMPLEMENTED Fix #1: added a `sky`/`skylight` branch mapping to `classPath:"SkyLight"`
  in `level.spawn_light` and `'Sky'` to the documented `lightType` enum, in
  `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:506-540`. Regression test
  `PinWright.level.spawn_light.SpawnsSkyLight` in
  `Source/PinWright/Private/Tests/World/TestLevelHandlers.cpp` spawns
  `lightType:"Sky"` through the production handler into the live editor world and
  asserts a real `ASkyLight` actor appears (the reverted code maps "Sky"→PointLight
  and spawns an `APointLight`, so the test fails on revert). Did not compile/run
  (a later phase does).
- `#2b-wontfix-dissent` developer (fuzz2, superseded by `#2-reword-and-fix`) — A
  parallel host independently reached WONTFIX before seeing the upstream fix; its
  status flip is dropped (the implemented+pushed fix above wins), but its analysis
  is preserved for the one new observation it adds. The ticket's load-bearing
  "arbitrariness" argument is factually false. It claims sibling
  `lighting.spawn_light` "already accepts sky" so omitting Sky from
  `level.spawn_light` is arbitrary. But `lighting.spawn_light` only
  *advertises* sky (docstring `LightingHandler.cpp:75`, enum map
  `:120-121,148-149` → `ASkyLight::StaticClass()`) and then **rejects**
  it at `LightingHandler.cpp:157`: `if (!LightClass ||
  !LightClass->IsChildOf(ALight::StaticClass()))` → `INVALID_ARGUMENT`.
  Engine hierarchy confirms the rejection: `ASkyLight : public AInfo`
  (`C:/UE_5.7/.../Engine/SkyLight.h:11`), NOT a child of `ALight :
  public AActor` (`Light.h:13`). So no existing light-spawn verb actually
  accepts sky except the dedicated `lighting.spawn_sky_light`
  (`LightingHandler.cpp:352`, spawns `ASkyLight` ungated at `:389`).
  That dedicated path works today: the reporter's own task succeeded
  cleanly (all four lights landed, zero blocked progress, no wasted error
  call). With the premise collapsed this reduces to a one-call-over
  discoverability nit at Low/ergonomic severity, fully achievable via
  `lighting.spawn_sky_light`. Adding a third Sky spawn surface to
  `level.spawn_light` to "mirror" a sibling that does not in fact accept
  sky is not warranted. (The genuine latent defect —
  `lighting.spawn_light`'s docstring/enum promising `sky` while the
  `IsChildOf(ALight)` gate silently rejects `ASkyLight` — is a separate
  doc-vs-behavior mismatch not captured by any current ticket; file it
  on its own rather than papering over it by widening
  `level.spawn_light`.) WONTFIX as scoped.
