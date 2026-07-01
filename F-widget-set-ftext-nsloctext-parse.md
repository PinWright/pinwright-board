---
id: F-widget-set-ftext-nsloctext-parse
title: "`widget.set` should parse NSLOCTEXT() macro strings into proper FText literals"
status: DONE
severity: Medium
category: feature
tags: [widget-set, ftext, localization, nsloctext]
---

# `widget.set` should parse NSLOCTEXT() macro strings into proper FText literals

When setting an FText property via `widget.set` (e.g. `ButonText`, `Text`), passing an `NSLOCTEXT("Namespace", "Key", "SourceString")` macro string as the value stores the **entire raw macro invocation as the FText's source string**, not as a localized FText with namespace/key/sourceString set correctly.

## Repro

UE editor's XML export of a hand-authored localized FText on a button looks like:

```
ButonText="NSLOCTEXT(&quot;[40550D265FEE9C4C0F2C07C68252F7D6]&quot;, &quot;W_HUD_RaceTrackEnd.CanvasPanel.SizeBox.AnalyzeButton.ButonText&quot;, &quot;Анализ&quot;)"
```

Calling:

```
widget.set(
  widgetPath: "/App/App/UI/LobbyAndMenu/HUD/W_HUD_RaceTrackEnd",
  widgetName: "SaveButton",
  properties: {
    "ButonText": "NSLOCTEXT(\"W_HUD_RaceTrackEnd\", \"SaveReplay\", \"Сохранить\")"
  }
)
```

Stores the literal string `NSLOCTEXT("W_HUD_RaceTrackEnd", "SaveReplay", "Сохранить")` as the text's source string. At runtime the button renders `NSLOCTEXT("W_HU...` (truncated by the button's internal text width) instead of `Сохранить`.

`widget.export_xml` afterwards shows:

```
ButonText="NSLOCTEXT(\"W_HUD_RaceTrackEnd\", \"SaveReplay\", \"Сохранить\")"
```

i.e. the XML export treats the whole string as the SourceString even though a valid NSLOCTEXT invocation appears inside.

## Workaround

Pass a plain string: `"ButonText": "Сохранить"`. That stores a **CultureInvariant** FText, which renders correctly but is not localized. Acceptable for hardcoded labels, not acceptable for anything that needs translation.

## Why it matters

The authoring flow for a localized button needs three round-trippable fields: Namespace, Key, SourceString. UE's FText literal serialization stores all three. BPIR / widget authoring via MCP should be able to round-trip through the same format UE uses. Today it can't, so every MCP-authored button either:
1. Ships without localization (CultureInvariant), or
2. Requires a manual designer pass in the UE editor to convert to localized.

Given that the project ships in multiple languages (Russian + English-era strings scattered), option 2 is a real cost.

## Proposal

In `WidgetSetHandler.cpp` (or wherever `FText` properties are marshalled in `widget.set`), when the incoming value is a string:

1. Detect the `NSLOCTEXT("ns", "key", "src")` macro shape (case-insensitive, escaped or unescaped quotes).
2. Parse out namespace/key/sourceString.
3. Emit an FText via `FText::ChangeKey(Namespace, Key, FText::FromString(SourceString))` or the UE 5.4+ equivalent (`FText::AsLocalizable_Advanced`).
4. Fall through to `FText::FromString()` (CultureInvariant) when the macro isn't detected, preserving current behaviour.

Alternative (structured): accept a JSON object form for FText props:

```json
"ButonText": { "Namespace": "W_HUD_RaceTrackEnd", "Key": "SaveReplay", "SourceString": "Сохранить" }
```

Either form works; the macro parser is more ergonomic for authors who copy from existing NSLOCTEXT references.

## History
- `#1-reported` `OPEN` reporter — Hit while setting the new SaveButton's `ButonText` on `W_HUD_RaceTrackEnd`. Passed `NSLOCTEXT("W_HUD_RaceTrackEnd", "SaveReplay", "Сохранить")` expecting it to parse; got a button rendering the raw macro string. Confirmed by `widget.export_xml` reading back the literal invocation as the stored value. Fell back to plain `"Сохранить"` (CultureInvariant) to get the button rendering correctly; filing this for a future pass that adds localized authoring.
- `#2-confirmed-property-set` `OPEN` reporter — `property.set` on a TextBlock's `Text` with the same NSLOCTEXT macro string exhibits identical behaviour: the macro invocation is stored as the raw source string, visible on export as `Text="NSLOCTEXT(\"...\", \"...\", \"3rd person\")"` and renders verbatim at runtime. `property.set` is not a workaround — both handlers route through the same FText-from-string path. Scope of the fix should cover any handler that marshals FText from a string value.
- `#3-parsed-nsloctext-in-property-utils` `IN-REVIEW` developer — Added `CoerceJsonStringToText` helper in `PropertyUtils.cpp` (unnamed namespace); called from both the scalar FText branch (line 439) and the array-of-FText branch (line 927). Detects `NSLOCTEXT("ns","key","src")` (case-insensitive, with `\"` / `\\` unescape), emits via `FText::AsLocalizable_Advanced`, falls through to `FText::FromString` on parse failure. Shared helper covers both `widget.set` and `property.set` since they route through the same FText marshalling path. Pinned by `FApplyJsonValueToProperty_NSLOCTEXT_Test` in `TestPropertyUtils.cpp`.
- `#4-replaced-with-ftextstringhelper` `IN-REVIEW` developer — Simplify pass replaced the hand-rolled `CoerceJsonStringToText` (~120 lines) with direct `FTextStringHelper::CreateFromBuffer(*TextValue)` calls at the two call sites in `PropertyUtils.cpp`. UE's built-in parser handles `NSLOCTEXT` / `LOCTEXT` / `INVTEXT` literal forms natively and falls back to `FText::FromString` on parse failure — same contract as the removed helper, zero new lines of parser code. `FApplyJsonValueToProperty_NSLOCTEXT_Test` remains the pin (assertions unchanged).
- `#5-verified-localized-roundtrip` `DONE` tester — Verified live on `/Game/App/UI/Test/W_McpVerifyTemp.TestText`. `mcp__editor_automation__.call path="widget.set" args={"widgetName":"TestText","properties":{"Text":"NSLOCTEXT(\"W_McpVerifyTemp\", \"TestText.Sample\", \"Сохранить\")"}}` succeeded. `mcp__editor_automation__.call path="widget.export_xml" args={...}` then read the property back as `Text="NSLOCTEXT(&quot;W_McpVerifyTemp&quot;, &quot;TestText.Sample&quot;, &quot;Сохранить&quot;)"` — proper localized form with namespace + key + source preserved, NOT the raw-source-string fallback. Cross-checked by setting a plain string `"PlainString"` and reading back via `mcp__editor_automation__.call path="property.get" args={...}`: returns `value: "PlainString"` (no NSLOCTEXT wrapping), confirming the serializer only emits NSLOCTEXT form for properly-localized FTexts. The macro is being parsed by `FTextStringHelper::CreateFromBuffer` as intended.
