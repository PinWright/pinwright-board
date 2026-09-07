---
id: F-add-mapping-batch-keys
title: "input.add_mapping is one-key-per-call — binding a movement action (WASD→one IA) forces 4+ near-identical calls, each a full IMC load-mutate-save"
status: OPEN
severity: Low
category: feature
tags: [input, enhanced-input, add_mapping, batch, ergonomic, churn]
encounters: 3
costly: 1
lastSeen: 2026-07-02T14:22:42.2644272+03:00
---

# `input.add_mapping` has no batch form — the canonical WASD bind is 4 redundant calls

`input.add_mapping` binds exactly **one** `(IMC, IA, key)` triple per call. The
single most common Enhanced Input authoring pattern — binding a 2D movement
action to WASD (and often the arrow keys too) — therefore requires 4 (or 8)
near-identical calls that differ only in the `key` argument, with `contextPath`
and `actionPath` repeated verbatim each time. A mouse-look IA likewise needs two
calls (`MouseX`, `MouseY`) to bind both axes to one `IA_Look`.

The wiki page itself already flags the per-call cost (`Docs/wiki-src/input.md`):

> The IMC asset is loaded, mutated, and saved every call — for many bindings
> prefer batching all `add_mapping` calls back-to-back with the editor closed for
> that IMC asset to avoid auto-reload churn.

So the cost is acknowledged, but the only mitigation offered is a usage
discipline ("do them back-to-back"); there is **no batch verb** that does one
load-mutate-save for N keys. Each per-key call re-loads, re-mutates, and re-saves
the same IMC asset — N full asset round-trips for one logical "bind this action
to these keys" intent.

## What it should do

Accept a batched form on `input.add_mapping` (or a sibling `input.add_mappings`)
so one call binds multiple keys to one IA — and/or multiple `(IA, key)` triples
to one IMC — in a single load-mutate-save. Candidate shapes (pick one downstream):

- `keys: ["W","A","S","D"]` alongside the existing single `actionPath` — bind all
  listed keys to that one action.
- `mappings: [{actionPath, key}, ...]` — bind an arbitrary set of triples to one
  `contextPath` in a single pass.

Either collapses the canonical movement-bind from 4 calls (and 4 IMC re-saves) to
1, and makes the "I'm setting up player input" intent a single readable call.
Keep the existing single-`key` form working for the simple case.

## Why it's process friction (clean per-call outcome)

Every call in the evidence task succeeded first-try — this is an
**efficiency/ergonomic** gap, not a bug. The friction is the call-count and the
repeated IMC re-save churn the wiki itself warns about, not any failure. It is
distinct from the two sibling input docs tickets, which are about
**discoverability/param naming**, not call-count:

- `E-create-input-action-valuetype-undiscoverable` (IN-REVIEW) — `valueType` set
  via `property.set`, not a create param (docs).
- `E-add-mapping-example-wrong-param` (IN-REVIEW) — the code example used
  `imcPath` instead of `contextPath` (docs).

Neither proposes a batch form; this ticket is the call-count/churn angle.

**Workaround:** issue the per-key calls back-to-back (as the wiki advises) — works,
but is N asset round-trips and N near-duplicate calls for one bind intent.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean Enhanced-Input setup task (focus `input.add_mapping`, namespace input; story: create IMC_Player + IA_Move/IA_Jump/IA_Look under /Game/Input, set IA_Move/IA_Look to Axis2D via `property.set`, then bind W/A/S/D→IA_Move, SpaceBar→IA_Jump, MouseX/MouseY→IA_Look, then verify with get_input_info). Outcome `clean`, friction note "none" — every call landed first try because the wiki already documents the `property.set ValueType` recipe and the correct `contextPath` param (the two IN-REVIEW docs fixes working as intended). The residual PROCESS cost is call-count/churn: **7** `add_mapping` calls to one IMC (4 of them — W/A/S/D — to the *single* action IA_Move), each repeating `contextPath`+`actionPath` and each a full IMC load-mutate-save. The wiki (`Docs/wiki-src/input.md`, `### input.add_mapping`) already warns of the per-call re-save churn but offers only "batch the calls back-to-back" — no batch verb exists. Proposing a batched `keys:[...]` / `mappings:[...]` form to collapse the canonical WASD bind from 4 calls (and 4 re-saves) to 1. Filed F- (batch/convenience feature). No existing add_mapping batch ticket on the board (ripgrep clean across OPEN/closed for `add_mapping`/batch/multi-key).
- `#2-more-evidence-17-mappings-and-remove-asymmetry` `OPEN` reporter — Second clean-outcome audit corroborating the same gap at larger scale. Task: full third-person input scheme under /Game/Input — IA_Move/Look/Jump/Sprint/Fire + IMC_Player, keyboard layout (W/A/S/D + 4 arrow keys → IA_Move, Mouse2D → IA_Look, SpaceBar → IA_Jump, LeftShift → IA_Sprint, LeftMouseButton → IA_Fire) plus 5 gamepad fallbacks. That is **17** sequential `input.add_mapping` calls to one IMC — **12** of them on the keyboard pass and **8** of those (W/A/S/D + Up/Down/Left/Right) targeting the *single* action IA_Move — each repeating `contextPath`+`actionPath` and each a full IMC load-mutate-save (17 re-saves for one "set up player input" intent). Outcome `clean`, friction note "none" (every call landed first try; `property.set IA_Move/IA_Look.ValueType=Axis2D` was prescribed by the now-fixed `E-create-input-action-valuetype-undiscoverable` overlay). New sharpening of the same point: the agent bound the paired-2D keys `Mouse2D`/`Gamepad_Left2D`/`Gamepad_Right2D` as one key each (so 2D look needs only 1 call, not the MouseX+MouseY split the #1 task hit) — but the *digital* movement keys still cannot be collapsed, so WASD+arrows remains the 8-call worst case. Strongest corroboration is the **verb asymmetry**: the companion `input.remove_mapping` is already batch-by-action — one `remove_mapping` on IA_Sprint removed BOTH bound keys (`[LeftShift, Gamepad_LeftShoulder]`) and reported them, dropping mappingCount 17→15 in a single call — while `add_mapping` has no such batch form. Removing N keys for one action = 1 call; adding them = N calls. That asymmetry is direct evidence the batched `keys:[...]`/`mappings:[...]` add form proposed here is the natural shape (it would mirror remove's existing by-action batch). Appending as evidence (not a new ticket) per cross-task aggregation. No new is_error in the 37-call log; pure efficiency/churn, exactly this ticket's scope.
- `#3-liveness` `OPEN` reporter — Same churn still observed (no new angle): a clean first-person-input setup task (namespace input; IMC_FirstPerson + IA_Move/Look/Jump/Interact under /Game/Input) issued **11** sequential `input.add_mapping` calls to one IMC, **8** of them (W/A/S/D + Up/Down/Left/Right) targeting the single action IA_Move — each repeating `contextPath`+`actionPath` and each a full IMC load-mutate-save. Outcome `clean`, friction note "none", zero is_error in the 33-call log. Reconfirms the WASD+arrows 8-call worst case; pure liveness marker, no new evidence beyond #1/#2.
