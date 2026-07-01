---
id: E-cloth-verbs-wiki-overpromise-create-section
title: "skeleton cloth-verb summaries advertise 'Create and bind' / 'Attach to a specific section' but both verbs only list/bind-existing — no skeleton.md overlay caveats them, so callers trial-and-error then read plugin C++"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [docs, skeleton, cloth, chaos-cloth, bind_cloth_to_skeletal_mesh, assign_cloth_asset_to_mesh, misleading-doc, discoverability, wiki]
---

# The cloth verbs' summaries oversell create/section-assign; the wiki gives no caveat, so the task degraded into trial-and-error + a source dive

This is the **docs-process** sibling of the judge's capability ticket
`F-cloth-create-and-section-assign` (feature: there is no verb that creates a
`UClothingAsset` or attaches one to a chosen section). That F-ticket's action is
C++ capability work (add a create verb / add `clothAssetName`+`sectionIndex`
params) and may or may not ever ship. This ticket is the **wiki-overlay fix that
stands on its own regardless of whether the feature lands**: today the only
documentation a caller reads before acting positively advertises a workflow the
verbs cannot perform, with no caveat — which is exactly what converted a
well-specified 5-step task into wasted process.

Same shape and same precedent as the accepted
`E-ik-rig-family-wiki-advertises-compiled-out-workflow` (the judge filed the
capability `F-` ticket; the auditor filed the docs-overlay process angle for a
wiki that sells a non-functional path). The over-advertisement is the worst kind
of doc gap: it does not merely omit a limitation, it presents a non-working path
as the canonical one.

## What the docs promise vs what the verbs do

- `skeleton.bind_cloth_to_skeletal_mesh` is summarized **"Create and bind a
  UClothingAsset to one or more sections of a SkeletalMesh."** The handler
  (`SkeletalMeshHandler.cpp` ~L848-868) **never creates anything**: with
  `clothAssetName` set it only looks up an *existing* asset and returns
  `CLOTH_NOT_FOUND` when absent; with the name omitted it merely lists the mesh's
  existing clothing assets.
- `skeleton.assign_cloth_asset_to_mesh` is summarized **"Attach an existing
  UClothingAsset to a specific section ... or share a cloth asset between
  sections."** The handler (~L920-960) declares **only** `skeletalMeshPath` as a
  valid param and merely lists clothing assets — it accepts no `clothAssetName`
  and no `sectionIndex`, so it can attach nothing to any section (those params are
  rejected `UNKNOWN_PARAMS`).

There is **no `### skeleton.bind_cloth_to_skeletal_mesh` / `###
skeleton.assign_cloth_asset_to_mesh` overlay** in `docs/wiki-src/skeleton.md` at
all — the misleading one-liners come straight from the C++ `REGISTER_RPC_HANDLER`
summaries, so a caller landing on either verb gets the overpromise with zero
caveat from any wiki page.

## PROCESS friction this caused (this task — skeleton, 11 calls)

The agent did the right thing per the docs: read the namespace index, then read
the `describe_mesh` / `get_info` / `bind_cloth_to_skeletal_mesh` /
`assign_cloth_asset_to_mesh` wiki pages (5 wiki-nav reads), inspected the mesh,
then tried to execute the documented "create → bind → share across sections"
pipeline. The misleading summaries then cost:

- **Misuse-then-correct, driven by the "Create and bind" wording.**
  `bind_cloth_to_skeletal_mesh` with `clothAssetName=DinoDragon_BodyCloth` →
  `[CLOTH_NOT_FOUND] Cloth asset 'DinoDragon_BodyCloth' not found on mesh`
  (is_error). The agent then **re-called the same verb without a name** ("list
  mode") to discover it lists rather than creates — a corrected probing call the
  doc made necessary.
- The **confusing-error angle**: `CLOTH_NOT_FOUND` contradicts a summary that
  says the verb *creates* the asset. A caller reasonably reads "Create and bind"
  + "not found" as "I supplied a bad name / wrong casing," not "this verb cannot
  create and there is no create verb anywhere."
- **Fallback to reading plugin C++ as the last resort** to reconcile docs vs
  behavior — the only place the truth is discoverable.

Friction note verbatim: *"the wiki summary for
skeleton.bind_cloth_to_skeletal_mesh says it 'Create[s] and bind[s] a
UClothingAsset' but the handler ... only looks up an existing cloth asset by name
and never creates one (CLOTH_NOT_FOUND); assign_cloth_asset_to_mesh's doc says
'Attach to a specific section' yet it only accepts skeletalMeshPath and merely
lists assets ...; there is no cloth-creation verb at all and describe_mesh has no
cloth field - had to read plugin C++ as last resort to confirm the docs vs
behavior mismatch."*

## Why this is distinct from `F-cloth-create-and-section-assign`

- `F-cloth-create-and-section-assign` is `category: feature` — its central ask is
  new capability (a create-cloth verb + real section-assign params), C++ work that
  may never ship. Its "at minimum correct the summaries" line is a secondary
  fallback, not the ticket's thrust.
- This is `category: ergonomic`/`docs` — the **honest-caveat overlay** that fixes
  the *process* cost (trial-and-error + source dive) independently and cheaply
  today, exactly the split that was accepted for the IK Rig family
  (`F-ik-rig-retargeter-family-not-compiled` capability + the
  `E-ik-rig-family-wiki-advertises-compiled-out-workflow` docs angle).

## Fix (wiki overlay — downstream wiki process, not this audit)

Add overlay sections to `docs/wiki-src/skeleton.md` for both verbs (and align the
C++ `REGISTER_RPC_HANDLER` summaries):

1. `### skeleton.bind_cloth_to_skeletal_mesh` — change the one-liner from "Create
   and bind a UClothingAsset" to **"bind / list an *already-existing*
   UClothingAsset"**; state plainly it does **not** create cloth, that
   `clothAssetName` must name an asset already present on the mesh (else
   `CLOTH_NOT_FOUND`), and that omitting the name lists the mesh's existing
   clothing assets.
2. `### skeleton.assign_cloth_asset_to_mesh` — change "Attach to a specific
   section ... share between sections" to **"list the mesh's clothing assets"**;
   document that it accepts **only** `skeletalMeshPath` (no `clothAssetName` /
   `sectionIndex`), so it cannot attach to or share across a section.
3. Both pages: note there is currently **no MCP verb that creates a
   UClothingAsset** and that `skeleton.describe_mesh` exposes no cloth field, and
   cross-reference `F-cloth-create-and-section-assign` for the capability work.

## Verbatim repro

Mesh: `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon` (2 sections, 0
clothing assets).

1. `skeleton.bind_cloth_to_skeletal_mesh`
   `{"skeletalMeshPath":".../SK_DinoDragon","clothAssetName":"DinoDragon_BodyCloth","meshLodIndex":0,"sectionIndex":0,"assetLodIndex":0}`
   → `[CLOTH_NOT_FOUND] Cloth asset 'DinoDragon_BodyCloth' not found on mesh`
   (summary says it would *create* the asset).
2. `skeleton.bind_cloth_to_skeletal_mesh` with no name → lists (the real
   behavior, discovered by retry).
3. `skeleton.assign_cloth_asset_to_mesh` with `skeletalMeshPath` only → lists
   (summary says "attach to a specific section").

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `skeleton.bind_cloth_to_skeletal_mesh` DinoDragon cloth-setup task (11 calls;
  judge filed `F-cloth-create-and-section-assign` for the capability/behavior root
  cause). PROCESS angle distinct from that feature ticket: the C++-registered
  summaries advertise "Create and bind a UClothingAsset" and "Attach ... to a
  specific section / share between sections," but `bind_cloth_to_skeletal_mesh`
  only binds/lists an *existing* asset (`SkeletalMeshHandler.cpp` ~L848-868,
  `CLOTH_NOT_FOUND` when absent) and `assign_cloth_asset_to_mesh` accepts only
  `skeletalMeshPath` and merely lists (~L920-960). No `skeleton.md` overlay
  caveats either verb, so the overpromise reaches the caller uncorrected. Cost
  this task: 5 up-front wiki-nav reads, a misuse-then-correct (`bind` with
  `clothAssetName` → `CLOTH_NOT_FOUND`, then re-called name-less to discover it
  lists), a confusing error ("Create and bind" vs "not found"), and a fall-back
  read of the plugin C++ as the only place the truth is discoverable. Fix is
  wiki-overlay-only and stands independent of the F- capability ticket: add
  `### skeleton.bind_cloth_to_skeletal_mesh` / `### skeleton.assign_cloth_asset_to_mesh`
  overlays to `docs/wiki-src/skeleton.md` (+ align the C++ summaries) saying both
  verbs bind/list an existing asset only, that no verb creates a UClothingAsset,
  and cross-reference `F-cloth-create-and-section-assign`. Same accepted split as
  `E-ik-rig-family-wiki-advertises-compiled-out-workflow` (docs angle) vs
  `F-ik-rig-retargeter-family-not-compiled` (capability). No existing E-/docs
  ticket covers the cloth-verb wiki summaries. Severity Medium — the summaries
  actively misdirect on a core, well-specified skeletal-mesh task.
- `#2-honest-summaries-and-overlay` `IN-REVIEW` developer — Implemented as the
  docs/process ticket (GO, not deferred on `F-cloth-create-and-section-assign`):
  the overlay sections are the dominant, independent deliverable the F- ticket
  does NOT carry, and per the accepted IK-rig precedent the docs angle is a real
  deliverable. To avoid double-editing the two shared summary strings, this ticket
  OWNS the C++ summary correction (the F- ticket is left scoped to pure capability
  — its "at minimum correct the summaries" fallback is now satisfied here).
  Changes: (1) `Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp`
  — rewrote both `REGISTER_RPC_HANDLER` summaries: `bind_cloth_to_skeletal_mesh`
  now says it binds/lists an ALREADY-EXISTING `UClothingAsset` (CLOTH_NOT_FOUND
  when absent), does NOT create one; `assign_cloth_asset_to_mesh` now says it
  lists only, takes ONLY `skeletalMeshPath` (clothAssetName/sectionIndex rejected
  UNKNOWN_PARAMS). (2) `Docs/wiki-src/skeleton.md` — added the two missing
  `### skeleton.bind_cloth_to_skeletal_mesh` / `### skeleton.assign_cloth_asset_to_mesh`
  H3 overlay sections (PLAIN headings, no backticks — `WikiOverlay::LoadMethodSection`
  keys on the exact post-`### ` text without backtick-stripping, so a backticked
  heading would never match the method name, the same reason `skeleton.list_bones`
  uses a plain heading; the older backticked describe_mesh/describe_skin_weights
  headings are in fact dead overlays for the same reason). The sections state no
  MCP verb creates a `UClothingAsset`, that `describe_mesh` has no cloth field, and
  cross-reference `F-cloth-create-and-section-assign`. (3) Regression test
  `Source/PinWright/Private/Tests/Infra/TestClothVerbHonestSummaryDocs.cpp` renders
  both method pages through the live `WikiHandler::RenderPage` (the gateway's doc
  path) and asserts the corrected summary text (no "Create and bind" / "Attach an
  existing"; presence of ALREADY-EXISTING/CLOTH_NOT_FOUND and List-only/ONLY
  skeletalMeshPath) AND the overlay markers (no-create note, UNKNOWN_PARAMS caveat,
  F- ticket cross-ref) — reverting either the summary edit or the overlay (or
  re-backticking the heading) fails it. Not compiled/run here (later phase).
</content>
</invoke>
