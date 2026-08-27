---
id: E-consolidate-editor-open-lookup
title: "The Niagara and Material editor-open guards duplicate a FindEditorForAsset lookup that should be shared"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, material, editor-open-guard, duplication, refactor, EDITOR_OPEN]
encounters: 1
lastSeen: 2026-08-28
---

# Share the lookup, not the policy

`PinWrightNiagara::FindOpenAssetEditorForNiagaraAsset` and `PinWright::Material::IsMaterialEditorOpen`
both wrap `UAssetEditorSubsystem::FindEditorForAsset(Asset, /*bFocusIfOpen=*/false)` in about fifteen
lines of the same code.

**Do not merge the guards themselves.** They are the same shape but not the same contract, and the
per-verb classification done for `B-niagara-editor-open-guard-missing-on-mutators` is why: Material's
is a blanket clobber guard fused into `LoadMaterialForMutationOrReportError`, so every material
mutator gets it by construction. Niagara's is explicitly per-verb with a hazard clause, because 28 of
its 32 mutators must **not** take it. Merging would either drag Material's blanket behaviour onto
Niagara or push Niagara's clause machinery onto Material for no gain.

**What is worth extracting** is the lookup alone, into a shared
`PinWright::EditorOpen::FindOpenAssetEditor(UObject*)`, leaving each namespace's reject-and-explain
wrapper where it is. Small and mechanical — but it touches `MaterialFinders.h`, which is a live
conflict surface, so it wants its own change rather than riding along with a fix.

## History
- `#1-scoped-out-of-the-guard-work` `OPEN` reporter — The `B-niagara-editor-open-guard-missing-on-mutators`
  ticket proposed consolidating the two guards; the agent that classified all 32 verbs argued against
  merging the policy and for extracting only the lookup.
