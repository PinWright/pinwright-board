---
id: E-compile-bpir-idempotent-omits-guid-regen
title: "compile_bpir default-mode 'idempotent' docs omit that node GUIDs regenerate on every re-apply"
status: OPEN
severity: Low
category: ergonomic
tags: [bpir, docs, compile_bpir]
encounters: 2
lastSeen: 2026-06-29T00:25:45Z
---

# compile_bpir default-mode 'idempotent' docs omit that node GUIDs regenerate on every re-apply

`blueprint.bpir-gotchas.md` (and the `blueprint.compile_bpir` wiki page it feeds)
describe default `append`/`replace` mode as making retries **idempotent** — line 7
verbatim: *"Default `compile_bpir` retries are idempotent through the RPC. The public
handler maps default `mode: "append"` and legacy `mode: "replace"` to the same upsert
path, deleting matching entries before recreating them."* `bpir.entry-points.md` (line
70) reinforces the same "idempotency fix" framing.

That guarantee is true for entry/node **count** and **decompiled text** (the prior
`B-compile-bpir-retry-duplicates`, DONE, fixed and verified the no-duplicate-accumulation
property). But the same delete-then-recreate mechanism the docs cite means every authored
node gets a **fresh GUID** on each identical re-apply — the "idempotent" headline never
flags that **node identity is NOT preserved**. A reader who interprets "idempotent" as a
true no-op — someone diffing the asset across applies, holding a node's GUID as an external
reference, or expecting `compile_bpir` to short-circuit when nothing changed — is surprised.

**Evidence (this task — focus `blueprint.compile_bpir`, `/Game/BP_BpirIdempotentProbe`):**
two **byte-identical** default-mode applies of the same 3-entry BPIR (BeginPlay → helper
`ComputeGreeting` cross-entry call, custom event `OnPulse`, helper function; `@(x,y)` on
every node) produced fully **disjoint** `createdNodes` GUID sets — apply#1 createdNodes
`[5486FFAD, 4E10F3C7, 6291FB91, 6B7E72D9, 02B8DE6B, 8AED8E3C]` vs apply#2
`[C47BCAA1, 589FB225, B9C9E2BA, 9B214787, E4DF6E5F, 98338952]`. `blueprint.graph.get_nodes`
confirms every authored node's GUID regenerated across the re-apply (BeginPlay event
`5486FFAD→C47BCAA1`; `OnPulse` custom event `4E10F3C7→589FB225`; `ComputeGreeting`
FunctionEntry `22EC6EE2→24BADD43`), while two untouched ghost events (ActorBeginOverlap
`92F2734F`, Tick `59168CC9`) kept theirs. Meanwhile `decompile` #2 was byte-identical to
#1 and per-graph node counts/positions stayed stable (EventGraph 7, ComputeGreeting 2).
Friction note verbatim: *"the docs call append/upsert 'idempotent' meaning
no-duplicate-accumulation, but it is delete-then-recreate so it does NOT preserve node
GUIDs/identity, which a user reading 'retries are idempotent' would not expect."* No
workflow friction resulted (13 RPCs, zero errors/retries; the Attempt agent expected and
detected the churn) — this is a pure docs-precision gap.

**What it should do:** add one sentence to `Docs/wiki-src/blueprint.bpir-gotchas.md`
(and optionally `bpir.entry-points.md`) clarifying that here "idempotent" means **no
duplicate accumulation and a stable decompile + node positions**, but **node GUIDs are
NOT preserved** — nodes are deleted and re-created, so any external reference to a node's
GUID is invalidated on every re-apply, and the re-apply is not a hardware no-op.

**Docs page to improve:** `Docs/wiki-src/blueprint.bpir-gotchas.md` (line 7 idempotency
bullet); secondarily `Docs/wiki-src/bpir.entry-points.md` (idempotency-fix paragraph).

Distinct from (complements, does not duplicate):
- `B-compile-bpir-retry-duplicates` (DONE) — fixed the COUNT-stability/no-duplicate property; says nothing about GUID identity.
- `E-add-event-then-default-compile-bpir-unundoable` (IN-REVIEW) — same docs page + same "idempotent" wording, but its surprising consequence is **un-undoability**, not GUID regeneration. Different reader concern, different remedy sentence.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit, focus `blueprint.compile_bpir`, `/Game/BP_BpirIdempotentProbe`. CallAnalyzer + friction note agree on a docs-precision gap: default-mode `compile_bpir` is documented "idempotent" but its delete-then-recreate path regenerates every authored node's GUID on each identical re-apply, which the headline wording never flags. Proven by two byte-identical applies returning disjoint `createdNodes` GUID sets and `get_nodes` showing all authored-node GUIDs changed (BeginPlay `5486FFAD→C47BCAA1`, ComputeGreeting FunctionEntry `22EC6EE2→24BADD43`) while decompile output and `@(x,y)` positions stayed byte-identical and counts stayed 7/2. No workflow friction (13 RPCs, zero errors/retries) — Low severity, docs-only. Page to fix: `Docs/wiki-src/blueprint.bpir-gotchas.md` line 7 (and `bpir.entry-points.md`).
- `#2-additional-auto-layout-variant` `OPEN` reporter — Additional evidence (focus `blueprint.compile_bpir`, `/Game/BP_BpirUpsertIdem`): same GUID-regen reproduced in the **auto-layout** variant (3 custom_events + wired downstream nodes, NO `@(x,y)` on any node), confirming the churn is not tied to explicit positioning. Two identical default-`append` applies of the same 14-node BPIR kept node count (17), the event set (no dupes), all 17 node positions, and compile (UpToDate, 0 err) identical across runs, yet every one of the 14 BPIR-authored nodes got a brand-new `NodeGuid` on the second apply. New observable beyond `#1`: the regenerated nodes' UObject names also **increment** (e.g. `K2Node_CustomEvent_0`→`_3`, `CallFunction_0`→`_8`), so neither GUID nor object-name identity survives a re-apply — reinforcing that "idempotent" reads as a no-op but is delete-then-recreate. Auto-layout coordinates were themselves stable across re-applies (deterministic). Still pure docs-precision, Low; remedy sentence in `#1` unchanged.
