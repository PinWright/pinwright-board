---
id: F-blueprint-diff-defaults
title: "No override-drift report: blueprint.inspect diffs a CDO only against its immediate parent, with no native baseline, no attribution and no multi-Blueprint sweep"
status: OPEN
severity: Low
category: feature
tags: [blueprint, cdo, defaults, drift, gap-analysis-2026-09-30]
encounters: 1
---

# No Blueprint override-drift report vs native defaults

**What exists today**
- `blueprint.inspect includeProperties` (`BlueprintInspectHandler.cpp:196-210`) calls `BuildClassPropertyJson(CDO, SuperClass CDO)`, which diffs against the **immediate parent** (`PropertyExport.cpp:1417-1530`, `PPF_DeepComparison`).
- Asset dumps write the same diff (`AssetDumpHandler.cpp:801`).
- `blueprint.scs.get` already diffs native subobjects and ICH records against the parent template (`PinWright_SCSHandlers.cpp:455-550`).

**What is missing**
- A native baseline.
- `introducedBy` attribution across intermediate Blueprints.
- One sweep report over many Blueprints.

`BuildSparsePropertyDiffJson` cannot be reused as-is: it rejects a superclass baseline (`PropertyDiff.cpp:22-25`). The only competitor with a yes is Monolith, `blueprint.audit_cdo_drift` (`FAuditAdapter.cpp:366-491`): a `/Game`-wide `ExportText` string compare against the nearest native class, with no components and no attribution, and likely noisy on object paths (inferred, not run).

**Fix:** a new verb `blueprint.diff_defaults`. It is a report, not a §18 audit: drift is not a defect, so there is no `pass`.
- Params: `assets[]` or `folder`, `baseline: native|parent` (required, §3), `includeComponents` (default true), `limit`/`offset`.
- Diff against the native CDO (`FBlueprintEditorUtils::FindFirstNativeClass`, `BlueprintEditorUtils.h:698`) with the existing deep-compare path, keeping only properties owned by the native chain.
- Attribute `introducedBy` by walking the Blueprint ancestors.
- Components: diff native default subobjects against the native CDO's same-named subobject, and ICH templates against `FindBestArchetype` (`InheritableComponentHandler.h:113`).
- Row: `{blueprint, property | component.field, ownerClass, baselineValue, value, introducedBy}`. Buckets `drifted/clean/unloadable` must sum.
- Errors: `NO_ASSETS_MATCHED`, `INVALID_ARGUMENT`. Read-only.
- Host: `BlueprintInspectHandler.cpp`, extracting the component half from `PinWright_SCSHandlers.cpp`.

**Acceptance:**
- A chain native → BP_A → BP_B with one value set in BP_A and one in BP_B returns both rows under `baseline: native`, each with the correct `introducedBy`. `baseline: parent` on BP_B returns only BP_B's row.
- An overridden component field appears.
- No false drift from object paths or instanced subobjects.

Effort S-M. Risk low.

## History
- `#1-gap-analysis` `OPEN` reporter — Filed from the 2026-09-30 competitor gap analysis (compare row "Blueprint override drift vs native defaults": PinWright partial, Monolith yes). Evidence and design above.
