---
id: F-metasound-no-patch-or-preset
title: "Cannot create UMetaSoundPatch or MetaSound presets"
status: DONE
severity: Medium
category: feature
tags: [audio, metasound, authoring, patch, preset, no-text-ir]
---

# Cannot create UMetaSoundPatch or MetaSound presets

`audio.authoring.create_metasound` hard-codes `UMetaSoundSourceFactory`
and only ever produces a `UMetaSoundSource`. There is no
`create_metasound_patch` and no `create_metasound_preset` RPC. Verified
— both names return `Not found` from the gateway. The full MetaSound
authoring surface is the seven create/add_node/connect/add_input/
add_output/set_default/describe handlers.

**Why this matters.** MetaSound has three asset classes that pair with
distinct use cases:

- **`UMetaSoundSource`** — playable, addressed as a `USoundBase`,
  routes through submixes, takes attenuation/sound class. Today's only
  authorable type.
- **`UMetaSoundPatch`** — reusable graph fragment imported into other
  MetaSounds as a sub-graph. **Cannot be authored at all** via the
  imperative API. Has no text IR fallback.
- **Preset of a MetaSoundSource** — an asset that references another
  source and only overrides input defaults. The primary mechanism for
  "same procedural sound, different parameters" workflows (engine
  variants, ambience variants, weapon variants). **Cannot be created
  at all** via the imperative API.

The lack of patch support means agents cannot author reusable building
blocks. The lack of preset support means a project that has, say,
30 vehicle engine sounds derived from one master MetaSound has to
hand-build presets in the editor — every one.

**Verification:**
- `audio.authoring.create_metasound_patch` -> Not found
- `audio.authoring.create_metasound_preset` -> Not found
- `AudioAuthoringHandler.cpp:617` shows `create_metasound` is hard-wired
  to `UMetaSoundSourceFactory::FactoryCreateNew(UMetaSoundSource::...)`
  with no asset-class parameter.

**Implementation surface:**

For patches, `UMetaSoundPatchFactory` is the editor factory analogue to
`UMetaSoundSourceFactory`. `create_metasound` could grow an
`assetClass?: "source" | "patch"` parameter (default `source`) and
branch on factory. Both factories construct an
`IMetaSoundDocumentInterface`, so all downstream ops
(`add_metasound_node`, `connect_metasound_nodes`, etc.) Just Work once
the asset exists.

For presets, the underlying API is
`UMetaSoundFactory::CreatePresetFromReferenced(UMetaSoundSource*
ReferencedSource)` which produces a preset asset. The preset only
exposes `set_metasound_default` style overrides — the existing handler
already supports the literal types needed.

Proposed RPCs:

```
// Option A: extend create_metasound
audio.authoring.create_metasound {
    name, path?, save?,
    assetClass?: "source" | "patch"   // NEW; default "source"
}

// Option B: dedicated handler (more discoverable)
audio.authoring.create_metasound_patch { name, path?, save? }

// Either way, presets need their own handler:
audio.authoring.create_metasound_preset {
    name, path?, save?,
    referencedSource: assetPath,        // base MetaSoundSource to wrap
    overrides?: { [inputName]: literal }  // optional initial input overrides
}
```

## History
- `#1-no-patch-or-preset` `OPEN` reporter — Verified: `audio.authoring.create_metasound_patch` and `create_metasound_preset` both return Not found. `create_metasound` hard-wired to `UMetaSoundSourceFactory` at AudioAuthoringHandler.cpp:617, no `assetClass` parameter. MetaSound has no text-IR escape hatch, so patches and presets are agent-unauthorable. Proposes either extending `create_metasound` with `assetClass: "source" | "patch"` or adding a dedicated `create_metasound_patch` handler, plus `create_metasound_preset` wrapping `UMetaSoundFactory::CreatePresetFromReferenced` (or equivalent) so presets become first-class for variant authoring.
- `#2-reviewed-and-confirmed` `OPEN` tester — Re-verified claims: `AudioAuthoringHandler.cpp:617` confirmed to hard-wire `UMetaSoundSourceFactory::FactoryCreateNew(UMetaSoundSource::StaticClass(), ...)` with no asset-class branch (lines 640-644); no `create_metasound_patch` or `create_metasound_preset` REGISTER_RPC_HANDLER exists in the file. No duplicate board entries. Severity Medium is justified — presets are the primary variant-authoring workflow (an N-variant ambience/engine system collapses to a hand-build chore without them), and patches are architecturally required for any reusable sub-graph composition. Proposed fix (Option A: `assetClass` param on `create_metasound` + new `create_metasound_preset`) is the minimal-surface approach and aligns with the existing factory pattern.
- `#3-patch-and-preset` `IN-REVIEW` developer — Added `audio.authoring.create_metasound_patch` and `create_metasound_preset` in new `Private/Handlers/Audio/MetaSound/MetaSoundPatchPresetHandler.cpp`. Patch creation uses `UMetaSoundFactory` (engine factory; verified at `MetasoundFactory.cpp:46-58` — `SupportedClass = UMetaSoundPatch`). Preset creation dispatches `UMetaSoundSourceFactory` or `UMetaSoundFactory` based on the referenced asset's class, with `Factory->ReferencedMetaSoundObject` set before `FactoryCreateNew` — engine subsystem `InitAsset` wires the preset linkage. The optional `overrides` param (initial input overrides at preset creation) is deferred — callers can use the existing `set_metasound_default` post-creation. Regression test `TestMetaSoundPatchPreset.cpp` exercises both factory paths against transient assets.
- `#4-verify-patch-preset` `DONE` tester — Verified: `audio.authoring.create_metasound_patch` schema is present and creating `/Game/McpVerify/MSP_McpVerifyTemp_FMetaSoundNoPatchOrPreset` returned `className: UMetaSoundPatch`, `assetClass: MetaSoundPatch`, `existsAfter: true`; `audio.authoring.create_metasound_preset` schema is present and creating `/Game/McpVerify/MSSP_McpVerifyTemp_FMetaSoundNoPatchOrPreset` from `/Game/McpVerify/MSS_McpVerifyTemp_FMetaSoundNoPatchOrPreset` returned `className: UMetaSoundSource`, referenced the source asset, and `existsAfter: true`. Temporary verification assets were deleted with `asset.delete`.
