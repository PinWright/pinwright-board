---
id: E-ik-chain-bone-name-discovery
title: "add_ik_chain's wiki section never cross-references skeleton.list_bones as the startBone/endBone source — and that method has no hand-authored overlay section — so from the animation.authoring IK workflow the bone-name reader is non-obvious (get_animation_info on a Skeleton returns only assetType, asset.get returns no bones), leading to an asset.dump + on-disk skeleton.json read"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, animation, animation-authoring, skeleton, ik-rig, add-ik-chain, bone-names, discoverability, list-bones, wiki, cross-reference]
---

# The IK-chain authoring workflow never points at `skeleton.list_bones` as the bone-name source, and that method has no hand-authored overlay section

`animation.authoring.add_ik_chain` requires `startBone` and `endBone`
(`docs/wiki-src/animation.authoring.md:214` — "a retarget chain is a bone span";
bad bones return `CHAIN_NOT_ADDED`; the IK-rig H3 section is at :178-216). So the
prerequisite for calling it is
knowing the rig skeleton's **bone names**. The RPC that answers this —
`skeleton.list_bones` — *is* discoverable on the `skeleton` namespace page:
because it registers under category `"skeleton"`
(`SkeletonHandler.cpp:137-188`), the wiki's auto-generated `## Methods` index
(`WikiHandler.cpp:397-408`, sourced from the registry at `:144`) lists it with
its summary on `call("skeleton")`, and `call("skeleton.list_bones")` renders its
full param spec from the registry. So the headline is NOT "list_bones is
undocumented/invisible" — a caller who navigates to the obvious `skeleton`
namespace finds it in one hop.

The real, narrower gap is a **missing cross-reference**: from where the agent
was actually standing — the `animation.authoring` IK-chain workflow — nothing
routes them over to `skeleton.list_bones`. The `add_ik_chain` overlay section
never names it as the `startBone`/`endBone` source, and `skeleton.list_bones`
has **no hand-authored overlay section** in `docs/wiki-src/skeleton.md` (which
documents only `describe_mesh` + `describe_skin_weights`) — only the bare
auto-generated registry line. So the two readbacks an agent naturally reaches
for from the animation namespace both dead-end and never redirect:

- `animation.authoring.get_animation_info` on a `USkeleton` returns only
  `{assetType:"Skeleton"}` — no bone array, no hierarchy (the rich
  `duration`/`numFrames`/… shape exists only on the `UAnimSequence` branch; the
  Skeleton branch is bare, the same per-asset-type thinness as
  `E-get-animation-info-thin-on-blend-space` for BlendSpaces).
- `asset.get` on the Skeleton (and on the `SkeletalMesh`) returns metadata with
  **no bones** in the result.

So a caller authoring an IK chain from the `animation.authoring` namespace — with
a legitimate, well-specified intent like "add a LeftArm chain from `shoulder_l`
to `hand_l`" — gets no in-workflow pointer to where the valid bone names come
from, neither before calling `add_ik_chain` nor after a `CHAIN_NOT_ADDED`
rejection.

The answer RPC exists and works: **`skeleton.list_bones`** enumerates the
reference skeleton's bones (`name`/`index`/`parentIndex`/`parentName`/`location`,
per `Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp:137-188`,
also cited by `E-skeleton-list-bones-no-limit-spills`). It is reachable by
navigating to the `skeleton` namespace (its auto-generated registry line +
summary appear there). What is missing is (a) any cross-reference from the
`add_ik_chain` overlay — the caller in the animation workflow never learns to look
at the `skeleton` namespace — and (b) a hand-authored `### skeleton.list_bones`
overlay section in `skeleton.md` describing what it returns (currently
`LoadMethodSection` finds nothing, so `call("skeleton.list_bones")` shows only the
auto param spec, no curated description). Lacking the cross-reference, the agent
concluded "no RPC lists skeleton bones" and fell back to `asset.dump` of the
Skeleton followed by a **Read of the on-disk `skeleton.json` sidecar** to extract
bone names — the last-resort file-dump fallback, on a routine bone-name lookup.

## Why this is a distinct PROCESS angle

- Not the judge's `B-create-pose-library-noop-fake-success` (the no-op
  pose-library stub — the task's central tool bug) nor the `add_ik_chain`
  capability problem already owned by `F-ik-rig-retargeter-family-not-compiled`
  (`create_ik_rig` only sets the preview mesh, never the controller skeleton, so
  every bone is rejected). This ticket is the upstream **discoverability** gap
  that bit the agent *before* the chain call even mattered: it could not find the
  bone names to try.
- Distinct from `E-skeleton-list-bones-no-limit-spills` (OPEN) — that ticket
  assumes the caller already **found** `list_bones` and is about its missing
  `limit`/`nameFilter`/projection (it overflows the inline budget). This ticket
  is one step earlier: from the `animation.authoring` IK-chain workflow,
  `list_bones` is not discoverable at all (undocumented + uncross-referenced), so
  the caller never reaches the pagination problem — they fall to `asset.dump`
  instead. Fixing this (documenting + cross-referencing `list_bones`) is what
  routes future callers *into* that ticket's method.
- Distinct from `E-get-animation-info-thin-on-blend-space` (OPEN) and
  `E-rpc-animation-extend-get-animation-info` (DONE) — those are the
  per-asset-type thinness of `get_animation_info` for BlendSpace / AnimSequence.
  This ticket's docs ask is the cross-namespace pointer for **bone-name
  discovery** during IK-chain authoring; the get_animation_info-Skeleton-is-bare
  observation is cited only as one of the two dead-ends the agent hit, not the
  fix surface.

## Friction evidence (this task — animation.authoring create_pose_library toolkit, 22 calls)

To add the `LeftArm`/`RightArm` chains the agent needed DinoDragon bone names and
burned five readback attempts hunting for them:

1. `animation.authoring.get_animation_info {assetPath:SK_DinoDragon_Skeleton}`
   → `assetType:Skeleton` only, **no bone list**.
2. wiki-nav of `animation` → no list-bones method surfaced.
3. `asset.get {assetPath:SK_DinoDragon}` → no bones in result.
4. `asset.dump SK_DinoDragon` → `skeletal_mesh.json` has **no bone list**.
5. `asset.dump SK_DinoDragon_Skeleton` → `skeleton.json` with the full bone
   hierarchy — obtained only by **reading the dumped file off disk**.

Friction note verbatim: *"a discoverability gap: no RPC lists skeleton bones
(animation.authoring.get_animation_info on a Skeleton returns only assetType,
asset.get returns no bones) - I had to asset.dump the Skeleton and read
skeleton.json on disk to get bone names."* The agent's premise ("no RPC lists
skeleton bones") is the symptom of a missing cross-reference, not a missing
method: `skeleton.list_bones` exists and is listed on the `skeleton` namespace
page, but nothing in the `animation.authoring` IK workflow the agent was in
points across to it.

## Fix (wiki overlay — add the cross-reference / dead-end redirect)

The scope is **not** "document an undiscoverable RPC" (the auto Methods index
already surfaces `list_bones` on the `skeleton` namespace page) — it is to add
the cross-reference from the IK-chain authoring workflow and a hand-authored
overlay section so the method page carries a curated description:

1. **`docs/wiki-src/animation.authoring.md`** — add a dedicated
   `### animation.authoring.add_ik_chain` H3 overlay section (so it renders on the
   `call("animation.authoring.add_ik_chain")` method page via `LoadMethodSection`,
   which the current multi-verb `### … create_ik_rig / add_ik_chain / …` heading
   at :178-216 does NOT — its non-method-name key matches no method page). Its
   one-line pointer: the valid `startBone`/`endBone` names come from
   `call("skeleton.list_bones", {skeletalMeshPath|skeletonPath})`, so a caller
   authoring an IK chain finds the bone-name source without leaving the workflow,
   and a `CHAIN_NOT_ADDED` rejection has an obvious next step instead of an
   `asset.dump` + on-disk-`skeleton.json` detour.
2. **`docs/wiki-src/skeleton.md`** — add a `### skeleton.list_bones` overlay
   section (currently only `describe_mesh` + `describe_skin_weights` have curated
   sections) describing that it enumerates the reference skeleton's bones
   (`name`/`index`/`parentIndex`/`parentName` + ref-pose `location`) from
   `skeletonPath` or `skeletalMeshPath`, and naming it as the bone-name source for
   `animation.authoring.add_ik_chain`. (Pair with
   `E-skeleton-list-bones-no-limit-spills` so the same section also carries the
   all-bones-overflows-inline caveat once that lands.)
3. Optionally note on the `get_animation_info` overlay that the **Skeleton** asset
   branch is bare (only `assetType`) and that bone enumeration is
   `skeleton.list_bones`, not `get_animation_info` — so the dead-end the agent hit
   at step 1 redirects in one hop.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `animation.authoring.create_pose_library` DinoDragon pose-library + IK-rig
  toolkit task (22 calls; outcome tool_bug, judge filed
  `B-create-pose-library-noop-fake-success` for the no-op stub). Distinct PROCESS
  angle: to author the `LeftArm`/`RightArm` IK chains via `add_ik_chain` (which
  requires `startBone`/`endBone`) the agent needed DinoDragon bone names but had
  no discoverable reader — `get_animation_info` on a `USkeleton` returns only
  `{assetType:"Skeleton"}` (no bones) and `asset.get` returns no bones, so it
  burned 5 readback attempts and fell back to `asset.dump` of the Skeleton + a
  **Read of the on-disk `skeleton.json` sidecar** to extract bone names. The RPC
  that answers this — `skeleton.list_bones`
  (`Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp:137-188`) — is
  **undocumented**: `docs/wiki-src/skeleton.md` documents `describe_mesh` and
  `describe_skin_weights` but not `list_bones`, and
  the `animation.authoring` IK-chain section never cross-references it, so it is
  invisible from where the agent was standing (the agent's friction-note premise
  "no RPC lists skeleton bones" is the symptom). Fix is wiki-overlay-only: add a
  `### skeleton.list_bones` section to `skeleton.md`, and a one-line
  bone-name-source pointer (`call("skeleton.list_bones", …)`) to the
  `add_ik_chain` section of `animation.authoring.md`, optionally noting the
  Skeleton branch of `get_animation_info` is bare. Deduped: distinct from
  `E-skeleton-list-bones-no-limit-spills` (that ticket is `list_bones`'
  pagination/projection once you've *found* it; this is one step earlier —
  finding it at all), from `E-get-animation-info-thin-on-blend-space` /
  `E-rpc-animation-extend-get-animation-info` (per-asset-type get_animation_info
  thinness), and from the judge bug + `F-ik-rig-retargeter-family-not-compiled`
  (the pose-library stub / the create_ik_rig-doesn't-set-controller-skeleton
  capability). Severity Low — a clear `asset.dump`+on-disk-read workaround exists
  and the answer RPC already ships; this is purely the discovery/cross-reference
  cost.
- `#2-reword-and-fix` `IN-REVIEW` developer (host A) — Reworded to the accurate scope: the
  headline "list_bones is undocumented/invisible" was overstated — because it
  registers under category `skeleton`, the auto-generated `## Methods` index
  (`WikiHandler.cpp:397-408`, sourced from the registry at `:144`) already lists
  it on `call("skeleton")`. The real defect is the **missing cross-reference**
  from the `animation.authoring` IK workflow plus the absent hand-authored
  overlay section. Implemented the wiki-overlay fix: (1)
  `Docs/wiki-src/animation.authoring.md` — added a dedicated
  `### animation.authoring.add_ik_chain` H3 section (the prior multi-verb
  `### … create_ik_rig / add_ik_chain / …` heading keyed a non-method name, so it
  surfaced on NO method page) naming `call("skeleton.list_bones", …)` as the
  `startBone`/`endBone` source and redirecting the bare-`get_animation_info` /
  `asset.get` dead-ends; (2) `Docs/wiki-src/skeleton.md` — added a
  `### skeleton.list_bones` overlay section describing its return shape
  (`name`/`index`/`parentIndex`/`parentName`/ref-pose `location`) and naming it
  the bone-name source for `add_ik_chain`. Regression test:
  `Source/PinWright/Private/Tests/Infra/TestIkChainBoneNameDiscoveryDocs.cpp`
  (`FAddIkChainBoneNameSourceDocTest`, `FSkeletonListBonesOverlayDocTest`) renders
  both method pages through the live `WikiHandler::RenderPage` path and asserts the
  overlay-exclusive cross-reference markers survive — reverting either overlay edit
  makes `LoadMethodSection` return empty and the test fails. Not compiled (later
  phase). Released the fuzz2 claim.
- `#2-reword-and-fix` `IN-REVIEW` developer (host B, merged) — Independently reworded to fix two factual slips
  (skeleton.md documents `describe_mesh` **and** `describe_skin_weights`, not "only
  describe_mesh"; stale citations corrected to
  `Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp:137-188` and the
  add_ik_chain prereq at `animation.authoring.md:214`, H3 at :178-216). Implemented
  the wiki-overlay fix on three surfaces: (1) added a `### skeleton.list_bones` H3 to
  `docs/wiki-src/skeleton.md` documenting the bone enumerator
  (`name`/`index`/`parentIndex`/`parentName`/`location` + `count`, `SKELETON_NOT_FOUND`)
  and framing it as the canonical bone-name reader (pointing at `add_ik_chain` /
  `get_bone_transform` consumers); (2) added a **dedicated**
  `### animation.authoring.add_ik_chain` H3 to `docs/wiki-src/animation.authoring.md`
  pointing at `skeleton.list_bones` as the `startBone`/`endBone` source — a dedicated
  H3 was required because the IK-rig family's combined
  `### create_ik_rig / add_ik_chain / …` heading matches no single registered method
  name, so `WikiOverlay::LoadMethodSection("…add_ik_chain")` (WikiHandler.cpp:418)
  returned nothing and the per-method page rendered no overlay notes; (3) noted in the
  `## Inspect-after-mutate` section that `get_animation_info` on a `USkeleton` is bare
  (`{assetType:"Skeleton"}` only) and routes bone enumeration to `skeleton.list_bones`.
  Regression test
  `Source/PinWright/Private/Tests/Infra/TestIkChainBoneNameDiscoveryDocs.cpp` (now
  carries both hosts' cases) renders the live `WikiHandler::RenderPage` path for
  `skeleton.list_bones`, `animation.authoring.add_ik_chain`, and the
  `animation.authoring` namespace page and asserts the overlay-only markers
  (`skeleton.list_bones` pointer, `CHAIN_NOT_ADDED`, the per-bone fields, the
  Skeleton-bare note) survive — reverting the overlay edits fails it. No handler code
  changed; the answering RPC already ships. Merged with host A's reword during rebase.
