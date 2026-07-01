---
id: B-material-instance-missing-quality-switch-params
title: "material_instance.json missing quality-level parameter overrides (clarification needed)"
status: WONTFIX
severity: Low
category: bug
tags: [material-instance, sidecar, clarification]
---

# material_instance.json missing quality-level parameter overrides

Reporter suggests adding per-quality-level parameter overrides to material_instance.json, citing `MIC->GetMaterialQualityLevelOverrides()` (or equivalent UE 5.6+ API). Investigation found:

1. No reference to `GetMaterialQualityLevelOverrides()` in plugin codebase, UE material headers, or public documentation.
2. Quality switches are shader-compilation permutations defined in UMaterial, not instance-overrideable data structures.
3. Material instances capture value overrides (scalar/vector/texture/static-switch/static-component-mask) but not per-quality-level conditional overrides.

The material_instance.json already captures all authorable instance-level overrides. Quality-level variants (if used) are shader permutations, resolved at runtime by the engine, not stored on the instance.

## History
- `#1-initial-repro` `OPEN` reporter — per-platform LOD switches missing from dump
- `#2-clarification-needed` `WONTFIX` developer — investigated API surface; no instance-level quality override API found. Quality switches are parent-material shader permutations, not instance overrides. material_instance.json captures all instance-authorable overrides. Close as clarification-needed / not-applicable-to-instances.
- `#3-spec-review-confirmed` `WONTFIX` reviewer — Re-confirmed WONTFIX during spec review. Re-checked UE 5.4–5.7 material instance headers: no `GetMaterialQualityLevelOverrides()` or any instance-level per-quality-level parameter-override structure exists. Quality switches remain UMaterial shader-compilation permutations resolved at runtime, not instance-overrideable data. material_instance.json already captures every instance-authorable override (scalar/vector/texture/static-switch/static-component-mask). No code change implements quality-level introspection because no such introspection is possible. NOTE on the diff: the `REGISTER_DUMP_JSON_SIDECAR` call (lines 326–341) is NOT the resolution to this ticket — it is general framework infrastructure from the registry-driven dump dispatch refactor (`E-asset-dump-registry-driven-dispatch`), applied uniformly across all 18 dump builders. Reverting it here is not an option: it would desync MaterialInstanceDumpBuilder from the shared registry dispatch and break the build. The sidecar registration is orthogonal to the quality-level request; this ticket stays WONTFIX on the merits of the API investigation.
- `#4-diff-ownership-clarified` `WONTFIX` developer — Addressed review issue: the `REGISTER_DUMP_JSON_SIDECAR` change in `MaterialInstanceDumpBuilder.cpp` is **not B's diff**. It is one of the 17 sidecar registrations introduced by `E-asset-dump-registry-driven-dispatch` (`#2-registry-implemented`, which explicitly lists `MaterialInstance` among the emitted records) and is load-bearing for E's refactor: E already deleted the inline MIC branch from `AssetDumpHandler.cpp::BuildAllFilesForAsset`, so the registration is what now routes `material_instance.json`. That change is owned by, attributed to, and verified under E — not B. B's WONTFIX resolution is documentation-only and touches no code: the sole file in B's footprint is `Docs/board/B-material-instance-missing-quality-switch-params.md`. The `#3` note's framing (treating the registration as part of "the diff" for this ticket) overstated B's scope; the code belongs to E and stays there. WONTFIX stands on the API-investigation merits: no instance-level quality-override API exists.
