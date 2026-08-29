---
id: B-param-type-never-validated
title: "FParamSpec::Type is declared on 5,418 parameters across 1,188 verbs, rendered into every wiki page as a contract, guarded by its own registry-wide grammar test — and read by no runtime code: the dispatcher validates parameter NAMES and never their JSON type, so a wrong-shaped value coerces silently and eight confirmed verbs answer success after losing, fabricating or deleting asset data"
status: OPEN
severity: Critical
category: bug
tags: [dispatcher, param-spec, type-validation, silent-coercion, silent-wrong-data, silent-false-success, destructive-write, json-null, guard-blind-spot, cross-cutting]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The declared type is a promise the dispatcher never reads

`FParamSpec::Type` (`Source/PinWright/Private/Handlers/ParamSpec.h:19`) is set on every parameter
the plugin declares. `FRpcDispatcher::ValidateHandlerParams`
(`Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:91-163`) checks that required *names* are
present (`:97-116`) and that no *unknown* name was sent (`:120-161`). It never compares the incoming
value's JSON type to the declared one. `PayloadHasParamOrAlias` (`:33-60`) tests `HasField` and
nothing else.

A caller who **misspells** a key gets `UNKNOWN_PARAMS` naming every valid parameter. A caller who
spells it **right and shapes it wrong** gets a success payload built on a coerced or defaulted
value. That asymmetry is the defect, and it is not per-verb: the gate runs inside the bridge lambda
that wraps every auto-registered handler (`RpcDispatcher.cpp:324`, plus the text-format path at
`:609`), so one function covers all 1,354 verbs — the same reason a fix belongs there.

`B-asset-search-array-class-filter-silently-dropped` (IN-REVIEW) is one instance of this root,
measured on one param. This ticket is the root.

## What `Type` is actually used for

Two production reads in the entire plugin, both display:

1. `Catalog/MarkdownHelpers.cpp:24` — `RenderParamBullet` prints ``- `name` (`string`, required)``
   into the wiki page (reached from `Catalog/WikiHandler.cpp:747`).
2. `Dispatch/RpcDispatcher.cpp:107` — the `(type: %s)` substring of the `MISSING_REQUIRED_PARAM`
   message.

`FParamAliasSpec::Type` (`ParamSpec.h:11`) is worse: exactly one read, `MarkdownHelpers.cpp:57`.
The dispatcher consumes `Alias.Name` only (`:52`, `:71`), so a typed alias's declared type is
documentation with no enforcement path at all.

There is also no machine-readable escape hatch. `BuildCallToolDescriptor`
(`Transport/McpRequestCore.cpp:239-267`) advertises `args` as a bare `{"type":"object"}` with no
`properties`, so the MCP client is handed zero per-verb type constraints and cannot validate on its
side either. The rendered markdown bullet is the only place a type reaches a caller.

**This is not a docs-only field, and that distinction is the reason this ticket is Critical rather
than Low.** The plugin maintains it as if it were enforced:

- `Tests/Infra/TestContractConsistency.cpp:78` asserts `Param.Type` is non-empty for every param.
- `Tests/Infra/TestContractConsistency.cpp:183-234`
  (`PinWright.infra.contract.ParamTypes.ValidTypeNames`) is a registry-wide test that parses `Type`
  as a `|`-separated union over `{string, number, integer, boolean, bool, object, array, any}`.
  **A grammar, and a test defending it, for a field no runtime code reads.**
- `Tests/Infra/TestContractConsistency.cpp:333-357` (`RequiredParamGate::SetPlaceholder`) branches
  on `Spec.Type` to synthesize a correctly-shaped probe value, carrying the comment:
  *"PayloadHasParamOrAlias (RpcDispatcher.cpp:92) tests HasField and never the type, so the value is
  irrelevant to the gate"*. The blindness is already written down inside the suite that depends on it.
- `docs/rpc-design.md:107` instructs developers: *"Enforcement is not in the handler body — it is
  `FRpcDispatcher::ValidateHandlerParams` ... **Declaring the param IS the coverage.**"* True for
  requiredness. False for type. Stated without qualification.
- `docs/rpc-design.md` §21, *"An accepted parameter is a promise"*: three honest outcomes —
  implement it, reject it with a typed error, or re-document the verb. *"Accepted-and-ignored is not
  one of them."* Every instance below is accepted-and-ignored.
- `Handlers/ErrorCodes.h:33` already reserves the vocabulary — *"wrong param type:
  `INVALID_PARAM_TYPE` / `INVALID_PARAMETER_TYPE`"*, both constants present at `:597`/`:599`. Every
  emitter is a **domain** type check (Niagara parameter types, material parameter types, PCG). The
  dispatcher emits neither. A new code (`PARAM_TYPE_MISMATCH`) is needed so the wire axis is
  distinguishable from the domain axis.

Twenty-five params already carry a union type (`array|string`, `object|number|boolean`, ten distinct
spellings). Authors have been hand-encoding "this one accepts two shapes" into a field nothing reads.

## The coercion matrix (verified against `C:\UE_5.8\Engine\Source\Runtime\Json\`)

`FJsonValue::AsString/AsNumber/AsBool` (`Private/Dom/JsonValue.cpp:26/13/50`) call `TryGet*` and,
on failure, `ErrorMessage()` — `UE_LOG(LogJson, Error, ...)`, `Warning` for null — then return the
zero value. **That log line goes to the editor log and never to the caller.** That is the whole of
"silent".

| declared | read via | wrong shape | result |
|---|---|---|---|
| `string` | `GetString` | number | `"5"` — coerced, usually fine (`JsonValue.cpp:446`) |
| `string` | `GetString` | bool | `"true"` / `"false"` — coerced (`:488`) |
| `string` | `GetString` | **array / object** | `""` + editor-only log — **silent drop** |
| `number` | `GetNumber` / `GetInt` | numeric string `"5"` | parsed (`JsonValue.h:451`) — fine |
| `number` | `GetNumber` / `GetInt` | `"2s"`, `"50%"`, `"100cm"`, `"1,000"`, `"auto"` | **0.0** + editor-only log |
| `number` | `GetNumber` / `GetInt` | array / object | **0.0** + editor-only log |
| `boolean` | `GetBool` | string | `FCString::ToBool` — `true/yes/on` + nonzero int → true; **`"y"`, `"enabled"`, `"always"`, `"default"` → false** |
| `boolean` | `GetBool` | **array / object** | **false** + editor-only log |
| `array` / `object` | `GetArray` / `GetObject` | any scalar | `nullptr`, **indistinguishable from absent** |

`FJsonObject::GetIntegerField` is `(int32)GetNumberField` (`JsonObject.cpp:409-411`), so
`Ctx.GetInt` routes through the double overload — a non-numeric string is 0 with a log, not the
`LexFromString`-returns-true trap (that trap is real but reachable only via
`TryGetNumberField(name, int32&)`).

### The `null` vector — same gate, same fix site, worst payload

`FJsonObject::HasField` (`JsonObject.cpp:366-375`) returns **true** for an `EJson::Null` value: it
tests only that the shared pointer is valid, never the type. `FJsonValueNull` overrides no `TryGet*`,
so it resolves to `""` / `0` / `false` everywhere. Nothing in `McpRequestCore.cpp` or
`RpcDispatcher.cpp` strips nulls (no `EJson::Null` / `IsNull()` reference exists in either).

So **`{"preserveProperties": null}` passes the required-param gate at `RpcDispatcher.cpp:40` and
arrives as `false`.** This is the most plausible wrong value in the whole matrix — a client whose
serializer emits nulls for unset optionals turns "I did not set this" into "I explicitly set this to
the destructive value" on every optional param in the registry. It is a distinct sub-defect
(presence, not type), but it lives in the same function and is closed by the same edit; splitting it
into its own ticket would put two halves of one guard in two work items.

## Measured surface

**Source.** `X:\src\unreal\unreal-fpv-new\Saved\PinWright\wiki\` — 1,363 method pages regenerated
2026-08-27 from **live** registrations, so helper-built and alias-factory specs are included, which a
macro grep of the source would miss (the exact blind spot `B-declared-param-guard-blind-to-helpers`
measures). `RenderParamBullet` emits one fixed one-line shape, and namespace pages contribute 0 of
the 5,425 bullets found, so there is no double-count.

**Declared surface** (excluding 9 `_test.*` verbs): **5,418 parameters across 1,188 verbs**,
1,693 required / 3,725 optional.

| declared type | count | | declared type | count |
|---|---|---|---|---|
| `string` | 2,658 | | `integer` | 118 |
| `number` | 1,096 | | `bool` | 62 |
| `boolean` | 831 | | `any` | 22 |
| `object` | 414 | | unions (`a\|b`) | 25 |
| `array` | 192 | | | |

The vocabulary is itself unmaintained: 62 `bool` and 118 `integer` sit outside the five types
`ParamSpec.h:19` documents. A type gate needs a normalization pass before it can switch on anything.

**Read-path classification.** Joined every declared param name against every
`Get<T>(TEXT("key"))` / `Require<T>(TEXT("key"))` literal in non-test handler source:

| | count | share |
|---|---|---|
| read **only** through a silent-default accessor | 2,813 | 52% |
| read **only** through a `Require*` | 135 | 2% |
| name appears under both (reused across verbs) | 1,577 | 29% |
| name matched no single-literal call site | 893 | 16% |

**932 of the 1,188 param-carrying verbs (78%) carry at least one lenient-only parameter.**

**Error directions — both understate the lenient share, so 52% is a floor, ~81% a ceiling:**

- The 893 unmatched are dominated by multi-key reads (`GetStringFirstOf({TEXT("a"), TEXT("b")})`)
  and reads one call frame away inside a helper, which a single-literal regex cannot see. Those
  reads are lenient.
- The 1,577 "both" bucket resolves majority-lenient, because only **65 of 473** `Require*` call
  sites actually reject a wrong type (below).

**Calibration.** The pipeline reproduces the known case exactly: `classFilter` maps to
`{GetArray, GetString}` and `asset.search` now declares it `array|string` — i.e. it correctly shows
the landed fix as a both-shapes read, while correctly showing `blueprint.build_api_index`'s
`classFilter` as array-only. **Measured false-positive rate:** of the 101 array-declared params the
join reports as having no scalar fallback, 13 are `fields`, which *does* accept a bare string —
inside `FHandlerContext::ReadFieldProjection`, one frame from the call site. 13/101 = **13%, all in
one direction**. Adjusted: 88 array params with no scalar fallback, 57 of them optional.

**The accessor layer is already inconsistent, and mostly on the wrong side.**
`Handlers/HandlerContext.cpp`:

- **Type-checking (refuse):** `RequireNumber` `:162`, `RequireBool` `:180`, `RequireObject` `:198`,
  `RequireArray` `:216` (all gate on `HasTypedField`); `GetObject` `:69`, `GetArray` `:78` return
  `nullptr`.
- **Not type-checking:** `GetString` `:17`, `GetNumber` `:25`, `GetBool` `:34`, `GetInt` `:43`,
  `GetVector`, `GetRotator`, `GetStringFirstOf`, `GetBoolFirstOf`, `GetIntFirstOf`;
  **`RequireString` `:88`** (array/object reach `""` and are refused with the *wrong* message,
  "Missing required string field"; number/bool coerce and **pass**); `RequireAssetPath` (built on
  `RequireString`); **`RequireInt` `:148`** (no check at all → 0).

Non-test `Require*` call sites: `RequireString` 337, `RequireAssetPath` 51, `RequireInt` 20 = **408
that do not type-check**, against `RequireNumber` 43, `RequireBool` 4, `RequireObject` 9,
`RequireArray` 9 = **65 that do**. Only **14% of the "required + validated" surface validates a type.**

Non-test silent-accessor call sites: `GetString` 1,269 · `GetBool` 584 · `GetNumber` 453 ·
`GetInt` 296 · `GetStringFirstOf` 126 · `GetArray` 80 · `GetObject` 62 · `GetBoolFirstOf` 26 ·
`GetJsonValueFirstOf` 19 · `GetVector` 19 · `GetIntFirstOf` 16 · `GetRotator` 7. Unfiltered upper
bounds are within 4% (`GetString` 1,280), so these counts are reliable.

## Confirmed instances

Each was traced from declaration to use in source. **All are reachable through the dispatcher** —
the name is declared, so `UNKNOWN_PARAMS` does not fire and only the type is wrong.

### Writes that lose, fabricate or destroy asset data

**C1 — `widget.replace_class` / `preserveProperties`. Undetectable total property loss.**
`Handlers/UI/WidgetReplaceClassHandler.cpp:25-26` declares `RPC_PARAM_DEF(..., "bool", ..., "true")`;
`:40` reads `Ctx.GetBool(TEXT("preserveProperties"), true)`. Send `null`, `["true"]`, `{"value":true}`
or `"y"` → **false** → `WidgetAuthoringUtils.cpp:1255-1258` skips the only call to
`CopyMatchingProperties`. The old widget has already been renamed to a `_REPLACED_<guid>` scratch
name and is discarded; the new one stays at class defaults. Response:
`{"success": true, "requiresCompile": true, "oldClass": ..., "newClass": ...}` — **the flag is not
echoed anywhere.** Every text, brush, color, padding, font and style value on that widget is gone
with no signal in the response.

**C2 — `vehicle.remove_wheel_setup` / `wheelIndex`. Deletes the wrong element, and the range guard
makes it look safe.** `Handlers/Physics/ChaosVehicleHandler.cpp:484` declares `"number"` required;
`:492` reads `Ctx.RequireInt` — the one `Require*` with no type check. Send `null`, `"rear-left"`,
`[2]` → **0**. The local is initialized to `-1`, but `RequireInt` overwrites it. The explicit
`WheelIndex < 0 || >= Num()` guard at `:503` **passes**, because 0 is in range; `:512`
`WheelSetups.RemoveAt(0)`, then compile + save. Response: `{"success": true, "wheelSetupCount": N-1}`
with **`wheelIndex` not echoed** (the sibling `set_wheel_setup` does echo it, `:474`). The caller
cannot determine which wheel was destroyed. The guard actively misleads — a caller reasonably
assumes a bad index is refused.

**C3 — `game_framework.configure_team_system` / `teamNames`. Fabricated data compiled and saved.**
`Handlers/Systems/GameFrameworkHandler.cpp:394` declares `"array"`; `:425` reads `Ctx.GetArray`.
Send `teamNames: "Red"` → `nullptr` → `:435-439` substitutes **invented** defaults
`["Team 1","Team 2"]`, writes them as blueprint variable defaults `Team0_Name`/`Team1_Name`
(`:442-447`), then `CompileBlueprint` + `McpSafeAssetSave`. Response: `success: true`,
`"Configured 2 teams"`, and the fabricated names echoed back as if requested.

**C4 — `material.authoring.set_blend_mode` / `save`. Work silently not persisted — and it is a
family of 149.** `Handlers/Material/MaterialAuthoringHandler.cpp:763` declares
`RPC_PARAM_OPT("save", "boolean", "Save after change (default true)")`; `:794` reads
`if (Ctx.GetBool(TEXT("save"), true)) SaveMaterialAsset(...)`. Identical pair at `:811`/`:839` for
`set_shading_model`. Send `save: null` → the material is mutated and `MarkPackageDirty()`'d
(`:791-792`), the save is skipped, and this call site does **not** route through
`AddAssetSaveReport` (unlike `:747` in the same file), so nothing in the response says so. The edit
is lost on editor restart. **`Ctx.GetBool(TEXT("save"), true)` appears at 149 sites** across
Animation, Audio, MetaSound, Material, Physics, Geometry, PoseSearch and Model handlers; 25 of them
live in files that never call `AddAssetSaveReport` at all.

**C5 — `sequencer.set_sub_section_range` / `startFrame` + `durationFrames`. Collapses an authored
section to nothing.** `Handlers/Sequencer/SequenceHandler.cpp:4062-4063` declare `"integer"`
required; `:4072`/`:4074` read `Ctx.RequireInt`. The description says *"frames (tick resolution)"*,
which invites a caller thinking in time to send `"2s"` or a timecode — all → 0. `:4106-4107`
`SetRange(TRange<FFrameNumber>(0, 0))` overwrites an existing section's range with an empty one,
`MarkPackageDirty()` at `:4119`. `range: {start: 0, end: 0}` **is** echoed, so it is detectable —
after the write. `sequencer.add_sub_sequence` (`:3970-3971`, reads `:3981`/`:3983`) has the same
shape and creates a zero-length section.

**C6 — `gas.set_ability_tags` / six tag params. No-op write, success, blueprint dirtied — past a
guard built for exactly this.** `Handlers/Systems/GASHandler.cpp:1031-1036` declare six `"array"`
params; `:1076` and `:1106` read them via `TryGetArrayField`. Send `abilityTags: "Ability.Fireball"`
→ false → zero tags applied → `MarkBlueprintAsModified` still runs → `SendSuccess` with
`tagsAdded: []`. `:1069-1073` builds an explicit validate-before-mutate mechanism *specifically so
tags are never silently dropped*; the wrong-shape input walks straight past it.
`gas.set_effect_tags` / `grantedTags` (`:1932`, read `:1971`) is the same shape.

**C7 — `chooser.set_cell` / `row` + `column`. Writes the wrong cell, past an explicit guard.**
`PinWrightChooser/.../ChooserAuthoringHandler.cpp:1279-1280` declare `"number"` required; `:1111`
reads `RequireInt(row)` and `:1115` `GetInt(column, INDEX_NONE)`. The handler has a deliberate
"column or columnIndex is required" refusal at `:1120-1123` keyed on `INDEX_NONE` — a wrong-typed
value produces **0**, not `INDEX_NONE`, so it walks past the guard and writes cell (0,0).

**C8 — `drive.key` / `modifiers`. Injects the wrong keystroke into the live editor.**
`Handlers/Drive/DriveActionHandlers.cpp:215` declares `"string"`; `:239` reads
`ParseModifiers(Ctx.GetString(TEXT("modifiers")))`, and `ParseModifiers` (`:62-66`) returns `None`
for an empty string. Send `modifiers: ["ctrl","shift"]` → `""` → **the plain keystroke is injected
with no modifiers held**, reported as a normal successful action. Ctrl+S becomes S. Nothing in the
response records what was actually sent.

### Reads that return a confident lie

**C9 — `asset.search_assets` / `classNames`. "This project contains no StaticMeshes."**
`Handlers/Asset/AssetQueryHandler.cpp:263-264` declare `"array"`; `:276`/`:346` read
`TryGetArrayField`. Send `classNames: "StaticMesh"` alone → the `FARFilter` is left fully empty →
`FARCompiledFilter::IsEmpty()` makes `UAssetRegistryImpl::GetAssets` return `false` **appending
nothing** → `{"success": true, "count": 0, "totalMatches": 0, "truncated": false}`. With
`packagePaths` also set, the class filter is dropped and **every asset under the path** comes back.
The confusion vector is concrete and documented: `asset.search`'s `classFilter` is declared
`"string"`, `asset.search_assets`'s `classNames` is `"array"`, and the two verbs cross-reference
each other in their own summaries.

**C10 — `blueprint.build_api_index` / `classFilter`. Minutes of scan and a multi-MB write for a
one-class request — and the same param name as the originating ticket, on the opposite axis.**
`Handlers/Blueprint/BlueprintApiIndexHandler.cpp:32` declares `"array"`; `:38` reads `Ctx.GetArray`;
the gate is `:80`. Send `classFilter: "AActor"` (the shape `asset.search` accepts for this same
name) → `nullptr` → `bHasFilter = false` → the job iterates **every `UClass` in the process** and
rewrites `Saved/AI/ApiIndex.json`, reported as a normal job success. `classFilter: [1, 2]` reaches
the same place: the element loop at `:42` skips non-strings silently and the set empties.

**C11 — `property.list` / `propertyNames`.** `Handlers/Utility/UtilityPropertyHandler.cpp:1879`
declares `"array"`; `:1911` reads it; the filter gate is `:1956`. Send `propertyNames: "bHidden"` →
allow-list empty → filter skipped → **every reflected property** on the object comes back with
values, defaults, override state and metadata (hundreds for an actor), `success: true`, and no field
echoes the requested filter. The sibling `nameMatch` on the same verb *is* a string, so the shape is
genuinely ambiguous from the page.

**C12 — `blueprint.search` / `timeoutSeconds`. The clamp turns a zero into a plausible lie.**
`Handlers/Blueprint/BlueprintIndexHandler.cpp:572` declares `RPC_PARAM_DEF(..., "number", "Total
seconds the call may spend waiting... Clamped to 1-600...", "100")`; `:582-583` read
`FMath::Clamp(Ctx.GetNumber(...), 1.0, 600.0)`. The name *and* the description say "seconds", so
`"120s"` / `"2 minutes"` / `"5m"` are exactly what a caller writes → 0.0 → **clamped to 1.0** →
`DeadlineSeconds = Started + 1.0` (`:631`). The Find-in-Blueprints kick cannot finish in one second
on a real project, so the answer is partial or empty with `timedOut: true` — practically, *"this
symbol is used nowhere"*, the worst answer a search verb can give. The clamp is what makes it
plausible instead of obviously broken.

**C13 — `asset.search_assets` / `limit` and `actor.list` / `limit`. The bound becomes unbounded.**
`AssetQueryHandler.cpp:267` declares `"number"`, `"0 for unlimited"`; `:376` reads
`Ctx.GetInt(TEXT("limit"), 100)` and `:377` gates on `Limit > 0`. Send `"max"`, `"none"`, `null` or
`[100]` → 0 → **the cap is disabled** and the whole registry is serialized. Same shape at
`Handlers/Actor/QueryHandler.cpp:128`/`:149`, `AnimGraphSearchHandler.cpp:103`,
`EnvironmentHandler.cpp:1319`, `RecorderHandler.cpp:63`.

**C14 — audits that return the wrong verdict.** `geometry.audit_static_meshes` /
`checks` + `excludeChecks` (`PinWrightGeometry/.../MeshAuditHandler.cpp:89`/`:94`, read `:172`/`:195`):
a bare string → `nullptr` → `Selected = DefaultCheckMask()`, documented as *"everything except
thin_shell"* — so asking for the one non-default check runs everything **but** it and still returns
`pass`. `excludeChecks: "inverted"` → not excluded → a ruled-out check can flip `pass` to false. The
comment at `:183-185` states the intent — *"A `checks` array that silently ran nothing would be
indistinguishable from a folder of correct meshes — the exact failure this verb exists to make
impossible"* — and the guard it describes covers unknown **ids**, not wrong **shape**. Identical at
`level.audit` (`Handlers/Level/LevelAuditHandler.cpp:443`/`:449-451`, via `AuditRpcCollectStrings`
`:99-113`). Both echo the resolved check set, so both are detectable by a careful caller.

**Also confirmed, less severe:** `system.inspect.search_classes` / `moduleFilter`
(`ClassSearchHandler.cpp:115`, read `:138`, gate `:177`) returns all modules;
`blueprint.graph.get_graph_connections` / `nodeIds`
(`BlueprintGraphInspectionHandler.cpp:623`, read `:634`) returns every edge in the graph while the
verb registered directly below it takes a singular `nodeId` as `"string"`;
`audio.authoring.describe_metasound` / `nodeIds` (`AudioAuthoringHandler.cpp:2855`, read `:2884` via
`GetStringSet`) dumps the entire graph at full vertex detail **and** suppresses the
`nodeCount`/`returnedNodeCount` summary header (`MetaSoundDumpBuilder.cpp:324-328`) because
`IsDefault()` is now true — byte-identical to an unfiltered dump, zero signal, producing exactly the
readback overflow the param exists to prevent; `widget.screenshot_designer` / `showOnly` + `hide`
(`WidgetDesignerScreenshotHandler.cpp:95-96`, helper `:37-49`) captures unisolated and **omits** the
`overridesApplied` block entirely; `material.authoring.set_material_instance_base_property_overrides`
/ `clear` (`MaterialAuthoringHandler.cpp:2740`, read `:2808`) clears nothing, then commits and saves;
`insights.start_session` / `channels` (`InsightsHandler.cpp:70`, read `:73`) traces the default
channel set and omits the `channels` echo, surfacing hours later as a `.utrace` missing its data;
`insights.set_channels` / `enable` + `disable` (`:122-123`) is a pure no-op reported as success.

### The structural rule the instances share

Every confirmed instance **uses the value accessor as the presence test** — `Ctx.GetArray()` /
`TryGetArrayField()` returning `nullptr`, `Ctx.GetStringSet()` returning an empty set, or
`Ctx.GetString().IsEmpty()`. In all three, *"wrong type"* and *"absent"* are the same observation,
and the absent branch is by design permissive: no filter, default set, skip the step. Every safe
handler either tests presence with type-independent `HasField` and validates the value separately,
or calls `HasTypedField` explicitly. That is exactly what a dispatcher-level type check enforces
uniformly, and it is why the fix belongs above the handlers rather than in 1,188 of them.

## Counter-examples: the codebase already solves this, one verb at a time

- **`system.run_tests` is the model implementation.** `Handlers/System/SystemControlHandler.cpp:78-92`
  hand-rolls precisely the missing check: `HasTypedField<EJson::String>` for `filter`/`test`,
  `HasTypedField<EJson::Array>` for `tests`, with honest messages. It exists on the verb where the
  cost is highest — a dropped `filter` falls through to `Automation RunAll` (`:489-491`), 4,500+ tests.
- `Handlers/Debug/TraceAnalysisHandler.cpp:36-50` (`GetNumberAlias`) gates on
  `HasTypedField<EJson::Number>` and otherwise returns the caller's default. This is the shape the
  510 silent number reads should have.
- `Handlers/Render/AnnotatedCaptureHandler.cpp:268-272`, `AudioSynthGenerateHandler.cpp:1464-1465`,
  `Spatial/MeasureHandler.cpp:265-402`, `Level/LevelStructureHandler.cpp:76`/`:493` — all
  `HasTypedField`.
- `FHandlerContext::ReadFieldProjection` (`HandlerContext.cpp:345-363`) accepts `fields` as an array
  **or** a bare string — one shared both-shapes parser, reused by 13 verbs.
- `McpActorUtils::CollectActorNames` (`Utils/ActorUtils.cpp:252-290`) dual-accepts
  `actorNames` array / `actorName` scalar and refuses a bare-string `actorNames` rather than dropping it.
- Every destructive bulk verb hand-refuses: `AssetWorkflowHandler.cpp:173, 252, 350, 487`
  (`asset.bulk_delete` / `bulk_rename` / `source_control_*`) and
  `SourceControlHandler.cpp:48-77`. **The required-array population is largely safe precisely
  because emptiness is fatal there.** The dangerous population is the *optional* array/object params
  where empty means "skip" — 522 of the 606 array/object declarations (86%).
- `views` on the four render verbs (`RenderHandler.cpp:596-597`, `CameraFrameHandler.cpp:820-821`)
  is safe **by accident of structure**: presence via type-independent `HasField`, then the value is
  validated, so an array is refused rather than dropped.

Two independent authors wrote two independent both-shapes parsers for the same dispatcher-shaped
problem, and `asset.search`'s fix (`AssetManageHandler.cpp:1425-1462`) is a third — roughly 30 lines
of hand-rolled parsing plus two new refusal branches, **for one parameter on one verb**. That is the
per-verb price, and it only ever covers the param someone happened to trip over.

## Fix options, priced

### Option A — dispatcher-wide type gate (recommended, staged)

Add a type check to `ValidateHandlerParams` after the unknown-name pass. It lands in one function
that already runs for every verb.

**Design decisions, and they are not close calls:**

1. **Coerce vs refuse must be split by direction, not applied uniformly.** A blanket strict gate
   breaks every currently-working caller who sends `"limit": "100"` or `"force": "true"` — routine
   LLM-client output, and *lossless*. Refuse only the **lossy** directions:
   array/object → scalar, non-numeric string → number, array/object → boolean, scalar → array/object,
   and `null` for any declared type. Keep the lossless coercions (numeric string ↔ number,
   `"true"`/`"false"` ↔ boolean, number/bool → string). This kills every confirmed instance above
   while breaking no caller who is working today. A full JSON-Schema-strict gate is both more
   expensive and more damaging.
2. **`any` accepts anything; unions accept any member.** The grammar already exists —
   `IsSupportedTypeExpr` (`TestContractConsistency.cpp:203-224`) parses `|` unions over the eight
   atomic tokens. **Lift it out of the test into shared production code** so the gate and the test
   read one grammar, and normalize `bool`→`boolean`, `integer`→`number` in the same pass.
3. **Resolve per matched wire name, not per spec.** `FParamAliasSpec` carries its own `Type`
   (`ParamSpec.h:11`) precisely because a typed alias differs from its canonical — e.g.
   `blueprintCandidates` is `array` where `path` is `string`. The gate must check the type of the
   alias that actually matched.
4. **New error code `PARAM_TYPE_MISMATCH`.** `INVALID_PARAM_TYPE` / `INVALID_PARAMETER_TYPE` are
   already taken by domain-type checks (`ErrorCodes.h:597`/`:599`, emitters in Niagara / material /
   PCG); reusing them makes the wire axis indistinguishable from the domain axis. The message must
   name the param, the declared type, and the type actually received.

**Staging — a warn stage is not optional.** The declarations themselves have never been checked
against handler behaviour: 62 params spell `bool`, 118 spell `integer`, 25 carry unions authors
added by hand, and nothing has ever verified that a param declared `string` is only ever sent as one.
Flipping straight to refusal would turn every *mis-declared* param into a live rejection with no
warning. So:

- **Stage 1 (warn).** Detect the mismatch, log it as `ValidateHandlerParams` already logs unknown
  params (`:145-148`), **and surface it to the caller** — a non-fatal `warnings` array on the success
  response. Caller-visibility is the whole point: today's `LogJson` error never leaves the editor.
  There is no dispatcher-level warnings channel yet (59 handlers roll their own), so this is the one
  piece of new plumbing. Ship, collect, fix the declarations the warnings expose.
- **Stage 2 (refuse).** Flip the lossy directions to `PARAM_TYPE_MISMATCH`.

Per-verb opt-in is the wrong staging axis: 1,188 opt-ins is not a migration, it is a permanent split,
and the flag would never get set on the verbs that need it.

**Price.** Gate + shared type-expr parser + alias resolution + null handling: ~150-200 lines of
production code in `RpcDispatcher.cpp` and one new shared header, plus the warnings channel and a
test file. The real cost is Stage 1's dwell time and the declaration cleanup it surfaces — which is
the point: it finds the wrong declarations without a 1,188-verb hand audit.

### Option B — make the accessors refuse (cheaper, but only a slice)

Moving the decision to the read site sounds cheaper and mostly is not. `GetString` returns `FString`
and has **no failure channel**; making it send an error would double-respond when the handler then
sends its own, and adding a refusing variant means migrating 2,700+ call sites. Merely making it
return the *default* instead of the coerced value converts silent-wrong into silent-default — no
better for any instance above.

**But two changes inside `HandlerContext.cpp` are genuinely cheap, zero-compat-risk, and worth
shipping first as an independent down payment — roughly 15 lines, no call-site changes:**

- **`RequireInt` `:148`** — add `HasTypedField<EJson::Number>` to match its four siblings. 20 call
  sites, every one already has a failure path. **This alone closes C2, C5 and C7** (the three
  wrong-element / collapsed-range writes).
- **`RequireString` `:88`** — add an explicit type check. Array/object already return false via the
  empty-string path, so control flow is unchanged; the caller stops being told
  *"Missing required string field"* about a field that was present. 337 call sites, none affected.

Both cover only the required half of the surface (1,693 params) and only the lossy directions. The
optional half — 3,725 params, where every confirmed read-side instance lives, because "wrong type"
and "not asked for" are the same observation there — is unreachable without Option A.

**Recommendation: ship Option B's two-accessor patch immediately, then Option A staged.**

## Why this is its own ticket

Three tickets already exist on the **name** axis of the declared-param mechanism:

- `B-declared-param-guard-blind-to-helpers` (OPEN, High) — the coverage test stops at the handler
  body, so reads one call frame away are invisible.
- `B-declared-param-guard-blind-to-nested-keys` (OPEN, High) — keys nested inside an object/array
  param are validated by nothing in either direction.
- `B-declared-param-guard-blind-spots` (IN-REVIEW, High) — the guard matches one read shape of four.

All three ask *"is this key declared, and is it read?"* This ticket asks *"does the value's shape
match what the declaration says?"* — a different field of `FParamSpec` (`Type`, not `Name`), a
different failure (a coerced value vs an unreachable one), a different fix site (a new branch in
`ValidateHandlerParams` vs the coverage test's scanner), and a different blast radius (a wire-contract
change across 1,188 verbs vs a test-baseline change). Folding it into any of them would bury a
compatibility-affecting dispatcher change inside a test-coverage ticket.

It is also **not** a duplicate of `B-asset-search-array-class-filter-silently-dropped` (IN-REVIEW).
That ticket fixed one param on one verb by hand and correctly identified the dispatcher root in
passing; its fix even widened the declared type to `array|string`, adding more prose to a field
nothing machine-reads. This ticket is that root, measured, with the fix priced. When it lands, the
`asset.search` hand-parse becomes redundant rather than wrong.

The `null` vector is a presence defect rather than a type defect, but it lives in the same function,
is closed by the same edit, and its payload is the same wrong value — splitting it would put two
halves of one guard in two work items.

## Severity

**Critical**, and the counter-argument is stated so a retriage pass can push back.

The rubric's Critical band is *"a write that corrupts or loses asset data."* C1 loses every property
value on a replaced widget with no echo of the flag that caused it. C2 deletes the wrong wheel and
does not echo the index. C3 writes fabricated team names and compiles + saves them. C4 discards an
applied material edit at a site that reports nothing about the save, and is a family of 149. All are
reachable through the dispatcher, all answer `success`, and the most plausible triggering value —
JSON `null` for an unset optional — is what a serializer emits by default. Reach is maximal: the gate
runs on every call in every session, so the rubric's reach modifier applies no downward adjustment.

**Against Critical:** each instance is individually a per-verb bug, and this board rates comparable
per-verb silent no-op writes (`B-data-table-row-values-silent-drop`,
`B-property-set-object-array-silent-null`) as High. A reader who holds that a root-cause ticket
should not outrank its instances would rate this High. **Against Medium:** the brief that opened this
investigation anticipated Medium if the one known instance were already fixed. It is fixed, and that
reading did not survive the measurement — fourteen further instances are confirmed by source trace,
eight of them writes.

## History
- `#1-dispatcher-never-reads-declared-type` `OPEN` reporter — Verified `FParamSpec::Type` has exactly two production reads (`MarkdownHelpers.cpp:24` wiki rendering, `RpcDispatcher.cpp:107` error-message text) and `FParamAliasSpec::Type` exactly one (docs); `ValidateHandlerParams` (`RpcDispatcher.cpp:91-163`) checks names and requiredness only, and `BuildCallToolDescriptor` (`McpRequestCore.cpp:239-267`) exports no per-verb schema. The field is nonetheless treated as enforced by `PinWright.infra.contract.ParamTypes.ValidTypeNames`, by `RequiredParamGate::SetPlaceholder` (whose own comment records the blindness), and by `docs/rpc-design.md:107`. Measured the surface off the live-generated wiki (2026-08-27, 1,363 method pages): 5,418 params / 1,188 verbs, of which 2,813 (52%, a floor) are read only through a silent-default accessor and 932 verbs (78%) carry at least one; only 65 of 473 `Require*` call sites type-check. Method calibrated against the known `classFilter` case with a measured 13% one-directional false-positive rate. Fourteen caller-facing instances confirmed by source trace, eight of them writes that lose, fabricate or destroy asset data; also confirmed that UE 5.8's `HasField` accepts `EJson::Null`, so a required param sent as `null` passes the presence gate and lands as `false`/`0`/`""`. Filed with both fix options priced.
