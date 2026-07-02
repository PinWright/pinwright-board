---
id: E-material-connect-nodes-target-input-arg-asymmetry
title: "connect_nodes target-pin arg is `inputName`, asymmetric with sourceNodeId/targetNodeId — invites a wrong `targetInput`/`targetPin` guess"
status: OPEN
severity: Low
category: ergonomic
tags: [material, material-authoring, connect-nodes, param-name, arg-asymmetry, docs]
encounters: 1
lastSeen: 2026-07-02T17:32:01+03:00
---

# `connect_nodes`'s target-input arg name (`inputName`) is asymmetric with its node-id pair, so callers guess `targetInput`

`material.authoring.connect_nodes` (and its `material.graph` sibling) names the
two node ends **symmetrically** — `sourceNodeId` / `targetNodeId` — and the
source pin is `sourcePin` / `sourceOutputIndex`. But the **target input pin** is
`inputName`, not the symmetric `targetInput` / `targetPin`. The visible
symmetry of the `sourceNodeId` ↔ `targetNodeId` pair (and `sourcePin` on the
source side) leads a caller who has not read the `connect_nodes` page to expect
a matching `targetInput` / `targetPin` on the target-pin side, and reach for the
wrong name.

This is a mild ergonomic asymmetry, **not** a bug: the RPC works, and its
`[UNKNOWN_PARAMS]` error is self-documenting — it lists every valid parameter,
so a caller recovers in a single retry. The cost is one wasted round-trip per
node wired by a caller who guessed `targetInput`.

## Why it's process friction (clean outcome, off the critical path)

Surfaced in a clean `material.authoring.clear_parameter_override` task (focus
`material.authoring.clear_parameter_override`, outcome clean) authoring
`/Game/Materials/M_MasterMetal`. The task's actual success check is a pure
material-instance override round-trip (create `MI_BrushedBrass`, clear the
Roughness override, confirm inheritance) that never touches `connect_nodes`; the
agent chose to wire the master's three params into the Main node to make it a
"real usable material." Wiring those three, it guessed `targetInput` for the
target pin and produced **3 back-to-back errored RPCs** before the error steered
it to `inputName`:

```
[UNKNOWN_PARAMS] Unknown parameter(s) for 'material.authoring.connect_nodes': [targetInput].
Valid parameters: [assetPath, materialPath, path, sourceNodeId, targetNodeId, inputName, sourcePin, sourceOutputIndex, x, y].
```

The agent had read the `create_material`, `add_scalar_parameter`,
`add_vector_parameter`, `create_material_instance`, and
`clear_parameter_override` method pages but **skipped `connect_nodes.md`**, and
inferred the target-pin arg from the symmetric-looking `sourceNodeId` /
`targetNodeId` pair. The good error message named `inputName`, so the single
retry made all three connects succeed (`{"message":"Connected to main material
node."}`). Agent SAY verbatim: *"The error message tells me the correct
parameter is `inputName`, not `targetInput`."*

Net cost: 3 errored `connect_nodes` RPCs on an otherwise clean 31-call trace,
fully recovered in one retry — peripheral to this iteration's seed goal, hence
Low.

## What it should do

Pick either (docs is the cheap immediate win):

- **Docs (minimum):** on the `connect_nodes` per-method overlay section (in
  `docs/wiki-src/material.authoring.md`, the `### material.authoring.connect_nodes`
  H3 that renders when an agent calls `connect_nodes` with no args) and in the
  `material.authoring` namespace prelude / "Common target pins" lead-in, state
  the arg vocabulary explicitly: source = `sourceNodeId` (+ `sourcePin` /
  `sourceOutputIndex`), target = `targetNodeId` (+ **`inputName`** for the input
  pin — *not* `targetInput` / `targetPin`), with the `"Main"`/empty sentinel for
  the main node. This is the same "publish the fixed arg/pin vocabulary on the
  method page so a single-page reader wires first-try" remedy applied to the
  sibling pin-name gap in `E-material-pin-input-type-undiscoverable` (#4-#7).
- **Alias (optional, more robust):** accept `targetInput` / `targetPin` as
  aliases for `inputName` via the `FParamSpec` alias machinery (the same
  approach `E-material-editor-param-name-drift` used for the `assetPath` /
  `materialPath` / `path` slot), so the intuitive guess just works and no
  round-trip is spent.

Filed E-/`docs` — the RPC is correct and its error is self-correcting; the
friction is purely the arg-name asymmetry inviting a wrong guess. Wiki page to
improve: `docs/wiki-src/material.authoring.md`.

severity rationale: impact=pure friction (naming asymmetry, self-documenting
error, one-retry recovery) × reach=connect_nodes is a common authoring method
but the mis-guess only bites callers who skip the connect_nodes page and the
error self-corrects -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Surfaced in a clean material.authoring.clear_parameter_override task (focus material.authoring.clear_parameter_override, outcome clean) authoring /Game/Materials/M_MasterMetal + MI_BrushedBrass. The seed method (clear_parameter_override) worked perfectly; this friction is on connect_nodes, which the agent used off the critical path to wire the master's three params (Roughness/Metallic/BaseColorTint) into the Main node. The agent had NOT read connect_nodes.md (read create_material/add_scalar_parameter/add_vector_parameter/create_material_instance/clear_parameter_override) and inferred the target-pin arg from the symmetric sourceNodeId/targetNodeId pair, guessing `targetInput` — 3 back-to-back errored RPCs: `[UNKNOWN_PARAMS] Unknown parameter(s) for 'material.authoring.connect_nodes': [targetInput]. Valid parameters: [assetPath, materialPath, path, sourceNodeId, targetNodeId, inputName, sourcePin, sourceOutputIndex, x, y].` The error named the correct `inputName`, so a single retry made all three connects succeed. SAY verbatim: "The error message tells me the correct parameter is `inputName`, not `targetInput`." Net cost 3 errored connects on an otherwise clean 31-call trace, fully recovered in one retry. Propose docs minimum (publish the source=`sourceNodeId`/`sourcePin` vs target=`targetNodeId`/`inputName` arg vocabulary on the connect_nodes overlay + namespace prelude in docs/wiki-src/material.authoring.md) or optionally alias `targetInput`/`targetPin` -> `inputName`. Low — recoverable in one retry via a self-documenting error, peripheral to this iteration's seed goal. Distinct from E-material-pin-input-type-undiscoverable (pin *type/name* discoverability) and E-material-editor-param-name-drift (asset-path slot drift): this is the target-*input* arg-name asymmetry on connect_nodes.
