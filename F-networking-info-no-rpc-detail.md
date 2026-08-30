---
id: F-networking-info-no-rpc-detail
title: "networking.get_networking_info omits the replication fields no other reader surfaces — per-RPC withValidation (FUNC_NetValidate), the RepNotify function NAME, the replication CONDITION, and the actor owner — so those setters can't be read back"
status: IN-REVIEW
severity: Medium
category: feature
tags: [networking, get-networking-info, rpc-validation, repnotify-name, replication-condition, set-owner, owner, readback, round-trip]
---

# `networking.get_networking_info` should round-trip the replication state that no reader currently surfaces

The `networking` namespace has a full set of *setters* that mutate
replication state on a Blueprint and on world actors:

- `networking.configure_rpc_validation` — toggles `FUNC_NetValidate` on an
  RPC function's entry node.
- `networking.set_rpc_reliability` — toggles `FUNC_NetReliable`.
- `networking.create_rpc_function` — creates a `Server`/`Client`/`NetMulticast`
  RPC.
- `networking.set_replicated_using` — sets a property's `ReplicatedUsing`
  (RepNotify) handler NAME.
- `networking.set_replication_condition` — sets a property's
  `ELifetimeCondition` (e.g. `COND_OwnerOnly`).
- `networking.set_owner` / `actor.attach` / `actor.detach` — write
  `AActor::Owner` on a world actor.

**Scope correction (reworded).** A cross-namespace reader,
`system.inspect.inspect_class` (ticket `F-inspect-class-members`, DONE),
*already* round-trips part of this: given a `/Game/…_C` Blueprint path it
walks the authoritative class and decodes per-function flags
(`Server`/`Client`/`NetMulticast` + `Reliable`/`Unreliable`) and per-property
flags (`Replicated`, `RepNotify`-presence) — see
`PropertyInspection.cpp` `DecodeFunctionFlags`/`DecodePropertyFlags`. So
`rpcType`, `reliable`, `replicated`, and RepNotify-*presence* are **not** the
gap. The genuine, still-unreadable state — written by the setters but decoded
by **no** reader — is:

1. **per-RPC `withValidation`** (`FUNC_NetValidate`) — `DecodeFunctionFlags`
   does not decode it; this is the server `_Validate` gate and the field that
   "matters most" for hardening authoritative RPCs.
2. **the RepNotify function NAME** (e.g. `OnRep_Health`) —
   `DecodePropertyFlags` emits only a boolean `RepNotify` tag; the name lives
   in `FBPVariableDescription::RepNotifyFunc` and is surfaced nowhere.
3. **the replication CONDITION** (`COND_OwnerOnly`, …) — lives in
   `FBPVariableDescription::ReplicationCondition`; no decoder emits the
   `ELifetimeCondition`.
4. **the actor `owner` / `ownerName`** — `set_owner` reports the new owner
   only in its transient `message` string; the actor branch emits no owner
   field, and a grep over `actor.*` shows `AActor::Owner` is only ever
   written, never read.

A reasonable, explicitly-requested task — "wire up these RPCs and replicated
vars, then confirm the *validation* gate, the *RepNotify handler name*, and
the *replication condition* landed the way I asked, and confirm the owner I
set" — is therefore still impossible through this MCP surface even with
`inspect_class` in hand.

## What the reader returns today vs. what's needed

**Repro (live, against a freshly-built character BP):**

```
mcp__editor-automation__call method="networking.create_rpc_function"
  args={"blueprintPath":"/Game/Pickups/BP_NetPlayer","functionName":"ServerApplyDamage","rpcType":"Server"}   # ok
mcp__editor-automation__call method="networking.set_rpc_reliability"
  args={"blueprintPath":"/Game/Pickups/BP_NetPlayer","functionName":"ServerApplyDamage","reliable":true}      # -> {"success":true,"reliable":true,...}
mcp__editor-automation__call method="networking.configure_rpc_validation"
  args={"blueprintPath":"/Game/Pickups/BP_NetPlayer","functionName":"ServerApplyDamage","withValidation":true} # -> {"success":true,"withValidation":true,...}
mcp__editor-automation__call method="networking.set_replicated_using"
  args={"blueprintPath":"/Game/Pickups/BP_NetPlayer","propertyName":"Health","repNotifyFunction":"OnRep_Health"} # ok
```

Then the documented readback:

```
mcp__editor-automation__call method="networking.get_networking_info"
  args={"blueprintPath":"/Game/Pickups/BP_NetPlayer"}
# {
#   "success": true,
#   "networkingInfo": {
#     "bReplicates": true, "bAlwaysRelevant": false, "bOnlyRelevantToOwner": false,
#     "netUpdateFrequency": 100, "minNetUpdateFrequency": 2,
#     "netCullDistanceSquared": 225000000, "netPriority": 3, "netDormancy": "DORM_Awake"
#   }
# }
```

No `rpcFunctions` array. No `replicatedProperties`. The
`withValidation`/RepNotify-name/condition that `configure_rpc_validation`,
`set_replicated_using`, and `set_replication_condition` just wrote are
invisible.

**`inspect_class` covers `rpcType`/`reliable`/`replicated`/RepNotify-presence
but NOT the four fields above.** `inspect_class {className:"…_C"}` decodes
the net role + Reliable/Unreliable per function and the `Replicated`/`RepNotify`
tags per property, but never `FUNC_NetValidate`, never the RepNotify function
*name*, never the `ELifetimeCondition`, and (being class-level) never an
instance's owner.

## Proposed extension (in-place, additive)

Extend `networking.get_networking_info`'s `networkingInfo` payload for the
`blueprintPath` branch with two additive arrays (old fields preserved so
existing callers don't break). The arrays emit a self-contained per-RPC /
per-property record so this one reader closes the round-trip without forcing
agents to cross-reference `inspect_class`; the **load-bearing** fields it
uniquely adds are `withValidation`, `replicatedUsing` (the name), and
`replicationCondition`:

```jsonc
"networkingInfo": {
  // ... existing actor-level fields ...
  "rpcFunctions": [
    { "name": "ServerApplyDamage",     "rpcType": "Server",       "reliable": true,  "withValidation": true  },
    { "name": "MulticastPlayHitReact", "rpcType": "NetMulticast", "reliable": false, "withValidation": false }
  ],
  "replicatedProperties": [
    { "name": "Health", "replicated": true, "replicatedUsing": "OnRep_Health", "replicationCondition": "COND_OwnerOnly" }
  ]
}
```

For the **actor branch** (the `actorName` form, lines 1300–1325), the
analogous additive field is the actor's **owner** — the one piece of state
the actor-instance setters `networking.set_owner`, `actor.attach`, and
`actor.detach` write that no reader surfaces:

```jsonc
"networkingInfo": {
  // ... existing actor-level fields (role/remoteRole/hasAuthority/etc.) ...
  "owner": "/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.BP_DemoDisplay_C_0",  // null when no owner
  "ownerName": "BP_DemoDisplay_C_0"   // null when no owner
}
```

**Implementation hint (actor branch):** emit `Actor->GetOwner()` →
`ownerName` (the owner's name) and `owner` (its object path), `null`
when `GetOwner()` is null — read the same `AActor::Owner` that
`set_owner`/`attach`/`detach` write via `SetOwner()` (`NetworkingHandler.cpp`
`set_owner` :609, `ActorPropertyHandler.cpp` :465/:525).

**Implementation hint (blueprintPath branch):** the RPC flags live on the
function graph's `UK2Node_FunctionEntry` extra-flags
(`FUNC_Net`, `FUNC_NetServer`/`FUNC_NetClient`/`FUNC_NetMulticast`,
`FUNC_NetReliable`, `FUNC_NetValidate`) — the exact source
`configure_rpc_validation`/`set_rpc_reliability` mutate. Walk
`Blueprint->FunctionGraphs`, find each `UK2Node_FunctionEntry`, read
`GetExtraFlags()`, and emit only graphs carrying `FUNC_Net`. For the
replicated-property record read `Blueprint->NewVariables` (the same
`FBPVariableDescription` the setters write): `RepNotifyFunc` →
`replicatedUsing` (the name) the way `set_replicated_using` writes it, and
`ReplicationCondition` → `replicationCondition` the way
`set_replication_condition` writes it (mirror the `GetReplicationCondition`
string↔enum mapping).

**Fix:** `get_networking_info` blueprintPath branch now emits
`rpcFunctions[]` (name, rpcType, reliable, **withValidation**) decoded from
each RPC graph's `UK2Node_FunctionEntry` extra-flags, and
`replicatedProperties[]` (name, replicated, **replicatedUsing** name,
**replicationCondition**) from `Blueprint->NewVariables`. The actor branch
now emits **owner**/**ownerName** from `AActor::GetOwner()` (null when no
owner). All old actor-CDO fields are preserved (additive).

**Acceptance check:** after the repro above, `get_networking_info` returns a
`rpcFunctions` entry for `ServerApplyDamage` with `reliable:true` and
`withValidation:true`, an entry for `MulticastPlayHitReact` with
`rpcType:"NetMulticast"` and `withValidation:false`, and a
`replicatedProperties` entry for `Health` with `replicatedUsing:"OnRep_Health"`
and `replicationCondition:"COND_OwnerOnly"`; and `get_networking_info{actorName}`
after `set_owner` returns `ownerName` equal to the owner actor's name.

**Workaround until implemented:** open the asset in the editor by hand, or
re-run the setter (which only re-asserts intent and recompiles — it doesn't
read state). Neither closes the round-trip loop an agent needs.

## History
- `#8-additional-replicate-movement-readback` `OPEN` reporter — Additional evidence: the same write-only gap covers another setter not yet in scope — `networking.configure_replicated_movement`, which writes the actor CDO's replicate-movement flag via `CDO->SetReplicatingMovement(bReplicateMovement)` (NetworkingHandler.cpp `configure_replicated_movement` :1397). Replayed live against a freshly-built Actor BP `/Game/Multiplayer/BP_HealthPickup` (the full multiplayer-pickup build: create → add_variable ×2 (HealthAmount int, bIsConsumed bool, both replicated) → set_property_replicated ×2 → set_net_role(ROLE_Authority) → configure_replicated_movement(replicateMovement:true) → set_net_dormancy(DORM_Initial) → create_rpc_function(ServerConsumePickup, Server, reliable) → compile → save). The setter worked correctly — NOT a bug: `property.get{objectPath:"/Game/Multiplayer/BP_HealthPickup", propertyName:"bReplicateMovement"}` → `{"propertyName":"bReplicateMovement","value":true,...}`. The documented readback `get_networking_info{blueprintPath:"/Game/Multiplayer/BP_HealthPickup"}` (run against THIS ticket's IN-REVIEW build — `rpcFunctions[]`/`replicatedProperties[]` present) returned: `{"bReplicates":true,"bAlwaysRelevant":false,"bOnlyRelevantToOwner":false,"netUpdateFrequency":100,"minNetUpdateFrequency":2,"netCullDistanceSquared":225000000,"netPriority":1,"netDormancy":"DORM_Initial","rpcFunctions":[{"name":"ServerConsumePickup","rpcType":"Server","reliable":true,"withValidation":false}],"replicatedProperties":[{"name":"HealthAmount","replicated":true,"replicatedUsing":null,"replicationCondition":"COND_None"},{"name":"bIsConsumed","replicated":true,"replicatedUsing":null,"replicationCondition":"COND_None"}]}`. So netDormancy (DORM_Initial), bReplicates (ROLE_Authority side-effect), both replicated vars, and the reliable Server RPC all round-trip correctly now — but there is **no `replicateMovement`/`bReplicateMovement` field anywhere in the payload**. The blueprintPath branch (NetworkingHandler.cpp :1434-1456) emits exactly the 8 CDO fields + `rpcFunctions`/`replicatedProperties` and never reads `CDO->IsReplicatingMovement()`; the actor branch (:1466-1482) likewise omits it despite reading many other CDO/actor fields. No sibling reader fills the gap: confirming the "replicate movement" success-check field (which the task explicitly names) required a separate cross-namespace `property.get{objectPath, propertyName:"bReplicateMovement"}`. The fix's existing actor-CDO field block in the blueprintPath branch is the obvious place to add it: emit a `bReplicateMovement` bool from `CDO->IsReplicatingMovement()` (the same flag `configure_replicated_movement` writes via `SetReplicatingMovement` at :1397), and the symmetric `Actor->IsReplicatingMovement()` on the actor branch. Stays a GAP about `get_networking_info`, not a bug in `configure_replicated_movement` (the setter and its `bReplicateMovement=true` effect are correct). Acceptance addendum: after `configure_replicated_movement{replicateMovement:true}`, `get_networking_info{blueprintPath}` carries `bReplicateMovement:true` (and `false` after `replicateMovement:false`). (Seed method `networking.set_net_role` itself works correctly — its ROLE_Authority bReplicates side-effect round-trips fine; culprit is `get_networking_info`.)
- `#7-additional-push-model-readback` `OPEN` reporter — Additional evidence: the same write-only gap covers a setter NOT in the current implementation's scope — `networking.configure_push_model`, which writes a per-variable `PushModel="true"` metadata key (the same `FBPVariableDescription` this ticket's `BuildReplicatedPropertiesArray` walks). Replayed live against a freshly-built Actor BP `/Game/Multiplayer/BP_ReplicatedPickup` (two replicated public vars `ChargesRemaining` int=3, `bIsActive` bool=true): `configure_push_model{usePushModel:true}` → `{"success":true,"usePushModel":true,...}`, `configure_net_update_frequency{30/5}` → success, `set_replication_condition{ChargesRemaining,COND_SkipOwner}` → success. `blueprint.get` confirms BOTH vars carry `metadata.PushModel:"true"` (so the setter worked correctly — NOT a bug). The documented readback `get_networking_info{blueprintPath:"/Game/Multiplayer/BP_ReplicatedPickup"}` (run against THIS ticket's IN-REVIEW build — `replicatedProperties[]` is present) returned: `{"bReplicates":false,...,"netUpdateFrequency":30,"minNetUpdateFrequency":5,...,"rpcFunctions":[],"replicatedProperties":[{"name":"ChargesRemaining","replicated":true,"replicatedUsing":null,"replicationCondition":"COND_SkipOwner"},{"name":"bIsActive","replicated":true,"replicatedUsing":null,"replicationCondition":"COND_None"}]}`. So `replicationCondition` (COND_SkipOwner) and the CDO frequency (30/5) now round-trip correctly — but each `replicatedProperties[]` entry emits ONLY `name`/`replicated`/`replicatedUsing`/`replicationCondition` (NetworkingHandler.cpp `BuildReplicatedPropertiesArray` :213-226); **no push-model field**, so "confirm push model is enabled" — the readback the task explicitly names — is structurally unsatisfiable through `get_networking_info`. The fix's `replicatedProperties[]` is the obvious place to add it: emit a `pushModel` bool per entry, read from `VarDesc.GetMetaData("PushModel")=="true"` (mirroring how `configure_push_model` at :1033 writes it). Stays a GAP about `get_networking_info`, not a bug in `configure_push_model` (seed; the setter and its `usePushModel:true` return are correct). Acceptance addendum: after `configure_push_model{usePushModel:true}`, each `replicatedProperties[]` entry for a push-model var carries `pushModel:true`.
- `#6-reword-and-implement-residual-readback` `IN-REVIEW` developer — REWORD + implement. Adversarial re-scope: `system.inspect.inspect_class` (DONE) already round-trips `rpcType`/`reliable`/`replicated`/RepNotify-*presence* (`PropertyInspection.cpp` `DecodeFunctionFlags` :456-483 decodes Server/Client/NetMulticast + Reliable/Unreliable; `DecodePropertyFlags` :439-440 emits Replicated/RepNotify tags), so the ticket was rewritten to own only the four fields NO reader surfaces: per-RPC **withValidation** (`FUNC_NetValidate`, not in `DecodeFunctionFlags`), the RepNotify function **NAME** (`FBPVariableDescription::RepNotifyFunc`; decoders emit only a boolean tag), the replication **CONDITION** (`FBPVariableDescription::ReplicationCondition`; no decoder), and the actor-instance **owner**/**ownerName** (class-level inspect can't reach an instance). Implemented the in-place additive extension in `NetworkingHandler.cpp`: new helpers `ReplicationConditionToString`, `NetFunctionTypeToString`, `BuildRpcFunctionsArray` (walks `Blueprint->FunctionGraphs` → `UK2Node_FunctionEntry::GetExtraFlags()`, emits `{name,rpcType,reliable,withValidation}` for each `FUNC_Net` graph) and `BuildReplicatedPropertiesArray` (walks `Blueprint->NewVariables`, emits `{name,replicated,replicatedUsing,replicationCondition}` for each `CPF_Net` var). The `get_networking_info` blueprintPath branch now sets `rpcFunctions`/`replicatedProperties`; the actorName branch now sets `owner`/`ownerName` from `Actor->GetOwner()` (JSON null when none). All 8 actor-CDO fields preserved. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Networking/NetworkingHandler.cpp`. Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Networking/TestNetworkingHandlers.cpp` → `EditorAutomationRpcGateway.networking.get_networking_info.RpcReplicationRoundTrip` — creates a real package-backed BP, drives `create_rpc_function`+`set_rpc_reliability`+`configure_rpc_validation` (ServerApplyDamage) and a replicated var + `set_replicated_using(OnRep_Health)` + `set_replication_condition(COND_OwnerOnly)`, then calls the production `get_networking_info` handler and asserts the round-trip of `rpcFunctions[].withValidation` and `replicatedProperties[].replicatedUsing`/`.replicationCondition` (fails if the reader extension is reverted). Released the fuzz4 claim. OPEN → IN-REVIEW.
- `#5-additional-coin-pickup-unreliable-multicast` `OPEN` reporter — Additional evidence on a fresh dedicated-server **coin-pickup** build against `/Game/Multiplayer/BP_CoinPickup` (Actor BP). Ran the full end-to-end sequence — `set_net_role(ROLE_Authority)` → `add_variable(CurrentValue, int)` → `set_property_replicated(true)` → `set_replicated_using(OnRep_CurrentValue)` → `set_replication_condition(COND_OwnerOnly)` → `create_rpc_function(ServerCollect, Server) + set_rpc_reliability(true)` → `create_rpc_function(MulticastPlayPickupFX, NetMulticast, reliable:false)` → `configure_net_update_frequency(30/5)` — all `success:true` with correct inline returns. The documented readback `get_networking_info{blueprintPath:"/Game/Multiplayer/BP_CoinPickup"}` returned ONLY the 8 actor-CDO fields: `{"bReplicates":true,"bAlwaysRelevant":false,"bOnlyRelevantToOwner":false,"netUpdateFrequency":30,"minNetUpdateFrequency":5,"netCullDistanceSquared":225000000,"netPriority":1,"netDormancy":"DORM_Awake"}` — the only mutation that round-trips is the actor-CDO frequency (30/5 shows up). No `role` (the `set_net_role` ROLE_Authority is invisible on the blueprintPath branch), no `replicatedProperties` (OnRep_CurrentValue + COND_OwnerOnly invisible), no `rpcFunctions`. `blueprint.inspect` reconfirmed insufficient: `CurrentValue` as `{"replicated":true}` with NO `replicatedUsing`/`replicationCondition`, and `ServerCollect`/`MulticastPlayPickupFX` listed by name only (`{"name":"ServerCollect","public":false,"pure":false,...}`) with NO rpcType/reliable. This run adds the **unreliable NetMulticast** case (`MulticastPlayPickupFX` reliable:false) as a load-bearing example for `rpcFunctions[].reliable` (distinguishing the unreliable cosmetic-FX multicast from the reliable ServerCollect is exactly what an agent must verify and currently cannot). Stays a GAP about `get_networking_info`, not a bug in any setter — every setter worked correctly.
- `#4-additional-actor-owner-roundtrip` `OPEN` reporter — Additional evidence extending the gap to the **actor branch** (`actorName` form) and the actor-instance setter `networking.set_owner`. Replayed live against the `ExampleProjectWelcome` level: `get_networking_info{actorName:"StaticMeshActor_1"}` → `{"bReplicates":false,"bAlwaysRelevant":false,"bOnlyRelevantToOwner":false,"netUpdateFrequency":100,"minNetUpdateFrequency":2,"netCullDistanceSquared":225000000,"netPriority":1,"netDormancy":"DORM_Awake","role":"ROLE_Authority","remoteRole":"ROLE_None","hasAuthority":true}`. Then `set_owner{actorName:"StaticMeshActor_1",ownerActorName:"BP_DemoDisplay_C_0"}` → `{"success":true,"message":"Set owner of StaticMeshActor_1 to BP_DemoDisplay_C_0",...}` (setter works correctly — NOT a bug). The post-set readback `get_networking_info{actorName:"StaticMeshActor_1"}` returns a **byte-identical** payload — no `owner`/`ownerName` field. `set_owner{ownerActorName:""}` (clear) → `{"success":true,"message":"Cleared owner of StaticMeshActor_1",...}`, and the post-clear readback is again byte-identical. So the explicitly-requested round-trip "set owner → read back to confirm → clear → read back to confirm cleared" is impossible: the owner change is visible ONLY in `set_owner`'s own transient `message` string, never as readable state. Source confirms the actor branch (`NetworkingHandler.cpp:1308-1324`) emits no owner field; `set_owner` (`:609`) puts the owner only in `message`, not a structured field. No sibling reader fills the gap: ripgrep over the entire `actor.*` namespace shows owner is only ever WRITTEN (`SetOwner` in `set_owner`/`actor.attach:465`/`actor.detach:525`) and NEVER read — `actor.describe` does not emit owner. Sharpens the proposal: add an `owner`/`ownerName` pair to the **actor branch** (above) so all three owner-writing setters (`set_owner`, `attach`, `detach`) round-trip; this is the actor-instance analogue of the blueprintPath `rpcFunctions[]`/`replicatedProperties[]` ask. (Seed `networking.check_is_locally_controlled` returned a clean definite `false` and works correctly — culprit is `get_networking_info`, stays a GAP not a bug.)
- `#3-additional-host-asset-roundtrip` `OPEN` reporter — Additional evidence on a **pre-existing host asset** (not a freshly-created BP): replayed the full health-replication build against `/Game/Global/Blueprints/PlayerCharacter` (a Character BP that already had `bReplicates:true`). After `add_variable(CurrentHealth/MaxHealth, replicated)`, `set_replicated_using(CurrentHealth, OnRep_CurrentHealth)`, `set_replication_condition(CurrentHealth, COND_SimulatedOrPhysics)`, `create_rpc_function(ServerApplyDamage, Server)` + `set_rpc_reliability(true)` + `configure_rpc_validation(true)`, and `configure_net_update_frequency(120)` — all `success:true` — the documented readback `get_networking_info{blueprintPath:"/Game/Global/Blueprints/PlayerCharacter"}` returns exactly the 8 actor-CDO fields: `{"bReplicates":true,"bAlwaysRelevant":false,"bOnlyRelevantToOwner":false,"netUpdateFrequency":120,"minNetUpdateFrequency":2,"netCullDistanceSquared":225000000,"netPriority":3,"netDormancy":"DORM_Awake"}`. Sharpens the gap: the **only** mutation that round-trips through the reader is the actor-CDO one (`netUpdateFrequency:120` shows up); every per-property/per-RPC mutation is invisible. Sibling readers reconfirmed insufficient on this asset: `blueprint.inspect` returns `CurrentHealth`/`MaxHealth` as `{"replicated":true}` with NO `replicatedUsing`/`replicationCondition`, and `ServerApplyDamage` as `{"name":"ServerApplyDamage","public":false,"pure":false,...}` with NO rpcType/reliable/withValidation; `asset.dump` properties.json only exposes raw CPF flag strings (`"CurrentHealth":{"flags":["BlueprintReadWrite","Edit","Net","RepNotify"]}`) — flag PRESENCE only, not WHICH RepNotify function nor the replication condition. So `OnRep_CurrentHealth`, `COND_SimulatedOrPhysics`, and the RPC's reliable/withValidation flags remain unverifiable through any surface, confirming the round-trip is impossible exactly as proposed (`replicatedProperties[].replicatedUsing`/`.replicationCondition` + `rpcFunctions[].reliable`/`.withValidation` are all load-bearing). (Seed `get_networking_info` works correctly for what it reports — stays a GAP, not a bug.)
- `#2-additional-repcondition` `OPEN` reporter — Additional evidence: confirmed the same write-only gap also covers `networking.set_replication_condition`. Replayed a full multiplayer-pickup build live against `/Game/Multiplayer/BP_HealthPickup` (Actor BP, two replicated vars `HealthAmount`/`bIsActive`): `set_property_replicated(HealthAmount,true)` → `{"success":true,...}`, `set_replicated_using(bIsActive, OnRep_IsActive)` → `{"success":true,"message":"ReplicatedUsing set to OnRep_IsActive for property bIsActive",...}`, `set_replication_condition(HealthAmount, COND_OwnerOnly)` → `{"success":true,"message":"Replication condition set to COND_OwnerOnly",...}`, `create_rpc_function(ServerPickup, Server)` + `set_rpc_reliability(ServerPickup, true)` all succeed. The documented readback `get_networking_info{blueprintPath:"/Game/Multiplayer/BP_HealthPickup"}` returns exactly the 8 actor-CDO fields and nothing else: `{"bReplicates":false,"bAlwaysRelevant":false,"bOnlyRelevantToOwner":false,"netUpdateFrequency":100,"minNetUpdateFrequency":2,"netCullDistanceSquared":225000000,"netPriority":1,"netDormancy":"DORM_Awake"}` — no `replicatedProperties`, no `replicationCondition`, no `replicatedUsing`, no `rpcFunctions`. The COND_OwnerOnly that `set_replication_condition` just wrote is invisible, so the proposed `replicatedProperties[].replicationCondition` field (line 103) is load-bearing, not just nice-to-have. Sibling readers reconfirmed insufficient: both `blueprint.get` and `blueprint.inspect` return `HealthAmount`/`bIsActive` as `replicated:true` and list `ServerPickup` by name, but emit no `replicatedUsing`/`replicationCondition` on the properties and no rpcType/reliable/validation on the function. (Seed method `set_property_replicated` itself works correctly — this stays a GAP about `get_networking_info`, not a bug in the setter.)
- `#1-initial-repro` `OPEN` reporter — Replayed live against `/Game/Pickups/BP_NetPlayer`: `networking.configure_rpc_validation(ServerApplyDamage, withValidation:true)` returns `{"success":true,"withValidation":true,...}` (the seed method itself works correctly — not a bug). But the documented readback `networking.get_networking_info{blueprintPath}` returns only actor-level CDO fields (`bReplicates`, `netUpdateFrequency`, `minNetUpdateFrequency`, `netCullDistanceSquared`, `netPriority`, `netDormancy`, relevance flags) — no `rpcFunctions`, no per-RPC reliable/validation/type, no per-property `replicatedUsing`. Source confirms: handler at `NetworkingHandler.cpp:1222` only reads CDO fields. `blueprint.inspect` lists RPC functions by name and `Health` as `replicated:true` but reports no net flags / no RepNotify. Verified no sibling reader exists across all 27 `networking.*` methods (namespace page + ripgrep). Board ripgrep + qmd semantic search found no existing ticket for this gap (`F-inspect-class-members` is adjacent but covers native UClass reflection via `system.inspect.inspect_class`, a different surface). Classified GAP: the setters (`configure_rpc_validation`, `set_rpc_reliability`, `set_replicated_using`) are write-only with no round-trip confirmation. Proposed in-place additive extension to `get_networking_info`: `rpcFunctions[]` (decoded from `UK2Node_FunctionEntry` extra-flags) + `replicatedProperties[]` (with `replicatedUsing`).
- `#9-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
