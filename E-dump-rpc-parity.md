---
id: E-dump-rpc-parity
title: "Asset-dump sidecars and live-read RPCs lack parity across multiple asset kinds"
status: DONE
severity: High
category: ergonomic
tags: [asset-dump, rpc-parity, hygiene, builder-consolidation, audit]
---

# Asset-dump sidecars and live-read RPCs lack parity across multiple asset kinds

## Dual-surface principle

Any structured asset-state output the plugin produces should be available
through **two** surfaces backed by **one** builder function:

1. **Asset-dump sidecar pipeline** — `asset.dump` / `asset.dump_folder`
   writes a per-asset JSON/text file (e.g. `static_mesh.json`,
   `cascade.json`) into the dump tree.
2. **Live-read RPC** — a `<asset_type>.describe` (or equivalent) handler
   returns the same shape synchronously without touching disk.

When these drift, agents that have ingested the asset-dump tree see a
different model of the asset than agents that hit the live RPC, and
neither knows which is authoritative. The fix is mechanical: one builder
function, two call sites.

## Known gap surface (verified against current code)

Confirmed by reading `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetDumpHandler.h`
(`namespace DumpFileNames`) and grepping `docs/rpc-method-reference.generated.md`
for `describe`:

Priority axis is **read demand**, not mutation frequency — agents read these
sidecars constantly during gameplay setup, material tuning, performance
audits, and content QA, even when the underlying asset rarely changes.

| Sidecar filename | Dump builder exists | Live-read RPC | Gap | Priority |
|------------------|---------------------|---------------|-----|----------|
| `data_table.json` | yes (`DataTableDumpBuilder`) | none | **gap** | **P1** — gameplay data lookup, weapon stats, loot tables |
| `material_instance.json` | yes (in `BuildAllFilesForAsset` dispatch) | none | **gap** | **P1** — material tuning is one of the most common Fab-buyer operations; param overrides vs parent state queried constantly |
| `static_mesh.json` | yes | none | **gap** | **P1/P2** — bounds, vert/tri count, LOD, collision, Nanite status; common in level setup, optimization audits, content QA |
| `anim_sequence.json` | yes (`AnimSequenceDumpBuilder`) | none generic (`E-rpc-animation-extend-get-animation-info`, `F-rpc-animation-list-*` are aspect-specific) | **gap** | **P2** — length, frame count, curves, notifies in anim setup workflows |
| `sound_wave.json` | yes | none | **gap** (`F-sound-wave-property-edit` adds writers, not a reader) | **P2** — format, sample rate, duration, looping, attenuation defaults |
| `user_defined_struct.json` | yes (`UserDefinedStructDumpBuilder`) | none generic (`F-rpc-blueprint-describe-struct` is the proposed reader) | **gap, tracked elsewhere** | **P3** — close once `F-rpc-blueprint-describe-struct` lands |
| `level_sequence.json` | yes | none generic (`F-rpc-sequencer-list-sections` / `F-rpc-sequencer-get-camera-cut-track` are partial) | **gap, partial coverage** | **P3** — narrow remaining gap after sequencer tickets |
| `cascade.json` | yes | none | **gap** | **P3** — legacy/deprecated since UE 4.20 in favor of Niagara |
| `agir.txt` | yes | `anim.decompile_agir`-style RPC TBD | **needs audit** | TBD |

Confirmed parity (no work needed):

| Sidecar filename | Live-read RPC |
|------------------|---------------|
| `sound_cue.json` | `audio.authoring.describe_sound_cue` |
| `metasound.json` | `audio.authoring.describe_metasound` |
| `niagara_model.json` | `niagara.decompile_model` (related: `niagara_system/emitters/parameters/stack/graphs/compile` may need their own audit) |
| `tree.xml` | `widget.export_xml` |
| `widget_animations.json` | `widget.export_animations_json` |
| `scs.json` | `blueprint.scs.get` |
| `texture.json` | `texture.describe` (covers any `UTexture` subclass — gap closed already) |
| `skeletal_mesh.json` | `skeleton.describe_mesh` (verify shape match) |
| `mgir.txt` | `material.decompile_mgir`-style (verify) |
| `bpir.txt` | `blueprint.decompile_bpir` |

The initial gap list circulated in the task brief listed
texture/texture_2d as a gap; that turned out to be already closed by
`texture.describe`. The corrected gap list above is what this audit
should enforce.

## Out of scope

- **Mutators.** `<asset_type>.set_*` / `<asset_type>.create_*` handlers
  are tracked by their own `F-` tickets and are not in this parity
  matrix. This ticket is read-side only.
- **File-shaped artifacts.** `preview.png` (widget screenshot) is a
  PNG blob; no JSON-shaped live-read makes sense.
- **Generic catch-alls.** `meta.json` and `properties.json` are
  produced by `asset.dump` itself with no per-type builder; the live
  equivalents are the existing `asset.*` introspection RPCs
  (`asset.get_properties`, `asset.get_metadata`, etc.). No new RPC
  needed.
- **Level-only artifacts.** `world_settings.json`, `level_bp.txt`,
  `sublevels.json`, `actors/manifest.json` describe a `UWorld` — the
  live surface is the `world.*` / `level.*` namespaces, which already
  cover these aspects through multiple RPCs. Parity here means *audit
  the world namespace covers each sidecar field*, not "add one new RPC
  per file." Tracked under `F-dump-world-metadata` already.

## Proposed work

1. **Audit pass.** Produce a parity matrix in this ticket (or in
   `docs/asset-dump-parity.md`) cross-referencing every entry in
   `DumpFileNames` against every `*.describe` / `*.dump` / `*.get_*` RPC
   in the catalog. Mark each row: PARITY / GAP / OUT-OF-SCOPE. The
   matrix above is the seed.
2. **Per gap: add a live-read RPC** following the
   `<asset_type>.describe` naming convention. Example targets:
   - `data_table.describe` (uses the same builder as `DataTableDumpBuilder`)
   - `mesh.describe_static` (parallels `skeleton.describe_mesh`)
   - `cascade.describe`
   - `material.describe_instance`
   - `audio.describe_sound_wave`
   - `animation.describe_sequence`
3. **Builder consolidation.** Where the dump-builder code currently
   lives as a static helper inside `AssetDumpBuilder.cpp` or a private
   TU, lift the function into a public header (`Public/Handlers/.../<X>Builder.h`)
   exporting a single `BuildXxxJson(UObject* Asset) → TSharedPtr<FJsonObject>`
   entry point. Both the dump pipeline and the RPC handler must call
   the **same** function. No copy-pasted "near-duplicate" builders.
4. **Test parity.** For each new RPC, add a single test that asserts
   the RPC response shape equals the corresponding sidecar shape for a
   fixture asset. This is the regression net against future drift.

## Effort

Medium audit. Each individual RPC add is ~30–50 LoC (handler boilerplate
+ delegating to the existing builder). Cumulative ~5–8 new RPCs across
the current gap surface. The audit and builder lift is the bulk of the
work, not the RPC handlers themselves.

## Priority

Varies per gap: **P1 to P3 — see matrix**. Two P1 gaps (`data_table`,
`material_instance`) drive the ticket-level severity to High. The P3
items (cascade, partial-coverage entries) are hygiene only. Not blocking
active IR work, but the P1 gaps materially affect everyday agent work
(gameplay data reads, material tuning) and shouldn't sit behind lower-
priority work.

## Cross-references

- `R-asset-dump-sidecar-registry` — the sidecar registry pattern is the
  natural place to enforce parity going forward: each registry entry
  should declare both a dump producer and a live-read RPC method name,
  and a CI check can fail the build if one is missing.
- `F-ir-authoring-guide` — already mandates the IR-specific case (every
  IR text sidecar like `bpir.txt`, `mgir.txt`, `agir.txt` has a
  corresponding `.decompile_*` and `.compile_*` RPC).
- `F-data-table-row-authoring` — adds write-side RPCs for data tables;
  this ticket backfills the missing read-side (`data_table.describe`)
  to close the round-trip surface.
- `F-rpc-blueprint-describe-struct` — partial coverage of the
  `user_defined_struct.json` gap; this ticket subsumes that surface
  into the broader parity matrix.
- `F-rpc-sequencer-list-sections`, `F-rpc-sequencer-get-camera-cut-track`
  — partial coverage of `level_sequence.json`; audit whether a single
  `sequencer.describe` is preferable to many aspect-specific RPCs.
- `F-rpc-mesh-describe-skeletal`, `F-rpc-texture-describe-generic` —
  these are the templates for the new RPCs proposed here; reuse their
  handler shape.

## History
- `#1-initial-audit` `OPEN` reporter — Filed parity audit. Initial brief listed texture/texture_2d as a gap, but `texture.describe` already covers any `UTexture` subclass — corrected the gap list against `DumpFileNames` and `docs/rpc-method-reference.generated.md`. Confirmed gaps: `data_table`, `static_mesh`, `cascade`, `material_instance`, `sound_wave`, `anim_sequence`. Partial: `level_sequence`, `user_defined_struct`. Proposed builder consolidation (single shared function per asset kind) plus per-gap `<asset_type>.describe` RPCs, with the sidecar registry (`R-asset-dump-sidecar-registry`) as the long-term enforcement mechanism.
- `#2-priority-correction` `OPEN` reporter — User-driven correction during ticket review. Framing error: original per-gap priorities used **mutation frequency** as the axis ("seldom-edited → low priority"), but parity is about **read access**, not writes. Agents read these sidecars constantly during gameplay setup, material tuning, perf audits, and content QA — independent of how often the underlying asset is mutated. Corrected per-gap priorities: **P1** — `data_table` (gameplay data lookup, weapon stats, loot tables), `material_instance` (one of the most common Fab-buyer operations; param overrides vs parent state queried constantly); **P1/P2** — `static_mesh` (bounds, vert/tri count, LOD, collision, Nanite status); **P2** — `anim_sequence` (length, frame count, curves, notifies), `sound_wave` (format, sample rate, duration, looping, attenuation); **P3** — `user_defined_struct` (mostly covered by `F-rpc-blueprint-describe-struct`), `level_sequence` (partial coverage by sequencer tickets), `cascade` (legacy/deprecated since UE 4.20). Also re-confirmed `texture.describe` already exists (verified at `rpc-method-reference.generated.md:10751`) — texture/texture_2d is **NOT a gap** and is parked in the confirmed-parity table. Ticket-level `severity` bumped Medium → High on the strength of the two P1 gaps; "Priority" section rewritten to "varies per gap — see matrix"; matrix re-sorted by priority with a new Priority column.
- `#3-four-describe-rpcs-shipped` `IN-REVIEW` developer — Shipped four new live-read describe RPCs closing the P1/P2 parity gaps, each delegating to the existing exported `*DumpBuilder::Build*Json(...)` function (no copy-pasted builder logic — the dump pipeline and the new RPC share one function per asset kind, as the ticket mandates). Methods + handler files + regression tests: (1) `data_table.describe` in `Source/EditorAutomationRpcGateway/Private/Handlers/DataTable/DataTableDescribeHandler.cpp` (test `Tests/Assets/TestDataTableDescribeHandler.cpp`); (2) `static_mesh.describe` in `Handlers/Asset/StaticMeshDescribeHandler.cpp` (test `Tests/Assets/TestStaticMeshDescribeHandler.cpp`) — named to parallel the existing `texture.describe` asset-type-namespace precedent and the `static_mesh.json` sidecar filename, not `mesh.describe_static`; (3) `audio.authoring.describe_sound_wave` appended to `Handlers/Audio/AudioAuthoringHandler.cpp` next to the sibling `describe_sound_cue`, reusing the file-local `LoadSoundWaveFromPath` helper for path normalization (test `Tests/Media/TestSoundWaveDescribeHandler.cpp`); (4) `animation.describe_sequence` in `Handlers/Animation/AnimSequenceDescribeHandler.cpp` (test `Tests/Assets/TestAnimSequenceDescribeHandler.cpp`). Each test creates a transient asset, invokes the RPC via the dispatcher (`InvokeHandlerWithCapture`), and asserts the documented dump-shape fields are present — counterfactual is "remove the `REGISTER_RPC_HANDLER` block → `bFound` assertion fails". **Material instance and cascade explicitly deferred / out-of-scope of this commit:** no `material_instance.json` sidecar exists in `DumpFileNames` and no `MaterialInstanceDumpBuilder` exists either (the ticket's matrix overstated this gap; reality is `BuildAllFilesForAsset` has no `UMaterialInstance` branch — it falls through to the default tail emitting only `properties.json`). The read surface for material instances is substantively covered by the existing `material.authoring.get_material_instance_info` (parent + per-type overrides + inherited defaults + parameter metadata). Cascade remains the P3 legacy gap and is left for a follow-up if demand warrants. Sprint plan persisted at `.claude/plans/mcp-sprint-20260514-E-dump-rpc-parity.md`; N+3 reviewer battery (4 spec + reuse + quality + efficiency) ran clean after a post-review rename of T2 (`mesh.describe_static` → `static_mesh.describe`) and a comment-hygiene pass on T4 to align with the plugin's no-WHAT-comments rule.
- `#4-verify-describe-rpcs` `DONE` tester — Verified: `data_table.describe` on `/Game/ContextEffects/DT_AnimEffectTags.DT_AnimEffectTags` returned `rowStruct`, `rowStructName`, `rowCount`, and `rows`; `static_mesh.describe` on `/Game/UltraDynamicSky/Meshes/Brush_Cube.Brush_Cube` returned `bounds`, `materials`, `lods`, `trianglesByLod`, and `verticesByLod`; `audio.authoring.describe_sound_wave` on `/Game/Audio/FPV_SOUND/UI/UISFX_Select_2.UISFX_Select_2` returned `duration`, `numChannels`, `sampleRate`, `bLooping`, `soundGroup`, `volume`, and `pitch`; `animation.describe_sequence` on `/Game/Characters/Heroes/Mannequin/Animations/Locomotion/Rifle/MM_Rifle_Idle_ADS.MM_Rifle_Idle_ADS` returned `assetKind: AnimSequence`, `path`, `lengthSeconds`, `frameRate`, `numFrames`, `notifies`, `curves`, and `syncMarkers`.
