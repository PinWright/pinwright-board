---
id: B-eqs-set-context-and-blueprint-create-saved-true-no-disk-write
title: "`eqs.set_context_class {save:true}` and `blueprint.create` both report `saved:true` while PIE blocks the write and nothing reaches disk — the same response also says `pendingSave:true`"
status: OPEN
severity: High
category: bug
tags: [eqs, blueprint, save, pie, silent-success, disk-verification]
---

# `eqs.set_context_class` and `blueprint.create` report `saved:true` while writing nothing to disk

Both verbs answer `saved: true` for a write that PIE blocked. The same response object simultaneously
carries `pendingSave: true` (and, for a newly created asset, `existsOnDisk: false`), so the payload
contradicts itself and the optimistic field is the one a caller reads.

## Reproduction (this checkout, 2026-09-05, another stream in PIE)

    blueprint.create {name:"EQC_LastKnown", savePath:"/Game/FPS/AI",
                      parentClass:"EnvQueryContext_BlueprintBase", waitForCompletion:true}
    -> {"saved": true, "existing": false, "mode": "created",
        "existsAfter": true, "existsOnDisk": false, "pendingSave": true}

    eqs.set_context_class {queryPath:"/Game/FPS/AI/EQS_Cover", generatorIndex:0, testIndex:0,
                           contextClass:"/Game/FPS/AI/EQC_LastKnown.EQC_LastKnown_C", save:true}
    -> {"saved": true, "existsOnDisk": true, "pendingSave": true, ...}

Four `set_context_class` calls and one `blueprint.create`, every one reporting `saved: true`. On
disk, minutes later:

    ls Content/FPS/AI/EQC_LastKnown.uasset   -> No such file or directory
    ls Content/FPS/AI/EQS_Cover.uasset       -> mtime 09-03 01:10:24  (two days stale)
    grep -ac EQC_LastKnown EQS_Cover.uasset  -> 0
    grep -ac EQC_Target    EQS_Cover.uasset  -> 1   (the value I replaced, still there)

So the asset the create claimed to save does not exist, and the query still references the old
context. The in-memory objects read back correctly — `find_object` on the tests shows
`context=EQC_LastKnown_C` — which is precisely why an in-memory read-back cannot catch this.

## Why it matters

`saved: true` is the field a caller checks before moving on. Here it is wrong in the one situation
the caller cannot see: another stream holding PIE in a shared editor. The work looks done, the
read-back agrees, and the change evaporates on the next editor restart. In this project that is a
whole build's asset work silently lost.

The information needed to answer correctly is already in the response — `pendingSave: true`,
`existsOnDisk: false` — so this is a reporting bug, not a missing capability. `editor.save_all`
already gets it right in the same conditions, returning
`"Saved 0 of 1 dirty assets (PIE active; 1 asset(s) locked by PIE)"` with `pieActive: true` and a
`failedAssets` array naming `BlockedByPie`. `asset.save` also now reports a real `saveState`
(`written` / `deferred`) with a `saveDetail` string. These two verbs have not been brought in line.

## Ask

1. Never report `saved: true` when the package was not written. Report `saved: false` with the same
   `BlockedByPie` reason `editor.save_all` already produces, or adopt `asset.save`'s
   `saveState: "written" | "deferred" | "blocked"` + `saveDetail` contract so one spelling covers
   every verb that can persist.
2. `saved` and `pendingSave` must not both be true in one response — if the flush is pending, the
   save has not happened.
3. `blueprint.create` should not claim `saved: true` alongside `existsOnDisk: false`.

## Workaround

Verify against the file, never the returned object or a `find_object` read-back: `ls` the `.uasset`
mtime and `grep -a` for the new value. Re-issue the save after PIE ends. Note that
`unreal.EditorAssetLibrary.save_asset(path, only_if_is_dirty=False)` is also silently ineffective
while PIE holds the package, so the retry has to wait for PIE rather than force harder.

## Relationship to existing tickets

Same failure shape as `B-property-set-saved-true-not-persisted` (IN-REVIEW), which covers
`property.set` hardcoding `saved:true` on a mark-dirty-only mutator, and adjacent to
`B-asset-save-pie-failure-reports-pendingflush`, `B-create-level-saved-true-no-umap` and
`B-metasound-create-save-no-disk-write`. Filed separately because it names two further verbs
(`eqs.set_context_class`, `blueprint.create`) that none of those tickets covers, and because the
PIE-blocked path is what makes it bite in a shared editor.
