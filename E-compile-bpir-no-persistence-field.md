---
id: E-compile-bpir-no-persistence-field
title: "blueprint.compile_bpir reports compiled:true / status:UpToDate and no persistence field at all, while its sibling blueprint.* writers auto-save — so authored graphs read as durable and die with the editor"
status: OPEN
severity: Medium
category: ergonomic
tags: [blueprint, compile_bpir, save, persistence, saveState, pendingFlush, safe-mutation-save, response-shape]
encounters: 1
lastSeen: 2026-09-02T19:44:57Z
---

# `compile_bpir` says nothing about durability, in a namespace where its siblings save themselves

## The inconsistency

Within `blueprint.*`, the write verbs disagree about saving and only some of them say so:

| Verb | Saves | Response says |
|---|---|---|
| `blueprint.add_variable` | yes, unconditionally | `"saved": true` (or `saved:false` + `pendingSave:true`) |
| `blueprint.add_function` | yes, unconditionally | `"saved": true` / `pendingSave` |
| `blueprint.add_dispatcher` | on `save:true` | `"saved": true` |
| `blueprint.add_interface` | on `save:true` | `"saved": true` |
| `blueprint.set_default` | never (documented) | `markedForSave:true, saved:false, pendingFlush:true` |
| **`blueprint.compile_bpir`** | **never** | **nothing** — `{nodeCount, createdNodes, errors, warnings, compiled:true, status:"UpToDate", success:true}` |

`compile_bpir` is the *only* one that neither saves nor reports that it did not. `compiled: true`
plus `status: "UpToDate"` plus `success: true` is a strong success signal with no durability
qualifier anywhere in it, and it arrives in a namespace where the neighbouring verbs the same
session just called did save. `safe-mutation-save` § "Dirty State And Runtime State" says to read
the persistence fields rather than top-level `success` — but there are no persistence fields on
this response to read.

## What it cost

Eight consecutive `compile_bpir` calls authored eight functions on `/Game/FPS/UI/WBP_HUD`, each
returning `compiled:true, status:"UpToDate", success:true`. The editor then crashed (separately
filed as `B-screenshot-designer-leaves-designer-open-compile-crash`). On disk afterwards:

```
grep -ac RefreshSource   Content/FPS/UI/WBP_HUD.uasset -> 0
grep -ac UpdateCrosshair Content/FPS/UI/WBP_HUD.uasset -> 0
grep -ac UpdateAmmo      Content/FPS/UI/WBP_HUD.uasset -> 0
...  (all eight: 0)
grep -ac HudSource       Content/FPS/UI/WBP_HUD.uasset -> 3     # add_variable, auto-saved
grep -ac MaxMagSeen      Content/FPS/UI/WBP_HUD.uasset -> 3     # add_variable, auto-saved
```

Every variable survived; every authored graph was gone. The split is exactly the auto-save /
no-save split in the table above — which is also what makes the trap convincing: the same session's
earlier responses had trained the caller that `blueprint.*` writes persist.

## What should happen

Either is fine; the point is that the response must state the durability it has:

1. Add the standard persistence block to `compile_bpir`'s response —
   `saveRequested` / `saved` / `saveState` / `pendingFlush`, the shape `safe-mutation-save`
   § "Save States" defines and `blueprint.set_default` already emits. Even
   `saved:false, pendingFlush:true` is enough: it tells the caller to flush.
2. Or accept an optional `save` boolean like `blueprint.add_dispatcher` and
   `blueprint.add_interface` do, defaulting to false, and report the outcome.

The wiki page should also state plainly that the verb does not persist and that `asset.save` must
follow — `blueprint.compile_bpir.md` currently discusses compile semantics, rollback and modes at
length and never mentions saving.

**Workaround:** call `asset.save {force:true}` after **every** `compile_bpir`, and verify the
`.uasset` mtime / `grep -a` for a symbol the call created. Batching several compiles before one
save is what turns a crash into lost work.

severity rationale: impact=friction, but the friction is a response shape that reads as durable
success and is inconsistent with its own namespace's siblings, so the caller has to already know
the answer to ask the question x reach=`compile_bpir` is the primary Blueprint graph-authoring
verb, used in nearly every authoring session -> Medium.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) authoring `/Game/FPS/UI/WBP_HUD` for the FPS UI stream. Eight `compile_bpir` calls (`RefreshSource`, `UpdateCrosshair`, `UpdateAmmo`, `UpdateHealth`, `UpdateHitMarker`, `UpdateArcs`, `UpdateFeed`, `UpdateCompass`) each returned `{nodeCount:N, createdNodes:[...], errors:[], warnings:[], compiled:true, status:"UpToDate", success:true}` with no `saved`, `saveState`, `pendingSave` or `pendingFlush` field. An editor crash before the next `asset.save` lost all eight; `grep -a` against `Content/FPS/UI/WBP_HUD.uasset` confirms none of the eight function names is in the package while `HudSource` and `MaxMagSeen` (added by `blueprint.add_variable`, which auto-saves and says `saved:true`) both are. The mixed behaviour inside one namespace is the ergonomic defect: `add_variable` / `add_function` save without being asked, `add_dispatcher` / `add_interface` take a `save` flag, `set_default` explicitly reports `pendingFlush:true`, and `compile_bpir` alone is silent. Not filed as a bug because `safe-mutation-save` does tell callers to save explicitly; filed as ergonomic because the response gives them nothing to check and the neighbouring verbs teach the opposite habit.
