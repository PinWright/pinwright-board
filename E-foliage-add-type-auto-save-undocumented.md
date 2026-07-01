---
id: E-foliage-add-type-auto-save-undocumented
title: "foliage.add_type already persists the new asset (calls McpSafeAssetSave + returns exists_after:true) but its wiki/registry entry never says so, prompting a redundant defensive asset.save — and it breaks the plugin's documented don't-auto-save convention"
status: WONTFIX
severity: Low
category: ergonomic
tags: [docs, foliage, wiki, add_type, save, persistence, redundant-call, convention]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `foliage.add_type` auto-saves on creation but doesn't advertise it, so callers add a redundant `asset.save`

`foliage.add_type` creates a `UFoliageType_InstancedStaticMesh` and **persists
it to disk inside the handler** — `FoliageHandler.cpp:592
McpSafeAssetSave(FoliageType)` — then returns a response that already signals the
asset is on disk: `created:true`, `exists_after:true`, `asset_path`, plus
`AddAssetVerification(Resp, FoliageType)` (`:594-603`). The asset is fully saved
before the call returns.

But neither the registry description nor the wiki page says the asset is saved on
creation. The registration (`:459`) is just *"Create a new foliage type asset
from a static mesh"*, and the param list (`:460-468`) documents only the inputs
(name, meshPath, density, minScale/maxScale, alignToNormal, randomYaw) — nothing
about persistence or the `exists_after` field in the output. So a careful caller,
not knowing the asset is already flushed, follows `add_type` with a defensive
single-asset save — which is a wasted call.

## Why it matters — the process friction (this task)

Seed story: dress the Blueprint_Communication demo with scattered `SM_Lightbulb`
foliage — create `FT_Lightbulb_Scatter` via `add_type`, add 4 instances, paint 3,
read back 7, remove all, verify 0. The run was otherwise clean (15 calls, every
one `ok`, no retries / errors / python fallback). But the call log shows the
redundant save directly:

- call #9 `foliage.add_type {name=FT_Lightbulb_Scatter, mesh=SM_Lightbulb,
  density=200, min0.5, max1.5, randomYaw=true, alignToNormal=false}` →
  `ok`, `assetPath=/Game/Foliage/FT_Lightbulb_Scatter` (this call already saved it)
- call #10 `asset.save {FT_Lightbulb_Scatter force=true}` → `ok`,
  same `assetPath` — a **second, redundant** persistence of an asset `add_type`
  had already written.

The friction note says "none … every call succeeded on first try," and that's
true — there was no *error*. But the extra `asset.save` is exactly the
"excessive steps for a simple intent / extra call where the handler already does
the work" friction shape: the agent didn't trust (or couldn't discover) that
`add_type` persists, so it paid one extra call to be safe. That's a
discoverability cost, not a correctness one.

## The sharper angle — it breaks the plugin's own documented convention

`F-data-table-row-authoring` records the plugin's stated write-verb convention
(verbatim): *"do **not** auto-save (consistent with the plugin's other write
verbs — saving is `system.save_asset`'s job)."* `foliage.add_type` **violates**
that convention by auto-saving (`:592`). So the ecosystem is inconsistent: most
write/create verbs leave the asset dirty for the caller to flush, but `add_type`
flushes itself — and nothing in `add_type`'s surface tells the caller which camp
it's in. An agent that has internalized "create verbs don't save, I must save"
will *always* add the redundant `asset.save` after `add_type`; an agent that
assumes the opposite for some other create verb will skip a needed save and lose
data. The silent inconsistency is the real cost; the redundant call here is the
cheap symptom.

## What it should do / how to fix

Docs-only (downstream wiki process — not a code change): the overlay
`docs/wiki-src/foliage.md` should add (or extend) a `### foliage.add_type`
authoring block noting that **the new foliage-type asset is saved to disk on
creation** (no follow-up `asset.save` needed) and documenting the output fields
that prove it — `created`, `exists_after:true`, `asset_path`, and the
`AddAssetVerification` block. That alone converts "save defensively just in case"
into "the response already confirms it's on disk."

Optionally (separate, code-side, out of scope for this docs ticket): reconcile
the auto-save inconsistency — either align `add_type` with the documented
don't-auto-save convention (drop `:592`, let the caller flush via the
forthcoming `asset.save` / `F-asset-save`), or, if auto-save-on-create is the
intended behavior for asset-*creating* verbs (vs. property mutators), state that
split convention once in the namespace overlay so it's discoverable across verbs.

**Workaround:** today, trust the `add_type` response — `exists_after:true` +
`AddAssetVerification` already confirm the `.uasset` is on disk, so the
follow-up `asset.save` can be dropped. (Discoverable only by reading
`FoliageHandler.cpp:592-603`.)

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of a
  `foliage.add_type` scatter-dressing task (seed `foliage.add_type`, mesh
  `SM_Lightbulb`, `FT_Lightbulb_Scatter`): 15 calls, all `ok`, no
  retries/errors/python fallback. Friction note says "none," but the call log
  shows a redundant `asset.save {FT_Lightbulb_Scatter force=true}` (call #10)
  immediately after `foliage.add_type` (call #9) — and `add_type` already
  persists the asset itself (`FoliageHandler.cpp:592 McpSafeAssetSave`, response
  `created/exists_after:true/asset_path` + `AddAssetVerification` `:594-603`), so
  the second save is wasted work. Root cause: the `add_type` registry
  description (`:459` "Create a new foliage type asset from a static mesh") and
  param list (`:460-468`) document inputs only — nothing about saving-on-create
  or the `exists_after` output — so a careful caller saves defensively. Sharper:
  this auto-save **breaks the plugin's documented convention** (per
  `F-data-table-row-authoring`: "do not auto-save … saving is system.save_asset's
  job"), making the create-verb save behavior silently inconsistent across the
  surface. Proposed deliverable: a `### foliage.add_type` block in
  `docs/wiki-src/foliage.md` stating the asset is saved on creation (no follow-up
  `asset.save` needed) and documenting the `created`/`exists_after`/`asset_path`
  verification fields. Dedup: ripgrep across OPEN/closed found no ticket on
  `add_type` auto-save / redundant-defensive-save; existing foliage tickets are
  `E-foliage-get-instances-drops-scale` (read-back, scale — fix present in working
  tree at `:406-408`) and `E-foliage-nested-input-schemas-undocumented`
  (add_instances/create_procedural nested input shapes) — different methods,
  different direction. The save-themed tickets (`F-asset-save`,
  `B-niagara-save-no-disk-write`, etc.) are about save verbs that *fail to*
  persist — the inverse of this (a create verb that *does* persist but doesn't
  say so).
- `#2-wontfix` `WONTFIX` developer — The ticket's load-bearing premise is
  factually inverted. `foliage.add_type` does NOT persist the asset to disk;
  `FoliageHandler.cpp:592 McpSafeAssetSave(FoliageType)` routes into
  `McpSafeAssetSave` (`Source/PinWright/Private/Utils/AssetUtils.cpp:214-226`),
  whose entire body is `Asset->MarkPackageDirty(); FAssetRegistryModule::AssetCreated(Asset); return true;`
  under the explicit comment "UE 5.7+ Fix: Do not immediately save newly created
  assets to disk. Saving immediately causes bulkdata corruption and crashes."
  The header restates it (`AssetUtils.h:123-127` "defers the disk write … returns
  true unconditionally"). So after `add_type` the `.uasset` is dirty-in-memory
  only — the reporter's call #10 `asset.save {force=true}` was the FIRST and ONLY
  durable disk write, i.e. REQUIRED, not redundant. The "proof it's on disk"
  fields prove nothing: `exists_after:true` is a hardcoded literal
  (`FoliageHandler.cpp:597`), and `AddAssetVerification` hardcodes
  `existsAfter:true` with no disk probe (`AssetUtils.cpp:1088`) — the real
  disk-presence check `VerifyAssetExists`/`DoesAssetExist` (`AssetUtils.cpp:1105-1114`)
  is a different helper add_type does not call. The proposed deliverable
  (document "saved to disk on creation, no follow-up asset.save needed") and the
  "Workaround" (drop the save) would assert a falsehood and CAUSE data loss on
  editor restart. The "breaks the don't-auto-save convention" angle is also
  inverted: the convention means mark-dirty-but-don't-flush, which is exactly
  what McpSafeAssetSave does, so add_type follows it. The only genuine
  phenomenon (exists_after:true is a misleading persistence signal after a
  mark-dirty-only create) is the INVERSE of this ticket and is already an owned
  defect class (B-niagara-save-no-disk-write, B-metasound-create-save-no-disk-write,
  B-audio-create-save-no-disk-write); it belongs there, not as a reword of this
  docs ticket. Correctness=invalid, adversarial=wontfix. No code change.
