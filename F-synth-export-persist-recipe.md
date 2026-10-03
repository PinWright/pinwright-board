---
id: F-synth-export-persist-recipe
title: "audio.synth.export does not persist the recipe on the USoundWave, so a shipped asset cannot be patched, varied or read back once its candidate is evicted"
status: OPEN
severity: Medium
category: feature
tags: [audio, synth, export, variations, patch, get_recipe, round-robin]
encounters: 1
costly: 1
---

# A shipped synth wave has no recipe behind it

Split out of `F-rpc-synth-get-recipe` `#2-round-robin-variants-have-no-source-to-vary-from`.
`audio.synth.get_recipe` now returns a RESIDENT candidate's canonical recipe, but the candidate
registry is session memory (64 slots, byte budget). `audio.synth.export` writes only the PCM
payload into the `USoundWave`, so once the candidate is evicted or the editor restarts the recipe
that produced a shipped asset is gone. `audio.synth.variations` and `audio.synth.patch` take only a
`candidateId`, so "author a round-robin `_B` of `SW_Fire_AR_Body`" means reverse-engineering the
original from `audio.analysis.*` (the reporter measured 21 `generate` iterations for four variants).

## Ask

- On a verified export, store the canonical recipe (`SerializeSynthRecipe`, condensed) on the wave:
  package metadata or an asset-registry-searchable tag, editor-only, preserved by the in-place
  rewrite path.
- Accept `assetPath` as an alternative source on `audio.synth.get_recipe`, `patch` and
  `variations`; an asset without a stored recipe is refused by name, not answered with an empty
  document.

## Notes for the implementer

- Touches `PwCreateSoundWaveAsset` (`Source/PinWright/Private/AudioGen/PwAudioExport.cpp`) or the
  export handler before save; the in-place rewrite's `verification.propertiesPreserved` must not
  flag the new metadata as a moved property.
- The metadata API differs across 5.3-5.8 (`UMetaData` vs `FMetaData` on `UPackage`); guard it and
  record the row in `docs/engine-version-support.md`.

## History
- `#1-split-from-get-recipe` `OPEN` developer — Split from `F-rpc-synth-get-recipe` while
  implementing `audio.synth.get_recipe` (resident candidates only). Not done in that change because
  it rewrites the export writer, which another change owns concurrently, and widens three verb
  contracts.
