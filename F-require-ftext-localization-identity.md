---
id: F-require-ftext-localization-identity
title: "Require namespace and key when authoring persisted FText values"
status: DONE
severity: High
category: feature
tags: [ftext, localization, property-set, widget-set, bpir]
---

# Require namespace and key when authoring persisted FText values

MCP can now parse `NSLOCTEXT(...)`, `LOCTEXT(...)`, and `INVTEXT(...)` through `FTextStringHelper::CreateFromBuffer`, but it still accepts plain strings for persisted authored `FText` values. That silently stores culture-invariant text in widget assets, Blueprint defaults, actor/component properties, and text pin defaults. Backward compatibility with old MCP scripts is not a constraint; tests that expect culture-invariant persisted text should be updated.

The stricter policy should be: when a persisted `FText` value is being authored, the incoming value must provide a non-empty namespace and key, unless the existing stored value already has a non-empty namespace and key and the operation is only replacing the source string.

## Scope

Enforce first on persisted user-authored text:

- `ApplyJsonValueToProperty` scalar `FTextProperty` writes in `Source/PinWright/Private/Utils/PropertyImport.cpp:361`.
- `ApplyJsonValueToProperty` array-of-`FText` writes in the same file.
- RPCs routed through that helper, including `property.set`, `blueprint.set_default`, `widget.set`, `widget.import_xml`, actor/component property writes, and SCS property writes.
- Blueprint graph and BPIR `PC_Text` pin defaults, including `blueprint.graph.set_pin_default_value`, `blueprint.graph.set_pin_default_values`, `FCodePinResolver::SetPinDefaultValue`, and the BPIR `FormatText` path that currently writes `DefaultTextValue` directly.
- Persisted text settings that bypass generic property writes, including widget binding struct text fields and Blueprint variable/function category settings.

Internal/editor-only labels that are not authored game/UI text are out of this task's first scope: transactions, slow-task descriptions, runtime-only `ui.set_widget_text`, and hardcoded internal variable-category labels used by helper generators. This is a scope boundary, not a backward-compatibility exception.

## Expected behavior

Accepted:

```json
{ "Text": "NSLOCTEXT(\"W_Menu\", \"StartButton.Text\", \"Start\")" }
```

Accepted when overwriting an already-localized property:

```json
{ "Text": "Start race" }
```

The implementation should preserve the existing namespace/key and replace only the source string.

Rejected when the current value is empty or culture-invariant:

```json
{ "Text": "Start race" }
```

The error should name the property/pin and explain that persisted `FText` values require a namespace and key.

## Implementation notes

Use Unreal's own text helpers instead of a custom parser:

- Parse with `FTextStringHelper::CreateFromBuffer`.
- Inspect identity with `FTextInspector::GetNamespace` and `FTextInspector::GetKey`.
- Preserve existing identity with `FText::ChangeKey(Namespace, Key, FText::FromString(NewSource))` in editor builds.

Centralizing the policy in `PropertyUtils` will cover most persisted property writes. Text pins need separate guards because graph handlers and BPIR call `TrySetDefaultValue` or assign `DefaultTextValue` outside `ApplyJsonValueToProperty`.

## Related

- `F-widget-set-ftext-nsloctext-parse` fixed macro parsing and round-trip behavior. This task is stricter: reject or identity-preserve plain strings instead of silently storing culture-invariant authored text.

## History

- `#1-reported-policy-gap` `OPEN` reporter — Research found that persisted property writes route mostly through `ApplyJsonValueToProperty`, where plain strings still fall back to culture-invariant `FText`. Separate text paths exist for graph `PC_Text` pin defaults and BPIR `FormatText`. File a feature to require non-empty namespace/key for persisted authored text, with an exception for replacing the source string of an already-localized value.
- `#2-no-legacy-compat` `OPEN` reporter — Clarified product constraint: there are no old MCP scripts to preserve, only outdated tests. Do not keep culture-invariant persisted authored `FText` behavior for backward compatibility; update tests that encode the old behavior.
- `#3-implemented-identity-policy` `IN-REVIEW` implementer — Added shared persisted `FText` coercion in `PropertyUtils`, enforced it for scalar/array property writes, graph `PC_Text` pin defaults, BPIR pin defaults, BPIR `FormatText`, widget binding text fields, and Blueprint variable/function category settings. Updated focused property, graph, and BPIR tests plus the UMG design guide. Static-only verification was used; Unreal tests are pending tester run.
- `#4-verified-widget-ftext-policy` `DONE` tester — Verified: `widget.set` on `/Game/App/UI/Test/W_McpReviewTemp_20260429` `BetaText.Text` rejected a plain string with `INVALID_PROPERTY` and the required namespace/key message, then accepted `NSLOCTEXT("McpReview", "BetaText.Label", "Localized beta")`; `blueprint.compile saveAfterCompile:false` returned `compiled:true`, `errors:[]`, `warnings:[]`.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. `Utils/PropertyUtils.cpp` was split and deleted; this ticket's code went to **`PropertyImport.cpp`**, the odd one out of the four split files (the other eleven tickets in the earlier split sweep all landed in `PropertyExport.cpp`). Scalar `FTextProperty` handling is `Source/PinWright/Private/Utils/PropertyImport.cpp:361` (`if (FTextProperty* TP = CastField<FTextProperty>(Property))` inside `ApplyJsonValueToProperty`, which starts `:305`), calling `CoerceJsonValueToPersistedFText` at `:371`; the array-of-`FText` case is `:929` with the write at `:963-967`; and the enforcement helper `HasLocalizationIdentity` is `:19`, used by `CoerceStringToPersistedFText` at `:245`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
