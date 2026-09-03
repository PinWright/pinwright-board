---
id: B-bpir-decompile-adjacent-terminal-blocks-read-as-fallthrough
title: "blueprint.decompile emits two independent TERMINAL blocks adjacently, and BPIR fall-through semantics then assert an exec edge the graph does not have — reads as 'the fallback unconditionally clobbers the result' over correct dispatch logic, and invites a destructive fix"
status: OPEN
severity: High
category: bug
tags: [bpir, bpir-decompiler, blueprint-decompile, terminal-blocks, fallthrough, round-trip, lossy-ir, silent-wrong-data, meaning-inverting, weapons]
encounters: 1
lastSeen: 2026-09-03T04:25:00Z
---

# Two terminal blocks printed back-to-back become a fall-through that does not exist

BPIR section 2.8 defines an unterminated block as falling through to the next label. The decompiler
has no end-of-block / no-successor terminator to emit, so when a graph contains **two sibling
terminal nodes** (both with empty `execOutputs`), it prints them as two adjacent label blocks. The
language's own documented semantics then assert a continuation between them. The graph has no such
edge.

This inverts meaning in the dangerous direction: the emitted IR claims a later unconditional
assignment overwrites an earlier conditional one, when in fact the two assignments sit on mutually
exclusive terminal branches. A reader concludes the function is broken and "fixes" working logic.

## Witness — `/Game/FPS/Weapons/BP_WeaponBase` :: `ResolveImpactAssets`

`blueprint.decompile_function` emits, as the last blocks:

```
@ok_2:
    %n6: object<SoundBase> = call Map_Find(TargetMap: %n5.AsBPDAImpactSFX.ImpactSounds, Key: $Surface)
    %n7 = branch(%n6) [false -> @merge_3, true -> @then_3]

@fail_2:
    exec -> @merge_3

@then_3:
    set ResolvedImpactSound = %n6.Value          # no terminator; section 2.8 says "falls through"

@merge_3:
    set ResolvedImpactSound = $FallbackImpactSound
```

Read literally, every path reaches `@merge_3`, so the per-surface sound is always overwritten by the
fallback and per-surface impact audio can never play.

**`blueprint.graph.get_execution_flow` on the same function says otherwise.** The two setters are
separate nodes and BOTH terminate:

| nodeId | title | data source pin | `execOutputs` |
|---|---|---|---|
| `B5ACB4CD495528A6CBC6C2B1FDE492F4` | Set ResolvedImpactSound | `Value` (map hit) | `[]` |
| `377FEDB646A2343E99D9669D73D9410D` | Set ResolvedImpactSound | `FallbackImpactSound` | `[]` |

Wiring:

- Branch `B09272E846071E39D05964A2F6FE7884`: `then -> B5ACB4CD`, `else -> 377FEDB6`
- Cast `AE7BD3F04B5FA6FE93085FA06505B9D2` (`Cast To BP_DA_ImpactSFX`): `CastFailed -> 377FEDB6`

There is no edge from `B5ACB4CD` to `377FEDB6` — `B5ACB4CD.execOutputs` is empty. The function is
**correct**: a map hit sets the per-surface sound and returns; a map miss or a failed cast sets the
fallback and returns.

## The control is in the same function

The decal path immediately above is a **genuine** reconvergence, and the decompiler renders that one
correctly with an explicit `exec -> @merge_2` on the branch that jumps:

- Branch `08A67D5049A61651AD50F9AE754622DA`: `then -> FDAF4C2742DC55B4B0A949BD11B2D956` (Set
  ResolvedImpactDecal from `Value`) `-> AE7BD3F0` (Cast To BP_DA_ImpactSFX); `else -> AE7BD3F0`
  directly. Two predecessors, one successor — a real join, correctly emitted.

So within one graph: reconvergence renders right, sibling-terminals render wrong. The distinguishing
factor is precisely that the sound blocks have **no successor to name**. That makes this cheap to
fix and cheap to regression-test — the fixture is one function carrying a correct control beside the
defect.

## Distinct from the shared-tail ticket

Not a duplicate of `B-bpir-decompile-shared-tail-absorbed-into-branch` (DONE, High). That one is
about a **real** join node being given the **wrong label** on reconvergence — an edge exists and is
mis-targeted. This one manufactures an edge between two blocks that **share no edge at all**; no
join node is involved and nothing is mis-labelled. The shared-tail fix does not address it, and this
reproduces on a current build with that fix in.

## Why High

Same band as the shared-tail ticket, for the same reason: silent, wrong, and meaning-inverting on
the normal path (`bpir.txt` is written by every `asset.dump` of every Blueprint). It is arguably
worse than a loss-of-detail defect because it does not merely omit — it **asserts a defect that is
not there**, and the natural remediation ("add the missing `exec ->` so the found branch skips the
fallback") rewires a correct graph and breaks the feature for real.

Measured cost in the seed encounter: the false reading was reported to the stream lead as a live
bug, the lead authorized the rewrite, and it was withdrawn only because the implementer consulted
`get_execution_flow` before editing. Two agents accepted the IR at face value; only an out-of-band
verification caught it.

## What is asked for

1. **Emit an explicit terminator for a node with no exec successor** — a bare `return` / `end`
   statement, or a `# BPIR: end of chain` marker line — so an adjacent block cannot be read as its
   continuation. A terminator is the minimal change and needs no new label allocation.
2. **Alternatively, force a fresh label with an explicit no-fall-through marker** whenever the
   previous block ended on a node with empty `execOutputs`.
3. Whichever is chosen, `compile_bpir` must accept the token, or the round-trip re-introduces the
   phantom edge — the same round-trip requirement `B-bpir-disabled-nodes-emitted-as-live` asks for.
4. Until then, a line in `bpir.md` section 2.8 warning that adjacent blocks are a fall-through only
   when the preceding block's last node actually has an exec successor, and that the IR cannot
   currently distinguish the two — so `get_execution_flow` is the arbiter for any control-flow
   question.

## Root cause — inference, no source read taken

The decompiler's chain walker almost certainly emits statements and starts a new label without
recording whether the previous block's terminal node had zero exec successors, so "next label" and
"successor" become indistinguishable in the output. Inferred from the two surfaces' behaviour on
this one function; no plugin source was opened and no `file:line` is claimed.

## Related

- `B-bpir-decompile-shared-tail-absorbed-into-branch` (DONE, High) — the reconvergence-labelling
  case; see "Distinct from" above.
- `B-bpir-disabled-nodes-emitted-as-live` (OPEN, Medium) — the other meaning-inverting BPIR defect
  on this same Blueprint family; shares the "IR asserts something false rather than omitting
  something true" character and the same round-trip requirement.
- `B-orphan-finder-vs-decompiler-disagree`, `B-bpir-break-struct-pin-not-named` — prior cases where
  an IR-only review of `/Game/FPS/Weapons/` reached a wrong conclusion about the graph.

## History
- `#1-filed` `OPEN` WEAPONS — Found while acting on a reported impact-sound defect in `BP_WeaponBase::ResolveImpactAssets`. The decompiled BPIR shows `@then_3: set ResolvedImpactSound = %n6.Value` with no terminator, immediately followed by `@merge_3: set ResolvedImpactSound = $FallbackImpactSound`; under BPIR section 2.8 fall-through semantics that reads as the fallback unconditionally clobbering every per-surface sound. `blueprint.graph.get_execution_flow` on the same function disproves it: the two setters are distinct nodes `B5ACB4CD495528A6CBC6C2B1FDE492F4` (source pin `Value`) and `377FEDB646A2343E99D9669D73D9410D` (source `FallbackImpactSound`), and BOTH report `execOutputs: []` — no edge between them. Branch `B09272E846071E39D05964A2F6FE7884` sends `then -> B5ACB4CD` and `else -> 377FEDB6`, and cast `AE7BD3F04B5FA6FE93085FA06505B9D2` sends `CastFailed -> 377FEDB6`. The function is correct; the IR is not. Control in the same graph: the decal path's branch `08A67D5049A61651AD50F9AE754622DA` is a genuine reconvergence (`then -> FDAF4C27 -> AE7BD3F0`, `else -> AE7BD3F0`) and IS rendered correctly with an explicit `exec -> @merge_2` — so reconvergence works and sibling-terminals do not, the difference being that terminal blocks have no successor to name and the emitter has no terminator token for that case. Distinct from the DONE `B-bpir-decompile-shared-tail-absorbed-into-branch`, which mis-labels a real join; here no join and no edge exist to mis-label, and this reproduces with that fix in. Filed High: it asserts a defect that is not present, and the obvious remediation (add the missing `exec ->` past the fallback) would rewire a correct graph and genuinely break per-surface impact audio. Real measured cost this session — the false reading was reported to the stream lead, the lead green-lit the rewrite, and it was retracted only because `get_execution_flow` was consulted before the edit; two agents accepted the IR at face value. Ask: emit an explicit end-of-chain terminator for a node with empty `execOutputs` (and have `compile_bpir` accept it, else the round-trip re-adds the phantom edge), plus a `bpir.md` section 2.8 note that adjacent blocks imply fall-through only when the preceding block's last node actually has an exec successor. Root cause is an inference from the two surfaces' behaviour; no source was opened and no `file:line` is claimed.
