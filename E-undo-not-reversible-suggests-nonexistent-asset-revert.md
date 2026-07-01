---
id: E-undo-not-reversible-suggests-nonexistent-asset-revert
title: "UNDO_NOT_REVERSIBLE error directs caller to non-existent asset.revert RPC"
status: OPEN
severity: Low
category: ergonomic
tags: [bpir, undo, error-message]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# UNDO_NOT_REVERSIBLE error directs caller to non-existent asset.revert RPC

When `blueprint.undo_last_bpir` correctly refuses to half-rollback a compile that
ran Phase 0 sweeps, it emits a `UNDO_NOT_REVERSIBLE` error whose remediation text
points the caller at two alternatives — one of which (`asset.revert`) does not
exist anywhere in the MCP surface:

```
[UNDO_NOT_REVERSIBLE] undo_last_bpir cannot reverse Phase 0 sweeps from the
previous compile. The compile ran in append/replace mode and deleted pre-existing
entry subgraphs that aren't snapshotted for restoration. Use git revert or
asset.revert instead.
```

The refusal itself is correct and intended (see `B-undo-last-bpir-doesnt-restore-phase0-sweeps`,
DONE — the fix deliberately refuses rather than leaving the asset half-rolled-back).
The ergonomic defect is narrower: the verbatim string `Use git revert or asset.revert instead`
names `asset.revert` as a fallback, but the `asset.*` namespace has no such method.
The full method list on `call("asset")` contains `asset.delete`, `asset.duplicate`,
`asset.rename`, `asset.validate`, etc. but nothing named `revert`. A caller who
follows the instruction will issue `asset.revert` and get a clean "method not found"
error, sending them in a circle. The only working fallback the message offers is
`git revert` (which on this LFS-backed content host means `git checkout` of the
on-disk `.uasset`).

This was already flagged in the resolved ticket's own verification note
(`B-undo-last-bpir-doesnt-restore-phase0-sweeps` history `#3-verified-undo-refused-on-replace`:
"ticket text references `asset.revert` as the suggested alternative, but no `asset.revert`
RPC exists ... Worth filing a follow-up to wire `asset.revert` (or update the error
message to drop the unimplemented suggestion)."), but no follow-up ticket was filed
and the error string is unchanged.

**What it should do:** either (a) drop `asset.revert` from the message and name only
working remediations (`git revert` / `git checkout` of the asset, or editor-side asset
reload), or (b) if an `asset.revert` capability is intended, file/implement it under
`F-` and keep the suggestion. Until one of those lands, the message advertises a
remedy that cannot be invoked.

**Fix:** edit the `UNDO_NOT_REVERSIBLE` `SendError` string at
`BpirCompilerHandler.cpp:747-751` to remove `or asset.revert` (or replace with a
real fallback). The same wording is mirrored in `docs/error-code-catalog.md` and the
`B-undo-last-bpir-doesnt-restore-phase0-sweeps` ticket body; update those for
consistency if the text changes.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed on `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic`: `blueprint.add_event` (custom FlickerOnce @(-224,700)) then `blueprint.compile_bpir` (default append mode, body `entry custom_event FlickerOnce() { call PrintString(InString:"Flicker!"); call SetActorHiddenInGame(bNewHidden:true); call SetActorHiddenInGame(bNewHidden:false) }`, returned `{nodeCount:4, success:true}`, Phase 0 swept the pre-existing add_event entry) then `blueprint.undo_last_bpir {assetPath: .../BP_Light_Bulb_Basic}` returned verbatim `[UNDO_NOT_REVERSIBLE] undo_last_bpir cannot reverse Phase 0 sweeps from the previous compile. The compile ran in append/replace mode and deleted pre-existing entry subgraphs that aren't snapshotted for restoration. Use git revert or asset.revert instead.` Confirmed via `call("asset")` method list that `asset.revert` is not a registered RPC — the suggested remedy is uninvokable. Refusal behaviour is correct (intended per the DONE ticket); only the dangling `asset.revert` suggestion is the issue.
