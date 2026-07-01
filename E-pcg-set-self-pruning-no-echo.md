---
id: E-pcg-set-self-pruning-no-echo
title: "pcg.set_self_pruning_settings returns only {nodeId} and echoes none of the pruningType/radiusSimilarityFactor/bRandomizedPruning it applied, so a caller cannot tell its write landed (or that its values equal class defaults) without a full decompile round-trip"
status: OPEN
severity: Low
category: ergonomic
tags: [pcg, set-self-pruning-settings, readback, round-trip, response-shape, echo, default-value, consistency]
encounters: 1
lastSeen: 2026-07-01T09:49:43.2556070+03:00
---

# `pcg.set_self_pruning_settings` doesn't echo what it set, so confirming the write (and noticing the values equal class defaults) costs a full decompile round-trip

`pcg.set_self_pruning_settings` parses and applies up to three knobs on a
`UPCGSelfPruningSettings` node — `pruningType`
(`LargeToSmall|SmallToLarge|AllEqual|None|RemoveDuplicates`),
`radiusSimilarityFactor` (float), and `bRandomizedPruning` (bool) — but its
success response carries **none of them**. It returns only
`{"nodeId":"SelfPruning_0"}`. A caller therefore cannot confirm from the
response that the write took effect **with the intended values**, and — the
sharp edge in this task — cannot tell from the response that the values it
chose happen to equal the node's **class defaults**. It has to read the node
back through `pcg.decompile` (or `pcg.inspect`) to learn either fact.

This is the same "the mutator already holds the answer but doesn't carry it,
so a working readback verb gets spammed" response-shape family the board tracks
per-method: `E-set-transition-settings-no-echo` (OPEN, `set_transition_settings`
echoes none of `logicType`/`blendMode`/`disabled`), `E-post-process-setters-no-echo`
(OPEN, the five non-AA `post_process.set_*` setters echo only `{success}`),
`E-lighting-set-ao-exposure-no-echo` (IN-REVIEW), and
`E-pcg-add-node-echo-pin-labels` (OPEN, the sibling PCG create-handlers omit pin
labels). Per that family's stated convention ("filed per-method because the fix
is per-handler-response, not a shared util"), this is the missing member for
`pcg.set_self_pruning_settings` — a distinct method none of those tickets covers.

## Why this is the WRITE-path half, distinct from the judge's decompile bug

The judge filed `B-decompile-struct-subfield-dropped` (High) for this same task:
that ticket is the **READ path** — `pcg.decompile`'s struct export drops a
non-default sub-field (`bRandomizedPruning=false`) because it exports against the
zero-value instead of the CDO default. Its fix plumbs the CDO default into the
struct-export call.

This ticket is the **complementary WRITE-path ergonomic**: the setter's own
response doesn't confirm what it applied, so the only way to learn the write's
effect is a decompile round-trip — and that is exactly the detour that turned a
one-shot write into a recovery cycle here. The two are complementary fix surfaces
(the authoring handler's response JSON vs. the decompiler's struct emit); neither
subsumes the other. Even with the decompile bug fully fixed, a value-echo on the
setter would still be the cheaper way to confirm a self-pruning write without a
save+decompile, and — critically for this trace — an "equals class default"
signal in the echo is the thing that would have told the agent up front that its
"reasonable" knobs (`LargeToSmall/0.25/true`) were all defaults, before it spent
a full decompile discovering that (correctly-suppressed) defaults show nothing.

Also distinct from `E-pcg-set-self-pruning-enum-undiscoverable` (OPEN): that is
the **docs/discoverability** gap (the wiki overlay enumerates no `pruningType`
values); this is the **response-shape/echo** gap. The handler writes fine and the
value is known — the gap is that the applied values aren't returned.

## What it should do

`pcg.set_self_pruning_settings`'s success response should echo the fields it
actually applied — `pruningType` as its **string input vocabulary** (not a raw
enum index), `radiusSimilarityFactor` as a number, and `bRandomizedPruning` as a
bool — only for the params present in the call (it is an optional-param update
RPC). Cheap: a few `SetStringField`/`SetNumberField`/`SetBoolField` calls on the
`Resp` object the handler already builds, off the values it just wrote onto the
node's settings. To directly prevent the friction seen here, additionally flag
whether an applied value **equals the node's CDO default** (e.g. an
`appliedDefaults:[...]` list or a per-field `isDefault`), so a caller authoring a
diffable export learns immediately — from the write itself — that a knob at its
default will not surface in the decompile text, instead of discovering it only
after save + decompile.

**Workaround:** after `pcg.set_self_pruning_settings`, `asset.save` then
`pcg.decompile {assetPath}` (or `pcg.inspect`) and read the node's `Parameters`
struct back to confirm the write — the exact round-trip this task paid, and one
that is itself lossy today per `B-decompile-struct-subfield-dropped`.

## Friction evidence (this task — focus `pcg.decompile`, namespace `pcg`, `PCG_RockScatter` rock-scatter build, 17 RPCs, outcome tool_bug)

The trace was otherwise clean (17 RPCs, no errors, no retries elsewhere). The one
friction cluster: the agent's first `pcg.set_self_pruning_settings
{pruningType:LargeToSmall, radiusSimilarityFactor:0.25, bRandomizedPruning:true}`
returned `{"nodeId":"SelfPruning_0"}` and `asset.save` succeeded, but the
following `pcg.decompile` showed N3 with **no** knobs (only `Seed`) — because the
chosen values were all class defaults, which decompile (correctly) suppresses.
The setter response gave no way to know that. Agent SAY (verbatim): *"the
self-pruning knobs I set (LargeToSmall / 0.25 / true) are all class defaults, so
the decompiler omits them from N3's block — only Seed shows."* The agent then ran
a full recovery cycle — 2nd `set_self_pruning_settings {LargeToSmall, 0.5, false}`
+ 2nd `asset.save` + 2nd `pcg.decompile` — **3 extra RPCs** — before any knob
surfaced. Friction note (verbatim): *"my first set_self_pruning_settings used the
class-default knob values (LargeToSmall/0.25/true), which the decompiler omits as
defaults so no pruning knob appeared on N3's block — I had to re-set to non-default
reasonable values to make a knob surface."* An echo of the applied values (with a
default-equality flag) would have collapsed that discover-via-decompile detour
into the first call. Recurs once per self-pruning node whose settings a caller
needs to verify or diff.

## Not a duplicate of

- `B-decompile-struct-subfield-dropped` (OPEN, the judge's filing) — the READ-path
  decompile struct-export bug (drops non-default `bRandomizedPruning=false`). This
  is the WRITE-path setter that never confirms its values; complementary surface.
- `E-pcg-set-self-pruning-enum-undiscoverable` (OPEN) — docs/discoverability of the
  `pruningType` enum values; this is the response-echo shape, not the wiki.
- `E-pcg-add-node-echo-pin-labels` (OPEN) — the PCG **create** handlers omit pin
  labels; this is the **setter** omitting applied values. Same "mutator should
  carry the answer" shape, different handler group.
- `E-set-transition-settings-no-echo` / `E-post-process-setters-no-echo` /
  `E-lighting-set-ao-exposure-no-echo` — same no-echo family, other namespaces;
  filed per-method per that family's convention. None covers `pcg.set_self_pruning_settings`.

severity rationale: impact=response-shape gap that forces a readback round-trip (Low, recoverable via decompile/inspect) × reach=rare (PCG self-pruning authoring is a specific procedural-scatter path, not an every-session method) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle/process audit of the `pcg.decompile` `PCG_RockScatter` rock-scatter build (namespace `pcg`, 17 RPCs, outcome tool_bug; judge filed `B-decompile-struct-subfield-dropped` for the READ-path decompile struct bug). Distinct WRITE-path PROCESS finding: `pcg.set_self_pruning_settings` returns only `{"nodeId":"SelfPruning_0"}` and echoes none of the `pruningType`/`radiusSimilarityFactor`/`bRandomizedPruning` it applied, so the caller cannot confirm the write — or notice its values equal the node's class defaults — without a decompile round-trip. Root of the trace's only friction: the first setter call used values equal to class defaults (LargeToSmall/0.25/true), returned `{nodeId}` with no echo, and only after `asset.save` + `pcg.decompile` (which correctly suppresses defaults) did the agent learn nothing surfaced — forcing a 3-RPC recovery cycle (2nd set {LargeToSmall,0.5,false} + 2nd save + 2nd decompile). Fix: echo the applied values in the response (string `pruningType`, numeric factor, bool) for params present in the call, plus an equals-CDO-default flag so a diffable-export author learns up front that a default-valued knob won't appear in the decompile text. Same no-echo family as `E-set-transition-settings-no-echo`/`E-post-process-setters-no-echo`/`E-lighting-set-ao-exposure-no-echo`/`E-pcg-add-node-echo-pin-labels`, filed per-method per that convention. Deduped via ripgrep over OPEN+closed board (self_pruning / set_self_pruning / no-echo / readback): the only PCG self-pruning tickets are `B-decompile-struct-subfield-dropped` (decompile read bug, judge's), `E-pcg-set-self-pruning-enum-undiscoverable` (enum docs), and `E-pcg-add-node-echo-pin-labels` (create-handler pin labels) — none covers the setter's response-echo. Severity Low (recoverable via decompile/inspect; rare authoring path).
</content>
</invoke>
