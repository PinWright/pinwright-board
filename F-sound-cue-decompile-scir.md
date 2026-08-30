---
id: F-sound-cue-decompile-scir
title: "Add SCIR (Sound Cue text IR) decompiler + `audio.authoring.decompile_sound_cue`"
status: DONE
severity: Low
category: feature
tags: [audio, sound-cue, ir, scir, decompile, asset-dump]
---

# Add SCIR (Sound Cue text IR) decompiler + `audio.authoring.decompile_sound_cue`

SoundCue graphs are currently inspectable as JSON via `audio.authoring.describe_sound_cue` and the `sound_cue.json` dump sidecar (both routed through `SoundCueDumpBuilder::BuildSoundCueJson` at `Source/PinWright/Private/Handlers/Asset/SoundCueDumpBuilder.cpp`). JSON is fine for machine consumption but heavy for prompt review and round-trip authoring: every USoundNode subobject takes ~15 lines of JSON for what is conceptually a one-line tree node.

There is no text-IR equivalent. `grep` across `Source/` for `decompile_sound_cue` / `SCIRDecompiler` / `scir` returns zero hits. SoundCue is currently the only audio asset without a text-IR view, while BPIR, AGIR, and MGIR all ship decompilers.

Propose a small, decompile-only text IR ("SCIR") with the tree-nested grammar below — mirrors AGIR's nested-block pattern, leverages the existing DFS traversal in `SoundCueDumpBuilder` (root `FirstNode` + recursive `ChildNodes`, cycle/shared-child handling via TSet `Visited`), and emits a sidecar `scir.txt` next to `sound_cue.json`.

**Grammar sketch:**

```
sound_cue MyCue {
  attenuation /Game/Audio/A_Default.A_Default
  volume 1.0
  pitch 1.0
  root mixer `Mixer` @(0, 0) {
    child wave_player `Footstep1` @(-200, -100) { wave: /Game/Audio/SW_Foot1.SW_Foot1 }
    child random `Random` @(-200, 100) {
      weights: [1.0, 1.0, 0.5]
      randomize_without_replacement: true
      child wave_player @(-400, 50) { wave: /Game/Audio/SW_V1.SW_V1 }
    }
  }
  orphan looping `Loop_Orphan_42` @(400, 0) { loop_count: 3 }
}
```

Cue-level fields cover `AttenuationSettings`, `ConcurrencySet`, `SoundClass`, `VolumeMultiplier`, and `PitchMultiplier`. Per-node blocks emit class-token (`mixer`, `wave_player`, `random`, `looping`, etc. — one token per USoundNode subclass), backticked instance name, `@(x, y)` editor position, and a CDO-delta property list using the same shared dispatch the other IRs use. Orphan nodes (in `AllNodes` but not reachable from `FirstNode`) emit as top-level `orphan` blocks.

**Scope:** decompile-only on day one. No compile path. Round-trip authoring is out of scope until a real use case appears (YAGNI — JSON describe + imperative `audio.authoring.add_cue_node` / `connect_cue_nodes` already cover authoring).

**Fix (proposed):**
- New files: `Private/SCIR/SCIRDecompiler.{h,cpp}` and `Private/Handlers/Audio/SCIRDecompileHandler.cpp`.
- Shared builder: `SCIRDecompiler::BuildSoundCueIrText(USoundCue*)` returns `{ bSuccess, Text, Warnings }` and covers only cue fields that exist on UE 5.6 `USoundCue`/`USoundBase`.
- Reuse: `SoundCueDumpBuilder` traversal shape (`FirstNode` + recursive `ChildNodes` + orphan `AllNodes` handling), `IrCore/IrTextUtils` (Quote, FormatNameToken, FormatPositionSuffix, FormatFieldList, IsSafeReflectedProperty), and the existing decompile handler pattern.
- Sidecar: add `scir.txt` to `AssetDumpHandler.h` `DumpFileNames` and emit alongside the preserved `sound_cue.json` from the SoundCue dispatch branch by calling the shared builder.
- RPC: `audio.authoring.decompile_sound_cue` (params: `assetPath`). Thin wrapper that loads `USoundCue` locally because the existing audio loader is file-local to `AudioAuthoringHandler.cpp`, calls the shared builder, and returns `{ ir, warnings }`.

**Effort:** ~250 LoC, ~1 dev-day. Smallest of the in-flight new-IR tickets — most of the traversal and property-emit infrastructure is already shared.

**Soft prereqs (recommended, not blocking):**
- [F-ircore-reflected-property-emit](F-ircore-reflected-property-emit.md) — extracts the shared CDO-delta dispatch SCIR would otherwise re-implement.
- IR grammar harmonization (not yet ticketed) — adopt the `:` field separator on day one so SCIR doesn't join the existing 5-way divergence across BPIR/AGIR/MGIR/etc.
- IR authoring guide doc (not yet ticketed) — would absorb SCIR's grammar reference page.

If F-ircore-reflected-property-emit lands first, SCIR drops to ~180 LoC.

## History
- `#1-initial-proposal` `OPEN` reporter — Verified `decompile_sound_cue` does not exist (grep on Source/ for decompile_sound_cue/SCIR/scir returned zero hits). Verified `SoundCueDumpBuilder.cpp` exists at `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/SoundCueDumpBuilder.cpp` and already walks `FirstNode` recursively with TSet cycle handling. Verified 10 imperative `audio.authoring.*_cue_*` RPCs already cover authoring, and `audio.authoring.describe_sound_cue` (DONE in F-rpc-audio-describe-sound-cue) already ships JSON inspect — so the only gap is a prompt-friendly text view. Proposed tree-nested SCIR grammar (sketch above), decompile-only scope, ~250 LoC estimate, reuse plan keyed off `SoundCueDumpBuilder` + `IrCore/IrTextUtils` + AGIR decompile handler template.
- `#2-dual-surface-mandate` `OPEN` reporter 2026-05-13 — Locking in the **dual-surface invariant** before implementation. BOTH surfaces are MANDATORY, neither is optional: (a) asset-dump sidecar `scir.txt` registered in `AssetDumpHandler.h` `DumpFileNames` and emitted from the SoundCue dispatch branch in `AssetDumpHandler.cpp`, written under `.editor-automation/asset-dumps/.../<asset>/scir.txt`; (b) MCP RPC `audio.authoring.decompile_sound_cue` handler at `Private/Handlers/Audio/SCIRDecompileHandler.cpp`, callable via `call("audio.authoring.decompile_sound_cue", { assetPath })`, returns `{ ir, warnings }`. Both surfaces MUST share ONE builder function `BuildSoundCueIrText(USoundCue*) → FIrResult { Text, Warnings, bSuccess }` living in `Private/SCIR/SCIRDecompiler.cpp`. The sidecar pipeline calls it for dump emission; the RPC dispatch calls it for direct response. Zero divergence between the two — never ship one without the other, never let the two implementations drift. If the dump branch needs format differences, route them through builder options, not a parallel implementation.
- `#3-revised-fix-scope` `OPEN` developer — Revised implementation scope after UE 5.6 API review: cue-level `SoundGroup` is not exposed on `USoundCue`, so SCIR covers `attenuation`, `concurrency`, `sound_class`, `volume`, and `pitch` only. The new RPC must load `USoundCue` locally because `LoadSoundCueFromPath()` is file-local to `AudioAuthoringHandler.cpp`; the shared SCIR builder remains the single generation path for RPC and asset-dump output.
- `#4-scir-dual-surface` `IN-REVIEW` developer — Implemented the shared `SCIRDecompiler::BuildSoundCueIrText(USoundCue*)` builder, wired `audio.authoring.decompile_sound_cue` to return `{ ir, warnings }`, preserved `sound_cue.json`, added `scir.txt` as the SoundCue asset-dump sidecar, and added planned regression tests for builder shape, sidecar emission, and RPC response shape.
- `#5-verify-scir-surfaces` `DONE` tester — Verified: `audio.authoring.decompile_sound_cue` on `/Game/Audio/FPV_SOUND/UI/UISFX_Start_Cue.UISFX_Start_Cue` returned SCIR with `sound_cue`, `root mixer`, child `wave_player` nodes, and empty warnings; `asset.dump` for the same SoundCue wrote both preserved `sound_cue.json` and new `scir.txt`, with `scir.txt` containing matching SCIR text.
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
