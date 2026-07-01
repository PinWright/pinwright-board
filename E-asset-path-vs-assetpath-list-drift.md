---
id: E-asset-path-vs-assetpath-list-drift
title: "asset.list declares its slot 'path'; asset.exists / asset.dump require 'assetPath' (no alias) — read-back verbs in the same namespace disagree, costing path->assetPath round-trips"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [asset, param-alias, path, assetpath, asset-list, asset-exists, asset-dump, drift]
---

# Within `asset.*`, `asset.list` names its slot `path` but `asset.exists` / `asset.dump` require `assetPath`

Same param-name-drift class as `E-blueprint-param-name-path-vs-assetpath`
(DONE, the canonical dispatcher-alias fix), `E-widget-asset-path-alias-drift`
(IN-REVIEW), `E-material-editor-param-name-drift` (DONE),
`E-level-create-name-path-alias` (OPEN), and the freshly-filed
`E-geometry-create-name-vs-actorname` (OPEN) — but here it surfaces **inside the
`asset.*` namespace itself**, between read-back verbs that an agent naturally
chains right after one another.

The slot spellings split per-verb, with no cross-alias:
- `asset.list` declares **`path`** — `RPC_PARAM_OPT("path", "string", "Package
  path to list (default /Game)")` (AssetManageHandler.cpp:689).
- `asset.exists` requires **`assetPath`** — `RPC_PARAM_REQ("assetPath", ...)`
  (AssetManageHandler.cpp:622), `Ctx.GetString(TEXT("assetPath"))` at :625, no
  `path` alias. **Same file as `asset.list`.**
- `asset.dump` requires **`assetPath`** — `RPC_PARAM_REQ("assetPath", "string",
  "Package path (e.g. /Game/UI/WBP_HUD)")` (AssetDumpHandler.cpp:1920).

So a caller who just did `asset.list {path:"/Game/Pickups"}` and then naturally
reuses `path` to probe one of the returned assets gets a hard
`MISSING_REQUIRED_PARAM 'assetPath'` from both `asset.exists` and `asset.dump`.
CLAUDE.md's "camelCase and snake_case aliases" rule does not cover this —
`path` and `assetPath` are distinct names, not casing variants — and neither
slot carries the other as an alias, so guessing wrong is a hard error, not a
silent accept.

This is distinct from `E-asset-list-path-ignored` (DONE), which is about
`asset.list` *silently dropping* its own top-level `path` when a `filter`
object is present — a value-routing bug inside one verb. This ticket is the
**cross-verb naming drift**: `path` vs `assetPath` across `asset.list` /
`asset.exists` / `asset.dump`, the same shape as the five precedent
param-alias-drift tickets, none of which touch the `asset.*` read-back slot.

## Repro (verbatim, from the audited task)

A `geometry.convert_to_static_mesh` build verified its baked
`/Game/Pickups/SM_CrystalShard` asset, then:

1. `asset.exists {path:"/Game/Pickups/SM_CrystalShard"}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`
   Retry with `{assetPath:"/Game/Pickups/SM_CrystalShard"}` → `exists=true`.
2. `asset.dump {path:"..."}`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`
   Retry with `{assetPath:"..."}` → succeeds (wrote static_mesh.json).

Friction note (verbatim): "inconsistent param naming across methods (... asset.*
wants 'assetPath' not 'path') caused a few first-try UNKNOWN_PARAMS/MISSING_PARAM
errors that I corrected immediately." Two `path`->`assetPath` round-trips, both
corrected on retry. Errors are accurate (not misleading); zero blocked progress.
Pure guessability / round-trip overhead, the exact shape of the precedent tickets.

The natural drift trap is specifically the `asset.list`->`asset.exists`/`asset.dump`
chain: `asset.list` *teaches* the agent that the asset slot is `path`, and the
very next probe verb rejects it.

## What it should do

Reuse the dispatcher `FParamSpec` alias machinery landed in
`E-blueprint-param-name-path-vs-assetpath #4` (the change that finally made
alias-only required params validate at the wire level, in
`RpcDispatcher::ValidateHandlerParams`). Standardize the alias *set*, not the
canonical name, so existing callers keep working:
- Annotate the `asset.exists` / `asset.dump` (and any other `assetPath`-slot
  asset verbs) `assetPath` spec with a `path` alias, and accept `path` in their
  body reads.
- Optionally annotate `asset.list`'s `path` slot with an `assetPath` alias for
  full symmetry, so the spelling is interchangeable in both directions across
  the namespace.

Do not solve this by making the canonical param optional per-handler (that
leaves aliases undiscoverable, the explicit anti-pattern called out in
`E-blueprint-param-name-path-vs-assetpath #3`).

The wiki pages are individually correct (each verb documents its own slot), so
this is a guessability / alias gap, not a docs gap — reading each verb's page
first avoids the wasted calls, but same-namespace same-asset reuse across a
list->probe chain is the natural friction.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `geometry.create_procedural_mesh` "CrystalShard" hand-authored-mesh task
  (outcome done). After baking the StaticMesh, the verify phase ate two param
  round-trips in the `asset.*` namespace: `asset.exists {path:...}` →
  `[MISSING_REQUIRED_PARAM] ... 'assetPath'`, then `asset.dump {path:...}` →
  same error, each corrected by re-spelling `path` as `assetPath`. Source:
  AssetManageHandler.cpp:622 (`asset.exists` `RPC_PARAM_REQ("assetPath", ...)`,
  no alias) and :689 (`asset.list` `RPC_PARAM_OPT("path", ...)`) live in the
  **same file**; AssetDumpHandler.cpp:1920 (`asset.dump`
  `RPC_PARAM_REQ("assetPath", ...)`). Sibling of the five DONE/IN-REVIEW/OPEN
  path/param-alias-drift tickets, here in the `asset.*` read-back slot — the
  list->probe chain primes `path` then the probe verbs reject it. Distinct
  PROCESS angle from `E-geometry-create-name-vs-actorname` (judge-filed, same
  task, but scoped to geometry's `name`/`actorName`) and from
  `E-asset-list-path-ignored` (DONE, which is `asset.list` dropping its own
  `path` when `filter` is present — a value-routing bug, not cross-verb naming).
  Fix: dispatcher `FParamSpec` alias from `E-blueprint-param-name-path-vs-assetpath #4`,
  aliasing the `asset.exists`/`asset.dump` `assetPath` slot to accept `path`.
- `#2-more-verbs-asset-validate-get` `OPEN` reporter — Same drift, two more
  `asset.*` read-back verbs, from an independent struggle audit of a dependency
  audit task (seed `asset.get_asset_graph` on
  `/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_ButtonLight_Bulb_Basic`,
  outcome done). The agent's first `asset.validate` and `asset.get` calls used
  `path=...` and both hard-failed `[MISSING_REQUIRED_PARAM] Missing required
  parameter 'assetPath' (type: string)`, each corrected by re-spelling `path` ->
  `assetPath` (two more `path`->`assetPath` round-trips, both recovered on retry,
  zero blocked progress). Source confirms both lack a `path` alias:
  `AssetManageHandler.cpp:645` (`asset.get` `RPC_PARAM_REQ("assetPath", ...)`) and
  `:1144` (`asset.validate` `RPC_PARAM_REQ("assetPath", ...)`) — same file/cluster
  as the `asset.exists` (:622) drift already cited here. So the `assetPath`-slot
  verb set in this namespace is now at least `asset.exists` / `asset.dump` /
  `asset.get` / `asset.validate`, all rejecting the `path` spelling that
  `asset.list` (:689, `RPC_PARAM_OPT("path", ...)`) teaches. The same dispatcher
  `FParamSpec` alias fix covers these four uniformly; annotate the shared
  `assetPath` spec with a `path` alias. Friction note (verbatim): "Also hit a
  wrong param name (path vs assetPath) on the first validate/get."
- `#3-asset-dump-again-audio-readback` `OPEN` reporter — Same `asset.dump`
  drift, third independent task. Struggle-audit of an `audio.authoring`
  SoundClass-hierarchy + SoundMix build (seed `audio.authoring.set_class_parent`,
  outcome done; the tool bug filed separately as `B-sound-class-parent-no-child-link`).
  After the audio readbacks, the agent pivoted to `asset.dump` to confirm
  scalar writes `get_audio_info` doesn't surface, and its first two `asset.dump`
  calls used `path=...` — both hard-failed `[MISSING_REQUIRED_PARAM] Missing
  required parameter 'assetPath' (type: string)` (on `path=Dialogue` and
  `path=DefaultMix`), each corrected by re-spelling `path` -> `assetPath` (two
  more `path`->`assetPath` round-trips, both recovered on retry, zero blocked
  progress). Confirms `asset.dump` still lacks the `path` alias cited in `#1`
  (AssetDumpHandler.cpp:1920, `RPC_PARAM_REQ("assetPath", ...)`). Friction note
  (verbatim): "asset.dump initially rejected param 'path' (needed 'assetPath'),
  one retry each." Notably the agent reached `asset.dump` from `audio.authoring`,
  not from `asset.list`, so the `path` priming here is the engine-wide habit, not
  just the list->probe chain — strengthening the case that the `path` alias
  belongs on the shared `assetPath` spec rather than being framed as an
  `asset.list`-local teaching artifact. Same dispatcher `FParamSpec` alias fix.
- `#4-exists-path-direct-list-probe-chain` `OPEN` reporter — Fourth independent
  task, and the first to exhibit the **exact `asset.list`->`asset.exists` chain**
  this ticket's `#1` only theorized (the prior three reached the `assetPath`-slot
  verb from a non-`asset.list` predecessor). Struggle-audit of a
  `foliage.get_instances` scatter-and-cleanup task (outcome done; tool bug filed
  separately as `E-foliage-get-instances-drops-scale`). The asset-discovery phase
  ate two first-try param failures back to back: `asset.exists {path:...SM_Lightbulb}`
  -> `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`,
  corrected to `{assetPath:...}` -> `exists=true`; then `asset.list {classNames,
  limit}` -> `[UNKNOWN_PARAMS] Unknown parameter(s) for 'asset.list': [classNames,
  limit]. Valid parameters: [path, filter, recursive, pagination, depth]. ...`,
  corrected via the wiki page to `{filter:{class:"StaticMesh"}, pagination}`. The
  `asset.exists` half is the same `path`->`assetPath` round-trip the dispatcher
  `FParamSpec` alias fix covers; confirms `asset.exists` still lacks the alias
  (AssetManageHandler.cpp:622). The `asset.list` half is an adjacent guessability
  symptom — the agent's flat mental model (`classNames`/`limit`) diverged from
  the actual `filter.class`/`pagination` nesting — but that error is *accurate*
  (lists valid params, points at the wiki) and recovered cleanly, so it is not a
  separate fileable defect; noted here only as corroborating signal that the
  whole `asset.*` param surface reads as un-guessable to a fresh caller, which
  the alias annotation alleviates for the `path`/`assetPath` axis. Friction note
  (verbatim): "first asset.exists/asset.list calls failed on wrong param names
  (param-shape guesses didn't match), fixed quickly via the verbatim error
  messages and the asset.list wiki page." Both recovered on retry, zero blocked
  progress. Same dispatcher `FParamSpec` alias fix.
- `#5-exists-from-convert-static-mesh-bake` `OPEN` reporter — Fifth independent
  task, same `asset.exists` `path`->`assetPath` round-trip. Struggle-audit of a
  `geometry.create_spiral_stairs` "TowerSpiralStaircase" greybox task (seed
  `geometry.create_spiral_stairs`, outcome clean; nothing else filed — every
  spiral-stairs/geometry/actor call passed first try from the wiki docs). The
  *only* friction in the whole 12-call task was the final verify probe: after
  `geometry.convert_to_static_mesh {assetPath:/Game/GeneratedMeshes/TowerSpiralStaircase}`
  baked the StaticMesh, `asset.exists {path:...}` hard-failed
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`,
  corrected to `{assetPath:...}` -> `exists=true` (one `path`->`assetPath`
  round-trip, recovered on retry, zero blocked progress). Like `#3` (and unlike
  the list->probe framing of `#1`/`#4`), the agent reached `asset.exists`
  directly after a `geometry.*` bake — no preceding `asset.list` to prime `path`
  — so `path` here is the engine-wide caller habit, reinforcing `#3`'s point that
  the alias belongs on the shared `assetPath` spec, not on an `asset.list`-local
  teaching artifact. Confirms `asset.exists` still lacks the `path` alias
  (AssetManageHandler.cpp:622). Friction note (verbatim): "Minor: asset.exists
  rejected param 'path' once; its own error message named the correct param
  'assetPath' and the retry succeeded. All spiral-stairs/geometry calls passed
  first try from the wiki docs." Same dispatcher `FParamSpec` alias fix.
- `#6-asset-list-classnames-from-eqs-authoring` `OPEN` reporter — Sixth
  independent task; corroborates the **`asset.list {classNames}`** guessability
  symptom this ticket's `#4` first noted (and judged accurate / self-correcting /
  not-separately-fileable, only a signal that the `asset.*` param surface reads as
  un-guessable). Struggle-audit of an `eqs.add_test` "FindBestVantagePoint"
  EQS-authoring task (outcome clean; nothing else judge-filed). Mid-task the agent
  needed to find a context class and called
  `asset.list {classNames:[EnvQueryContext_BlueprintBase]}` ->
  `[UNKNOWN_PARAMS] Unknown parameter(s) for 'asset.list': [classNames]. Valid
  parameters: [path, filter, recursive, pagination, depth].`, corrected on retry
  to `asset.list {filter.class:EnvQueryContext_BlueprintBase, /Game}` -> `ok:true`
  (the only `is_error` in an otherwise clean ~37-call task). Identical error and
  valid-param list to `#4`. The `classNames` priming here is the cross-verb drift
  from `asset.search_assets` (whose class-list slot **is** `classNames[]`, per
  `E-class-name-format-inconsistency` and `E-asset-search-vs-search-assets-overlap`)
  carried onto `asset.list` (whose slot is the nested `filter.class`) — same
  "adjacent class-list verbs disagree on the class-filter param name" trap. Per
  `#4`'s disposition this `classNames`/`filter.class` axis is accurate and
  self-correcting, so no new defect; logged here as another data point that the
  `asset.*` param surface (class-filter spelling included) is un-guessable to a
  fresh caller. Friction note (verbatim): "one wrong asset.list param (classNames)
  bounced with a clear UNKNOWN_PARAMS error listing valid params, fixed via
  filter.class on retry." Recovered on retry, zero blocked progress.
- `#7-exists-first-probe-from-montage-task` `OPEN` reporter — Seventh independent
  task, same `asset.exists` `path`->`assetPath` round-trip, reached as the very
  first probe (no preceding `asset.list`, like `#3`/`#5`). Struggle-audit of an
  `animation.authoring.create_montage` "AM_DinoDragon_RoarBite" combat-montage
  build task (outcome ergo; the judge filed the wiki-example casing separately as
  `E-montage-wiki-example-snake-case-params`). Before creating the montage, the
  agent probed the target skeleton existence and its first call was
  `asset.exists {path:SK_DinoDragon_Skeleton}` ->
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`,
  corrected on retry to `{assetPath:SK_DinoDragon_Skeleton}` -> ok (one
  `path`->`assetPath` round-trip, recovered immediately, zero blocked progress;
  the only is_error in an otherwise clean ~15-call montage task). Distinct PROCESS
  angle from the judge's docs ticket (which is the create_montage wiki example's
  snake_case casing): this is the `asset.exists` param-alias round-trip on the
  shared `assetPath` slot, the same dispatcher `FParamSpec` alias fix tracked
  here. Friction note (verbatim): "asset.exists' prose said 'path' but the real
  key is 'assetPath' (one wasted call before correcting)." Notably the agent
  attributes the wrong guess to the **wiki prose** for `asset.exists` saying
  'path' — so beyond the missing alias, the `asset.exists` overlay page text may
  itself name 'path'; the alias fix makes the prose-vs-schema mismatch harmless
  either way. Confirms `asset.exists` still lacks the `path` alias
  (AssetManageHandler.cpp:622). Same dispatcher `FParamSpec` alias fix.
- `#8-exists-first-probe-pose-library-task` `OPEN` reporter — Eighth independent
  task, same `asset.exists` `path`->`assetPath` round-trip as the very first call,
  reached as the opening skeleton-existence probe (no preceding `asset.list`, like
  `#3`/`#5`/`#7`). Struggle-audit of an `animation.authoring.create_pose_library`
  DinoDragon pose-library + IK-rig toolkit task (outcome tool_bug; the judge filed
  `B-create-pose-library-noop-fake-success` for the no-op stub). The agent's very
  first call confirmed the skeleton existence as
  `asset.exists {path:SK_DinoDragon_Skeleton}` ->
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`,
  corrected on retry to `{assetPath:/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton}`
  -> `exists:true` (one `path`->`assetPath` round-trip, recovered immediately,
  zero blocked progress). This is the **same DinoDragon-skeleton existence probe**
  as `#7` (which was a create_montage task) reached from a different seed task —
  reinforcing that `path` is the engine-wide caller habit on the shared `assetPath`
  slot, independent of the task family. Confirms `asset.exists` still lacks the
  `path` alias (AssetManageHandler.cpp:622). Same dispatcher `FParamSpec` alias fix.
- `#9-niagara-inspect-path-and-intra-namespace-alias-split` `OPEN` reporter —
  Ninth independent task, and the first to surface this drift **inside the
  `niagara.*` namespace** (prior eight were all `asset.*`). Struggle-audit of a
  `niagara.set_property` art-pass task on
  `/Game/ExampleContent/Niagara/Simple/RendererOverrides_System` (outcome ergo;
  the judge filed the unrelated `target.kind` alias issue as
  `E-niagara-set-property-emitter-alias`). The agent's **very first call** was
  `niagara.inspect {path:...}` -> `[MISSING_REQUIRED_PARAM] Missing required
  parameter 'assetPath' (type: string)`, corrected on retry to `{assetPath:...}`
  -> ok (one `path`->`assetPath` round-trip, recovered immediately, zero blocked
  progress; one of two is_errors in the task, the other being the unrelated
  emitter-alias issue). What makes the niagara case sharper than the `asset.*`
  cluster: the niagara namespace **itself** is internally split on this axis —
  `niagara.decompile_model` is the lone niagara verb that *does* carry the alias
  (`NiagaraModelHandler.cpp:87-88`: `RPC_PARAM_OPT("assetPath", ...)` +
  `RPC_PARAM_OPT("path", "Alias for assetPath")`, read via
  `GetStringFirstOf({"assetPath","path"})`), while every other niagara verb —
  `niagara.inspect` (NiagaraInspectHandler.cpp:209), `niagara.set_property` /
  edit verbs (NiagaraEditHandler.cpp:1942+, NiagaraAdvancedEditHandler.cpp:88+),
  `niagara.compile` / `niagara.save` (NiagaraCompileHandler.cpp:20,95),
  `niagara.validate` (NiagaraInspectHandler.cpp), curve/graph verbs
  (NiagaraCurveHandler.cpp:231,420; NiagaraGraphHandler.cpp:174+) — declares a
  bare `RPC_PARAM_REQ("assetPath", ...)` with **no** `path` alias. So one verb in
  the namespace establishes `path` as valid and the rest reject it, the same
  intra-namespace teaching trap as `asset.list`->`asset.exists` in `#1`. The same
  dispatcher `FParamSpec` alias fix covers these — annotate the niagara handlers'
  `assetPath` spec with a `path` alias (matching what `niagara.decompile_model`
  already does ad-hoc) for namespace-wide symmetry. Friction note (verbatim):
  "missing assetPath probe (used path param)". Recovered on retry, zero blocked
  progress.
- `#10-asset-list-classnames-second-eqs-authoring` `OPEN` reporter — Tenth
  independent task; second EQS-authoring instance of the **`asset.list {classNames}`**
  guessability symptom logged on `#6` (and judged accurate / self-correcting /
  not-separately-fileable in `#4`). Struggle-audit of an `eqs.create`
  "FindShootingPositions" EQS-authoring task (outcome clean; the readback elision
  filed separately as `E-eqs-readback-asset-dump-elides-tests #2`, the no-Player-context
  authoring as `E-eqs-context-class-no-project-discovery-path #2`). Mid-task, hunting
  for a project `EnvQueryContext` to satisfy the user's "Player context", the agent
  first ran `asset.list {/Game, recursive, class:EnvQueryContextBlueprint}` (ok, empty)
  then guessed `asset.list {classNames}` ->
  `[UNKNOWN_PARAMS] Unknown parameter(s) for 'asset.list': [classNames]. Valid
  parameters: [path, filter, recursive, pagination, depth].` — identical error and
  valid-param list to `#4`/`#6`, corrected by re-spelling to the nested
  `filter.class`. This is the **second EQS-authoring task** (after `#6`) to prime
  `classNames` from the sibling `asset.search_assets` slot and have `asset.list`
  reject it, reinforcing `#6`'s point that the `classNames`/`filter.class` axis is a
  recurring cross-verb-drift symptom (per `#4`/`#6` disposition: accurate,
  self-correcting, no new defect — logged as corroborating signal that the `asset.*`
  param surface, class-filter spelling included, reads as un-guessable to a fresh
  caller). Friction note (verbatim): "one UNKNOWN_PARAMS (asset.list classNames) …
  [was an] expected probe, not [a] blocker." Recovered on retry, zero blocked progress.
- `#11-fix-asset-readback-path-alias` `IN-REVIEW` developer — Annotated the four
  `asset.*` read-back verbs' `assetPath` slot with a `path` alias via the dispatcher
  `FParamSpec` alias machinery (the same mechanism landed by
  `E-blueprint-param-name-path-vs-assetpath #4` / reused by the DONE material + widget
  fixes), so `{path:...}` now validates at the wire level and is read body-side, ending
  the `path`->`assetPath` round-trip. Added a header-only helper
  `Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AssetPathParamUtils.h`
  (`AssetPathKeys()` = {assetPath, path}, `AssetPathParamReq/Opt` populating
  `FParamSpec.Aliases`), mirroring `MaterialHandlerUtils.h`. Applied it to:
  `asset.exists` (AssetManageHandler.cpp:622), `asset.get` (:647), `asset.validate`
  (:1146) — spec now via `AssetPathParamUtils::AssetPathParamReq`, body reads
  `Ctx.GetStringFirstOf(AssetPathKeys())` instead of `Ctx.GetString("assetPath")`; and
  `asset.dump` (AssetDumpHandler.cpp:1916) — spec aliased, body reads via
  `GetStringFirstOf` with an explicit MISSING_REQUIRED_PARAM guard (no
  `RequireStringFirstOf` helper exists). Scoped to the `asset.*` namespace per the
  ticket title/id; the `niagara.*` widening in `#9` is left for its own ticket, and the
  `classNames`/`filter.class` symptom in `#4`/`#6`/`#10` was judged non-fileable by its
  reporters so it is not touched. Regression test:
  `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestAssetPathParamAlias.cpp`
  — a static registration check that all four read-back verbs' required `assetPath` spec
  carries the `path` alias (covers async `asset.validate`), plus an end-to-end dispatch
  check that the synchronous verbs accept `{path:...}` without a
  MISSING_REQUIRED_PARAM/UNKNOWN_PARAMS rejection. Both fail if the alias annotation is
  reverted. Did not make the canonical param optional (the anti-pattern called out in
  `#3`).
- `#12-additional-metadata-verbs-missed-by-fix` `IN-REVIEW` reporter — Additional
  evidence: the `#11` fix's four-verb scope (`asset.exists`/`asset.get`/`asset.validate`/
  `asset.dump`) **missed the `asset.*` metadata pair**, which live in a different file
  (`AssetMetadataHandler.cpp`, untouched by the fix) and still reject the `path` spelling
  the now-aliased siblings accept. Surfaced by a struggle-audit of a "/Game/Branding logo
  PNG import + metadata tagging" task (seed `asset.import`, outcome clean — the task itself
  succeeded; the agent used `assetPath` correctly). Replay-confirmed against the same live
  asset `/Game/Branding/EAContentExamples57` (a Texture2D): `asset.get_metadata
  {path:"/Game/Branding/EAContentExamples57"}` and `asset.set_metadata {path:...,
  metadata:{...}}` BOTH hard-fail `[MISSING_REQUIRED_PARAM] Missing required parameter
  'assetPath' (type: string)`, while the sibling `asset.get {path:"/Game/Branding/
  EAContentExamples57"}` (fixed in `#11`) SUCCEEDS on the identical `path` arg and returns
  the Texture2D — a direct same-namespace, same-asset asymmetry. Source confirms no `path`
  alias on either: `asset.set_metadata` `RPC_PARAM_REQ("assetPath", ...)`
  (AssetMetadataHandler.cpp:26) and `asset.get_metadata` `RPC_PARAM_REQ("assetPath", ...)`
  (AssetMetadataHandler.cpp:137); `asset.set_tags` (:210) and the other metadata verbs in
  that file (`:287`, `:513`) carry the same bare `assetPath` slot and likely drift too,
  though only get/set_metadata were demonstrated here. The set/get_metadata pair is a
  natural write-then-read chain right after a sibling `asset.get`/`asset.exists` probe, so
  the `path` priming is the same engine-wide caller habit `#3`/`#5` noted. Same dispatcher
  `FParamSpec` alias fix — extend the `#11` `AssetPathParamUtils::AssetPathParamReq`
  treatment (spec alias + `GetStringFirstOf(AssetPathKeys())` body read) to the
  `AssetMetadataHandler.cpp` verbs so the namespace is uniformly aliased.
- `#13-asset-save-new-verb-misses-alias` `OPEN` reporter — Additional evidence: the
  **newly-added `asset.save` verb** (`F-asset-save #3`, IN-REVIEW — landed after the
  `#11` fix's four-verb scope was chosen) also rejects the `path` spelling its now-aliased
  `asset.*` siblings accept, so the same namespace drift recurs on the freshest save path.
  Surfaced by a struggle-audit of a `networking` replicated-pickup build (seed
  `networking.set_property_replicated`, outcome tool_bug — the judge filed the unrelated
  RPC-input type-token bug `B-rpc-input-class-path-silent-wildcard`). To persist the finished
  `/Game/Multiplayer/BP_HealthPickup`, the agent's first save call used the `path` slot:
  `asset.save {path:"/Game/Multiplayer/BP_HealthPickup"}` →
  `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`,
  corrected on retry to `{assetPath:"/Game/Multiplayer/BP_HealthPickup"}` → ok (one
  `path`->`assetPath` round-trip, recovered immediately, zero blocked progress — the only
  param-name is_error in the task). `asset.save` is the natural write-then-persist tail of
  a build-then-save chain, so the `path` priming is the same engine-wide caller habit
  `#3`/`#5`/`#7` noted. Friction note (verbatim): "asset.save rejected 'path' and required
  'assetPath' (one retry)." When `F-asset-save` lands its handler, the same dispatcher
  `FParamSpec` alias treatment (spec alias + `GetStringFirstOf(AssetPathKeys())` body read,
  per `#11`) must be applied to the new `AssetSaveHandler.cpp` so the namespace stays
  uniformly aliased — otherwise the fix's coverage drifts away the moment a new asset verb
  is added.
- `#14-asset-save-replay-animation-task` `OPEN` reporter — Additional evidence (replay,
  same `asset.save` drift as `#13`): an `animation` scaffold task ("clean Animation
  Blueprint scaffold" — create `ABP_HeroLocomotion` + `BS_HeroLocomotion` bound to
  `SK_Mannequin`, add a 3-state locomotion state machine, then save) hit the identical
  `asset.save` `path`->`assetPath` wall, but **twice in a row** before correcting: the
  agent saved both freshly-created assets with the `path` slot first —
  `asset.save {path:".../ABP_HeroLocomotion"}` AND `asset.save {path:".../BS_HeroLocomotion"}`,
  each → `[MISSING_REQUIRED_PARAM] Missing required parameter 'assetPath' (type: string)`
  — then re-issued both with `{assetPath:...}` → ok. So in a single task the missing
  alias cost **two** is_error round-trips (one per saved asset), not one; a multi-asset
  build-then-save tail multiplies the friction by the asset count. Friction note
  (verbatim): "asset.save uses 'assetPath' not the 'path' key every other asset.* verb
  takes (two MISSING_REQUIRED_PARAM errors before I corrected it)." Note the agent's own
  belief that `path` is what "every other asset.* verb takes" — that is exactly the
  `#11`-aliased read-back siblings' behavior priming the wrong spelling on the un-aliased
  save verb, confirming the engine-wide caller habit `#3`/`#5`/`#7`/`#13` named. Same fix:
  apply the `#11` `AssetPathParamUtils::AssetPathParamReq` alias treatment to the new
  `AssetSaveHandler.cpp` when `F-asset-save` lands.
- `#15-asset-delete-reverse-direction-assetpath-rejected` `OPEN` reporter — First
  evidence of this drift in the **reverse direction**, and the first time the
  `path`-declaring side of the namespace (the `asset.list`-style slot this ticket's
  "What it should do" flags for *optional* symmetric aliasing) actually bites a caller:
  here the verb is **`asset.delete`**, which declares `path`/`paths` and rejects
  `assetPath`. Struggle-audit of a `blueprint.compile_bpir` pixel-faithful BPIR
  round-trip task (`/Game/BP_BpirRoundTrip`: seed positioned FirstEvent → upsert a
  positioned SecondEvent → save → decompile and check exact `@(x,y)` coords + exec/data
  wiring; outcome tool_bug — the judge filed the unrelated branch-false-arm reconverge
  drop as `B-bpir-fallthrough-reconverge-dropped`). Mid-task the agent did a
  delete+recreate clean-slate and its delete call reused the spelling **every neighbour
  in the trace uses** — `asset.delete {assetPath:"/Game/BP_BpirRoundTrip"}` →
  `[UNKNOWN_PARAMS] Unknown parameter(s) for 'asset.delete': [assetPath]. Valid
  parameters: [path, paths].`, corrected on retry to `{path:"/Game/BP_BpirRoundTrip"}` →
  `{deletedCount:1, existsAfter:false}` (one round-trip, recovered immediately, zero
  blocked progress; the only param-name is_error in the otherwise-clean 19-RPC task).
  What makes this entry distinct from `#1`–`#14`: (a) the error class is the **mirror
  image** — `UNKNOWN_PARAMS` on a rejected `assetPath` key, not `MISSING_REQUIRED_PARAM`
  on a missing `assetPath` — proving the drift is bidirectional, not just "callers say
  `path`, `assetPath`-verbs reject"; (b) the priming source is stronger than usual —
  `blueprint.create` had **returned** `assetPath:"/Game/BP_BpirRoundTrip"` to the agent
  one call earlier, and `asset.save`/`asset.validate`/`blueprint.create`/`compile_bpir`/
  `decompile`/`graph.get_graph_connections` all accept/return `assetPath` in this same
  trace, so `asset.delete` is the lone asset/blueprint verb the agent touched that
  rejects it. Source confirms `asset.delete` is on the `path`-declaring side with **no
  `assetPath` alias**: `AssetManageHandler.cpp:495` `REGISTER_RPC_HANDLER("asset.delete",...)`,
  `:497` `RPC_PARAM_OPT("path", ...)`, `:498` `RPC_PARAM_OPT("paths", "array", ...)` —
  the same shape as `asset.list` (:690) that the `#11` fix deliberately scoped *out*
  (it annotated the four `assetPath`-slot read-back verbs with a `path` alias, the
  `assetPath→path` direction only). So the `#11` fix does **not** cover this; closing it
  requires the symmetric half from "What it should do": annotate `asset.delete`'s
  `path`/`paths` slots (and `asset.list`'s `path`) with an `assetPath` alias + a
  `GetStringFirstOf` body read, so the spelling is interchangeable in both directions
  across the namespace. Precedent already exists one file over: `asset.create_folder`
  (:579-580) carries a `directoryPath` alias on its `path` slot, so the alias-on-path
  pattern is established. Friction note (verbatim): "asset.delete rejects `assetPath`,
  wants `path`." CallAnalyzer flagged the same call as a `frustrating` E/low. Severity
  stays Low (self-correcting, accurate error, zero blocked progress), but this is the
  data point that promotes the deferred "optional symmetry" aliasing to a real,
  caller-observed need on `asset.delete`.
