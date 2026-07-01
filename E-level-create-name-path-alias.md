---
id: E-level-create-name-path-alias
title: "level.create / level.structure.create_level reject the natural name param; only levelName/levelPath accepted"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [level, param-alias]
---

# level.create rejects the natural `name` param

Same class of friction as `E-blueprint-param-name-path-vs-assetpath` (DONE),
`E-widget-asset-path-alias-drift` (DONE), `E-material-editor-param-name-drift`
(DONE), and `E-geometry-create-name-vs-actorname` (IN-REVIEW — the closest
precedent: a per-namespace synonym-alias for a create-verb name slot, fixed
independently), but in the `level.*` namespace none of those sweeps touched. The
`level.create` handler declares `levelName` (opt) and `levelPath` (opt) as its
only two params (`LevelHandler.cpp:344-345`, both `RPC_PARAM_OPT`) with **no
aliases**. A caller who reaches for the obvious generic name `name` — the exact
name the task brief used, and the name many sibling "create" RPCs expose — eats
an `UNKNOWN_PARAMS` round-trip before guessing the right one.

**Repro:** `level.create {name:"MRQTestMap", path:"/Game/Maps/MRQTestMap"}`
→ `[UNKNOWN_PARAMS] Unknown parameter(s) for 'level.create': [name, path]. Valid
parameters: [levelName, levelPath].` Retry with
`level.create {levelName:"MRQTestMap"}` (a leaf starting with `/` is used
verbatim as the package path) succeeds.

Scope note — **`name` only, NOT `path`.** The cheap synonym alias here is `name`
→ `levelName` (an unambiguous leaf/short-name slot, exactly the
`E-geometry-create-name-vs-actorname` shape). Do **not** alias `levelPath` with
`path`: on `level.create`, `levelPath` is a *full destination package path*
(`LevelHandler.cpp:345`), whereas the just-settled create-verb convention in
`Handlers/Material/MaterialCreatePathParamUtils.h` makes `path` mean a
*destination folder* (with a combined full path carried by `assetPath`, split
server-side). Advertising `path` to mean "full package path" on `level.create`
while it means "folder" on every material/audio create verb would re-introduce
exactly the cross-verb drift the create-verb sweep exists to kill — so the
combined-full-path angle is explicitly left to the
`E-material-create-combined-assetpath-split` umbrella, and this ticket fixes only
the unambiguous `name` synonym. (`level.create` already accepts a full package
path two ways without `path`: via `levelPath`, and via `levelName`/`name` when
the value starts with `/`.)

The wiki page is correct — `wiki-generated/level-create.md` documents both
`levelName` and `levelPath` — so this is purely a guessability/alias gap, not a
docs gap; reading the page first avoids the wasted call, but the natural-name
rejection is the friction.

**Fix:** Mirror the per-namespace synonym-alias helper from
`E-geometry-create-name-vs-actorname` (`Handlers/Geometry/GeometryNameParamUtils.h`,
built on `ParamAliasUtils::MakeAliasParamSpec`). Add a `name` alias to the
`levelName` slot of `level.create` and read the value via
`Ctx.GetStringFirstOf({levelName, name})` so the alias resolves end-to-end.
Standardize the alias set, not the canonical name, so existing callers keep
working. Sweep `level.structure.create_level` (`LevelStructureHandler.cpp:116-117`,
same `levelName`/`levelPath` canonicals — there `levelName` is required) for the
same `name` alias. Leave the destination-path slots (`levelPath`) unaliased.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an `mrq` task. The agent
  called `level.create {name:..., path:...}` (the brief's natural names) →
  `[UNKNOWN_PARAMS] ... [name, path]. Valid parameters: [levelName, levelPath]`;
  retried with `levelName`/`levelPath` and succeeded (1 wasted round-trip). Source:
  `LevelHandler.cpp:344-348` declares `levelName`/`levelPath` (both `RPC_PARAM_OPT`)
  with no alias fallback; body reads them at `:356-357`. Sibling of the four DONE
  path/param-alias-drift tickets, in the previously-untouched `level.*` namespace.
  The `level.create` case is the same "create"-verb destination-slot pattern that
  `E-material-editor-param-name-drift #2` deliberately left unaliased for
  `create_material` — calling it out here so the `level.*` sweep can decide
  consistently. Companion finding to `B-sequencer-create-dangling-else-hangs`
  (the same task's tool-bug, filed by the per-finding judge).
- `#2-reword-and-fix` `IN-REVIEW` developer — Rewrote the ticket to the
  `name`-synonym-only scope and implemented it. REWORD: dropped the proposed
  `path`/`assetPath`-on-`levelPath` aliases — on `level.create`, `levelPath` is a
  FULL destination package path (`LevelHandler.cpp:345`), which conflicts with the
  just-settled create-verb convention where `path` means a destination FOLDER and a
  combined full path rides `assetPath` (split server-side) —
  `Handlers/Material/MaterialCreatePathParamUtils.h`; advertising `path`=fullpath
  here would re-introduce the cross-verb drift the sweep kills, so the combined-path
  angle stays with the `E-material-create-combined-assetpath-split` umbrella. The
  remaining `name`→`levelName` part is the unambiguous synonym-alias shape already
  shipped independently by `E-geometry-create-name-vs-actorname` (IN-REVIEW), so it
  does NOT belong behind the umbrella the way the audio split ticket does. FIX: new
  `Handlers/Level/LevelNameParamUtils.h` (mirrors `GeometryNameParamUtils`, built on
  `ParamAliasUtils::MakeAliasParamSpec`) supplies `CreateNameParam(Desc, bRequired)`
  (canonical `levelName`, alias `name`) and `ResolveCreateName(Ctx, Default)`
  (`Ctx.GetStringFirstOf({levelName, name})`). Applied to `level.create`
  (`LevelHandler.cpp`: spec → `CreateNameParam(..., /*bRequired=*/false)`, body
  `LevelName` now via `ResolveCreateName`) and `level.structure.create_level`
  (`LevelStructureHandler.cpp`: spec → `CreateNameParam(..., /*bRequired=*/true)`,
  body resolves through the alias set then keeps the existing empty-name
  INVALID_ARGUMENT guard). `levelPath` left unaliased on both. The dispatcher already
  honors `FParamSpec.Aliases` for both required-param satisfaction
  (`PayloadHasParamOrAlias`) and the UNKNOWN_PARAMS known set (`AddKnownParamNames`),
  so the `name` alias resolves end-to-end with no dispatcher change. Regression test
  `Tests/World/TestLevelCreateNameAlias.cpp`: (1) static — both create verbs'
  `levelName` spec carries the `name` alias AND their `levelPath` spec does NOT carry
  a `path`/`assetPath` alias (the deliberate scope boundary); (2) end-to-end — the
  real dispatcher accepts `level.structure.create_level {name:""}` without
  UNKNOWN_PARAMS or MISSING_REQUIRED_PARAM (alias satisfied both the known-params set
  and the required-param check) and the body then rejects the blank value with
  INVALID_ARGUMENT, proving end-to-end alias resolution with no asset created.
  Reverting the alias fails the static check and the dispatch falls back to
  UNKNOWN_PARAMS/MISSING_REQUIRED_PARAM. Did not compile/run tests (a later phase does).
