---
id: B-niagara-editor-open-guard-missing-on-mutators
title: "Only add_emitter and remove_emitter carry the EDITOR_OPEN guard; four other niagara.* mutator files have none"
status: OPEN
severity: High
category: bug
tags: [niagara, editor-open-guard, crash-adjacent, slate, multi-agent, shared-editor, EDITOR_OPEN]
encounters: 1
lastSeen: 2026-08-27
---

# The open-asset-editor guard covers two verbs out of a family

`B-niagara-edit-with-open-asset-editor-slate-crash` is fixed for `niagara.add_emitter` and
`niagara.remove_emitter`: `Handlers/Niagara/NiagaraEditorOpenGuard.h` refuses with `EDITOR_OPEN` when
any asset editor holds the target open, because reshaping the emitter-handle array underneath a live
`FNiagaraSystemToolkit` faults on the next Slate redraw.

The guard is asset-class-agnostic (`UObject*`) and adopting it is a one-line call, but four other
mutator files still have none: `NiagaraEditHandler.cpp`, `NiagaraAdvancedEditHandler.cpp`,
`NiagaraCurveHandler.cpp`, `NiagaraGraphHandler.cpp`.

Whether each of those verbs can actually fault an open toolkit is **not established** -- the proven
crash is specifically the emitter-handle reshape. A structural graph edit under an open Niagara editor
is at minimum in the same neighbourhood, and the guard costs one line. This should be decided per
verb rather than blanket-applied: refusing a harmless edit because a toolkit is open is its own
ergonomic cost, and in a shared editor the toolkit is often someone else's.

**Related, worth doing at the same time:** the guard duplicates
`PinWright::Material::IsMaterialEditorOpen` in shape. Consolidating both into one plugin-wide
`EDITOR_OPEN` guard is close to a rename and would put every namespace on the same behaviour.

## History
- `#1-scope-note-from-the-fix` `OPEN` reporter -- Raised by the agent that fixed
  `B-niagara-edit-with-open-asset-editor-slate-crash`, which deliberately scoped itself to the two
  emitter-handle mutators the crash was proven on. Source-level claim; no crash reproduced on the
  other four files.
