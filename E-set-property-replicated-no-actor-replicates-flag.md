---
id: E-set-property-replicated-no-actor-replicates-flag
title: "networking.set_property_replicated marks a property Replicated but its success response never warns that the actor's bReplicates is still false, so the property silently won't replicate"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [networking, set-property-replicated, breplicates, set-replication, cross-namespace, discoverability, inert-replication]
encounters: 3
lastSeen: 2026-07-11T14:33:32+03:00
---

# `networking.set_property_replicated` sets `CPF_Net` on the property but its `success:true` response says nothing about the actor's `bReplicates` — when that flag is still false the property won't replicate, and the dependency is invisible at the call that introduced it

**Rescoped (reword):** the underlying observation is real and distinct from
the reader ticket `F-networking-info-no-rpc-detail` — but the original fix
preference (Option 1, *auto*-flip `CDO->SetReplicates(true)` from inside a
per-property setter) over-reaches: it would silently change a persistent
actor-level CDO toggle the caller never asked for, mutate state for every
existing caller, and make `get_networking_info` report a `bReplicates` the
user never set. The landed remedy is the **non-mutating** Option 2: when the
property is flagged replicated but the actor CDO's `bReplicates` is still
false, the `success` response carries `bReplicatesEnabled:false` plus a
`warning` naming the follow-up verb (`misc.set_replication {replicates:true}`
or `networking.set_net_role {role:ROLE_Authority}`). No CDO mutation, no
contract change for callers that already enabled replication. The docs half
(former Option 3) is folded into the already-open
`E-networking-wiki-stub-no-readback-contract` rather than duplicated here.

The whole `networking.*` replication-authoring sequence an agent naturally runs
— `set_property_replicated`, `set_replicated_using`, `set_replication_condition`,
`create_rpc_function` — operates **per-property / per-function** and **never
touches the actor-level `bReplicates` flag**. `networking.set_property_replicated`
in particular only does `Property->SetPropertyFlags(CPF_Net)`
(`NetworkingHandler.cpp:260`); it does not call `CDO->SetReplicates(true)`. So
after marking every property replicated, the actor still has `bReplicates=false`,
and in Unreal a `Replicated` property on an actor whose `bReplicates` is false
**does not replicate at all** — the replication the task asked for is silently
inert.

There is no flag-flipping verb anywhere in the `networking.*` namespace. The only
way to enable actor replication through the MCP is `misc.set_replication`
(`MiscHandler.cpp:410`, `CDO->SetReplicates(...)` at :444) — a verb in the
**`misc` / Utility** namespace, which a caller working entirely inside
`networking.*` (because the task is "set up replication") has no reason to look
for. (The one networking verb that *does* flip `bReplicates` as a side effect is
`networking.set_net_role` with `ROLE_Authority` — `NetworkingHandler.cpp:1306` —
but that is a role setter, not an obvious "make this actor replicate" verb, and
the task here didn't call for a role change.) So the single intent "make this
actor's properties replicate" is split across two namespaces, and the
`networking.*` half silently does only the property-flag half.

## Why this bites even on an otherwise-clean task

`set_property_replicated` returns `success:true` with a confident message
(`"Property HealAmount replication set to true"`) and nothing in the response
hints that the actor itself still won't replicate. The gap only becomes visible
at verification: `get_networking_info` reports `bReplicates=false` despite the
replicated properties, which is the success-check item that fails. The agent then
has to *discover* `misc.set_replication` (out-of-namespace) and run it before the
"full networking info" the task asked for is correct. (Note `F-networking-info-no-rpc-detail`
`#5`/`#7` independently logged this same `bReplicates=false` observation, but
treats it as a **reader** gap and explicitly calls the `set_property_replicated`
setter "works correctly — NOT a bug." This ticket is the distinct **setter
ergonomics / cross-namespace discovery** angle: the setter succeeds yet leaves
the actor non-replicating, and the enabling verb is in another namespace the
networking surface never names.)

## Fix (landed)

**Surface the dependency without mutating it.** `set_property_replicated`, after
stamping `CPF_Net` on the property, inspects the actor CDO's replication state.
When `replicated:true` was requested but the actor's `bReplicates` is still
false, the `success` response now carries:
- `bReplicatesEnabled:false` — the actor-level state, so the gap is machine-visible
  at the call that introduced it instead of only at a downstream `get_networking_info`;
- a `warning` string naming the required follow-up verb
  (`misc.set_replication {replicates:true}` or `networking.set_net_role {role:ROLE_Authority}`).

When the actor already replicates (e.g. the caller ran `misc.set_replication`
first), `bReplicatesEnabled:true` and no `warning` — the response is otherwise
unchanged, so existing callers are unaffected. The handler does **not** call
`CDO->SetReplicates(...)` (rejected Option 1): a per-property setter must not
silently flip a persistent actor-level toggle the caller never asked for.

### Rejected / deferred
- **Auto-enable (former Option 1).** Mutating `bReplicates` from a property
  setter changes the verb's contract for every existing caller and would make
  `get_networking_info` report a state the user never set. Rejected.
- **Docs note (former Option 3).** The networking wiki overlay
  (`docs/wiki-src/networking.md`, a one-sentence stub) should state that
  property-level replication setters do **not** enable actor replication. That
  doc work is folded into the already-open
  `E-networking-wiki-stub-no-readback-contract`, not duplicated here.

**Workaround (pre-fix):** after the `networking.*` replication setters, call
`misc.set_replication {blueprintPath, replicates:true}` (or
`networking.set_net_role {ROLE_Authority}`) to flip the actor's `bReplicates`
before reading `get_networking_info`.

## Evidence

Task `networking.set_property_replicated` (the seed/focus), a `/Game/Multiplayer/BP_HealthPickup`
multiplayer-pickup build. The mutation path was otherwise smooth, but the
friction note's point (2) verbatim: *"Marking properties replicated does NOT flip
the actor's bReplicates flag (get_networking_info showed bReplicates=false); I had
to discover misc.set_replication to satisfy that part of the success check."* Call
log: `set_property_replicated(HealAmount,true)` and `set_property_replicated(bIsActive,true)`
both `success:true`, but the post-build `get_networking_info` showed
`bReplicates=false`; the agent then did a `misc.md` + `misc.set_replication.md`
wiki-nav (a cross-namespace doc hunt) followed by
`misc.set_replication {replicates:true, replicateMovement:false}` to enable
actor replication, then re-compiled and re-read. Source confirms the split:
`set_property_replicated` only sets `CPF_Net` (`NetworkingHandler.cpp:260`, no
`SetReplicates`); the only `SetReplicates(true)` calls are in `misc.set_replication`
(`MiscHandler.cpp:444`) and the `ROLE_Authority` branch of `set_net_role`
(`NetworkingHandler.cpp:1306`).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `networking.set_property_replicated` BP_HealthPickup multiplayer-pickup build (outcome tool_bug; the judge filed the unrelated RPC-input type-token bug `B-rpc-input-class-path-silent-wildcard`, this task's other friction point). Distinct setter-ergonomics / cross-namespace-discovery angle: `set_property_replicated` only stamps `CPF_Net` on the property (`NetworkingHandler.cpp:260`) and never flips the actor CDO's `bReplicates`, so after marking both properties replicated the actor still had `bReplicates=false` (replication functionally inert) and the only enabling verb — `misc.set_replication` (`MiscHandler.cpp:444`, `CDO->SetReplicates`) — lives in the **`misc`/Utility** namespace the `networking.*` surface never points to, forcing a cross-namespace doc hunt (`misc.md` + `misc.set_replication.md` wiki-nav) before `get_networking_info` was correct. Friction note (verbatim): "Marking properties replicated does NOT flip the actor's bReplicates flag (get_networking_info showed bReplicates=false); I had to discover misc.set_replication to satisfy that part of the success check." Proposed: auto-enable `CDO->SetReplicates(true)` when the first `CPF_Net` property is flagged on a non-replicating actor (echo `bReplicatesEnabled:true`), or at minimum return a `warning` naming `misc.set_replication`/`set_net_role ROLE_Authority` as the required follow-up; plus a docs note on `docs/wiki-src/networking.md` (currently a stub, see `E-networking-wiki-stub-no-readback-contract`). Distinct from `F-networking-info-no-rpc-detail` `#5`/`#7` (which logs the same `bReplicates=false` readback but as a **reader** gap and calls the setter correct) — this owns the setter/discovery friction.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded (title/body/**Fix:**) to drop the rejected Option-1 auto-enable: a per-property setter must not silently flip the persistent actor-level `bReplicates` (it would change the verb's contract for every caller and make `get_networking_info` report a state the user never set). Landed the non-mutating Option 2 instead. **Code:** `networking.set_property_replicated` (`Source/EditorAutomationRpcGateway/Private/Handlers/Networking/NetworkingHandler.cpp`) — after stamping `CPF_Net`, resolves the actor CDO and adds `bReplicatesEnabled` (= `CDO->GetIsReplicated()`) to the success response; when `replicated:true` was requested but `bReplicates` is still false it also adds a `warning` naming `misc.set_replication {replicates:true}` / `networking.set_net_role {role:ROLE_Authority}` as the follow-up. No `CDO->SetReplicates(...)` — the actor flag is reported, never mutated; callers that already enabled replication see `bReplicatesEnabled:true` and no warning (response otherwise unchanged). Docs half (former Option 3) folded into `E-networking-wiki-stub-no-readback-contract`, not duplicated. **Test:** `FNetworkingSetPropertyReplicatedWarnsWhenActorNotReplicating` in `Source/EditorAutomationRpcGateway/Private/Tests/Networking/TestNetworkingHandlers.cpp` — builds a real package-backed `AActor` blueprint, marks a property replicated while `bReplicates=false` and asserts the response carries `bReplicatesEnabled:false` + a non-empty `warning` AND that the CDO's `bReplicates` was NOT auto-flipped; then enables replication via `misc.set_replication` and re-marks the property, asserting `bReplicatesEnabled:true` and no `warning`. Fails if the surfacing is reverted or if the setter starts silently auto-enabling `bReplicates`.
- `#3-post-fix-evidence` `IN-REVIEW` reporter — Recurred (post-fix sighting, fix observed working) on a `/Game/Multiplayer/BP_ReplicatedPlayerState` Health+Ammo replication build (networking namespace, outcome clean). The Option-2 surfaced `warning` is now present in the call log: both `set_property_replicated(Health,true)` and `set_property_replicated(CurrentAmmo,true)` returned `success:true` carrying the verbatim warning *"actor bReplicates still false, <prop> will NOT replicate"*, and the agent acted on it directly — next call was `misc.set_replication {replicates:true, replicateMovement:false}` (summary notes "(clears warning)"), no cross-namespace doc hunt needed this time (the earlier `misc.md`/`misc.set_replication.md` wiki-nav from `#1` did not recur). Friction note (verbatim): *"set_property_replicated warned that nothing would actually replicate because actor bReplicates was still false — the property-level calls don't enable actor replication, so I had to discover and call misc.set_replication separately to make the 'replicated Actor Blueprint' real; otherwise smooth."* Confirms the warning makes the dependency discoverable at the introducing call (the fix's intent) — residual friction is now just the two-namespace split itself (one extra `misc.set_replication` call), the irreducible part Option 2 deliberately left rather than auto-mutating `bReplicates`. No new ticket; appended as recurrence evidence. The lingering docs gap (networking wiki should state property-level setters don't enable actor replication) remains owned by `E-networking-wiki-stub-no-readback-contract`.
- `#4-liveness` `IN-REVIEW` reporter — Post-fix reconfirm, Option-2 warning still observed working. Clean `/Game/B_HealthPickup` co-op health-pickup build (focus `networking.set_net_dormancy`, outcome clean, judge filed nothing). `set_property_replicated(Health,true)` returned `success:true` carrying the verbatim warning "actor bReplicates still false, Health will NOT replicate until enabled"; the agent resolved it in one `misc.set_replication {replicates:true, replicateMovement:false}` call (`misc.set_replication.md` was read up front in the initial wiki-nav batch, so no reactive mid-task doc-hunt loop recurred), then `asset.save` + `get_networking_info` confirmed netDormancy=DORM_Initial, bAlwaysRelevant=true, Health replicated=true, bReplicates=true all stuck. Same friction as `#3`, no new angle — residual is the irreducible two-namespace split Option 2 left. Docs half still owned by `E-networking-wiki-stub-no-readback-contract`.
- `#5-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
