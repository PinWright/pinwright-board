---
id: B-bpir-disabled-nodes-emitted-as-live
title: "blueprint.decompile / bpir.txt renders DISABLED nodes as live BPIR with no marker — a fully disabled EventGraph decompiles to three entry events calling parent, and the BPIR language has no syntax for node enabled-state at all"
status: OPEN
severity: Medium
category: bug
tags: [bpir, bpir-decompiler, blueprint-decompile, asset-dump, disabled-nodes, node-state, enabledState, lossy-ir, silent-wrong-data, round-trip, weapons]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# The IR says the subclass overrides three lifecycle events; the graph says every node is compiled out

BPIR is the surface a reviewer reads to answer "what does this Blueprint do". For a disabled node it
answers wrong, and there is no token in the language that could answer right.

## What was called

```
asset.dump {assetPath: "/Game/FPS/Weapons/BP/BP_Weapon_AR"}      → bpir.txt
blueprint.graph.get_node_details {nodeId: "2FDCE57B482B8CD531FE198D476FAB95", ...}
```

## What happened — measured

`BP_Weapon_AR`'s **entire `EventGraph` is disabled**. `blueprint.graph.get_node_details` says so
unambiguously, on all six nodes:

```
nodeState.enabledState: "disabled"
nodeState.isEnabled: false
```

`bpir.txt` for the same asset emits **three live `entry event` blocks** — `BeginPlay`,
`ActorBeginOverlap`, `Tick` — each calling parent, with no marker, no comment, no warning, and
nothing in the `# ==== Warnings ====` block.

A reviewer reading only the IR concludes the subclass overrides those three events and chains to
the parent implementation. In fact every node in the graph is compiled out: the subclass overrides
nothing, and the parent's behaviour runs unmodified.

## The gap is in the language, not just this asset

`bpir.md` — the BPIR language reference, read at
`X:\src\unreal\EAContentExamples58\Saved\PinWright\wiki\bpir.md` — contains **no mention of
`disabled`, `enabledState`, or node state of any kind**. This is not a decompiler that forgot to
emit a token that exists; there is no token. The IR is structurally incapable of representing a
disabled node, which means:

- **Decompile is lossy in a way that inverts meaning.** Most IR omissions lose detail. This one
  turns "does nothing" into "calls parent".
- **Round-trip is lossy in the same direction.** A `compile_bpir` of that IR would author *live*
  nodes back over what were disabled ones — the decompile→edit→recompile loop silently enables a
  graph the author deliberately turned off.

## What was expected

The signal is already in hand at decompile time, on the same node objects the decompiler walks, and
the plugin already exposes it one layer down: `blueprint.graph.get_nodes` / `get_node_details` emit
a structured `nodeState` block (`isEnabled`, `enabledState`, `isDisabledByUser`) whenever
`includeNodeState: true`. Only BPIR drops it.

## What is asked for

1. **A BPIR token for node enabled-state**, and a line in the language reference defining it. The
   two engine states that matter are `disabled` (compiled out) and `disabled-and-not-passing-flow`
   vs `enabled-and-not-passing-flow` (the "development only" middle state) — at minimum, disabled
   must be distinguishable from enabled.
2. **`compile_bpir` must accept it**, or the round-trip re-enables what it reads. A read-only marker
   is strictly worse than none for the decompile→edit→recompile loop this IR exists to serve.
3. **Fallback if (1) is too large a language change for now: a warning.** A
   `# BPIR_WARN: Node emitted as live but is disabled in the graph: <graph> :: <class> '<name>' nodeId=<guid>`
   line per disabled node, in the existing warnings block, is a small diff and stops the IR from
   reading clean over a graph that is switched off. It does not fix the round-trip.
4. **Entry-level shorthand.** A graph where *every* node is disabled (this case) deserves to say so
   once at the top rather than per-node — a reviewer's first question about `BP_Weapon_AR` is "does
   this subclass do anything", and the answer is one line.

## Root cause — guess, no source read taken

The decompiler almost certainly walks nodes and emits statements without consulting
`UEdGraphNode::GetDesiredEnabledState()` / `IsNodeEnabled()`, the same accessors the graph-inspection
handler uses to build its `nodeState` block. **Inference from the two surfaces' behaviour; no plugin
source was opened for this ticket and no `file:line` is claimed.**

## Severity

**Medium**, sitting just under the High silent-wrong-data band. It is silent, it is wrong, and it is
on the normal path — `bpir.txt` is written by every `asset.dump` of every Blueprint. It is held at
Medium rather than High because a second published verb (`get_node_details` with
`includeNodeState: true`) does carry the truth, so the information is one call away for a caller who
thinks to doubt the IR — which is exactly the caller this defect does not produce. A fixer who
judges that "the IR reads as live over a compiled-out graph, and recompiling it enables the graph"
belongs in the High band will get no argument from this reporter.

## Related

- `E-inspect-events-omits-disabled-stub-flag` (IN-REVIEW, Medium) — **the same missing signal on the
  third surface.** `blueprint.inspect`'s `events[]` lists disabled default stubs indistinguishably
  from authored handlers; its fix adds an `enabled` boolean derived from `SourceNode->IsNodeEnabled()`
  in `CollectBlueprintEvents`. That ticket's own scope note says the graph-node layer "is already
  solved" via `nodeState` and narrows itself to `events[]` — leaving BPIR as the remaining surface,
  which is this ticket. The three should be consistent: `nodeState` (done), `events[].enabled`
  (in review), BPIR token (here).
- `B-bpir-decompiler-emitter-coverage-gap`, `B-bpir-delegate-signature-lost`,
  `B-bpir-composite-inline-name-lost`, `B-bpir-fallthrough-reconverge-dropped` — the existing family
  of BPIR fidelity losses. This one is distinguished by direction: those lose information, this one
  asserts information that is false.
- `B-bpir-orphan-warning-diagnostics-lossy` — the warnings block this ticket's fallback (3) would
  extend, and the precedent for BPIR emitting a per-node `# BPIR_WARN:` line with a nodeId.
- `B-orphan-finder-vs-decompiler-disagree` — the other case this review round found of two verbs
  disagreeing about the same graph, also on `/Game/FPS/Weapons/`.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 2. `BP_Weapon_AR`'s entire `EventGraph` is disabled — `blueprint.graph.get_node_details` on `2FDCE57B482B8CD531FE198D476FAB95` returns `nodeState.enabledState: "disabled"` and `isEnabled: false`, on all six nodes — yet the asset's `bpir.txt` emits three live `entry event` blocks (`BeginPlay`, `ActorBeginOverlap`, `Tick`), each calling parent, with no marker, no comment and nothing in the warnings block. A reviewer reading only the IR concludes the subclass overrides those three events and chains to the parent; in fact every node is compiled out and the subclass overrides nothing. The gap is in the language, not just the emitter: the BPIR language reference read at `X:\src\unreal\EAContentExamples58\Saved\PinWright\wiki\bpir.md` contains no mention of `disabled`, `enabledState`, or node state of any kind, so BPIR cannot represent a disabled node — which makes decompile lossy in a meaning-inverting direction ("does nothing" becomes "calls parent") and makes the round-trip lossy the same way, since a `compile_bpir` of that IR would author live nodes back over deliberately disabled ones. The signal is already in hand: the plugin's own `blueprint.graph.get_nodes` / `get_node_details` emit a structured `nodeState` block (`isEnabled`, `enabledState`, `isDisabledByUser`) under `includeNodeState:true`; only BPIR drops it. Ask, in order: a BPIR token for node enabled-state plus its definition in the language reference; `compile_bpir` acceptance of that token (a read-only marker would leave the round-trip re-enabling what it reads); as a smaller fallback, a per-node `# BPIR_WARN:` line naming each disabled node emitted as live, following `B-bpir-orphan-warning-diagnostics-lossy`'s precedent; and an entry-level shorthand for the all-nodes-disabled case, which is the first question a reviewer has about this asset. Root cause is a **guess**: the decompiler likely emits statements without consulting `GetDesiredEnabledState()` / `IsNodeEnabled()`, the accessors the graph-inspection handler already uses — inferred from the two surfaces' behaviour, no plugin source was opened and no `file:line` is claimed. Severity Medium, just under the High silent-wrong-data band: silent, wrong, and on the path every `asset.dump` of a Blueprint takes, but held down because `get_node_details` with `includeNodeState:true` carries the truth one call away — for a caller who thinks to doubt the IR, which is the caller this defect does not produce.
