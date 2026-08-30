---
id: E-navmesh-config-requires-bounds-volume-prereq
title: "configure_nav_mesh_settings / set_nav_agent_properties fail [NO_NAVMESH] until a NavMeshBoundsVolume exists — the ordering prerequisite is undocumented"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [navigation, configure_nav_mesh_settings, set_nav_agent_properties, no-navmesh, prerequisite, ordering, discoverability, docs]
---

# `navigation.configure_nav_mesh_settings` / `set_nav_agent_properties` need a RecastNavMesh to already exist, and nothing documents that a NavMeshBoundsVolume is the way to create one

Both `navigation.configure_nav_mesh_settings` and
`navigation.set_nav_agent_properties` mutate generation/agent settings on the
level's **RecastNavMesh** nav-data actor. On a level that has no
RecastNavMesh yet (the common starting state for an empty/fresh level), both
hard-fail with:

`[NO_NAVMESH] No RecastNavMesh found in level`

The RecastNavMesh is not something these methods (or any obvious
`create_*` verb) materialize on demand — it is **auto-registered by the
navigation system as a side effect of placing a `NavMeshBoundsVolume`**. So
the real authoring order for "set up nav for this level" is:

`volume.create_nav_mesh_bounds_volume` (auto-registers the nav data) →
`configure_nav_mesh_settings` → `set_nav_agent_properties` → … →
`rebuild_navigation`.

A caller following the natural intent — "configure the nav mesh, then the
nav agent, then add links, then rebuild" — reaches for
`configure_nav_mesh_settings` **first**, eats `[NO_NAVMESH]`, then has to
discover (not from the docs) that a `NavMeshBoundsVolume` is the prerequisite
that brings the RecastNavMesh into existence. The error text names *what* is
missing ("No RecastNavMesh found in level") but not *how* to create one, so it
doesn't point at the `volume.create_nav_mesh_bounds_volume` remedy.

This is the same overlay-omits-the-precondition failure mode already tracked
for other namespaces:
[E-gas-execution-capture-attribute-set-prereq-undocumented](E-gas-execution-capture-attribute-set-prereq-undocumented.md)
(a hidden `blueprint.compile` step before captures resolve) and
[E-session-wiki-pie-prerequisite-undocumented](E-session-wiki-pie-prerequisite-undocumented.md)
(a live-PIE prerequisite for the local-player/split-screen methods) —
here the missing precondition is "a NavMeshBoundsVolume must exist so a
RecastNavMesh is registered" before the two nav-config setters resolve.

The `navigation` wiki overlay (`docs/wiki-src/navigation.md`) is a 4-line
descriptive stub. It enumerates what the namespace covers ("Recast nav mesh
settings, area costs, link endpoints, … navigation rebuilds") but says
**nothing** about which methods require a pre-existing RecastNavMesh, that
`[NO_NAVMESH]` is the failure when none exists, or that a `NavMeshBoundsVolume`
is what creates one. So the ordering dependency is discoverable only by eating
the failed call.

This is a **process/discoverability** gap, not a wrong result — both methods
behave correctly once the bounds volume exists, the task outcome was clean, and
the cost is two wasted round-trips plus a recovery `create_nav_mesh_bounds_volume`
on the first nav-setup of a fresh level.

**Evidence (this task — focus `navigation.get_navigation_info`, outcome clean,
12 calls):** the agent ran the baseline `get_navigation_info`, then called
`configure_nav_mesh_settings {tile 1000, cell 19, height 10, step 35}` →
`[NO_NAVMESH] No RecastNavMesh found in level`, then
`set_nav_agent_properties {radius 35, height 144, slope 44, step 35}` →
the same `[NO_NAVMESH]`. It recovered by calling
`volume.create_nav_mesh_bounds_volume` (`PatrolNavBounds @500,0,60
ext2000x2000x500`), which auto-registered the nav data, then **retried both**
`configure_nav_mesh_settings` and `set_nav_agent_properties` successfully, and
the rest of the flow (`create_nav_link_proxy`, `configure_nav_link`,
`rebuild_navigation` + `job_status` poll, final `get_navigation_info`) ran
clean. Friction note (verbatim): "Mild: configure_nav_mesh_settings and
set_nav_agent_properties both failed first with [NO_NAVMESH] because the level
had no RecastNavMesh; I had to create a NavMeshBoundsVolume first
(auto-registers the nav data) then retry — that ordering dependency isn't
documented on those two wiki pages." Call-log cost: 2 failed setter calls + 1
recovery `create_nav_mesh_bounds_volume` + 2 retries — i.e. two wasted
round-trips and an out-of-order volume placement, all to recover an
undocumented prerequisite.

**Fix (downstream wiki process; optional error sharpening):** extend the
`docs/wiki-src/navigation.md` overlay to state that
(a) `configure_nav_mesh_settings` and `set_nav_agent_properties` (and any
method that mutates RecastNavMesh generation/agent settings) require a
RecastNavMesh nav-data actor to already exist in the level, and fail with
`[NO_NAVMESH] No RecastNavMesh found in level` otherwise; and
(b) the way to bring a RecastNavMesh into existence is to place a
`NavMeshBoundsVolume` via `volume.create_nav_mesh_bounds_volume`, which
auto-registers the nav data — so the level-nav-setup order is
`create_nav_mesh_bounds_volume → configure_nav_mesh_settings →
set_nav_agent_properties → … → rebuild_navigation`. Optionally sharpen the
`[NO_NAVMESH]` error text to name the remedy ("create a NavMeshBoundsVolume
first"), the way `[NO_GAME_INSTANCE]` already says "Start Play-In-Editor
first." This is a wiki edit (plus an optional one-line error string), not a
behavior change.

## History
- `#2-wiki-and-error-sharpen` `IN-REVIEW` developer — Documented the prerequisite + authoring order and sharpened the `[NO_NAVMESH]` error to name the remedy (parity with `[NO_GAME_INSTANCE]`'s "Start Play-In-Editor first."). (a) Wiki: extended `docs/wiki-src/navigation.md` with a `## Prerequisite: a RecastNavMesh must exist before you configure it` section — states that `configure_nav_mesh_settings`/`set_nav_agent_properties` require a pre-existing RecastNavMesh (else `[NO_NAVMESH] No RecastNavMesh found in level`), that placing a `NavMeshBoundsVolume` via `volume.create_nav_mesh_bounds_volume` auto-registers the nav data, and the level-nav-setup order (`create_nav_mesh_bounds_volume → configure_nav_mesh_settings → set_nav_agent_properties → … → rebuild_navigation`); the section lives below the prelude so it renders on the namespace page but stays out of the root index per the wiki overlay rules. (b) Code: sharpened both `[NO_NAVMESH]` `SendError` strings (`Source/EditorAutomationRpcGateway/Private/Handlers/AI/NavigationHandler.cpp:157` and `:253`) to append "Create a NavMeshBoundsVolume first (volume.create_nav_mesh_bounds_volume) — it auto-registers the nav data." Error code/behavior unchanged. (c) Regression test `EditorAutomationRpcGateway.navigation.no_navmesh_error_names_remedy` added in `Source/EditorAutomationRpcGateway/Private/Tests/Gameplay/TestAIHandlers.cpp` — invokes both setters via `InvokeHandlerWithCapture`, and when the `NO_NAVMESH` branch is hit asserts `Capture.Message` contains "NavMeshBoundsVolume" and "create_nav_mesh_bounds_volume" (gated on the branch so it never flakes if the test world carries a nav mesh); reverting the message sharpening fails it. Wiki edit is documentation-only; no behavior change.
- `#3-field-evidence-error-sharpen-worked` `OPEN` reporter — Cross-task evidence (process-audit of a second, independent two-tier scout-drone nav-setup task, seed `navigation`, outcome clean, 12 calls): confirms the `#2` error-sharpening lands in the field. This task's first `navigation.set_nav_agent_properties` (r24/h90/slope50/step45) returned the sharpened text verbatim — `[NO_NAVMESH] No RecastNavMesh found in level. Create a NavMeshBoundsVolume first (volume.create_nav_mesh_bounds_volume) — it auto-registers the nav data.` — and the agent recovered in a **single** call by following exactly that remedy (`volume.create_nav_mesh_bounds_volume` ScoutNavBounds @500,150,125 ext2000x2000x1000), then retried the setter successfully; the rest of the flow ran clean. Friction note (verbatim): "set_nav_agent_properties first failed with [NO_NAVMESH] because the level had no RecastNavMesh; the error message clearly directed me to volume.create_nav_mesh_bounds_volume, which auto-registers nav data, so one extra prerequisite call resolved it cleanly." Contrast with `#1`'s pre-sharpen task, where the remedy was discoverable only by self-investigation (2 failed setters + a recovery volume + 2 retries). Cost here is down to 1 failed setter + 1 recovery volume + 1 retry — the error text now carries the remedy, so the only residual friction is the (still-undocumented-in-overlay-as-first-step) ordering prerequisite this ticket's wiki edit addresses. Net: the code half of this ticket (`#2` error sharpening) is field-validated; the wiki-overlay half remains the open work (note: `#2` already added the prereq section to `docs/wiki-src/navigation.md`, so this is effectively verification pending). The same task's *async-rebuild* friction (`rebuild_navigation` returns a ticket needing `system.job_status`, not flagged on the overlay) is the orthogonal angle filed separately as `E-navigation-rebuild-async-poll-undocumented`.
- `#1-initial-audit` `OPEN` reporter — Process/struggle audit of a level-nav-setup task (focus `navigation.get_navigation_info`, outcome clean, 12 calls). On a fresh level, the first `navigation.configure_nav_mesh_settings` and `navigation.set_nav_agent_properties` both returned `[NO_NAVMESH] No RecastNavMesh found in level`; the agent recovered by placing a `NavMeshBoundsVolume` via `volume.create_nav_mesh_bounds_volume` (which auto-registers the nav data), then retried both successfully. Friction note (quoted): "configure_nav_mesh_settings and set_nav_agent_properties both failed first with [NO_NAVMESH] because the level had no RecastNavMesh; I had to create a NavMeshBoundsVolume first (auto-registers the nav data) then retry — that ordering dependency isn't documented on those two wiki pages." Call-log cost: 2 failed setter calls + 1 recovery `create_nav_mesh_bounds_volume` + 2 retries. The `navigation` wiki overlay (`docs/wiki-src/navigation.md`, 4-line stub) documents no RecastNavMesh prerequisite and never names the NavMeshBoundsVolume as the way to create one, so the ordering dependency is discoverable only via the failed call. Same overlay-omits-the-precondition failure mode as `E-gas-execution-capture-attribute-set-prereq-undocumented` and `E-session-wiki-pie-prerequisite-undocumented`, different precondition (a registered RecastNavMesh via NavMeshBoundsVolume). Distinct from the existing nav tickets: `E-nav-modifier-create-no-areaclass-echo` (readback echo on create_nav_modifier_component) and `B-navigation-rebuild-navigation-no-completion-signal` (async completion signal). E-/docs: works once the bounds volume exists; the gap is discoverability + two wasted round-trips. The per-finding judge saw outcome clean and filed nothing (filed_id empty). No existing board ticket on the navmesh-config bounds-volume prerequisite (ripgrep across OPEN + closed; qmd unavailable). Proposes documenting the prerequisite + authoring order on `docs/wiki-src/navigation.md`, with optional `[NO_NAVMESH]` error sharpening to name the remedy.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. 2 citations sit in history rows and are left verbatim per the append-only rule. The one citation is in a history row and stays verbatim. Map: `Handlers/AI/NavigationHandler.cpp:157` → `Source/PinWright/Private/Handlers/AI/NavigationHandler.cpp:178`, and its sibling `:253` → `:274`; `:157` at HEAD is an unrelated NOT_FOUND guard. The row's claim holds — both sites now pass the shared literal `NoNavMeshRemedyMsg` (`:40`) carrying the `volume.create_nav_mesh_bounds_volume` remedy. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
