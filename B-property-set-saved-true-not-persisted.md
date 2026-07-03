---
id: B-property-set-saved-true-not-persisted
title: "property.set hardcodes saved:true on a mark-dirty-only mutator, falsely signalling disk persistence"
status: OPEN
severity: High
category: bug
tags: [property, property-set, saved, false-success, mark-dirty, naming, persistence]
---

# `property.set` hardcodes `saved:true` though it only marks the package dirty

`property.set` returns `{"saved":true}` on every successful write, but it never
saves the package — it sets the value, calls `MarkPackageDirty()` (gated on the
`markDirty` param, default true) + `PostEditChange()`, then unconditionally
stamps `saved:true`. The package stays dirty until a separate `editor.save_all`
(or `asset.save`/`level.save`). An agent that trusts `saved:true` skips the
explicit save, so the edit never reaches disk/git and is lost on editor restart.

The field is a **factual lie**, not a missing feature: `property.set` is
*correctly* a mark-dirty-only writer per the documented contract —
`safe-mutation-save.md:49` ("Most asset-side writer calls mutate in-memory editor
state and mark packages dirty. They are not persisted until a save method
succeeds") and step 5 ("Save explicitly unless the method page states that the
writer already saves"). The defect is that the response field named `saved`
directly contradicts that contract. Five emit sites all hardcode it:
`Source/PinWright/Private/Handlers/Utility/UtilityPropertyHandler.cpp:958`
(ActorLocation), `:993` (ActorRotation), `:1029` (ActorScale), `:1053`
(bHidden), and the generic reflected path `:1083-1088` (`MarkPackageDirty()`
then `SetBoolField(TEXT("saved"), true)`).

Distinct from the "save verb writes no file" family
(`B-niagara-save-no-disk-write`, `B-audio-create-save-no-disk-write`,
`B-create-level-saved-true-no-umap`): those are *save*/`save:true` verbs that are
expected to persist and don't. `property.set` is a mutator that should NOT
auto-persist — the batch-edit-then-`save_all` workflow depends on it staying
dirty.

**Fix:** Do NOT make `property.set` auto-save (option rejected — it breaks the
batch-mutate/save-once workflow). Rename/repurpose the field so it stops claiming
persistence: report `applied:true` (the in-memory write succeeded) and
`dirtied:<markDirty>` (package now needs a save), dropping `saved` entirely.
Document that `editor.save_all`/`asset.save` is required to persist.

## History
- `#1-initial-repro` `OPEN` reporter — Handler-confirmed at `UtilityPropertyHandler.cpp:1083-1088`: `MarkPackageDirty()` then hardcoded `saved:true`; no package-save call anywhere in the handler. Replayable: (1) `call("property.set", {objectPath:"/App/App/PM_Drone", propertyName:"Friction", value:0.8})` → `{"propertyName":"Friction","saved":true,"assetPath":"/App/App/PM_Drone","value":0.800000011920929}`; (2) `git status --short -- Plugins/App/Content/App/PM_Drone.uasset` → EMPTY (no on-disk change), repeated across multiple property.set calls; (3) `call("editor.save_all", {})` → `{"success":true,"savedCount":2,"totalDirty":2}` — PM_Drone was still dirty and only now written (git then showed it modified), proving the earlier `saved:true` was false. Dedup: grepped board for `property.set`/`saved`/`mark dirty` — the no-disk-write tickets all target *save* verbs (niagara/audio/metasound/geometry/level) expected to persist; none covers `property.set`'s mislabelled `saved` field on an intentionally mark-dirty-only mutator. `F-asset-save` confirms property mutators leaving assets dirty is the expected contract.
