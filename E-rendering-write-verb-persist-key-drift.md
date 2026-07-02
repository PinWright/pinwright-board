---
id: E-rendering-write-verb-persist-key-drift
title: "rendering write verbs disagree on the persistence-path response key: set_dynamic_gi_method returns `configFile`, set_lumen_method/set_project_settings return `savedTo`"
status: OPEN
severity: Low
category: ergonomic
tags: [response-key-naming-drift, rendering, project-settings, set_dynamic_gi_method, set_lumen_method, set_project_settings, persist, docs]
encounters: 1
lastSeen: 2026-07-01T23:23:45.8237013+03:00
---

# `rendering.*` write verbs disagree on the persistence-path response key (`configFile` vs `savedTo`)

Three sibling write verbs in the **same `rendering` namespace and the same source
file** (`RenderingProjectSettingsHandler.cpp`) report "the DefaultEngine.ini I
persisted to" under **two different JSON keys**:

| Method | Persistence-path response key | Source |
|--------|-------------------------------|--------|
| `rendering.set_project_settings` | **`savedTo`** | `RenderingProjectSettingsHandler.cpp:132` |
| `rendering.set_lumen_method` | **`savedTo`** (delegates to the same `ApplyUpdates`) | `RenderingProjectSettingsHandler.cpp:132` |
| `rendering.set_dynamic_gi_method` | **`configFile`** | `RenderingProjectSettingsHandler.cpp:321` |

A caller who scripts the natural "write the setting, then confirm where it
persisted" flow across these three verbs — the exact flow of a Lumen-setup task
that touches all three — keys post-write verification on `savedTo` after
`set_lumen_method`/`set_project_settings`, then gets `null`/`undefined` after the
sibling `set_dynamic_gi_method`, whose path is under `configFile` instead. The
value is present, just spelled differently. This is **response-key naming drift**,
the same pattern already tracked per-namespace by `E-inspect-object-class-key-drift`
(IN-REVIEW, `class`/`className`) and `E-node-comment-field-name-drift` (OPEN,
`comment`/`nodeComment`); this occurrence is the rendering namespace's persist path.

## Second, subtler asymmetry: the two keys don't carry the same signal

Beyond the spelling, the two keys mean different things:

- `savedTo` (`ApplyUpdates`, `:122`-`:132`) is a **proof of persistence** — it is
  populated *only* when `bSave && Applied.Num() > 0`, otherwise it is the empty
  string. A caller can read `savedTo == ""` as "nothing was written to disk".
- `configFile` (`set_dynamic_gi_method`, `:321`) is emitted **unconditionally**
  inside the `if (bPersist)` block via `S->GetDefaultConfigFilename()` — it is the
  ini path regardless of whether any UPROPERTY actually applied/persisted. It is
  *not* a did-it-save signal.

So a caller who learned `savedTo`'s empty-means-not-saved contract from
`set_project_settings` and then keys the same check on `set_dynamic_gi_method`'s
`configFile` gets a **false "it saved"** on any persist that applied zero
properties (the path is always populated). Note `set_dynamic_gi_method` internally
already computes the correct signal — it calls `ApplyUpdates` (whose `WriteResp`
carries `savedTo`) at `:318`, copies `applied`/`rejected` from it at `:319`-`:320`,
then **discards** `WriteResp`'s `savedTo` and re-emits the raw filename under
`configFile` at `:321`.

(`rendering.get_project_settings`'s `configFile` at `:48` is a *read* verb — there
`configFile` = "which ini these settings live in" is appropriate and not part of
this drift; the drift is strictly among the three *write* verbs.)

## What it should do

Pick one canonical persistence-path key across the three write verbs, matching the
`savedTo` semantics (empty = not persisted). Cheapest non-breaking fix: in
`set_dynamic_gi_method`, forward `WriteResp`'s `savedTo` (already computed at
`:318`) as `savedTo` instead of re-labelling the raw filename as `configFile` at
`:321` — optionally keep `configFile` as a back-compat alias. That both aligns the
key spelling with its two siblings and makes it carry the real did-it-save signal
(empty when `save:false` or nothing applied) rather than an always-populated path.
Alternatively (docs-only), document on the `docs/wiki-src/rendering.md` overlay
that `set_dynamic_gi_method` reports its persist path under `configFile` (always
populated) while the other two use `savedTo` (empty unless a write persisted).

## Evidence

Replayed live at HEAD via `mcp__pinwright__call` (verbatim responses):

- `rendering.set_dynamic_gi_method {method:"LumenGI", persist:true}` →
  `{"method":"LumenGI","applied":["DynamicGlobalIllumination","Reflections"],"rejected":[],"configFile":"X:/src/unreal/EAContentExamples57-fuzz1/Config/DefaultEngine.ini"}`
- `rendering.set_lumen_method {hardwareRT:true, reflectionMethod:"Lumen", softwareRTMode:"Detail", save:true}` →
  `{"applied":["bUseHardwareRayTracingForLumen","Reflections","LumenSoftwareTracingMode"],"rejected":[],"savedTo":"X:/src/unreal/EAContentExamples57-fuzz1/Config/DefaultEngine.ini"}`
- `rendering.set_project_settings` wiki (`rendering.set_project_settings.md`) documents its
  response verbatim as `{ applied: [...], rejected: [...], savedTo: "<configFile or empty>" }`.

Surfaced from a Lumen cinematic-setup task (`rendering` namespace, focus
`rendering.get_project_settings`, outcome clean — every call `ok`/non-error). The
attempt's own friction note flagged it verbatim: *"set_dynamic_gi_method returns
its persistence indicator as `configFile` while set_lumen_method/set_project_settings
return it as `savedTo` (inconsistent key naming across the three write verbs)."*
Recovered without error (the agent read both response shapes by hand); the cost is
the silent-miss / false-"saved" risk for a parser keyed on one spelling+contract
across the three sibling write verbs.

severity rationale: impact=naming-drift + weak false-signal (Low) × reach=rare (rendering config writes are not an every-session path) -> Low.

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a Lumen cinematic-setup task (focus `rendering.get_project_settings`, namespace `rendering`, outcome clean). Replay-confirmed live at HEAD: `rendering.set_dynamic_gi_method` reports its persisted ini path under `configFile` while its two sibling write verbs `rendering.set_lumen_method` and `rendering.set_project_settings` report it under `savedTo` — response-key naming drift within one namespace/source file (`RenderingProjectSettingsHandler.cpp`: `savedTo` at `:132` in the shared `ApplyUpdates`, `configFile` at `:321` in `set_dynamic_gi_method`). Also a semantic asymmetry: `savedTo` is populated only when `bSave && Applied.Num() > 0` (empty = not persisted), whereas `set_dynamic_gi_method`'s `configFile` is emitted unconditionally via `GetDefaultConfigFilename()` (`:321`) regardless of whether a write persisted — even though it already computed the correct `savedTo` in `WriteResp` at `:318` and discarded it. Distinct from `E-rendering-readback-enum-token-asymmetry` (OPEN, read-back enum token forms), `E-rendering-unknown-property-no-hint` (OPEN, unknown-property name hint), and `B-rendering-lumen-software-mode-token-alias` (input-token rejection). Same symptom-family as `E-inspect-object-class-key-drift` (IN-REVIEW) and `E-node-comment-field-name-drift` (OPEN) but a different namespace/concept requiring its own rendering-side fix, so filed as its own file per board precedent (those two are themselves kept separate). Proposed: forward `ApplyUpdates`'s `savedTo` from `set_dynamic_gi_method` (keep `configFile` as a back-compat alias if desired), or document the divergence on the `rendering.md` overlay.
