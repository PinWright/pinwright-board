---
id: E-blueprint-compile-silent-about-orphans
title: "blueprint.compile returns errors:[] warnings:[] over a graph it has just orphaned, while find_orphaned_nodes and the decompiler both report it — the natural gate is the one surface that cannot see it"
status: OPEN
severity: Medium
category: ergonomic
tags: [blueprint, compile, orphans, find-orphaned-nodes, decompile, false-clean, agent-gate, cross-verb-inconsistency]
encounters: 1
lastSeen: 2026-09-05T19:52:13Z
---

# Three surfaces, one graph, and the one an agent gates on says nothing

Rewiring a node's inputs away from an existing getter leaves that getter connected to nothing.
Measured on `BP_WeaponBase::ProcessHit`, 2026-09-05, after moving `MakeVector`'s X/Y/Z from a
`Get DecalSize` to a `Select` output:

| surface | verdict |
|---|---|
| `blueprint.compile` | `{compiled: true, status: "UpToDate", errors: [], warnings: []}` |
| `blueprint.decompile_function` | `warnings: [{ text: "Orphaned node not reachable from any entry point: ProcessHit :: K2Node_VariableGet 'Get DecalSize' nodeId=67702A60-... @(0,472)" }]` |
| `blueprint.graph.find_orphaned_nodes` | `orphanedCount: 1` |

The plugin therefore *has* a notion of "orphan is a finding" and applies it on two surfaces. The
third — the one a caller reaches for to answer "is this graph OK now" — is silent.

## Why this is worth fixing rather than documenting

**UE is not wrong here and that is the trap.** A disconnected pure node is legal Blueprint; the
engine compiler has no reason to complain, so `errors: []` is a truthful report of what UE thinks.
The problem is that an agent's gate after a graph edit is naturally "compile came back clean", and
that sentence is true and useless in the same breath. It is the same shape as several defects
already on this board: a green field answering a narrower question than the caller asked.

**The orphan is usually created by the very edit being compiled.** This is not a pre-existing-mess
problem. Rewiring an input pin is the single most common structural edit through these verbs, and it
orphans the old feeder every time. So the case where compile is silent is exactly the case where an
agent has just caused it.

**It compounds a documented hazard.** `E-compile-bpir-upsert-leaves-data-only-orphans` records
`compile_bpir` upsert leaving data-feeder orphans its own sweep misses, and
`B-orphan-sweep-mathexpression-fatal-load` and `B-orphan-sweep-deletes-isolated-timeline` record
that the blanket sweep is not always safe to run afterwards. So a caller is told neither that
orphans exist nor that the cleanup verb is safe — they have to know to run a third verb and then
judge each result.

## Fix

Report the count on the compile response — an `orphanedCount` field, or an entry in `warnings`
naming the nodes. Not an error: orphans are legal and a caller may intend them. The ask is only that
the surface an agent gates on stops being the one that cannot see what the other two can.

## History
- `#1-compile-clean-over-self-inflicted-orphan` `OPEN` reporter - Found on the FPS build, 2026-09-05. Rewired `MakeVector`'s X/Y/Z inputs in `BP_WeaponBase::ProcessHit` from `Get DecalSize` to a `Select` output via `connect_pins`; the getter was left connected to nothing. `blueprint.compile` returned `compiled: true, status: "UpToDate", errors: [], warnings: []`, while `decompile_function` emitted an explicit orphan warning naming the node and id, and `find_orphaned_nodes` returned `orphanedCount: 1`. Deleting the node restored 0 orphaned across 35 graphs / 420 nodes. UE is not wrong to compile it — a disconnected pure node is legal — but the caller's natural gate after a graph edit is the compile result, and the orphan is created by the edit being compiled, so silence there is maximally unhelpful. Compounds `E-compile-bpir-upsert-leaves-data-only-orphans` (upsert leaves data-feeder orphans) and the two orphan-sweep hazards, since a caller is told neither that orphans exist nor that the sweep is safe. Ask: surface an `orphanedCount` or a `warnings` entry on the compile response; not an error, since orphans are legal and may be intended.
