---
id: E-configure-slot-behavior-behaviortype-undiscoverable
title: "ai.configure_slot_behavior wiki page enumerates no behaviorType values and omits the gameplay_tags.add prereq for activityTags — discoverable only by a wrong-guess retry plus reading C++"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [ai, smart-object, configure-slot-behavior, behaviortype, activity-tags, gameplay-tags, discoverability, docs]
---

# `ai.configure_slot_behavior` is undiscoverable from its docs: no enumerated `behaviorType` values, no `activityTags` registration prereq

`ai.configure_slot_behavior` takes a `behaviorType` whose only schema/wiki
description is the bare phrase **"Type of behavior"** — it enumerates no accepted
values. A caller's natural first guess (a plausible class name like
`SmartObjectGameplayBehaviorDefinition`) is rejected, and the accepted set is not
written down anywhere the caller looks; it has to be learned by either eating the
`[INVALID_PARAMS]` (which now usefully echoes `availableBehaviorTypes`) or by
reading the plugin's `AIHandler.cpp`. The same page also never states that
`activityTags` must be registered via `gameplay_tags.add` first (the runtime now
rejects unregistered tags after the `B-configure-slot-behavior-ignores-behavior-and-tags`
fix). So a from-scratch smart-object-slot author hits two undocumented gates on
the natural path.

This is the **docs/discoverability** angle, distinct from the two behavior
tickets this same task already produced:
- `B-configure-slot-behavior-ignores-behavior-and-tags` (IN-REVIEW) is the
  **runtime bug** fix — it made `behaviorType` actually attach a behavior and
  made bad values / unregistered tags fail loud with `availableBehaviorTypes` /
  `droppedTags`. That fix improves the *error path* but does not document the
  accepted `behaviorType` values or the tag prereq up front, so the caller still
  learns them only by tripping the (now-loud) error.
- `E-asset-dump-instanced-uobject-class-elided` (the per-finding judge's
  `filed_id`) is the **readback** gap (the attached behavior dumps as `{}`).

It is the same **overlay-omits-the-precondition / no-enumerated-values** failure
mode already tracked for sibling namespaces:
[E-state-tree-onevent-tag-registration-prereq](E-state-tree-onevent-tag-registration-prereq.md)
(an `OnEvent` trigger needs the event tag registered first, not stated on the
overlay) and
[E-gas-set-effect-tags-drops-unregistered](E-gas-set-effect-tags-drops-unregistered.md)
(its `#3` history adds the same "register-first prereq not flagged on the wiki
page" docs angle). Here the precondition is "registered `activityTags`" and the
extra gap is "no enumerated `behaviorType` values."

The `ai` wiki overlay (`docs/wiki-src/ai.md`, 31 lines) documents perception /
blackboard / behavior-tree readback in detail but mentions **neither**
`configure_slot_behavior`, its `behaviorType` accepted values, **nor** the
`gameplay_tags.add` prerequisite for `activityTags` (verified: ripgrep of
`behaviorType|slot|activitytag|smart.?object` over `ai.md` returns only the
namespace blurb, no `configure_slot_behavior` section).

## Evidence (this task — focus `ai.configure_slot_behavior`, outcome clean, 15 calls)

Coffee-station smart-object task. The author pre-registered three tags via
`gameplay_tags.add` (already having learned the tag prereq), then:

- `ai.configure_slot_behavior {slot0, behaviorType:"SmartObjectGameplayBehaviorDefinition", tags:coffee}`
  → **`[INVALID_PARAMS] Unknown behaviorType 'SmartObjectGameplayBehaviorDefinition'.
  No concrete USmartObjectBehaviorDefinition subclass matches; see
  availableBehaviorTypes. availableBehaviorTypes: /Script/SmartObjectsTestSuite.SmartObjectTestBehaviorDefinition,
  /Script/MassSmartObjects.SmartObjectMassBehaviorDefinition`** — wrong guess rejected.
- `ai.configure_slot_behavior {slot0, behaviorType:"/Script/MassSmartObjects.SmartObjectMassBehaviorDefinition", tags:coffee, enabled}`
  → ok (picked from the error's `availableBehaviorTypes`).

Friction note (verbatim): "behaviorType has no enumerated values in the wiki
(only 'Type of behavior'), and activityTags silently-reject path required
registering tags via gameplay_tags.add first — I learned both by reading the
plugin's AIHandler.cpp C++ (a last-resort discoverability gap). My first guessed
behaviorType 'SmartObjectGameplayBehaviorDefinition' was rejected, but the error
usefully returned availableBehaviorTypes so I picked the loaded
/Script/MassSmartObjects.SmartObjectMassBehaviorDefinition on the retry."

Process cost: one wasted `configure_slot_behavior` (wrong-guess `behaviorType`) +
the up-front C++ read to learn both gates. The `availableBehaviorTypes` echo
softened it to a single retry rather than a dead-end — but only because the agent
read the error; a value list on the page would have avoided the wrong guess
entirely.

## What it should do

Extend the `docs/wiki-src/ai.md` overlay with a `configure_slot_behavior` note
that:
(a) states the `behaviorType` accepted values are concrete
`USmartObjectBehaviorDefinition` subclasses identified by `/Script/Module.Class`
path (or short alias), that the loadable set depends on which optional plugins
are linked, and that an unknown value is rejected with `[INVALID_PARAMS]` whose
`availableBehaviorTypes` lists the currently-resolvable classes (so "pass an
empty/throwaway value once and read `availableBehaviorTypes`" is the documented
discovery move) — and names a sensible default like
`/Script/MassSmartObjects.SmartObjectMassBehaviorDefinition`;
(b) states that `activityTags` must already be registered via `gameplay_tags.add`
or the call is rejected with `[INVALID_PARAMS]` + `droppedTags`, so the authoring
order is `gameplay_tags.add` → `configure_slot_behavior`.
This is a downstream wiki-overlay edit (not mine to apply); no behavior change —
the runtime already errors correctly after the `B-` fix.

## History
- `#2-already-resolved` `IN-REVIEW` developer — Already fixed in current committed source; no code written. The requested docs fix is present at HEAD: `Docs/wiki-src/ai.md:33-57` is a full `### ai.configure_slot_behavior` H3 overlay covering both asks — (a) `behaviorType` accepts a concrete `USmartObjectBehaviorDefinition` subclass by `/Script/Module.Class` path, names the default `/Script/MassSmartObjects.SmartObjectMassBehaviorDefinition`, states the loadable set is optional-plugin-dependent, and documents the `availableBehaviorTypes`-echo discovery move (throwaway value → read the error); (b) `activityTags` must be registered via `gameplay_tags.add` or the call is rejected `[INVALID_PARAMS]` + `droppedTags`, authoring order `gameplay_tags.add` → `configure_slot_behavior`. A git-tracked regression test naming this ticket id verbatim already guards it: `Source/PinWright/Private/Tests/Infra/TestSmartObjectSlotBehaviorDocs.cpp` (`wiki_handler.MethodPage.SmartObjectSlotBehaviorDiscoverability`) renders `ai.configure_slot_behavior` through the live `WikiHandler::RenderPage` path and asserts every overlay marker (USmartObjectBehaviorDefinition, the default /Script path, optional/plugin, availableBehaviorTypes, gameplay_tags.add, droppedTags). Both files were committed together in `75b6be8`; `git diff --stat HEAD -- Docs/wiki-src/ai.md` shows no working-tree delta. The runtime cited in the ticket is already correct (`availableBehaviorTypes` echo + `droppedTags` rejection in `AIHandler.cpp`, the IN-REVIEW `B-configure-slot-behavior-ignores-behavior-and-tags` fix). The ticket's premise ("ripgrep returns no configure_slot_behavior section") is now stale/false. Flipped OPEN → IN-REVIEW and released the claim for a tester to verify-green. (Sibling docs tickets `E-state-tree-onevent-tag-registration-prereq` and `E-gas-set-effect-tags-drops-unregistered` target different overlays — state_tree.md / gas.md — and remain genuinely open; this ai.md edit does not satisfy them.)
- `#1-initial-audit` `OPEN` reporter — Process/struggle audit of the coffee-station smart-object task (focus `ai.configure_slot_behavior`, namespace `ai`, outcome clean, 15 calls). Trial-and-error retry: `configure_slot_behavior {behaviorType:"SmartObjectGameplayBehaviorDefinition"}` rejected `[INVALID_PARAMS] Unknown behaviorType ... see availableBehaviorTypes: .../SmartObjectTestBehaviorDefinition, /Script/MassSmartObjects.SmartObjectMassBehaviorDefinition`, then succeeded on the second call with the `/Script/MassSmartObjects...` value picked from the error. Friction note (verbatim): "behaviorType has no enumerated values in the wiki (only 'Type of behavior'), and activityTags silently-reject path required registering tags via gameplay_tags.add first — I learned both by reading the plugin's AIHandler.cpp C++ ... My first guessed behaviorType 'SmartObjectGameplayBehaviorDefinition' was rejected, but the error usefully returned availableBehaviorTypes so I picked the loaded /Script/MassSmartObjects.SmartObjectMassBehaviorDefinition on the retry." Verified the `ai` overlay (`docs/wiki-src/ai.md`, 31 lines) documents no `configure_slot_behavior` section, no enumerated `behaviorType` values, and no `gameplay_tags.add` prereq for `activityTags` (ripgrep over the file). Distinct PROCESS/docs angle from the two behavior tickets this task spawned: `B-configure-slot-behavior-ignores-behavior-and-tags` (IN-REVIEW — the runtime fix that made `behaviorType`/tags work + fail-loud, but documents nothing up front) and `E-asset-dump-instanced-uobject-class-elided` (the judge's filed_id — readback class elision). Same overlay-omits-the-precondition / no-enumerated-values failure mode as `E-state-tree-onevent-tag-registration-prereq` and `E-gas-set-effect-tags-drops-unregistered#3`. E-/docs: works once the values/tags are known; the gap is discoverability + one wrong-guess round-trip + a C++ read. Proposes documenting `behaviorType` accepted values (with the `availableBehaviorTypes`-echo discovery move) and the `activityTags` registration prereq on `docs/wiki-src/ai.md`.
