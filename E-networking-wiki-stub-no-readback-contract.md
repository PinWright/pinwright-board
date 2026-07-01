---
id: E-networking-wiki-stub-no-readback-contract
title: "networking wiki page is a one-line stub — get_networking_info's blueprintPath readback contract is undiscoverable without reading NetworkingHandler.cpp"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, networking, get-networking-info, wiki, discoverability, readback]
encounters: 4
lastSeen: 2026-06-23T10:08:23Z
---

# `docs/wiki-src/networking.md` doesn't document `get_networking_info`'s readback shape — confirming what it returns requires reading the handler C++

The `networking` namespace has 27 methods, but its wiki overlay page
`docs/wiki-src/networking.md` is a **single content sentence** (4 lines
total) — a blurb that enumerates capabilities in prose but names **no
individual method**, lists **no parameters**, and crucially describes **no
return/readback shape** for any reader. Sibling namespace pages are far
richer (`actor.md` ~5 KB, `asset.md` ~26 KB, `blueprint.md` ~39 KB,
`blueprint.graph.md` ~23 KB), so the bare networking stub is an outlier,
not a house style.

The concrete process cost: when a task asks for the standard build-then-read-back
loop ("wire up replication/RPCs, then `get_networking_info` to confirm"),
there is **no docs surface** that tells the caller `get_networking_info`
on a `blueprintPath` returns only 8 actor-CDO fields
(`bReplicates`/`bAlwaysRelevant`/`bOnlyRelevantToOwner`/`netUpdateFrequency`/
`minNetUpdateFrequency`/`netCullDistanceSquared`/`netPriority`/`netDormancy`)
and nothing per-property or per-RPC. The only way to learn the readback
contract — and to confirm the round-trip is impossible — was to **open
`NetworkingHandler.cpp` (~line 1240) and read the blueprintPath branch by
hand.** Having to read plugin C++ to discover a documented method's output
shape is the discoverability gap.

This is the PROCESS/discoverability sibling of the capability ticket
`F-networking-info-no-rpc-detail` (which proposes *adding* `rpcFunctions[]`
and `replicatedProperties[]` to the payload). Distinct angle: even if that
feature is deferred or wontfixed, the wiki page should at minimum **document
the current readback contract** so an agent learns from docs (not from
source) that the blueprintPath branch is CDO-only and that per-property
replication condition / RepNotify / RPC flags are not surfaced. If the
feature lands, the same page is where the new fields must be documented.

## What's awkward
- `networking.md` is prose-only with zero per-method detail across 27 methods.
- No documented return shape for the namespace's lone reader,
  `get_networking_info`, nor any note that its output differs by argument
  (`blueprintPath` → 8 actor-CDO fields; world/actor branches differ).
- Confirming the limitation required reading the handler source, the exact
  friction the auditor flagged ("Needing to read the plugin C++ to confirm
  this is itself the gap").

## What it should do
Expand `docs/wiki-src/networking.md` to a method-level overlay matching
sibling pages: at minimum a `get_networking_info` section that states its
input branches and documents the blueprintPath return as the 8 actor-CDO
fields only — explicitly calling out that per-property replication
condition, RepNotify (`ReplicatedUsing`), and per-RPC reliability/validation/type
are **not** part of the readback (cross-link `F-networking-info-no-rpc-detail`).
(The wiki edit itself is a downstream docs process, not this audit's job —
this ticket only names the page to improve.)

## Evidence
Task `networking.set_property_replicated` (focus): a clean 12-call build of
`/Game/Multiplayer/BP_HealthPickup` (create → add_variable ×2 →
set_property_replicated → set_replicated_using → set_replication_condition →
create_rpc_function → set_rpc_reliability → get_networking_info → compile →
get_networking_info). **Zero is_error, zero retries, zero wiki-nav loops,
no python.execute fallback** — every mutation was smooth. The *only*
friction was at verification, and it was a docs one: from the friction note,
"I confirmed via the plugin handler source (NetworkingHandler.cpp ~line 1240)
that for a blueprintPath it emits exactly 8 actor-CDO fields … Needing to
read the plugin C++ to confirm this is itself the gap." The networking wiki
page offered nothing to consult, forcing the source read.

## History
- `#1-initial-audit` `OPEN` reporter — Filed as the docs/discoverability sibling of `F-networking-info-no-rpc-detail`. On a fully-clean task (12 calls, no errors/retries/wiki-nav/python fallback), the sole friction was confirming `get_networking_info`'s blueprintPath readback contract: `docs/wiki-src/networking.md` is a 4-line prose stub (vs. actor.md ~5 KB / asset.md ~26 KB / blueprint.md ~39 KB) that names no method and documents no return shape, so the auditor had to read `NetworkingHandler.cpp` ~line 1240 to confirm the blueprintPath branch emits only 8 actor-CDO fields. Names the page to improve: `docs/wiki-src/networking.md` — add a `get_networking_info` section documenting branch-dependent output and the CDO-only blueprintPath contract.
- `#3-additional-coin-pickup-blueprintpath-source-read` `OPEN` reporter — Additional cross-task evidence on the **blueprintPath branch** (same branch `#1` cites), from a fully-clean end-to-end build of a freshly-created Actor BP `/Game/Multiplayer/BP_CoinPickup` (25 calls: the full networking sequence — `set_net_role(ROLE_Authority)` → `add_variable(CurrentValue,int)` → `set_property_replicated` → `set_replicated_using(OnRep_CurrentValue)` → `set_replication_condition(COND_OwnerOnly)` → `create_rpc_function(ServerCollect,Server)` + `set_rpc_reliability(true)` → `create_rpc_function(MulticastPlayPickupFX,NetMulticast,unreliable)` → `configure_net_update_frequency(30/5)` → `get_networking_info` → `blueprint.inspect`). **Zero is_error, zero retries, zero python.execute fallback**; up-front method discovery was smooth (each method's wiki page read once, no nav loops). The only friction was again at verification and again purely a docs one: the friction note states the readback gap was confirmed "against the handler source (NetworkingHandler.cpp ~L1276-1299, last resort) and the wiki" — i.e. the auditor had to open the C++ blueprintPath branch (lines 1276-1299, the same branch `#1` cites at the line-drifted `~1240`) because `docs/wiki-src/networking.md` documents no return shape. Sharpens the page-to-improve ask with a concrete consequence the stub hides: on the blueprintPath branch **`role` is never emitted** (it exists only on the actor branch), so a task that asks to "confirm ROLE_Authority via get_networking_info on the blueprint" is structurally unsatisfiable — and nothing in the wiki warns of this, forcing the source read to prove it's a real contract limit and not a param mistake. The `get_networking_info` section must state, per branch, exactly which fields each emits (blueprintPath: 8 CDO fields, **no role**; actorName: adds role/remoteRole/hasAuthority, **no owner**) so an agent can rule out the round-trip from docs alone. (Cross-links capability sibling `F-networking-info-no-rpc-detail` `#5`, same task; that ticket proposes adding the missing fields, this one covers documenting the current contract regardless.)
- `#4-additional-push-model-source-read` `OPEN` reporter — Additional cross-task evidence on the **blueprintPath branch**, this time for the **push-model** state (`networking.configure_push_model`) — a setter neither `#1` (frequency) nor `#3` (role) covered. Task `networking.configure_push_model` (focus), a fully-clean 11-call build of a freshly-created Actor BP `/Game/Multiplayer/BP_ReplicatedPickup` (create → add_variable ×2 (ChargesRemaining int=3, bIsActive bool=true, both replicated+public) → compile → configure_push_model{usePushModel:true} → configure_net_update_frequency{30/5} → set_replication_condition{ChargesRemaining,COND_SkipOwner} → get_networking_info → blueprint.get). **Zero is_error, zero retries, zero python.execute fallback**; up-front discovery was the normal 2 wiki-navs (`networking` namespace + root index, everything found first try — no nav loop). The only friction was again at verification and again purely a docs one: the friction note states the readback gap was confirmed only after "I had to read the plugin's NetworkingHandler.cpp (last-resort source dive) to confirm the readback structurally lacks the field rather than my call being wrong; the get_networking_info wiki page also does not document the response shape, which would have flagged the gap sooner." Concrete consequence the stub hides: even on THIS ticket's IN-REVIEW build (where `replicatedProperties[]` is now present), each entry emits only `name`/`replicated`/`replicatedUsing`/`replicationCondition` — there is **no push-model field** (NetworkingHandler.cpp `BuildReplicatedPropertiesArray` :204; the metadata `configure_push_model` writes is `VarDesc.SetMetaData("PushModel","true")` at :1033), so "confirm push model is enabled" via `get_networking_info` is structurally unsatisfiable and the wiki gives no readback contract to rule it out — forcing a third, separate source read (the `BuildReplicatedPropertiesArray` body, distinct from `#1`'s CDO branch and `#2`'s actor branch). Sharpens the page-to-improve ask: the `get_networking_info` section in `docs/wiki-src/networking.md` must enumerate, per `replicatedProperties[]` entry, exactly which fields are emitted (`name`/`replicated`/`replicatedUsing`/`replicationCondition`) and explicitly note that **push-model is not among them**, so an agent confirms from docs (not source) that push model can't be read back here. (Cross-links capability sibling `F-networking-info-no-rpc-detail` `#7`, same task — that ticket proposes adding a per-entry `pushModel` bool; this docs ticket covers documenting the current contract regardless of whether the field lands.)
- `#2-additional-actor-branch-source-read` `OPEN` reporter — Additional evidence that the undocumented-readback friction extends to the **actor branch** (`actorName` form), not just the blueprintPath branch `#1` cited. Task `networking.check_is_locally_controlled` (focus), a fully-clean 12-call actor-networking audit against `ExampleProjectWelcome` (actor.list → check_is_locally_controlled ×3 → check_has_authority → set_owner set/clear round-trip → get_networking_info ×3): **zero is_error, zero retries, one wiki-nav (root index, found everything first try), no python.execute fallback** — method discovery was smooth. The lone friction was again at verification and again a docs one: to confirm the set_owner round-trip is unconfirmable through `get_networking_info`, the auditor read the plugin C++ as a last resort ("I read the plugin C++ (last resort) to confirm the field is genuinely absent rather than a param mistake on my end"). Critically this was a **different source location** than `#1` — the **actor branch at `NetworkingHandler.cpp:1308-1324`** (vs. the blueprintPath branch ~1240) — so the page's missing readback contract forces a *second, separate* source read for the actor-instance form. Sharpens the page-to-improve ask: the `get_networking_info` section in `docs/wiki-src/networking.md` must document **both** input branches' return shapes — the actor branch (`actorName`) emits role/remoteRole/hasAuthority/relevance/CDO fields and **no owner**, distinct from the blueprintPath branch — so an agent can confirm from docs (not source) that the `set_owner`/`attach`/`detach` round-trip is not readable here. (Cross-links the capability sibling `F-networking-info-no-rpc-detail` `#4`, which proposes adding the actor-branch `owner`/`ownerName` field; this docs ticket covers documenting the contract regardless of whether that feature lands.)
