---
id: E-material-function-input-optional-default
title: "add_function_input has no way to mark an input optional / give a preview default — caller compiles fail with \"Missing function input\""
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [material, material-authoring, material-function, add-function-input]
---

# `add_function_input` always creates a REQUIRED input, with no optional/default knob

`material.authoring.add_function_input` creates every input as a **required**
`UMaterialExpressionFunctionInput` (no `PreviewValue`, `bUsePreviewValueAsDefault=false`).
A required function input that has no incoming wire **in the calling material**
makes that material fail to compile — the engine reports `Missing function input
<name>`. There is no parameter on `add_function_input` to mark an input optional
or to give it a preview/default value, so the only way to make the function safely
callable (without wiring every input at every call site) is an undocumented engine
detail: set `bUsePreviewValueAsDefault=true` **and** a `PreviewValue` on each
`FunctionInput` expression via raw `property.set`.

This is the ergonomic gap independent of (and downstream of)
`F-material-function-internal-authoring`: even once the function body IS authored
and the call node IS wired into the material, the **caller's** compile still fails
because the function's *other* inputs are required-with-no-default. The
fix-by-property.set is reachable only by diving into the engine header for
`UMaterialExpressionFunctionInput` to learn which two fields toggle optionality —
nothing in the `material.authoring` surface hints at it.

**Evidence (this task — authoring MF_TintBrighten = BaseColor * Brightness, then
M_TintedSurface calling it):** wiring was correct, but the first
`material.authoring.compile_material` returned `compiledWithErrors:true` with
`Missing function input BaseColor/Brightness`. Friction note: *"The function inputs
were created as REQUIRED, so the first compile_material failed … even though wiring
was correct; I had to set bUsePreviewValueAsDefault+PreviewValue on each
FunctionInput via property.set (engine header dive) to make them optional before the
caller compiled clean."* Call-log cost: **4 extra `property.set` calls** (2 per
input: `bUsePreviewValueAsDefault=true` + `PreviewValue=(…)`) plus a second
`compile_material` — all of which a single `optional`/`defaultValue` parameter on
`add_function_input` would eliminate. The agent also called this out explicitly as a
capability/discoverability gap: *"create_material_function/add_function_input give no
way to mark an input optional."*

**What it should do:** Add an optional knob to `add_function_input`, e.g.
`optional: true` (sets `bUsePreviewValueAsDefault=true`) and/or `defaultValue: <typed
literal>` (sets `PreviewValue`, and implies optional). With it, the function's inputs
are optional-with-preview at creation time and the calling material compiles clean
without per-input `property.set` plumbing. (Orthogonal to authoring the function's
internal nodes — that's `F-material-function-internal-authoring`.)

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS audit of task `material.authoring.create_material_function` (MF_TintBrighten / M_TintedSurface). Distinct from the judge-filed `F-material-function-internal-authoring` (internal node/wire authoring): this is the ergonomic gap that `add_function_input` always creates a REQUIRED input with no optional/default parameter, so the calling material's first `compile_material` failed `Missing function input BaseColor/Brightness` despite correct wiring. Workaround cost on the call-log: an engine-header dive plus 4 `property.set` calls (`bUsePreviewValueAsDefault=true` + `PreviewValue` per input) and a re-compile — all of which an `optional`/`defaultValue` param on `add_function_input` would remove. Not a dup: searched board for `add_function_input` / `FunctionInput` / `PreviewValue` / "optional input" — no existing ticket covers input optionality (the FunctionInput hits are about call-node pin-name decoration in `B-material-function-call-input-name-decoration`).
- `#2-add-function-input-optional-default` `IN-REVIEW` developer — Added two optional knobs to `material.authoring.add_function_input` so inputs can be created optional-with-preview instead of always REQUIRED. `optional:true` sets `bUsePreviewValueAsDefault=true`; `defaultValue` (numeric array `[x,y,z,w]` or a bare scalar) populates `PreviewValue` and implies optional. A present `defaultValue` is detected via the raw payload so "absent" vs "explicitly zero" is distinguishable. The handler now also echoes `optional` in its result. Files: `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Material/MaterialAuthoringHandler.cpp` (the `add_function_input` registration `RPC_PARAMS` + body just after editor X/Y, ~L1463-1556) and the discovery overlay `Plugins/EditorAutomationRpcGateway/Docs/wiki-src/material.authoring.md` (new `### material.authoring.add_function_input` H3 documenting the knobs and that `defaultValue` implies optional). Regression test: `FMaterialAuthoringAddFunctionInputOptionalDefaultTest` (`EditorAutomationRpcGateway.material.authoring.add_function_input.OptionalAndDefaultValue`) in `Private/Tests/Assets/TestMaterialHandlers.cpp` — creates a real on-disk `UMaterialFunction` via `create_material_function`, drives the production `add_function_input` handler three ways, then reads the engine fields back off the created `UMaterialExpressionFunctionInput`: default input asserts `bUsePreviewValueAsDefault==false` (the bug's baseline), `optional:true` asserts it flips true, and `defaultValue:[0.25,0.5,0.75]` asserts both the flag and `PreviewValue.X/Y/Z`. Fails if the optionality plumbing is reverted. Not compiled/run here (later phase).
