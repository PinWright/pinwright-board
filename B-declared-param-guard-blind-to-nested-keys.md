---
id: B-declared-param-guard-blind-to-nested-keys
title: "Keys nested inside an object/array parameter are validated by nothing and checked by no test in either direction: 325 of 1218 verbs carry the surface, only 2 close it, and five hand-confirmed verbs accept a documented nested key and silently discard it"
status: OPEN
severity: High
category: bug
tags: [dispatcher, unknown-params, undeclared-parameter, param-spec, nested-object, schema, silent-drop, test-coverage, guard-blind-spot, response-honesty]
encounters: 1
lastSeen: 2026-08-29
---

# The whole mechanism stops one level above where half the payload lives

`PinWright.infra.declared_params.HandlersOnlyReadDeclaredParams`
(`Source/PinWright/Private/Tests/Infra/TestDeclaredParamCoverage.cpp`) catches a verb that READS a
wire key its `RPC_PARAMS` never DECLARES. For a key nested inside an object- or array-typed
parameter **neither direction of that guard can see anything** — not reads-without-declarations, and
not declarations-without-reads — and the dispatcher does not validate nested keys either. An
accepted-but-inert nested key is invisible to the entire mechanism: no gate rejects it, no test
names it, and the verb answers `success`.

Two sibling tickets cover the *top-level* blind spots: `B-declared-param-guard-blind-spots`
(IN-REVIEW, four in-body read shapes) and `B-declared-param-guard-blind-to-helpers` (OPEN, High, 444
pairs one call frame out). This is the third, and it is a different mechanism with a different fix —
see [Why this is a new ticket](#why-this-is-a-new-ticket-and-not-an-entry-on-the-helpers-ticket).

## The exclusion is deliberate, documented, asserted — and correct

Three places in the tree, all verified first-hand:

1. **Stated.** `TestDeclaredParamCoverage.cpp:64-67`, KNOWN LIMITS: *"A raw-payload read off a NESTED
   object (`Payload->GetObjectField(...)` then reading that) is out of scope by design — the nested
   keys are not top-level wire params. Only reads whose receiver IS the payload are counted."*
2. **Enforced.** `CollectRawPayloadKeys` (`:269-316`) collects a member read only when its receiver
   is in `Bound`, the set of locals assigned directly from `Ctx.GetRawPayload()`. A nested
   sub-object binds a different variable and drops out — a one-variable taint, deliberately not a
   call graph (`:197-200`).
3. **Pinned by a passing assertion.** `FDeclaredParamScannerShapesTest` (`:685-688`) asserts BOTH
   directions: `probe_raw_nested_owner` (the top-level key *naming* the nested object) IS collected,
   and `probe_raw_nested` (a key read off that object) is NOT. Widening the scanner means deleting a
   green test on purpose.

**The reason is not laziness, it is that the comparison would be a category error.** The right-hand
side of the diff is `ParamSpecTestHelpers::CollectAcceptedParamNames(Method)` — the same set
`FRpcDispatcher::ValidateHandlerParams` builds from `Spec.Name` / `Spec.Aliases` /
`Spec.TypedAliases[].Name`. That gate iterates `for (const auto& Field : Params->Values)`
(`Dispatch/RpcDispatcher.cpp:135-141`) — strictly one level, never descending into an object or
array value. A nested key lives in a different namespace from that set, so diffing them reports
every nested key as an undeclared parameter. This is not hypothetical: the helpers sweep recorded
that a "call argument mentions the payload" rule counted nested reads as top-level keys and produced
**15 false `x`/`y`/`z`/`pitch`/`roll`/`yaw` pairs on `level.structure.*`** before the rule was
tightened. **Do not remove the exclusion.**

What the file does *not* say, and should, is the consequence: because the dispatcher never validates
a nested key either, the exclusion is not merely a scanner limit — it is the point at which the
plugin stops checking caller input at all.

## Measured surface

**Method.** A replica scanner over `Plugins/PinWright/Source`, `.cpp` only, `/Tests/` excluded — the
same file set, the same comment/raw-string neutralizer, the same brace matcher and the same
`REGISTER_RPC_HANDLER` recovery as the shipped guard. **1218 registrations recovered.** Declaration
side is over-accepting on purpose: every string literal in the macro args plus the literals of every
macro or function named there, transitively to depth 3, which resolves the alias factories and the
object-like param macros.

**Calibration against the shipped guard's live baseline.** Restricted to the guard's four in-body
read shapes, the replica returns **exactly 8 pairs, all `environment.build`, identical to
`KnownUndeclaredReads()`** — zero extras, zero omissions. The registration and declaration parsers
are therefore adequate on this tree, which is what licenses the nested numbers built on them. (1218
vs the 1217 recorded by `B-declared-param-guard-blind-to-helpers`: one registration has landed
since.)

| measure | value |
|---|---|
| registered verbs | 1218 |
| verbs declaring ≥1 parameter whose declared type contains `object` or `array` | **325 (26.7%)** |
| verbs reading ≥1 nested key **inside the handler body** | **53** |
| (verb, nested key) pairs read in-body | **245** over 131 distinct key names |
| verbs whose nested objects reject unknown keys | **2 families** (see Fix) |
| top-level declared names read nowhere in `Source` | **1** (see the other direction) |

Tree-wide declared-type histogram: 400 `object`, 157 `array`, 22 `any`/mixed (`array|object`,
`bool|object`, `string|array|object`, …).

The 53 in-body readers, by namespace: `landscape` 6, `blueprint` 4, `lighting` 4, `material` 4,
`niagara` 4, `spatial` 4, `animation` 3, `volume` 3, `audio` 2, `image` 2, `render` 2, `skeleton` 2,
and 13 namespaces with 1 each.

**The other 272 read their nested keys one further call frame out, so they are doubly invisible** —
nested *and* behind a helper. Two sub-shapes, and the second is the larger:
* **Inside the accessor.** `Ctx.GetVector(TEXT("location"))` → `ExtractVectorField` →
  `ReadVectorFieldImpl` (`Utils/JsonUtils.cpp:16-49`) reads `x`/`y`/`z`, *and* the undocumented
  `X`/`Y`/`Z` capitals, *and* a bare `[x,y,z]` array form. One contract, ~150 verbs, uniform — and
  two thirds of it undocumented.
* **Inside a per-verb helper.** `ParseGeoreference`, `ParseSamplingRange`, `ResolveWidgetInsertIndex`,
  `ReadStructFieldSpec`, `ParseNamedTypePinParams`, `WriteLayeredBlendLayers`, … — confirmed by hand
  on 9 of the sampled groups.

**Error directions, stated.**
* **245 is a floor.** It counts only literal keys read off a variable the taint can tie back to
  `Ctx.GetObject/GetArray/RequireObject/RequireArray` or to a nested field of the raw payload, inside
  the macro body. Every helper-side nested read is excluded, as is every runtime-assembled key.
* **It can over-count in one direction:** a body reading a nested JSON object that did not come from
  the wire (a parsed file, a response object being re-read) would be attributed. The sampled subset
  showed none, but all 53 were not audited.
* **325 over-counts slightly the other way:** some `object`-typed parameters are opaque values
  forwarded to a property serializer (`property.set`'s `value`), where the "nested keys" are struct
  field names rather than a fixed wire schema. Those still receive zero validation, but "documented
  nested key" is not a meaningful notion for them.

## Confirmed instances — this is a defect ticket, not a latent-risk ticket

22 verb/parameter groups across 12 namespaces, ~50 distinct nested keys, hand-checked against source
by four independent auditors. Each key was classified READ-IN-BODY / READ-IN-HELPER / NOT-READ, with
a whole-tree grep for the literal before any NOT-READ verdict.

**Five verbs confirmed accepting a nested key and discarding it silently:**

1. **`animation.create_state_machine` — `states[].animation`, `states[].isExit`.** The parameter
   description (`Handlers/Animation/AnimationHandler.cpp:634`) promises `{name, animation, isEntry,
   isExit}`. The body loop (`:696-727`) reads `name` (`:703`) and `isEntry` (`:723`) and nothing
   else; `CreateState(SMGraph, FName, FVector2D)` at `:709` takes no JSON. The literal `isExit`
   occurs **nowhere in `Source`** except that description string. A caller sending
   `{"name":"Idle","animation":"/Game/Anims/A_Idle"}` — exactly what the description advertises —
   gets `statesCreated: N` and an empty state with no bound player node. The sibling
   `transitions[].condition` at least emits a `warning` (`:821`); this emits nothing. **This is the
   clean instance: a documented nested key, a live verb, a silent false success.**
2. **`material.authoring.create_material_instance` — anything under `parameters` that is not one of
   the four bucket names.** `parameters` is probed only for `scalar` / `vector` / `texture` /
   `staticSwitch` (`Handlers/Material/MaterialInstanceOverrides.h:66,84,109,135`). A payload
   `{"parameters":{"r":1,"g":0,"b":0}}` — the correct shape one level too shallow — or any typo'd
   bucket (`vectors`, `Scalar`, `static_switch`) passes the top-level gate, matches nothing, and
   returns `success` with `applied:[]` **and** `failed:[]`. Two empty arrays are the only evidence
   the caller gets, and nothing says which key was wrong.
3. **`render.capture_annotated` — `grid.between`, `grid.lines`, `grid.origin`.** None of the three
   literals exists anywhere in `Source`; only `spacing` (`Handlers/Render/AnnotatedCaptureHandler.cpp:239`)
   and `extent` (`:240`) are read. They are invented by reading the parameter description, which is
   written as a JSON-shaped brace containing English — `"{ spacing: cm between lines, extent: cm
   half-size from origin }"` (`:207`) — and is mirrored verbatim into `docs/wiki-src/render.md:347`.
   Weaker than #1 as an *intended* promise, and I record it as such; the caller-visible behaviour is
   identical, and `overlays.grid.spacing: 100` is echoed back as if it were the caller's request.
4. **`blueprint.add_function` / `networking.create_rpc_function` — string elements in
   `inputs[]`/`outputs[]`.** `ParseNamedTypePinParams` (`Handlers/Blueprint/BlueprintHandlerUtils.cpp:206-215`)
   does a bare `continue` on any array element that is not a JSON object, so
   `inputs: ["Damage","Target"]` produces zero pins and a success response.
5. **`eqs.set_test_filter` — `filter.min` / `filter.max` off the match branch.**
   `Handlers/AI/EQSHandler.cpp:647-663` reads `value` under `kind:'match'`/`'bool'` and `min`/`max`
   only in the `else`, so `{kind:'bool', value:true, min:1, max:2}` succeeds with both numbers
   discarded. Under `kind:'minimum'` the supplied `max` IS written into `FloatValueMax` (`:659-662`),
   which `EEnvTestFilterType::Minimum` ignores at query-eval time — a write that lands in the asset
   and changes no behaviour.

**Candidates rejected on inspection, recorded so the number stays honest:**
`material.authoring.set_material_instance_parameters`'s `ParamName` / `TextureAssetPath` are
**metavariables** in the description, not literal keys — the maps are keyed by caller-chosen
parameter names, and the extractor that surfaced them was wrong. Everything else in the sample was
READ-IN-HELPER and therefore correct-but-unguarded: the `image.*` `georeference`/`axes`/world-point
tree, `pose_search` `samplingRange`, `insights.export_trace` `windows`, `widget.{add,duplicate,
reparent_widget}` `placement`, `skeleton.set_vertex_weights` `weights[]`,
`blueprint.add_struct_field` `flags`, `blueprint.graph.find_node_types` `contextPins`,
`animation.authoring.set_layered_blend_layers` `layers[].branchFilters[]`, `actor.spawn_batch`
`transforms[]`.

**The inverse direction showed up in the same sample** — nested keys read but promised nowhere:
`pose_search`'s `samplingRange.startTime`/`endTime` aliases (`PoseSearchHandler.cpp:201-202`), and
`ReadVectorFieldImpl`'s capitalised `X`/`Y`/`Z` and bare 3-element array form (`JsonUtils.cpp:31-44`),
which every one of the ~150 vector-taking verbs silently accepts.

Two adjacent findings from the same audit, same class, different rung: `blendDepth` is read but
never validated in `WriteLayeredBlendLayers`' validation pass (`AnimGraphConstructionUtils.cpp:573-600`
validates only `boneName`), so a string or missing value silently becomes `0` at `:653-657`; and
`animation.authoring.add_modify_bone`'s quaternion `rotation.w` is inert whenever `pitch`/`yaw`/`roll`
is also present, because the Euler branch (`AnimationAuthoringHelpers.cpp:226`) wins and returns
first.

## The declarations-without-reads direction, assessed separately

**Measured at the top level, and it is nearly clean.** Method: blank every `RPC_PARAM_*( … )` span
tree-wide, then ask whether each declared name still occurs as a string literal anywhere in `Source`.
A name that does not is read by *nothing* — not the body, not a helper, not a key-list factory — so
every false-positive class the class is usually accused of (a read on one branch only, a read
consumed by a shared helper, a key assembled at runtime from a literal factory) is structurally
suppressed. Error direction: **false negatives only.**

**Result: 1 hit across 1218 verbs** — `gas.set_ability_input:abilitySetPath`
(`Handlers/Systems/GASHandler.cpp:2735`). And it is the benign class the brief predicted: the
description says *"only honored when bindOn=ability_set"*, and `bindOn == "ability_set"` returns
`NOT_IMPLEMENTED` at `:2746` before anything could read it. The parameter is unreachable by
construction and honestly labelled.

**Assessment.** At the top level this check is worth having and cheap — ~20 lines, one baseline
entry, no toolchain, and it composes with the existing scan because it reuses the same declaration
side. Its price is that the strict formulation catches only names mentioned *nowhere*; a parameter
that is mentioned but not *used* slips through, and tightening it to per-verb scope re-introduces
exactly the helper-resolution problem `B-declared-param-guard-blind-to-helpers` exists to solve. It
is a small win: file it as step 3 below, not as a reason to open a fourth ticket.

**Nested, the same direction cannot be measured this way at all**, and that is the asymmetry worth
naming: a nested key has no declaration to diff against, so there is no set to subtract from. The
five instances above were found by reading parameter *descriptions* as the de facto schema — which
is what a caller and an agent both do, and which is why the fix has to make that schema real rather
than make the test smarter.

## Fix

Four options, priced; the recommendation is **C then B', and explicitly not A**.

**A. Extend the scanner to walk into nested reads and diff them against `RPC_PARAMS`. Rejected —
it is wrong, not merely expensive.** All 245 in-body nested pairs would be reported as undeclared
parameters, because they are not top-level wire names. The failure mode is on record (15 false
`level.structure.*` pairs), and the exclusion is asserted by a passing test precisely so nobody does
this by accident.

**C. Close the nested objects at runtime, reusing what the tree already has. — DO THIS FIRST.**
`RejectUnknownKeys` already exists **three times over**: `Handlers/Image/ImageOps.cpp:807`
(declared `ImageOps.h:196`), `AudioGen/PwMusicScore.cpp:738`, `AudioGen/PwSynthRecipe.cpp:786`. It is
applied to `georeference` (`ImageOps.cpp:399`), `axes` (`:318`), every world point (`:767`), `tile`
and `georeferenceFromTile` (`ImageTileHandler.cpp:139`, `ImageAnnotateHandler.cpp:199`),
`image.annotate`'s own `grid` (`:336`), and to every object in the music-score and synth-recipe
schemas. **So exactly two verb families out of 325 close their nested objects** — `image.tile` /
`image.annotate`, and the `audio.music.*` / `audio.synth.*` schema takers — and the other ~319 do
not. The same mistake is a typed `INVALID_PARAMS` naming the offending key on `image.annotate`'s
`grid` and total silence on `render.capture_annotated`'s. The mechanism is proven in-tree; it is
duplicated and unenforced.
   1. Promote one copy to a shared utility (`Utils/JsonUtils.h`) and point the three call sites at
      it. ~40 lines, no behaviour change.
   2. Adopt it on the five confirmed verbs above, each with its allow-list beside the reads. ~5-10
      lines per verb.
   3. **Adopt per verb, never as a blanket sweep.** Closing an object is a compatibility break: a
      caller sending a stray nested key gets `success` today and `INVALID_PARAMS` afterwards. That
      is the right direction, but it lands one verb at a time with the doc updated in the same
      commit — not across 319 verbs in one go.

**B. A nested-schema declaration beside `RPC_PARAMS`, diffed against the nested reads by a widened
guard. Rejected as the first move, for the reason the helpers ticket rejected its analogue:** a
declaration for ~325 verbs drifts from the body exactly the way `RPC_PARAMS` drifts today, and
keeping it honest requires the scan, at which point the declaration is redundant. It becomes
worthwhile only *after* C, when the allow-list already exists in code and the macro would merely
render it — i.e. as documentation generation, not as a second source of truth.

**B'. The cheap guard that does have no false-positive class, once C exists.** With allow-lists in
code, assert that every nested key a verb's parameter *description* promises (backticked, or listed
inside a `{...}`) appears in that verb's allow-list. That is direction-2 for nested keys, needs no
schema macro and no call graph, and its baseline today is exactly the five confirmed instances.
Without C it needs the one-hop call closure `B-declared-param-guard-blind-to-helpers` prices at ~250
lines / <15 s, because the promised key is usually read in a helper — which is a reason to sequence
this after that ticket's step 1, not a reason to merge the two.

**D. Accept and document. Rejected as a whole — but its documentation half is mandatory regardless.**
`TestDeclaredParamCoverage.cpp:64-67` must say more than "out of scope by design": it must say that
the dispatcher does not validate nested keys either (`RpcDispatcher.cpp:135-141` walks
`Params->Values` only), that the exclusion therefore leaves nested input unchecked by anything, and
that the reads-without-declarations direction must NOT be widened into it. Add the
declarations-without-reads gap to the same block — the file never mentions that it only ever checks
one of the two directions.

**Step 3 (independent, cheap): ship the top-level declarations-without-reads assertion** described
above, with `gas.set_ability_input:abilitySetPath` as its single baseline entry and a comment saying
why it is exempt.

**Workaround (per verb, not for the class):** none for `states[].animation` — build the state
machine, then populate each state with `animation.authoring.add_graph_node` /
`bind_player_asset`. For `material.authoring.create_material_instance`, check that `applied[]` is
non-empty; a success with two empty arrays means every key was discarded. There is no general
workaround, because the failure is indistinguishable from success at the wire.

## Why this is a new ticket and not an entry on the helpers ticket

`B-declared-param-guard-blind-to-helpers` is OPEN and unstarted, so appending would have been
defensible on lifecycle grounds. It is filed separately on technical grounds, which are stronger:

* **Its Fix would not touch this.** Steps 1-6 there widen the scanner across a call frame while
  still diffing against the top-level accepted-name set. Perfect helper resolution still sees
  nothing here: `Obj->GetStringField("animation")`, where `Obj` came from `states[i]->AsObject()`,
  is not a top-level wire key and must not be compared to `RPC_PARAMS`.
* **The two pull the same code in opposite directions.** That ticket wants the scanner to follow
  more reads; this one says the nested reads it must *not* follow are exactly the ones it already
  correctly drops — and records that its own measurement was corrupted once by blurring the two
  (the 15 false `level.structure.*` pairs). Merging them makes the implementer's brief ambiguous
  about which reads to chase.
* **Different defect, different remedy.** There, the code is unreachable and the fix is an
  `RPC_PARAM_OPT` line. Here, the code runs, the caller's input is dropped, and the fix is a runtime
  allow-list plus a doc — nothing to declare in `RPC_PARAMS` at all.

**Cross-references.** `B-declared-param-guard-blind-spots` (IN-REVIEW) closed the four in-body
shapes. `B-declared-param-guard-blind-to-helpers` (OPEN, High) is the call-frame hop; its step 1
closure is what B' above would ride on. `B-test-invokehandler-bypasses-param-gate` (IN-REVIEW) is
why per-verb tests cannot see any of this. `B-actor-list-fields-unknown-key-silently-dropped` (DONE,
High) is a single-verb instance of this exact class — a list-valued key silently dropped — and was
banded High. `E-foliage-nested-input-schemas-undocumented` (IN-REVIEW, Low) documents nested shapes
for two foliage verbs; it is the doc half of the same gap, for two verbs out of 325.
`B-create-procedural-ignores-scale-and-normal-fields` (IN-REVIEW) is where this blind spot was first
named: its nested `foliageTypes[].minScale`/`maxScale`/`alignToNormal` were unreachable to both
directions of the guard. That verb's fix landed; the class did not.

severity rationale: impact=High — five confirmed live verbs answer `success` while discarding a
nested input, and `animation.create_state_machine` discards a documented asset path and reports
`statesCreated: N`, which is the rubric's "silent false-success … the caller trusts a result that is
a lie and builds on it"; the caller cannot distinguish it from a correct call at the wire × reach=
normal, 325 of 1218 verbs (27%) carry the unvalidated surface and the confirmed set spans everyday
material, animation and blueprint authoring rather than an edge path, so no modifier applies -> High.
Banded with `B-declared-param-guard-blind-to-helpers` and `B-verbs-read-undeclared-parameters`, and
with the single-verb precedent `B-actor-list-fields-unknown-key-silently-dropped`.

## Not RPC-verified

Source-read only; the editor was not driven and no verb was called. A full automation suite was
running against the compiled DLL throughout, so nothing was built and no test was executed. Every
line and count above comes from reading the tree at the current working state. Two things a live
pass would settle: whether `animation.create_state_machine` with `states[].animation` really returns
an unqualified success (predicted from source, not observed), and whether
`material.authoring.create_material_instance` with a mis-nested `parameters` really returns
`applied:[]`/`failed:[]` alongside `success`. Neither changes the fix.

## History
- `#1-nested-keys-unguarded-both-directions` `OPEN` reporter — Source-read only; no editor, no
  build, no test run (a suite was live against the DLL). **Exclusion verified in three places:**
  KNOWN LIMITS `TestDeclaredParamCoverage.cpp:64-67`, the receiver check in `CollectRawPayloadKeys`
  (`:269-316`), and the paired assertions in `FDeclaredParamScannerShapesTest` (`:685-688`) that pin
  both `probe_raw_nested_owner` IS collected and `probe_raw_nested` is NOT. Its reason is sound and
  was confirmed at the source rather than taken from the comment: `ValidateHandlerParams` walks
  `for (const auto& Field : Params->Values)` (`Dispatch/RpcDispatcher.cpp:135-141`), strictly one
  level, so the accepted-name set the guard diffs against contains no nested names and diffing them
  would report every nested key as undeclared — the failure the helpers sweep already hit as 15
  false `level.structure.*` pairs. **Method:** replica scanner over the same file set with the same
  neutralizer/brace matcher/registration recovery; 1218 registrations; declaration side
  over-accepting to depth 3. **Calibrated** against the shipped guard's four in-body shapes to
  exactly its live baseline — 8 pairs, all `environment.build`, matching `KnownUndeclaredReads()`
  with zero extras and zero omissions. **Surface: 325 of 1218 verbs (26.7%) declare an object/array
  parameter; 53 read 245 (verb, nested key) pairs over 131 distinct names inside the handler body;
  the other 272 read theirs inside a helper or inside `ReadVectorFieldImpl`, so they are nested AND
  behind a call frame.** 245 is a floor (helper-side and runtime-assembled nested keys excluded);
  it can over-count if a body reads non-wire JSON; 325 over-counts where an `object` parameter is an
  opaque serializer payload. **Confirmed instances, not latent risk:** four auditors hand-checked 22
  verb/parameter groups across 12 namespaces (~50 keys) with a whole-tree literal grep before any
  NOT-READ verdict, and found five verbs accepting a nested key and discarding it silently —
  `animation.create_state_machine:states[].animation`/`isExit` (description `AnimationHandler.cpp:634`
  promises both; `isExit` occurs nowhere else in `Source`), `material.authoring.create_material_instance`
  (any key under `parameters` outside the four buckets → `success` with `applied:[]` and `failed:[]`),
  `render.capture_annotated:grid.{between,lines,origin}` (invented from a brace of English prose at
  `AnnotatedCaptureHandler.cpp:207`, mirrored into `render.md:347`), `blueprint.add_function` /
  `networking.create_rpc_function` (string elements in `inputs[]` `continue`d at
  `BlueprintHandlerUtils.cpp:206-215`), and `eqs.set_test_filter:filter.{min,max}` off the match
  branch. One candidate was rejected on inspection and is recorded as such
  (`set_material_instance_parameters`'s `ParamName`/`TextureAssetPath` are metavariables, not keys).
  **Second direction measured separately:** blanking every `RPC_PARAM_*` span tree-wide and asking
  whether a declared name occurs as a literal anywhere in `Source` yields **1 hit in 1218 verbs**,
  `gas.set_ability_input:abilitySetPath`, which is honest (its mode returns `NOT_IMPLEMENTED` at
  `GASHandler.cpp:2746`) — so the top-level half of that direction is worth a ~20-line assertion
  with a one-entry baseline, while the nested half cannot be posed at all for want of a declaration
  to subtract from. **Fix chosen:** promote the already-existing `RejectUnknownKeys`
  (`ImageOps.cpp:807`, duplicated at `PwMusicScore.cpp:738` and `PwSynthRecipe.cpp:786`; today
  applied by only 2 of 325 verb families) to a shared utility and adopt it per verb on the confirmed
  five, then add the description-promises-vs-allow-list assertion; explicitly NOT widening the
  scanner into nested reads. Dedup: grepped the board for `declared_param`, `nested`,
  `UNKNOWN_PARAMS`, `schema`, `RPC_PARAMS`; filed separately from the OPEN, unstarted
  `B-declared-param-guard-blind-to-helpers` because that ticket's Fix (one-hop call resolution still
  diffed against the top-level accepted-name set) cannot see a nested key even when perfect, and
  because the two ask the scanner for opposite things about the same reads.
