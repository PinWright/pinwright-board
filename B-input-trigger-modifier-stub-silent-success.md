---
id: B-input-trigger-modifier-stub-silent-success
title: "input.set_input_trigger / set_input_modifier are no-op stubs that return triggerSet/modifierSet: true"
status: IN-REVIEW
severity: High
category: bug
tags: [input, enhanced-input, stub, silent-failure]
---

# Stub input authoring handlers return success without mutating the asset

`input.set_input_trigger` and `input.set_input_modifier` in
`Handlers/Input/InputHandler.cpp` are implemented as success-returning
no-ops. They load the `UInputAction`, then build a result that echoes the
requested type back with a `triggerSet: true` / `modifierSet: true` flag —
without ever touching the action's `Triggers` / `Modifiers` array and
without saving.

**`input.set_input_trigger`** (InputHandler.cpp lines ~189–214):
```cpp
UInputAction* InAction = Cast<UInputAction>(UEditorAssetLibrary::LoadAsset(ActionPath));
// ... no mutation of InAction->Triggers, no save ...
Result->SetStringField(TEXT("actionPath"), ActionPath);
Result->SetStringField(TEXT("triggerType"), TriggerType);
Result->SetBoolField(TEXT("triggerSet"), true);
AddAssetVerification(Result, InAction);
Ctx.SendSuccess(Result);
```

**`input.set_input_modifier`** (lines ~216–241): identical shape, replies
`modifierSet: true`, never mutates `InAction->Modifiers`, never saves.

The wiki `input` page Gotchas section admits both are "stub-level today:
they record the request in the response but do not mutate the IA's
`Triggers` / `Modifiers` arrays." But the handlers still **return
success** with an affirmative `triggerSet`/`modifierSet: true` boolean, so
a caller wiring these into automation has no way to detect that nothing
happened. This is the same defect class already accepted and fixed as
`B-material-stub-handlers-silent-success` (DONE) for
`material.authoring.set_cast_shadows` / `set_material_parameter`.

**Why it matters:**

- A realistic Enhanced-Input authoring task ("put a Pressed trigger on
  IA_Jump so it reads as a button press") calls `set_input_trigger`, gets
  `triggerSet: true`, and concludes the trigger was authored. It was not.
  The IA opens in the editor with an empty Triggers list. No RPC-visible
  signal distinguishes "applied" from "ignored".
- `bConsumeInput` / `valueType` from `input.get_input_info` look correct,
  reinforcing the false belief that the IA is fully configured.

**Fix options (any of):**

1. **Implement them.** Append the requested trigger/modifier instance to
   `InAction->Triggers` / `InAction->Modifiers` (resolve the
   `UInputTrigger*`/`UInputModifier*` subclass from the type name, e.g.
   `UInputTriggerPressed`, `UInputModifierNegate`), then
   `SaveLoadedAssetThrottled(InAction, ...)`.
2. **Make them fail loud.** Replace `SendSuccess` with
   `SendError("NOT_IMPLEMENTED", "set_input_trigger is a stub; author
   triggers in the editor")` until the real implementation lands — the
   one-line minimum-safe change, matching the accepted resolution of
   B-material-stub-handlers-silent-success.

## Repro

1. `input.create_input_action({name: "IA_ReplayJump", path: "/Game/Input"})`
   → success.
2. `input.set_input_trigger({actionPath: "/Game/Input/IA_ReplayJump",
   triggerType: "Pressed"})`
   → returns `{"triggerType":"Pressed","triggerSet":true, ...}`.
3. `property.get({objectPath: "/Game/Input/IA_ReplayJump.IA_ReplayJump",
   propertyName: "Triggers"})`
   → returns `{"propertyName":"Triggers","value":[]}` — the Triggers array
   is empty; the "Pressed" trigger was never added.
4. Symmetric repro for `set_input_modifier` against `Modifiers`.

## History
- `#1-initial-repro` `OPEN` reporter — Realism-mode Enhanced-Input authoring task replayed live. `input.set_input_trigger` on a freshly created `IA_ReplayJump` with `triggerType=Pressed` returned `triggerSet: true`, but `property.get` on `Triggers` afterwards returned `[]` — confirmed silent success-with-no-effect (no mutation, no save). `input.set_input_modifier` shares the identical stub shape in InputHandler.cpp (lines ~216–241). Wiki Gotchas already flags both as stubs, but the handlers still send `SendSuccess` with `triggerSet`/`modifierSet: true`, so callers cannot detect the no-op. Same class as DONE ticket B-material-stub-handlers-silent-success; no existing input-namespace ticket (ripgrep + qmd both clean). Minimal honest fix: swap `SendSuccess` → `SendError("NOT_IMPLEMENTED", …)`.
- `#2-stubs-return-not-implemented` `IN-REVIEW` developer — Applied Option 2 (the DONE B-material-stub-handlers-silent-success pattern). In `Handlers/Input/InputHandler.cpp`, replaced both stub bodies of `input.set_input_trigger` and `input.set_input_modifier` with `SendError("NOT_IMPLEMENTED", …)` (message points callers at editor authoring) — no more `triggerSet`/`modifierSet: true` fake success, and the now-dead `LoadAsset`/result-build code was removed. Updated both REGISTER_RPC_HANDLER summaries to state they return NOT_IMPLEMENTED, and updated the `docs/wiki-src/input.md` Gotchas line accordingly. Regression tests added to `Private/Tests/Gameplay/TestInputHandlers.cpp` (`FInputSetInputTriggerReturnsNotImplementedTest`, `FInputSetInputModifierReturnsNotImplementedTest`) invoke the real handlers via `InvokeHandlerWithCapture` and assert `bSuccess == false` and `ErrorCode == "NOT_IMPLEMENTED"`; they would fail if the silent-success stub were restored. Not compiled/run here (later phase verifies).
