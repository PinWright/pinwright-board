---
id: F-add-blackboard-key-default-value
title: "`ai.add_blackboard_key` has no `defaultValue` param — seeding each key's default costs a second `ai.set_blackboard_value` round-trip"
status: OPEN
severity: Low
category: feature
tags: [ai, blackboard, add_blackboard_key, set_blackboard_value, default-value, add-verb-no-inline-default, batch]
encounters: 1
lastSeen: 2026-07-01T21:58:20+03:00
---

# `ai.add_blackboard_key` can't pre-seed a key's default value at creation

## What's missing
`ai.add_blackboard_key` accepts `keyType` / `baseObjectClass` / `isInstanceSynced`,
but **no `defaultValue`**. So a key can be created pre-synced in one call, yet its
starting value always needs a *second* `ai.set_blackboard_value` round-trip. This
is an internal asymmetry: the instance-sync flag is settable at creation, the
default value is not — even though both are per-key `UBlackboardKeyType_*` state.
It is also a cross-namespace parity gap: the sibling authoring verb
`blueprint.add_variable` **does** take a `defaultValue` param (see
`B-add-variable-default-value-ignored`), so blackboard-key authoring is the odd
one out.

## What it should do
Add an optional `defaultValue` string param to `ai.add_blackboard_key`, mirroring
`ai.set_blackboard_value`'s existing value coercion (plain scalars, `Name`/`String`
literals, the `X=.. Y=.. Z=..` vector form), so a key can be created pre-seeded in
one call. (A batched `ai.set_blackboard_value` accepting multiple key→value pairs
would be an alternative shape and belongs to the same `add-verb-no-inline-default` /
`batch` convenience family as `F-add-variables-batch`, `F-batch-pin-defaults`,
`F-console-batch-get-cvar-values`.) The success response should echo the applied
default so a caller can confirm it stuck.

## Evidence
Struggle audit of a **clean, GREEN** `BB_Guard` blackboard-authoring task (focus
`ai.set_blackboard_value`, namespace `ai`, outcome `done`, 16 MCP RPCs, zero errors,
zero retries; transcript
`.../subagents/workflows/wf_695386e4-c0d/agent-adf6dbfd88a1d78f2.jsonl`, asset
`/Game/AI/Blackboards/BB_Guard`). To author a 6-key guard blackboard the agent
issued **6 `ai.add_blackboard_key` + 5 `ai.set_blackboard_value` = 11 calls**
(AlertLevel:Int→2, PatrolSpeed:Float→300, CanSeePlayer:Bool(isInstanceSynced)→false,
CurrentState:Name→Patrol, HomeLocation:Vector→X=1200 Y=800 Z=90, TargetActor:Object/Actor).
Every non-default key needed an add-then-set pair; `CanSeePlayer` set its sync flag
inline on the add call, highlighting the asymmetry (sync inline, default not). All 11
calls returned `ok:true` first try — the attempt agent reported friction "none"; the
focus verb `ai.set_blackboard_value` worked perfectly (accepted plain-string scalars,
the `X=1200 Y=800 Z=90` vector, and the type-default `false` all first try). This is a
pure convenience/granularity observation surfaced by the call-trace analyzer, not a
failure: a `defaultValue` param on the add verb could roughly halve the call count.

## Distinct from
- `B-add-blackboard-key-base-object-class-dropped` — a *bug* (the add verb's
  `baseObjectClass` param is silently dropped); this is a *missing* param, not a
  dropped one.
- `B-add-variable-default-value-ignored` — the same shape on `blueprint.add_variable`,
  but that param *exists and is buggy*; here the param does not exist at all, on a
  different namespace/method.
- `E-get-blackboard-value-omits-value` — the *read* side (getter omits the stored
  default); this is the *write/creation* side.
- `F-add-variables-batch` — batch member-variable authoring on `blueprint.*`; same
  convenience family, different namespace and a batch (not inline-default) shape.

severity rationale: impact=pure friction (one extra call per non-default key; the
`ai.set_blackboard_value` workaround already exists and works first-try) × reach=rare
(blackboard default authoring is not an every-session path) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the struggle audit of a clean/GREEN `BB_Guard` blackboard task (focus `ai.set_blackboard_value`, 16 RPCs, 0 errors/retries; transcript `agent-adf6dbfd88a1d78f2.jsonl`, asset `/Game/AI/Blackboards/BB_Guard`). Call-trace analyzer flagged an add-then-set granularity workaround: 6 `ai.add_blackboard_key` + 5 `ai.set_blackboard_value` = 11 calls, because `add_blackboard_key` accepts `keyType`/`baseObjectClass`/`isInstanceSynced` but no `defaultValue` — so every non-default key needs a second set round-trip. Asymmetry: `CanSeePlayer` set `isInstanceSynced` inline on the add call, but the default value cannot be inlined. Parity gap: `blueprint.add_variable` has a `defaultValue` param; `ai.add_blackboard_key` does not. Attempt agent reported friction "none" (all calls `ok:true` first try, no struggle) — this is a convenience/ergonomic observation, not a defect; the focus verb `ai.set_blackboard_value` itself worked perfectly and is NOT at fault. Proposed: add an optional `defaultValue` string param to `ai.add_blackboard_key` mirroring `set_blackboard_value`'s coercion, echoing the applied default in the response. Dedup: ripgrep across OPEN/IN-REVIEW/DONE — no existing ticket covers a missing `defaultValue` on `add_blackboard_key`; the two `add_blackboard_key` matches are the `baseObjectClass`-dropped bug and an incidental mention. Genuinely new; seeded family tag `add-verb-no-inline-default`. severity Low (pure friction, easy existing workaround, rare path).
