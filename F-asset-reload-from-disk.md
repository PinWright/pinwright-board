---
id: F-asset-reload-from-disk
title: "No RPC to force-reload an asset from disk — persistence/corruption verification can only read the live in-memory object"
status: IN-REVIEW
severity: Medium
category: feature
tags: [asset, verification, cold-load, persistence, corruption, in-memory-vs-disk]
encounters: 1
lastSeen: 2026-06-28T14:47:56Z
---

# No RPC to force-reload an asset from disk for persistence verification

## What's missing

Every read RPC — `blueprint.decompile`, `blueprint.graph.get_graph_connections`,
`blueprint.graph.find_orphaned_nodes`, `blueprint.inspect`, the property
getters — reflects the **live in-memory `UObject`**, never a fresh read of the
persisted bytes. After `asset.save`, there is no way within one editor session
to **evict the package and re-load it from disk**, so an agent cannot prove that
the saved `.uasset` reloads to the same graph it just wrote.

This is the single most common verification dead-end in the test/fix workflows:
the goal "save the Blueprint so a cold reload can confirm it is not corrupted"
has **no backing RPC**. The Attempt can only approximate it with
`asset.save force:true` → `blueprint.decompile`, which re-reads the same
in-memory object and therefore **cannot** detect corruption that manifests only
on a fresh load.

## What it should do

Add `asset.reload` (e.g. `asset.reload` / `blueprint.reload_from_disk`):
force-evict the loaded package (`UPackageTools::ReloadPackages` /
`ReloadPackage`) and re-read it from disk, so a subsequent
`blueprint.decompile` / `blueprint.graph.*` reflects the **persisted bytes**
within the same editor session. Return whether the reload succeeded and whether
the post-load object still compiles / passes the integrity gate.

This directly unblocks a recurring blocker on the corruption tickets: testers on
`B-bp-saved-state-corruption-mcp-edits` repeatedly logged `SKIP` because
"cold-reload of a real consumer .uasset after editor restart is not testable
from a single MCP session" (history `#11`/`#12`/`#13`). An in-session reload RPC
would let that whole class of corruption regressions be verified without an
out-of-band editor restart.

## Evidence

From this task (`blueprint.compile_bpir` failure-rollback probe, transcript
`agent-ac9ff02aeac548e24.jsonl`, 20 call RPCs):
- The agent searched for a reload verb — `Glob **/*reload*.md` → "No files
  found" — then reasoned in THINK: "No dedicated package-reload RPC exists; I'll
  verify persistence via save + re-decompile + orphan scan."
- `asset.validate`'s doc only "Validate if an asset exists and can be loaded" —
  it does **not** evict the already-loaded package, so it cannot serve as a
  cold-load.
- Net effect: the post-save verification (`asset.save force:true` →
  `blueprint.decompile`) reflected the **in-memory** graph, not a fresh load of
  the persisted bytes — it could not have proven (or disproven) on-disk
  integrity. In this task the in-memory graph was already corrupt so the bug was
  caught anyway, but the verification path itself is blind to disk-only corruption.

## Severity

**Medium.** Soft blocker with a partial workaround (save + re-decompile reads
the live object, not disk) on a verification need that recurs in essentially
every save-then-confirm task and is the documented gating reason multiple
corruption tickets sit unverified. No crash, no data loss in the tool itself —
it is a missing read-side capability — so not High.

## History
- `#1-initial-feature-request` `OPEN` reporter — Filed from the CallAnalyzer trace of the `blueprint.compile_bpir` failure-rollback probe (`agent-ac9ff02aeac548e24.jsonl`). The task's explicit goal was to confirm a saved Blueprint is not corrupted on a cold reload, but no `asset.reload` / `blueprint.reload_from_disk` RPC exists: `Glob **/*reload*` → No files found; agent THINK "No dedicated package-reload RPC exists..."; `asset.validate` only validates load-ability of the already-loaded asset and does not evict. Every decompile/graph read reflects the live in-memory object, so persistence verification can only be approximated and silently misses disk-only corruption. Recurring blocker — see `B-bp-saved-state-corruption-mcp-edits` history `#11`/`#12`/`#13` (testers SKIP because cold-reload is not testable in one MCP session).
- `#2-implement-asset-reload` `IN-REVIEW` developer — GO (all three lenses valid; no dup/regression/prior-art). Confirmed absent in current source: no `asset.reload`/`ReloadPackages` usage anywhere, and `asset.validate` (AssetManageHandler.cpp:1145) only re-`LoadAsset`s the resident object. Added the `asset.reload` RPC (category `asset`) in `Source/PinWright/Private/Handlers/Asset/AssetManageHandler.cpp`: force-evicts the resident package via `UPackageTools::ReloadPackages(..., AssumePositive)` and re-serializes it from the on-disk `.uasset` (falls back to `LoadPackage` when the package isn't resident); refuses the active editor level package with `CANNOT_RELOAD_ACTIVE_LEVEL` (steers to `editor.open_level`); returns `reloaded`/`wasLoaded` and, for a Blueprint, `compiled` + `blueprintStatus`. Regression test `Source/PinWright/Private/Tests/Assets/TestAssetReloadHandler.cpp` (`PinWright.asset.reload`) saves a SoundClass with Volume=0.5, mutates the resident object to 0.123 in-memory without saving, calls `asset.reload`, and asserts the reloaded object reads back the on-disk 0.5 — fails if the verb is removed or weakened to a validate-style re-`LoadAsset`. Also covers registration/category, missing-param `INVALID_ARGUMENT`, and `ASSET_NOT_FOUND`.
