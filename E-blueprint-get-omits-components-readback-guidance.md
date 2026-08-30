---
id: E-blueprint-get-omits-components-readback-guidance
title: "`blueprint.get` registration summary advertises a `components` field it never emits — drop it and route component readback to `blueprint.scs.get`"
status: IN-REVIEW
severity: Low
category: bug
tags: [docs-contract, blueprint, blueprint-get, blueprint-scs-get, components, readback, discovery]
---

# `blueprint.get`'s own summary promises components, but the handler never emits them

A caller who has just added a component to a Blueprint class (e.g.
`ai.add_smart_object_component` adding `ParkBenchSmartObject` to `BP_ParkBench`)
and wants to *confirm the component is present* reaches for `blueprint.get` as
the obvious "read the blueprint back" verb. In practice `blueprint.get` returns
only `variables` / `functions` / `events` and **no `components` field at all** —
so a "verify the component via `blueprint.get`" check is unsatisfiable, and the
caller has to fall back to `blueprint.scs.get` (or `blueprint.inspect`).

The root contradiction is **in the source, not just the wiki**: the registered
handler summary at `BlueprintInfoHandler.cpp:52` advertised
`"Return summary metadata for a Blueprint: parent class, variables, functions,
events, components. Lighter than blueprint.inspect …"` — listing `components`
as a returned field. That summary string is the agent-facing discovery surface
returned by `call("blueprint.get")` and the source of the auto-generated
`## Methods` index / per-method page (there is no `### blueprint.get` H3 overlay
to override it). Meanwhile `BuildBlueprintSnapshot`
(`BlueprintHandlerUtils.cpp:1066-1086`) emits only
`variables`/`functions`/`events` (+ optional `metadata`), and the registry
merge in the handler (`BlueprintInfoHandler.cpp:86-154`) only folds in
`defaults`/`metadata`/`functions`/`events` — never a `components` field. So the
tool's own self-description was an **active false claim**, not merely a docs
omission. The reporter's friction note ("blueprint.get is documented to return
components but returned NO components field") points exactly at this summary.

The data itself is **not lost** — class-level component templates are reachable
via `blueprint.scs.get` — so this stays Low severity; the cost is one corrective
call plus a self-contradicting discovery surface. This is the same shape the
board has already fixed at the source string elsewhere (cf.
`B-inspect-object-omits-component-properties`, DONE: "registration doc promises
a field the handler omits" → fix/align the contract).

The wiki (`Docs/wiki-src/blueprint.md`) nudged the same wrong way and is a
secondary fix surface:

- The `### blueprint.inspect` fan-out line described `blueprint.inspect` as
  replacing "`blueprint.get` + `blueprint.scs.get` + per-graph …", reading as
  though `blueprint.get` contributes the component surface.
- The same section called `blueprint.get` "a lighter summary (no graphs, no
  references)" — naming the two things it drops but not that it also drops
  components, so a reader infers components survive the "lighter summary."

**Workaround:** read class-level components with `blueprint.scs.get` (component
templates with names, classes, and `DefinitionRef`/property refs) or with
`blueprint.inspect`. Reserve `blueprint.get` for the
variables/functions/events summary.

**Fix:** (primary, code) correct the `blueprint.get` registration summary at
`BlueprintInfoHandler.cpp:52` so it no longer lists `components` as a returned
field and instead routes component readback to `blueprint.scs.get` (class
templates) / `blueprint.inspect` (full structural dump). (secondary, docs) in
`Docs/wiki-src/blueprint.md`: clarify the `blueprint.inspect` fan-out line that
the component surface in that fan-out comes from `blueprint.scs.get` (not
`blueprint.get`); add "no components" to the "lighter summary (no graphs, no
references)" phrasing; and add a `### blueprint.get` H3 overlay that states
`blueprint.get` omits components and routes component / hierarchy readback to
`blueprint.scs.get`. (regression) a unit test that fails if the summary re-adds
the `components` promise OR if a live `blueprint.get` response ever grows a
`components` field while still claiming none.

## Evidence
Task `ai.add_smart_object_component` (park-bench smart object). After
`ai.add_smart_object_component {blueprint:BP_ParkBench, component:ParkBenchSmartObject, definition:SOD_ParkBench}`
reported success, the readback step `blueprint.get {BP_ParkBench}` "returned NO
components field (only vars/funcs/events)" — quoting the task friction note:
"blueprint.get is documented to return components but returned NO components
field, so the literal success check (verify component via blueprint.get) is
unsatisfiable — I had to confirm the SmartObject component via
blueprint.scs.get instead." The follow-up `blueprint.scs.get {BP_ParkBench}`
returned `ParkBenchSmartObject` (SmartObjectComponent, `DefinitionRef=SOD_ParkBench`)
correctly. So this is a 1-extra-call docs/discovery cost, not a data loss.
Distinct from `B-get-components-renders-empty-to-caller` (that is
`actor.get_components` rendering empty due to large-payload MCP transport, DONE)
and from `B-inspect-object-omits-component-properties` (that is
`system.inspect.inspect_object` never emitting a `properties` field, DONE) —
this one is purely the `blueprint.get` ↔ `blueprint.scs.get` readback-routing
documentation for class components.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of task `ai.add_smart_object_component`. The readback `blueprint.get {BP_ParkBench}` returned no `components` field (only vars/funcs/events), making the task's "verify component via blueprint.get" success check unsatisfiable; the caller fell back to `blueprint.scs.get` (which correctly showed `ParkBenchSmartObject` / `DefinitionRef=SOD_ParkBench`). `docs/wiki-src/blueprint.md` ~line 185 lists `blueprint.get` as part of the component-bearing fan-out and ~line 189 calls it a "lighter summary (no graphs, no references)" without noting components are also omitted, steering callers to the wrong read verb. Docs-only ergonomic gap; overlay page to fix = `docs/wiki-src/blueprint.md`. Distinct PROCESS angle from the judge-filed tool bug `B-configure-slot-behavior-ignores-behavior-and-tags`, and from the (DONE) `B-get-components-renders-empty-to-caller` / `B-inspect-object-omits-component-properties` which are other methods/causes.
- `#2-inspect-omits-hierarchy-evidence` `OPEN` reporter — Cross-task evidence (struggle-audit of `blueprint`/BP_InteractiveDoor build, 15 calls, outcome clean). Same readback-routing friction, now seen on `blueprint.inspect` itself: the agent ran `blueprint.compile` → `blueprint.inspect {/Game/Blueprints/BP_InteractiveDoor}` to confirm structure, but inspect's components list surfaces the component *names* (DoorFrame, DoorPanel) without the parent/child *links*, so it could not confirm "DoorPanel parented under DoorFrame" from inspect and made a 16th corrective call `blueprint.scs.get` to verify the hierarchy. Friction note verbatim: "used scs.get to confirm hierarchy since blueprint.inspect's components list omits parent/child structure." Reinforces the docs fix on `docs/wiki-src/blueprint.md`: the overlay recommends `blueprint.inspect` as the structural entry point, but inspect's component list does not express the SCS tree — the wiki should route *hierarchy* confirmation specifically to `blueprint.scs.get` (whose `scs.txt`/`children {}` is the hierarchy view). Note the underlying scs.get hierarchy emission is itself being fixed in `B-scs-get-local-child-parent-link-missing` (IN-REVIEW); this ticket stays the docs/discovery layer that points callers to scs.get in the first place.
- `#3-reword-and-fix-registration-summary` `IN-REVIEW` developer — Reworded (docs-only/ergonomic → code-contract bug): the real root cause is the `blueprint.get` registration summary at `BlueprintInfoHandler.cpp:52` actively advertising a `components` field that `BuildBlueprintSnapshot` (`BlueprintHandlerUtils.cpp:1066-1086`) and the handler's registry merge (`BlueprintInfoHandler.cpp:86-154`) never emit — that summary string is the agent-facing discovery surface and the source of the auto-generated method page, so a wiki-only edit would have left the worse lie in place (same shape as DONE `B-inspect-object-omits-component-properties`). Fix: (1) CODE — rewrote the `blueprint.get` summary to drop "components" from the returned-field list and route component readback to `blueprint.scs.get`/`blueprint.inspect` (`Source/EditorAutomationRpcGateway/Private/Handlers/Blueprint/BlueprintInfoHandler.cpp:52`). (2) DOCS — `Docs/wiki-src/blueprint.md`: clarified the `### blueprint.inspect` fan-out line that the component surface comes from `blueprint.scs.get` (not `blueprint.get`); added "no components" to the "lighter summary (no graphs, no references)" phrasing; added a `### blueprint.get` H3 overlay stating it omits components and routing component/hierarchy readback to `blueprint.scs.get`. (3) TEST — `FBlueprintGetDoesNotAdvertiseOrEmitComponentsTest` (`EditorAutomationRpcGateway.blueprint.get.NoComponentsContract`) in `Source/EditorAutomationRpcGateway/Private/Tests/Blueprint/TestBlueprintHandlers.cpp`: asserts the registered summary no longer lists `components` as a returned field (and, if components are mentioned at all, only as a "not include components" routing note pointing at `blueprint.scs.get`), and invokes the live `blueprint.get` handler on a transient BP to assert the response has `variables`/`functions`/`events` but NO `components` field — fails if either the summary re-adds the promise or the handler grows a contradicting field. (corrected ticket factual nit from `#1`: `B-get-components-renders-empty-to-caller` is IN-REVIEW, not DONE — not material.)
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. 2 citations sit in history rows and are left verbatim per the append-only rule. The one citation is in history row `#3` and stays verbatim. Map: `Handlers/Blueprint/BlueprintInfoHandler.cpp:52` → `Source/PinWright/Private/Handlers/Blueprint/BlueprintInfoHandler.cpp:55`, a 3-line drift; the described summary is there verbatim (“Does NOT include components — for class-level component templates use `blueprint.scs.get`, or `blueprint.inspect`…”). Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
