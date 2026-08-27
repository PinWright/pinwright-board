---
id: B-niagara-edits-lost-while-emitter-toolkit-open
title: "Every niagara.* write against an emitter asset is silently discarded if its toolkit is open, because the toolkit edits a duplicate and overwrites the original on Apply"
status: OPEN
severity: High
category: bug
tags: [niagara, emitter, asset-editor, silent-data-loss, shared-editor, toolkit, multi-agent]
encounters: 1
lastSeen: 2026-08-28
---

# The toolkit edits a copy, then overwrites the original wholesale

`FNiagaraSystemToolkit::InitializeWithEmitter` (`:214-240`) builds a **transient system** and edits a
**duplicate** of the emitter asset. `UpdateOriginalEmitter` (`:1149`) then overwrites the original
wholesale when the user hits Apply.

So any `niagara.*` write against an emitter asset path while that emitter's toolkit is open lands on
the original, is not seen by the toolkit, and is **destroyed by the next Apply**. The write reports
success and nothing anywhere reports the loss.

This is distinct from `B-niagara-edit-with-open-asset-editor-slate-crash`, which is about a system
toolkit holding stale pointers and crashing. This one does not crash — it silently discards work, which
in a shared editor means somebody else's Apply eats your edits.

**Scope note:** the existing `EDITOR_OPEN` guard is per-verb and keyed on hazard, deliberately, because
28 of 32 Niagara mutators must not take it. This is different: it is an **asset-kind-level** condition —
any write to an emitter asset with an open emitter toolkit is unsafe regardless of which verb makes it.
So it wants a guard at the resolve step, not per verb.

**Not reproduced** — read from toolkit source while classifying the per-verb guard. Confirm the Apply
path actually clobbers before choosing the fix; if it merges rather than overwrites, the severity drops.

## History
- `#1-found-while-classifying-the-guard` `OPEN` reporter — Found by the agent that classified all 32
  Niagara mutators for `B-niagara-editor-open-guard-missing-on-mutators`, which scoped itself to
  per-verb hazards and flagged this as the broader asset-kind case.
