---
id: F-blueprint-implement-interface
title: "No RPC to implement a Blueprint Interface on a Blueprint"
status: DONE
severity: High
category: feature
tags: [blueprint, interface, mutator]
---

# No RPC to implement a Blueprint Interface on a Blueprint

The gateway can READ implemented interfaces via `blueprint.inspect`
(see `BlueprintInspectHandler.cpp` iterating `BP->ImplementedInterfaces`),
but no mutator pushes into `ImplementedInterfaces`. The only related write
is `interaction.create_interactable_interface`, which is a demo-niche
helper that creates a fresh interface asset rather than attaching an
existing interface to a target blueprint.

Without this RPC, an agent cannot complete a common workflow: "make
`BP_Foo` implement `BPI_Damageable`". The only fallback today is manual
work in the editor or hand-editing `.uasset` data, both of which defeat
the point of an automation gateway.

**Proposal:** Add two mutating RPCs under the `blueprint.` namespace.

- `blueprint.add_interface(blueprintPath, interfaceClass)` — append the
  given interface class to `ImplementedInterfaces` if not already present;
  regenerate the skeleton class, mark the blueprint structurally modified,
  and recompile by default (with an `autoCompile: bool = true` opt-out).
  Idempotent: re-adding an already-implemented interface is a no-op
  success rather than an error.
- `blueprint.remove_interface(blueprintPath, interfaceClass)` — symmetric
  removal. Idempotent on a missing interface.

`interfaceClass` accepts both short names (`BPI_Damageable`) and full
object paths (`/Game/Path/BPI_Damageable.BPI_Damageable_C`), matching
existing class-resolution conventions in other handlers.

Optionally: when adding, seed empty function graphs for each pure-virtual
`UFUNCTION` on the interface — mirroring the editor's "Implement Function"
behavior — and return the newly-created graph names in the response so a
follow-up `blueprint.compile_bpir` can target them.

## History
- `#1-no-mutator-for-implementedinterfaces` `OPEN` reporter — Verified gateway has read-only access to `ImplementedInterfaces` via `blueprint.inspect`. No write path exists in `Handlers/Blueprint/`. Grep across `docs/rpc-method-reference.generated.md` confirms zero `add_interface` / `implement_interface` RPCs on the `blueprint` namespace; only `audio.authoring.add_metasound_interface` and `interaction.create_interactable_interface` exist, neither of which addresses Blueprint Interface implementation.
- `#2-add-blueprint-interface-mutators` `IN-REVIEW` developer — Added blueprint.add_interface and blueprint.remove_interface using FBlueprintEditorUtils interface helpers, with idempotent add/remove behavior and regression coverage in TestBlueprintInterfaceHandler.cpp.
- `#3-verify-interface-mutators` `DONE` tester — Verified: created `/Game/__EA_GatewayTests/BPI_McpVerifyTemp_FBlueprintImplementInterface`, then ran `blueprint.add_interface` twice and `blueprint.remove_interface` twice against `/AsyncTickPhysics/Blueprints/ATP_TestPawn`; observed add `changed:true compiled:true`, re-add `changed:false`, remove `changed:true compiled:true`, second remove `changed:false`, then deleted the temporary interface.
