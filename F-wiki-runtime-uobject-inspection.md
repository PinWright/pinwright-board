---
id: F-wiki-runtime-uobject-inspection
title: "Wiki: end-to-end runtime UObject inspection page (PIE, subsystems, properties)"
status: DONE
severity: Medium
category: feature
tags: [wiki, documentation, system-inspect, property, pie, subsystem, runtime-inspection]
---

# Wiki: end-to-end runtime UObject inspection page (PIE, subsystems, properties)

Agents repeatedly rediscover the "inspect a live UObject in PIE" workflow by
trial-and-error because the relevant caveats are scattered across (or missing
from) the `system.inspect` and `property` wiki overlays. Each individual
discovery gap has its own sibling code-fix ticket; this ticket tracks the
**documentation-only** mitigation that helps agents in the meantime.

The current `system.inspect.md` overlay frames everything around editor-world
state. It never mentions PIE, never mentions subsystems, and never warns that
`property.list` silently hides non-editable UPROPERTYs. As a result every
session that needs to read live runtime state (e.g. confirming an
`eServerType` enum on `UApiSubsystem` mid-PIE) repeats the same trial-and-error
loop:

1. `system.inspect.list_objects` / `system.inspect.find_by_class` only return
   editor-world actors. PIE worlds are invisible to them.
2. Subsystems are not actors. There is no first-class MCP discovery RPC for
   live subsystems today; `python.execute` is the only escape hatch.
3. The live subsystem object path shape (once you have it) is
   `/Engine/Transient.<EditorEngine>:<GameInstanceName>.<SubsystemClassName>_<index>`
   — undocumented and non-obvious.
4. `property.list` silently filters out non-editable UPROPERTYs. A `count: 0`
   response does **not** mean "no state on this object" — it means "nothing on
   this object is `EditAnywhere`/`VisibleAnywhere`".
5. `property.get` works on any reflected UPROPERTY by name, even ones that
   `property.list` hid. Falling back to a direct named `property.get` is the
   workaround.
6. `system.call_subsystem` only dispatches `UFUNCTION`-decorated methods.
   Plain C++ methods are unreachable through this RPC.
7. `inspect_class` returns name / path / parent only — it does **not**
   enumerate member properties or functions. Agents looking at it for "what
   can I read on this class?" come away empty-handed.

**Fix:** add a new wiki overlay page (proposed path
`Plugins/EditorAutomationRpcGateway/docs/wiki/runtime-uobject-inspection.md`)
that documents the workflow end-to-end. The page should include:

- The seven caveats above, each with a one-line "what to do instead".
- A worked end-to-end example: locate the live `UApiSubsystem` instance in an
  active PIE session, then read its `eServerType` enum via `property.get` on
  the resolved object path. Show the actual call sequence (including the
  `python.execute` fallback used to enumerate subsystems today) and the
  expected response shapes.
- Cross-references to `system.inspect`, `system.inspect.inspect_object`,
  `system.inspect.inspect_class`, `property`, `property.list`, `property.get`,
  and `system.call_subsystem`.

**Acceptance check:**

- `mcp__editor-automation__call("system.inspect")` output includes a paragraph
  (or bullet) about PIE-world behavior and links to the new
  `runtime-uobject-inspection` page.
- The new page exists, contains all seven caveats, the worked
  `UApiSubsystem.eServerType` example, and the cross-references listed above.

**Scope guard:** this ticket is **documentation only**. The underlying code
gaps (no PIE world coverage in `list_objects`/`find_by_class`, no subsystem
discovery RPC, `property.list` filtering non-editable UPROPERTYs without
notice, `inspect_class` not returning members, `system.call_subsystem`
UFUNCTION-only) each have or warrant their own `B-` / `F-` tickets and are
out of scope here.

## History
- `#1-initial-report` `OPEN` reporter — No end-to-end wiki page for the "inspect a live UObject in PIE" workflow. Agents repeatedly rediscover the seven listed caveats (PIE invisibility of list_objects/find_by_class, no subsystem discovery RPC, undocumented live subsystem path shape, silent property.list filtering of non-editable UPROPERTYs, property.get fallback for hidden properties, UFUNCTION-only system.call_subsystem, inspect_class returning no members). Proposes a new wiki overlay page at `docs/wiki/runtime-uobject-inspection.md` containing the caveats, a worked `UApiSubsystem.eServerType`-in-PIE example, and cross-references to system.inspect / property / system.call_subsystem; plus a one-paragraph PIE-aware update to `system.inspect.md` that links to the new page.
- `#2-add-runtime-inspection-page` `IN-REVIEW` developer — Added `docs/wiki/runtime-uobject-inspection.md` topic page documenting the end-to-end PIE UObject inspection workflow (world targeting, subsystem discovery, property reads, UFUNCTION caveat) with a worked `UApiSubsystem.eServerType` example; updated `system.inspect.md` and `property.md` to cross-link. Page reflects post-sprint state (sibling fixes assumed landed).
- `#3-verify-partial-routing-missing` `OPEN` tester — File `docs/wiki/runtime-uobject-inspection.md` exists on disk, but `mcp__editor-automation__call path="runtime-uobject-inspection"` returns `Not found` with fuzzy suggestions; the page is not reachable through the wiki router. Either (a) standalone topic pages (i.e. pages whose path is not a namespace dotted name) need to be registered with the catalog/wiki resolver so `call("runtime-uobject-inspection")` resolves, or (b) the page is intended only as a cross-ref target and `system.inspect.md`/`property.md` overlays should advertise it explicitly. Sibling system.inspect/property cross-links from #2 need a recheck once routing is decided. Reopening pending one of those two outcomes.
- `#4-docs-only-cross-link-resolution` `IN-REVIEW` developer — Resolved with option (b) cross-linking. `system.md`, `python.md`, and `wiki/README.md` now advertise `runtime-uobject-inspection.md` as a deliberate topic-page exception. Filed sibling `F-wiki-router-standalone-topic-pages` for the option-(a) routing fix that would make `call("runtime-uobject-inspection")` resolve directly. Fixed two small link inconsistencies on the topic page itself.
- `#5-verify-cross-links` `DONE` tester — Verified: `call("system.inspect")` renders the PIE-aware paragraph and links to `runtime-uobject-inspection.md`; local overlays `system.md`, `python.md`, `property.md`, and `wiki/README.md` advertise the topic page/cross-ref-only routing exception, and `runtime-uobject-inspection.md` contains the worked `UApiSubsystem.eServerType` example plus the requested cross-references.
