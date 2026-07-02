---
id: E-set-cast-shadows-summary-stale-noop
title: "material.authoring.set_cast_shadows wiki/method summary still advertises the removed 'no-op that records the request' behavior, contradicting the handler's actual NOT_IMPLEMENTED error (and the namespace note that already says cast-shadows toggling is not exposed)"
status: OPEN
severity: Low
category: ergonomic
tags: [material-authoring, wiki, docs, stub, wiki-advertises-stub, stale-summary]
encounters: 1
lastSeen: 2026-07-02T04:23:44.5612491+03:00
---

# `set_cast_shadows` method summary advertises a removed silent-no-op

The `REGISTER_RPC_HANDLER` summary string for `material.authoring.set_cast_shadows`
still describes the **pre-fix** silent-no-op behavior. It renders (unedited) into
both the namespace `## Methods` index and the per-method wiki page:

`Saved/PinWright/wiki/material.authoring.set_cast_shadows.md:7`
(and `material.authoring.md:129`), verbatim:

> Stub: acknowledge a cast-shadows toggle on a material. **Currently a no-op that
> records the request** — full plumbing requires editing the per-shading-model
> property surface.

But the method no longer records anything — the handler body now fails loud:

`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:3131-3134`
```cpp
if (!Ctx.RequireAssetPath(MaterialHandlerUtils::MaterialAssetPathKeys(), AssetPath)) return true;

Ctx.SendError(TEXT("NOT_IMPLEMENTED"),
    TEXT("set_cast_shadows is a stub. Per-shading-model cast-shadow property routing is not yet implemented."));
```

## Why it's misleading

- "a no-op that **records the request**" reads as a soft success: the caller
  infers the call is accepted (the request is stored), and would plan to call it
  and move on. The method actually returns a hard `[NOT_IMPLEMENTED]` error, so
  the summary contradicts the runtime contract.
- The **same wiki page contradicts itself**: the namespace note
  (`material.authoring.md:52`) is already correct —
  > `set_cast_shadows` and `set_material_parameter` previously returned silent
  > success; both now return `NOT_IMPLEMENTED`. ... cast-shadows toggling is not
  > exposed.

  yet the auto-generated method summary two sections down still describes the
  removed no-op.

## Root cause

`B-material-stub-handlers-silent-success` (DONE) swapped `SendSuccess` →
`SendError("NOT_IMPLEMENTED", ...)` and updated the `docs/wiki-src` overlay note,
but left the C++ `REGISTER_RPC_HANDLER` summary string
(`MaterialAuthoringHandler.cpp:3124`) stale:
```cpp
REGISTER_RPC_HANDLER("material.authoring.set_cast_shadows", "material.authoring",
    "Stub: acknowledge a cast-shadows toggle on a material. Currently a no-op that records the request — full plumbing requires editing the per-shading-model property surface.",
```
`WikiHandler` renders that registration summary into both the `## Methods` index
and the per-method page, so the stale text is what discovery-first callers read.

## What it should do

Reword the registration summary at `MaterialAuthoringHandler.cpp:3124` to match
the honest runtime: state that the method returns `NOT_IMPLEMENTED` and that
material-level cast-shadows toggling is not exposed. If a fallback is worth
naming, cast-shadow is a per-primitive-component property (`bCastShadow` on the
placed mesh component / actor), not a material property — so the correct in-MCP
path is to toggle it on the placed prop, not via `material.authoring`. Fixing the
one summary string auto-corrects both the `## Methods` index and the per-method
page.

## Repro

1. `call("material.authoring.set_cast_shadows", {assetPath:"/Game/SciFi/Materials/M_Hologram", castShadows:false})`
   -> `Error [NOT_IMPLEMENTED] set_cast_shadows is a stub. Per-shading-model cast-shadow property routing is not yet implemented.`
2. Read `material.authoring.set_cast_shadows` wiki page -> summary advertises
   "Currently a no-op that records the request" (a soft-success framing), which
   the runtime error directly contradicts.

## Distinct from neighbours

- `B-material-stub-handlers-silent-success` (DONE) owns the **runtime**
  silent-success defect (now fail-loud) and the overlay note; it did **not**
  touch the registration summary, which is this ticket's sole residual.
- Same shape as the accepted overlay-honesty precedents
  `E-texture-create-wiki-advertises-stub`,
  `E-environment-snapshot-wiki-advertises-stub`, and
  `E-widget-style-workflow-wiki-advertises-stub` (wiki advertises a capability the
  runtime does not deliver), except here the stale text is the C++ registration
  summary itself, and it describes REMOVED behavior rather than merely
  over-promising.

severity rationale: impact=docs/discoverability × reach=rare -> Low.

## History
- `#1-initial-repro` `OPEN` reporter — Seed `material.authoring.set_cast_shadows` (realism task: author a hologram material that must not cast shadows). Replayed `set_cast_shadows({assetPath:/Game/SciFi/Materials/M_Hologram, castShadows:false})` -> `[NOT_IMPLEMENTED] set_cast_shadows is a stub. Per-shading-model cast-shadow property routing is not yet implemented.` — the intended fail-loud behavior from `B-material-stub-handlers-silent-success` (DONE), NOT a regression. Residual ergonomic defect: the C++ `REGISTER_RPC_HANDLER` summary at `MaterialAuthoringHandler.cpp:3124` still reads "Stub: acknowledge a cast-shadows toggle on a material. Currently a no-op that records the request …", which `WikiHandler` renders into `material.authoring.md:129` and `material.authoring.set_cast_shadows.md:7`. That "no-op that records the request" wording is a soft-success framing that contradicts the actual NOT_IMPLEMENTED error AND the same page's already-correct namespace note (`material.authoring.md:52`: "both now return NOT_IMPLEMENTED … cast-shadows toggling is not exposed"). Fix: reword the one registration summary string to match the honest runtime (and optionally point at the per-primitive-component `bCastShadow` path). Verified no dedup: only prior `set_cast_shadows` files are `B-material-stub-handlers-silent-success` (DONE, runtime fix) plus incidental precedent cites in `B-input-trigger-modifier-stub-silent-success` / `B-material-graph-edit-clobbered-by-open-editor` — none own the stale-summary angle.
