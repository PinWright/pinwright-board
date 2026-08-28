---
id: B-niagara-refused-edit-dirties-package
title: "Every error return inside set_module_input's transaction happens after Graph->Modify(), so a refused call dirties the package with no change"
status: IN-REVIEW
severity: Medium
category: bug
tags: [niagara, set_module_input, transaction, package-dirty, refused-write, side-effect]
encounters: 2
lastSeen: 2026-08-28T08:30:00+05:00
---

# A refusal still marks the asset dirty

In `niagara.set_module_input`, every error return inside the `FScopedTransaction` block happens
*after* `ModifyResolvedTarget(Target)` and `Target.Graph->Modify()`. So a call that is refused --
`INVALID_STACK`, `PARAMETER_TYPE_MISMATCH`, `UNSUPPORTED_INPUT_VALUE`, and now
`MODULE_INPUT_OVERRIDE_LINKED` -- leaves the package dirty having changed nothing.

The consequence is not cosmetic in a shared editor: a later legitimate save of that package persists
whatever else was pending, and a caller checking "is this asset dirty" to decide whether its own write
landed gets a false positive from somebody else's rejected call.

**Fix:** move the guards ahead of the transaction. That fixes all four refusal paths at once and is
the structural version rather than one `Modify()` audit per error return. It restructures the shared
handler prologue, which is why the agent that hit it did not do it -- two other agents were editing
that function at the time.

## History
- `#1-pre-existing-made-more-reachable` `OPEN` reporter -- Recorded by the agent fixing
  `B-niagara-literal-over-linked-override-pin`, whose new refusal joins the three existing ones on the
  same path. Pre-existing, not introduced by that fix. Source-level claim.

- `#2-still-reproduces` `OPEN` verifier — 2026-08-28. Plugin rebuilt from a clean tree at `b79ba53e` and verified against disk, not against the build's own success message: `UnrealEditor-PinWright.dll` 39,898,624 -> 40,644,096 bytes at 2026-08-28 08:11:48, `UnrealEditor-PinWrightGeometry.dll` 4,983,296 -> 5,113,344, canonical link with no `-000N` artifacts in `UnrealEditor.modules`. Editor restarted on that DLL and the ticket's own repro re-run. Still reproduces on the rebuilt plugin at `b79ba53e`; untouched by this batch. Two `niagara.graph.create_node` calls that were **refused** - one `[UNSUPPORTED_NODE_CLASS]`, one `[INVALID_OP]` - left `/Game/PinWrightScratch/NS_TplProbe` listed in `editor.list_dirty_packages`, alongside no other write to that asset in the session. Noticed while confirming the `B-niagara-create-node-unfinalized-graph-node-creator-fatal` fix: the crash is gone, the spurious dirty is not. Consequence worth naming - a refused edit makes the next `editor.quit` or restart prompt about, or silently discard, an asset the caller never successfully modified.

- `#3-dirty-baseline-restore-on-refusal` `IN-REVIEW` developer — "Added `FNiagaraCleanPackageBaseline` to NiagaraEditHandler.cpp: `set_module_input` now samples the resolved target's package dirty flags after `ValidateModulePayload` and before the transaction, captures `ApplyModuleMutation`'s error instead of sending it inline, and restores the previously-clean packages after the transaction closes — one restore covering all four refusal codes (and any future one) rather than a per-error-return `Modify()` audit. Mirrored in NiagaraGraphHandler.cpp's `create_node` `PayloadError` path (`INVALID_OP`), which now restores the graph package's dirty flag beside the existing `Graph->RemoveNode`. NOT the hoist-all-guards-ahead-of-the-transaction shape the ticket proposed: the guards are interleaved with the mutations inside the shared `ApplyModuleMutation`, so hoisting them means duplicating ~150 lines of stack/type/binding resolution at a second call site. `UNSUPPORTED_NODE_CLASS` from `#2` was already refused before the transaction opens and cannot be the source of that dirt. Regression tests: `PinWright.niagara.set_module_input.RefusalLeavesPackageClean` and `PinWright.niagara.graph.create_node.RefusalLeavesPackageClean` — both clear the fixture package's `RF_Transient` flag first, without which `UObjectBaseUtility::MarkPackageDirty` bails on the outer-chain walk and the bug cannot reproduce in a test at all."
