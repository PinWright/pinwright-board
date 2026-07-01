---
id: E-set-default-none-clear-conversion-failed
title: "`blueprint.set_default value:\"None\"` errors `CONVERSION_FAILED` when clearing an object reference; only `value:\"\"` works"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [blueprint, set-default, object-reference, clear-null, conversion-failed, confusing-error, docs]
---

# Clearing an object-reference CDO default is a misuse-then-correct trap

To null out an object-reference Blueprint default (e.g. remove the behavior tree from `DefaultBehaviorTree`), the obvious value to pass is `"None"` — that is UE's canonical null/empty-object sentinel and what the engine prints for a null `UObject*`. But `blueprint.set_default {propertyName, value: "None"}` fails with:

```
[CONVERSION_FAILED] Failed to load object at path: None
```

The handler (`ApplyJsonValueToProperty` in `Private/Utils/PropertyImport.cpp`) treats `"None"` as a literal asset path and tries to load it, instead of recognizing it as the null sentinel. The only value that actually clears the reference is the empty string `value: ""`. An agent that does the natural thing first burns a failed call and a corrective call, and the error text is actively misleading — it implies a missing asset named "None" rather than "I don't accept the null sentinel; pass an empty string."

## Per-property-type behavior (verified against source)

The exact `[CONVERSION_FAILED] Failed to load object at path: None` symptom is specific to **`FObjectProperty`** (`PropertyImport.cpp:622-655`): a non-empty string is treated as a load path, `LoadObject(nullptr, "None")` fails, and that error is emitted. The other object-reference kinds reject/mishandle `"None"` differently — so the "same error for all of them" framing in the original report was inaccurate:

- **`FObjectProperty`** (`:622-655`): `"None"` → `[CONVERSION_FAILED] Failed to load object at path: None`. Also does **not** accept a JSON `null` (line 653 → "Unsupported JSON type for object property"); only `""` clears.
- **`FClassProperty`** (`:559-619`): `"None"` → a parallel `Failed to resolve class at path: None` error. Accepts `""` and JSON `null` to clear.
- **`FSoftObjectProperty`** (`:658-692`) / **`FSoftClassProperty`** (`:695-751`): `"None"` does **not** error — it is silently stored as the literal path `FSoftObjectPath("None")` (silent-wrong, a worse failure mode than the loud error). Both accept `""` and JSON `null` to clear.

Internal precedent that `"None"` == null: the decompiler already short-circuits object-ref pins on `DefaultValue ∈ {"", "None"}` (`docs/wiki-src/blueprint.md:162`), so the import path is inconsistent with the rest of the codebase in not recognizing it.

## Friction evidence (this task, `ai.stop_behavior_tree` story)

Call-log pair:
- `blueprint.set_default DefaultBehaviorTree='None'` → `ok:false`, `[CONVERSION_FAILED] Failed to load object at path: None`
- `blueprint.set_default DefaultBehaviorTree=''` → `ok:true` (cleared)

Friction note: *"clear BT via empty string after value:'None' errored with CONVERSION_FAILED"*. The judge-filed `B-stop-behavior-tree-noop-wrong-var` mentions this only as a one-line workaround footnote; it is a distinct, reusable ergonomic trap on `blueprint.set_default` itself (any object/soft-object/class-reference clear hits it, not just the AI case). The same coercion path is shared by `widget.set` (`B-widget-set-partial-application`, DONE — which fixed partial-application rollback but not sentinel rejection).

## What it should do

1. `ApplyJsonValueToProperty` should accept the canonical null sentinels — the strings `"None"` and `"null"` (case-insensitive), the empty string `""`, and a JSON `null` — and clear `FObjectProperty` / `FSoftObjectProperty` / `FClassProperty` / `FSoftClassProperty` references to null without attempting a path load or storing a literal `"None"` path. Special-case the sentinel before the "load object at path" branch in each object-reference branch. This also fixes `FObjectProperty` rejecting JSON `null` and the soft-ref branches silently storing a literal `"None"`.
2. Docs angle: document the clear-to-null idiom on the `blueprint.set_default` page. Overlay to improve: `docs/wiki-src/blueprint.md` (under the existing `### blueprint.set_default` / CDO-defaults section, note that `value: ""` / `"None"` / JSON `null` all null an object reference).

**Workaround (pre-fix):** to clear an object reference via `blueprint.set_default`, pass `value: ""` (empty string), not `"None"`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `ai.stop_behavior_tree` task. PROCESS friction (misuse-then-correct + confusing error): `blueprint.set_default {propertyName:"DefaultBehaviorTree", value:"None"}` failed `[CONVERSION_FAILED] Failed to load object at path: None`; the corrective `value:""` succeeded. The error wrongly implies a missing asset named "None" rather than rejecting the null sentinel. Reusable across any object/soft-object/class-reference clear, not AI-specific. Distinct from `B-stop-behavior-tree-noop-wrong-var` (where this appears only as a workaround footnote). Docs overlay to improve: `docs/wiki-src/blueprint.md`.
- `#2-reword` `OPEN` developer — Reworded to scope the exact `Failed to load object at path: None` symptom to `FObjectProperty`, and to document the divergent per-kind behavior (`FClassProperty` gives a parallel resolve-class error; the soft-ref branches silently store a literal `"None"` path; `FObjectProperty` also wrongly rejects JSON `null`). Narrowed Fix to: recognize the `{""/"None"/"null"/JSON-null}` sentinel set in each object-reference branch of `ApplyJsonValueToProperty` and clear-to-null, plus the docs overlay. Verified all claims against `PropertyImport.cpp` line ranges cited above.
- `#3-fix` `IN-REVIEW` developer — Added a `IsNullObjectSentinel(value)` helper (trims, then case-insensitive match against `""`/`"None"`/`"null"`) in `Private/Utils/PropertyImport.cpp` and applied it as the clear-to-null guard in all four object-reference branches of `ApplyJsonValueToProperty`: `FObjectProperty` (no longer reaches `LoadObject("None")`; also now accepts JSON `null`), `FClassProperty`, `FSoftObjectProperty` and `FSoftClassProperty` (no longer store the literal `FSoftObjectPath("None")`). A non-sentinel string still loads as a path; an unloadable path still returns `Failed to load object at path: <path>`. Docs: added a "Clearing an object reference to null" subsection to the `### blueprint.set_default` overlay in `docs/wiki-src/blueprint.md` documenting that `""`/`"None"`/`"null"`/JSON-null all clear the reference. Files: `Private/Utils/PropertyImport.cpp`, `docs/wiki-src/blueprint.md`. Regression test: `EditorAutomationRpcGateway.utils.property_utils.ApplyJsonValueToProperty.NullSentinelClears` in `Private/Tests/Utility/TestPropertyUtils.cpp` — seeds `UStaticMeshComponent::StaticMesh` (FObjectProperty, all 5 sentinels incl. JSON null), `UChildActorComponent::ChildActorClass` (FClassProperty, `"None"`), and `UBoundsCopyComponent::BoundsSourceActor` (FSoftObjectProperty, `"None"`) non-null, then asserts each sentinel clears to null via the real `ApplyJsonValueToProperty`. Would fail if the guard were reverted (the object/class branches would error, the soft branch would retain the literal `"None"` path).
