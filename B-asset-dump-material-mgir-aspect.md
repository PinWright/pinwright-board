---
id: B-asset-dump-material-mgir-aspect
title: "asset.dump should emit MGIR for materials and material functions"
status: DONE
severity: High
category: feature
tags: [asset-dump, material, mgir, round-trip]
---

# asset.dump should emit MGIR for materials and material functions

`asset.dump` / `asset.dump_folder` currently dump `UMaterial` and `UMaterialFunction` assets only through the generic aspect files:

```text
meta.json
properties.json
```

That is insufficient for material graph review and round-trip authoring. The plugin already has an MGIR surface (`material.decompile_mgir` / `material.compile_mgir`), but the asset dump pipeline does not call it and does not write an MGIR file.

## Repro

Dump `/App` via `asset.dump_folder`, then inspect material assets under:

```text
C:\Unity\unreal-fpv\.editor-automation\asset-dumps\App\App\Drone\Mini\Materials\M_Opaque_Master1
C:\Unity\unreal-fpv\.editor-automation\asset-dumps\App\App\Drone\Pioneer\MaterialFunctions\MF_BaseLayer
```

Observed:

```text
meta.json
properties.json
```

`meta.json` identifies these as `className: "Material"` and `className: "MaterialFunction"`, but no `mgir.txt` or `.mgir` file is written.

## Expected

For every dumped `UMaterial`, write an MGIR aspect, for example:

```text
mgir.txt
```

For every dumped `UMaterialFunction`, write the same MGIR aspect using the function entry form.

The MGIR text should come from the same implementation used by `material.decompile_mgir`, so dump output and live decompile output stay consistent. If decompile fails for a material, the dump should still write `meta.json` / `properties.json` and record the MGIR failure in a structured warning field or companion diagnostic file rather than aborting the whole folder dump.

## Required Fix

1. Add an `MGIR` dump filename constant, likely `mgir.txt`, to the asset dump file-name set.
2. In `AssetDumpHandler`, add explicit `UMaterial` and `UMaterialFunction` branches that call the MGIR decompiler and write `mgir.txt`.
3. Reuse `FMGIRDecompiler` or the same service path behind `material.decompile_mgir`; do not create a second material graph serializer.
4. Ensure `asset.dump_folder` writes MGIR for materials/functions during recursive dumps without changing existing `meta.json` / `properties.json` behavior.
5. Add tests or live verification on one material and one material function: dumped folders must contain `mgir.txt`, and the text must include the expected MGIR entry kind.
6. Update docs/wiki for `asset.dump` / `asset.dump_folder` aspect list to include `mgir.txt`.

## History

- `#1-initial-report` `OPEN` reporter — After a fresh `/App` folder dump, material assets existed in `.editor-automation\asset-dumps`, but only `meta.json` and `properties.json` were written. No `mgir` files appeared because the dump pipeline has no material/function MGIR branch.
- `#2-material-mgir-aspect` `IN-REVIEW` developer — Added `mgir.txt` for Material and Material Function asset dumps using `FMGIRDecompiler`, registered the aspect for diff baseline loading, preserved generic dump files on MGIR failures, added regression coverage, and updated asset dump docs.
- `#3-verify-mgir-aspect` `DONE` tester — Verified via live `asset.dump` on `/App/App/Drone/Mini/Materials/M_Opaque_Master1` (UMaterial) and `/App/App/Drone/Pioneer/MaterialFunctions/MF_BaseLayer` (UMaterialFunction): both dump folders now contain `mgir.txt` (12,739 bytes for the material, 3,076 bytes for the function) with proper `entry material`/`entry function` MGIR content alongside `meta.json` and `properties.json`.
