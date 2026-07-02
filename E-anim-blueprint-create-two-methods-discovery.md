---
id: E-anim-blueprint-create-two-methods-discovery
title: "THREE near-identically-named anim-BP creators (`animation.create_animation_bp`, top-level `animation.create_anim_blueprint`, and `animation.authoring.create_anim_blueprint`) and the overlay prose never disambiguates which to pick — forces wiki-nav across pages"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [animation, anim-blueprint, create, docs, discoverability, naming, wiki, meshPath, skeletonPath]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# Picking the right anim-BP creator is undocumented in the overlay prose

There are **three** methods that create an Animation Blueprint, with
confusingly near-identical names — and the verb `create_anim_blueprint`
exists in **both** the `animation` and `animation.authoring` namespaces:

- `animation.create_animation_bp` (top-level convenience,
  `AnimationHandler.cpp:329`). Accepts **`meshPath`** (auto-resolves the
  Skeleton from a SkeletalMesh) OR **`skeletonPath`**, plus an optional
  `parentClass`. Routes through `FAssetToolsModule::CreateAsset` (line 393),
  which de-dupes a name collision safely.
- `animation.create_anim_blueprint` (top-level, a **sibling** convenience,
  `AnimationHandler.cpp:1293`). Mostly equivalent; also accepts **`meshPath`**
  OR **`skeletonPath`**, but requires an explicit `savePath`. Same safe
  `CreateAsset` path (line 1358). **The ticket's own seed verb** — yet it is a
  distinct top-level method, not the authoring one.
- `animation.authoring.create_anim_blueprint` (authoring namespace,
  `AnimationAuthoringHandler_AnimBlueprint.cpp:562`). Requires an explicit
  **`skeletonPath`** (REQ, line 567; **no `meshPath`**), via raw
  `UAnimBlueprintFactory::FactoryCreateNew` (line 620). On an in-session name
  collision it returns a clean **`ASSET_EXISTS`** rejection (guarded by
  `PrepareBlueprintPackageGuardingNameCollision`, line 601) — it no longer
  crashes the editor (see `B-create-anim-blueprint-duplicate-name-crash`,
  now IN-REVIEW with that guard shipped).

So the title's old "meshPath vs skeletonPath" split is wrong: **both** top-level
verbs accept `meshPath`+`skeletonPath`; only the authoring verb is
`skeletonPath`-only. The discovery friction is real, but the cause is narrower
than first reported.

## What is and isn't already documented

The auto-generated `## Methods` index on each namespace page is sourced from the
registry Summary strings (per the plugin CLAUDE.md "Wiki Authoring Constraints"),
and those summaries **already cross-reference** the siblings by name:

- `create_animation_bp` summary (`AnimationHandler.cpp:330`): "Largely overlaps
  with animation.create_anim_blueprint…".
- top-level `create_anim_blueprint` summary (`:1294`): "…mostly equivalent to
  animation.create_animation_bp. Resolves the Skeleton from skeletonPath or
  extracts it from meshPath."
- authoring `create_anim_blueprint` summary (`:563`): "Largely overlaps with
  animation.create_animation_bp; pick this entry when you want full control over
  save path and parent."

So the bare cross-link the original Fix asked for *already renders* via the auto
Methods index. What is **missing** is a single "which anim-BP creator?"
disambiguation in the hand-authored overlay prose:

- `docs/wiki-src/animation.md` "How to use" (line 9) lists `create_animation_bp`
  as a one-shot creator but never groups the three creators, never explains the
  `meshPath` auto-resolve, and never names the safe-`CreateAsset` vs
  authoring-`FactoryCreateNew` split.
- `docs/wiki-src/animation.authoring.md` "Workflow gotchas" (line 30) states the
  authoring `create_anim_blueprint` needs a pre-existing Skeleton, but does not
  point at the top-level `meshPath`-capable siblings or note the `ASSET_EXISTS`
  guard.

(There is a real, larger duplication smell here — three self-described "mostly
equivalent" creators — but consolidating/aliasing them is a separate refactor,
not this docs ticket. Filed scope below is the proven docs-signpost pattern.)

## Evidence

From this task's call-log (seed `animation.create_anim_blueprint`, locomotion
ABP build-out). The wiki-nav block navigated to **both** creators before the
execute phase picked the right one:
- `animation.authoring.create_anim_blueprint` (wiki-nav)
- `animation.create_animation_bp` (wiki-nav, annotated "meshPath variant")
…then executed `animation.create_animation_bp` with
`name=ABP_MannequinLocomotion savePath=/Game/Animations meshPath=SK_Mannequin`.

The outcome was clean (no retry on the wrong method), so this is pure discovery
friction, not a tool bug.

## Fix

Wiki-only (docs). Follow the accepted overlay-cross-reference pattern proven by
`E-animation-sequence-info-reader-name-split` (reciprocal `##` sections in both
overlays + a `WikiHandler::RenderPage` regression test). Add a
**"Which anim-BP creator?"** `##` section to **both** overlays (these render on
the namespace page, not the root index):

- `docs/wiki-src/animation.md` — list all three creators, name
  `animation.create_animation_bp` as the recommended one-shot creator that
  accepts `meshPath` (auto-resolve) OR `skeletonPath` and de-dupes safely via
  `CreateAsset`, note the sibling top-level `animation.create_anim_blueprint`,
  and point at the `animation.authoring` peer for full save-path/parent control.
- `docs/wiki-src/animation.authoring.md` — near the Workflow gotchas, note the
  authoring `create_anim_blueprint` is `skeletonPath`-only (no `meshPath`) and
  rejects an in-session name collision with `ASSET_EXISTS` (NOT "crashes" — the
  guard shipped), and cross-link the top-level `meshPath`-capable
  `animation.create_animation_bp`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the locomotion-ABP build-out task (seed `animation.create_anim_blueprint`, outcome "ergo", clean). Discovery friction: the wiki-nav block navigated to BOTH `animation.authoring.create_anim_blueprint` and `animation.create_animation_bp` ("meshPath variant") before executing the latter — two near-identically-named creators across two namespaces with different skeleton inputs (`meshPath` vs `skeletonPath`), where neither wiki overlay (`animation.md:9`, `animation.authoring.md:12,28`) cross-links the other or documents the `meshPath` auto-resolve. The obvious-named in-namespace `create_anim_blueprint` is also the worse pick (no `meshPath`; crashes on in-session dup — `B-create-anim-blueprint-duplicate-name-crash`). Distinct PROCESS angle from the judge-filed `E-anim-graph-omits-state-machine-entry-state` (readback gap) and from the crash bug. Severity Low (docs/discoverability), not bumped — anim-BP creation is not an every-session method. Fix is wiki-only: cross-link the two creators in both overlay pages with a short "which creator?" disambiguation.
- `#2-reword-three-creators-and-fix` `IN-REVIEW` developer — REWORDED to match source (three lens reports agreed the body mis-stated facts): there are THREE creators, not two — `animation.create_animation_bp` (`AnimationHandler.cpp:329`), a distinct TOP-LEVEL `animation.create_anim_blueprint` (`:1293`, the ticket's own seed verb), and `animation.authoring.create_anim_blueprint` (`AnimationAuthoringHandler_AnimBlueprint.cpp:562`); BOTH top-level verbs accept `meshPath`+`skeletonPath` (so the old "meshPath vs skeletonPath" title axis was wrong — only the authoring verb is skeletonPath-only); the authoring verb no longer "crashes the editor" — it returns a guarded `ASSET_EXISTS` (`:601`, `B-create-anim-blueprint-duplicate-name-crash` IN-REVIEW); and the registry summaries already cross-reference, so only the overlay prose lacks a "which creator?" disambiguation. Implemented the docs fix following the `E-animation-sequence-info-reader-name-split` pattern: added a `## Which anim-BP creator?` section to `Docs/wiki-src/animation.md` (all three creators, the `meshPath` auto-resolve, the safe `CreateAsset` vs authoring `FactoryCreateNew` split) and a reciprocal `## Which anim-BP creator?` section to `Docs/wiki-src/animation.authoring.md` (authoring verb is `skeletonPath`-only and returns `ASSET_EXISTS`, cross-links the top-level `meshPath`-capable creator). Regression test `Source/PinWright/Private/Tests/Infra/TestAnimBlueprintCreatorDisambiguationDocs.cpp` renders both overlays via the live `WikiHandler::RenderPage` path and asserts the overlay-exclusive markers (the `Which anim-BP creator` heading, all three creator names, the `meshPath` auto-resolve note, and the `ASSET_EXISTS` guard wording) — reverting the overlay edits fails the assertions. No code change to the handlers.
