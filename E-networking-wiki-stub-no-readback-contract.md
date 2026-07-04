---
id: E-networking-wiki-stub-no-readback-contract
title: "networking wiki documents no return shape for get_networking_info — its branch-dependent readback contract (blueprintPath: 8 CDO fields + rpcFunctions[]/replicatedProperties[]; actorName: + role/remoteRole/hasAuthority + owner/ownerName) is discoverable only by reading NetworkingHandler.cpp"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, networking, get-networking-info, wiki, discoverability, readback]
encounters: 4
lastSeen: 2026-06-23T10:08:23Z
---

# `docs/wiki-src/networking.md` documents no return shape for `get_networking_info` — confirming what it reads back requires reading the handler C++

`get_networking_info` is the `networking` namespace's lone structured
**reader** (the round-trip confirm for its ~26 setters), but **no wiki page
documents its return shape**. The method's *parameters* (`blueprintPath`,
`actorName`) are discoverable from the auto-generated method-page schema, but
the *return* is documented nowhere: there is no `RPC_RETURN`-style spec
(`Handlers/ParamSpec.h` declares only the input macros `RPC_PARAM_REQ/OPT/DEF`),
so the auto-content layer **cannot** emit a return shape — a reader's output
contract has to be hand-authored in its overlay, exactly as the rich pages do
(`actor.md` documents `actor.describe`/`actor.get_components` return shapes in
prose; `blueprint.md` likewise). `get_networking_info` has no such overlay
section, so an agent that wants to know *which* fields round-trip — and which
setter state is simply not readable here — has to open
`NetworkingHandler.cpp` and read the two branches by hand.

## Current readback contract (verified against HEAD)
`get_networking_info` returns `{ success, networkingInfo: { … } }` whose fields
**differ by which input branch is taken** (exactly one of `blueprintPath` /
`actorName` is required):

- **`blueprintPath`** (reads the generated-class CDO + authored graphs/vars) —
  `NetworkingHandler.cpp:1426-1457`:
  - 8 CDO fields: `bReplicates`, `bAlwaysRelevant`, `bOnlyRelevantToOwner`,
    `netUpdateFrequency`, `minNetUpdateFrequency`, `netCullDistanceSquared`,
    `netPriority`, `netDormancy`.
  - `rpcFunctions[]` (one per `FUNC_Net` function): `name`, `rpcType`,
    `reliable`, `withValidation` (`BuildRpcFunctionsArray` :179-206).
  - `replicatedProperties[]` (one per `CPF_Net` variable): `name`, `replicated`,
    `replicatedUsing` (RepNotify function name, or `null`), `replicationCondition`
    (`BuildReplicatedPropertiesArray` :212-237).
  - Does **not** emit `role`/`remoteRole`/`hasAuthority` or `owner` — those are
    the actor-instance branch only.
- **`actorName`** (reads the live placed actor instance) —
  `NetworkingHandler.cpp:1458-1499`:
  - the same 8 CDO fields, **plus** `role`, `remoteRole`, `hasAuthority`
    (:1480-1482) and `owner`/`ownerName` (`null` when unowned, :1488-1498).
  - Does **not** emit `rpcFunctions[]`/`replicatedProperties[]` — a placed actor
    exposes no authored Blueprint graphs/vars to walk (those come from the
    `blueprintPath` branch).

**Not in the readback at all (write-only setters — do not round-trip here):**
push-model (`configure_push_model` stamps `PushModel` metadata, which no
`replicatedProperties[]` entry surfaces) and replicated-movement
(`configure_replicated_movement` sets `bReplicateMovement`, emitted by neither
branch). An agent should learn from docs — not a source dive — that these two
states are unconfirmable via `get_networking_info`, so their absence from the
payload is the contract, not a call mistake.

## Not an "outlier stub" — the fix is a method H3 section, not a fat prelude
The ticket originally framed `networking.md` (523 B, a 2-sentence prelude) as a
stub *outlier* versus richer siblings. That framing is wrong: thin branch
preludes are the **house style** (`physics.md` 452 B, `object.md` 391 B,
`session.md` 456 B, `lighting.md` 707 B are all comparable), and the plugin's
wiki rules cap a namespace prelude at ~2 sentences. The rich pages
(`actor.md`, `blueprint.md`) are large only because they carry per-method
`### namespace.method` H3 sections, which render **on the method page only**
(not the namespace prelude, not the root index). So the correct fix documents
`get_networking_info`'s return shape in a `### networking.get_networking_info`
H3 overlay section — it surfaces exactly when an agent reads that method's
page, costs nothing on the namespace/root indices, and matches how the rich
pages document their readers.

## What it should do
Add a `### networking.get_networking_info` section to
`docs/wiki-src/networking.md` documenting the branch-dependent return shape
above: the 8 CDO fields, the `rpcFunctions[]`/`replicatedProperties[]` arrays
and their per-entry fields on the `blueprintPath` branch, the
`role`/`remoteRole`/`hasAuthority` + `owner`/`ownerName` fields on the
`actorName` branch, and an explicit note that push-model
(`configure_push_model`) and replicated-movement
(`configure_replicated_movement`) are write-only and do **not** appear in the
readback — so an agent confirms the round-trip (or rules it out) from docs, not
from `NetworkingHandler.cpp`.

## Evidence
Task `networking.set_property_replicated` (focus): a clean 12-call build of
`/Game/Multiplayer/BP_HealthPickup` (create → add_variable ×2 →
set_property_replicated → set_replicated_using → set_replication_condition →
create_rpc_function → set_rpc_reliability → get_networking_info → compile →
get_networking_info). **Zero is_error, zero retries, zero wiki-nav loops,
no python.execute fallback** — every mutation was smooth. The *only*
friction was at verification, and it was a docs one: from the friction note,
"I confirmed via the plugin handler source (NetworkingHandler.cpp) that for a
blueprintPath it emits … Needing to read the plugin C++ to confirm this is
itself the gap." The networking wiki page offered no return-shape contract to
consult, forcing the source read. (The specific fields the early encounters
called "structurally unsatisfiable" — per-RPC/per-property detail, owner — have
since LANDED via the sibling `F-networking-info-no-rpc-detail` fix `d96d57b`;
the residual gap is that the *documentation* of the now-richer readback shape
still does not exist.)

## History
- `#1-initial-audit` `OPEN` reporter — Filed as the docs/discoverability sibling of `F-networking-info-no-rpc-detail`. On a fully-clean task (12 calls, no errors/retries/wiki-nav/python fallback), the sole friction was confirming `get_networking_info`'s blueprintPath readback contract: `docs/wiki-src/networking.md` is a 4-line prose stub (vs. actor.md ~5 KB / asset.md ~26 KB / blueprint.md ~39 KB) that names no method and documents no return shape, so the auditor had to read `NetworkingHandler.cpp` ~line 1240 to confirm the blueprintPath branch emits only 8 actor-CDO fields. Names the page to improve: `docs/wiki-src/networking.md` — add a `get_networking_info` section documenting branch-dependent output and the CDO-only blueprintPath contract.
- `#3-additional-coin-pickup-blueprintpath-source-read` `OPEN` reporter — Additional cross-task evidence on the **blueprintPath branch** (same branch `#1` cites), from a fully-clean end-to-end build of a freshly-created Actor BP `/Game/Multiplayer/BP_CoinPickup` (25 calls: the full networking sequence — `set_net_role(ROLE_Authority)` → `add_variable(CurrentValue,int)` → `set_property_replicated` → `set_replicated_using(OnRep_CurrentValue)` → `set_replication_condition(COND_OwnerOnly)` → `create_rpc_function(ServerCollect,Server)` + `set_rpc_reliability(true)` → `create_rpc_function(MulticastPlayPickupFX,NetMulticast,unreliable)` → `configure_net_update_frequency(30/5)` → `get_networking_info` → `blueprint.inspect`). **Zero is_error, zero retries, zero python.execute fallback**; up-front method discovery was smooth (each method's wiki page read once, no nav loops). The only friction was again at verification and again purely a docs one: the friction note states the readback gap was confirmed "against the handler source (NetworkingHandler.cpp ~L1276-1299, last resort) and the wiki" — i.e. the auditor had to open the C++ blueprintPath branch (lines 1276-1299, the same branch `#1` cites at the line-drifted `~1240`) because `docs/wiki-src/networking.md` documents no return shape. Sharpens the page-to-improve ask with a concrete consequence the stub hides: on the blueprintPath branch **`role` is never emitted** (it exists only on the actor branch), so a task that asks to "confirm ROLE_Authority via get_networking_info on the blueprint" is structurally unsatisfiable — and nothing in the wiki warns of this, forcing the source read to prove it's a real contract limit and not a param mistake. The `get_networking_info` section must state, per branch, exactly which fields each emits (blueprintPath: 8 CDO fields, **no role**; actorName: adds role/remoteRole/hasAuthority, **no owner**) so an agent can rule out the round-trip from docs alone. (Cross-links capability sibling `F-networking-info-no-rpc-detail` `#5`, same task; that ticket proposes adding the missing fields, this one covers documenting the current contract regardless.)
- `#4-additional-push-model-source-read` `OPEN` reporter — Additional cross-task evidence on the **blueprintPath branch**, this time for the **push-model** state (`networking.configure_push_model`) — a setter neither `#1` (frequency) nor `#3` (role) covered. Task `networking.configure_push_model` (focus), a fully-clean 11-call build of a freshly-created Actor BP `/Game/Multiplayer/BP_ReplicatedPickup` (create → add_variable ×2 (ChargesRemaining int=3, bIsActive bool=true, both replicated+public) → compile → configure_push_model{usePushModel:true} → configure_net_update_frequency{30/5} → set_replication_condition{ChargesRemaining,COND_SkipOwner} → get_networking_info → blueprint.get). **Zero is_error, zero retries, zero python.execute fallback**; up-front discovery was the normal 2 wiki-navs (`networking` namespace + root index, everything found first try — no nav loop). The only friction was again at verification and again purely a docs one: the friction note states the readback gap was confirmed only after "I had to read the plugin's NetworkingHandler.cpp (last-resort source dive) to confirm the readback structurally lacks the field rather than my call being wrong; the get_networking_info wiki page also does not document the response shape, which would have flagged the gap sooner." Concrete consequence the stub hides: even on THIS ticket's IN-REVIEW build (where `replicatedProperties[]` is now present), each entry emits only `name`/`replicated`/`replicatedUsing`/`replicationCondition` — there is **no push-model field** (NetworkingHandler.cpp `BuildReplicatedPropertiesArray` :204; the metadata `configure_push_model` writes is `VarDesc.SetMetaData("PushModel","true")` at :1033), so "confirm push model is enabled" via `get_networking_info` is structurally unsatisfiable and the wiki gives no readback contract to rule it out — forcing a third, separate source read (the `BuildReplicatedPropertiesArray` body, distinct from `#1`'s CDO branch and `#2`'s actor branch). Sharpens the page-to-improve ask: the `get_networking_info` section in `docs/wiki-src/networking.md` must enumerate, per `replicatedProperties[]` entry, exactly which fields are emitted (`name`/`replicated`/`replicatedUsing`/`replicationCondition`) and explicitly note that **push-model is not among them**, so an agent confirms from docs (not source) that push model can't be read back here. (Cross-links capability sibling `F-networking-info-no-rpc-detail` `#7`, same task — that ticket proposes adding a per-entry `pushModel` bool; this docs ticket covers documenting the current contract regardless of whether the field lands.)
- `#2-additional-actor-branch-source-read` `OPEN` reporter — Additional evidence that the undocumented-readback friction extends to the **actor branch** (`actorName` form), not just the blueprintPath branch `#1` cited. Task `networking.check_is_locally_controlled` (focus), a fully-clean 12-call actor-networking audit against `ExampleProjectWelcome` (actor.list → check_is_locally_controlled ×3 → check_has_authority → set_owner set/clear round-trip → get_networking_info ×3): **zero is_error, zero retries, one wiki-nav (root index, found everything first try), no python.execute fallback** — method discovery was smooth. The lone friction was again at verification and again a docs one: to confirm the set_owner round-trip is unconfirmable through `get_networking_info`, the auditor read the plugin C++ as a last resort ("I read the plugin C++ (last resort) to confirm the field is genuinely absent rather than a param mistake on my end"). Critically this was a **different source location** than `#1` — the **actor branch at `NetworkingHandler.cpp:1308-1324`** (vs. the blueprintPath branch ~1240) — so the page's missing readback contract forces a *second, separate* source read for the actor-instance form. Sharpens the page-to-improve ask: the `get_networking_info` section in `docs/wiki-src/networking.md` must document **both** input branches' return shapes — the actor branch (`actorName`) emits role/remoteRole/hasAuthority/relevance/CDO fields and **no owner**, distinct from the blueprintPath branch — so an agent can confirm from docs (not source) that the `set_owner`/`attach`/`detach` round-trip is not readable here. (Cross-links the capability sibling `F-networking-info-no-rpc-detail` `#4`, which proposes adding the actor-branch `owner`/`ownerName` field; this docs ticket covers documenting the contract regardless of whether that feature lands.)
- `#5-reword-and-implement` `IN-REVIEW` developer — REWORD then implemented as GO. Independently verified against plugin HEAD that the ticket's central premise was STALE: the sibling capability fix `F-networking-info-no-rpc-detail` (`d96d57b`, ancestor of HEAD) already landed, so `get_networking_info`'s blueprintPath branch now emits `rpcFunctions[]` and `replicatedProperties[]` beyond the 8 CDO fields (`NetworkingHandler.cpp:1455-1456`), and the actorName branch emits `role`/`remoteRole`/`hasAuthority` (:1480-1482) and `owner`/`ownerName` (:1491-1497) — the "8 CDO fields only / round-trip impossible" framing was false. Also corrected the false "stub is an outlier, not house style" claim (physics.md 452 B / object.md 391 B / session.md 456 B / lighting.md 707 B are all comparably thin branch preludes; thin preludes ARE the house style). The residual gap is real and un-actioned: `get_networking_info`'s RETURN shape is documented on no page, and — because `ParamSpec.h` has no `RPC_RETURN` macro — a reader's output contract must be hand-authored in its overlay (as `actor.md`/`blueprint.md` do). Fix: added a `### networking.get_networking_info` H3 overlay section to `Plugins/PinWright/Docs/wiki-src/networking.md` documenting the CURRENT branch-dependent readback contract (blueprintPath: 8 CDO fields + `rpcFunctions[]{name,rpcType,reliable,withValidation}` + `replicatedProperties[]{name,replicated,replicatedUsing,replicationCondition}`; actorName: + role/remoteRole/hasAuthority + owner/ownerName) and an explicit note that the write-only setters `configure_push_model` and `configure_replicated_movement` do NOT round-trip through this reader. The H3 section surfaces on the `networking.get_networking_info` method page only (not the namespace prelude / root index), matching house style. Regression test `PinWright.infra.wiki_handler.Method.NetworkingInfoReadback` (`Source/PinWright/Private/Tests/Infra/TestNetworkingInfoReadbackDocs.cpp`) renders the method page through the live `WikiHandler::RenderPage` path and asserts the overlay-exclusive readback markers survive; reverting the H3 section makes the assertions fail.
</content>
</invoke>
