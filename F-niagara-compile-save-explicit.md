---
id: F-niagara-compile-save-explicit
title: "No standalone niagara.compile / niagara.save / niagara.mark_dirty (only as side effects of edits)"
status: DONE
severity: Medium
category: feature
tags: [niagara, compile, save, dirty, workflow]
---

# No standalone compile / save / dirty management for Niagara assets

Every `niagara.*` edit RPC accepts optional `compile` and `save` boolean
flags. There is **no standalone `niagara.compile`, `niagara.save`, or
`niagara.mark_dirty`**. Confirmed by `call("niagara.compile")` returning
NOT FOUND.

Comparable surfaces:

- Blueprint has `blueprint.compile` as a standalone RPC.
- Material has BPIR-flavored compile via `material.compile_mgir`; no
  plain `material.compile`.
- Niagara has no standalone compile.
- **Save** is global-only: `editor.save_all` (every dirty package) and
  `level.save` (active world). There is no per-asset `asset.save`,
  `blueprint.save`, or `material.save` either — so the save half of
  this ticket is really a plugin-wide gap, not niagara-specific.

## Why it matters

Workflow shapes that need this:

1. **Batch edits, single compile.** Most edit RPCs do N independent
   mutations. With `compile:true` on each, the system recompiles after
   every edit — slow for a deep stack. The clean shape is
   `compile:false` on every edit + one trailing `niagara.compile` call.
   Today the trailing call has to be a no-op edit (e.g.
   `set_stack_enabled` to the existing value) just to trigger compile.
2. **Inspect-only validation.** After editing without save/compile, the
   agent wants to compile to validate (`niagara.validate` returns
   warnings only if compile state is populated; see the on-demand
   compile note in the wiki). No way to trigger that explicitly.
3. **Save batch.** When editing multiple emitters in a system, the
   agent wants to compile the whole system once and save once. Today
   `save:true` saves after every edit — works but generates spurious
   `Saved/` writes.
4. **Force-dirty without edit.** A workflow that wants to reload the
   asset from disk after an external edit may need to clear the
   in-memory state — there's no `niagara.mark_dirty` to surface this.

The wiki specifically calls out that UE 5.6 ships with
`fx.Niagara.OnDemandCompile=1` so freshly-loaded assets are uncompiled
until "open in editor or spawn". An explicit `niagara.compile` would let
agents trigger this without going through `effect.spawn_actor` (which
also creates a world actor).

## Workaround today

- Use `compile:true` / `save:true` flags on edits — works but couples
  workflow with edit ordering.
- Trailing no-op edit purely to flip the compile flag.
- `effect.spawn_actor` to force compile (creates a world actor).
- `python.execute` calling
  `UNiagaraSystem::RequestCompile(/*bForce=*/true)`.

The wiki warns explicitly that synchronous `RequestCompile` from inside
the dumper would mutate persistent state (`UNiagaraScript::CachedScriptVM`
is non-Transient), so an `niagara.compile` RPC must use the *async*
compile path or rely on the existing on-edit compile machinery — not
the dump-time read path.

**Implemented v1 (reshape):** Shipped `niagara.compile` (sync-only) and
`niagara.save` as standalone RPCs in `NiagaraCompileHandler.cpp`.
`niagara.mark_dirty` was dropped — the tester review (#2) already flagged
it as low-value, and removing it from scope keeps the surface minimal.
`niagara.compile(wait:false)` returns `INVALID_PARAMS` with message
"async compile (wait:false) not yet implemented"; it is deferred pending a
Niagara compile-completion observer that can surface a job-id without
blocking the game thread. Both handlers resolve assets via
`LoadObject<UNiagaraSystem>` then `LoadObject<UNiagaraEmitter>` and reuse
`NiagaraEdit::FinalizeNiagaraEdit` (compile path) and `McpSafeAssetSave`
(save path). Tests in `TestNiagaraCompileSave.cpp` assert dispatcher
registrations match this reshape and run counterfactual anchors against
`FinalizeNiagaraEdit` directly.

## Proposal

```
niagara.compile(
    assetPath: string,
    force?: boolean,            // default false; true bypasses up-to-date check
    wait?: boolean              // default true; false returns immediately with a job id
) -> {
    compiled: true,
    status: "Success"|"Error"|"Warning"|"Unknown",
    issues: [...],
    durationMs: number,
    jobId?: string              // when wait:false
}

niagara.save(
    assetPath: string,
    force?: boolean             // save even if not dirty
) -> { saved: true, package: string, sizeBytes: number }

niagara.mark_dirty(
    assetPath: string
) -> { dirty: true, package: string }
```

Implementation surface: `UNiagaraSystem::RequestCompile` (async path),
`UPackage::Save`, `UPackage::MarkPackageDirty`. Reuse the
session-scoped view-model cache established by
`F-niagara-event-handler-simstage-authoring`. For `wait:false`, route
through the existing job registry (`F-job-registry`).

## Cross-ref

- All `niagara.*` edit RPCs that take `compile?` / `save?` — these
  become orthogonal to the edit, not embedded in it.
- `F-job-registry` — `wait:false` mode uses the same job pattern.
- Wiki note: "Compile-state honesty" in `call("niagara")` — explains
  why compile state can be `null` after edits.

## History
- `#1-no-standalone-compile-save` `OPEN` reporter — Confirmed `niagara.compile`, `niagara.save`, `niagara.mark_dirty` don't exist as standalone RPCs. Only available as `compile:`/`save:` flags on edit RPCs, forcing trailing no-op edits to trigger compile-only flows and per-edit saves that generate spurious package writes. UE 5.6 `OnDemandCompile=1` makes the gap worse since freshly-loaded assets need an explicit compile to populate validation state. Proposes `niagara.compile` (sync or async via job id), `niagara.save`, `niagara.mark_dirty`. Must use the async compile path, not synchronous, per the wiki's caching warning.
- `#2-reviewed-and-confirmed` `OPEN` tester — Re-verified against source: no `niagara.compile|save|mark_dirty` handlers exist (grep `REGISTER_RPC_HANDLER("niagara.*` in `Source/Handlers/Niagara/`). `niagara.validate` reads cached compile state via `NiagaraDumpBuilder::BuildCompileJson`; does NOT trigger compile. Corrected misclaims in the body: there is no `asset.save`, `blueprint.save`, `material.save`, or plain `material.compile` either — only `blueprint.compile`, `material.compile_mgir`, `editor.save_all`, and `level.save`. The save half of this ticket is really plugin-wide; consider splitting into a generic `asset.save` ticket. Severity Medium upheld for the compile gap (`niagara.compile`): existing workarounds (no-op edit, `effect.spawn_actor` creating a world actor, `python.execute` of `RequestCompile`) are awkward and the OnDemandCompile=1 case is real. `mark_dirty` is low-value and could be dropped from scope.
- `#3-reshape-and-implement` `IN-REVIEW` developer — Implemented `niagara.compile` and `niagara.save` only; dropped `niagara.mark_dirty` (low value) and deferred `wait:false` (needs Niagara compile-completion observer). Both new handlers in `NiagaraCompileHandler.cpp` reuse `NiagaraEdit::FinalizeNiagaraEdit` and `McpSafeAssetSave`. Tests assert dispatcher registrations match the reshape.
- `#4-verify-compile-save` `DONE` tester — Verified: `call("niagara.compile")` and `call("niagara.save")` document standalone RPCs, and live calls on `/Water/Effects/Niagara/Shoreline/NiagaraShore_System.NiagaraShore_System` returned `compiled: true`, `status: "requested"`, `saved: true`, package `/Water/Effects/Niagara/Shoreline/NiagaraShore_System`, and `sizeBytes: 702088`.
