---
id: B-declared-param-guard-blind-spots
title: "The declared-param guard matches one read shape out of four, so its now-empty baseline reads as 'class eradicated' while ~140 undeclared (verb, param) pairs remain — including 11 verbs whose own param description promises names_only"
status: IN-REVIEW
severity: High
category: bug
tags: [dispatcher, unknown-params, undeclared-parameter, param-spec, unreachable-code, test-coverage, guard-blind-spot, alias, sweep]
encounters: 1
lastSeen: 2026-08-28
---

# `HandlersOnlyReadDeclaredParams` is blind to three of the four ways a handler reads a wire key

`PinWright.infra.declared_params.HandlersOnlyReadDeclaredParams`
(`Source/PinWright/Private/Tests/Infra/TestDeclaredParamCoverage.cpp`) ratcheted
`B-verbs-read-undeclared-parameters` from 66 pairs to an empty `KnownUndeclaredReads()` baseline
(`:284`). An empty baseline plus a green test reads as **the class is gone**. It is not: the guard
measures one read shape, and the same defect is alive in the other three at roughly **twice the
volume of the original sweep**.

## The two instances the previous pass named — both still live

**1. `actor.set_collision` — `collision_enabled`, read off the raw payload.**
`Source/PinWright/Private/Handlers/Actor/ActorPropertyHandler.cpp:182-198`. The declaration is
`RPC_PARAM_OPT("collisionEnabled", "boolean", "... Snake_case alias collision_enabled also
accepted.")` (`:187`) — no alias list. The body reads
`Payload->HasField(TEXT("collision_enabled"))` / `GetJsonBoolField(Payload, TEXT("collision_enabled"))`
(`:196-197`) off `Ctx.GetRawPayload()` (`:193`). The description promises a spelling the dispatcher
refuses with `UNKNOWN_PARAMS` before the body runs. Note the sibling parameter on the same
registration *was* fixed by the 21-file sweep (`actorName`/`actor_name` via
`ParamAliasUtils::MakeAliasParamSpec`, `:184-186`) — the scanner saw that read and not this one.

**2. `behavior_tree.attach_decorator` / `attach_service` — `behaviorTreePath`, `path`, read inside a
shared helper.** `Source/PinWright/Private/Handlers/AI/BehaviorTreeHandler.cpp`. Both registrations
(`:854`, `:868`) declare a bare `RPC_PARAM_REQ("assetPath", ...)` (`:856`, `:870`) and delegate to
`HandleAttachBTSubNode` (`:395`), whose first statement is
`Ctx.GetStringFirstOf({TEXT("assetPath"), TEXT("behaviorTreePath"), TEXT("path")})` (`:397`). Their
five siblings — `add_node` (`:666`), `connect_nodes` (`:884`), `remove_node` (`:955`),
`break_connections` (`:988`), `set_node_properties` (`:1021`) — declare the same three spellings
through the `BTAssetPathParamReq()` factory (`:64`) and accept them. Two verbs in one namespace
reject an alias their five siblings take, and the guard cannot see the read because it is one call
frame away.

Neither was fixed by this session's sweep.

## What the guard actually matches

`ScanFile` locates each `REGISTER_RPC_HANDLER(` whose **first macro argument is a literal dotted
string** (`MethodPattern`, `:206`), brace-matches the handler body, and runs exactly one regex over
it (`ReadSitePattern`, `:135-141`):

```
\bCtx\s*\.\s*(<19 accessor names>)\s*\(\s*(\{[^}]*\}|TEXT\s*\(\s*"[^"]*"\s*\)|"[^"]*")
```

Three conditions must all hold for a read to be seen: the receiver is literally spelled `Ctx`; the
member is one of nineteen enumerated names (`GetString|GetNumber|GetBool|GetInt|GetVector|GetRotator|
GetObject|GetArray|GetStringSet|GetStringFirstOf|GetBoolFirstOf|GetIntFirstOf|RequireString|
RequireAssetPath|RequireInt|RequireNumber|RequireBool|RequireObject|RequireArray`); and the first
argument is a string literal or a `{...}` list containing no nested `}`. Anything else is a
**false negative** — a silent pass, indistinguishable from a clean verb.

### Blind spot 1 — key-taking accessors missing from the enumerated list

`FHandlerContext` (`Handlers/HandlerContext.h`) exposes two more key-taking accessors that the
pattern never names:

- `GetJsonValueFirstOf(const TArray<FString>& Keys)` (`HandlerContext.h:113`, impl `.cpp:297`).
- `ReadFieldProjection(const TArray<FString>& NamesOnlyKeys)` (`HandlerContext.h:131`, impl
  `.cpp:329`) — it reads four wire keys of its own that appear at **no** call site:
  `fields`, `field` (`.cpp:345,358`) and `namesOnly`, `names_only` (`.cpp:366`).

28 in-body call sites, **≥16 undeclared pairs**, and the cluster is self-indicting:

| verbs | undeclared key | their own description says |
|---|---|---|
| `asset.list`, `asset.search`, `skeleton.list_bones`, `skeleton.list_physics_bodies`, `gameplay_tags.list`, `system.console.search`, `material.graph.list_expression_types`, `blueprint.graph.{get_nodes,get_graph_details,get_node_details_batch}`, `audio.synth.list_candidates` (11) | `names_only` | "Snake_case names_only accepted." — verbatim, in the `namesOnly` `RPC_PARAM_OPT` on every one of them |
| `sequencer.set_playhead` | `frameNumber`, `frame_number`, `seconds`, `timeSeconds`, `time_seconds` | the `time` param reads "Target position in seconds (alias: seconds)" (`Sequencer/SequenceHandler.cpp:1988`); the reads are at `:2010` and `:2015` |

`sequencer.set_playhead` is the sharpest illustration: the previous pass fixed its six *baseline*
pairs (`sequencePath`, `openIfNeeded`, `force_update`, `update_method`, …) on this exact
registration and walked past five more on the two lines below, because those go through
`GetJsonValueFirstOf`. The singular `field` shorthand is undeclared too — `asset.list` declares
`fields` only (`Asset/AssetManageHandler.cpp:937`) — and is not in the ≥16.

### Blind spot 2 — reads off `Ctx.GetRawPayload()`

Documented in the file's KNOWN LIMITS as a false-negative direction, unquantified. **414 of 1,212
handler bodies (34%) touch `GetRawPayload()`.** Tracking only the variable a body binds directly
from it (so nested sub-object reads are excluded) yields **36 undeclared pairs**; 7 of those are
`environment.build:*` already covered by `B-environment-build-dispatcher-rejects-forwarded-params`,
leaving **29 new**. Verified clusters:

- `level.*` — 8 pairs in `Handlers/Level/LevelHandler.cpp`, every one a fallback spelling read after
  the declared one: `add_sublevel:levelPath` (declares `subLevelPath`, `:956`; reads `:970`),
  `delete:path` (`:1053`), `duplicate:levelPath` (`:1143`), `export:destinationPath` (`:883`),
  `rename:sourcePath` (`:1094`), and `level_path` on `set_visibility` (`:1297`), `set_locked`
  (`:1341`), `remove_from_world` (`:1383`).
- `effect.draw_debug_shape` — 7 pairs in `Handlers/VFX/EffectHandler.cpp:201-212`. The declaration
  covers `preset/shapeType/location/rotation/scale/color/duration/size/thickness`; the body reads
  `autoDestroy`, `length`, `angle`, `halfHeight`, `endLocation`, `direction`, `boxSize`. **This one
  has no workaround** — the per-shape geometry of the line/arrow/cone/capsule shapes the summary
  advertises is unreachable through any spelling.
- `landscape.create` — 5 pairs (`Handlers/Environment/LandscapeHandler.cpp:387-396` declares
  `name/location/componentsX/componentsY/quadsPerComponent/sectionsPerComponent/materialPath`; the
  body reads `landscapeName`, `sizeX`, `sizeY`, `componentCount`, `quadsPerSection`).
- `actor.set_collision:collision_enabled`, plus singles on `asset.get_dependencies_classified`,
  `blueprint.{list,remove_function,set_function_settings}`, `foliage.add_instances`,
  `system.console_command`.

### Blind spot 3 — reads inside a shared `FHandlerContext&` helper

Named in KNOWN LIMITS, deferred by `B-verbs-read-undeclared-parameters` as "~88 medium-confidence".
An independent pass here (collect key literals from every free function taking `FHandlerContext&`,
attribute by bare-name call in the handler body) finds **80 key-reading helpers and 95 pairs across
31 verbs** — same order, so the estimate is stable under two methods. Both methods resolve callees
by bare name and both are noisy in the same direction: `attach_decorator:serviceClass` is a false
attribution (the shared helper branches on `bDecorator`), while `attach_decorator:behaviorTreePath`
is real. Call it **~60-95 real**, concentrated in `drive.*` (`browser_index`, `surface`,
`timeout_ms`, `mark_cap`, `full_diff`), `ui.activatable_*` / `ui.get_active_widget` /
`ui.list_stack_widgets` (`host`, `layerTag`, `playerIndex`, `stack`), `editor.{frame_graph,
resize_window,screenshot_window}` (`window_index`/`window_title` — see
`B-drive-window-selector-param-unreachable`), `eqs.*` / `ai.*`, `geometry.{bend,taper,twist}`,
and the four `audio.authoring.*` MetaSound verbs.

### Blind spot 4 — keys that are not literals at the call site

**53 sites across 51 verbs** call a matched accessor with a non-literal first argument
(`Ctx.GetString(SomeVar)`). These are not merely unmeasured — the key is **unknowable from source
text at all**, so no widening of a source scanner can ever cover them.

### Blind spot 5 — structural skips

`MethodPattern` (`:206`) requires a literal dotted method name, so a registration made through
`REGISTER_RPC_HANDLER_INNER` with a macro-parameter method name is dropped silently (none exist
today — `HandlerRegistration.h:39-48` — but nothing prevents one). A method present in source but
absent from the live registry is skipped by design (`:368`), so a host with an integration
sub-module's engine plugin disabled measures nothing for that sub-module and still reports a pass.
The vacuity guard (`Verbs.Num() >= RegisteredCount / 2`) only catches a *total* parser failure, not
a shape that quietly stops matching.

## Scale

Reproduced offline against current source; the declaration side is parsed from text and deliberately
over-accepts (literals in the `RPC_PARAMS` region plus two levels of called-function literals), so
every count below is a **lower bound**. Calibration: run against the shape the shipped guard
matches, the same harness returns **0 pairs** — it agrees with the live empty baseline exactly.

| shape | measurable pairs | quality |
|---|---|---|
| omitted accessors (`GetJsonValueFirstOf`, `ReadFieldProjection`) | ≥16 | high — hand-verified, plus undeclared `field` not counted |
| raw-payload reads | 36 (29 excl. an already-ticketed verb) | high — clusters hand-verified against declarations |
| shared `FHandlerContext&` helpers | 95 / 31 verbs | medium — bare-name attribution, ~60-95 real |
| non-literal keys | 53 sites / 51 verbs | unknowable, not countable |

**≈140 undeclared (verb, param) pairs remain against an empty baseline** — about twice what the
original sweep found and fixed.

**Severity:** High. Impact is "silent false-success" applied to the guard itself — a green ratchet
with an empty baseline actively asserts a class is eradicated while ~140 instances stand, and it is
the artefact a reviewer trusts instead of re-sweeping; reach is every test pass and 10+ namespaces,
and at least one cluster (`effect.draw_debug_shape` shape geometry) rejects valid input with no
workaround. Bands with the parent `B-verbs-read-undeclared-parameters` (High), which is the same
class measured through the one shape that was visible.

**Workaround (per verb, not for the class):** use the canonical declared spelling — `collisionEnabled`,
`assetPath`, `namesOnly`, `time`, `levelPath`. Does not exist for `effect.draw_debug_shape`'s
per-shape geometry or `landscape.create`'s grid, where the key is the only way in.

**Fix:** three separable pieces; the first two are the defects, the third is the reason they were
invisible.

1. *The two named instances.* `actor.set_collision`: swap `RPC_PARAM_OPT("collisionEnabled", ...)`
   for `ParamAliasUtils::MakeAliasParamSpec(TEXT("collisionEnabled"), …, {collisionEnabled,
   collision_enabled})`, matching the `actorName` slot two lines above, and drop the now-redundant
   raw-payload branch in favour of `Ctx.GetBoolFirstOf`. `behavior_tree.attach_decorator` /
   `attach_service`: replace the bare `RPC_PARAM_REQ("assetPath", ...)` at `:856` and `:870` with
   `BTAssetPathParamReq()` — the factory already exists and its five siblings already use it, so
   this is a one-token change per verb.
2. *The measured clusters.* Same decision procedure the parent ticket used (declare where the
   description already promises the spelling; delete where the read is a dead fallback). The
   `names_only` group is 11 mechanical declarations whose descriptions already promise the key;
   `effect.draw_debug_shape` and `landscape.create` need real `RPC_PARAM_OPT` entries, not aliases.
   These should land **before** the scanner is widened, or the widening turns the build red.
3. *Widening the guard.* Two of the four shapes are cheap and safe, one is not, one is impossible:
   - **Omitted accessors — do it.** Add `GetJsonValueFirstOf` to the alternation in
     `ReadSitePattern`, and special-case `ReadFieldProjection` to contribute the fixed key set
     `{fields, field, namesOnly, names_only}` (its braced argument is *column* names, not wire keys
     — counting it produces false positives). No new machinery, and it closes 16+ pairs of exposure.
     Guard against recurrence: a companion assertion that every `const FString& Key` /
     `const TArray<FString>& Keys` accessor declared in `HandlerContext.h` appears in the
     alternation, so the next accessor added cannot silently widen the blind spot.
   - **Raw-payload reads — do it, scoped.** Within a handler body, bind the variables assigned from
     `Ctx.GetRawPayload()` and match only `<thatvar>->{Has,TryGet,Get}*Field(TEXT("k"))` and
     `GetJson*Field(<thatvar>, TEXT("k"))`. A one-variable taint, no call-graph. Measured here at 36
     pairs with no hand-visible false positives (nested sub-object reads bind a different variable
     and drop out); expect to seed a small baseline for the deliberate ones.
   - **Shared helpers — do not.** The file's own reasoning holds: bare-name resolution across
     modules produced false attributions in both independent sweeps
     (`widget.describe`/`ParseRequest` last time, `attach_decorator:serviceClass` this time). Fix
     these by sweep, not by test. If they must be guarded, the tractable version is narrower —
     assert that verbs delegating to the *same* named helper declare the *same* accepted-name set
     (which catches the `behavior_tree` divergence exactly, without resolving any call graph).
   - **Non-literal keys — impossible.** State it in KNOWN LIMITS as permanent, not deferred.

   Whatever is widened, update the KNOWN LIMITS block and the "THE LIST IS NOW EMPTY" note at
   `:275-283`: as written, the empty baseline claims more than the scan can support.

## History
- `#1-guard-blind-spot-quantified` `OPEN` reporter — Filed as one ticket, not two: the guard gap and
  the defects are the same fact at two altitudes (the guard is blind, so the class survives), they
  share a file set, and either half shipped alone is wrong — fixing the instances leaves ~140 more,
  widening the scanner first turns the build red. Verified both instances named by
  `B-verbs-read-undeclared-parameters` are still present after that ticket's 21-file sweep
  (`ActorPropertyHandler.cpp:196`, `BehaviorTreeHandler.cpp:397` reached from `:854`/`:868`).
  Characterised `ReadSitePattern` (`TestDeclaredParamCoverage.cpp:135-141`) and reproduced its
  results offline: same shape → 0 pairs, matching the live empty baseline, which calibrates the
  three shapes it cannot see at ≥16 / 36 / ~95 pairs plus 53 unknowable sites.
- `#2-widened-scan-and-cluster-sweep` `IN-REVIEW` developer — Scanner widened from **1 read shape
  to 4**, and the empty baseline replaced by a **true one of 8**. Offline replica of the widened
  scanner (declaration side parsed from source, calibrated against the shipped shape: 0 pairs, same
  as the live empty baseline) measured **77 pairs** the old scan could not see —
  **22** `ReadFieldProjection` (11 verbs × `field` + `names_only`), **5** `GetJsonValueFirstOf`
  (`sequencer.set_playhead`'s `frameNumber`/`frame_number`/`seconds`/`timeSeconds`/`time_seconds`),
  **50** raw-payload. **69 fixed, 8 left**, and the 8 are one verb: `environment.build`, whose
  forwarded sub-action payload is `B-environment-build-dispatcher-rejects-forwarded-params`' own
  defect and needs a forwarding contract, not eight `RPC_PARAM_OPT` lines. Both named instances
  fixed: `actor.set_collision` declares `collision_enabled` and reads it through
  `Ctx.GetBoolFirstOf` (the raw-payload branch is gone), `behavior_tree.attach_decorator` /
  `attach_service` now call `BTAssetPathParamReq()` like their five siblings. Other clusters
  closed: 12 `level.*` fallback spellings, `landscape.create`'s grid (`landscapeName`,
  `quadsPerSection`, `componentCount`, `sizeX`, `sizeY`, flat `x`/`y`/`z`),
  `effect.draw_debug_shape`'s 7 per-shape geometry params (the no-workaround cluster),
  `blueprint.list`'s nested `filter`/`pagination`, `blueprint.*_function:memberName`,
  `asset.get_dependencies_classified:dependencyMode`/`Role`, `foliage.*:foliageType`/`location`,
  `effect.*_niagara:actorName`; `system.console_command`'s `params` envelope was DELETED as a dead
  fallback (`command` is required, so the envelope-only request never reached the body). Scanner
  now matches: the 20-name `KeyTakingAccessors()` list (was 19, `GetJsonValueFirstOf` added);
  `Ctx.ReadFieldProjection(...)` contributing its fixed `{fields, field, namesOnly, names_only}`
  set (its braced argument is column names and is deliberately NOT read as keys); raw-payload reads
  off a local bound from `Ctx.GetRawPayload()` (one-variable taint, no call graph) plus the unbound
  `Ctx.GetRawPayload()->…Field(key)` chain, which measured 0 new pairs and is therefore free.
  Recurrence guard + regression test: new
  `PinWright.infra.declared_params.ScannerSeesEveryCoveredReadShape` runs `ScanFile` over synthetic
  source with one case per covered shape, pins the two shapes that must NOT be collected (a nested
  sub-object read; a non-literal key), and fails when a `const FString& Key` /
  `const TArray<FString>& Keys` accessor is added to `HandlerContext.h` without being added to
  `KeyTakingAccessors()` (`Send*`/`Make*` exempt — error codes and request ids, not wire keys).
  Two shapes stay uncovered and are now stated as such in KNOWN LIMITS: shared `FHandlerContext&`
  helpers (~60-95 real pairs, fix by sweep — bare-name resolution false-attributed in two
  independent passes) and non-literal keys (53 sites / 51 verbs, **permanent**, not deferred).
  New shared macros `RPC_PARAM_REQ_ALIAS` / `RPC_PARAM_OPT_ALIAS` in `Handlers/ParamAliasUtils.h`
  carry the 37 mechanical alias declarations without re-wrapping their descriptions. 21 files
  changed. **Not compiled and not run** — the wave orchestrator builds and runs the suite; the 8
  baseline entries and every fix were derived from an offline replica of the widened scanner, so
  the first live run is what confirms the replica agreed with the registry.
