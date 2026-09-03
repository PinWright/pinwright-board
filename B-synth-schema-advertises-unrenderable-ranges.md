---
id: B-synth-schema-advertises-unrenderable-ranges
title: "audio.synth.describe_schema publishes parameter ranges the render path rejects — ringmod.rateHz and filter.cutoffHz disagree between the spec table and the DSP"
status: OPEN
severity: Medium
category: bug
tags: [audio, synth, describe_schema, validation, single-source-of-truth, contract]
encounters: 1
lastSeen: 2026-08-18T00:00:00+03:00
---

# The schema and the renderer disagree about what is legal

`audio.synth.describe_schema` renders its parameter ranges from the spec table in
`PwSynthRecipe.cpp`, which is documented as the single source of truth. Two effect
parameters are enforced more narrowly by the DSP than the table advertises, so a
recipe that is valid per the published schema is rejected at render time.

| parameter | published range | actually enforced | where |
|---|---|---|---|
| `ringmod.rateHz` | `0.1 .. 20000` | `10 .. 10000` | `PwFxChainB.cpp:826` |
| `filter.cutoffHz` | `.. 20000` | `.. 0.45 * sampleRate` | `PwFxChainA.cpp:286` |

Both narrowings are **correct and deliberate**, and should not simply be removed.
`Audio::FRingModulation` silently clamps its carrier outside roughly 10-10000 Hz, and
`Audio::FBiquadFilter` silently clamps cutoff to about `[5 Hz, 0.45 * SampleRate]`.
Reporting back a requested value the filter did not use is the defect class
`rpc-design.md` §1 exists to prevent, so rejecting is right. The bug is that the
schema does not say so.

## Why this matters more than it looks

No preset library ships with the synth namespace. `describe_schema` plus the cookbook
(`docs/wiki-src/audio.synth.cookbook.md`) is the **entire** cold-start path for an
agent authoring a recipe. An agent that trusts the published range and picks
`rateHz: 5` gets an error on its first call, from a schema it was told is
authoritative.

## The two cases need different fixes

**`ringmod.rateHz` is a plain range error.** The spec-table row should publish
`10 .. 10000` to match the enforcement. One-line fix, and `describe_schema` becomes
true again.

**`filter.cutoffHz` cannot be fixed by a static range**, because the ceiling depends
on the render's `sampleRate` — a cross-field constraint no single spec-table row can
express. `FPwSynthKindSpec` already carries a `Constraint` string for exactly this
purpose (the `modal` row uses a `Validator` hook for its equal-length arrays). Publish
the rule there so it reaches `describe_schema` output, e.g.
"cutoffHz must be below 0.45 * sampleRate; at 44.1 kHz the effective ceiling is
19845 Hz, below the published 20000".

## Sweep for the same class before closing

These two were found while writing the cookbook against the implementation, not by a
test — which means the general case is unguarded. Before calling this DONE, grep every
`PwFx*.cpp` and `PwGen*.cpp` rejection against its spec-table row and account for each
divergence as fixed, deliberately-published, or absent. A contract test asserting that
every published range is renderable at the default sample rate would make the whole
class unrepresentable, and is the better fix if it is cheap.

## History

- 2026-08-18 - Filed OPEN. Found while authoring `audio.synth.cookbook.md` against the
  real generator and effect sources rather than against the plan; the cookbook
  documents the effective windows so it does not repeat the schema's claim.
