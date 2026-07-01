---
id: E-spawn-category-name-discovery
title: "debug.spawn_category's only category-name guidance is a wrong in-tool example ('Behavior' should be 'BehaviorTree') with no way to discover valid names — case-sensitivity + the silent-noop bug make a typo unrecoverable from the RPC, forcing an engine-source dive"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, debug, gameplay-debugger, spawn-category, discovery, case-sensitive, misleading-example]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `debug.spawn_category` gives a wrong, case-sensitive category-name example and no way to discover the real names

`debug.spawn_category` takes one required param, `categoryName`, whose only
authoring guidance is the in-tool param description
(`DebugHandler.cpp:7`):

> `"Identifier of a registered gameplay debugger category (e.g. 'AI', 'EQS', 'Behavior'); case-sensitive."`

Two problems compound here for a caller whose intent is "turn on the
BehaviorTree overlay":

1. **The example is wrong.** The engine-registered Gameplay Debugger category
   is `BehaviorTree`, not `Behavior` (registered in the AIModule —
   `FGameplayDebuggerCategory_BehaviorTree` under the name `"BehaviorTree"`).
   The param doc's own example string `'Behavior'` does not name a real
   category.
2. **The doc emphatically declares the field `case-sensitive`** while handing
   the caller the wrong case/spelling. So a caller who trusts the example
   passes `Behavior`, which is silently a typo.

Because there is **no `debug.list_categories` / enumeration RPC** and the
namespace overlay (`docs/wiki-src/debug.md`) is a one-line blurb that names no
categories, the only authoritative source of the real names is reading the
engine source (`AIModule`'s category registrations: `Perception`,
`BehaviorTree`, `EQS`, etc.). There is no in-band way to discover them.

The wrong example is normally a soft cost, but it is made **unrecoverable from
the RPC** by the sibling silent-noop bug
(`B-spawn-category-silent-noop-fake-existsafter`, OPEN): a typo'd or
wrong-case category name returns the identical success-shaped
`{"commandExecuted":false,"existsAfter":true}` as a real one, so a caller who
copies `'Behavior'` from the doc gets a clean success and no signal that
nothing toggled. The example error can never surface as an error.

## What it should do

Two cheap, independent docs/ergonomic fixes (the result-reporting bug itself is
`B-spawn-category-silent-noop-fake-existsafter`'s job — this ticket is the
*authoring/discovery* angle):

1. **Fix the in-tool example** in `DebugHandler.cpp`'s `categoryName`
   `RPC_PARAM_REQ` description: replace `'Behavior'` with the real registered
   name `'BehaviorTree'` (and keep the others valid: `Perception`, `EQS`).
   A wrong example on a field the same sentence calls "case-sensitive" is the
   active footgun.
2. **Surface the valid category names where callers look.** Extend the
   `docs/wiki-src/debug.md` overlay (the `debug.spawn_category` page is
   generated from the handler) to list the common engine-registered categories
   verbatim and case-exact — at minimum `Perception`, `BehaviorTree`, `EQS`,
   `Navmesh`, `AbilitySystem` — and note that names are case-sensitive and that
   the set comes from the registered Gameplay Debugger categories (engine +
   plugins), so this list is illustrative not exhaustive. Optionally add a
   `debug.list_categories` enumeration RPC so the names are discoverable
   in-band rather than by reading AIModule.

Either of (1)+(2) removes the source dive; together with the bug ticket's
existsAfter/validation fix, a typo would also surface as an error instead of a
fake success.

## Evidence

From the gameplay-debugger overlay struggle audit (namespace `debug`, outcome
`tool_bug`). Friction note, verbatim:

> *"the wiki's spawn_category example says 'Behavior' but the engine-registered
> name is 'BehaviorTree' — I verified exact case-sensitive names
> (Perception/BehaviorTree/EQS) in UE AIModule.cpp to avoid a silent mismatch."*

Call-log: the task ran 9 `debug.spawn_category` calls (Perception/BehaviorTree/EQS
on, Perception off→on, all three off) — all returned success — but the friction
was a **pre-call source dive**: the agent did not trust the in-tool `'Behavior'`
example and instead read `AIModule.cpp` to confirm the exact case-sensitive
category names before issuing any toggle, precisely because the silent-noop
behavior meant a wrong name would never have surfaced as an error. The mis-named
example cost a discovery step (engine-source read) rather than a wasted RPC,
exactly because the bug masks the failure path. Confirmed in-handler:
`DebugHandler.cpp:7` carries the literal `e.g. 'AI', 'EQS', 'Behavior'`.

Distinct from `B-spawn-category-silent-noop-fake-existsafter` (OPEN), which owns
the *result-reporting* defect (hardcoded `existsAfter:true`, `commandExecuted:false`
on success); that ticket only mentions the wrong example name in passing as one of
its fix options. This ticket is the *authoring/discoverability* friction: a wrong,
case-sensitive in-tool example with no in-band way to find the right names. Same
"obvious guess has no on-page cue" family as `E-actor-list-no-class-filter` (OPEN)
and `E-viewport-screenshot-name-discovery` (OPEN), but here the cue that exists is
actively *wrong*, not merely absent.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the debug gameplay-debugger overlay struggle audit (outcome tool_bug). `debug.spawn_category`'s only category-name guidance is the in-tool param example `e.g. 'AI', 'EQS', 'Behavior'` (`DebugHandler.cpp:7`), declared `case-sensitive` — but the real engine-registered name is `BehaviorTree`, not `Behavior`, and there is no `debug.list_categories` or wiki list of valid names. The silent-noop bug (`B-spawn-category-silent-noop-fake-existsafter`) makes a wrong-name typo return the same fake success as a real toggle, so the example error can never surface. The agent's friction was a pre-call engine-source dive (read AIModule.cpp to confirm exact case-sensitive names Perception/BehaviorTree/EQS) rather than a wasted RPC. Proposed: docs/ergonomic — fix the `'Behavior'`→`'BehaviorTree'` example in the handler param doc and list valid categories (Perception/BehaviorTree/EQS/Navmesh/AbilitySystem, case-exact) in `docs/wiki-src/debug.md`; optionally add an in-band `debug.list_categories` enumeration. Tagged `docs`; overlay to edit is `docs/wiki-src/debug.md`. Distinct from the bug ticket (result-reporting) — this is the authoring/discovery angle.
