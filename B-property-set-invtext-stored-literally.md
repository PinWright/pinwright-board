---
id: B-property-set-invtext-stored-literally
title: "`property.set` stores `INVTEXT(\"...\")` literally (macro text becomes the value) when the target FText already has a localization identity"
status: OPEN
severity: High
category: bug
tags: [property, property-set, ftext, invtext, text-macro, silent-wrong-data]
encounters: 1
lastSeen: 2026-09-30T09:05:00Z
---

# `property.set` stores `INVTEXT("...")` literally when the target FText is already localized

`property.md` documents that FText values accept "raw text or UE text macros", with `INVTEXT` among the supported
forms. For an FText property whose current value already carries a namespace and key, an `INVTEXT(...)` value is
not parsed: the response reports `applied: true`, and the new value is the macro source itself.

Repro (UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1`, plugin source `2580e7f4`, runtime PIE widget):

```
property.set {objectPath: "/Engine/Transient.LyraEditorEngine_0:B_DroneGameInstance_C_1.W_OverallUILayout_C_0.WidgetTree_0.W_RaceOnlineResultsFrame_C_0.WidgetTree_0.W_LoginUtils",
              propertyName: "AnonimousName", value: "INVTEXT(\"Fierce Mole\")", markDirty: false}
-> applied: true, value: NSLOCTEXT("[9D4D2ABF...]", "81C8524C...", "INVTEXT(\"Fierce Mole\")")
```

The same call with the plain string `"Fierce Mole"` stored `Fierce Mole` under the existing key, as intended.

**Root cause:** `CoerceStringToPersistedFText` (`Source/PinWright/Private/Utils/PropertyImport.cpp`, around
line 551) parses the value with `FTextStringHelper::CreateFromBuffer`. An `INVTEXT` parse has no localization
identity, so the code falls through to the "keep the existing namespace/key" branch. That branch rebuilds the text
from the **raw input string** (`FText::FromString(TextValue)`), not from `ParsedText`.

**Impact:** silent wrong data. The success response echoes the stored value, but a caller that checks only
`applied` builds on a label that reads `INVTEXT("...")`. It happened here on a transient runtime widget; the same
branch serves persisted asset FText.

**Fix (proposed):** in the existing-identity branch, rebuild from `ParsedText.ToString()` when
`CreateFromBuffer` recognised a macro (for example when `ParsedText.ToString() != TextValue`, or via
`FTextStringHelper::ReadFromBuffer` returning a consumed length), and keep `TextValue` only for plain strings. Add
a test: an FText with an existing key, set to `INVTEXT("x")`, reads back as `x` with the key preserved.

**Workaround:** pass the plain string.

## History
- `#1-invtext-literal-on-keyed-ftext` `OPEN` reporter - Filed from a PDS race-results repro, UE 5.8, host `/sdb-disk/src/unreal/unreal-fpv-wt1` (Linux), plugin source `2580e7f4` (loaded binary built 2026-09-29 18:30Z). `property.set` of `AnonimousName` on a live `W_LoginUtils` instance inside a 2-instance listen PIE, with `INVTEXT("Fierce Mole")`, returned `applied: true` and the value `NSLOCTEXT("[9D4D2ABF...]", "81C8524C...", "INVTEXT(\"Fierce Mole\")")`. A retry with the plain string worked. Cheap: one extra call, but only because the echoed value was read.
