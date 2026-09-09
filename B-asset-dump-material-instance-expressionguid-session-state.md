---
id: B-asset-dump-material-instance-expressionguid-session-state
title: "Material-instance parameter dumps carried ExpressionGUID, which serializes all-zero and only resolves once an editor session reconciles the instance against its parent — dump content that records the session, not the package"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset-dump, properties, material-instance, expressionguid, nondeterminism, mirror-churn, regression]
encounters: 1
lastSeen: 2026-09-09T10:00:00Z
---

# A per-parameter GUID that means "what this editor session happened to do"

`properties.json` for material instances emitted `ExpressionGUID` for every scalar,
vector and texture parameter. The value is the cached link from the instance's parameter
to the expression node in the **parent material**, and it is:

- **all zeros as serialized on disk**, until something reconciles the instance against its
  parent — opening the material editor, `material.update_material_instance`, a parent
  change;
- **resolved to a real GUID afterwards**, for as long as that editor session lasts.

So whichever value a sweep captured recorded the state of the editor that ran it, not the
content of the package. Two effects: an agent reading the mirror sees `00000000-...` and
cannot tell "no link" from "not yet reconciled", and the committed mirror churns whenever
a sweep happens to run after something touched the instance.

The parameter's durable identity is `ParameterInfo.Name`, which is dumped and is what any
consumer should match on.

**Fix (shipped, plugin `fa755a4f`):** `ExpressionGUID` is dropped from the dump as
non-semantic; `ParameterInfo.Name` stays. Aspect version `properties.json 7->8`
(dump core `2->3` in the same commit).

Prior art on this field: `B-asset-dump-struct-array-as-export-text-strings` (DONE) is what
made `ExpressionGUID` a visible JSON field in the first place, by expanding
`TextureParameterValues` / `ScalarParameterValues` struct-array elements into objects.
That expansion is correct and stays; only this member is removed.

## History
- `#1-guid-is-session-state` `OPEN` reporter — Observed on `X:\src\unreal\unreal-fpv-dev` (UE 5.8) while reviewing material-instance dumps in `asset-dumps/App`: `ExpressionGUID` read all-zero across the mirror, and re-dumping an instance after touching it in the editor produced a different, non-zero value for the same on-disk package. The field therefore reports editor session state rather than package content, and is both misleading to read and a source of mirror churn. Became visible as a field through `B-asset-dump-struct-array-as-export-text-strings` (DONE), whose struct-array expansion is not itself in question.
- `#2-dropped-as-non-semantic` `IN-REVIEW` developer — Fixed in plugin commit `fa755a4f` ("Fix three asset-dump regressions in property and bpir export"): the per-parameter `ExpressionGUID` is no longer dumped, on the reasoning that it is a cached parent-material link rather than authored data; `ParameterInfo.Name` carries the durable identity. Aspect version `properties.json 7->8`. **Verified** by re-sweeping `asset-dumps/App` with this build (8,388 / 8,388 assets, 0 skips): `ExpressionGUID` is gone from every dump. Awaiting an independent tester; the alternative design — keep the field but reconcile the instance before reading it, so the dump carries the resolved GUID deterministically — was rejected rather than tested, because reconciling would dirty the package during a read-only sweep.
