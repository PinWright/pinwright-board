---
id: F-metasound-no-variables-or-validate
title: "MetaSound graph variables and compile/validate not exposed"
status: DONE
severity: Medium
category: feature
tags: [audio, metasound, authoring, variables, compile, validate, no-text-ir]
---

# MetaSound graph variables and compile/validate not exposed

Two related capability gaps in the `audio.authoring.*` MetaSound
surface, both stemming from the same constraint — MetaSound has no
text IR (no MSIR), so anything the imperative API doesn't expose is
unreachable.

**Gap 1: Graph variables.** MetaSound graphs support **variables**:
typed graph-local cells that can be set from one place and read from
many, used for stateful coordination (e.g. holding a phase accumulator
or a delay-line tap value). `MetaSoundDumpBuilder::BuildMetaSoundJson`
already emits a `variables` array on `describe_metasound` output, but
there is no RPC to add, remove, or set defaults on variables. Verified
— `audio.authoring.add_metasound_variable` returns Not found.

**Gap 2: Explicit compile / validate.** MetaSound graphs compile lazily
— graph errors surface only when the asset is opened in-editor or
referenced by a runtime player. `create_metasound`'s wiki page notes
this verbatim ("MetaSounds compile lazily — graph errors surface only
when the asset is opened or referenced"). There is no RPC to **force
a compile** or **return validation diagnostics**. An agent that wires
a graph wrong gets no signal back — `connect_metasound_nodes` returns
success on a structurally connectable but semantically invalid graph
(missing interface vertex, type mismatch on a Reference vs Value pin,
etc.), and the failure only manifests when something tries to play it.

Verified:
- `audio.authoring.compile_metasound` -> Not found
- `audio.authoring.validate_metasound` -> Not found

**Why both belong in one ticket.** Variables and compile/validate are
independent capability axes, but both are sub-critical compared to
the destructive-ops gap (`F-metasound-no-destructive-graph-ops`), the
interface-attach gap (`F-metasound-no-interface-attach`), and the
input/output mutation gap (`F-metasound-no-input-output-mutation`).
They share the same root cause (imperative-only surface missing
inverses and probes) and the same fix shape (thin
`FMetaSoundFrontendDocumentBuilder` wrappers plus an explicit invoke
of the MetaSound auto-update / compile path).

**Implementation surface:**

Variables — `FMetaSoundFrontendDocumentBuilder` exposes
`AddGraphVariable(FName, FName TypeName, FMetasoundFrontendLiteral
DefaultValue)` and `RemoveGraphVariable(FName)`. Mirror the existing
`add_metasound_input` handler.

Compile/validate — `Metasound::Engine::FMetaSoundAssetManager` and
`UMetaSoundBuilderSubsystem` provide validation/auto-update hooks
(`Builder.ValidateGraph()`, `MetaSoundAsset->RebuildReferencedAssetClasses()`,
`MetaSoundAsset->AutoUpdate(true)`). A `compile_metasound` handler
should run validation, return any diagnostics, and surface the same
"missing interface vertex" / "type mismatch on edge" warnings the
in-editor compiler shows.

Proposed RPCs:

```
audio.authoring.add_metasound_variable {
    assetPath, variableName, variableType,
    floatValue?|intValue?|boolValue?|stringValue?,
    save?
}
audio.authoring.remove_metasound_variable { assetPath, variableName, save? }
audio.authoring.set_metasound_variable_default { ...same literal payload as set_metasound_default... }

audio.authoring.validate_metasound { assetPath } -> {
    valid: bool,
    diagnostics: [{ severity, code, message, nodeId?, vertexName? }]
}
audio.authoring.compile_metasound { assetPath, save? } -> { ...validate output + did the graph rebuild... }
```

## History
- `#1-no-variables-or-validate` `OPEN` reporter — Verified: `audio.authoring.add_metasound_variable` / `compile_metasound` / `validate_metasound` all return Not found. `describe_metasound` already emits a `variables` array via `MetaSoundDumpBuilder`, so the read side knows about variables but the write side can't author them. Compile is lazy — `connect_metasound_nodes` reports success on semantically-invalid graphs (missing required interface vertex, type mismatch). MetaSound has no text-IR escape hatch. Proposes thin wrappers around `FMetaSoundFrontendDocumentBuilder::AddGraphVariable` / `RemoveGraphVariable` plus a `validate_metasound` / `compile_metasound` pair that surface the editor compiler's diagnostics.
- `#2-reviewed-and-confirmed` `OPEN` tester — Re-verified against `AudioAuthoringHandler.cpp`: registered MetaSound RPCs are `create_metasound`, `add_metasound_node`, `connect_metasound_nodes`, `add_metasound_input`, `add_metasound_output`, `set_metasound_default`, `describe_metasound`. No variable/compile/validate handlers exist. Severity Medium justified — graphs compile lazily on editor-open, so workaround exists but is friction. Keeping single ticket consistent with sibling MetaSound feature tickets that bundle related axes under shared root cause.
- `#3-variables-and-validate` `IN-REVIEW` developer — Added `audio.authoring.add_metasound_variable`, `remove_metasound_variable`, `set_metasound_variable_default`, `validate_metasound`, `compile_metasound` in new `Private/Handlers/Audio/MetaSound/MetaSoundVariableHandler.cpp`. Variable handlers wrap `FMetaSoundFrontendDocumentBuilder::AddGraphVariable` / `RemoveGraphVariable` / `SetGraphVariableDefault` and reuse the shared default-literal helper. Validate/compile gracefully degrade to a note when the engine doesn't expose builder-level validation on the running UE version. Regression test `TestMetaSoundVariables.cpp` exercises the AddGraphVariable / RemoveGraphVariable cycle.
- `#4-verify-variable-compile` `DONE` tester — Verified: `audio.authoring.create_metasound` created `/Game/McpVerify/MS_McpVerifyTemp_F_MetasoundVariables`; `audio.authoring.add_metasound_variable` added `VerifyFloat`; `audio.authoring.set_metasound_variable_default` changed it to `2.500000` as observed through `audio.authoring.describe_metasound`; `audio.authoring.compile_metasound` returned `compiled: true`, `valid: true`, `diagnostics: []`; cleanup `asset.delete` returned `success: true`.
