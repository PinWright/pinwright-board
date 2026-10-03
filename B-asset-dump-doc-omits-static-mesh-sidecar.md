---
id: B-asset-dump-doc-omits-static-mesh-sidecar
title: "asset.dump docs (asset.md § \"Files by asset type\" and Schema summary) omit Static Meshes, SoundWaves and texture.txt although static_mesh.json / static_mesh.txt / sound_wave.json / texture.txt all ship"
status: OPEN
severity: Low
category: bug
tags: [docs, wiki, asset-dump, static-mesh, sound-wave, sidecar, discoverability, doc-omits-shipped-behavior, weapons]
encounters: 1
costly: 1
lastSeen: 2026-09-03T00:00:00Z
rice: [2, 2, 1, 1]
priority: 33
---

# The list of what `asset.dump` writes is missing files it writes

`docs/wiki-src/asset.md` § `asset.dump` is the page a caller reads to decide whether a dump gives
them a per-asset baseline. Its "Files by asset type" list (`docs/wiki-src/asset.md:244-255`) has no
**Static Meshes** or **SoundWaves** bullet, and the Textures bullet (`:254`) names `texture.json`
but not its compact `texture.txt` companion. The "Schema summary" (`:257-277`) has lines for
`texture.json`, `cascade.json`, `material_instance.json` and others, but none for
`static_mesh.json`, `static_mesh.txt`, `sound_wave.json` or `texture.txt`.

The files ship: `Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.h:34-35,41-42` declares
`static_mesh.json`, `static_mesh.txt`, `texture.txt` and `sound_wave.json`, and
`AssetDumpCache.cpp:776,782,789-790` versions them. Only the sidecars page mentions them
(`docs/wiki-src/asset.dump-sidecars.md:9`).

Measured cost: a reviewer looking for a greppable per-mesh baseline to diff a weapons kit across
builds read the list, found no static-mesh entry, and concluded none existed, while 384
`static_mesh.json` files were already on disk in that checkout.

**Workaround:** `meta.json` `sidecarsEmitted` lists the files of an asset that has already been
dumped; `asset.dump-sidecars.md:9` names them.

**Fix:** in `docs/wiki-src/asset.md`:
1. Add a **Static Meshes** bullet: `meta.json`, `properties.json`, `static_mesh.json`, compact
   `static_mesh.txt`.
2. Add a **SoundWaves** bullet: `meta.json`, `properties.json`, `sound_wave.json`.
3. Add `texture.txt` to the Textures bullet.
4. Add Schema-summary lines. `static_mesh.json` (`StaticMeshDumpBuilder.cpp:31-152`): `bounds`,
   `materials[]` `{slot, path}`, `lods`, `trianglesByLod`, `verticesByLod`, `sections[]` (per LOD,
   material index/slot, triangle and index ranges, `boundingBox`), `uvChannelsByLod`, `slotUsage[]`
   (LOD0 triangles and `boundingBox` per slot), `lightmapResolution`, `lightMapCoordinateIndex`,
   `collision` element counts, `collisionTraceFlag`. `sound_wave.json`
   (`SoundWaveDumpBuilder.cpp:31-54`): `duration`, `numChannels`, `sampleRate`, `bLooping`,
   `soundGroup`, `volume`, `pitch`, `compressionQuality`. Say that the `.txt` files are compact text
   companions of the JSON sidecars.

**Acceptance:** the generated `asset.md` / `asset.dump` page lists Static Meshes and SoundWaves under
"Files by asset type", names `texture.txt` under Textures, and has Schema-summary lines for
`static_mesh.json`, `static_mesh.txt`, `sound_wave.json` and `texture.txt` whose field names match
the builders.

Related: `F-asset-dump-native-summary-aspects` (DONE, shipped the sidecars),
`E-asset-dump-meta-sidecars-emitted-field`, `E-dump-rpc-parity`.

## History
- `#1-filed` `OPEN` WEAPONS-critic — **Reshaped from a false premise, recorded here rather than filed as reported.** The report was "asset.dump has no Static Mesh sidecar, so there is no greppable per-mesh baseline for diffing a mesh across builds". Checked and disproven: `static_mesh.json` **and** `static_mesh.txt` ship on every static-mesh dump. Evidence — `asset.dump-sidecars.md:11` names `static_mesh.json` + `static_mesh.txt` and `sound_wave.json` as emitted by builders in `Private/Handlers/Asset/`; 384 dump directories under `X:\src\unreal\EAContentExamples58\asset-dumps\` contain a `static_mesh.json` (25 contain `sound_wave.json`); and `asset-dumps\Game\Dota2\Creeps\Meshes\SM_Creep_D_Siege\meta.json` self-reports `"sidecarsEmitted": ["properties.json", "static_mesh.json", "static_mesh.txt"]`. They came from `F-asset-dump-native-summary-aspects` (DONE) `#2`, which shipped `static_mesh.json`, `texture_2d.json` and `sound_wave.json` together. The real defect is the doc: `asset.dump.md` § "Files by asset type" (`:33-42` of the generated page at `X:\src\unreal\EAContentExamples58\Saved\PinWright\wiki\asset.dump.md`) lists Widget BPs, Blueprints, Materials/Material Functions, Material instances, Levels, Niagara, Cascade, DataTables, Textures and DataAssets/generic UObjects — and omits Static Meshes and SoundWaves entirely, with no schema-summary line for `static_mesh.json`, `static_mesh.txt` or `sound_wave.json` where `texture.json`, `material_instance.json`, `data_table.json` and `cascade.json` each get one. Measured cost is this ticket's own origin: a reviewer hunting a per-mesh baseline read that list, found nothing, and concluded none existed — while 384 were already on disk in this checkout. The `meta.json` `sidecarsEmitted` array is a per-asset escape hatch but only helps a caller who has already dumped the asset, not the caller deciding whether to. Ask: add Static Meshes and SoundWaves bullets to § "Files by asset type", add matching schema-summary lines naming what `static_mesh.json` carries (`bounds` origin/extent/sphereRadius, `materials[]` `{slot,path}`, `lods`, `trianglesByLod`, `verticesByLod`, `lightmapResolution`, `collisionTraceFlag`, `collision` element counts — quoted from a dumped asset) and stating plainly what it does not (no sections, no UV channels, no per-LOD detail beyond triangle/vertex counts), and document the `*.txt` compact companions on the primary page rather than only on the sidecars page. The content gap itself is out of scope here and belongs to `F-static-mesh-section-material-map` and `F-static-mesh-uv-channel-readout`. Severity Low, pure docs friction — no call fails and an escape hatch exists — not bumped despite `asset.dump.md` being an every-consumer page, because the failure is a caller not finding a file rather than believing something false about their asset.
- `#2-rephrased` `OPEN` developer — The old text cited the generated page `asset.dump.md` (the wiki-src overlay is `docs/wiki-src/asset.md:244-277`) and asked the doc to say `static_mesh.json` carries no sections or UV channels, which is now false: v2/v3 added `sections[]`, `slotUsage[]`, `uvChannelsByLod` and per-row `boundingBox` (`AssetDumpCache.cpp:787-789`, `StaticMeshDumpBuilder.cpp:76-127`). Reworded to the overlay path, the current field list, the missing `texture.txt`, and added Acceptance. Severity unchanged (Low).
