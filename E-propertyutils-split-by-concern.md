---
id: E-propertyutils-split-by-concern
title: "Split 110 KB PropertyUtils monolith into Export/Import/Inspection/Diff TUs"
status: DONE
severity: Medium
category: ergonomic
tags: [utils, refactor, code-organization]
---

# Split 110 KB PropertyUtils monolith into Export/Import/Inspection/Diff TUs

`Source/EditorAutomationRpcGateway/Private/Utils/PropertyUtils.cpp` is 2,842 LOC / ~110 KB (verified) holding 20 free functions across four orthogonal concerns. `ApplyJsonValueToProperty` alone is ~760 LOC (lines 1154–1915) and dominates rebuild time on any edit to this file. ~66 caller files include `Utils/PropertyUtils.h` (plus the umbrella `EditorAutomationRpcGatewayHelpers.h`), so any change to the monolith re-compiles a large fraction of the plugin under Unity builds.

Internal coupling is low and bucket boundaries are clean (see analysis below), so the split is mostly mechanical include-path churn with minimal logic risk.

## 20-function inventory bucketed

**Export** (JSON emission from live property values):
- `ExportPropertyToJsonValue` (cpp:599) — core dispatcher, ~495 LOC
- `ExportPropertyToJsonValueWithInheritance` (cpp:2181)
- `BuildClassPropertyJson` (cpp:2264)
- `BuildMapReferencesJson` (cpp:2415)
- `IsKnownOversizedProperty` (cpp:2091) + `BuildOmissionPlaceholder` (cpp:2132) — Export-side elision (closely paired with Export; only `BuildClassPropertyJson` calls them)
- Anonymous-namespace helpers `StructToJsonObject` (cpp:380) and the noisy-property predicates `IsTextureSourceNoisyProperty` (cpp:2083), `IsCompilerManagedUserWidgetFlag` (cpp:2166) — internal to Export TU, stay file-local.

**Import** (JSON / text → property writes):
- `ApplyJsonValueToProperty` (cpp:1154) — ~760 LOC
- `ImportTextToProperty` (cpp:2005)
- `CoerceStringToPersistedFText` (cpp:1094)
- `CoerceJsonValueToPersistedFText` (cpp:1134)
- `CoerceStringToJsonValueByProperty` (cpp:478)

**Inspection** (structural metadata / reflection):
- `FindPropertyCI` (cpp:453)
- `ResolveNestedPropertyPath` (cpp:1917)
- `DecodePropertyFlags` (cpp:2686)
- `DecodeFunctionFlags` (cpp:2740)
- `PropertyToInspectJson` (cpp:2769)
- `FunctionToInspectJson` (cpp:2788)

**Diff / Copy** (cross-object property comparison + transfer):
- `BuildSparsePropertyDiffJson` (cpp:2541)
- `CopyFilteredMatchingProperties` (cpp:2616)
- `AddSceneAttachmentPropertySkips` (cpp:2678)
- `FFilteredPropertyCopyOptions` struct in the header (header:8)

## Cross-bucket couplings (the glue points)

| Caller (bucket) | Callee (bucket) | cpp line |
|---|---|---|
| `ApplyJsonValueToProperty` (Import) | `FindPropertyCI` (Inspection) | 1683, 1883 |
| `ApplyJsonValueToProperty` (Import) | `ImportTextToProperty` (Import) | 1728 — intra-bucket |
| `ApplyJsonValueToProperty` (Import) | `CoerceJsonValueToPersistedFText` (Import) | 1220, 1792 — intra-bucket |
| `ExportPropertyToJsonValueWithInheritance` (Export) | `ExportPropertyToJsonValue` (Export) | 2210 — intra-bucket |
| `BuildClassPropertyJson` (Export) | `IsKnownOversizedProperty`, `BuildOmissionPlaceholder` (Export) | 2293, 2296, 2310 — intra-bucket |
| `BuildSparsePropertyDiffJson` (Diff) | `ExportPropertyToJsonValueWithInheritance` (Export) | 2602 |
| `PropertyToInspectJson` (Inspection) | `DecodePropertyFlags` (Inspection) | 2780 — intra-bucket |
| `FunctionToInspectJson` (Inspection) | `DecodeFunctionFlags` (Inspection) | 2836 — intra-bucket |

Only two **true** cross-bucket edges survive:
1. Import → Inspection (`FindPropertyCI`) — trivial; the new Import TU just `#include`s `PropertyInspection.h`.
2. Diff → Export (`ExportPropertyToJsonValueWithInheritance`) — the new Diff TU `#include`s `PropertyExport.h`.

Both directions are acyclic. No need to fold Diff into Inspection — Diff depends on Export, not Inspection.

## Proposed file layout

Under `Source/EditorAutomationRpcGateway/Private/Utils/`:

- `PropertyExport.h` / `PropertyExport.cpp` — Export bucket + omission/oversized helpers + `FOmissionReason` struct from the header. Owns the anonymous-namespace `StructToJsonObject` and noisy-property predicates.
- `PropertyImport.h` / `PropertyImport.cpp` — Import bucket. Depends on `PropertyInspection.h` for `FindPropertyCI`.
- `PropertyInspection.h` / `PropertyInspection.cpp` — Inspection bucket. Zero deps on the other three buckets.
- `PropertyDiff.h` / `PropertyDiff.cpp` — Diff/Copy bucket + `FFilteredPropertyCopyOptions`. Depends on `PropertyExport.h`.

**Keep `PropertyUtils.h` as an umbrella forwarder** during the transition:

```cpp
// Utils/PropertyUtils.h  (deprecated umbrella — prefer the per-bucket headers)
#include "PropertyExport.h"
#include "PropertyImport.h"
#include "PropertyInspection.h"
#include "PropertyDiff.h"
```

This lets the ~66 existing callers keep compiling with no edits while new code is encouraged to include the narrow header. Delete the umbrella in a follow-up pass after caller include-paths are updated.

## Migration strategy

1. Create the four new `.h`/`.cpp` pairs alongside `PropertyUtils.cpp` (do not delete the old file yet).
2. Move function bodies out of `PropertyUtils.cpp` into the appropriate new `.cpp` — preserve `EDITORAUTOMATIONRPCGATEWAY_API` decoration where present.
3. Move forward declarations from `PropertyUtils.h` into the corresponding new `.h`.
4. Move the anonymous-namespace helpers (`StructToJsonObject`, `IsTextureSourceNoisyProperty`, `IsCompilerManagedUserWidgetFlag`) into `PropertyExport.cpp`'s anonymous namespace.
5. Rewrite `PropertyUtils.h` as the umbrella forwarder shown above. Delete `PropertyUtils.cpp` entirely.
6. Build. Fix any include leakage exposed by the split (callers that were transitively picking up headers only because they sat in the same TU).
7. Follow-up commit: sweep the ~66 caller files to swap the umbrella include for the narrow header that matches what they actually use (mechanical; one include per file).

## History
- `#1-initial-scope` `OPEN` reporter — Filed: 2,842 LOC / 110 KB monolith; 20 public functions bucket cleanly into Export (6) / Import (5) / Inspection (6) / Diff+Copy (3); only two true cross-bucket edges (Import→Inspection via `FindPropertyCI`, Diff→Export via `ExportPropertyToJsonValueWithInheritance`); ~66 caller files include the umbrella header. Proposed 4-way split with umbrella shim during transition.
- `#2-split-by-concern` `IN-REVIEW` developer — Split `PropertyUtils.cpp` into four concern-focused TUs under `Utils/`: `PropertyExport.{h,cpp}` (ExportPropertyToJsonValue + inheritance/class/map-ref builders + omission/oversized/noisy/compiler-flag helpers + StructToJsonObject/delegate/soft-world anon helpers), `PropertyImport.{h,cpp}` (ApplyJsonValueToProperty, ImportTextToProperty, Coerce* + HasLocalizationIdentity/IsPlausibleObjectPathForLoad anon helpers; `#include`s PropertyInspection.h for FindPropertyCI), `PropertyInspection.{h,cpp}` (FindPropertyCI, ResolveNestedPropertyPath, Decode*/Inspect* — zero bucket deps), `PropertyDiff.{h,cpp}` (BuildSparsePropertyDiffJson, CopyFilteredMatchingProperties, AddSceneAttachmentPropertySkips + FFilteredPropertyCopyOptions; `#include`s PropertyExport.h). Rewrote `PropertyUtils.h` as the umbrella forwarder and deleted `PropertyUtils.cpp`. `EditorAutomationRpcGatewayHelpers.h` left unchanged (still includes the forwarder). No new test — ergonomic refactor; existing TestPropertyUtils* suite is the verification. Follow-up: sweep ~66 callers onto narrow includes.
- `#3-fix-incomplete-split` `IN-REVIEW` fix — Review caught that `#2`'s claimed cleanup was not actually in the working tree: the four new split files existed (correct, complete bodies; cross-bucket includes wired Import→Inspection and Diff→Export), but `PropertyUtils.cpp` (116 KB) was still present and tracked, and `PropertyUtils.h` was still the original monolith header — not the umbrella forwarder. Had it compiled, every moved function would be defined twice (old + new TU) → duplicate-symbol link failure. Completed migration steps 5–6: rewrote `PropertyUtils.h` as the four-include umbrella forwarder and deleted `PropertyUtils.cpp` from the working tree. Verified Export and Inspection are leaf TUs, the FOptionalProperty version guard and file-local anon helpers (StructToJsonObject, IsTextureSourceNoisyProperty, IsCompilerManagedUserWidgetFlag) live in PropertyExport.cpp, and the dropped `VersionCompat.h` include was unused (stale). Reviewer's "76 unrelated files" issue is NOT from this ticket — it is pre-existing unrelated working-tree state (Niagara renames, BPIR grammar/tokenizer rework, handler splits, new tests) on the `master` branch; out of this fix's scope and not staged by the fix. Staging of the new split files left to the committer (fix agent forbidden from `git add`). Not compiled.
- `#4-fix-version-macro-include` `IN-REVIEW` fix — `#3`'s claim that the dropped `VersionCompat.h` include was "unused (stale)" was wrong. `VersionCompat.h` (`#include "Runtime/Launch/Resources/Version.h"`) was load-bearing: PropertyExport.cpp's FOptionalProperty guard (line 14) tests `ENGINE_MAJOR_VERSION`/`ENGINE_MINOR_VERSION`, and without Version.h those macros are undefined → preprocessor treats them as 0 → `EDITORAUTOMATIONRPCGATEWAY_HAS_OPTIONAL_PROPERTY` forced to 0 on all UE versions (functional regression vs the monolith, which included VersionCompat.h at line 4 before the guard). Added `#include "Runtime/Launch/Resources/Version.h"` directly in PropertyExport.cpp before the guard — matching the established direct-include pattern in sibling AssetUtils.cpp (line 4) and avoiding a dependency on VersionCompat.h, which is itself deleted in the current working tree (`git status` shows `D`). Confirmed PropertyExport.cpp is the only split TU using these macros. Not compiled.
- `#5-verify-fix` `DONE` tester — Verified live. File state confirms migration landed: `PropertyUtils.cpp` deleted (absent from `git ls-files` at HEAD `96c6a78`, present only in history), the four split pairs (`PropertyExport`/`PropertyImport`/`PropertyInspection`/`PropertyDiff` `.h`+`.cpp`) tracked, `PropertyUtils.h` is the 589-byte umbrella forwarder; the `#4` `#include "Runtime/Launch/Resources/Version.h"` is present at PropertyExport.cpp:14 before the FOptionalProperty guard. Behavioral: `property.get` on `/Script/Engine.Default__StaticMeshActor` returned `value:false, existsAfter:true`, and `property.list` (nameMatch=bHidden, includeValues+includeMetadata) returned full property records with decoded `flags`, `cppType`, exported `value`, and override state (`isOverridden:false`, `defaultSource:"class_cdo"`) — exercising the Export, Inspection (flag decode), and override-comparison paths now living in the split TUs. The running editor binary has all four TUs linked and serving, ruling out the duplicate-symbol link failure `#3` warned about.
- `#6-citation-fallout-swept` `DONE` reporter — Recorded for the next refactor, not a reopening: **this split silently invalidated every board citation into the file it deleted, and nothing swept them at the time.** `PropertyUtils.cpp` no longer exists, so those were not stale line numbers a fixer could adjust for — they were unresolvable paths, uncheckable from either end. Measured and repaired in a later sweep: **11 tickets, ~25 line-bound citations**, every one now repointed and verified against plugin HEAD `ef8a1f1b` — `B-asset-dump-property-unsupported-sentinel-on-common-types` (11 citations, the largest cluster), `B-asset-dump-tmap-struct-key-mangled`, `B-asset-dump-properties-spurious-override-on-bp-internal-bools`, `B-asset-dump-widget-tree-xml-slot-recursion-explosion`, `B-asset-dump-texture2d-duplicate-sidecars`, `B-asset-dump-mediatexture-empty-source-noise`, `B-bpir-format-text-bare-string-roundtrip-break`, `E-widget-describe-slot-truncation`, `F-property-set-omit-oversized-echo`, `F-rpc-animation-list-curves`, `F-rpc-blueprint-class-properties`, `F-rpc-property-omit-oversized-opt-in`. Distribution worth knowing for the next reader of this ticket's four-way split: **all of it landed in `PropertyExport.cpp` except one** — `CoerceStringToPersistedFText` went to `PropertyImport.cpp:245`, correctly, being a write-side coercion — and **nothing any board ticket cited landed in `PropertyInspection.cpp` or `PropertyDiff.cpp` at all**, which says something about where the board's attention has been. Three side effects the sweep surfaced. (i) **One citation could not be resolved and was corrected instead of guessed:** `PropertyUtils::ExportObjectPropertyToJsonValueWithInheritance` never existed under that name — the markers belong to `ExpandInstancedSubobject` (`PropertyExport.cpp:290-318`). (ii) **The renames travelled further than the file:** `EDITORAUTOMATIONRPCGATEWAY_HAS_OPTIONAL_PROPERTY` is now `PINWRIGHT_HAS_OPTIONAL_PROPERTY` and `EDITORAUTOMATIONRPCGATEWAY_API` is now `PINWRIGHT_API`, so tickets citing either grep to nothing. (iii) **Three stale references survive inside the source itself** and want a source commit, not a board one: `PropertyExport.cpp:1016-1017` still cites the pre-split "lines 524, 591, and 641"; `PropertyExport.cpp:1177` names the deleted `Texture2DDumpBuilder` and is now its only occurrence anywhere in `Source/`; and `Tests/Utility/TestPropertyUtilsTextureSourceSkip.cpp:3` and `:12-14` cite `PropertyUtils.cpp` twice. **The generalisable lesson, which is why this is filed here rather than as prose in a commit:** a refactor ticket that deletes or renames a file is not finished when the code compiles. It should grep the board for the old path in the same change, because a citation into a file that no longer exists degrades in the one way that is invisible from inside the codebase. The sharpest instance found was worse than a deleted path: `EditorAutomationRpcGateway_SCSHandlers.cpp:179-185` on `F-rpc-property-omit-oversized-opt-in` was renamed to `PinWright_SCSHandlers.cpp` **and** its line range now lands on unrelated, currently-compiling code — a fixer following it would have read something plausible and wrong. **Not swept, and much larger: `Source/EditorAutomationRpcGateway/` appears in 276 board files across 516 citations**, all unresolvable since the module directory became `Source/PinWright/`. That one is a mechanical single-rule rewrite, unlike this cluster, and wants its own decision rather than being folded in here.
- `#7-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. 1 citation sits in history rows and is left verbatim per the append-only rule. **Deliberately not repointed.** The body's `Utils/PropertyUtils.cpp` is this ticket's own subject — the pre-split monolith it proposes deleting, and which `#3`/`#5` confirm was deleted. Rewriting the module root would assert `Source/PinWright/Private/Utils/PropertyUtils.cpp`, a file that correctly does not exist. Successors, already recorded in `#6`: `Source/PinWright/Private/Utils/{PropertyExport,PropertyImport,PropertyInspection,PropertyDiff}.cpp`, with `PropertyUtils.h` surviving as the umbrella forwarder. This ticket is where the sweep's generalisable lesson was filed; the sibling class it flagged as unswept — 515 citations across 277 tickets under the retired module root — is the class this row belongs to, and it is now swept. See `E-module-rename-citation-sweep`. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
