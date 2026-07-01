---
id: B-bpir-ftext-localization-metadata-lost
title: "BPIR FText decompile strips NSLOCTEXT namespace+key — round-trip silently un-localizes BP text defaults"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, ftext, localization]
---

# FText decompile loses namespace / key

`Decompiler/BpirTextEmitter.cpp:~1838` emits FText pin defaults
via `EscapeBpirString(DefaultTextValue.ToString())`. This produces
the **display string only** — the FText's namespace and key are
dropped.

The compiler's `CoerceStringToPersistedFText` accepts a plain
escaped string as a culture-invariant unlocalized literal. Any BP
with a localized FText default (an actual NSLOCTEXT entry tied to
a translation table) decompiles to its display string and
re-compiles as **unlocalized invariant text**, silently breaking
localization.

## Why it matters in PDS

PDS uses Russian + English localization. Widget BP defaults that
are NSLOCTEXT-keyed will round-trip to invariant strings,
detaching them from the translation tables. Translators' work
silently drops out of any decompile-edit-recompile workflow.

## Repro

Pick a widget BP with a localized text default. Decompile via
`blueprint.decompile`. Compare the BPIR with the asset's
`properties.json` value of the same field — `properties.json`
shows the canonical `NSLOCTEXT("Namespace", "Key", "Display")`
form, BPIR shows only `"Display"`.

## Fix

1. **Decompile**: when the FText has a `Namespace` / `Key` (via
   `FTextInspector::GetNamespace` / `GetKey`), emit the canonical
   NSLOCTEXT form as a BPIR string literal:

   "NSLOCTEXT(\"Namespace\", \"Key\", \"Display\")"

   Bare quoted strings remain valid for invariant-text cases.
2. **Compile**: no FText-specific changes needed.
   `PropertyUtils::CoerceStringToPersistedFText` already calls
   `FTextStringHelper::CreateFromBuffer`, which understands
   `NSLOCTEXT(...)` natively.
3. **Compile-side unescape** (general fix, not FText-specific):
   `CodePinResolver` strips outer quotes from BPIR string-literal
   defaults but did not unescape inner `\"` and `\\` sequences.
   Adding the unescape step after the outer-quote strip fixes FText
   round-trip and any other string-literal default containing
   escaped quotes.

## Acceptance

- Decompile a BP with NSLOCTEXT default; BPIR contains
  `nstext("Ns", "Key", "Display")`.
- Recompile; the resulting FText property has the same
  Namespace/Key as the original.
- Round-trip a localized widget BP: translation table linkage
  preserved.

## History
- `#1-initial-spec` `OPEN` reporter — Type-system parity audit confirmed FText decompile strips NSLOCTEXT metadata. Compiler accepts plain string as invariant. Round-trip silently un-localizes BP text defaults — direct hit on PDS Russian+English localization workflow. `nstext("Ns","Key","Display")` literal form proposed; bare strings remain invariant-only.
- `#2-nsloctext-form-implemented` `IN-REVIEW` developer — Decompile emits NSLOCTEXT(...) when FText has namespace/key (via FTextInspector). Compile already handles NSLOCTEXT via FTextStringHelper::CreateFromBuffer. Added missing inner-escape unescape in CodePinResolver after outer-quote strip — fixes FText round-trip and is a general fix for any string default with embedded quotes. Regression test FFTextLocalizationRoundTripTest.
- `#3-review-iteration-1` `IN-REVIEW` developer — Fix iteration 1: pre-escape inner NSLOCTEXT namespace/key/display before format (was emitting unparseable BPIR for any value containing quote/backslash); factored duplicated unescape into BpirStructLiteralUtils::UnescapeBpirString and call from both BpirValueResolver and CodePinResolver; added a production-decompile-path scenario to TestBpirFTextLocalizationRoundTrip closing the counterfactual gap.
- `#4-review-iteration-2` `IN-REVIEW` developer — Fix iteration 2: extracted `BpirStructLiteralUtils::EscapeBpirStringInner` so the NSLOCTEXT inner-escape in BpirDecompiler shares the same transform as `EscapeBpirString` (`EscapeBpirString` now wraps the inner helper with quotes; behaviour unchanged).
- `#5-verify-nsloctext-roundtrip` `DONE` tester — Verified: created temp Blueprint `/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_ftext_localization_metadata_lost`, ran `blueprint.compile_bpir` with `PrintText(InText: "NSLOCTEXT(\"McpVerify\", \"FTextLocalization.Metadata\", \"Localized verify\")")`, observed `compiled:true`, `status:"UpToDate"`, `errors:[]`, then `blueprint.decompile` returned BPIR containing the escaped `NSLOCTEXT(\"McpVerify\", \"FTextLocalization.Metadata\", \"Localized verify\")` literal; deleted the temp asset afterward.
