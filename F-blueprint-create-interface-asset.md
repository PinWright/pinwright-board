---
id: F-blueprint-create-interface-asset
title: "`blueprint.create` cannot produce a Blueprint Interface asset"
status: DONE
severity: Medium
category: feature
tags: [blueprint, interface, creation]
---

# `blueprint.create` cannot produce a Blueprint Interface asset

`blueprint.create` always builds a regular `UBlueprint` with
`BPTYPE_Normal` — it hardcodes `UBlueprint::StaticClass()` and
`UBlueprintFactory` in `BlueprintCreationHandler.cpp` (both the
test-context branch around line 148 and the live branch around line 272).
The `blueprintType` parameter today is only a convenience hint to pick a
parent class (`'actor' | 'pawn' | 'character'`) when `parentClass` is
omitted; it does not affect the asset type.

There is no path through any handler to produce a Blueprint Interface
(`BPTYPE_Interface`, parent `UInterface`). `BlueprintInspectHandler.cpp:75`
proves the inspect side already understands `BPTYPE_Interface` and reports
`blueprintType:"Interface"`, so the read surface is complete — only the write
surface is missing.

This blocks a common workflow: defining an interface contract from
automation before any blueprint implements it. The companion ticket
`F-blueprint-implement-interface` covers attaching an existing interface
asset to a regular BP, but assumes the interface asset already exists.
Without this ticket's RPC there is no way to create that interface asset
in the first place except by hand in the editor.

**Acceptance:**

- `blueprint.create` accepts `blueprintType:"interface"` and produces a
  `UBlueprint` with `BlueprintType == BPTYPE_Interface` and parent
  `UInterface` (default when `parentClass` is omitted). Explicit
  `parentClass` must resolve to a `UInterface` subclass or the call
  errors with `INVALID_ARGUMENT`.
- Follow-up `blueprint.add_function` on an interface BP creates a
  signature-only declaration (no implementation graph), matching the
  editor's interface-function authoring behavior.
- `blueprint.inspect` on the created asset reports
  `blueprintType:"Interface"` (already wired via
  `BlueprintInspectHandler.cpp`).

**Fix sketch:** branch on `BlueprintTypeSpec.ToLower() == "interface"`
before factory construction; use `UBlueprintFactory` with
`BlueprintType = BPTYPE_Interface` (or the editor's interface-blueprint
factory if one is exposed) and default `ParentClass` to
`UInterface::StaticClass()`. Reject non-`UInterface` `parentClass` specs
in that branch. The two branches in `BlueprintCreationHandler.cpp`
(test-context and live) need parallel updates.

## History
- `#1-initial-spec` `OPEN` reporter — Verified `blueprint.create` hardcodes `UBlueprint::StaticClass()` + `UBlueprintFactory` at `BlueprintCreationHandler.cpp:148` (test path) and `:272` (live path); `blueprintType` is only a parent-class hint per the RPC param at `:103`. `BPTYPE_Interface` appears only on the read side (`BlueprintInspectHandler.cpp:75`). Existing board: `F-blueprint-implement-interface` is OPEN but covers attaching an existing interface to a BP — orthogonal to creating the interface asset itself. `F-agir-interface-function-decls` is DONE and AnimGraph-scoped.
- `#2-correct-inspect-field` `OPEN` developer — Analysis confirmed the write-surface gap but corrected the inspect acceptance field: blueprint.inspect emits blueprintType:"Interface"; do not add a duplicate inspect field.
- `#3-create-interface-assets` `IN-REVIEW` developer — Added blueprint.create support for blueprintType:"interface", validated interface parents, preserved interface add_function signature declarations, and covered create/inspect/invalid-parent behavior in TestBlueprintCreateInterface.cpp.
- `#4-verify-interface-create` `DONE` tester — Verified: `blueprint.create` created `/Game/App/UI/Test/BI_McpVerifyTemp_F_blueprint_create_interface_asset_20260515` with `blueprintType:"interface"` and `blueprint.inspect` reported `blueprintType:"Interface"`, `parentClass:"Interface"`, `nativeParentClass:"Interface"`; `blueprint.add_function` added `VerifyInterfaceCall` and follow-up inspect showed a single-node interface graph; invalid `parentClass:"Actor"` returned `INVALID_ARGUMENT`. Temp asset deleted via `asset.delete`.
