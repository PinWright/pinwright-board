---
id: B-asset-references-returns-dependencies-not-referencers
title: "asset.references answers the wrong question: `references` and `dependencies` come back byte-identical, so the verb cannot tell a caller what depends on an asset — the exact check needed before a safe delete"
status: OPEN
severity: Medium
category: bug
tags: [asset, asset-references, dependencies, referencers, asset-registry, safe-delete, readback]
encounters: 2
---

# `asset.references` returns what the asset depends ON, in both fields

## Symptom

`asset.references` was called to answer "is anything still using this asset, so is it safe to
delete". The response carries two arrays, `references` and `dependencies`, plus
`referenceCount` and `dependencyCount`. **Both arrays contained the same 57 entries, and both
counts were 57.**

Every entry was an outbound dependency — `/Niagara/Modules/...`, `/Niagara/Enums/...`,
`/Script/Niagara`, the material instances the emitters bind, `/Engine/BasicShapes/Sphere`.
Not one was a package that *uses* the target.

```
call asset.references { assetPath: "/Game/FPS/VFX/NS_Impact_Flesh" }
  -> referenceCount: 57, dependencyCount: 57
     references[]  == dependencies[]   (identical lists)
     e.g. /Niagara/Modules/Update/Forces/Drag, /Engine/BasicShapes/Sphere,
          /Game/FPS/VFX/Materials/MI_FPS_Blood_Spray, /Script/NiagaraEditor
```

The true answer, obtained through the asset registry from `python.execute`:

```python
reg.get_referencers("/Game/FPS/VFX/NS_Impact_Flesh", unreal.AssetRegistryDependencyOptions())
  -> ['/Game/FPS/Test/T_VFX']      # one referencer, a level holding a placed actor
```

One referencer, which is the single fact that decides whether the delete is safe. The verb
reported 57 unrelated packages instead and never mentioned it.

## Second, independent encounter the same session

Another agent on this build called `asset.references` on
`/Game/FPS/VFX/Materials/MI_FPS_Water_Crown` and got `referenceCount: 1`, naming only the
material's **parent material** — again an outbound dependency. Two emitters
(`E_ImpactWater_Column`, `E_ImpactWater_Crown`) hard-reference that instance and neither
appeared. They fell back to grepping the `.uasset` bytes to find the real users.

So the failure is not specific to Niagara systems or to a large dependency set: a 57-entry
result and a 1-entry result both contained only dependencies.

## Why it matters

Referencers and dependencies answer opposite questions, and the referencer direction is the
one with consequences:

- **Safe delete.** "Can I remove this?" is exactly "who still references it?". A caller
  trusting this verb sees a long list of engine modules, reads it as "heavily referenced",
  and declines a safe delete — or sees a short list of unrelated packages and deletes
  something a level still points at.
- **Blast radius before an edit.** "What breaks if I change this material?" is a referencer
  query. In a shared multi-agent editor that is the difference between a scoped edit and
  clobbering another stream.
- **Orphan detection.** "Is this asset dead content?" is unanswerable through this verb.

Neither is served today, and the response looks authoritative: two distinctly named fields,
two counts, no warning, no note in the wiki page saying the two are the same list.

## Suggested fix

Populate `references` from `IAssetRegistry::GetReferencers` and leave `dependencies` on
`GetDependencies`, so the two named fields mean what they say. If only one direction can be
supported, name the field for the direction it actually returns and say so on the wiki page —
a verb called `references` that returns dependencies is worse than one honestly called
`asset.dependencies`.

Worth adding either way: a `referencersOnly` / `direction` argument, since the two questions
have different callers and a caller almost never wants both fused.

## Workaround

`python.execute` with `unreal.AssetRegistryHelpers.get_asset_registry().get_referencers(path,
unreal.AssetRegistryDependencyOptions())`, filtering to `/Game` to drop engine noise. That is
what produced the correct one-entry answer above.

## History

- `#1-filed` `OPEN` reporter — Found on UE 5.8 / EAContentExamples58 while deciding whether the orphaned `NS_Impact_Flesh` could be deleted after a VFX critic asked for it to be removed so it could not fire. `asset.references` returned 57 entries in both fields; `get_referencers` returned exactly one, `/Game/FPS/Test/T_VFX`, a level holding a leftover placed actor — the one thing that had to be cleared first. Had I trusted the verb I would have concluded the asset was heavily referenced and left dead content live, which is precisely the outcome the critic flagged. Second, independent encounter the same session from another agent on `MI_FPS_Water_Crown`: `referenceCount: 1` naming only the parent material while two emitters hard-reference the instance; they resorted to grepping `.uasset` bytes. Two different asset classes, two different result sizes, same direction error. No plugin source read — evidence is the two RPC responses and the registry readback that contradicts them. severity rationale: impact = a wrong answer to the question asked before every delete and every shared-asset edit, presented with no hedge (Medium-High) x reach = any caller doing dependency analysis, which in a multi-agent editor is routine -> Medium.
