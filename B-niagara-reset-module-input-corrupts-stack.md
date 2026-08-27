---
id: B-niagara-reset-module-input-corrupts-stack
title: "niagara.reset_module_input on a dynamic-input override leaves the emitter's ParticleSpawn stack unusable — every later set_module_input on that stage fails INVALID_STACK, and the only recovery found was remove_emitter + add_emitter + a full stack rebuild"
status: OPEN
severity: High
category: bug
tags: [niagara, reset_module_input, dynamic-input, override-pin, stack-corruption, invalid-stack, unrecoverable, unverified-mechanism]
encounters: 1
lastSeen: 2026-08-27T18:56:59+05:00
---

# `niagara.reset_module_input` on a dynamic-input override appears to destroy the ParticleSpawn stack, not just the override

Resetting a module input whose override is a **dynamic-input chain** left the
emitter in a state where **every subsequent `niagara.set_module_input` against any
ParticleSpawn module returned `INVALID_STACK`**, while ParticleUpdate modules kept
accepting writes normally. No sequence of further `niagara.*` calls recovered the
emitter. The only fix found was `niagara.remove_emitter` + `niagara.add_emitter`
and re-authoring the whole stack.

Setting a **literal** over a dynamic input does **not** trigger it (that path has
its own, different defect — see `B-niagara-literal-over-linked-override-pin`).
It is `reset_module_input` specifically.

## What was observed, and what was NOT

Observed 2026-08-27, UE 5.8, in this checkout (PinWright at `8e76cad5`):

- `reset_module_input` was called on an input whose override was a dynamic-input chain.
- Afterwards, `set_module_input` on **any** ParticleSpawn module of that emitter
  returned `INVALID_STACK`; ParticleUpdate was unaffected.
- The emitter never recovered; the stack was rebuilt from scratch.

**NOT observed** — the graph was never dumped after the reset. The session log
describes the ParticleSpawn `UNiagaraNodeInput` being "deleted outright, orphaning
that stack's `ParameterMapSet`", but **no `niagara.decompile_nir` /
`niagara.inspect includeGraphs` readback was taken to confirm any node was actually
removed.** That description is an inference from the symptom, not a reading of the
graph. Treat it as such.

## The `INVALID_STACK` evidence does not by itself prove corruption — run this control first

`INVALID_STACK` is raised from exactly one place,
`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp:755-757`:

```cpp
        if (SourceGroupIndex <= 0 || SourceGroupIndex >= Groups.Num() - 1)
        {
            return FNiagaraEditError::Make(TEXT("INVALID_STACK"), FString::Printf(TEXT("Module '%s' is not in a valid stack group."), *Payload.Target.EntryId));
        }
```

`SourceGroupIndex` comes from `Groups.IndexOfByPredicate(...)` at `:750`, so the
guard fires **identically** for `INDEX_NONE` (module not present in the resolved
stack at all) and for a genuinely malformed group. **Two different causes produce
the same error string**, and the session did not separate them:

1. **The already-known cause** — `B-niagara-module-input-stack-infer` (IN-REVIEW,
   High): with `scriptUsage` omitted the handler used to default to
   `ParticleUpdateScript`, so ParticleSpawn modules were "not found" and reported
   `INVALID_STACK` while ParticleUpdate worked. **That is exactly the
   ParticleSpawn-fails / ParticleUpdate-works split reported here.**
2. **Actual stack corruption** — a missing or orphaned node makes the same lookup miss.

Two things keep (1) from being a complete explanation, but neither settles it:

- That ticket's fix **is already in this tree**: `NiagaraEditHandler.cpp:733-737`
  infers the owning stack via `FindOwningStackOutputNode(Target)` whenever
  `Payload.Target.ScriptUsage` is empty, so an omitted `scriptUsage` should now
  resolve ParticleSpawn correctly.
- But `FindOwningStackOutputNode` (`:582`) walks the module's links to its owning
  output node — **on a corrupted stack that walk fails too**, falls through to
  `ResolveStackOutputNode` (which defaults to ParticleUpdate) at `:740`, and lands
  on the same `:755` guard. So corruption also reproduces symptom (1).

**The control that separates them was never run, and must be run before this ticket
is worked:** reset a dynamic-input override, then retry `niagara.set_module_input`
on a ParticleSpawn module **with an explicit `scriptUsage:"ParticleSpawnScript"`**.

- Still `INVALID_STACK` with explicit `scriptUsage` → the stack is genuinely broken;
  this ticket is real.
- Succeeds with explicit `scriptUsage` → the observation is
  `B-niagara-module-input-stack-infer`, not corruption, and this ticket should be
  closed against it.

Pair it with a `niagara.decompile_nir` before and after the reset and diff the
`graph ParticleSpawn*` block for a disappeared node — that turns the inference into
a reading.

## Candidate mechanism — HYPOTHESIS, explicitly unverified

If a node really is being deleted, the machinery it would live in is
`NiagaraResetModuleInput::RemoveOverridePinAndChainedNodes`
(`NiagaraEditHandler.cpp:2040-2119`), which `reset_module_input` reaches on its
override-pin path. It collects every node reachable **upstream** of the override pin
and removes them all:

```cpp
                NodesToRemove.Add(UpstreamNode);                        // :2088
                ...
                        PinsToVisit.Append(InputPin->LinkedTo);         // :2096
        ...
        Graph.GetSchema()->BreakPinLinks(OverridePin, true);            // :2102
        ...
            OverrideNode->RemovePin(&OverridePin);                      // :2108
        for (UNiagaraNode* NodeToRemove : NodesToRemove)                // :2111
            NodeToRemove->GetGraph()->RemoveNode(NodeToRemove);         // :2116
```

The walk at `:2096` follows **every** input pin of every visited upstream node, with
**no check that the node it collects belongs exclusively to this override chain**.
A dynamic-input function-call node has its own parameter-map input; if that input is
wired back into the stack, the walk would follow it out of the override chain and
into the stack proper, and `:2116` would delete stack nodes. That would match the
reported symptom exactly — but **this has not been traced, stepped, or reproduced
against the graph.** It is where to look, not a finding.

Corroborating context, not proof: `F-niagara-reset-module-input` (DONE) shipped this
exact function, and its own `#4-fix-review-issues` records that it added "`AddInfo`
for **deferred** `RemoveOverridePinAndChainedNodes` integration coverage" — i.e. the
removal walk went out with its integration coverage explicitly postponed. Its two
`DONE` verifications (`#5`, `#6`) both exercised the **static-switch** reset path,
not the override-pin removal path. So the code this ticket suspects is the one piece
of that feature that shipped unverified end to end.

## Verbatim repro (shape; the failing call's arguments were not captured)

```
niagara.set_module_input   {assetPath, emitter, entryId, inputName,
                            value: {dynamicInput: "<dynamic input script path>"}}   -> success
niagara.reset_module_input {assetPath, emitter, entryId, inputName}                 -> success
niagara.set_module_input   {assetPath, emitter, <any ParticleSpawn module entryId>,
                            inputName, value: <literal>}
  -> [INVALID_STACK] Module '<entryId>' is not in a valid stack group.
```

Every later ParticleSpawn `set_module_input` on that emitter fails the same way;
ParticleUpdate modules still succeed.

The session log records no argument values for the `reset_module_input` call itself.
The nearest concrete instance of the required precondition (a ParticleSpawn module
input driven by a dynamic input) that the same session **did** capture is
`/Game/Atlantis/VFX/NS_Bubbles_Stream`, emitter `Bubbles`, ParticleSpawn module
`InitializeParticle` (`entryId 601B69A048DB6BF934AEE58DFF6C7378`), whose `Lifetime`
is a `RandomRangeFloat` and whose `Sprite Size` is a `RandomRangeVector2D` —
reproduce against a **duplicate** of that asset.

## Workaround

Do not call `niagara.reset_module_input` on an input whose override is a
dynamic-input chain. `niagara.clear_module_overrides` walks the same removal code
(`NiagaraEditHandler.cpp:3079` also calls `RemoveOverridePinAndChainedNodes`) and
should be assumed to carry the same risk until the control above is run. To change
such an input, `niagara.remove_module` + `niagara.add_module` + `niagara.move_module`
back into position, and set literals on the fresh node before attaching any dynamic
input.

## Distinct from related tickets

- `B-niagara-module-input-stack-infer` (IN-REVIEW, High) owns the
  "`set_module_input` on ParticleSpawn returns `INVALID_STACK` while ParticleUpdate
  works when `scriptUsage` is omitted" half, with its own cause
  (`ResolveStackOutputNode` defaulting to `ParticleUpdateScript`). **This ticket
  claims only the node-deletion / stack-corruption half.** As stated above, its
  symptom and this one's are currently indistinguishable, and the control that
  separates them has not been run. If the control clears `reset_module_input`, this
  ticket folds into that one.
- `F-niagara-reset-module-input` (DONE) built the verb. This is the first report of
  it damaging anything; its removal-walk coverage was explicitly deferred (`#4`).
- `B-niagara-literal-over-linked-override-pin` is the *other* defect on an input
  driven by a dynamic input: a literal write silently does nothing. Opposite failure
  mode (silent no-op vs loud unrecoverable), different code path (the literal branch
  at `:985-992`, the one branch that never calls the removal walk).

severity rationale: impact=authored emitter stack rendered unwritable with no in-API recovery, costing a full stack rebuild (asset-state damage, not merely a blocked call) x reach=`reset_module_input` is the documented way to undo an override, and dynamic inputs are how idiomatic Niagara content parameterizes module inputs -> High. Would be **Critical** if the node deletion is confirmed by a graph diff (a write that destroys authored asset data); held at High because the mechanism is unverified and the `INVALID_STACK` signal is currently ambiguous with `B-niagara-module-input-stack-infer`.

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at `8e76cad5` in this checkout. `niagara.reset_module_input` on an input whose override was a dynamic-input chain left every subsequent `set_module_input` on that emitter's ParticleSpawn modules returning `INVALID_STACK` while ParticleUpdate kept working; no `niagara.*` call recovered it, and the emitter was rebuilt via `remove_emitter` + `add_emitter`. **Filed for the corruption claim only.** Honesty bounds stated in the body: the "node deleted / `ParameterMapSet` orphaned" description is an inference — no post-reset graph readback was taken — and the `INVALID_STACK` signal is currently ambiguous with `B-niagara-module-input-stack-infer` (IN-REVIEW), because `NiagaraEditHandler.cpp:755-757` fires identically for "module not in the resolved stack" and for a malformed stack. Named the unrun control that separates them (reset a dynamic-input override, then retry `set_module_input` on a ParticleSpawn module WITH explicit `scriptUsage:"ParticleSpawnScript"`, plus a before/after `decompile_nir` diff). Source re-verified in this tree: the `:755-757` guard and the `:750` `IndexOfByPredicate` confirmed; that guard lives in `ApplyModuleMutation` (declared `:669`) and is **not** on `reset_module_input`'s path (handler `:2856`), so the session log's "`:757` is the cause" attribution is wrong and is not repeated here; the stack-inference fix from `B-niagara-module-input-stack-infer` IS present at `:733-737`, and `FindOwningStackOutputNode` (`:582`) would itself fail on a corrupted stack and fall back to the ParticleUpdate default at `:740`. Candidate deletion site recorded as an explicit hypothesis: the unbounded upstream walk in `RemoveOverridePinAndChainedNodes` (`:2040-2119`, collect `:2088`, follow `:2096`, delete `:2116`) has no exclusivity check on the nodes it collects. Corroborating context: `F-niagara-reset-module-input` (DONE) `#4` shipped that function with integration coverage explicitly deferred, and its `#5`/`#6` verifications exercised only the static-switch path.
