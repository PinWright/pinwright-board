---
id: B-declared-param-guard-blind-to-helpers
title: "The declared-param guard stops at the handler body, so 444 measured undeclared (verb, key) pairs across 81 verbs sit one call frame away from an 8-entry baseline that reads as near-eradicated"
status: OPEN
severity: High
category: bug
tags: [dispatcher, unknown-params, undeclared-parameter, param-spec, unreachable-code, test-coverage, guard-blind-spot, shared-helper, alias, sweep]
encounters: 1
lastSeen: 2026-08-29
---

# `HandlersOnlyReadDeclaredParams` measures the macro body and nothing it calls

`PinWright.infra.declared_params.HandlersOnlyReadDeclaredParams`
(`Source/PinWright/Private/Tests/Infra/TestDeclaredParamCoverage.cpp`) scans four read shapes
inside each `REGISTER_RPC_HANDLER` body and diffs them against the live registration. Its baseline
is 8 entries, all `environment.build`. A read one call frame away — inside a helper taking
`FHandlerContext&` or the raw payload — is invisible to it. The file states that gap in KNOWN
LIMITS and prices it at "~60-95 real pairs across 31 verbs", sourced from a bare-name sweep it
also (correctly) says false-attributed twice.

**That estimate is wrong by roughly 5×. A scope-resolved measurement finds 444 pairs across 81 of
1,217 registered verbs.** Sibling ticket `B-declared-param-guard-blind-spots` (IN-REVIEW) closed the
four in-body shapes; this is the hop it explicitly scoped out.

## Attribution method, and why it is not the bare-name sweep

The two prior sweeps matched a helper **by name** and attributed its reads to every verb whose file
mentioned that name. This one resolves each call to a **definition site**:

1. **Registration side.** Locate every `REGISTER_RPC_HANDLER` whose first macro argument is a
   literal dotted name; brace-match the body. 1,217 recovered.
2. **Declaration side, over-accepting on purpose.** The accepted-name set is every string literal
   in the `RPC_PARAMS` region **plus** the literals of every function it calls and every macro it
   names, transitively to depth 6. This is what resolves `BTAssetPathParamReq()`,
   `ParamAliasUtils::MakeAliasParamSpec`, `VisitBlueprintPathScalarFieldNames`, and the object-like
   macros (`DRIVE_COMMON_ACTION_PARAMS` → `DRIVE_WINDOW_SELECTOR_PARAMS`). Over-acceptance makes
   every count below a **lower bound**: a key wrongly believed declared is dropped, never added.
3. **Helper index.** Every function definition whose parameter list contains `FHandlerContext&`
   (236 found, 90 key-reading) or a `TSharedPtr<FJsonObject>` (505, 128 key-reading).
4. **Call resolution.** For a call from handler file F: candidates are definitions of that name;
   a qualified call must scope-match (the qualifier segments must appear contiguously in the
   definition's enclosing namespace/class chain — this is what the earlier sweeps lacked, and what
   makes `PinWright::MetaSound::BuildMetaSoundLiteralFromParams` and the nested
   `FCommonParams::Extract` resolve); a `static` / anonymous-namespace definition in another file is
   discarded; a `.cpp` definition in another file requires a declaration in F's transitive include
   closure. **0 unresolved, 0 ambiguous** across 832+ call edges — the bare-name failure mode is
   gone, not mitigated.
5. **The call must actually pass the context**, and for payload helpers an argument must **be**
   `Ctx.GetRawPayload()` or a local bound from it — not merely mention it.
   `GetVectorFromJsonLS(GetObjectFieldLS(Payload, TEXT("instanceLocation")))` passes a **nested**
   object; counting it produced 15 false `x`/`y`/`z`/`pitch`/`roll`/`yaw` pairs on
   `level.structure.*` before the rule was tightened.
6. **Branch evaluation.** A helper that branches on a caller-fixed discriminator
   (`HandleAttachBTSubNode(Ctx, /*bDecorator=*/true)`,
   `ParseModulePayload(Payload, ENiagaraEditOperation::SetModuleInput, Out)`) is evaluated per key:
   enclosing `if`/`while` conditions, the condition a key literal sits *inside*, and `?:` arms are
   resolved against the literal the caller passes. **36 pairs are dropped as provably unreachable.**

**Calibration.** Restricted to the shipped guard's four in-body shapes, the replica returns
**exactly 8 pairs, all `environment.build`, identical to `KnownUndeclaredReads()`** — 1,217
registrations, zero extras, zero omissions. The declaration parser is therefore adequate on this
tree, which is what licenses the helper numbers built on it.

**Error directions.**
- *False positives:* flow insensitivity only. Three independent verification agents hand-audited an
  earlier 134-pair snapshot against source: **112 REAL, 22 FALSE, 0 FALSE-DECLARED, 0
  FALSE-NOT-CALLED, 0 FALSE-NESTED.** All 22 were discriminator-gated and all 22 were flagged by the
  tool; step 6 now removes them. One residual class survives, 2 known instances: a read after an
  **early-return guard clause** the caller's argument makes fire —
  `blueprint.graph.{list_graphs,list_node_types}:graphName`, where `bGraphRequired=false` returns
  before the read.
- *False negatives, i.e. the count is low:* the same audit found real pairs the scanner misses.
  `chooser.add_column:contextIndex` is still absent; so are ~27 `ParseSubject` legacy-target keys
  (`assetPath`, `actorName`, `actor_name`, `actorPath`, `objectPath`, `radius`, `closeAfterCapture`,
  `kind` across six capture verbs). Three method corrections came out of that audit and are already
  folded into the 444: taint propagation through `TSharedPtr<FJsonObject> X = Payload;`, free-field
  accessors beyond `GetJson*Field` (`GetStringField(Payload, …)`, `TryGetIntField(…)` — the
  locally-named wrappers `NiagaraEditTypes.cpp` uses), and stratum C below.

**Number I would defend: 444 pairs / 81 verbs as a floor; true population plausibly 470-520.** Not
60-95. If only pairs whose helper is verified reachable on every path are counted, the floor is 262.

## Measured population

| stratum | what it is | pairs |
|---|---|---|
| A | helper takes `FHandlerContext&` | 124 |
| B | helper takes the raw payload object | 311 |
| C | the key is a helper **parameter** bound to a literal at the call site | 9 |
| | **total** | **444 across 81 verbs** |

262 read on every path through the helper; 173 on a fallback/branch that is still reachable; 36
further candidates were dropped as unreachable. 204 of 1,217 verbs route through at least one
key-reading helper, so the exposed surface is ~17% of the registry and 40% of that surface is
defective. 414 pairs are in the main module, 19 in `PinWrightChooser`, 2 in `PinWrightGeometry` —
a host with an integration's engine plugin disabled measures fewer, as the guard already documents.

By namespace: `niagara` 297, `drive` 25, `ai` 25, `game_framework` 21, `chooser` 19, `eqs` 18,
`render` 8, `audio` 8, `blueprint` 4, `behavior_tree` 4, `state_tree` 2, `geometry` 2, `camera` 2.

**Stratum C is the interesting one, because the guard calls this class permanent.** KNOWN LIMITS
says a non-literal key "does not exist in the source text, so no widening of a source scanner can
ever recover it", 53 sites / 51 verbs. Reproduced: 52 sites / 51 verbs in handler bodies, plus 41
in helper bodies. **65 of those 94 sites pass a named key-list factory**
(`MaterialHandlerUtils::MaterialAssetPathKeys()` 21×, `WidgetAssetPathParamNames()` 15×,
`AssetPathParamUtils::AssetPathKeys()`, `EditorHandlerUtils::ToggleKeys()`, `ActorNameKeys()`, …)
whose literals **are** in source text and are already resolved by the declaration side. Only 29
sites across 18 bare local names need real dataflow. Resolving the factories surfaces **0** new
pairs — every such factory also feeds the declaration — so the classification is *wrong* while its
consequence today is *nil*. Substituting call-site literals into helper parameters, however,
surfaces 9 real pairs the guard cannot see, one of which
(`state_tree.add_condition:conditionStruct`) an auditor found independently.

## Helper enumeration

Verbs / keys / whether the callers' declarations agree. "agree" = the caller verbs' accepted-name
sets are identical **restricted to the keys this helper reads**.

| helper (file:line) | verbs | pairs | callers agree? |
|---|---|---|---|
| `ParseModulePayload` `Niagara/NiagaraEditTypes.cpp:819` | 9 | 122 | no |
| `ParseRendererPayload` `:778` | 3 | 44 | no |
| `ParsePinPayload` `:908` | 3 | 41 | no |
| `ParsePropertyPayload` `:690` | 2 | 30 | **yes** |
| `HandleSetTestScoring` `AI/EQSHandler.cpp:679` | 2 | 27 | no |
| `FCommonParams::Extract` `Systems/GameFrameworkHandler.cpp:172` | 7 | 21 | **yes** |
| `ParseDataInterfacePayload` `NiagaraEditTypes.cpp:1491` | 4 | 19 | no |
| `ParseEventHandlerPayload` `:1397` | 2 | 17 | no |
| `ParseSimulationStagePayload` `:1446` | 2 | 14 | no |
| `FDriveActionCommon::RunAction` `Drive/DriveActionCommon.cpp:141` | 6 | 12 | **yes** |
| `ParseParameterPayload` `NiagaraEditTypes.cpp:723` | 3 | 10 | no |
| `HandleSetContextClass` `AI/EQSHandler.cpp:518` | 2 | 6 | no |
| `ParseSubject` `Render/CaptureSubject.cpp:484` | 5 | 5 | **yes** |
| `HandleSetTestFilter` `AI/EQSHandler.cpp:593` | 1 | 5 | n/a |
| `HandleSetCell` `Chooser/ChooserAuthoringHandler.cpp:1107` | 1 | 5 | n/a |
| `ReadRequestedIds` `Audio/AudioSynthCandidateHandler.cpp:68` | 1 | 4 | n/a |
| `ParseViewportCaptureRequest` `Render/PreviewViewportCaptureUtils.cpp:1483` | 3 | 4 | no |
| `HandleAttachBTSubNode` `AI/BehaviorTreeHandler.cpp:394` | 2 | 4 | no |
| `BuildMetaSoundLiteralFromParams` `Audio/MetaSound/MetaSoundLiteralParams.h:137` | 4 | 4 | **yes** |
| `HandleSetResult` / `HandleCompile` / `HandleAddRow` / `HandleAddColumn` `ChooserAuthoringHandler.cpp` | 1 each | 3 each | n/a |
| `HandleAddTest` `AI/EQSHandler.cpp:453` | 2 | 3 | no |
| `SkeletalIOResolveSkeleton` `Geometry/SkeletalMeshAssetIOHandler.cpp:399` | 2 | 2 | **yes** |
| `ResolveTransition` `AI/StateTreeAuthoringHandler.cpp:434` | 2 | 2 | no |
| `ResolveBlueprintAndGraph` `Blueprint/BlueprintGraphHelpers.cpp:64` | 22 | 2 | no (both are the early-return false positive) |
| `ReadStructFieldSpec` `Blueprint/BlueprintTypeDefinitionHandler.cpp:215` | 1 | 2 | n/a |
| `HandleCreate` `ChooserAuthoringHandler.cpp:925`, `GetStructParam` / `FindKeyArray` (stratum C) | 1-3 | 1-6 | n/a |

67 helpers have ≥2 caller verbs and read ≥1 key; **41 pairs are in helpers with exactly one caller**
— `HandleSetCell`, `ReadRequestedIds`, `HandleAddColumn`, the eight `FDriveWebHandlers::*Web`
entry points. Those are not shared helpers at all; they are handler bodies moved one line out of the
macro (`{ return PinWrightChooser::HandleSetCell(Ctx); }`). The guard would have seen every one of
them had the code stayed inline. **This is a scanner-scope defect, not a call-graph problem.**

Worked instances worth naming, each self-indicting:
- **`audio.authoring.*` × 4 — `objectPath`.** The `objectValue` parameter's own description reads
  `"…object to bind (alias: objectPath). NOT assetPath…"` (`AudioAuthoringHandler.cpp:1382`,
  `MetaSoundVariableHandler.cpp:81`/`:287`, `MetaSoundNodeInputDefaultHandler.cpp:64`). The wiki
  renders the promise; the dispatcher refuses the key.
- **`chooser.*` × 5 — `chooserPath`/`assetPath`/`tablePath`.** `LoadChooser`'s own error text says
  `"chooserPath, assetPath, tablePath, or path is required"` (`ChooserAuthoringHandler.cpp:103`).
  Three of the four spellings it names are refused.
- **`ai.add_eqs_context` vs `eqs.set_context_class`.** Same helper; the accepted sets are *inverted*
  on the context slot (`contextType` only vs `contextClass` only), so neither verb accepts both
  spellings the helper reads, and the deprecated alias also loses `generatorIndex`, `testIndex`,
  `propertyName`, `save`.
- **`niagara.*` — 297 pairs, one file.** Eight `Parse*Payload` helpers all route through
  `ParseTargetSpec` (`NiagaraEditTypes.cpp:146-227`), which reads the flat form of the target
  descriptor off the **top-level** payload as a fallback: `targetKind`, `emitter`, `emitterName`,
  `scope`, `scriptUsage`, `scriptType`, `nodeId`, `node`, `pin`, `pinName`, `entryId`, `moduleId`,
  `index`, `rendererIndex`, `toIndex`. No niagara verb declares more than a handful. E.g.
  `niagara.set_property` (`NiagaraEditHandler.cpp:2396`) declares
  `assetPath/target/propertyPath/value/compile/save` and the parser honours 15 more.
- **`render.capture_{asset_preview,open_level,annotated}`.** Three verbs, one parser, three
  *different* holes (`viewDistanceScale`+`hideEditorSprites`, `previewScene`, `allowBlank`). Three
  of the four already carry source comments admitting the wire refuses them
  (`RenderHandler.cpp:412-420`, `:1550-1566`) and clear the field for the direct-invoke path —
  i.e. the defect was found by hand, patched at the call site, and left undeclared.

## The proposed check: verbs sharing a helper must declare the same accepted-name set

Evaluated in both forms against the measurement.

**Full accepted-name-set equality is not enforceable.** 65 of the 67 shared key-reading helpers
fail it, because sibling verbs legitimately differ on parameters that have nothing to do with the
helper — the seven `game_framework.configure_*` verbs share `FCommonParams::Extract` and differ on
`maxPlayers` vs `maxRounds` vs `scoreToWin`. A gate that fails 97% of its subjects for reasons
unrelated to the defect is a baseline, not a check.

**Restricted to the keys the helper actually reads, it is enforceable.** 18 of 67 helpers fail; it
surfaces 358 pairs, of which 322 are real and **36 are the discriminator false positives** (a naive
implementation without step 6 emits `behavior_tree.attach_decorator:serviceClass` and the 32
operation-gated niagara keys — the exact false attribution that killed both earlier sweeps, so the
check must carry branch evaluation or seed those into its baseline).

**It catches the case it was proposed for.** `behavior_tree.attach_decorator` / `attach_service`
disagree on the class slot through `HandleAttachBTSubNode`; `ai.*` vs `eqs.*` disagree through four
EQS helpers; the three capture verbs disagree through `ParseViewportCaptureRequest`;
`state_tree.add_condition` vs `set_transition_trigger` disagree through `ResolveTransition`.

**It misses 113 of 444 — and it misses them structurally, not incidentally:**
- **41** because the helper has exactly one caller verb. Nothing to compare against, ever.
- **72** because every sibling is *uniformly* undeclared. All seven `game_framework` verbs agree
  with each other about not declaring `name`/`path`/`blueprintPath` (21 pairs); all six `drive`
  action verbs agree about not declaring `instanceName`/`rootIndex` (10); all five `ParseSubject`
  callers agree about `point` (5); all four MetaSound verbs agree about `objectPath` (4);
  `ParsePropertyPayload`'s two callers agree (30). **The check compares siblings to each other, not
  to the helper, so agreeing on being wrong reads as clean.** It would report the four helpers named
  in the guard's own KNOWN LIMITS as passing.

An independent auditor reached the same conclusion unprompted: *"the file's own proposal would catch
none of these, because in all three families the callers agree with each other. They agree on being
wrong."* Note `Tests/Render/TestCaptureVerbParameterParity.cpp` already implements this rule for
four capture slots (`MakeHelperGroups()`, `:417-485`), which is exactly why the
`ParseViewportCaptureRequest` gaps are commented and the `ParseSubject` gaps are not — `ParseSubject`
is not in the group list. The pattern works; its ceiling is 73%.

**Failure message it would need.** Naming the disagreeing verbs is not actionable — the reader
cannot tell which of them is wrong. It must print: the helper (file:line), the key, the verbs that
accept it, the verbs that do not, and the read site inside the helper — e.g.
`HandleAttachBTSubNode (BehaviorTreeHandler.cpp:442) reads 'decoratorType'; accepted by <none>,
refused for behavior_tree.attach_decorator. Declare it on the slot it feeds or delete the read.`

## Alternatives considered and rejected

**A real call-graph pass (clang).** UBT ships `-Mode=GenerateClangDatabase`
(`C:\UE_5.8\Engine\Source\Programs\UnrealBuildTool\Modes\GenerateClangDatabase.cs`), so
`compile_commands.json` is obtainable — but it must be generated with `-NoUnity` to be per-file,
and **nothing else is present**: no `compile_commands.json` in the tree, no `.clang-tidy`, no clang
reference anywhere in `ci/`, `scripts/`, `Content/Python/`. This host has
`clang-format.exe` + `clang-tidy.exe` under VS 2022 and **no `clang.exe`, no `libclang`, no
`clang-query`**. The decisive objection is not the toolchain cost: the guard is an in-editor
automation test, and an AST pass cannot run there. It becomes a second CI job with a toolchain the
repo does not have, a UE-header clang parse the tree has never done, and a separate failure surface.
Rejected — the shapes that occur are decidable from source text, as demonstrated.

**Helpers declare their own key set in a macro beside the definition, which the guard attributes to
callers.** Touches 218 key-reading helpers, and the declaration drifts from the body exactly the way
`RPC_PARAMS` drifts today — it recreates this defect one level down with nothing guarding the new
declaration. Making it safe requires verifying the macro against the body, i.e. the scan, at which
point the macro is redundant. Rejected.

**Refactor helpers to take explicit keys.** 218 helpers, real behaviour change, and 41 of the pairs
are single-caller helpers where the refactor is pure churn. It also cannot be verified without the
scan. Rejected.

**Accept and document.** This is the status quo and it is what produced the mispricing: KNOWN LIMITS
states the gap honestly but at 60-95 pairs and with the reason "resolving call targets from source
text means resolving overloads and same-named functions across modules, which produced false
attributions in two independent sweeps and is not worth a fragile parser inside a test" — a claim
this measurement refutes (scope resolution: 0 unresolved, 0 ambiguous, 832 edges). The text must be
corrected regardless of what is built.

**Fix:** extend the existing source scanner by one bounded hop; do not build a call graph and do not
build the sibling check as the primary gate.

1. **Widen `ScanFile` to a one-hop-plus-forwarding closure.** Index functions taking
   `FHandlerContext&` or a `TSharedPtr<FJsonObject>`; resolve each call from a handler body by
   definition site (file identity, `static`/anonymous-namespace linkage, scope-chain suffix match
   for qualified calls, transitive include closure for cross-TU calls); attribute only when an
   argument **is** `Ctx` / the payload; close transitively over helper→helper forwarding. Carry
   per-key branch evaluation (`if (Disc == EnumLit)`, `if (bDisc)`, `?:`) against the literal each
   caller passes — without it the sweep re-creates the `attach_decorator:serviceClass` false
   attribution that discredited the two previous attempts. ~250 lines, no toolchain, runs inside the
   existing test. Measured cost of the whole analysis over the tree: <15 s.
2. **Fold in the two corrections this measurement forced,** both of which also affect the in-body
   scan: taint through `TSharedPtr<FJsonObject> X = <tainted>;`, and free-field accessors beyond
   `GetJson*Field` — `(?:Try)?Get[A-Za-z]*Field(payload, key)`, which is how `NiagaraEditTypes.cpp`
   reads everything. Neither adds a single direct-body pair today (verified: still exactly 8), so
   they can land ahead of the sweep with zero baseline churn.
3. **Add stratum C:** when a helper reads `Ctx.Get*(P)` and `P` is one of its own parameters,
   substitute the literal the call site passes. 9 pairs, and it retires the "permanently invisible"
   half of the non-literal claim.
4. **Seed a baseline of the measured pairs and ratchet**, exactly as the in-body scan does — these
   are 81 verbs and several owners, and some are deliberate (three `render.capture_*` pairs are
   already commented as knowingly refused). The 297 `niagara.*` pairs are one design decision, not
   297 bugs: `ParseTargetSpec`'s flat target form is either declared once through a shared
   `RPC_PARAMS` factory on all 27 verbs, or deleted. Treat it as one work item, like
   `B-environment-build-dispatcher-rejects-forwarded-params`.
5. **Keep the sibling-agreement check as a cheap extra assertion once step 1 exists** — it needs the
   same resolution and adds a second, differently-shaped failure ("these siblings disagree") that is
   often the more actionable message. It is not a substitute: ceiling 73%, blind to single-caller
   helpers and to uniformly-wrong families.
6. **Correct KNOWN LIMITS**: the helper gap is ~444, not 60-95; bare-name resolution is not the only
   option; and the non-literal-key bucket is 65/94 sites recoverable, permanent only for the 29
   bare-local sites.

**Workaround (per verb, not for the class):** use the canonical declared spelling. None exists for
the flat `niagara.*` target descriptor (`emitterName`, `scriptType`, `nodeId`, `pinName`,
`rendererIndex`, `toIndex` — the nested `target` object is the only reachable form), for
`chooser.set_cell:columnIndex`, or for `audio.authoring.*:objectPath`.

## History
- `#1-scope-resolved-measurement` `OPEN` reporter — Filed separately from
  `B-declared-param-guard-blind-spots` rather than appended: that ticket is IN-REVIEW awaiting a
  tester's verdict on a shipped fix (four in-body read shapes, 69 declarations, an 8-entry
  baseline), and it scoped the helper hop OUT explicitly ("Shared helpers — do not") as an accepted
  KNOWN LIMIT. Appending would reopen work a tester is about to close and make `DONE` ambiguous
  about which fix was verified. Cross-references both ways; bands with that ticket and with
  `B-verbs-read-undeclared-parameters` at High. Method: replica calibrated against the shipped
  guard's four shapes reproducing its live result exactly (1,217 registrations, 8 pairs, all
  `environment.build`, identical to `KnownUndeclaredReads()`), then extended with definition-site
  call resolution (0 unresolved, 0 ambiguous over 832 edges), a payload-argument identity rule, and
  per-key branch evaluation. **444 pairs across 81 of 1,217 verbs**, vs the file's recorded ~60-95.
  Three independent verification agents hand-audited a 134-pair snapshot against source: 112 REAL /
  22 FALSE, every false one discriminator-gated and every one flagged by the tool, now removed by
  branch evaluation; the same audit found further real pairs the scanner still misses
  (`chooser.add_column:contextIndex`, ~27 `ParseSubject` legacy-target keys), so 444 is a floor.
  Two method defects were found and corrected mid-measurement and are recorded because they are the
  traps for whoever implements this: unexpanded object-like macros in `RPC_PARAMS`
  (`DRIVE_COMMON_ACTION_PARAMS`) inflated `drive.*` by 96 phantom pairs, and a "call argument
  mentions the payload" rule counted nested-object reads as top-level wire keys on
  `level.structure.*`.
