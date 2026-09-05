---
id: B-pcg-noise-invalid-type-leaves-node
title: "pcg.add_noise_filter adds a node before validating noiseType, so INVALID_NOISE_TYPE returns with an orphan node left in the graph"
status: IN-REVIEW
severity: High
category: bug
tags: [pcg, noise-filter, partial-mutation, validate-before-mutate, rollback, orphan-node, error-path]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# A rejected noise type still mutates the graph

`PCGAddNoiseFilter.cpp:110-118` calls `Graph->AddNodeOfType` before reading `noiseType`. The spatial
parser rejects an unknown token at `:132-140`; the attribute parser rejects it at `:170-178`.
Both branches call `Ctx.SendError(INVALID_NOISE_TYPE, ...)` and return immediately without removing
the node. The graph's ordinary dirty mark at `:210` is skipped too, so the error response discloses
neither the new node nor the dirty-memory side effect.

A typo such as `noiseType:"perlin"` therefore looks atomic to the caller but leaves an unconfigured
noise node that later inspection, save, or retry must clean up. The namespace-wide missing undo
transaction is tracked separately; adding a transaction alone does not roll back an error return.

## What should happen

Parse and validate `noiseType` and every other fallible option before `AddNodeOfType`. As a second
line of defence, wrap the mutation in a transaction and explicitly cancel/restore it on every error
after node creation. A regression should compare node count and package dirtiness before and after
both invalid spatial and invalid attribute requests.

**Workaround:** validate `noiseType` against the exact documented enum before calling, and inspect
the graph after any error to remove the orphan.

## Fix

Root cause: the handler created the node before parsing `noiseType`, so both invalid-mode returns
left the newly created node in `UPCGGraph::GetNodes()`. The handler now parses the optional string
and resolves the spatial or attribute enum before `AddNodeOfType`, returning the registered
`INVALID_NOISE_TYPE` code without mutating the graph.

Changed files:

- `Plugins/PinWright/Source/PinWrightPCG/Private/Handlers/PCG/PCGAddNoiseFilter.cpp`
- `Plugins/PinWright/Source/PinWrightPCG/Private/Handlers/PCG/PCGHandlerHelpers.h`
- `Plugins/PinWright/Source/PinWrightPCG/Private/Tests/PCG/PCGTypedHelpersTests.cpp`
- `Plugins/PinWright/Docs/wiki-src/pcg.md`

Tests added: `PinWright.pcg.add_noise_filter.InvalidSpatialNoiseTypeNoMutation` and
`PinWright.pcg.add_noise_filter.InvalidAttributeNoiseTypeNoMutation`; each asserts
`INVALID_NOISE_TYPE`, unchanged node count, and preserved package dirtiness through
`InvokeHandlerWithCapture`.

Deliberately unchanged: successful noise settings, node positioning, package dirty marking, and
the separate successful-mutation undo ticket.

## Related

`B-pcg-graph-mutators-no-transaction` owns undo support for successful mutations; this ticket owns
the non-atomic failure path.

## History
- `#1-source-pattern-scan` `OPEN` reporter — Both `INVALID_NOISE_TYPE` branches occur after `AddNodeOfType` and return without removal or rollback. Source-only; no RPC was run.
- `#2-pre-mutation-noise-validation` `IN-REVIEW` developer — Moved spatial and attribute `noiseType` validation before node creation; added atomicity tests and documented the typed error contract. Static-only; Unreal was not run.
