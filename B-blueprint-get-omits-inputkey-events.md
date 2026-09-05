---
id: B-blueprint-get-omits-inputkey-events
title: "blueprint.get's events[] silently omits every K2Node_InputKey entry point — a pawn with four live input bindings reads as having none, and a published review filed a High defect over the gap"
status: OPEN
severity: High
category: bug
tags: [blueprint, blueprint-get, blueprint-inspect, events, input, K2Node_InputKey, get_execution_flow, silent-wrong-data, false-defect, weapons]
encounters: 1
lastSeen: 2026-09-05T00:00:00Z
---

# The summary verb answers "what does this Blueprint handle?" with a list that leaves out the input

`blueprint.get`'s `events[]` is the array a caller reads to learn a Blueprint's entry points. It
enumerates `K2Node_Event` and custom events and drops `K2Node_InputKey` entirely. Nothing in the
response says a class of entry point was filtered out, so the omission reads as an absence.

## What was called

```
blueprint.get {assetPath: "/Game/FPS/Weapons/Test/BP_WeaponTestPawn"}
blueprint.graph.get_execution_flow {assetPath: "/Game/FPS/Weapons/Test/BP_WeaponTestPawn"}
```

## What came back — measured

| Verb | Entry points reported |
|---|---|
| `blueprint.get` `events[]` | **4** — `ReceiveActorBeginOverlap`, `ReceiveTick`, `ReceiveBeginPlay`, `AddRecoil` |
| `blueprint.graph.get_execution_flow` | **11** |

The **7 missing entries are all `K2Node_InputKey`**, on the same asset in the same session. Four of
them are the pawn's entire weapon-firing input surface:

- `K2Node_InputKey_0` — LMB **Pressed** → `StartFire`
- `K2Node_InputKey_7` — LMB **Released** → `StopFire`
- `K2Node_InputKey_1` — RMB **Pressed** → `StartADS`
- `K2Node_InputKey_3` — RMB **Released** → `StopADS`

All four carry `enabledState: "enabled"`. They are not stubs, not disabled, and not orphaned:
`blueprint.graph.find_orphaned_nodes` returns **0 of 60 nodes** on this asset, and the asset's mtime
is **2026-09-02 23:00:56** — untouched since before the review that got it wrong.

## The failure this actually caused

**A published review filed a false High defect off this response.** `Docs/fps/reviews/weapons-review-02.md`
claimed `BP_WeaponTestPawn` had no `StopFire` / `StopADS` binding at all, and identified two orphan
nodes as the amputated stumps of the missing handlers. Ground truth: both bindings exist, are
enabled, and are wired; the pawn has zero orphans. The review's chain of reasoning was sound — it
read the verb that names itself as the events readback, saw four events, and concluded the release
handlers were absent. The verb returned a positive-looking, complete-looking list over a graph with
seven more entry points than it reported.

This is the silent-wrong-data shape: not an error, not an omitted field, but a shorter list that
looks like a whole one.

## What was expected

Either `events[]` includes input entry points (with enough shape to tell `Pressed` from `Released`
— they are distinct nodes on the same key and the difference is the whole binding), or the response
states what it filtered. A caller has no way to know from the response alone that a whole node class
was skipped; `get_execution_flow` is the only surface that reveals it, and a caller only reaches for
that after already doubting the summary.

## What is asked for

1. **Emit `K2Node_InputKey` entries in `events[]`**, each carrying at least the key name, the
   pressed/released edge, and the target it calls — the same three facts `get_execution_flow` has.
   `blueprint.inspect` shares `CollectBlueprintEvents`, so both surfaces gain it together.
2. **If input events are deliberately out of scope for this array**, say so in the response (an
   `omittedEventClasses` / `inputEventCount` field) and in `blueprint.get`'s wiki Notes, and route
   the caller to `get_execution_flow`. A documented exclusion is recoverable; a silent one is not.
3. **Regression test:** a Blueprint with one `K2Node_InputKey` must not produce an `events[]` whose
   length equals the count of its `K2Node_Event` nodes.

## Adjacent, measured the same round: `defaults` covers only class-own variables

`blueprint.get`'s `defaults` map is populated from the Blueprint's own `NewVariables`, so a child
class that overrides **inherited** properties reports nothing: `BP_Weapon_AR` has **19 overridden
inherited properties** and returns `defaults: {}`. That is the same "the map is there and it is
empty" experience `E-blueprint-get-defaults-always-empty` (IN-REVIEW) was filed for, on a case its
shipped fix does not reach. Appended as evidence there rather than duplicated here; recorded in this
ticket only because both gaps were hit on the same call and compound — a caller reading
`blueprint.get` on `BP_Weapon_AR` sees neither its input bindings nor its overridden defaults.

## Root cause — guess, no source read taken

`CollectBlueprintEvents` (`BlueprintHandlerUtils.cpp`, named by
`E-inspect-events-omits-disabled-stub-flag`'s fix) almost certainly filters on `UK2Node_Event` and
the custom-event path and never considers `UK2Node_InputKey`, which is a sibling entry-point class
rather than a subclass. **Inference from the response shapes; no plugin source was opened for this
ticket and no `file:line` is claimed.**

## Severity

**High** on the silent-false-success band. The caller trusts a result that is a lie and builds on it
— demonstrably, in a published document. Reach is not every-session but `blueprint.get` is the
primary Blueprint summary read, which holds it at High rather than stepping it down.

## Related

- `E-inspect-events-omits-disabled-stub-flag` (IN-REVIEW, Medium) — **the same array, the adjacent
  gap.** That ticket adds an `enabled` flag to each `events[]` entry; this one says whole entries are
  missing. A fixer in `CollectBlueprintEvents` should land both: an `enabled` flag on a list that
  omits a third of the entry points is still not an answer to "what does this Blueprint handle?".
- `E-blueprint-get-defaults-always-empty` (IN-REVIEW, Low) — same method, the `defaults` half above.
- `B-bpir-upsert-skips-inputkey-entries`, `B-bpir-key-released-entry-silently-dropped`,
  `B-bpir-dual-active-inputkey`, `B-bpir-input-event-entry-signatures-unknown` — the **write/IR** side
  of the same node class being mishandled. Four existing tickets say BPIR cannot round-trip input
  entries; this one says the readback cannot see them either, so the node class is invisible on both
  sides of the loop.
- `B-orphan-sweep-treats-enhanced-input-graph-as-dead` — a third verb misjudging input-driven graphs.
- `B-bpir-disabled-nodes-emitted-as-live` (OPEN, High) — filed the same round on the same asset
  family; there BPIR asserts entry points that are inert, here `blueprint.get` omits entry points
  that are live. Opposite directions, same consequence for a reviewer.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 3. `blueprint.get {assetPath:"/Game/FPS/Weapons/Test/BP_WeaponTestPawn"}` returns `events: [ReceiveActorBeginOverlap, ReceiveTick, ReceiveBeginPlay, AddRecoil]` — **4** — while `blueprint.graph.get_execution_flow` on the same asset in the same session returns **11** entry points; the 7 missing are all `K2Node_InputKey`. Four of them are the pawn's whole firing input surface: `K2Node_InputKey_0` (LMB Pressed → `StartFire`), `_7` (LMB Released → `StopFire`), `_1` (RMB Pressed → `StartADS`), `_3` (RMB Released → `StopADS`), all `enabledState:"enabled"`. Not stubs, not orphans: `find_orphaned_nodes` returns 0 of 60 nodes, and the asset mtime is 2026-09-02 23:00:56 — untouched since before the review that got this wrong. **This produced a false High defect in a published review:** `Docs/fps/reviews/weapons-review-02.md` claimed the pawn had no `StopFire`/`StopADS` binding and named two orphan nodes as the stumps of the missing handlers; both bindings exist and the pawn has zero orphans. The verb returned a complete-looking list over a graph with seven more entry points than it reported, with nothing in the response marking a filtered node class. Ask: emit `K2Node_InputKey` entries in `events[]` with key name, pressed/released edge and call target (shared with `blueprint.inspect` via `CollectBlueprintEvents`); or, if the exclusion is deliberate, name it in the response and the wiki Notes and route to `get_execution_flow`; plus a regression test that a BP with one input key does not produce an `events[]` sized to its `K2Node_Event` count only. Also measured on the same call: `defaults` is populated from class-own `NewVariables` only, so `BP_Weapon_AR` with **19 overridden inherited properties** returns `defaults: {}` — appended as evidence to `E-blueprint-get-defaults-always-empty` (IN-REVIEW) whose shipped fix does not reach inherited overrides. Root cause is a **guess**: `CollectBlueprintEvents` likely filters on `UK2Node_Event` plus the custom-event path and never considers the sibling class `UK2Node_InputKey` — inferred from response shapes, no plugin source opened, no `file:line` claimed. Severity **High** on the silent-false-success band, evidenced by the published false defect; held at High rather than stepped down because `blueprint.get` is the primary Blueprint summary read.
