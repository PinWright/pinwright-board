---
id: B-asset-dump-ftext-not-structured
title: "asset.dump FText properties dumped as raw NSLOCTEXT(...) macro strings instead of structured {namespace, key, defaultValue}"
status: WONTFIX
severity: Medium
category: bug
tags: [asset-dump, ftext, localization]
---

# asset.dump FText properties dumped as raw NSLOCTEXT(...) macro strings instead of structured {namespace, key, defaultValue}

Localized strings are dumped as the raw C++ NSLOCTEXT macro form: `"NSLOCTEXT(\"[D52A6186DB11FA07F459BB0902FDA085]\", \"71F3A5474531ABBB2032FC9AF0B57DEC\", \"Hero Pawn\")"`.

Consumers wanting either (a) the displayed text or (b) ns/key for translation lookups must regex-parse the string. No structured triple emitted, and 1,056 properties.json files contain NSLOCTEXT-formatted FText values.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Input/Mappings/IMC_Default/properties.json`.
2. Observe: FText fields appear as `"NSLOCTEXT(\"...\", \"...\", \"...\")"` strings, not structured objects.

**Fix (proposed):** FText property exporter uses `FTextStringHelper::WriteToBuffer`-style ExportText. Replace with a structured emitter that surfaces `{namespace, key, defaultValue, displayText}` as a JSON object. (NSLOCTEXT ns/key being hex GUIDs is a separate content-authoring issue per `feedback_nsloctext_readable.md`, not a dump bug.)

## History
- `#1-initial-repro` `OPEN` reporter — FText properties dump as raw `NSLOCTEXT(\"<ns>\", \"<key>\", \"<text>\")` macro strings instead of `{namespace, key, defaultValue}` JSON objects. Sample path: `Game/Input/Mappings/IMC_Default/properties.json`. 1,056 properties.json files contain NSLOCTEXT-formatted FText values.
- `#2-wontfix-loctext-canonical` `WONTFIX` user — Project uses LOCTEXT/loctable style throughout; the `NSLOCTEXT(ns, key, default)` macro form IS the canonical representation we want in the dump. Restructuring into `{namespace, key, defaultValue}` would re-shape data the consumer already reads natively. Closing without code changes.
