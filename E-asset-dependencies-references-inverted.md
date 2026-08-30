---
id: E-asset-dependencies-references-inverted
title: "asset.dependencies / asset.references method names, result fields, and a get_dependencies cross-ref are direction-inverted: 'dependencies' returns referencers (inbound), 'references' returns dependencies (outbound)"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [asset, dependencies, references, referencers, direction, naming, result-field, wiki-crossref]
---

# `asset.dependencies` returns referencers and `asset.references` returns dependencies — names + result fields + a wiki cross-ref are all inverted

The two non-`get_`-prefixed asset reference verbs are semantically swapped
end-to-end. Each method's public name and result-field labels are the exact
opposite of the AssetRegistry call it actually makes (and of its own local
variable names):

- `asset.references` — registered summary "Get asset references (what this asset
  depends on)" — actually calls `AssetRegistry.GetDependencies(...)` and returns
  the asset's **outbound dependencies** (what it references), under result fields
  `references` / `referenceCount`. (`UtilityPropertyHandler.cpp:2621`, GetDependencies
  at :2646.)
- `asset.dependencies` — registered summary "Get asset dependencies (what
  references this asset)" — actually calls `AssetRegistry.GetReferencers(...)` and
  returns the asset's **inbound referencers** (who references it), under result
  fields `dependencies` / `dependencyCount`. (`UtilityPropertyHandler.cpp:2673`,
  GetReferencers at :2698.)

So the method literally named `dependencies` is the inbound-referencer verb, and
the one named `references` is the outbound-dependency verb — and the result JSON
doubles down by labeling referencers as `"dependencies"` and dependencies as
`"references"`. The handler bodies are internally honest (local vars are named
`Referencers` and `Dependencies` correctly); only the public surface is flipped.

Two further compounding hazards point an agent the wrong way:

1. **Self-contradictory wiki summaries.** `asset.dependencies` reads "Get asset
   **dependencies** (what **references** this asset)" — the noun and the
   parenthetical describe opposite directions in one line. Same for
   `asset.references`: "Get asset **references** (what this asset **depends
   on**)".
2. **A wrong cross-reference in `asset.get_dependencies`.** Its wiki/registry
   summary says: "Return assets that the given asset directly hard-references
   (Hard package dependencies). For the inverse (who references this asset) use
   `asset.references`; ...". But `asset.references` returns the **same** direction
   (outbound dependencies), not the inverse. The verb that actually answers "who
   references this asset" is `asset.dependencies`. So an agent following the
   documented breadcrumb to find inbound referencers is sent to the wrong method
   and gets outbound deps back — exactly the dead-end the audited task hit.

Net effect: an agent doing a "what would break if I move this asset" audit (the
canonical reason to want inbound referencers) cannot trust the names, the result
fields, OR the one cross-reference that mentions the inverse direction. It only
discovers the right verb (`asset.dependencies`) by trial and inspection of the
returned data, after ~8 exploratory/probe calls per the audited task.

This is filed as ERGONOMIC, not a bug: every call succeeds and returns correct,
complete data for the direction it actually queries — the data is right, only its
names/labels/cross-ref are inverted and actively misleading.

## Repro (verbatim, replay-confirmed against mcp__editor-automation__call)

Root asset `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_ButtonLight_Bulb_Basic`.

1. `asset.dependencies {assetPath: "...BP_ButtonLight_Bulb_Basic"}` →
   ```json
   {"assetPath":"...BP_ButtonLight_Bulb_Basic",
    "packageName":"...BP_ButtonLight_Bulb_Basic",
    "dependencies":[{"packageName":"/Game/Maps/ExampleProjectWelcome"},
                    {"packageName":"/Game/Maps/Blueprint/Blueprint_Communication"}],
    "dependencyCount":2}
   ```
   The two entries are the maps that **reference** the blueprint (inbound
   referencers — the things that would break if it moved), yet the field is
   `"dependencies"` and the method is named `dependencies`.

2. `asset.references {assetPath: "...BP_ButtonLight_Bulb_Basic"}` →
   ```json
   {"references":[{"packageName":"...BP_Light_Bulb_Basic"},
                  {"packageName":".../BP_Button_Parent"},
                  {"packageName":".../BP_Button_Parent_Toggle"}],
    "referenceCount":3}
   ```
   These are the blueprint's outbound **dependencies** (byte-identical to
   `asset.get_dependencies recursive=false`, which returns the same three paths),
   yet the field is `"references"` and the method is named `references`.

3. `asset.get_dependencies` wiki summary (verbatim): "... For the inverse (who
   references this asset) use `asset.references`; ...". Following it lands on the
   call in step 2, which returns outbound deps — NOT the inverse.

## What it should do

Keep the existing verb names and wire fields. Renaming/retiring
`asset.references`/`asset.dependencies` (the originally-filed "Stronger" option)
is **out of scope** — it is dispatcher- and wire-touching, high blast radius, and
speculative. The defect lives entirely in the doc/summary layer plus the
misleading result-field labels, so the fix is exactly that, non-breaking:

- **Fix the three contradictory doc strings** so each noun matches the behavior:
  - `asset.references` → "Get assets this asset references (outbound dependencies
    / hard package dependencies). For the inverse (who references this asset) use
    `asset.dependencies`."
  - `asset.dependencies` → "Get assets that reference this asset (inbound
    referencers). For the inverse (what this asset references) use
    `asset.references`."
  - `asset.get_dependencies` cross-ref → point the "who references this asset"
    breadcrumb at `asset.dependencies`, not `asset.references`. (The wrong
    cross-ref is the worst offender — it is the one breadcrumb an agent is most
    likely to trust.)
- **Add direction-true result-field aliases alongside the legacy keys** (do not
  remove existing keys, for wire compat): `dependencies`/`dependencyCount` on
  `asset.references` (outbound), and `referencers`/`referencerCount` on
  `asset.dependencies` (inbound). The JSON becomes self-describing while every
  existing consumer keeps working.

**Fix:** Correct the two registry summaries + the `asset.get_dependencies`
cross-ref, and emit non-breaking direction-true result-field aliases next to the
legacy keys. No verb rename, no removed keys.

## History
- `#1-initial-repro` `OPEN` reporter — Found during a dependency-audit task on
  `BP_ButtonLight_Bulb_Basic` (seed `asset.get_asset_graph`; the graph verb
  itself worked correctly — this finding is about neighbor verbs
  `asset.dependencies` / `asset.references`). Replay-confirmed against
  `mcp__editor-automation__call`: `asset.dependencies` returned the 2 inbound
  referencer maps labeled `"dependencies"`/`"dependencyCount"`; `asset.references`
  returned the 3 outbound deps labeled `"references"`/`"referenceCount"` (== the
  three `asset.get_dependencies recursive=false` paths). Root cause in source:
  `UtilityPropertyHandler.cpp` — `asset.references` (:2621) calls
  `GetDependencies` (:2646); `asset.dependencies` (:2673) calls `GetReferencers`
  (:2698) — names/result-fields are the inverse of the registry call and the
  local var names in both. Compounded by two self-contradictory registry
  summaries and a wrong `asset.get_dependencies` cross-ref ("For the inverse ...
  use asset.references" — but asset.references is the same direction). The audited
  attempt burned ~8 exploratory/probe calls hunting the inbound-referencer verb
  before discovering `asset.dependencies` does it despite its name. NOT a tool bug
  — data is correct for the direction each verb queries; the names, result-field
  labels, and cross-ref are misleading. Prior transient review note of the same
  naming smell exists in `logs/codex-cycles/0006-.../agent2-executor-*.txt`
  (ISSUE-1, severity LOW) but was never filed to the board; this is the first
  board entry, with the added cross-ref/result-field evidence.
- `#2-mrq-reverse-lookup-workaround` `OPEN` reporter — Cross-task evidence (focus `mrq.list_presets`, clean outcome). Task needed the inbound referencer of a level sequence — "locate the map/World it plays in" for `/Game/ExampleContent/MouseInterface/MoustInteraction_Master`. Rather than trust `asset.references` (or the wrong `asset.get_dependencies` "use asset.references for the inverse" cross-ref documented in #1), the agent worked around the direction-ambiguous verbs entirely: it GUESSED a map name (`/Game/Maps/Blueprint/Blueprint_Mouse_Interaction`) and probed it with `asset.exists`, then ran FORWARD `asset.get_dependencies` on that guessed map to confirm it hard-references the sequence — a guess-then-confirm substitute for the inbound-referencer query the inverted/mistrusted `asset.references` should have answered in one call. Self-reported friction was "none" (all calls green), so this surfaced only on process audit: the inverted naming silently steers agents to indirection instead of the direct reverse lookup. Same root cause as #1; adds a second namespace/seed (`mrq`) and a distinct workaround shape (guess+exists+forward-deps) to the evidence.
- `#4-task-plan-breadcrumb-forced-extra-call` `IN-REVIEW` reporter — Cross-task
  process evidence (focus `asset.get_dependencies`, outcome `tool_bug` for a
  *separate* graph defect; this entry is the inverted-naming PROCESS friction).
  A DemoRoom material-library cleanup task on `MI_Metal_Light` had an explicit
  step 6: "for the parent material `M_Metal`, run `asset.references` to find who
  ELSE in the project references that parent — this tells me whether
  deleting/moving the instance is safe." That is a textbook **inbound-referencer**
  ("what would break if I move this") intent, and the success-check the task
  carried named `asset.references(parent)` as the verb that should list the
  instance among the parent's referencers. But `asset.references` is OUTBOUND:
  `asset.references {assetPath:"/Game/Global/DemoRoom/Materials/M_Metal.M_Metal"}`
  returned `M_Metal`'s own texture + MaterialFunction dependencies and did **not**
  include `MI_Metal_Light`. The agent could not satisfy the "who references the
  parent" check from that call, so it had to add an unplanned
  `asset.dependencies {assetPath:"...M_Metal.M_Metal"}` which returned the 32 real
  inbound referencers (meshes, blueprints, maps, and `MI_Metal_Light` itself),
  confirming the parent is heavily shared and unsafe to move. Friction note,
  verbatim: "success-check (d) names the wrong method: it expects
  asset.references(parent) to include the instance as a referencer, but the wiki
  and observed behavior make asset.references OUTBOUND ...; the inverse 'who
  references this' is asset.dependencies, which I had to add to actually confirm
  MI_Metal_Light is among M_Metal's 32 referencers." Same root cause as #1–#3;
  adds a third workaround shape — a downstream consumer (the task's own
  success-check / plan breadcrumb) wired the inbound-referencer intent to the
  outbound-named `asset.references`, costing one extra recovery call
  (`asset.dependencies`) — independent corroboration that the inverted names
  actively mis-route "safe to move?" audits even when the prose breadcrumb is
  external to the wiki. Confirms the #3 doc/alias fix targets the right surface;
  no new fix needed. Object-path form of `M_Metal` was tolerated by both reference
  verbs here (orthogonal to the #3-fixed `get_dependencies` suffix bug).
- `#3-reword-and-fix-docs-aliases` `IN-REVIEW` developer — Reworded to drop the
  speculative verb-rename/retire option (dispatcher/wire-touching, out of scope);
  scope is now the doc-string + non-breaking result-alias fix only, matching the
  adversarial-lens finding that the ticket over-scoped. Implemented: (1) corrected
  the two self-contradictory registry summaries in
  `Source/EditorAutomationRpcGateway/Private/Handlers/Utility/UtilityPropertyHandler.cpp`
  — `asset.references` now reads "Get assets this asset references (outbound
  dependencies / hard package dependencies). For the inverse (who references this
  asset) use asset.dependencies."; `asset.dependencies` now reads "Get assets that
  reference this asset (inbound referencers). For the inverse (what this asset
  references) use asset.references."; (2) fixed the wrong cross-ref in
  `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetQueryHandler.cpp`
  (`asset.get_dependencies`) so the "who references this asset" breadcrumb points
  at `asset.dependencies` instead of `asset.references`; (3) added non-breaking
  direction-true result-field aliases next to the legacy keys — `asset.references`
  now also emits `dependencies`/`dependencyCount` (outbound), `asset.dependencies`
  now also emits `referencers`/`referencerCount` (inbound); no existing keys
  removed. Local var names were already honest, so the bodies were untouched apart
  from the alias-field emission. Regression test added:
  `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAssetReferenceDirection.cpp`
  (`EditorAutomationRpcGateway.asset.references.DirectionAndAliases`) — asserts the
  corrected registered summaries from `FAutoRegisterHandler::GetPendingRegistrations()`,
  builds a real on-disk A→B hard dependency (same fixture pattern as
  `TestAssetReferencersWarning`), and checks that `asset.references(A)` reports B
  as an outbound dep under both `references` and the new `dependencies` alias while
  `asset.dependencies(B)` reports A as an inbound referencer under both
  `dependencies` and the new `referencers` alias. Reverting any of the three
  doc-string edits or removing the aliases fails the test. Not compiled/tested in
  this phase.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 3 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
