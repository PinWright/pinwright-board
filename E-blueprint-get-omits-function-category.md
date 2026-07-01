---
id: E-blueprint-get-omits-function-category
title: "`blueprint.get` / `blueprint.inspect` function entries omit `category` — the one `set_function_settings` field that can't be read back from the summary verbs"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [blueprint, blueprint-get, blueprint-inspect, set-function-settings, function-category, readback, discovery]
encounters: 2
lastSeen: 2026-06-25T07:14:35Z
---

# `blueprint.get` / `blueprint.inspect` surface every function flag except `category`

`blueprint.set_function_settings` accepts and applies five settable knobs on a
Blueprint function — `access` (public/protected/private), `isPure`, `isConst`,
`callInEditor`, and `category` — and reports each one it applied in its
`appliedSettings` array. Four of the five round-trip cleanly through the two
summary readback verbs: `blueprint.get` and `blueprint.inspect` both emit
`public`/`protected`/`private`/`pure`/`const`/`callInEditor` on every function
entry. **`category` is the one knob that is silently dropped** — neither readback
emits it, and the top-level `metadata` map is `{}`.

So the obvious "set the function's category, then confirm it via `blueprint.get`
/ `blueprint.inspect`" check is unsatisfiable. The caller either trusts the
write-time `appliedSettings` echo (which says `category` was applied even when
it was — see the related write-path bug below) or falls back to
`asset.dump` → `bpir.txt` / `blueprint.decompile_function`, which DO carry the
category as a `@meta(Category="…")` decorator. The data is **not lost** — it is
genuinely on the asset — so this is a readback / discovery gap, not data loss.

## The category provably persists — it's only the summary readback that omits it

Replayed live against `mcp__editor-automation__call` on
`/Game/Utilities/BP_InventoryMathLib` (an Actor BP with three functions whose
categories were set via `set_function_settings`):

- `blueprint.set_function_settings {functionName:"CalculateTotalWeight",
  isPublic:true, isPure:true, category:"Inventory|Weight"}` →
  `appliedSettings` includes `"category"` (write accepted).
- `blueprint.decompile_function {functionName:"CalculateTotalWeight"}` →
  `"bpir":"@meta(Category=\"Inventory|Weight\")\n@flags(Public, Pure)\nentry function CalculateTotalWeight(int Items, float TotalWeight) …"`
  — the category is provably on the asset.
- `blueprint.get {path:"/Game/Utilities/BP_InventoryMathLib"}` → the
  `CalculateTotalWeight` function entry is
  `{"name":"CalculateTotalWeight","public":true,"protected":false,"private":false,"pure":true,"const":false,"callInEditor":false,"inputs":[…]}`
  — **no `category` field** — and the response's top-level `"metadata":{}` is
  empty.
- `blueprint.inspect {assetPath:"/Game/Utilities/BP_InventoryMathLib"}` → the
  same function entry, same flag set, **same omission of `category`**.

All three functions show the pattern: `IsOverEncumbered` round-trips
`pure`+`const` but not its `Inventory|Checks` category; `ResetInventory`
round-trips `callInEditor` but not its `Inventory|Actions` category.

## Root cause (one-line readback omission)

`CollectBlueprintFunctions` (`BlueprintHandlerUtils.cpp:947-993`) — the shared
function-entry builder behind both `blueprint.get` and `blueprint.inspect` —
reads `EntryNode->MetaData.bCallInEditor` (`:975`) and the entry's function
flags (`:969-974`), then emits `public`/`protected`/`private`/`pure`/`const`/
`callInEditor` (`:982-987`) plus `inputs`/`outputs`. It never reads
`EntryNode->MetaData.Category`, so the category is dropped at the readback site.
The data is right there: `set_function_settings` writes the category into that
same `EntryNode->MetaData` struct (the function-category path at
`BlueprintFunctionHandler.cpp:~939`, the sibling of the `bCallInEditor` write at
`:947`). `blueprint.inspect`'s text formatter compounds it — its function-flag
line builder (`BlueprintInspectFormatter.cpp:187-193`) joins
`pure`/`const`/`callInEditor` into the `[...]` flag suffix and likewise never
considers category.

## What it should do

Emit `category` on each function entry in `CollectBlueprintFunctions` straight
from `EntryNode->MetaData.Category` (one `Fn->SetStringField(TEXT("category"),
EntryNode->MetaData.Category)` next to the `callInEditor` set at `:987`, emitted
only when non-empty to keep entries lean). That makes the function-flag readback
symmetric with the `set_function_settings` write surface — every knob the writer
sets becomes confirmable from the summary verb that already reports the other
four. Optionally fold it into `blueprint.inspect`'s `[...]` flag suffix
(`BlueprintInspectFormatter.cpp:191`) as `category=Inventory|Weight`.

**Workaround:** confirm a function's category via
`blueprint.decompile_function` (the `@meta(Category="…")` decorator) or an
`asset.dump` `bpir.txt` read — both already carry it. Treat `blueprint.get` /
`blueprint.inspect` as authoritative for access/pure/const/callInEditor but
blind to category.

## Relationship to existing tickets

This is the **function-flags analogue** of the established `blueprint.get` /
`blueprint.inspect` readback-omission family, but a distinct field with a
distinct fix surface:

- `E-blueprint-get-omits-components-readback-guidance` (IN-REVIEW) — omits
  `components` (routes to `blueprint.scs.get`).
- `E-blueprint-get-defaults-always-empty` (IN-REVIEW) — emits empty `defaults`
  (populated from the CDO).
- `E-variable-readback-instanceeditable-always-true` — a *variable* flag, not a
  function one.

None of those touch function `category`. Distinct from the **write-path** bug
`B-variable-category-ftext-localization-error` (IN-REVIEW): that fixed
`set_function_settings` / `add_variable` *rejecting* a plain-string `category`
with `INVALID_TEXT_LOCALIZATION_IDENTITY` — i.e. the category now *writes*
(decompile confirms it lands). This ticket is purely that the category, once
written, can't be **read back** from `blueprint.get` / `blueprint.inspect`.
Distinct from the DONE `B-bpir-function-metadata-stripped`, which restored
`@meta`/`@flags` decorators on the **decompile/BPIR** path — that fix is exactly
why the category IS visible via decompile here; this ticket asks the JSON
summary verbs to carry it too, so a caller doesn't have to drop to a dump/
decompile fallback for one field.

Severity Medium: a readback that omits a field and forces a fallback
(per the board severity rubric — soft blocker with a documented workaround). It
is conspicuous because the function-flag set looks complete (five of five other
`set_function_settings` knobs round-trip), so a caller reasonably reads the
missing `category` as "no category set" rather than "not reported."

## History
- `#1-initial-repro` `OPEN` reporter — Seed `blueprint.set_function_settings`. Struggle-audit of an end-to-end "Inventory Math" utility BP build (`/Game/Utilities/BP_InventoryMathLib`: 3 functions, each given access/pure/const/callInEditor + a `category` via `set_function_settings`, compiled, saved, read back). Every `set_function_settings` call reported `category` in `appliedSettings`, and the category provably persisted — `blueprint.decompile_function {CalculateTotalWeight}` returns `@meta(Category="Inventory|Weight")\n@flags(Public, Pure)…`. But the readback step found `blueprint.get` and `blueprint.inspect` both emit the function entry with `public/protected/private/pure/const/callInEditor` and **no `category` field** (top-level `metadata:{}` empty), so the task's "confirm each function landed with the category I asked for" check was unsatisfiable from the summary verbs; the attempt fell back to `asset.dump`/`bpir.txt`. Replayed live against `mcp__editor-automation__call` (get + inspect + decompile_function on the surviving asset) — confirmed. Root cause: `CollectBlueprintFunctions` (`BlueprintHandlerUtils.cpp:947-993`) reads `EntryNode->MetaData.bCallInEditor` and the function flags but never `EntryNode->MetaData.Category`; `BlueprintInspectFormatter.cpp:187-193` likewise omits category from the flag suffix. Fix: emit `category` from `EntryNode->MetaData.Category` next to the `callInEditor` field. Distinct from the write-path bug `B-variable-category-ftext-localization-error` (category now writes) and from the components/defaults readback-omission siblings (different fields).
- `#2-process-heavy-fallback-routing` `OPEN` reporter — PROCESS angle (struggle-audit, same `BP_InventoryMathLib` task, 15 calls, outcome clean). Beyond the readback omission itself, the friction had a *routing* cost: when `blueprint.get`/`blueprint.inspect` returned no category, the agent reached for **`asset.dump`** — the heaviest available confirm verb (full per-asset dump to disk, then a `bpir.txt` read) — to recover one string per function, rather than the cheap targeted `blueprint.decompile_function {functionName}` (which this very ticket shows returns `@meta(Category="…")` in a single in-process call). So the call-log shows BOTH summary readbacks AND a disk dump (3 confirm verbs for one field) where one decompile would have sufficed. Cause is discovery: there is **no `### blueprint.set_function_settings` overlay** in `docs/wiki-src/blueprint.md` (grep finds zero `set_function_settings`/`callInEditor`/`isPure` mentions), and the `### blueprint.get` (line 193) / `### blueprint.inspect` (line 201) overlays describe themselves as the function-surface readbacks without noting that `category` is omitted there and confirmable via `blueprint.decompile_function`. Secondary docs fix (rides the code fix): add a `### blueprint.set_function_settings` overlay listing the five knobs and stating "to confirm `category` before the get/inspect fix lands, use `blueprint.decompile_function` (`@meta(Category=…)`) — NOT `asset.dump`," and add a one-line "(category omitted — see `decompile_function`)" caveat to the `### blueprint.get`/`### blueprint.inspect` function-surface descriptions. Overlay page to improve = `docs/wiki-src/blueprint.md`. Reinforces this ticket's "forces a fallback" Medium framing: the fallback that's actually reached is the worst one.
- `#3-readback-fix` `IN-REVIEW` developer — Fixed the readback omission at root cause. `CollectBlueprintFunctions` (`Source/PinWright/Private/Handlers/Blueprint/BlueprintHandlerUtils.cpp`) — the shared function-entry builder behind both `blueprint.get` (`BuildBlueprintSnapshot`) and `blueprint.inspect` — now captures `EntryNode->MetaData.Category.ToString()` alongside the existing `bCallInEditor`/flags read and emits a `category` string field on each function entry, guarded emit-when-non-empty exactly like the variable-entry path in `BuildVariableJson` (`if (!CategoryStr.IsEmpty()) SetStringField("category", …)`). Also folded category into `blueprint.inspect`'s text flag suffix (`BlueprintInspectFormatter.cpp`) as `category=<value>` so the `[...]` rendering is symmetric too. The category source is the same `EntryNode->MetaData` struct the writer fills via `FBlueprintEditorUtils::SetBlueprintFunctionOrMacroCategory` (`BlueprintFunctionHandler.cpp:952-960`), so no data path changes. Regression test `PinWright.blueprint.get.FunctionCategoryReadback` (`FBlueprintFunctionCategoryReadbackTest` in `Source/PinWright/Private/Tests/Blueprint/TestBlueprintHandlers.cpp`): seeds a function, sets its category via `blueprint.set_function_settings`, then asserts BOTH `blueprint.get` and `blueprint.inspect` return the matching function entry carrying `category == "Inventory|Weight"`. Reverting the `CollectBlueprintFunctions` emit drops the field and fails both halves. Did not compile/run (later phase). Note for verifier: history #2's secondary docs change (a `### blueprint.set_function_settings` overlay in `docs/wiki-src/blueprint.md`) was NOT made here — this fix removes the underlying readback gap, so get/inspect now carry category natively; a docs overlay can still be added opportunistically but is no longer load-bearing.
- `#3b-emit-function-category` `IN-REVIEW` developer (fuzz3, merged-duplicate) — Independent fuzz3 host concurrently produced the same root-cause fix while `#3-readback-fix` was being published from another host; reconciled at plugin-sync. The two code edits are functionally identical (`CollectBlueprintFunctions` now emits `category` from `EntryNode->MetaData.Category`, guarded non-empty, plus the `BlueprintInspectFormatter.cpp` `category=<value>` flag), so the shared source landed once (took the `#3-readback-fix` version). fuzz3's separate regression test was kept ALONGSIDE the other: `FBlueprintReadbackEmitsFunctionCategoryTest` (`PinWright.blueprint.get.FunctionCategoryRoundTrips`) — seeds via `blueprint.add_function`, sets `category:"Inventory|Weight"` via the real handler, then drives `blueprint.get` AND `blueprint.inspect` through a shared `AssertCategoryRoundTrips` lambda asserting each function entry carries `category=="Inventory|Weight"`. Both tests now coexist in `TestBlueprintHandlers.cpp` (distinct names + automation paths + transient-BP names). The `#2` docs-overlay rider is likewise not included. No work lost.
