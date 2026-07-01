---
id: E-audio-create-metasound-name-path-vs-assetpath
title: "audio.authoring create verbs split the destination into name+path while every operate/read verb in the namespace takes a single assetPath — the create→author→validate→describe chain flips slot spelling mid-build, forcing per-method param-table reads"
status: OPEN
severity: Low
category: ergonomic
tags: [audio, authoring, metasound, param-alias, create_metasound, name, path, assetpath, split, drift]
blockedBy: [E-material-create-combined-assetpath-split]
encounters: 3
lastSeen: 2026-06-23T10:08:23Z
---

# Within `audio.authoring.*`, the `create_*` verbs want `name`+`path` (split); every operate/read verb wants a single `assetPath`

Same param-name-drift family as the create-verb split-slot tickets
`E-material-create-combined-assetpath-split` (OPEN, the direct analog in
`material.authoring`), `E-level-create-name-path-alias` (OPEN),
`E-geometry-create-name-vs-actorname` (OPEN), `E-volume-create-name-vs-volumename`
(OPEN), and the cross-verb `E-asset-path-vs-assetpath-list-drift` (OPEN), plus the
DONE alias precedents (`E-blueprint-param-name-path-vs-assetpath`,
`E-widget-asset-path-alias-drift`, `E-material-editor-param-name-drift`) — but here
it surfaces **inside `audio.authoring`**, as the slot spelling flipping between the
*create* verb and every subsequent verb in the same MetaSound build chain.

The split is per-verb-role, with no cross-alias:
- **Create verbs take `name` (req) + `path` (opt).** `create_metasound`
  (`AudioAuthoringHandler.cpp:696-697`: `RPC_PARAM_REQ("name", ...)` +
  `RPC_PARAM_OPT("path", "...default /Game/Audio/MetaSounds")`), and the whole
  sibling create family on the same pattern: `create_sound_cue` (:347-348),
  `create_sound_class` (:1246-1247), `create_sound_mix` (:1453-1454),
  `create_attenuation` (:1690-1691), the dialogue/reverb/effect-chain/submix
  creators (:1913, :1972, :2134, :2198, :2319, :2354). None carries an `assetPath`
  alias.
- **Every operate/read verb on that same asset takes `assetPath` (req).** For the
  MetaSound surface specifically: `add_metasound_input` (:975, plus `inputName`
  :976), `add_metasound_output` (:1067, plus `outputName` :1068),
  `set_metasound_default` (:1146, plus `inputName` :1147), `add_metasound_node`
  (:892), `connect_metasound_nodes` (:775), and the read verbs
  `describe_metasound` / `validate_metasound` (:2402 etc.) — all
  `RPC_PARAM_REQ("assetPath", ...)`, no `name`/`path` alias.

So a single authored MetaSound build —
`create_metasound {name, path}` → `add_metasound_input {assetPath, inputName}` →
`add_metasound_output {assetPath, outputName}` → `set_metasound_default {assetPath,
inputName}` → `add_metasound_node {assetPath}` → `connect_metasound_nodes
{assetPath}` → `validate_metasound {assetPath}` → `describe_metasound {assetPath}` —
changes the destination-slot spelling exactly once, at the boundary between the
create call and everything after it. The asset slot is `name`+`path` for one call,
then `assetPath` for the remaining seven. CLAUDE.md's "camelCase and snake_case
aliases" rule does not cover this: `name`+`path` (split) vs `assetPath` (one
combined path) is a *shape* difference, not a casing variant.

## Repro (verbatim, from the audited MS_MenuBeep task)

Struggle-audit of a `audio.authoring.validate_metasound` seed task: build a
procedural UI beep MetaSound (create `MS_MenuBeep` at `/Game/Audio/MetaSounds` →
`Frequency` Float input default 440 → `Out` Audio output → Sine osc + Multiply gain
nodes → wire Sine.Audio→Multiply.PrimaryOperand and Multiply.Out→Out →
`validate_metasound` → `describe_metasound` readback). **Outcome: done** — every
one of the 12 calls passed first try and `validate_metasound` returned `valid:true`
with empty diagnostics.

The task succeeded *because the agent paid the avoidance tax up front*: it read each
method's actual param table rather than carrying one mental model across the chain.
That up-front per-method reading is the friction this ticket files — the drift did
not produce a hard error here (unlike the analog material ticket's
`MISSING_REQUIRED_PARAM 'name'` round-trip) precisely because the agent pre-empted
it, so the cost shows up as discovery overhead, not a failed call.

Friction note (verbatim): "parameter naming was inconsistent across the
audio.authoring surface (create_metasound uses name/path while add_input/output/connect
use assetPath/inputName/etc, and its own Notes example showed stale
asset_path/node_class/from_node params), so I relied on each method's actual param
table." Zero blocked progress; pure guessability / per-method-table-read overhead,
the exact shape of the precedent create-verb-split tickets.

## Distinct from the docs ticket

`E-metasound-node-add-docs-misleading` (OPEN, judge-filed; its `#2` quotes this same
friction note) is the **docs/discovery** angle — the `docs/wiki-src/audio.authoring.md`
"Typical sequence" example uses stale snake_case params (`asset_path`, `node_class`,
`from_node`) that contradict every live handler, so the example mis-teaches the whole
create→connect chain. That ticket's fix is to rewrite the wiki example. **This ticket
is the live-API ergonomic axis that survives a perfect docs fix:** even with a
correct, current example, the actual slot spelling still *changes* between
`create_metasound` and the rest of the surface, so a caller must either memorize the
boundary or re-read per method. Correcting the example removes the wrong-spelling
trap; aliasing the slots removes the boundary itself.

## What it should do

Two non-exclusive options, both preserving existing `name`+`path` callers (mirror
`E-material-create-combined-assetpath-split`'s proposal):

1. **Accept a combined `assetPath` on the `audio.authoring.create_*` family** and
   split it server-side: when `assetPath` (or a `path` ending in an object name) is
   supplied and `name` is absent, derive `name` = leaf and `path` = parent folder
   (`FPackageName::GetLongPackagePath` / `GetShortName`). This makes the create verb
   accept the same single-`assetPath` shape every operate/read verb in the namespace
   already takes, closing the last spelling boundary in the build chain. Apply to the
   create family cited above (`:696`, `:347`, `:1246`, `:1453`, `:1690`, `:1913`,
   `:1972`, `:2134`, `:2198`, `:2319`, `:2354`).
2. If splitting is undesirable, annotate the create verbs' `name`/`path` specs (or add
   an `assetName` alias) to make the "split, do not pass a combined `assetPath`"
   contract explicit in the schema — the dispatcher `FParamSpec` alias machinery from
   `E-blueprint-param-name-path-vs-assetpath #4` is the vehicle, same as the sibling
   tickets.

Do not solve this by making `name` optional per-handler without the split (that leaves
the combined-path caller silently building under a wrong/empty name).

## Secondary note (not separately filed): describe_metasound readback size

The same task's `describe_metasound` final readback (and a mid-build discovery
`describe_metasound` used to learn real pin names) "exceeded the display limit and had
to be read off disk twice." This is the known-large MetaSound graph JSON
(`F-rpc-audio-describe-metasound #3` notes ~77 KB for a real asset) and recovered
cleanly via disk, so it is not filed here as a separate defect; it belongs to the
broad oversized-readback class (`E-asset-dump-oversized-fields`,
`F-rpc-property-omit-oversized-opt-in`, `E-graph-connections-pagination`) and is noted
only as corroborating signal that the MetaSound describe surface would benefit from a
header-only / paginated mode for the discover-pin-names use case.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the
  `audio.authoring.validate_metasound` "MS_MenuBeep" procedural-beep build task
  (focus `audio.authoring.validate_metasound`, outcome `ergo`/done — all 12 calls
  passed first try, `validate_metasound` returned `valid:true`, judge filed
  `E-metasound-node-add-docs-misleading` for the stale wiki example). Distinct
  PROCESS angle from that docs ticket: the **live-API** destination-slot drift inside
  `audio.authoring` — `create_metasound` declares `RPC_PARAM_REQ("name", ...)` +
  `RPC_PARAM_OPT("path", ...)` (`AudioAuthoringHandler.cpp:696-697`) while every
  operate/read verb in the same MetaSound build chain
  (`add_metasound_input` :975, `add_metasound_output` :1067, `set_metasound_default`
  :1146, `add_metasound_node` :892, `connect_metasound_nodes` :775, `describe`/
  `validate` :2402+) takes `RPC_PARAM_REQ("assetPath", ...)` with no cross-alias. The
  slot spelling flips once, at the create→operate boundary, forcing the agent to read
  each method's param table individually (friction note verbatim: "parameter naming
  was inconsistent across the audio.authoring surface (create_metasound uses name/path
  while add_input/output/connect use assetPath/inputName/etc) … so I relied on each
  method's actual param table"). No hard error this run because the agent pre-read the
  tables; cost is pure guessability/discovery overhead. Direct analog of
  `E-material-create-combined-assetpath-split` (OPEN) one namespace over; sibling of
  the create-verb-split tickets `E-level-create-name-path-alias`,
  `E-geometry-create-name-vs-actorname`, `E-volume-create-name-vs-volumename` (all
  OPEN) and the cross-verb `E-asset-path-vs-assetpath-list-drift` (OPEN). Fix:
  accept+split a combined `assetPath` on the `audio.authoring.create_*` family, or
  alias/document the split contract via the dispatcher `FParamSpec` machinery from
  `E-blueprint-param-name-path-vs-assetpath #4`. Secondary, not separately filed:
  `describe_metasound` readback exceeded the display limit and was read off disk twice
  — known-large graph JSON (`F-rpc-audio-describe-metasound #3`, ~77 KB), recovered
  cleanly, belongs to the oversized-readback class.
- `#2-engine-hum-replay-same-create-operate-boundary` `OPEN` reporter — Second
  independent task exhibiting the same live-API destination-slot drift inside
  `audio.authoring`. Struggle audit of an `audio.authoring.create_metasound`
  "MS_EngineHum" procedural engine-hum build (focus `audio.authoring.create_metasound`,
  outcome `ergo`/done): create `MS_EngineHum` at `/Game/Audio/MetaSounds` → `RPM`
  Float input default 900.0 → `MasterGain` Float input default 0.8 → `Out` Audio
  output → Sine osc (`UE.Sine.Audio`) + Multiply gain (`UE.Multiply.Audio` by Float)
  nodes → pre-wire `describe_metasound` to read real GUIDs/pins → wire
  Sine.Audio→Multiply.PrimaryOperand and Multiply.Out→Out → `validate_metasound`
  (returned `valid:true`, no diagnostics) → final `describe_metasound`. All 13
  execute calls passed first try. Same create→operate boundary: the build issued
  `create_metasound {name, path}` then `add_metasound_input {assetPath, inputName}` ×2,
  `add_metasound_output {assetPath, outputName}`, `set_metasound_default {assetPath,
  inputName}` ×2, `add_metasound_node {assetPath}` ×2, `connect_metasound_nodes
  {assetPath}` ×2, `validate_metasound {assetPath}`, `describe_metasound {assetPath}`
  — the destination slot is `name`+`path` for exactly the first call and `assetPath`
  for the remaining eleven, confirming the per-verb-role split cited in `#1`
  (`AudioAuthoringHandler.cpp:696-697` vs :975/:1067/:1146/:892/:775/:2402+). No hard
  error again because the agent pre-read each method's param table rather than carrying
  one mental model; cost is the same pure guessability/per-method-table-read overhead.
  Friction note (verbatim): "create_metasound's wiki page lists params as name/path but
  its example body shows asset_path (inconsistent) and the input/output/node calls use
  assetPath/inputName camelCase, so param-name conventions are not uniform across the
  namespace." (The `name/path` vs `asset_path` *example* mismatch within
  `create_metasound`'s own page is the docs angle already logged at
  `E-metasound-node-add-docs-misleading #3` from this same task; this entry records the
  **live-API** `name`+`path`→`assetPath` create→operate boundary, which survives a
  docs-only fix.) Same fix: accept+split a combined `assetPath` on the
  `audio.authoring.create_*` family, or alias/document the split via the dispatcher
  `FParamSpec` machinery from `E-blueprint-param-name-path-vs-assetpath #4`.
- `#3-defer-to-material-umbrella` `OPEN` developer — Deferred behind
  `blockedBy: [E-material-create-combined-assetpath-split]`. This is 1 of ≥5
  identical-shape OPEN create-verb tickets ("create wants split `name`+`path`,
  every operate/read verb wants one combined `assetPath`"): the self-declared direct
  analog `E-material-create-combined-assetpath-split` (OPEN) plus
  `E-level-create-name-path-alias`, `E-geometry-create-name-vs-actorname`,
  `E-volume-create-name-vs-volumename` (all OPEN). The requested option-1 fix is NOT
  the cheap dispatcher synonym-alias the cited `E-blueprint #4` precedent shipped
  (`ParamAliasUtils::MakeAliasParamSpec` makes `assetPath`/`path`/`materialPath`
  interchangeable for ONE slot, no string-splitting); it asks to DERIVE two slots
  (`name` + parent folder) from one combined path via
  `FPackageName::GetLongPackagePath`/`GetShortName`, new parsing + folder-vs-asset
  ambiguity, applied across ~11 audio create verbs — a fleet-wide create-surface
  convention decision, not a per-namespace tweak. Both audited builds here (`#1`
  MS_MenuBeep 12 calls, `#2` MS_EngineHum 13 calls) finished `outcome=done` with
  EVERY call passing first try and `validate_metasound` returning `valid:true` — zero
  hard errors (unlike the material analog's `MISSING_REQUIRED_PARAM 'name'`
  round-trip), so the residual cost is pure discovery overhead, and the discovery
  half is already in-flight on the IN-REVIEW docs ticket
  `E-metasound-node-add-docs-misleading` (its `#4` added a `name`+`path` vs
  `assetPath` shape-split note to `audio.authoring.md`). The split-vs-combined
  convention should be decided once on the OPEN umbrella analog
  `E-material-create-combined-assetpath-split` and then applied uniformly to the whole
  create-* surface, rather than band-aided here. Gate: this ticket unblocks when
  `E-material-create-combined-assetpath-split` reaches DONE/WONTFIX (the convention is
  then settled — follow the same pattern for the audio create family, or close in
  step). Released the `fuzz3` claim.
