---
id: B-crir-replace-duplicates-hierarchy
title: "CRIR compile mode=replace never clears the rig hierarchy — recompiling decompiled text silently duplicates every element (_2) and a save=true recompile persists corruption"
status: IN-REVIEW
severity: High
category: bug
tags: [control-rig, crir, hierarchy, roundtrip, replace, duplication, idempotency, saved-state-corruption]
---

# CRIR compile mode=replace never clears the rig hierarchy — recompiling decompiled text silently duplicates every element

`controlrig.compile_crir` is documented as round-trippable (decompile → compile
→ decompile is byte-equal — `crir-language-reference.md:489`, "the authoritative
correctness test for CRIR"). For the **graph** body, `mode=replace` honors that
by clearing first (`ClearGraphForReplace`). But the **hierarchy** path
(`rig_hierarchy { bone / null / control … }`) is **append-only in BOTH `extend`
and `replace` modes** — `ClearGraphForReplace` only wipes the RigVM graph, never
the `URigHierarchy`. The compiler drives `URigHierarchyController::Add*`
unconditionally; an element whose name already exists is **auto-renamed to
`<name>_2`** rather than upserted or rejected.

The practical consequence: feeding a rig's **own decompiled hierarchy** back to
`compile_crir` (the exact documented round-trip) does not reproduce the rig — it
**doubles** it. There is no warning, no `is_error`; the call returns success.

## Repro (live, this task)

Target `/Game/ExampleContent/ControlRig/Rigs/CR_FK` — baseline decompile reported
16 bones / 0 nulls / 6 controls. The user decompiled it, then recompiled that same
text in `extend` mode with `save=true`:

```
controlrig.compile_crir(context=CR_FK, mode=extend, text=<CR_FK's own decompile>, save=true)
  → ok=true, is_error=false              # silent success
controlrig.decompile_crir(context=CR_FK)
  → 32 bones / 16 controls, every element duplicated with a _2 suffix (22487 chars)
```

The asset was now **persisted** in the doubled state (`save=true`), and because
this host has **no editor SCC provider** (`source_control.get_provider` →
`providerName=None, isEnabled=false`) and the MCP exposes **no element-delete /
hierarchy-remove RPC**, there was no MCP-side path to restore CR_FK. The user had
to (a) read the plugin's `CRIRCompiler.cpp` to confirm replace never clears the
hierarchy (a last-resort source read), and (b) abandon the in-place round-trip
entirely, creating two throwaway scratch rigs (`CR_FK_RoundTrip`,
`CR_FK_RoundTrip2`) via `animation.authoring.create_control_rig` just to
demonstrate a byte-equal round-trip against fresh assets. CR_FK was left
duplicated-and-saved.

## Why this is a process trap (not just the forward-ref bug)

This is distinct from `B-crir-decompile-forward-ref-wire` (which is about
exec-**wire** ordering inside the `rig_graph`, a *parse* failure). Here the
**hierarchy** compile **succeeds** and silently corrupts the asset. The two
together make the round-trip doubly broken on a real rig: `mode=replace` fails on
the graph (forward-ref) *and* duplicates the hierarchy; `mode=extend` parses but
duplicates the hierarchy. Either way, recompiling a rig's own decompiled text —
the one operation CRIR exists for — corrupts the asset on a stock Content-Examples
rig with zero authoring intent.

There is direct precedent: `B-compile-bpir-retry-duplicates` was the identical
class of friction on the BPIR side (default/replace compile accumulated duplicate
entry nodes, corrupting BP state and requiring manual node-by-node cleanup). It
was fixed by making compile **idempotent** (Phase 0 signature-based upsert;
`mode=replace` an alias with clear-first semantics) and adding a duplicate-entry
warning scan. The CRIR hierarchy path never received that treatment.

## Impact

- The documented byte-equal round-trip is **unachievable in place** on any rig
  with a pre-existing hierarchy — the only safe target is a fresh/empty rig.
- `mode=replace` is a **lie** for the hierarchy: it does not replace, it appends.
- Silent + persisted: with `save=true`, the corruption is written to disk with no
  error or warning surfaced to the caller.

## What it should do

1. `mode=replace` should clear (or upsert) the hierarchy as well as the graph, so
   recompiling decompiled text reproduces the rig byte-for-byte (match the
   BPIR upsert precedent).
2. Same-name collisions in the hierarchy compile path should **upsert in place**
   (or hard-error with a precise code), never silently auto-rename to `_2`.
3. At minimum, emit a duplicate-element **warning** in the compile response when
   `Add*` had to rename — so a silent doubling becomes visible (mirrors BPIR's
   duplicate-entry warning scan).

**Workaround (this task):** round-trip only against a fresh
`create_control_rig` asset; never recompile a populated rig's hierarchy in place.

**Secondary gap (contributing factor, not the root cause):** there is no
MCP-side `controlrig`/`animation.authoring` element-delete or hierarchy-remove
RPC, and this host has no SCC provider, so a bad hierarchy mutation has **no
recovery path** through the MCP. Fixing the duplication root cause removes most
of the need for delete, but a hierarchy-remove RPC is independently useful for
the "clean up a rig as text" workflow this task was exercising.

## History
- `#2-crir-replace-hierarchy-upsert` `IN-REVIEW` developer — Fixed the root cause: the hierarchy compile path now honors `Options.Mode`. `CompileRigHierarchyBlock` (`Source/EditorAutomationRpcGateway/Private/CRIR/CRIRCompiler.cpp`) gained a `const FCRIRCompileOptions&` parameter (caller `CompileHierarchyOrFunction` updated to pass `Options`). In `Replace` mode, before each `Controller->Add*`, a new `FindExistingHierarchyElement` helper looks up any live element of that name (bone/null/control/curve/socket) and `Controller->RemoveElement`s it first — clear-first/upsert per element, mirroring the graph path's `ClearGraphForReplace` and the BPIR Phase 0 upsert precedent (`B-compile-bpir-retry-duplicates`). So recompiling a rig's own decompiled hierarchy in place reproduces it instead of doubling to `<name>_2`. Added the silent-collision floor for BOTH modes: after a successful add, if `AddedKey.Name != ElemFName` (GetSafeNewName had to rename), a `CRIR_HIERARCHY_DUPLICATE_ELEMENT` warning is emitted (mirrors BPIR's duplicate-entry warning scan), so a doubling is never silent. Fresh-target round-trips are unaffected (empty target → no collision → no removal/rename/warning), so existing CRIR round-trip tests are backward-compatible. Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestCRIRReplaceHierarchyIdempotent.cpp` (`EditorAutomationRpcGateway.CRIR.RoundTrip.ReplaceHierarchyIdempotent`) builds a 5-element rig (root/child bone, null, control, curve), decompiles it, recompiles that text into the SAME asset in `Replace` mode, and asserts the element count stays at 5 (pre-fix: 10), no `_2` element exists, no spurious duplicate warning fires, and decompile #2 == decompile #1 byte-for-byte — it fails if the clear-first/upsert is reverted. Did not address the secondary gap (no MCP element-delete/hierarchy-remove RPC) — that is an independently-useful follow-up, not the root cause. NOTE: did not compile/run tests (later phase).
- `#1-initial-audit` `OPEN` reporter — Process audit of a controlrig round-trip task (21 calls). The CRIR hierarchy compile path is append-only in both `extend` and `replace` (`ClearGraphForReplace` wipes only the graph; `URigHierarchyController::Add*` auto-renames same-name elements to `_2`). Friction note: recompiling CR_FK's own decompiled text in extend mode with `save=true` silently grew it 16→32 bones / 6→16 controls (`_2` duplicates, 22487-char decompile) and persisted the corrupt asset; with no editor SCC (`source_control.get_provider` → providerName=None) and no MCP element-delete RPC, CR_FK could not be restored through the MCP, so the user fell back to two scratch `create_control_rig` assets and a last-resort `CRIRCompiler.cpp` source read to confirm the cause. Distinct from `B-crir-decompile-forward-ref-wire` (graph exec-wire parse failure) — this is a silent, save-persisted hierarchy duplication. Direct precedent: `B-compile-bpir-retry-duplicates` fixed the same class on BPIR via Phase 0 upsert idempotency + duplicate-entry warning; the CRIR hierarchy path needs the analogous clear/upsert + warning.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
