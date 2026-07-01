---
id: E-asset-dump-oversized-fields
title: "Oversized dump files exceed LLM single-read budgets (impulse responses, batched mesh geom, packed-actor instance data)"
status: DONE
severity: Medium
category: ergonomic
tags: [asset-dump, properties-json, scs-json, niagara, size-cap]
---

# Oversized dump files exceed LLM single-read budgets

The asset-dump cache under `.editor-automation/asset-dumps/` is intended as the
preferred read surface for agents (CLAUDE.local.md says: "Prefer reading from
this cache over calling analysis MCP tools"). For a meaningful subset of assets,
the emitted sidecar files are too large for a single LLM `Read` to consume —
the data is there, but it is unreadable.

## Inventory (size >500 KB, current PDS sweep)

207 dump files exceed 500 KB. Breakdown by sidecar:

| Sidecar             | Files >500 KB | Notes                                                  |
|---------------------|---------------|--------------------------------------------------------|
| `niagara_graphs.json` | 92          | Pin/link/node arrays for complex Niagara systems       |
| `niagara_model.json`  | 47          | Niagara module/script model serialization              |
| `niagara_stack.json`  | 23          | Niagara emitter stack groups                           |
| `tree.xml`            | 20          | Widget hierarchy XML (legit deep widgets)              |
| `properties.json`     | 16          | **Primary focus of this ticket**                       |
| `scs.json`            | 6           | Packed-actor Simple Construction Scripts               |
| `cascade.json`        | 2           | Legacy particle systems                                |
| `bpir.txt`            | 1           | UDS Blueprint (legit huge BP, 10k lines)               |

## Top 5 offenders, characterised

1. **`Game/SportsStadium/Maps/Packed/BPP_BP_AllSeats/scs.json`** — 15.9 MB.
   Root cause: `InstancedStaticMesh.PerInstanceSMData` — every per-instance
   transform serialized as an ExportText string `(XPlane=...,YPlane=...,...)`
   under a single TArray. Thousands of seats × ~280-byte string per entry.

2. **`Game/SportsStadium/Maps/Packed/BPP_BP_SeatSplines/scs.json`** — 15.6 MB.
   Same shape as above.

3. **`Game/Audio/Impulses/1978_LargeRoom_IR/properties.json`** — 4.5 MB.
   Root cause: `ImpulseResponse` UPROPERTY is a raw `TArray<float>` of audio
   samples; each float is serialized as a JSON number with full mantissa
   (`3.0517578125e-05`). Identical shape across all 12 `IR_Reverb_*` and
   `1978_*` impulse assets (each 0.9–4.5 MB).

4. **`Game/UltraDynamicSky/Particles/Rain/niagara_graphs.json`** — 7.3 MB.
   Root cause: legitimate Niagara graph complexity — thousands of
   `links`/`nodes` entries with full GUID pin identifiers. Cross-cuts the rain,
   lightning, dust, snow, gunpad, impact, jumppad, explosion, character-FX
   subtrees (92 files).

5. **`Game/Maps/New_InfinityMap/SM_BatchedGrid/properties.json`** — 3.4 MB.
   Root cause: `BodySetup.AggGeom` — convex-element vertex/index data
   serialized as one giant ExportText string `(ConvexElems=((VertexData=...,
   IndexData=...,ElemBox=...),...))`. The whole physics geometry is in a
   single string value of one property.

The UDS `bpir.txt` (1.5 MB, 10 069 lines) is a legitimate dense BP graph —
not a bug, just a big asset. Agents needing it must already use targeted
`Read` offset/limit.

## Why this matters

`Read` rejects files past its token budget. With the 25 000-token cap currently
in effect, every file above ~700 KB is unreadable in one shot. The cache stops
being a drop-in replacement for live MCP analysis calls precisely on the
classes of asset where the live calls are slowest (impulse responses load
massive bulk data; packed actors enumerate thousands of instances).

## Existing infrastructure

`PropertyUtils.cpp:BuildClassPropertyJson` iterates all properties and emits
each via `ExportPropertyToJsonValueWithInheritance`. Skips are pluggable but
narrow:

- `IsTextureSourceNoisyProperty` — hard-coded `UTexture::Source` skip.
- `ShouldSuppressUnstableTransientReference` — transient-ref filter.

There is no size cap, no per-property byte budget, no known-huge-name skip
list, no chunking, no placeholder-on-overflow behaviour.

For `scs.json`, the `PerInstanceSMData` array is walked under
`BuildSparsePropertyDiffJson` / component property serialization — same lack
of any size guard.

## Proposed fix — pick the lowest-disruption option

**Recommendation: option (a) with a small allowlist of known-huge UPROPERTY
names, replaced inline with a typed placeholder.** Rationale below.

### (a) Known-huge-property skip with size-summary placeholder *(recommended)*

Add an exclusion list (analogous to the existing texture-source skip) keyed
by `OwnerClass + PropertyName`, e.g.:

- `UAudioImpulseResponse::ImpulseResponse`
- `UBodySetup::AggGeom` (or coarser: skip the convex/tri-mesh sub-arrays)
- `UInstancedStaticMeshComponent::PerInstanceSMData`

Replace the value with a structured placeholder so consumers still see the
property exists:

```json
"ImpulseResponse": {
  "type": "TArray",
  "$omitted": "binary-like data, 524288 elements, ~4.5 MB",
  "$reason": "exceeds-llm-budget"
}
```

Why this is lowest-disruption:

- Surgical, additive. One new helper next to `IsTextureSourceNoisyProperty`.
- Zero schema churn for all other dumps.
- Consumers parsing properties.json can detect `$omitted` if they need to
  fall back to a live MCP call (`asset.dump_property`, `property.read`, etc.).
- Re-uses the existing skip plumbing in `BuildClassPropertyJson` and the
  per-component path in `scs.json`.
- Captures the dominant offenders (impulse responses 12 files, batched grid
  1 file, packed-actor scs 6 files) — accounts for ~80 MB of the bloat.

### (b) Shard `properties.json` by top-level key

Emit `properties/<key>.json` files plus an index. Captures *every* big file
generically.

Tradeoffs against (a):
- Schema break — every consumer (this plugin's own tests, the wiki, external
  agents, the CLAUDE.local.md guidance) must learn the sharded layout.
- Doesn't help `scs.json` (single top-level structure with one huge nested
  field), `niagara_graphs.json` (single huge array), `bpir.txt` (not JSON).
- Many small files instead of one — more directory churn, harder to git-diff.
- Pays a complexity tax on the 99% of dumps that are already small.

### (c) Generic `--max-property-size` flag

Per-property byte cap during emission; over the cap, write the same
placeholder shape as (a).

Tradeoffs against (a):
- More principled but needs a serialization-cost measurement step (write to a
  scratch buffer, measure, decide to keep or discard) — duplicates work for
  every property, even tiny ones, or requires a two-pass emit.
- Risks accidentally truncating legitimate medium-large properties (a 40-row
  data-asset array that happens to be 600 KB) where the agent *wants* the
  data.
- No real-world tunable default — what byte threshold? Different agents have
  different read budgets. A name-based skip is unambiguous.

### Why not just stop dumping these assets entirely

Impulse responses and packed actors are valuable to dump for the *structural*
properties (`bIsEvenChannelCount`, `Mobility`, `ComponentTags`,
`InstancingRandomSeed`, etc.). The only field that needs to go is the one
unreadable payload. A full-asset skip throws out the good metadata too.

## Concrete next step

Land (a) first. Schema for the placeholder:

```json
{ "type": "<UE type>", "$omitted": "<one-line summary>", "$reason": "exceeds-llm-budget" }
```

Driver: a `TMap<TTuple<UClass*, FName>, FOmissionReason>` populated at module
init, queried in `BuildClassPropertyJson` between the texture-source check
and the property export.

If, after (a) lands, `niagara_graphs.json` remains a pain point, treat it as
a separate ticket — the root cause there is graph-fidelity, not a single
huge UPROPERTY, and it warrants its own design call.

## History
- `#1-initial-report` `OPEN` reporter — Surveyed `.editor-automation/asset-dumps/`: 207 files >500 KB; top properties.json offenders are `ImpulseResponse` (12 files, up to 4.5 MB) and `BodySetup.AggGeom` (SM_BatchedGrid, 3.4 MB); top scs.json offenders are `PerInstanceSMData` on packed-actor InstancedStaticMeshComponents (BPP_BP_AllSeats 15.9 MB, BPP_BP_SeatSplines 15.6 MB). Dumper at `PropertyUtils.cpp:BuildClassPropertyJson` has no size cap and no per-property exclusion beyond `UTexture::Source` and transient refs. Recommend a name-based known-huge skip list emitting a `{ "$omitted": "...", "$reason": "exceeds-llm-budget" }` placeholder — additive, no schema break for other dumps, captures ~80 MB of the bloat without sharding or per-byte truncation.
- `#2-add-oversized-skip-list` `IN-REVIEW` developer — Added `IsKnownOversizedProperty` + `BuildOmissionPlaceholder` helpers in `Utils/PropertyUtils.{h,cpp}` with initial known-huge entries for `UAudioImpulseResponse::ImpulseResponse`, `UBodySetup::AggGeom`, `UInstancedStaticMeshComponent::PerInstanceSMData`. Wired skip into `BuildClassPropertyJson` (between texture-source and transient-ref filters) and into `EditorAutomationRpcGateway_SCSHandlers.cpp::AddOverriddenProperties` for `scs.json` coverage. Emits structured placeholder `{type, $omitted, $reason: "exceeds-llm-budget", is_overridden_locally: true}`. Class lookup uses path-name strings to avoid hard Synthesis-plugin module dependency for `UAudioImpulseResponse`. Regression test `TestPropertyUtilsOversizedSkip.cpp::FPropertyUtilsOversizedSkipTest` creates a transient `UInstancedStaticMeshComponent` and asserts the placeholder shape on `PerInstanceSMData`. Counterfactual: reverting the new skip causes `PerInstanceSMData` to fall back to FArrayProperty ExportText emission — no `$omitted` key, assertion fails.
- `#3-verify-fix` `DONE` tester — Verified: re-dumped `/Game/Audio/Impulses/1978_LargeRoom_IR` via `asset.dump`; resulting `properties.json` dropped from 4.5 MB to 652 bytes. `ImpulseResponse` field now emits the exact placeholder shape `{"type": "TArray<float>", "$omitted": "audio samples, 258128 elements", "$reason": "exceeds-llm-budget"}` while structural fields (`bIsEvenChannelCount`, `NormalizationVolumeDb`, `NumChannels`, `SampleRate`) are preserved as recommended in the ticket.
