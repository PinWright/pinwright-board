---
id: B-asset-dump-doc-omits-static-mesh-sidecar
title: "asset.dump.md § \"Files by asset type\" omits Static Meshes and SoundWaves although static_mesh.json / static_mesh.txt / sound_wave.json all ship — a reviewer read the list and concluded no per-mesh baseline exists"
status: OPEN
severity: Low
category: bug
tags: [docs, wiki, asset-dump, static-mesh, sound-wave, sidecar, discoverability, doc-omits-shipped-behavior, weapons]
encounters: 1
costly: 1
lastSeen: 2026-09-03T00:00:00Z
---

# The list of what `asset.dump` writes is missing two of the things it writes

## Correction to the report that produced this ticket

This ticket was opened to report that **`asset.dump` has no Static Mesh sidecar**. That is **false**,
and the claim is retracted here rather than filed. The sidecar exists, ships, and is written on every
static-mesh dump. What is true is narrower and is the actual defect: **the page a caller reads to
find out does not list it.**

## Measured

**The doc omits it.** `asset.dump.md` § "Files by asset type" (the bulleted list at `:33-42` of the
generated page, read at
`X:\src\unreal\EAContentExamples58\Saved\PinWright\wiki\asset.dump.md`) enumerates Widget Blueprints,
Blueprints, Materials and Material Functions, Material instances, Levels/UWorlds, Niagara systems and
emitters, Cascade particle systems, DataTables, Textures, and DataAssets/generic UObjects. **Static
Meshes and SoundWaves are not in the list**, and neither `static_mesh.json`, `static_mesh.txt`, nor
`sound_wave.json` appears anywhere in that page's schema summary either — where `texture.json`,
`material_instance.json`, `data_table.json`, `cascade.json` and the rest each get a line.

**The sidecars ship.** Three independent confirmations:

- `asset.dump-sidecars.md:11` names them directly: *"`static_mesh.json` + `static_mesh.txt`,
  `texture.json` + `texture.txt`, `sound_wave.json` — emitted by the matching builders/text emitters
  in `Private/Handlers/Asset/`."*
- **384 dump directories** under `X:\src\unreal\EAContentExamples58\asset-dumps\` contain a
  `static_mesh.json`; 25 contain a `sound_wave.json`.
- The dumps' own manifest agrees. From
  `asset-dumps\Game\Dota2\Creeps\Meshes\SM_Creep_D_Siege\meta.json`:
  `"sidecarsEmitted": ["properties.json", "static_mesh.json", "static_mesh.txt"]`.

`F-asset-dump-native-summary-aspects` (DONE) is where they came from — `#2` shipped
`static_mesh.json`, `texture_2d.json` and `sound_wave.json` together. Textures were later folded into
the doc; static meshes and sound waves were not.

## Impact

The measured cost is this ticket's own origin: a reviewer looking for a greppable per-mesh baseline
to diff a weapons kit across builds read the "Files by asset type" list, found no static-mesh entry,
and concluded the baseline did not exist. It does — 384 of them were already on disk in this
checkout. A dump the caller already has is not usable if the doc says it is not there.

There is a per-asset escape hatch: `meta.json`'s `sidecarsEmitted` array is documented and does list
the files, so a caller who *already dumped* an asset can see them. That does not help the caller
deciding whether dumping is worth it, which is who reads the list.

## What is asked for

In the wiki-src overlay behind `asset.dump.md` (sibling tickets place these at
`docs/wiki-src/<page>.md` in the plugin repo — path convention taken from those tickets, not verified
for this page):

1. Add a **Static Meshes** bullet to § "Files by asset type": `meta.json`, `properties.json`,
   `static_mesh.json`, and compact `static_mesh.txt`.
2. Add a **SoundWaves** bullet: `meta.json`, `properties.json`, `sound_wave.json`.
3. Add the matching schema-summary lines, in the style of the existing `texture.json` line, naming
   what each carries. For `static_mesh.json` that is, verbatim from a dumped asset: `bounds`
   (origin/extent/sphereRadius), `materials[]` as `{slot, path}` pairs, `lods`, `trianglesByLod`,
   `verticesByLod`, `lightmapResolution`, `collisionTraceFlag`, and `collision` element counts
   (sphere/box/sphyl/convex/taperedCapsule). Say plainly what it does **not** carry — no sections, no
   UV channels, no per-LOD detail beyond triangle and vertex counts — so the next reviewer learns the
   limit from the doc instead of from a dump.
4. While in there: the same list's Textures bullet is the model to copy, and `static_mesh.txt` /
   `texture.txt` as a *class* of compact text companions is documented only on the sidecars page, not
   on the page most callers read.

The **content** gap — that the sidecar carries no sections or UV data — is not this ticket. It is
`F-static-mesh-section-material-map` and `F-static-mesh-uv-channel-readout`. This ticket is only that
the file's existence is undocumented on its primary page.

## Severity

**Low** — pure friction in the rubric's docs/discoverability band, and honestly so: nothing is wrong
with the data, no call fails, and the escape hatch exists. Bumped no higher despite the reach
modifier arguing for it (`asset.dump.md` is read by every dump consumer, and the list is the page's
most-used section), because the failure is a caller not finding something rather than a caller
believing something false about their asset.

## Related

- `F-asset-dump-native-summary-aspects` (DONE) — shipped `static_mesh.json` / `sound_wave.json` in
  `#2`. The doc gap dates from there.
- `F-static-mesh-section-material-map` (OPEN, High) and `F-static-mesh-uv-channel-readout` (OPEN,
  Medium) — what the sidecar should *also* carry. Both were filed in this same review round by a
  reviewer who could not diff meshes across builds; documenting the file and enriching it are the two
  halves of making that possible.
- `B-static-mesh-missing-collision-body-counts` (DONE) — an earlier content gap in the same sidecar,
  since fixed; its `collision` element counts are in the dumped output quoted above.
- `E-asset-dump-meta-sidecars-emitted-field` — the `sidecarsEmitted` field that is the current
  per-asset escape hatch.
- `B-asset-dump-missing-type-sidecars-for-anim-physics-skeleton` — genuinely missing sidecars, for
  contrast: there the file does not exist; here it exists and is unlisted.
- `E-dump-rpc-parity` — the rule that dump sidecars and their live RPCs carry the same fields.
  `static_mesh.json` and `static_mesh.describe` are already in parity; both are missing the same
  things, and both are documented unevenly.

## History
- `#1-filed` `OPEN` WEAPONS-critic — **Reshaped from a false premise, recorded here rather than filed as reported.** The report was "asset.dump has no Static Mesh sidecar, so there is no greppable per-mesh baseline for diffing a mesh across builds". Checked and disproven: `static_mesh.json` **and** `static_mesh.txt` ship on every static-mesh dump. Evidence — `asset.dump-sidecars.md:11` names `static_mesh.json` + `static_mesh.txt` and `sound_wave.json` as emitted by builders in `Private/Handlers/Asset/`; 384 dump directories under `X:\src\unreal\EAContentExamples58\asset-dumps\` contain a `static_mesh.json` (25 contain `sound_wave.json`); and `asset-dumps\Game\Dota2\Creeps\Meshes\SM_Creep_D_Siege\meta.json` self-reports `"sidecarsEmitted": ["properties.json", "static_mesh.json", "static_mesh.txt"]`. They came from `F-asset-dump-native-summary-aspects` (DONE) `#2`, which shipped `static_mesh.json`, `texture_2d.json` and `sound_wave.json` together. The real defect is the doc: `asset.dump.md` § "Files by asset type" (`:33-42` of the generated page at `X:\src\unreal\EAContentExamples58\Saved\PinWright\wiki\asset.dump.md`) lists Widget BPs, Blueprints, Materials/Material Functions, Material instances, Levels, Niagara, Cascade, DataTables, Textures and DataAssets/generic UObjects — and omits Static Meshes and SoundWaves entirely, with no schema-summary line for `static_mesh.json`, `static_mesh.txt` or `sound_wave.json` where `texture.json`, `material_instance.json`, `data_table.json` and `cascade.json` each get one. Measured cost is this ticket's own origin: a reviewer hunting a per-mesh baseline read that list, found nothing, and concluded none existed — while 384 were already on disk in this checkout. The `meta.json` `sidecarsEmitted` array is a per-asset escape hatch but only helps a caller who has already dumped the asset, not the caller deciding whether to. Ask: add Static Meshes and SoundWaves bullets to § "Files by asset type", add matching schema-summary lines naming what `static_mesh.json` carries (`bounds` origin/extent/sphereRadius, `materials[]` `{slot,path}`, `lods`, `trianglesByLod`, `verticesByLod`, `lightmapResolution`, `collisionTraceFlag`, `collision` element counts — quoted from a dumped asset) and stating plainly what it does not (no sections, no UV channels, no per-LOD detail beyond triangle/vertex counts), and document the `*.txt` compact companions on the primary page rather than only on the sidecars page. The content gap itself is out of scope here and belongs to `F-static-mesh-section-material-map` and `F-static-mesh-uv-channel-readout`. Severity Low, pure docs friction — no call fails and an escape hatch exists — not bumped despite `asset.dump.md` being an every-consumer page, because the failure is a caller not finding a file rather than believing something false about their asset.
