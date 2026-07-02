---
id: E-bt-attach-service-blackboard-name-misleads
title: "behavior_tree.attach_service rejects 'Blackboard' (a documented short name) with a dead-end INVALID_CLASS, and the wiki lumps it in for both verbs despite the decorator/service asymmetry"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [behavior-tree, attach-service, ai, class-resolution, error-message, discoverability, docs]
---

# `attach_service serviceClass:"Blackboard"` fails with a dead-end error, and the wiki lists "Blackboard" as a valid service name even though no concrete `UBTService_Blackboard` exists

A user asking to "attach a **Blackboard service** to the Selector so the AI
periodically re-checks its target" is using the canonical BT phrasing for the
blackboard-backed service. The obvious call —
`behavior_tree.attach_service {serviceClass:"Blackboard"}` — fails with:

```
[INVALID_CLASS] Could not resolve service class 'Blackboard'.
```

This is a **discovery dead-end**, and it is made worse by two things:

1. **The wiki actively endorses the failing name.** The `behavior_tree`
   overlay (`docs/wiki-src/behavior_tree.md:46`) reads: *"Short native class
   names are accepted with or without their native prefix, so `Blackboard`,
   `BTDecorator_Blackboard`, `DefaultFocus`, and `BTService_DefaultFocus`
   resolve to the matching AIModule classes…"* — lumping all four names
   together for both `attach_decorator` and `attach_service`. But there is an
   **asymmetry the docs hide**: `UBTDecorator_Blackboard` is concrete (so
   `attach_decorator decoratorClass:"Blackboard"` works — and is the exact case
   the DONE ticket `F-bt-attach-decorator-service-to-parent` verified), whereas
   the service side is `UBTService_BlackboardBase` declared `UCLASS(Abstract)`
   — there is **no concrete `UBTService_Blackboard`**. So the same short name
   "Blackboard" resolves for the decorator verb and is unresolvable for the
   service verb. A reader of the wiki would reasonably expect
   `attach_service serviceClass:"Blackboard"` to work; it cannot.

2. **The error names no recovery path.** `[INVALID_CLASS] Could not resolve
   service class 'Blackboard'` does not say the class is abstract, does not list
   the concrete blackboard-backed service the user almost certainly wants
   (`DefaultFocus` / `UBTService_DefaultFocus`, which re-checks its key while
   relevant), and offers no "did you mean / available services" hint. The agent
   recovered only by reading engine C++ to confirm
   `BTService_BlackboardBase` is abstract, then guessing `DefaultFocus`. That is
   out-of-band knowledge a tool user should not need.

## What it should do

Both of:

- **Fix the docs.** In `docs/wiki-src/behavior_tree.md` "Decorators and
  services" section, stop presenting `Blackboard` as a service name. Note the
  asymmetry explicitly: `Blackboard` is a *decorator* (gates a branch on a
  blackboard key); the blackboard-backed *service* is `DefaultFocus`
  (`BTService_DefaultFocus`) — there is no `BTService_Blackboard` because the
  AIModule service base `UBTService_BlackboardBase` is `UCLASS(Abstract)`. Name
  the concrete stock services (`DefaultFocus`, `RunEQS`) so callers stop
  guessing.
- **Improve the error.** On a failed `serviceClass`/`decoratorClass`
  resolution, the `INVALID_CLASS` error should carry an `availableClasses` list
  enumerating the concrete (non-abstract) native subnode names for the requested
  base, so the caller can pick a real one without reading engine C++. This
  reuses the board's established enumerate-concrete-candidates convention
  (`AIHandler.cpp` ~1890, `GetDerivedClasses` + `CLASS_Abstract` filter for
  SmartObject `behaviorType`). No `Blackboard`-specific "did you mean
  DefaultFocus" special-casing and no separate abstract-class detection — the
  generic candidate list naturally omits the abstract base and names
  `DefaultFocus`.

This is a pure PROCESS/ergonomic friction — the underlying `attach_service`
handler works correctly for valid names (it was verified in
`F-bt-attach-decorator-service-to-parent`). It is **distinct from** the bug the
per-finding judge filed for this same task,
`B-bt-set-node-properties-silent-noop` (silent no-op on a *different* method);
here the call correctly *errors*, but the error and the docs send the caller
down a dead end instead of toward the concrete class.

**Workaround:** Use `serviceClass:"DefaultFocus"` (the concrete
`UBTService_DefaultFocus`) for a blackboard-backed service that re-checks its
key while relevant. Reserve `Blackboard` for `attach_decorator` only.

## Evidence

From this task's friction note (BT_GuardAI authoring, namespace
`behavior_tree`): *"attach_service rejected the short name 'Blackboard'
([INVALID_CLASS]) because stock AIModule has no concrete BTService_Blackboard
(BTService_BlackboardBase is UCLASS(Abstract)); I confirmed via engine C++ and
used DefaultFocus, the concrete Blackboard-backed service that re-checks its key
while relevant."*

Call log: one `attach_service {Blackboard on Selector}` → `is_error:true`
(`[INVALID_CLASS] Could not resolve service class 'Blackboard'.`) immediately
followed by a corrected `attach_service {DefaultFocus on Selector}` → `ok:true`.
A misuse-then-correct pair (1 failed + 1 working variant) driven by docs that
endorse the failing name and an error that names no alternative.

## History
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded to the defensible scope (dropped the over-specified `Blackboard`→`DefaultFocus` special-casing and the inaccurate "resolver lands on an abstract class" framing — the resolver never returns the abstract base for input "Blackboard"; it is a plain not-found). Implemented BOTH options. (1) Docs: `docs/wiki-src/behavior_tree.md` "Decorators and services" now states the decorator/service asymmetry explicitly — `Blackboard`→concrete `UBTDecorator_Blackboard` for `attach_decorator`, but no `BTService_Blackboard` (abstract `UBTService_BlackboardBase`) so `attach_service` takes `DefaultFocus`/`RunEQS` — and notes the failing call's `availableClasses` data. (2) Error: `Source/PinWright/Private/Handlers/AI/BehaviorTreeHandler.cpp` — added `EnumerateConcreteBTSubNodeNames(base, prefix)` (`GetDerivedClasses` + `CLASS_Abstract|Deprecated|NewerVersionExists` filter, native-only, prefix-stripped, sorted), and reworked the `INVALID_CLASS` site in `HandleAttachBTSubNode` to attach an `availableClasses` array and a "no concrete native … class matches; see availableClasses" message (generic to both verbs, reusing the `AIHandler.cpp` ~1890 convention; new includes `UObject/UObjectHash.h`, `Utils/JsonUtils.h`). Regression test `Source/PinWright/Private/Tests/Gameplay/TestBehaviorTreeAttachSubnodes.cpp` → `FBTAttachServiceBlackboardHintsConcreteNamesTest` (`PinWright.behavior_tree.attach_service.RejectsBlackboardWithConcreteHint`): drives the real `attach_service` handler with `serviceClass:"Blackboard"`, asserts `INVALID_CLASS` + an `availableClasses` array that names `DefaultFocus` and omits `Blackboard`, and confirms the same short name still resolves for `attach_decorator`. Fails if the enumeration is reverted (no `availableClasses` → the "names DefaultFocus" assertion fails).
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the BT_GuardAI authoring task. `behavior_tree.attach_service {serviceClass:"Blackboard"}` failed `[INVALID_CLASS] Could not resolve service class 'Blackboard'`, agent recovered to `DefaultFocus` only after reading engine C++ to learn `UBTService_BlackboardBase` is `UCLASS(Abstract)` (no concrete `UBTService_Blackboard`). Two coupled process frictions: (a) `docs/wiki-src/behavior_tree.md:46` lists `Blackboard` among names that "resolve to the matching AIModule classes" without flagging that it resolves for `attach_decorator` (concrete `UBTDecorator_Blackboard`) but NOT `attach_service` (abstract base); (b) the `INVALID_CLASS` error names no concrete alternative (`DefaultFocus`) and does not say the class is abstract. Propose docs fix (drop `Blackboard` from the service-name list, name `DefaultFocus`, list concrete stock services) and/or an error-message improvement that enumerates concrete candidates. Distinct from judge-filed `B-bt-set-node-properties-silent-noop` (different method, silent success vs this correct-but-dead-end error). Evidence: 1 failed `attach_service`(Blackboard) + 1 working `attach_service`(DefaultFocus) in the call log; friction note quoted above.
