---
id: B-metasound-add-input-asserts-on-missing-literal
title: "`audio.authoring.add_metasound_input` crashes editor — missing DefaultLiteral on FMetasoundFrontendClassInput"
status: DONE
severity: Critical
category: bug
tags: [metasound, audio-authoring, crash, assertion]
---

# `audio.authoring.add_metasound_input` crashes editor — missing DefaultLiteral on FMetasoundFrontendClassInput

Calling `audio.authoring.add_metasound_input` on a `UMetaSoundSource`
asset hard-crashes the editor with an engine-side `Literal` assertion
fired from `FMetaSoundFrontendDocumentBuilder::AddGraphInput`. The
plugin constructs `FMetasoundFrontendClassInput` with `Name`,
`TypeName`, `VertexID`, `NodeID`, and `AccessType` but never sets
`DefaultLiteral`. The engine builder later calls
`SetDefaultLiteralOnInputNode` which asserts when the literal has no
type set.

**Stack (from user-reported crash):**

```
Assertion failed: Literal
[File:.../MetasoundFrontendDocument.cpp] [Line: 1297]

MetasoundFrontend!DocumentBuilderPrivate::SetDefaultLiteralOnInputNode()  (MetasoundFrontendDocumentBuilder.cpp:261)
MetasoundFrontend!FMetaSoundFrontendDocumentBuilder::AddNodeInternal()    (MetasoundFrontendDocumentBuilder.cpp:1844)
MetasoundFrontend!DocumentBuilderPrivate::FModifyInterfacesImpl::Execute  (MetasoundFrontendDocumentBuilder.cpp:1152)
MetasoundFrontend!FMetasoundFrontendGraphClass::IterateGraphPages()       (MetasoundFrontendDocument.cpp:1568)
MetasoundFrontend!FMetaSoundFrontendDocumentBuilder::AddGraphInput()      (MetasoundFrontendDocumentBuilder.cpp:1174)
EditorAutomationRpcGateway!AutoHandler_30_()                              (AudioAuthoringHandler.cpp:948)
```

**Repro:**

1. Have a `UMetaSoundSource` asset (e.g. created via
   `audio.authoring.create_metasound`).
2. Call `audio.authoring.add_metasound_input(assetPath=..., inputName="Foo", inputType="Float")`.
3. Editor hard-crashes — not an MCP error response, the whole process
   asserts and dies.

**Root cause (verified in UE 5.6 engine source):**

`AudioAuthoringHandler.cpp:941-948`:

```cpp
FMetasoundFrontendClassInput ClassInput;
ClassInput.Name = FName(*InputName);
ClassInput.TypeName = FName(*InputType);
ClassInput.VertexID = FGuid::NewGuid();
ClassInput.NodeID = FGuid::NewGuid();
ClassInput.AccessType = EMetasoundFrontendVertexAccessType::Reference;

const FMetasoundFrontendNode* InputNode = Builder.AddGraphInput(ClassInput);
```

`FMetasoundFrontendClassInput` carries a private
`TArray<FMetasoundFrontendClassInputDefault> Defaults` (one entry per
graph page). The handler never adds any entry to that array.

Engine flow:
1. `FMetaSoundFrontendDocumentBuilder::AddGraphInput` (called at
   line 948) calls
   `DocumentBuilderPrivate::SetDefaultLiteralOnInputNode`
   (`MetasoundFrontendDocumentBuilder.cpp:237`).
2. That helper calls
   `InClassInput.FindConstDefaultChecked(Frontend::DefaultPageID)`
   (`MetasoundFrontendDocumentBuilder.cpp:261`).
3. `FindConstDefaultChecked` does
   `check(Literal)` at `MetasoundFrontendDocument.cpp:1297` — and
   `Literal` is the result of `Defaults.FindByPredicate(...)`, so the
   assertion fires when **`Defaults` is empty**.

Note: the public `FMetasoundFrontendClassInput::DefaultLiteral` field
is marked deprecated in UE 5.5 (DeprecationMessage:
*"Direct access will be revoked … Field has been rolled into
DefaultLiterals Array"*). Writing to it does **not** populate
`Defaults` and would not avoid the crash; the fix must go through
`InitDefault(...)` / `AddDefault(...)`.

**Fix:**

Populate `ClassInput`'s `Defaults` array via the public
`InitDefault(FMetasoundFrontendLiteral)` API before calling
`AddGraphInput`. Mapping for the four advertised types (from the
`inputType` doc string at handler line 907 — `"Float, Int, Bool,
Audio"`):

```cpp
FMetasoundFrontendLiteral Literal;
if (InputType.Equals(TEXT("Float"), ESearchCase::IgnoreCase))
{
    Literal.Set(0.0f);
}
else if (InputType.Equals(TEXT("Int"), ESearchCase::IgnoreCase) ||
         InputType.Equals(TEXT("Int32"), ESearchCase::IgnoreCase))
{
    Literal.Set((int32)0);
}
else if (InputType.Equals(TEXT("Bool"), ESearchCase::IgnoreCase) ||
         InputType.Equals(TEXT("Boolean"), ESearchCase::IgnoreCase))
{
    Literal.Set(false);
}
else if (InputType.Equals(TEXT("String"), ESearchCase::IgnoreCase))
{
    Literal.Set(FString());
}
else if (InputType.Equals(TEXT("Audio"), ESearchCase::IgnoreCase))
{
    // Audio inputs default to a null object literal
    Literal.Set((UObject*)nullptr);
}
else
{
    // Unknown type — return graceful error instead of crashing
    Ctx.SendError(TEXT("INVALID_TYPE"),
        FString::Printf(TEXT("Unsupported MetaSound input type: %s"), *InputType));
    return true;
}
ClassInput.InitDefault(Literal); // populates Defaults[DefaultPageID]

const FMetasoundFrontendNode* InputNode = Builder.AddGraphInput(ClassInput);
```

Do **not** assign `ClassInput.DefaultLiteral = Literal;` — that field
is deprecated and does not populate the `Defaults` array the engine
asserts against.

Also update the handler's `inputType` doc string (line 907) to list
`"Float, Int, Bool, String, Audio"` if String is supported, or keep
the four-type list and reject `"String"` in the switch above.

Optional ergonomic improvement: accept a `defaultValue` parameter on
the RPC so callers can seed a non-zero default at creation time
(matches what `set_metasound_default` does post-hoc but avoids the
extra round-trip).

**Output side does NOT have the same bug.**
`FMetasoundFrontendClassOutput`
(`MetasoundFrontendDocument.cpp` header at line 1033) is a thin
wrapper over `FMetasoundFrontendClassVertex` with no `Defaults` array
and no `DefaultLiteral` field. There is no
`SetDefaultLiteralOnOutputNode` in the engine builder. The output
handler at `AudioAuthoringHandler.cpp:1031-1038` is structurally
correct and should not be touched as part of this fix.

**Impact:** Critical because it hard-crashes the editor with no
recovery. The entire MetaSound authoring story is gated on this fix —
every higher-level authoring workflow needs `add_metasound_input` to
work without process death. Compounds with the broader MetaSound
authoring gaps tracked in
[`F-metasound-no-destructive-graph-ops`](F-metasound-no-destructive-graph-ops.md)
and [`F-metasound-no-input-output-mutation`](F-metasound-no-input-output-mutation.md).

**Workaround:** None — the call cannot complete without crashing the
editor. Callers must avoid `add_metasound_input` entirely until fixed.

## History
- `#1-literal-assert-crash` `OPEN` reporter — User-observed crash invoking `audio.authoring.add_metasound_input` against a real `UMetaSoundSource` (stack pasted into ticket). Root-cause verified in `AudioAuthoringHandler.cpp:941-948`: `FMetasoundFrontendClassInput::DefaultLiteral` is never populated before `Builder.AddGraphInput(ClassInput)`, leaving the engine builder's literal-type assertion to fire when it sets the default literal on the new input node. Fix is a small literal-population switch on `InputType` before the builder call; the `add_metasound_output` symmetric path likely needs the same treatment. Severity Critical because the failure mode is editor process death, not an MCP error response.
- `#2-reviewed-and-confirmed` `OPEN` reporter — Reviewed against UE 5.6 engine source. Crash mechanism confirmed but root-cause description and fix corrected: (1) the engine assertion at `MetasoundFrontendDocument.cpp:1297` is `check(Literal)` inside `FindConstDefaultChecked`, which fires because the private `FMetasoundFrontendClassInput::Defaults` array is empty — not because a literal has type `Invalid`. (2) The public `DefaultLiteral` field is deprecated in UE 5.5 ("rolled into DefaultLiterals Array"); assigning to it does not populate `Defaults` and would not avoid the crash. Correct fix uses `ClassInput.InitDefault(Literal)`, which adds an entry under `DefaultPageID`. (3) The "same gap on output side" claim is wrong — `FMetasoundFrontendClassOutput` (header line 1033) has no `Defaults` or `DefaultLiteral` fields and no `SetDefaultLiteralOnOutputNode` helper exists; the output handler at `AudioAuthoringHandler.cpp:1031-1038` is structurally fine and should not be touched. Severity Critical confirmed (editor hard-crash, no recovery). Board-folder scan found no duplicate crash ticket; related feature gaps are tracked in `F-metasound-no-input-output-mutation.md` and `F-metasound-no-destructive-graph-ops.md`. Ticket body updated with corrected root-cause analysis, corrected fix snippet, and corrected output-side conclusion.
- `#3-init-default-literal` `IN-REVIEW` developer — Added shared helper `MakeDefaultLiteralForMetaSoundType` at `Private/Handlers/Audio/MetaSound/MetaSoundLiteralFromTypeName.{h,cpp}` covering Float/Int/Int32/Bool/Boolean/String/Audio. Updated `AudioAuthoringHandler.cpp:941-948` to call helper + `ClassInput.InitDefault(Literal)` before `Builder.AddGraphInput(ClassInput)`. Doc string at line 907 updated to advertise String. Unknown types now return `INVALID_TYPE` error rather than crashing. Regression test `TestAddMetaSoundInputLiteral.cpp` (`FAddMetaSoundInputLiteralPopulatesDefaultsTest`) asserts helper-driven literal makes `Builder.AddGraphInput` succeed without hitting the engine literal-check assertion.
- `#4-verify-init-default` `DONE` tester — Verified: created `/Game/Audio/MetaSounds/MS_McpVerifyTemp_BMetaSoundInputLiteral`, ran `audio.authoring.add_metasound_input` with `inputType=Float` and observed nodeId `F187BEB8475B2A57F2F6298DEB686E0B` plus success message; ran unsupported `inputType=NotAType` and observed `INVALID_TYPE`; deleted the temp asset with `existsAfter=false`.
