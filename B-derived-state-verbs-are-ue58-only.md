---
id: B-derived-state-verbs-are-ue58-only
title: "The derived-state changeset calls APIs that do not exist before UE 5.8 / 5.6, against a plugin that still advertises UE 5.3-5.8"
status: OPEN
severity: Medium
category: bug
tags: [build, engine-compat, version-guards, water, landscape, material-authoring, packaging]
---

# Two unguarded calls narrow the plugin's supported engine range

The derived-state-honesty changeset (`099b83b3`..`0fe35187`) was authored under an explicit
"**target UE 5.8 only, no version guards**" instruction and builds clean on 5.8. It contains **zero**
`UE_VERSION_*` macros, against 221 elsewhere in the plugin. Two calls in it do not exist on older
engines:

| call | site | first available |
|---|---|---|
| `ALandscapeProxy::RetrieveAllLandscapeMaterials` | `Handlers/Material/MaterialLandscapeConsumers.h` | **UE 5.8** (`LandscapeProxy.h:1319`) |
| `UWaterBodyRiverComponent::{Get,Set}River{Width,Depth}AtSplineInputKey` | `Handlers/Water/WaterHandler.cpp` | **UE 5.6** (`WaterBodyRiverComponent.h:45,51`) |

Reported as absent from a recursive grep of the corresponding engine sources in 5.3, 5.5 and 5.7.
`#if MCP_HAS_WATER` does not help for the second: it gates on "Water plugin present", not on engine
version.

`ALandscapeProxy::UpdateAllComponentMaterialInstances`, the sibling call on the next lines of the
same header, **is** present in all of 5.3-5.8 — so the landscape header is otherwise portable and
one unguarded call is what pins it to 5.8.

## The thing to decide, which is not a code question

`CLAUDE.md` still states the supported range as **UE 5.3-5.8** in three places (project overview,
the version-compat convention, the `Build.cs` notes), and the whole `UE_VERSION_*` convention exists
to hold that line. This changeset is the first that knowingly breaks it. Either:

- the advertised range is now 5.8-only (or 5.6+), and `CLAUDE.md`, the `.uplugin` and any store
  listing should say so; or
- the range still stands, and these two call sites need `UE_VERSION_NEWER_THAN_OR_EQUAL` guards
  with a `SendUnsupportedEngineVersion` rejection on older engines, per the existing convention.

Filed rather than fixed because the instruction for the pass that shipped it was explicit, and
choosing the supported range is a product decision, not a build fix. Nothing is broken on 5.8.

## History
- `#1-found-by-audit` `OPEN` reporter — Found by auditing the changeset against `Docs/rpc-design.md`
  during integration pass 11. The pass's own brief said "Target UE 5.8 only — no version guards", so
  the finding is out of scope for that pass by instruction; recorded here so the conflict with
  `CLAUDE.md`'s stated 5.3-5.8 range is not lost. Engine-version availability was reported by the
  audit from grepping the installed 5.3/5.5/5.7 trees; **not independently re-verified** against
  those trees by the integration pass.
