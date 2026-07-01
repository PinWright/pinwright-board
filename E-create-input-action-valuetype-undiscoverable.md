---
id: E-create-input-action-valuetype-undiscoverable
title: "input.create_input_action wiki is silent that it always creates a Boolean IA and that valueType is set via property.set, not a create param — lures a UNKNOWN_PARAMS probe"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [input, enhanced-input, wiki, docs, create-input-action, value-type, discoverability]
---

# `input.create_input_action` never tells you it always makes a Boolean IA (or how to author Axis2D)

The `input` namespace prelude says it "authors `UInputAction` (IA) assets [that]
describe an abstract action (Jump, Fire, Look)", and `input.get_input_info`
prominently returns each IA's `valueType` — so the value type reads as a
first-class IA property the caller is expected to set. But `create_input_action`
takes only `name` + `path`; it always creates a **Boolean** IA
(`AssetTools.CreateAsset(... UInputAction::StaticClass() ...)`, no value type
configured, so `ValueType` stays its default `EInputActionValueType::Boolean`).
`valueType` is **not** a create parameter (passing it returns `UNKNOWN_PARAMS`).
The wiki page for `create_input_action` says nothing about either fact.

There **is** an MCP way to author an Axis1D/Axis2D/Axis3D IA — it's just not on
`create_input_action` and not discoverable from this page. `UInputAction::ValueType`
is a reflected `UPROPERTY(EditAnywhere, BlueprintReadOnly) EInputActionValueType
ValueType` (engine `InputAction.h`), and the generic reflection setter
`property.set` reaches it: `property.set({objectPath:
"/Game/Input/IA_Move.IA_Move", propertyName:"ValueType", value:"Axis2D"})`. The
enum-property apply path accepts the string enum name (`PropertyImport.cpp`
`FEnumProperty`/`GetValueByNameString`), and `property.set` calls `Modify()` /
`PostEditChange()` — which fires `UInputAction::PostEditChangeProperty`. This is
the board's established escape-hatch for "no typed setter for an asset field"
(same shape as the `property.set SequenceLength` / `property.set FirstNode`
precedents). The page just never points there.

Two compounding consequences fall out of the silence:

1. **A guaranteed trial-and-error probe.** A caller authoring a 2D-vector action
   (WASD/stick movement, mouse-look) naturally passes `valueType` (or
   `valueType:"Axis2D"`) to `create_input_action`, because the namespace frames
   value type as an IA property and nothing on the page rules it out. That call
   fails with `[UNKNOWN_PARAMS] ... Valid parameters: [name, path]`, then the
   caller retries with just name+path. The `UNKNOWN_PARAMS` error itself is
   correct and clear (per the DONE `B-unknown-params-error-suggests-deleted-
   question-mark-suffix`) — the friction is upstream: the page should have said
   "valueType is not a create parameter; set it afterward with `property.set`"
   before the probe.

2. **The Axis2D round-trip looks unreachable until you find `property.set`.**
   Because `create_input_action` always returns a Boolean IA and exposes no
   value-type knob, a caller targeting an Axis2D `IA_Move`/`IA_Look` has no
   in-page signal that the second step (`property.set` on `ValueType`) exists.
   Each `create_input_action` returns ok, every `add_mapping` returns ok, and
   the IA quietly stays Boolean — `get_input_info` reports `valueType "0"` at the
   final verify. An up-front note ("IAs are created Boolean; to author
   Axis1D/Axis2D/Axis3D set `ValueType` via `property.set`") converts a
   late-surfacing surprise into an up-front two-step recipe.

## Why it's process friction (clean per-call outcomes)

Every individual call behaved correctly; the friction is purely discoverability.
The reporter's original friction note claimed "there is NO way via this MCP to
set an Input Action's valueType" — that claim is **wrong**: `property.set` reaches
the reflected `ValueType` enum (see above). The genuine gap is that
`create_input_action`'s page never states the Boolean default and never points at
the `property.set` round-trip. The call log shows the probe shape:
`create_input_action IA_Move` ok → `get_input_info IA_Move (probe default
valueType)` ok → `create_input_action IA_Look + valueType:Axis2D (probe)`
**is_error** `[UNKNOWN_PARAMS]` → `create_input_action IA_Look` (name+path) ok.
The agent even front-loaded a `get_input_info` probe to learn the default value
type, a discovery round-trip the doc note erases.

This is the **discoverability/docs** angle. It is distinct from the sibling input
tickets: `E-add-mapping-example-wrong-param` (add_mapping example key),
`E-get-input-info-consume-field-name-mismatch` (get_input_info output field name),
and `B-input-trigger-modifier-stub-silent-success` (trigger/modifier stubs). None
of those cover `create_input_action`'s value-type silence.

## What it should do

Docs is the cheap immediate win (it converts a guaranteed probe + a late-surfacing
Axis2D surprise into an up-front two-step recipe). In `docs/wiki-src/input.md`,
add a `### input.create_input_action` overlay section (currently absent — the
page has no per-method section for it) stating:

- `create_input_action` takes only `name` + `path`; it always creates a
  **Boolean** (`EInputActionValueType::Boolean`) IA. `valueType` is **not** a
  create parameter (passing it returns `UNKNOWN_PARAMS`).
- To author an **Axis1D / Axis2D / Axis3D** action, set the IA's `ValueType`
  after creation with **`property.set`** — `property.set({objectPath:
  "/Game/Input/IA_Move.IA_Move", propertyName:"ValueType", value:"Axis2D"})`.
  `ValueType` is a reflected `UPROPERTY`, so the generic reflection setter reaches
  it; the value is the enum name (`Boolean`/`Axis1D`/`Axis2D`/`Axis3D`).
- The typed IA setters `set_input_trigger`/`set_input_modifier` are
  `NOT_IMPLEMENTED` (per `B-input-trigger-modifier-stub-silent-success`) and do
  not touch value type — `property.set` is the route.

**Fix:** Add the `### input.create_input_action` H3 overlay section above to
`docs/wiki-src/input.md`. Do **not** repeat the false "MCP cannot set value type"
claim — `property.set` on `ValueType` is the working round-trip. Keep the note to
the Boolean-default fact + the `property.set` recipe + the trigger/modifier
clarification. (A dedicated `valueType` param on `create_input_action` or an
`input.set_input_value_type` verb remains a possible downstream capability, but is
out of scope here.)

Filed E-/`docs`. Wiki page to improve: `docs/wiki-src/input.md`.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean-per-call Enhanced-Input setup task (focus null, namespace input, story: create IA_Move/IA_Look (Axis2D) + IA_Jump (Boolean) + IMC_Default, bind 9 keys, enable @0, then verify value types/consume/mappingCount). Outcome agent_fail. 23 calls, all ok except one is_error: `create_input_action IA_Look + valueType:Axis2D (probe)` → `[UNKNOWN_PARAMS] Unknown parameter(s) for 'input.create_input_action': [valueType]. Valid parameters: [name, path].`, then retried name+path. Friction note: no MCP way to set an IA's valueType, so IA_Move/IA_Look stay Boolean instead of Axis2D (final get_input_info reports valueType "0"). Two process costs: (1) a guaranteed trial-and-error probe (plus a pre-emptive get_input_info "probe default valueType" round-trip) because the namespace frames value type as an IA property and the create page doesn't rule it out; (2) the unmeetable Axis2D requirement surfaces only at the final verify — every create/add_mapping returns ok, hiding the gap until the end. The UNKNOWN_PARAMS error itself is clear (B-unknown-params-...-question-mark-suffix DONE); the gap is upstream docs.
- `#2-reword` `OPEN` developer — Reworded: the reporter's central claim "no MCP surface to change an IA's valueType / Axis2D can't be authored" is FALSE in current source. `UInputAction::ValueType` is a reflected `UPROPERTY(EditAnywhere, BlueprintReadOnly) EInputActionValueType` (engine InputAction.h:115-116), and the generic reflection setter `property.set` (UtilityPropertyHandler.cpp:823) reaches it via the enum apply path (PropertyImport.cpp:528 `FEnumProperty`/`GetValueByNameString`) — `property.set({objectPath:"/Game/Input/IA_Move.IA_Move", propertyName:"ValueType", value:"Axis2D"})` works, the same escape-hatch pattern as the `property.set SequenceLength`/`FirstNode` board precedents. Retitled, rewrote body + Fix to drop the "MCP cannot round-trip" wording and instead document (a) `create_input_action` always makes a Boolean IA / `valueType` is not a create param, and (b) the `property.set ValueType` recipe to author Axis1D/2D/3D. Real discoverability defect confirmed: `Docs/wiki-src/input.md` has no `### input.create_input_action` section and the Gotchas block omits value type. Scope unchanged (docs-only).
- `#3-implement` `IN-REVIEW` developer — Added a `### input.create_input_action` H3 overlay section to `Docs/wiki-src/input.md` (above `### input.add_mapping`): states the verb takes only name+path and always creates a Boolean IA (`valueType` is not a create param → UNKNOWN_PARAMS), then gives the two-step Axis1D/2D/3D recipe (`create_input_action` then `property.set({objectPath, propertyName:"ValueType", value:"Axis2D"})`) with a code example and the trigger/modifier-stub clarification. Regression test `FWikiHandlerCreateInputActionDocumentsValueType` in `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` renders `input.create_input_action` via `WikiHandler::RenderPage` and asserts the page is not Not-Found, contains the overlay-exclusive markers `Boolean` / `property.set` / `ValueType` / `Axis2D`, and does NOT contain the false "cannot complete that round-trip" claim — it fails if the H3 overlay is reverted or regresses to the original mis-statement. Files: `Docs/wiki-src/input.md`, `Source/.../Private/Tests/Infra/TestWikiHandler.cpp`. Not compiled/tested here (later phase).
