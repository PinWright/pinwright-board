---
id: B-asset-references-returns-dependencies-not-referencers
title: "asset.references answers the wrong question: `references` and `dependencies` come back byte-identical, so the verb cannot tell a caller what depends on an asset — the exact check needed before a safe delete"
status: DONE
severity: Medium
category: bug
tags: [asset, asset-references, dependencies, referencers, asset-registry, safe-delete, readback]
encounters: 2
costly: 1
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
- `#2-both-directions-explicit-keys` `IN-REVIEW` developer — Still reproducible before the fix, by source: `asset.references` (`Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp`) ran only `GetDependencies` and wrote that one array to both `references` and `dependencies` (and the one count to both `referenceCount` and `dependencyCount`) — the duplication was the wire-compat alias added by E-asset-dependencies-references-inverted #3, which kept `references` as outbound. Fix: the handler now also calls `GetReferencers` and emits each direction under an explicit key — `dependencies`/`dependencyCount` outbound (unchanged), new `referencers`/`referencerCount` inbound. **Behaviour change (CHANGELOG):** legacy `references`/`referenceCount` now carry the referencers, the reading both encounters gave the name; chosen over removing the keys because a dropped key read with a default comes back `[]`, i.e. "nothing references this" on the delete path. `asset.dependencies` is untouched (its legacy `dependencies` is still inbound; the wiki now warns it is the opposite of `asset.references`' `dependencies`). Registered summary rewritten to name both directions. No `direction` arg added — both lists are one registry call each; add one if a caller needs to trim the payload. Test: `PinWright.asset.references.DirectionAndAliases` (`Source/PinWright/Private/Tests/Assets/TestAssetReferenceDirection.cpp`) now asserts a registry-read precondition on the scratch A -> B Blueprint pair, then on A: `dependencies` has B, `referencers` and `references` do NOT; on B: `referencers` and `references` have A, `dependencies` does NOT, `referenceCount == referencerCount >= 1`. Reverting the handler fails the A-side `references` and B-side `referencers`/`references` assertions. The summary assertion that asset.references must not mention referencers was inverted with that reason. Docs: `docs/wiki-src/asset.md` (asset.references, asset.dependencies), README.md asset.references row, CHANGELOG.md. Syntax-checked with the clang -fsyntax-only fastcheck (UBT module flags; no UHT-visible declarations changed); not run in an editor here.
- `#3-verified-linux` `DONE` tester — Verified on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d). run3/full, non-skipped: `PinWright.asset.references.DirectionAndAliases`. A registry precondition holds on a scratch A -> B Blueprint pair. On A, `dependencies` contains B and `referencers`/`references` do not. On B, `referencers` and `references` contain A, `dependencies` does not, and `referenceCount == referencerCount >= 1`. The other asset.references and asset.dependencies tests also passed. This meets the suggested fix: the inbound direction comes from GetReferencers, outbound stays on GetDependencies, and the two fields no longer duplicate each other. The legacy `references` now means referencers; the behaviour change is in CHANGELOG, and asset.md warns that asset.dependencies' legacy field is the opposite direction. No `direction` argument was added; the ticket made it optional.
