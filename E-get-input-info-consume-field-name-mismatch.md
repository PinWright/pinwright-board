---
id: E-get-input-info-consume-field-name-mismatch
title: "input.get_input_info wiki documents output field 'bConsumeInput' but the handler returns 'consumeInput'"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [input, enhanced-input, wiki, docs, field-name, get-input-info]
---

# `input.get_input_info` documents an output field name that differs from what it returns

The `input.get_input_info` handler summary (which generates the wiki page)
promises a `bConsumeInput` output field for a `UInputAction`, but the
handler actually serializes that field under the key `consumeInput` — no
`b` prefix. A caller who reads the wiki and looks for `bConsumeInput` in the
response will not find it; the value is there, just under a different key.

Evidence from `Handlers/Input/InputHandler.cpp`:

- Documented (REGISTER_RPC_HANDLER summary, line ~274):
  > "For UInputAction returns valueType and **bConsumeInput**; for
  > UInputMappingContext returns mappingCount."
- Emitted (line ~303):
  ```cpp
  Result->SetBoolField(TEXT("consumeInput"), InputAction->bConsumeInput);
  ```

So the runtime key is `consumeInput` while the doc (and the underlying
`UInputAction::bConsumeInput` UE property it mirrors) say `bConsumeInput`.

**Why it matters:** this is the inspect/verify step of an Enhanced-Input
authoring flow — the whole point is to read back the field and sanity-check
it. A caller who keys the response by the documented `bConsumeInput` reads
`undefined`/missing and may wrongly conclude the IA has no consume flag, or
has to discover the real key by dumping the raw response. The corroborating
ticket `B-input-trigger-modifier-stub-silent-success` (#1 history, body)
also refers to this field as `bConsumeInput`, showing the documented name is
the one people expect.

**Fix:** make the doc and the emitted key agree. Prefer (a) emit the field
as `bConsumeInput`, matching the documented name, the `UInputAction::bConsumeInput`
UE property it mirrors, and the dominant codebase convention (bool UPROPERTY
mirrors keep their `b` prefix across the handlers — `consumeInput` is the
outlier). The alternative (b) — change the REGISTER_RPC_HANDLER summary in
`Handlers/Input/InputHandler.cpp` to say `consumeInput` — is less discoverable
and fights the property name, so it is not preferred.

## Repro

1. `input.create_input_action({name: "IA_ReplayConsume", path: "/Game/Input"})`
   → success.
2. `input.get_input_info({assetPath: "/Game/Input/IA_ReplayConsume.IA_ReplayConsume"})`
   → response contains `"consumeInput": true` (or false) but no
   `"bConsumeInput"` field, contradicting the documented field name.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of an Enhanced-Input authoring task (8 add_mapping calls + 4 get_input_info inspections, all ok=true). Attempt agent's friction note: "get_input_info returns the IA consume field as `consumeInput` though the wiki documents it as `bConsumeInput`." Confirmed in source: `InputHandler.cpp` REGISTER_RPC_HANDLER summary (line ~274) says `bConsumeInput`, but the handler emits `SetBoolField(TEXT("consumeInput"), ...)` at line ~303. The task's own story asked the agent to report `bConsumeInput`, and the agent had to remap it to `consumeInput`. Distinct from the judge-filed E-add-mapping-example-wrong-param (that ticket is the add_mapping example's imcPath/contextPath contradiction; this is a get_input_info output-field-name mismatch). No existing board ticket for the consumeInput/bConsumeInput naming (ripgrep clean). Did not block the task — both fields discoverable — but it is a guaranteed wrong-key lookup for anyone following the documented field name.
- `#2-fix` `IN-REVIEW` developer — Reworded the muddled **Fix:** paragraph (a garbled sentence about `valueType`'s convention) to cleanly recommend option (a). Implemented option (a): `input.get_input_info` now emits the consume flag under the documented key `bConsumeInput` (was `consumeInput`), matching the REGISTER_RPC_HANDLER summary, the `UInputAction::bConsumeInput` UE property it mirrors, and the codebase's dominant bool-UPROPERTY-mirror convention. File: `Source/PinWright/Private/Handlers/Input/InputHandler.cpp` (line ~303). Regression test added: `PinWright.input.get_input_info.EmitsDocumentedConsumeKey` in `Source/PinWright/Private/Tests/Gameplay/TestInputHandlers.cpp` — creates a real UInputAction via `input.create_input_action`, calls `input.get_input_info`, and asserts the response carries `bConsumeInput` and NOT the legacy `consumeInput` key (plus asserts the summary documents `bConsumeInput`); it would fail if the emit reverted to the stripped-b key. Not compiled/run here (later phase verifies green).
