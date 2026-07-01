---
id: E-niagara-standard-stack-recipe-undocumented
title: "Building a standard particle stack costs many trial-and-error search_modules calls — no canonical recipe / module-path list, and keyword matching misses obvious terms"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, niagara, search-modules, add-module, module-discovery, stack-authoring]
encounters: 3
lastSeen: 2026-06-24T06:10:13Z
---

# Building a standard particle stack from scratch forces guess-and-check `search_modules` discovery

The "make a basic particle effect from scratch" intent — create system, create
emitter, add it, then add the four near-universal stack modules
(**SpawnRate** → EmitterUpdate, **InitializeParticle** → ParticleSpawn,
**AddVelocity** → ParticleSpawn, **SolveForcesAndVelocity** → ParticleUpdate)
plus a sprite renderer — is the single most common Niagara authoring task. But
there is no documented canonical recipe that names these four engine module
asset paths or the stack group each belongs to. The agent must rediscover all
four at runtime through `niagara.search_modules`, and the keyword matching is
finicky enough that several natural queries return nothing, producing a
guess-and-check loop.

In this task (`/Game/VFX/NS_FloatingEmbers`, a plain floating-embers effect),
resolving those four module paths took **9 `niagara.search_modules` calls**, of
which three were dead ends caused by query phrasing, not by the module being
absent:

- `Module 'spawn rate' -> 0 results` (wrong: the script is under EmitterUpdate
  usage; "spawn rate" as a free Module query found nothing — only
  `EmitterUpdate 'spawn'` surfaced `SpawnRate`)
- `Module 'initialize particle' -> ribbon only` (the obvious term returned only a
  ribbon variant; `ParticleSpawn 'initialize'` was needed to get the real
  `InitializeParticle`)
- `ParticleUpdate 'solve forces velocity position' -> 0 results` and
  `ParticleUpdate 'solve forces' -> 0` before the bare token
  `'SolveForcesAndVelocity'` finally matched the solver path

So the caller had to iterate on both the `usage`/`stage` scoping **and** the
keyword spelling — collapsing a multi-word description down to the exact
PascalCase module leaf name — before each of the four modules resolved. That is
discovery friction layered on top of an RPC (`niagara.search_modules`, shipped by
`F-search-api-niagara-modules`) that already works: the RPC is fine, but with no
documented recipe and no robustness against descriptive multi-word queries, the
common case is still guess-and-check.

This is the Niagara-stack analogue of
[`E-effect-create-niagara-systempath-discovery`](E-effect-create-niagara-systempath-discovery.md)
(which covers the `effect.create_*` `systemPath` discovery burden) — same class of
PROCESS friction (a routine create intent forces several discovery calls for
values that are effectively constants), different code path.

**Workaround:** Know the four standard engine module paths up front and pass them
straight to `niagara.add_module`:
`/Niagara/Modules/Emitter/SpawnRate.SpawnRate` (EmitterUpdate),
`/Niagara/Modules/Spawn/Initialization/InitializeParticle.InitializeParticle`
(ParticleSpawn),
the `AddVelocity` and `SolveForcesAndVelocity` update/spawn modules — verify the
exact leaf paths against a stock UE 5.7 install before relying on them, then skip
`search_modules` entirely for the canonical stack.

**Fix (docs-first, downstream wiki process — not done here):** In
`docs/wiki-src/niagara.authoring.md`, add a short `## Standard particle stack`
recipe under the existing Workflow section that (a) lists the four canonical
engine module asset paths above with the stack group each attaches to and a
one-line "minimal visible particle system" call sequence
(create_system → create_emitter → add_emitter → 4× add_module → add_renderer
SpriteRenderer → set_module_input → compile/validate/save), and (b) adds a
`search_modules` usage note that the query matches the module's **leaf/display
name** scoped by `usage` + `stage`, so a free-text multi-word description
(e.g. "spawn rate", "solve forces velocity position") often returns 0 — scope to
the right `usage`/`stage` and search a single distinctive token (or the
PascalCase leaf) instead. Confirm every listed path ships with a stock UE 5.7
install before publishing. A secondary RPC-side angle worth a follow-up: make
`search_modules` keyword matching tolerant of multi-word descriptive queries
(split on whitespace / fuzzy-match against name+keywords) so "spawn rate" finds
`SpawnRate` without the caller pre-collapsing it to one token.

## History
- `#3-additional-order-sensitive-phrase` `OPEN` reporter — Additional evidence (separate "campfire-ember VFX from scratch" fuzz task, `/Game/VFX/FX_CampfireEmbers`) confirming the literal-phrase matcher is also **order-sensitive**, and that the conjunction of two individually-matching tokens returns 0. Replayed live against `mcp__editor-automation__call`: `niagara.search_modules {query:"spawn rate", usage:"Module"}` -> `{"results":[],"totalMatches":0}`; capitalized `{query:"Spawn Rate", usage:"Module"}` -> `0`; yet each token alone matches `SpawnRate`: `{query:"spawn", usage:"Module"}` includes `/Niagara/Modules/Emitter/SpawnRate` (53 results) and `{query:"rate", usage:"Module"}` includes `SpawnRate` (33 results, score 100) and the bare leaf `{query:"SpawnRate", usage:"Module"}` -> exactly that asset (score 1000). So both tokens of "spawn rate" hit SpawnRate individually, but their conjunction returns 0 — proving the match is not a token-AND. A clean order-sensitivity discriminator on a *second* module: `{query:"gravity force", usage:"Module"}` -> `/Niagara/Modules/Update/Forces/GravityForce` (score 50), but the reversed `{query:"force gravity", usage:"Module"}` -> `0` (same two tokens, swapped) — and "gravity force" itself is not a contiguous substring of the name "GravityForce" (no space) nor of the description "Applies a gravitational force" ("gravitational", not "gravity"), so even the lone two-word success is fragile/coincidental. Net: the matcher treats `query` as an ordered literal phrase against name+description; analogous canonical modules behave inconsistently ("gravity force" finds GravityForce, "spawn rate"/"Spawn Rate" finds nothing). Same `'spawn rate' -> 0` symptom and culprit RPC `niagara.search_modules`; reinforces the #1/#2 follow-up (split on whitespace / order-insensitive AND-match against name+keywords). Task otherwise completed clean (system+emitter+renderer+2 modules built, compile/validate/save all green); discovery cost ~6 search_modules calls due to the phrasing retries.
- `#2-additional-spawn-burst-phrase` `OPEN` reporter — Additional evidence (separate "spark-burst muzzle-flash from scratch" fuzz task, `/Game/VFX/SparkBurst`): the multi-word `query` is matched as a **literal phrase/substring**, not tokenized, so a natural two-word query whose tokens both occur in the target's name+description still returns 0. Replayed live against `mcp__editor-automation__call`: `niagara.search_modules {query:"spawn burst", usage:"Module"}` -> `{"results":[],"totalMatches":0}`; reversed `{query:"burst spawn", usage:"Module"}` -> `0`; yet `{query:"burst", usage:"Module", stage:"EmitterUpdate"}` -> 3 results topped by `/Niagara/Modules/Emitter/SpawnBurst_Instantaneous` (description verbatim: `"Spawns a burst of particles instantaneously."`, score 100), and the bare leaf `{query:"SpawnBurst", usage:"Module"}` -> exactly that asset (score 500). So "spawn burst" — which describes `SpawnBurst_Instantaneous` perfectly and matches BOTH its name tokens — finds nothing only because the literal string "spawn burst" (with the space) never appears; the asset is `SpawnBurst` (no space) / "Spawns a burst". This confirms the #1-proposed follow-up (split on whitespace / AND-match tokens against name+keywords) on a second module, and there is no hint in the empty result that the query was treated as a phrase. Same class as the `'spawn rate' -> 0` symptom; culprit RPC `niagara.search_modules`.
- `#1-initial-audit` `OPEN` reporter — Surfaced as PROCESS friction in a "build a floating-embers Niagara system from scratch" fuzz task (`/Game/VFX/NS_FloatingEmbers`). Resolving the four standard stack module paths (SpawnRate/EmitterUpdate, InitializeParticle/ParticleSpawn, AddVelocity/ParticleSpawn, SolveForcesAndVelocity/ParticleUpdate) cost 9 `niagara.search_modules` calls, three of them dead ends from query phrasing rather than a missing module: `Module 'spawn rate' -> 0`, `Module 'initialize particle' -> ribbon only`, `ParticleUpdate 'solve forces velocity position' -> 0` (and `'solve forces' -> 0`) before the bare PascalCase token matched. The `search_modules` RPC itself works (shipped by F-search-api-niagara-modules); the gap is no documented canonical stack recipe / module-path list and no robustness against descriptive multi-word queries, so the most common Niagara create intent is guess-and-check. Distinct from the judge-filed tool bug `B-niagara-set-module-input-vec2` (the FVector2D set_module_input defect) and from the DONE `F-search-api-niagara-modules` (built the RPC). Analogue of `E-effect-create-niagara-systempath-discovery` for the stack-build path. Page to improve: `docs/wiki-src/niagara.authoring.md`.
</content>
</invoke>
