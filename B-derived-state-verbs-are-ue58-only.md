---
id: B-derived-state-verbs-are-ue58-only
title: "The derived-state changeset calls APIs that do not exist before UE 5.8 / 5.6, against a plugin that still advertises UE 5.3-5.8"
status: DONE
severity: Medium
category: bug
tags: [build, engine-compat, version-guards, water, landscape, material-authoring, packaging, docs, decision]
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

## Decision (resolved in `776ec8f5`)

**Neither option as written. The range statement was made true without narrowing the ambition.**
The user's standing decision is that backporting is *deferred, not cancelled*, so 5.3-5.7 stays the
intent; but a document advertising a range the code cannot build is the same defect class as a verb
reporting success for work it did not do. `CLAUDE.md` and `README.md` now say **5.8 is what is built
and tested today; 5.3-5.7 is intended and deferred**, and point at the blocker list.

No guards were added and no call site was changed. `Ctx.SendUnsupportedEngineVersion` is not the
remedy here: it answers *runtime* absence, and these are *compile-time* absences with no runtime
remedy at all.

The durable artefact is `Docs/engine-version-support.md` — every known blocker with symbol, call
site, affected verbs, first-available engine version and older-engine substitute, so the backport is
costed without a build. It collects three more, from the skeletal-mesh verbs (`4ba04d5b`), which had
already put the floor at 5.6 before this changeset raised it to 5.8:
`EGeometryScriptBoneHierarchyMismatchHandling` (5.6+), `CopyBonesFromSkeleton` (5.5+),
`FGeometryScriptCopyMeshFromAssetOptions::bUseBuildScale` (5.4+). `docs/rpc-design.md` §14 is the new
rule; the two call sites from this changeset now carry the per-version comment it requires.

Corrections to the report above: the `.uplugin` needs no edit (it carries no `EngineVersion` key),
and `product-facts.json` is **generated** from the `package.yml` CI matrix, so narrowing the
advertised range would mean narrowing what CI builds — left at 5.3-5.8 deliberately, as the intent.

## History
- `#1-found-by-audit` `OPEN` reporter — Found by auditing the changeset against `Docs/rpc-design.md`
  during integration pass 11. The pass's own brief said "Target UE 5.8 only — no version guards", so
  the finding is out of scope for that pass by instruction; recorded here so the conflict with
  `CLAUDE.md`'s stated 5.3-5.8 range is not lost. Engine-version availability was reported by the
  audit from grepping the installed 5.3/5.5/5.7 trees; **not independently re-verified** against
  those trees by the integration pass.
- `#2-resolved` `DONE` maintainer — **Re-verified, then resolved as a documentation decision; no code
  change.** Availability confirmed directly against all six installed trees (`C:\UE_5.3` … `C:\UE_5.8`),
  not just the three previously grepped: `RetrieveAllLandscapeMaterials` appears **only** in 5.8
  (`LandscapeProxy.h:1319`) and is absent from the *entire* Landscape module on 5.7, while the sibling
  `UpdateAllComponentMaterialInstances` is on all six (`:1213` on 5.3 … `:1414` on 5.8); the four
  `{Get,Set}River{Width,Depth}AtSplineInputKey` accessors appear from 5.6 (`WaterBodyRiverComponent.h:45,48,51,54`)
  and are absent on 5.3-5.5. Both `#1` claims stand. **Neither disposition in the ticket was taken:**
  the range is not narrowed (backporting is deferred, not cancelled) and no guards were added
  (unverifiable here, and out of scope by the same standing decision). Instead the claim was made
  true — `CLAUDE.md` (overview + *UE version compat*) and `README.md` now state 5.8-built-and-tested
  / 5.3-5.7-intended-and-deferred, and new `Docs/engine-version-support.md` collects every known
  blocker with call site, first-available version and substitute. That file also carries three
  pre-existing rows from `4ba04d5b` (`BoneHierarchyMismatchHandling` 5.6+, `CopyBonesFromSkeleton`
  5.5+, `bUseBuildScale` 5.4+), so the floor was already 5.6 before this changeset — this changeset
  raised it to 5.8, it did not break an intact range. `docs/rpc-design.md` §14 records the rule
  (compile-time absence has no runtime remedy, `#if MCP_HAS_WATER` is a plugin-presence gate not a
  version gate, unguarded is allowed only when written down in both the matrix and a call-site
  comment); `MaterialLandscapeConsumers.h:182-188` and `WaterHandler.cpp:526-534` now carry that
  comment, modelled on the `SkeletalMeshAssetIOHandler.cpp:35-71` header. Shipped in `776ec8f5`,
  pushed to origin/master as a fast-forward. Not built and not test-run: documentation and `//`
  comments only, no compiled code touched. **Still open, deliberately, and not tracked by this
  ticket:** the backport itself. Its scope is the table in `Docs/engine-version-support.md`, which
  states that the table is a list of what has been found, not a completed audit — the CI and Package
  workflows already carry a 5.3-5.8 matrix with an `only:` input and neither has been dispatched
  against this code, so the compiler errors from `only: 5.3` are the remaining rows.
