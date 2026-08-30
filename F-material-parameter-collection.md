---
id: F-material-parameter-collection
title: "No RPCs to author UMaterialParameterCollection assets or wire collection parameter nodes"
status: DONE
severity: Medium
category: feature
tags: [material, material-authoring, material-parameter-collection, mpc, asset-creation, coverage-gap]
---

# No RPCs to author UMaterialParameterCollection assets or wire collection parameter nodes

`material.authoring` covers `UMaterial` parent authoring (graph + parameter
expressions) and `UMaterialInstanceConstant` overrides, but there is **zero
surface for `UMaterialParameterCollection` (MPC)**. Grepping
`docs/rpc-method-reference.generated.md` and
`Source/PinWright/Private/Handlers/Material/` for
`MaterialParameterCollection` / `parameter_collection` /
`CollectionParameter` returns no matches.

MPCs are the standard UE pattern for shared, globally-tweakable scalars and
linear-color vectors that many materials read from a single source:

- Time-of-day / sky parameters (sun direction, fog density, ambient color)
- Wind / global animation phase
- Team color, faction tint, planet biome
- Gameplay-driven shader state (player health overlay intensity, low-power
  desaturation strength) set from Blueprint via
  `SetScalarParameterValue(Collection, Name, Value)`

Without RPCs, an agent driving content authoring can build the parent
material and instance overrides but cannot wire either side of an MPC: it
can't create the collection asset, can't declare its parameters, and can't
drop a `UMaterialExpressionCollectionParameter` node into a material graph
that reads from one.

## Proposed RPCs

```
material.authoring.create_parameter_collection
    name, path, save? (default true)
    -> { assetPath }

material.authoring.add_collection_scalar_parameter
    assetPath, name, default? (number, default 0.0)
    -> { added: bool, parameterId: <guid> }

material.authoring.add_collection_vector_parameter
    assetPath, name, default? ({R,G,B,A}, default {0,0,0,1})
    -> { added: bool, parameterId: <guid> }

material.authoring.set_collection_parameter_default
    assetPath, parameterName, parameterType ("scalar"|"vector"), value
    -> { applied: bool }

material.authoring.remove_collection_parameter
    assetPath, parameterName, parameterType ("scalar"|"vector")
    -> { removed: bool }

material.authoring.get_parameter_collection_info
    assetPath
    -> {
        scalars: [{ name, default, parameterId }],
        vectors: [{ name, default: {R,G,B,A}, parameterId }]
    }

material.authoring.add_collection_parameter_node
    assetPath, collectionPath, parameterName, x, y, comment?
    -> { nodeId, classPath: "MaterialExpressionCollectionParameter" }
```

The graph-authoring node drop is the critical link — without
`add_collection_parameter_node`, even after the MPC exists, a material can't
sample it. `material.graph.add_node` accepts the class name, but the node
needs both `Collection` (UMaterialParameterCollection*) and `ParameterName`
(FName, bound to the right `ParameterId` GUID) properties set atomically, or
the node renders as "(Invalid Parameter)". A typed RPC handles the lookup
and binding.

## Implementation surface

- `UMaterialParameterCollection` exposes `ScalarParameters` and
  `VectorParameters` (TArray of `FCollectionScalarParameter` /
  `FCollectionVectorParameter`), each with `ParameterName`, `DefaultValue`,
  `Id` (FGuid, auto-generated, used by sampling nodes for stable binding).
- `UMaterialParameterCollectionFactoryNew` for `create_parameter_collection`
  via `AssetTools.CreateAsset`.
- After param add/remove/edit, call
  `Collection->Modify()` + `Collection->PostEditChangeProperty(...)` so
  dependent materials and live `UMaterialParameterCollectionInstance`s
  rebuild. Save via `UEditorAssetLibrary::SaveLoadedAsset` (matches the
  existing `material.authoring.create_*` save pattern).
- `add_collection_parameter_node`: load both assets, instantiate
  `UMaterialExpressionCollectionParameter`, set `Collection` +
  `ParameterName` + lookup `ParameterId` from the collection's array, add
  to `Material->GetExpressionCollection()` (UE 5.6 API), place at `(x,y)`.

## Why it matters

MPC is the canonical UE pattern for cross-material shared state and the
only sanctioned path for runtime Blueprint -> shader value flow at the
collection-granularity tier (vs per-instance overrides). Shipped projects
use them heavily for time-of-day, weather, and gameplay overlays. The
existing handler family covers `UMaterial` and `UMaterialInstanceConstant`
end-to-end; MPC is the third leg of that triangle and is completely absent.

## History
- `#1-no-mpc-rpcs` `OPEN` reporter — Verified zero MPC coverage: grep of `docs/rpc-method-reference.generated.md` and `Source/EditorAutomationRpcGateway/Private/Handlers/Material/` for `MaterialParameterCollection` / `parameter_collection` / `CollectionParameter` returns no matches. Existing material surface covers `UMaterial` (graph + parameter expressions via `material.authoring.add_scalar_parameter` etc.) and `UMaterialInstanceConstant` (overrides via `set_*_parameter_value`) but has nothing for the third asset class in the family, `UMaterialParameterCollection`, which is the standard UE pattern for cross-material shared scalars/colors (time-of-day, wind, team color, gameplay-driven shader state). Proposes `create_parameter_collection`, `add_collection_scalar_parameter`, `add_collection_vector_parameter`, `set_collection_parameter_default`, `remove_collection_parameter`, `get_parameter_collection_info` for the asset side, plus `add_collection_parameter_node` to drop a `UMaterialExpressionCollectionParameter` into a material graph bound to a specific (collection, parameterName, parameterId) — the typed node-drop is critical because the expression requires both `Collection` and `ParameterName` set atomically against the right `FGuid` Id or it renders as "(Invalid Parameter)".
- `#2-mpc-rpcs-and-node-drop` `IN-REVIEW` developer — Added MaterialParameterCollectionHandler.cpp with 7 RPCs (create_parameter_collection, add_collection_{scalar,vector}_parameter, set_collection_parameter_default, remove_collection_parameter, get_parameter_collection_info, add_collection_parameter_node). Added regression test TestMaterialParameterCollectionHandlers.cpp asserting MPC CRUD + atomic Collection+ParameterName+ParameterId binding on the typed node-drop.
- `#3-verify-mpc-end-to-end` `DONE` tester — Verified: `create_parameter_collection` produced `/Game/App/UI/Test/MPC_McpVerifyTemp_F_material_parameter_collection` (assetClass=MaterialParameterCollection); `add_collection_scalar_parameter`(WindStrength, 0.5) returned parameterId `2C2CD82541B17814F2BF60A7D3EA6C5F`; `add_collection_vector_parameter`(TeamColor, {1,0.25,0.1,1}) returned parameterId `D66427C44E0AD23B292B04B6F0AB17AE`; `get_parameter_collection_info` echoed both with matching ids and defaults. Critical node-drop test: `add_collection_parameter_node` against a temp `M_McpVerifyTemp_F_mpc` material wired the new node with all three atomic fields per `get_node_details` — `Collection`=MPC asset path, `ParameterName`="WindStrength", `ParameterId`=`2C2CD82541B17814F2BF60A7D3EA6C5F` (exact GUID match to the scalar's id), so the node will not render as "(Invalid Parameter)". Cleaned up both temp assets via `asset.delete`.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
