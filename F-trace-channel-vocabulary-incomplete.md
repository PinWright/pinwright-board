---
id: F-trace-channel-vocabulary-incomplete
title: "Every tracing verb speaks the same four-token channel vocabulary — visibility / camera / worldstatic / worlddynamic — hand-copied into four separate if-chains, so `Pawn` is not a legal value anywhere and the one question level-building actually asks (\"does this block a player?\") cannot be put to the RPC surface at all; on one level 12 components block Visibility while ignoring Pawn, so a Visibility trace is not a proxy"
status: OPEN
severity: Medium
category: feature
tags: [spatial, raycast, raycast_screen, ground_actors, ground_instances, verify_grounding, collision-channel, pawn, ECollisionChannel, custom-channel, duplicated-table, missing-parameter-values, level-building]
encounters: 1
lastSeen: 2026-08-29T21:55:00+03:00
---

# Four tokens, four copies, and the one channel gameplay cares about is not among them

Every collision query this plugin issues resolves against a channel chosen from the same list of
four strings. The list is written out **four separate times**, all at HEAD `962275fa`, all under
`Private/Handlers/Spatial/`:

| copy | function | lines | consumed by |
|---|---|---|---|
| `RaycastHandler.cpp` | `RaycastMapChannel` | `:72-96` (tokens `:75`, `:80`, `:85`, `:90`) | `spatial.raycast` (`:248`), doc `:263-264`, default `:343`, error `:347-349` |
| `RaycastScreenHandler.cpp` | `RaycastScreenMapChannel` | `:44-52` | `spatial.raycast_screen` (`:74`) |
| `GroundPlacementHandler.cpp` | `GroundRpcMapChannel` | `:125-133` | the `surface.channel` schema on the three ground verbs |
| `GroundPlacementUtils.cpp` | inline if-chain in `ParseSurfaceJson` | `:314-328` | `spatial.ground_actors`, `spatial.verify_grounding`, `spatial.ground_instances` |

Accepted, in all four: `visibility`, `camera`, `worldstatic`, `worlddynamic`. Anything else is
`INVALID_PARAMS`. `RaycastHandler.cpp:70-71` states the judgement out loud —

> "Only the channels a caller realistically raycasts against are accepted; anything else is a caller
> error."

— and `RaycastScreenHandler.cpp:42-43` says the second copy exists "so both spatial verbs speak one
channel vocabulary". The intent is one vocabulary; the implementation is four hand-maintained
copies of it, and the vocabulary itself is four of the engine's eighteen-plus channels.

**`ECC_Pawn` appears exactly once in the whole plugin**, and not on a trace:
`Private/Utils/CollisionSummaryUtils.cpp:64`, feeding `editor.set_view_mode`'s collision report.
`GameTraceChannel` appears **zero** times, so a project-defined channel is unreachable by any
spelling. There is no profile-based variant either — `ByProfile` has zero occurrences in `Source/`.

## Why this is not a naming nit

Build: the editor's **10:44** PinWright build; source read at HEAD `962275fa`. Level
`/Game/Maps/PW_VegetationTest`. Receipts: `X:/src/unreal/EAContentExamples58/dev/grasscollide/`.

**A Visibility trace is not a proxy for a Pawn trace on a real level.** After a pass that made
groundcover walkable while keeping it a valid ground-probe surface, the level's component census
(`out/census_after.json`, 147 collision-bearing rows) reads:

- **12** components at `QUERY_AND_PHYSICS` with Visibility `Block` and Pawn `Ignore` — deliberately
  divergent, because `spatial.ground_actors` / `spatial.verify_grounding` and the shipped exclusion
  fixture in `Docs/map/vegetation-zone-f.md` all depend on those components still answering a
  Visibility probe.
- **71** more at `NO_COLLISION` whose response container still reads Pawn `Block`, which is a
  different trap (`E-get-components-omits-collision-state`) but the same lesson: the channel you
  trace is the channel you learn about.
- **83 of 147** rows differ between the two channels.

So the divergence is not pathological content — it is what you get the moment anyone tunes what a
player can walk through, and it is invisible to every trace this plugin can issue.

**The pass had to leave the RPC surface to ask its question.** `python.execute` with
`SystemLibrary.capsule_trace_multi_by_profile` on the stock `Pawn` profile, at 8 frozen sites plus
120 sweeps over 8 ground-following traverses. Nothing typed could have been used, at any channel.

## Ask

**Stop hand-listing the channels; ask the engine.** UE already owns the mapping from a display name
to a channel, and it is the same table the project's `Config/DefaultEngine.ini` writes:

    C:/UE_5.8/Engine/Source/Runtime/Engine/Classes/Engine/CollisionProfile.h:241
        int32 ReturnContainerIndexFromChannelName(FName& InOutDisplayName) const;
    C:/UE_5.8/.../CollisionProfile.h:245
        ECollisionChannel ConvertToCollisionChannel(bool TraceType, int32 Index) const;
    C:/UE_5.8/.../CollisionProfile.h:281
        TArray<FName> ChannelDisplayNames;

One shared helper over `UCollisionProfile::Get()` gives every tracing verb the stock channels
(`Pawn`, `PhysicsBody`, `Vehicle`, `Destructible`, `Visibility`, `Camera`, `WorldStatic`,
`WorldDynamic`) **and** every `ECC_GameTraceChannel*` the host project has named, without the plugin
knowing any of them, and it cannot drift from the project's ini. The error message can then
enumerate the channels this project actually has rather than a fixed four.

Two things that come with it:

1. **One copy, not five.** The fix must collapse the four existing chains, not add a fifth — and
   `F-component-collision-channel-write` needs the same name -> channel map for its
   `channelResponses` keys, so it is the fifth caller waiting to happen.
2. **A `profile` alternative is worth the same helper.**
   `UCollisionProfile::GetChannelAndResponseParams` (`CollisionProfile.h:213`) turns a profile name
   into a channel plus a response container, which is what "trace as a pawn would" actually means
   and what the measured workaround used. That is strictly more expressive than a channel name and
   costs one more call to the same object.

## Scope note: this does not, by itself, answer the question

A Pawn *line* trace is still not a capsule sweep, and a walking character is a capsule. The two
tickets are separable — a fixer can land either alone — but neither alone answers "can a player walk
here". See `F-spatial-swept-shape-query`.

## Not a duplicate of

- **`F-spatial-swept-shape-query`** (OPEN, filed from this pass) — its partner, split deliberately.
  This is the channel vocabulary of the queries that exist; that is the absence of a swept-shape
  query at all. Different files, different sizes: this is one shared helper replacing four
  if-chains, that is a new verb. Whoever takes one should read the other.
- **`F-spatial-raycast-no-batch-multi-origin`** (OPEN, Low) — `spatial.raycast` being one ray per
  call. It enumerates the verb's arguments, lists `channel` among them, and never questions the
  enum. Arity, not vocabulary.
- **`B-raycast-screen-lacks-trace-diagnostics`** (OPEN, Medium, encounters 2) — the accept-filter /
  `multiHit` / `simpleCollisionShapes` parity that landed on `spatial.raycast` and never reached
  `spatial.raycast_screen`. Feature parity between two verbs; `channel` is listed there as
  already-emitted. Note the overlap that is worth flagging to a fixer: that ticket is about the two
  verbs having drifted apart, and this one is about a table that was **copied** to stop them
  drifting. The same shared-helper move would serve both.
- **`B-trace-complex-hits-render-geometry`** (DONE, High) and
  **`B-ground-probe-hits-hull-not-render`** (DONE, High) — both about `traceComplex`, i.e. *which
  geometry* a trace resolves against. Orthogonal axis: this is *which channel*.
- **`E-ground-preset-excludes-only-foliage-actors`** (DONE, High) — added `excludeComponentClasses`
  to the surface spec. A class filter, applied after the hit; it cannot express "ask the Pawn
  channel" and would not want to.

## Severity

**Medium**, on the rubric's soft-blocker band: *"doable, but only via a documented workaround, a
source dive, or many extra calls."* The measurement was made — through `python.execute` and a
hand-written 128-sample sweep harness — so this is not the "hard blocker with no workaround" band.

**Bump-up to High declined.** The trigger is real on paper: the affected methods are
`spatial.raycast`, `spatial.raycast_screen` and all three ground verbs, which do run in almost every
level-building session. But the honest scope of the affected method is *tracing when the caller
needs a channel other than the four* — the same reading `B-property-set-object-hop-notification-noop`
used to decline this bump — and the great majority of ground probes genuinely want Visibility. The
stronger High argument, that "can a player walk here" is unanswerable, is only true jointly with
`F-spatial-swept-shape-query`, and pricing a joint impact into both tickets counts it twice in the
picker's ordering.

**Bump-down to Low declined.** Low is docs, discoverability or naming. This is a rejected valid
input: the engine has the channel, the project's ini names it, the profile table in
`Config/DefaultEngine.ini` is full of `Channel="Pawn"` rows, and the surface refuses the string.

## History
- `#1-four-tokens-four-copies-no-pawn` `OPEN` reporter — Filed from a collision-and-seating fix pass
  on `/Game/Maps/PW_VegetationTest` (editor build 10:44; plugin source HEAD `962275fa`). The user
  reported "why does grass have player collision?"; verifying the fix needed a pawn-channel query
  and none exists, so the pass dropped to `python.execute` /
  `SystemLibrary.capsule_trace_multi_by_profile` for 128 sweeps. All four channel maps were read
  line by line rather than taken from a report — `RaycastHandler.cpp:72-96`,
  `RaycastScreenHandler.cpp:44-52`, `GroundPlacementHandler.cpp:125-133`,
  `GroundPlacementUtils.cpp:314-328` — and all four accept the identical four tokens.
  `grep -c GameTraceChannel` over `Source/` is 0; `ECC_Pawn` occurs once, in
  `Utils/CollisionSummaryUtils.cpp:64`, on the `editor.set_view_mode` report rather than a trace;
  `ByProfile` occurs zero times. Dedup: searched the board for `channel`, `ECollisionChannel`,
  `worldstatic`/`worlddynamic`, `ECC_`, `raycast`, `walkab`, "player walk", `navmesh`, and every
  `F-spatial-*` / `B-raycast-*` / `B-trace-*` file. Nothing questions the channel set; the
  navigation tickets on this board (`B-level-build-navigation-no-completion-signal`,
  `B-navigation-rebuild-navigation-no-completion-signal`, `B-nav-agent-properties-restored-on-registration`,
  `B-nav-link-proxy-appends-default-link`) are all navmesh build/config defects, none a query. One
  premise from the field report was corrected before filing: it said "18 components blocked
  Visibility while ignoring Pawn"; the census (`dev/grasscollide/out/census_after.json`) says **12**,
  with 83 of 147 rows differing between the two channels once the 71 disabled-collision rows are
  included. The corrected figure is used above.
