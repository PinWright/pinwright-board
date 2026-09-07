---
id: E-asset-save-force-no-clean-check
title: "asset.save{force:true} has no cheap already-clean no-op, so agents defensively over-call force-save; editor went connection-refused exactly at the first such call (possible force-resave crash — needs repro)"
status: OPEN
severity: Medium
category: ergonomic
tags: [asset, save, force, dirty, no-op, persistence, possible-crash, needs-repro]
encounters: 1
costly: 1
lastSeen: 2026-07-01T00:17:53.1197644+03:00
blockedBy: [F-asset-save]
---

# asset.save force-resave: no cheap clean-check → defensive over-calls, and a suspicious editor drop at the first force-save

Two intertwined process-friction observations on `asset.save{force:true}` from a
clean material-authoring task:

**1. No cheap way to know an asset is already clean/on-disk, so agents over-call
defensive force-saves.** After a flawless 25-call graph/compile/instance run
(`compile_material` returned `compiledWithErrors:false`, `saved:true`;
`create_material_instance` used `save:true`; both verified by readback), the
caller still issued **3 trailing redundant `asset.save{force:true}` calls** on
the two materials "to be safe." There is no read surface that cheaply reports
"this package is not dirty / already on disk," so an agent that wants persistence
certainty defaults to force-saving everything it touched — wasted calls on
already-persisted (or already-clean-in-memory) packages. `asset.save` should
**no-op and report `alreadySaved:true` / `dirty:false`** when the package is not
dirty, instead of forcing a write; that single signal removes the defensive
over-call entirely. (Cross-ref `F-list-dirty-packages` for the read-side
"what's dirty" surface, and `F-asset-save` which defines the handler.)

**2. The editor went connection-refused exactly at the first force-save (possible
crash — causation UNCERTAIN, needs a controlled repro).** The editor had been
fully responsive through **27 prior RPCs including a heavy shader compile**, then
the very first `asset.save{force:true}` — and both retries — returned
`Editor not reachable at http://127.0.0.1:24966/mcp … (WinError 10061, connection
refused)`. The timing is suspicious: the drop coincides exactly with the first
force-resave of an already-saved material. This MAY be (a) a coincidental editor
death, or (b) `asset.save` force-resaving an unmodified/already-saved package
bringing the editor down. **This is not yet confirmed** — the only cold-restart
check in the sibling investigation (`B-material-authoring-save-no-disk-write`)
used a *fresh* editor running `open_asset`, which stayed responsive, so the
force-resave-crash hypothesis was never actually exercised.

## What it should do

- **Repro first (gates severity):** call `asset.save{force:true}` on a
  freshly-saved, unmodified material in a controlled session and confirm whether
  the editor survives. If it crashes, this is a Critical save-path crash and the
  save path must be hardened; if it survives, axis (2) drops and only the
  ergonomic axis (1) remains.
- **Ergonomic (independent of the crash question):** `asset.save` no-ops on a
  clean (non-dirty) package and reports `alreadySaved:true` / `dirty:false`
  rather than forcing a write, so agents stop defensively force-saving.

**Workaround:** skip the trailing force-saves entirely when
`compile_material`/`create_material_instance` already reported `saved:true` —
though note that in this task those success reports were themselves unreliable
(see `B-material-authoring-save-no-disk-write`: the material save helpers are a
mark-dirty no-op), which is exactly *why* the agent reached for the defensive
force-saves in the first place.

severity rationale: impact=soft-blocker (defensive over-call = many extra calls;
no dirty/clean readback signal) plus an unverified possible-crash that needs a
controlled repro × reach=common (asset.save is a general persistence verb) ->
Medium; escalate to Critical if the force-resave crash reproduces.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of a clean glowing-sci-fi-panels material.authoring task (focus null, namespace material.authoring, outcome tool_bug; 28 calls). After a first-try graph/compile/instance build whose creating RPCs reported `saved:true`, the caller issued 3 trailing redundant `asset.save{force:true}` calls on `/Game/SciFi/Materials/M_EnergyPanel` and `MI_EnergyPanel_RedAlert`; the first and both retries failed with `Editor not reachable … (WinError 10061, connection refused)` after 27 healthy RPCs incl. a heavy shader compile — suspicious timing, causation UNCERTAIN (coincidental editor death vs. force-resave crash). The sibling `B-material-authoring-save-no-disk-write` cold-restart used a fresh editor running open_asset (stayed responsive), so the force-resave-crash path was never exercised — this needs its own controlled repro. Two axes: (1) ergonomic — no cheap already-clean/on-disk signal, so agents over-call defensive force-saves; `asset.save` should no-op + report `alreadySaved:true`/`dirty:false` when not dirty (cross-ref `F-list-dirty-packages`, `F-asset-save`); (2) possible-crash — repro `asset.save{force:true}` on a freshly-saved material to confirm. Distinct from the judge-filed `B-material-authoring-save-no-disk-write` (that ticket is the mark-dirty-no-op SaveMaterialAsset trio not reaching disk; this is the generic `asset.save` handler going unreachable on force-resave + the missing clean-check). Dedup: ripgrep over OPEN + closed found no asset.save crash/force-resave/clean-check ticket (the one WinError-10061 board hit is `B-texture-normal-from-height-crash`, unrelated). severity rationale: impact=soft-blocker + unverified possible-crash × reach=common -> Medium; escalate to Critical if the crash reproduces.
- `#2-defer-behind-f-asset-save` `OPEN` developer — DEFER (gate `blockedBy: [F-asset-save]`). The only actionable content here is axis 1: a `Package->IsDirty()` short-circuit in `Handlers/Asset/AssetSaveHandler.cpp:58` that no-ops the force-write and reports `alreadySaved:true`/`dirty:false`. That is a follow-on modification to the exact same unmerged handler that `F-asset-save` (IN-REVIEW) just landed, whose `force` contract ("mark dirty + force write", AssetSaveHandler.cpp:58-62, SaveLoadedAsset(bOnlyIfIsDirty=!bForce) at AssetUtils.cpp:670) E reworks — reworking that file before F-asset-save is verified-green/DONE would collide with in-flight work, so defer until F-asset-save reaches DONE/WONTFIX, then re-evaluate. Axis 2 (the "possible force-resave crash") stays parked: no source-level crash vector in the force path (a plain headless MarkPackageDirty()+SaveLoadedAsset), WinError 10061 is generic editor-death, and the only crash+10061 board hit is the unrelated `B-texture-normal-from-height-crash` — it does not drive severity or the disposition. The defensive over-call that produced this observation is driven by the separate in-review Critical `B-material-authoring-save-no-disk-write` (material save helpers' mark-dirty no-op made success reports untrustworthy), not by any asset.save defect. Released the fuzz1 claim.
