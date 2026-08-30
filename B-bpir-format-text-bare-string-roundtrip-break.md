---
id: B-bpir-format-text-bare-string-roundtrip-break
title: "BPIR decompile emits bare-string FText defaults that the require-namespace-key compile gate rejects"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, ftext, localization, round-trip, format-text]
---

# BPIR bare-string FText defaults break round-trip under the require-namespace-key policy

When a `PC_Text` pin's `DefaultTextValue` carries no localization
identity — neither Namespace/Key nor a string-table entry (the FText
is culture-invariant, e.g. authored via `FText::FromString` or pasted
as a raw literal in the editor) — `BpirDecompiler::ResolveInputValue`
falls through to `EscapeBpirString(DisplayString)` and emits a bare
quoted string. This happens in two places in
`Source/EditorAutomationRpcGateway/Private/Decompiler/BpirDecompiler.cpp`:
line 2119 (the early `PC_Text` branch keyed on `DefaultTextValue`) and
line 2298 (the secondary `DefaultTextValue` branch later in the same
function). `FormatText` Format pins go through the same path, so they
decompile to `call Format_Text(Format: "literal", ...)` rather than
an NSLOCTEXT/LOCTABLE form.

Under `F-require-ftext-localization-identity` (DONE),
`CoerceStringToPersistedFText` in `Utils/PropertyImport.cpp:245` rejects
plain strings (`"Persisted FText values require a non-empty namespace
and key..."`, `:281`) unless an existing localized value is being
overwritten in place (that in-place branch is `:270-279`). `FCodeNodeEmitter::CreateFormatTextNode`
(`Compiler/CodeNodeEmitter.cpp:1108`) calls
`CoerceStringToPersistedFText(FormatString, /*ExistingText=*/nullptr, ...)`,
so the no-existing-context path is the only one available during a
fresh `blueprint.compile_bpir` and the call returns
`[COMPILE_FAILED] Line N: FormatText Format requires an FText namespace
and key: Persisted FText values require a non-empty namespace and key`.

Net effect: any legacy asset with a culture-invariant authored FText
default — including the `Format_Text` Format pin — can be dumped via
`asset.dump_folder` and inspected as BPIR, but the resulting BPIR is
not re-submittable. The decompile side advertises a syntax the
compile side rejects.

**Distinct from `B-bpir-ftext-localization-metadata-lost` (DONE)**:
that fix made the decompiler emit `NSLOCTEXT(...)` when the FText
**has** namespace/key. This is the inverse: when there is **no**
identity, the decompiler still emits a bare string, but the compile
gate added by `F-require-ftext-localization-identity` no longer
accepts bare strings without an existing localized value to inherit
identity from.

## Repro

Asset: `/App/App/UI/W_FoundGasLeaks`. Dumped BPIR in
`.editor-automation/asset-dumps/App/App/UI/W_FoundGasLeaks/bpir.txt:15`:
`%n1: text = call Format_Text(Format: "{current}/{total}", current: $`Search Track`.LeaksFound, total: %n0)`.
Submitting that BPIR back to `blueprint.compile_bpir` fails with
`[COMPILE_FAILED] Line N: FormatText Format requires an FText namespace
and key`. Compile succeeds only after wrapping the Format string as
`NSLOCTEXT("W_FoundGasLeaks", "ScoreFormat", "{current}/{total}")`.

**Workaround:** Hand-edit the BPIR to wrap each bare FText literal in
`NSLOCTEXT("Asset", "AutoKey", "Source")` before recompile. Lossy —
the namespace/key is synthesized, not recovered from the original
asset.

**Fix:** Either (a) synthesize a deterministic namespace/key in the
decompiler when `DefaultTextValue` lacks identity (e.g.
`NSLOCTEXT("<asset>", "<node>.<pin>", "<source>")`) so round-trip
produces a re-compilable BPIR, or (b) relax the compile gate to
accept bare strings as invariant FText when there is no existing
identity to preserve, mirroring engine `FText::FromString` semantics
for legacy un-localized defaults. Option (a) is consistent with the
DONE `F-require-ftext-localization-identity` policy; option (b) is a
narrower exception. `FCodeNodeEmitter::CreateFormatTextNode` and the
two `BpirDecompiler.cpp` branches at lines 2119 and 2298 are the
focal points.

## History
- `#1-initial-repro` `OPEN` reporter — Confirmed in source: `BpirDecompiler.cpp:2119` and `:2298` fall through to `EscapeBpirString(DisplayString)` when `DefaultTextValue` has neither namespace/key nor a string-table entry. `Utils/PropertyUtils.cpp:952` rejects bare strings unless an existing localized value is being overwritten in place; `CodeNodeEmitter.cpp:1118` passes `ExistingText=nullptr` for `FormatText`, so the new-node path can never inherit identity. Verified replay on `/App/App/UI/W_FoundGasLeaks` (dumped BPIR contains bare-string `Format_Text(Format: "{current}/{total}", ...)`, recompile fails with the documented `FormatText Format requires an FText namespace and key` message). Distinct from `B-bpir-ftext-localization-metadata-lost` (DONE, identity-present case) and complementary to `F-require-ftext-localization-identity` (DONE, the policy that introduced the gate).
- `#2-synthesize-nsloctext` `IN-REVIEW` developer — Decompiler now synthesizes a deterministic NSLOCTEXT identity (namespace from owning asset short name, key from node-guid-prefix + pin name) at both the early PC_Text branch (`BpirDecompiler.cpp` ~2119) and the secondary `DefaultTextValue` branch (~2298) when the FText pin lacks namespace/key. New `SynthesizeNSLocTextLiteral` helper in the BpirDecompiler anonymous namespace reuses the existing `BpirStructLiteralUtils::EscapeBpirStringInner` for the three sub-literals. Compile gate from `F-require-ftext-localization-identity` is unchanged. Round-trip restored; test extended in `TestBpirFTextLocalizationRoundTrip.cpp` (Scenario 5) — asserts the synthesized namespace matches the asset short name, the key carries the pin segment, and a second decompile of the same Blueprint is byte-identical (idempotency).
- `#3-verified-synthesized-nsloctext` `DONE` tester — Verified: `blueprint.decompile` of `/App/App/UI/W_FoundGasLeaks` now emits `Format: "NSLOCTEXT(\"W_FoundGasLeaks\", \"DE680131.Format\", \"{current}/{total}\")"` for the `Format_Text` Format pin that the on-disk asset stores as a culture-invariant FText. Namespace is the asset short name; key prefix `DE680131` matches the owning K2Node's GUID-prefix segment. This BPIR is re-submittable to `blueprint.compile_bpir` without the previous `[COMPILE_FAILED] FormatText Format requires an FText namespace and key` rejection (re-confirmed in same session via successful surgical insert on the same widget's graph).
- `#4-repoint-citations-after-property-utils-split` `DONE` reporter — Citation maintenance only; **no behavioural claim changes and the status is untouched**. `Utils/PropertyUtils.cpp` was split into `PropertyExport.cpp` / `PropertyImport.cpp` / `PropertyInspection.cpp` / `PropertyDiff.cpp` (`PropertyUtils.h` survives only as a deprecated umbrella forwarder), so this ticket's `PropertyUtils.cpp` citations were unresolvable paths, not stale line numbers. **This is the one ticket in the sweep whose code landed in `PropertyImport.cpp` rather than `PropertyExport.cpp`** — correctly, since `CoerceStringToPersistedFText` is a write-side coercion. Verified at plugin HEAD `ef8a1f1b`. Map: body `Utils/PropertyUtils.cpp:916` → `Utils/PropertyImport.cpp:245` (the function spans `:245-283`), repointed in place; the rejection at "line 952" → `PropertyImport.cpp:281`, which is verbatim the `"Persisted FText values require a non-empty namespace and key; pass NSLOCTEXT(\"Namespace\", \"Key\", \"Source\") or update an existing localized value"` string the body quotes; the "unless an existing localized value is being overwritten in place" branch, which the body asserts without citing, is `PropertyImport.cpp:270-279` and turns on `FText::ChangeKey` at `:274` — now cited so a fixer can check the claim rather than take it. `#1`'s `Utils/PropertyUtils.cpp:952` is left verbatim per the append-only rule and maps to `PropertyImport.cpp:281`.
