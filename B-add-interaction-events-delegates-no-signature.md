---
id: B-add-interaction-events-delegates-no-signature
title: "interaction.add_interaction_events creates OnInteraction* multicast delegates with no SignatureFunction ('No SignatureFunction in MulticastDelegateProperty' compile warnings)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [interaction, add-interaction-events, delegate-no-signature, multicast-delegate, compile-warning, blueprint]
encounters: 1
lastSeen: 2026-07-02T09:14:38.1349323+03:00
---

# `interaction.add_interaction_events` creates signature-less multicast delegates

`interaction.add_interaction_events` reports success — it echoes
`{"eventsAdded":["OnInteractionStart","OnInteractionEnd","OnInteractableFound",
"OnInteractableLost"],"eventCount":4}` — but the four event dispatchers it adds
are created as `FMulticastDelegateProperty` with **no bound SignatureFunction**.
On the next compile (here `blueprint.add_interface`) the engine warns for each:
`No SignatureFunction in MulticastDelegateProperty 'OnInteractionStart'` (and the
other three). The Blueprint compiles `UpToDateWithWarnings`, and the dispatchers
— the whole point of the task's "wire it up with interaction events so my
gameplay code can hook into open/close" — have no signature, so a consumer
cannot reliably bind against the intended parameter set and the asset never
compiles clean.

A correct dispatcher needs a delegate signature graph named `<Name>` whose
generated `<Name>__DelegateSignature` UFunction backs the property (the exact
recipe `F-blueprint-add-dispatcher` implemented for `blueprint.add_dispatcher`).
`add_interaction_events` appears to add the multicast-delegate variables without
creating those signature graphs.

## Evidence

Struggle-audit of the `BP_TreasureChest` interaction build (namespace
interaction, 29 RPCs). `interaction.add_interaction_events` returned
`{"eventsAdded":["OnInteractionStart","OnInteractionEnd","OnInteractableFound",
"OnInteractableLost"],"eventCount":4}` (success). The following
`blueprint.add_interface` compile surfaced `No SignatureFunction in
MulticastDelegateProperty '<name>'` for all four dispatchers
(OnInteractionStart / OnInteractionEnd / OnInteractableFound / OnInteractableLost).

## Fix (proposed)

Have `add_interaction_events` create each dispatcher with a real delegate
signature graph + generated `__DelegateSignature` UFunction (reuse the
`blueprint.add_dispatcher` machinery from `F-blueprint-add-dispatcher`, DONE),
so the properties carry a SignatureFunction and the Blueprint compiles without
the `No SignatureFunction` warnings.

severity rationale: impact=dispatchers created signature-less (degraded/partly-unusable event surface, undermines the verb's stated purpose) with a visible compile warning (not silent) and the asset still compiles (soft degradation) × reach=interaction namespace, not every-session (no bump) -> Medium.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of `BP_TreasureChest`. `add_interaction_events` returned `eventCount:4` (OnInteractionStart/End, OnInteractableFound/Lost); the next `blueprint.add_interface` compile warned `No SignatureFunction in MulticastDelegateProperty '<name>'` for all four — the dispatchers are created without a signature UFunction, so the BP compiles `UpToDateWithWarnings` and gameplay code can't bind the intended signature. Not a dup of `F-blueprint-add-dispatcher` (DONE — that ADDED the general `blueprint.add_dispatcher` verb; this is `add_interaction_events` emitting malformed dispatchers) nor `B-bpir-delegate-signature-lost` (DONE — BPIR type-grammar delegate signatures). Proposed: build each dispatcher via the `add_dispatcher` signature-graph recipe so it carries a valid SignatureFunction.
- `#2-signature-graph-fix` `IN-REVIEW` developer — Extracted `blueprint.add_dispatcher`'s inline signature-graph recipe into a shared `BlueprintHandlerUtils::AddDispatcherWithSignatureGraph(Blueprint, Name, Params)` helper (adds the PC_MCDelegate member variable, creates+configures the `<Name>` signature graph, authors params as output pins, registers it in `DelegateSignatureGraphs`, rolls back on failure). `interaction.add_interaction_events` now creates each of the four `OnInteraction*` events through it (parameterless), so the `FMulticastDelegateProperty` compiles with a real `<Name>__DelegateSignature` UFunction instead of a null SignatureFunction — the `No SignatureFunction in MulticastDelegateProperty` warnings are gone and consumers can bind the events. `blueprint.add_dispatcher` was refactored to call the same helper (identical error codes/messages; guarded by its existing `PinWright.blueprint.add_dispatcher.CreatesSignatureGraphAndProperty` test). `eventCount` now reports the actual number added. Files: `Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.h`, `.../BlueprintHandlerUtils.cpp`, `.../BlueprintDispatcherHandler.cpp`, `.../Interaction/InteractionHandler.cpp`. Regression test: `PinWright.interaction.add_interaction_events.DispatchersHaveSignatureFunction` (drives the real handler on a fresh in-code Actor BP, compiles it, asserts each event's compiled `FMulticastDelegateProperty` has a non-null SignatureFunction plus a matching `__DelegateSignature` UFunction; fails if reverted to the signature-less `AddMemberVariable`).
