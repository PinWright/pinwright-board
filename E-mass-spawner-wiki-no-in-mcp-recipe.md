---
id: E-mass-spawner-wiki-no-in-mcp-recipe
title: "ai wiki advertises 'Mass entity configuration' but documents no working Mass-spawner recipe — add_mass_spawner's component framing is a phantom-success dead end, and the real in-MCP path (blueprint.reparent -> AMassSpawner + set_default Count / property.set EntityTypes[].EntityConfig on the CDO) is undocumented, forcing a C++ source-dive + 6-call workaround"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [ai, mass-entity, mass-spawner, docs, wiki, discoverability, dead-end, no-in-mcp-recipe]
encounters: 1
lastSeen: 2026-07-02T00:12:02.5572365+03:00
---

# The `ai` wiki over-promises Mass-spawner authoring and never documents the working route

The `ai` namespace overview (`docs/wiki-src/ai.md:3`) lists "**Mass entity
configuration**" as a flat capability of the namespace. A reader scanning the
prelude reads an unqualified working-capability claim — but **no page in the
`ai` wiki documents how to actually wire a Mass spawner**. The obvious
first-class verb an agent reaches for, `ai.add_mass_spawner`, is framed as
component-based (default `componentName` "MassSpawner", param doc "Name for the
spawner component") and is a **phantom-success no-op stub** (owned by the
per-finding bug ticket `B-add-mass-spawner-silent-noop`). The component framing
is itself conceptually wrong: engine UE 5.7 Mass has **no `UMassSpawnerComponent`**
— the spawner is the `AMassSpawner` **actor** (`Count` +
`EntityTypes[].EntityConfig`), which is why `blueprint.scs.get(componentClass=
MassSpawner)` legitimately rejects it with "must derive from UActorComponent".

So the discovery surface steers a caller straight onto a dead verb and gives no
pointer at what actually works. The real in-MCP recipe — which the agent in this
task had to reconstruct by **reading the plugin's `AIHandler.cpp` as a last
resort** and applying engine knowledge — is:

1. `blueprint.reparent` the target Blueprint to `AMassSpawner`
   (`/Script/MassSpawner.MassSpawner`).
2. `blueprint.set_default {propertyName:"Count", value:<N>}` on the CDO.
3. `property.set {path:"EntityTypes", ...}` with
   `EntityTypes[0].EntityConfig = <MEC config>` (and `Proportion:1`) on the CDO.
4. Verify with `property.get` on `Count` and `EntityTypes[0].EntityConfig`.

None of that appears anywhere in `docs/wiki-src/ai.md`.

## Distinct from the bug ticket

`B-add-mass-spawner-silent-noop` (OPEN, per-finding judge) owns the **C++ honesty
defect** — `ai.add_mass_spawner` fabricates a success payload that echoes
`spawnCount`/`configPath` while doing nothing but `MarkPackageDirty()`+save; its
fix options are "implement it honestly" or "fail loud with NOT_IMPLEMENTED." This
ticket is the **docs-overlay companion** that survives *either* of those fixes: a
fail-loud `NOT_IMPLEMENTED` still leaves the caller without a documented working
alternative, and even an honest implementation wants the `ai` prelude qualified
and the `AMassSpawner` recipe written down. Same accepted shape as
`E-widget-style-workflow-wiki-advertises-stub`,
`E-spawn-category-not-implemented-no-in-mcp-recipe`,
`E-level-structure-wp-wiki-advertises-dead-end`, and
`E-ai-bt-authoring-verbs-dead-end`: the namespace advertises a path that
dead-ends, and the fix is a `docs/wiki-src/` overlay edit (downstream wiki
process, not this audit) naming the working in-MCP route.

## Process evidence (this task)

Focus `ai.create_mass_entity_config` (Mass village-crowd authoring). The two
config creates + `ai.configure_mass_entity` (Villager inherits base) worked
cleanly. The friction was entirely on the spawner wiring:

- `ai.add_mass_spawner {blueprintPath:BP_VillagerSpawner, configPath:MEC_Villager,
  componentName:MassSpawner, spawnCount:250}` → affirmative success echoing
  `count=250`/`cfg` — but the immediately-following `blueprint.scs.get` showed
  only `DefaultSceneRoot` (count:1) and `asset.dump` confirmed the SCS/CDO
  unchanged: a silent no-op.
- The agent then read `AIHandler.cpp` (**its own last-resort source-dive**) to
  confirm the handler only marks dirty + saves, then pivoted to the undocumented
  recipe: `blueprint.reparent -> AMassSpawner`, `blueprint.set_default Count=250`,
  `property.set EntityTypes[0].EntityConfig=MEC_Villager`, re-save, and 3
  `property.get` readbacks — **6 extra RPCs + a C++ dive** to reach green.
- `blueprint.scs.get {componentClass:MassSpawner}` correctly errored
  `[INVALID_ARGUMENT] componentClass must derive from UActorComponent: MassSpawner`
  — a downstream symptom of the wrong component framing, not a separate bug.

Friction note (verbatim): *"ai.add_mass_spawner is a phantom-success STUB ... I
had to read the plugin AIHandler.cpp (last resort) to confirm the no-op, then
pivot to blueprint.reparent -> AMassSpawner + property.set/blueprint.set_default
on the CDO."* The cost is a **process** one, separable from the underlying stub:
the working route was inferable-not-documented, so recovery cost a source-dive
plus a 6-call reparent+CDO workaround.

## What the wiki should do

In `docs/wiki-src/ai.md`:
- Qualify the prelude so "Mass entity configuration" is not an unqualified
  working-capability claim for spawner *wiring* (`ai.add_mass_spawner` wires
  nothing).
- Add a namespace-page-visible `## ` section (below the prelude, mirroring the
  existing EQS / StateTree / Behavior Tree up-front redirects) documenting the
  real Mass-spawner path: a Mass spawner is the **`AMassSpawner` actor**, not a
  component — reparent the spawner Blueprint to `AMassSpawner`
  (`/Script/MassSpawner.MassSpawner`) via `blueprint.reparent`, then set `Count`
  via `blueprint.set_default` and `EntityTypes[0].EntityConfig` (+ `Proportion`)
  via `property.set` on the CDO, and **verify with `property.get`**. Note that
  `ai.add_mass_spawner`'s component framing does not apply
  (`blueprint.scs.get(componentClass=MassSpawner)` will reject it).

severity rationale: impact=discoverability — the wiki advertises the capability while the real recipe is undocumented, steering the agent into a last-resort C++ source-dive + 6-call workaround (worse than a Read) × reach=rare (experimental Mass Entity crowd-authoring path) -> Medium (matches the retriaged `E-widget-style-workflow-wiki-advertises-stub` precedent: wiki advertises a capability backed by a silent no-op stub, forcing a source-dive on a niche path)

## History
- `#2-wiki-overlay-fix` `IN-REVIEW` developer — Implemented the `docs/wiki-src/ai.md` overlay fix (three edits, no code behavior change): (1) narrowed the prelude claim `Mass entity configuration` → `Mass entity config assets` so the namespace/root index no longer over-promises spawner wiring (create/configure MEC assets is what actually works); (2) added a namespace-page-visible `## Mass spawner authoring` section documenting that a Mass spawner is the `AMassSpawner` **actor** (UE 5.7 has no `UMassSpawnerComponent`, so `blueprint.scs.get(componentClass=MassSpawner)` rightly rejects it) and the real in-MCP recipe — `blueprint.reparent` → `AMassSpawner` (`/Script/MassSpawner.MassSpawner`), `blueprint.set_default Count`, `property.set EntityTypes[0].EntityConfig`(+`Proportion`) on the CDO, verify with `property.get`; (3) added a point-of-use `### ai.add_mass_spawner` H3 warning the verb does not wire a spawner (phantom-success no-op) and redirecting to that route. Same accepted overlay shape as the already-shipped `## Behavior Tree authoring` redirect in this same file (`E-ai-bt-authoring-verbs-dead-end`) and the `E-texture`/`E-widget` stub-overlay precedents. Distinct from the C++ bug `B-add-mass-spawner-silent-noop` (that owns the handler honesty defect); this overlay content survives either B outcome. Regression test `Tests/Infra/TestAiMassSpawnerAuthoringDocs.cpp` renders the live `WikiHandler::RenderPage` path for both the `ai` namespace page and the `ai.add_mass_spawner` method page and asserts overlay-exclusive markers (`AMassSpawner`, `UMassSpawnerComponent`, `EntityTypes`, `/Script/MassSpawner.MassSpawner`, `blueprint.reparent`, `does **not** wire`, `Mass entity config assets`) that appear in no registration summary — reverting any edit fails an assertion. Tests: `PinWright.infra.wiki_handler.Namespace.AiDocumentsMassSpawnerAuthoring` + `PinWright.infra.wiki_handler.MethodPage.AddMassSpawnerRedirect`.
- `#1-initial-audit` `OPEN` reporter — Struggle/process audit of focus `ai.create_mass_entity_config` (Mass village-crowd task; the focus creates + `ai.configure_mass_entity` inheritance worked cleanly). Distinct PROCESS/discoverability angle from the judge's C++ bug `B-add-mass-spawner-silent-noop`: `docs/wiki-src/ai.md:3` advertises "Mass entity configuration" but no page documents the working Mass-spawner recipe, and the obvious verb `ai.add_mass_spawner` is a phantom-success no-op with a wrong component framing (engine has no `UMassSpawnerComponent`). The agent source-dived `AIHandler.cpp` (last resort) then did a 6-call `blueprint.reparent -> AMassSpawner` + `set_default Count` / `property.set EntityTypes[].EntityConfig` CDO workaround to reach green. Survives even a fail-loud fix of the bug ticket (a NOT_IMPLEMENTED still wants a documented alternative). Same accepted overlay shape as `E-widget-style-workflow-wiki-advertises-stub` / `E-spawn-category-not-implemented-no-in-mcp-recipe` / `E-level-structure-wp-wiki-advertises-dead-end` / `E-ai-bt-authoring-verbs-dead-end`. Dedup: ripgrep board for `add_mass_spawner`/`AMassSpawner`/`mass.entity`/`no-in-mcp-recipe`/`advertises` — only `B-add-mass-spawner-silent-noop` (the C++ bug) and the unrelated `E-ai-*` tickets exist; no Mass-spawner docs ticket. Names page to edit: `docs/wiki-src/ai.md`.
