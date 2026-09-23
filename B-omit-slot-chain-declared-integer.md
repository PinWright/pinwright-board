---
id: B-omit-slot-chain-declared-integer
title: "`widget.export_xml`'s `omit_slot_chain` is declared `integer` while the handler reads it with `GetBool` and the wiki documents it as a flag defaulting to false, so `{\"omit_slot_chain\": true}` is refused with PARAM_TYPE_MISMATCH"
status: OPEN
severity: Medium
category: bug
tags: [widget, export-xml, umg, param-spec, declared-type, param-type-mismatch, boolean-flag, discrete-index-sweep]
encounters: 1
lastSeen: 2026-09-17T00:00:00Z
---

# A boolean flag declared `integer`, one line above its own alias declared `boolean`

Observed 2026-09-17 exporting UMG trees through the MCP against UE 5.8, host project
`X:\src\unreal\unreal-fpv`, plugin at `c86eb581`. `widget.export_xml` with
`{"omit_slot_chain": true}` is refused before the handler runs:

```
[PARAM_TYPE_MISMATCH] 'omit_slot_chain' is declared integer and was sent as boolean
```

`{"omit_slot_chain": 1}` is accepted and behaves as the flag. So the parameter is reachable, but
only through a spelling that neither the wiki page nor the handler's own description implies.

## Evidence

The declaration, its alias, and the reader disagree inside one file:

`Source/PinWright/Private/Handlers/UI/WidgetXmlExportHandler.cpp:65-68`

```cpp
        RPC_PARAM_OPT("omit_slot_chain", "integer",
            "Drop the Slot.Parent/Slot.Content sibling-chain expansion that causes O(N^2) blow-up on panel-heavy widgets (default false)"),
        RPC_PARAM_OPT("compact", "boolean",
            "Alias for omit_slot_chain")
```

Line `:65` is the defect. Its own description says "default false" - a boolean's default, not an
ordinal's - the alias two lines down is declared `boolean`, and the reader takes both keys as
booleans through one expression:

- `WidgetXmlExportHandler.cpp:142` -
  `const bool bOmitSlotChain = Ctx.GetBool(TEXT("omit_slot_chain"), false) || Ctx.GetBool(TEXT("compact"), false);`
- `Handlers/HandlerContext.cpp:37-44` - `GetBool` is `Payload->GetBoolField(Key)` with a default;
  there is no integer path for either key. The repo has zero `GetInt*(TEXT("omit_slot_chain"))` sites.

The refusal happens in the dispatcher's declared-type gate, before the handler body:

- `Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:215` - `CollectDeclaredTypesByWireName`
- `RpcDispatcher.cpp:229` - per-key `PinWrightCheckDeclaredType`
- `RpcDispatcher.cpp:241` - the fault string the caller sees, `'%s' is declared %s and was sent as %s`
- `RpcDispatcher.cpp:252` - `SendError(ErrorCodes::ERR_PARAM_TYPE_MISMATCH, ...)`, `:256` `return false`
- `Handlers/ParamTypeCheck.h:349-359` - `integer` delegates to `TryParseStrictJsonInteger`;
  `Utils/JsonUtils.cpp:295-298` is its `else` branch, which refuses every value that is not
  `EJson::Number` or `EJson::String`, so an `EJson::Boolean` can never pass. The header's own
  acceptance table (`:26`) lists `bool` in the integer row's refused set, one line above the
  `boolean` row (`:27`) that would have accepted it.

**Why `1` works and `"true"` does not.** `1` is an integral `EJson::Number`, so it clears the gate
(`JsonUtils.cpp:278-293`), and `GetBoolField` then reads it through `FJsonValueNumber::TryGetBool`
(`C:\UE_5.8\Engine\Source\Runtime\Json\Private\Dom\JsonValue.cpp:440-444`, `OutBool = (Value != 0.0)`),
yielding `true`. `{"omit_slot_chain": "true"}` is refused as well - `ParseStrictSignedIntegerString`
rejects it - even though the `boolean` row accepts boolean-spelled strings. The one accepted
spelling is therefore an accident of number-to-bool coercion, not a designed contract.

**The generated wiki inherits the wrong token and contradicts itself on the same page.**
`Saved/PinWright/wiki/widget.export_xml.md:29` renders `` `omit_slot_chain` (`integer`, optional):
... (default false) `` directly above `:30` `` `compact` (`boolean`, optional): Alias for
omit_slot_chain ``, and the overlay prose merged onto the same page (`:44`, authored at
`docs/wiki-src/widget.md:283`) discusses `omit_slot_chain` purely as a flag. An agent reading the
page has every reason to send `true` and no way to learn that only `1` is admitted.

Unlike `B-widget-set-slot-declared-integer`, this is not a silent false-success: the only shapes the
gate admits (`1` / `0`) are the shapes the reader handles correctly, so no caller is told a write
happened that did not. The damage is a refused valid call plus a published contract that is wrong.

## Cause

Same defect class as `B-widget-set-slot-declared-integer` and
`B-layertag-declared-integer-blocks-tag-addressing`: the discrete-index name-token sweep. Its
tokenizer splits on `_` / `-` and camelCase (`Source/PinWright/Private/Tests/Infra/TestParamTypeGate.cpp:346-374`),
so `omit_slot_chain` tokenizes to `omit` / `slot` / `chain`, and `slot` is a literal member of
`DiscreteTokens` (`:376-379`, alongside `index`, `count`, `lod`, `level`, `layer`, `frame`). The
exception list at `:394-405` excludes `path` / `name` / `rate` / `time` / `duration` / `distance` /
`tag` / `autoIndex` / `lodType` - nothing that spares a boolean flag whose name happens to mention a
slot. As on the sibling tickets, `git log -S` resolves only to `8748c637` ("PinWright 0.8.0: first
open-source release"), the squash, so the attribution is by mechanism, not by revision.

**Fix the class, not this literal.** Sweeping the registry for `RPC_PARAM*` declared `"integer"`
whose parameter has a `GetBool` / `GetObjectField` reader in the same file finds exactly four
casualties, all of them `slot`-token false positives, three of which are not yet ticketed:

| declaration | reader | ticket |
|---|---|---|
| `Handlers/UI/WidgetXmlExportHandler.cpp:65` `omit_slot_chain` | `:142` `GetBool` | this one |
| `Handlers/UI/WidgetDescribeHandler.cpp:170` `include_slot` ("default true") | `:275` `GetBool` | none |
| `Handlers/UI/WidgetDuplicateHandler.cpp:209` `copySlot` ("Defaults true.") | `:284` `GetBool` | none |
| `Handlers/UI/WidgetSetHandler.cpp:122` `slot` | `:141` `GetObjectField` | `B-widget-set-slot-declared-integer` |

`widget.describe`'s `include_slot` and `widget.duplicate`'s `copySlot` are the same bug with no
alias to fall back on: neither has a `boolean`-declared twin, so `include_slot: false` and
`copySlot: false` - the only values a caller ever sends, both being default-true flags - are
refused outright, and the only admitted disable value is `0`. Fix all four in one pass and add the
missing exception rather than re-filing this shape a fourth time.

## Fix

One token at `WidgetXmlExportHandler.cpp:65`: `"integer"` -> `"boolean"`, matching the `compact`
alias at `:67`. Same one-token flip at `WidgetDescribeHandler.cpp:170` and
`WidgetDuplicateHandler.cpp:209`. Then regenerate the wiki (it regenerates at editor startup; do
not hand-edit `Saved/PinWright/wiki/widget.export_xml.md`).

The durable half is in the sweep's classifier, not the call sites: `IsDiscreteReaderName`
(`TestParamTypeGate.cpp:346`) should not classify a name as a discrete selector when its reader is
a boolean or an object reader. The cheapest correct form is the one the `tag` exception already
models - the classifier reasons from the name alone, so the reader-shape fact has to be encoded as
a name exception (a `bOmit`/`include`/`copy` prefix, or a flag-token set) - but the honest version
correlates against the actual reader, the way the ratchet's second pass already does for `GetInt`.

Nothing in the tree blocks the flips. The ratchet
`PinWright.infra.dispatcher.ParamTypeGate.DiscreteReadersHaveIntegerDeclarations`
(`TestParamTypeGate.cpp:446-447`) only asserts integer declarations for names collected from
`GetInt` / `GetIntOr` / `RequireInt` call sites (`:540-568`, gated by
`IsIntegerReaderDeclarationCandidate` at `:551`), and none of these four has an integer reader. Its
scanner-health helper `CheckDerivedReaderShape` is wired only to `actor.set_instance_transforms`,
`drive.observe` and `sequencer.set_playhead` (`:512-518`), so no `widget.*` method is covered.

**No test catches it.** `widget.export_xml`'s compact path is exercised at
`Tests/WidgetXml/TestWidgetXmlHandlers.cpp:1064-1067`, which sets `compact` as a JSON bool and
invokes through `InvokeHandlerWithCapture` - a direct `Reg.Func(Ctx)` call that never runs
`ValidateHandlerParams`. The canonical key `omit_slot_chain` has zero coverage at any level, and
even the covered key would pass in-process against a declaration the real dispatcher refuses. This
is the identical blind spot recorded on both sibling tickets; a gate-level test through
`FRpcDispatcher::ProcessRequest` (`Tests/Infra/DispatcherTestHelpers.h`) asserting
`{"omit_slot_chain": true}` reaches the handler body is what would have caught all four.

## Workaround

Two, both source-verified, the first live-verified:

- Pass `1` instead of `true`: `{"omit_slot_chain": 1}`. Integral numbers clear the gate and
  `GetBoolField` coerces non-zero to `true`. Use `0` to disable.
- Use the alias `compact`, which is correctly declared `boolean` at `WidgetXmlExportHandler.cpp:67`,
  so `{"compact": true}` clears the gate and reaches the same `bOmitSlotChain`. This is also the
  spelling `E-widget-export-xml-token-limit`'s verification used, which is why that ticket's `DONE`
  evidence remains reproducible - the alias, not the canonical key, was tested.

## Related

- `B-widget-set-slot-declared-integer` (OPEN) - filed the same day, same file family, same sweep,
  same untested gate. That one is `High` because its declaration admits no working value at all and
  the single shape it does admit returns a silent false-success; this one has a working spelling and
  a working alias, hence `Medium`. Fix them together.
- `B-layertag-declared-integer-blocks-tag-addressing` (IN-REVIEW) - first instance of this class;
  its `layer`-token false positive is what motivated the `tag` exception at `TestParamTypeGate.cpp:403`.
- `B-discrete-index-params-truncate-fractions` (IN-REVIEW) - the sweep whose name-token heuristic
  collects `slot`; claims semantic-name exceptions were restored, names none of these four.
- `B-param-type-never-validated` - the gate itself, working as designed here.
- `E-widget-export-xml-token-limit` (DONE) - added `omit_slot_chain` / `compact`; its `#2` records
  the parameters being introduced, its `#5` verified through `compact=true`.
- `B-asset-dump-widget-tree-xml-slot-recursion-explosion` (DONE) - the `asset.dump` half of the same
  slot-chain blow-up; references the flag at its `:111`.

## History
- `#1-boolean-flag-refused-by-declared-type` `OPEN` reporter - Filed 2026-09-17 exporting UMG trees on UE 5.8, host `X:\src\unreal\unreal-fpv`, plugin `c86eb581`. `widget.export_xml` with `{"omit_slot_chain": true}` refused with `[PARAM_TYPE_MISMATCH] 'omit_slot_chain' is declared integer and was sent as boolean`; `{"omit_slot_chain": 1}` works. Verified in source both sides: declaration `"integer"` at `WidgetXmlExportHandler.cpp:65`, alias `compact` declared `"boolean"` at `:67`, both consumed as booleans at `:142` via `Ctx.GetBool` (`HandlerContext.cpp:37-44`), refusal at `RpcDispatcher.cpp:252` before the handler runs, `integer` refusing `EJson::Boolean` at `ParamTypeCheck.h:349-359` -> `JsonUtils.cpp:295-298`. Traced why `1` is admitted (integral number clears the gate, `FJsonValueNumber::TryGetBool` at engine `JsonValue.cpp:440-444` coerces non-zero to true) and why `"true"` is not. Generated page `Saved/PinWright/wiki/widget.export_xml.md:29` publishes `integer` with a "default false" description directly above the `boolean`-typed alias at `:30`, and overlay prose (`docs/wiki-src/widget.md:283`) treats it as a flag. Attributed to the discrete-index sweep by mechanism: `omit_slot_chain` tokenizes to `omit`/`slot`/`chain` and `slot` is in `TestParamTypeGate.cpp:376-379`'s `DiscreteTokens` with no applicable exception at `:394-405`; `git log -S` reaches only the 0.8.0 squash. Swept the whole registry for `"integer"` declarations with `GetBool`/`GetObjectField` readers and found exactly four, all `slot`-token false positives: this one, `widget.describe`'s `include_slot` (`WidgetDescribeHandler.cpp:170`, read `:275`), `widget.duplicate`'s `copySlot` (`WidgetDuplicateHandler.cpp:209`, read `:284`), and the already-ticketed `widget.set` `slot`. The two unticketed ones are default-true flags with no boolean alias, so `false` is unreachable on both. Severity Medium: a documented flag rejects the value its own docs imply, but `1` and the `compact` alias both work and no result is silently wrong. Recommended fixing the class (the classifier at `TestParamTypeGate.cpp:346`) plus all four declarations in one pass, not this literal. No fix attempted.
