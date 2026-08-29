---
id: B-game-view-suppresses-landscape-grass
title: "editor.set_game_view {enabled:true} suppresses part of the landscape grass, so an acceptance shot taken in game view understates vegetation density — game view is documented to hide editor chrome, not geometry, and the two grass show flags are provably not what changes"
status: OPEN
severity: Medium
category: bug
tags: [editor, set_game_view, game-view, landscape, grass, vegetation, capture, acceptance-shot, unattributed-mechanism, show-flags]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Game view thins the grass, and it is documented not to touch geometry

Same pose, same capture settings, one boolean apart:

| game view | `meanLuminance` | grass |
|---|---|---|
| ON | 0.4013 | **less** |
| OFF | 0.3769 | more |

**Read the direction carefully — the arithmetic misleads.** Grass darkens the frame, so the
*higher* number is the frame with *less* grass. Game view ON is 0.4013, i.e. game view ON is the
one that lost vegetation. A reader skimming the table and assuming "bigger is more" gets the
finding exactly backwards.

The verb's own summary
(`Source/PinWright/Private/Handlers/Editor/ViewportHandler.cpp:584`) promises the opposite of this:
game view "hides editor-only sprites/icons/billboards so the viewport renders like the shipped
game". Landscape grass is shipped-game geometry. If anything, a game-view frame should show *more*
of it, not less.

## What the verb actually toggles (attributed)

`ViewportHandler.cpp:638` calls `FEditorViewportClient::SetGameView(bEnabled)`, and the real
post-toggle state is read back at `:647` rather than echoed. `SetGameView`
(`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/EditorViewportClient.cpp:7221-7249`) swaps the
whole `EngineShowFlags` set between a game set and an editor set, preferring an existing saved set
over a fresh `FEngineShowFlags(ESFIM_Game)` when either the current or the last flags already claim
to be the game set (`:7229-7241`).

## The obvious cause is ruled out

The two show flags that gate landscape grass and instanced foliage are **not** part of the
game/editor difference:

- `SHOWFLAG_ALWAYS_ACCESSIBLE(InstancedGrass, ...)` —
  `C:/UE_5.8/Engine/Source/Runtime/Engine/Public/ShowFlagsValues.inl:197`
- `SHOWFLAG_ALWAYS_ACCESSIBLE(InstancedFoliage, ...)` — `ShowFlagsValues.inl:189`
- `SHOWFLAG_ALWAYS_ACCESSIBLE(InstancedStaticMeshes, ...)` — `ShowFlagsValues.inl:187`

The `FEngineShowFlags` constructor (`ShowFlags.h:181`) memsets every flag on and then selectively
disables a list — its own comment at `:390` reads *"Most flags are on by default. With the following
line we only need disable flags"* — and that disable list contains no `SetInstancedGrass`,
`SetInstancedFoliage` or `SetInstancedStaticMeshes` call for any `EShowFlagInitMode`. The
init-mode-dependent entries in that block are things like `SetEditor(...)` (`:464`) and
`SetGame(...)` (`:480`). So `FEngineShowFlags(ESFIM_Game).InstancedGrass == FEngineShowFlags(ESFIM_Editor).InstancedGrass == 1`,
and the flag swap cannot be turning the grass off.

## The mechanism was not attributed

Beyond the ruling-out above, **the cause is unknown and none is proposed here.** The measured
effect is real and reproducible; what in the game flag set, or in what `SetGameView` does besides
swapping flags, reduces the grass has not been identified.

Open questions for whoever picks this up, none of them answered:

- whether the loss is instances not drawn or instances not built (the frame is a single pose, so
  the grass build state at the moment of each capture was not separately measured);
- whether the `Game` flag itself (`ShowFlags.h:480`) changes a density or LOD path in the grass
  renderer;
- whether the effect survives a second toggle — `SetGameView` reuses a *saved* flag set when one is
  available (`EditorViewportClient.cpp:7229-7241`), which is already known on this board to make
  game view's result depend on history (see `B-game-view-reports-invariant-show-flags` and the
  `overlayWarning` at `ViewportHandler.cpp:689-702`), so the measurement may not be a pure function
  of the boolean.

## Consequence

Acceptance captures are routinely taken in game view, because that is what the verb is for. Every
one of them understates vegetation density, and nothing says so — `gameViewEnabled: true` is
accurate, the `overlayShowFlags` block is accurate, and none of it speaks about geometry. Two
frames taken with the same game-view state remain comparable, so the damage is confined to
judgements about *absolute* density: "is this zone grassed enough" answered from a game-view shot
is answered low.

**Workaround:** take vegetation-density shots with `enabled: false`, and hold game view constant
across any pair of frames being compared.

## Fix

1. Identify the mechanism before changing behaviour. If it turns out game view legitimately
   changes a grass density/LOD path, the fix is a disclosure, not a suppression.
2. Whatever the cause, this belongs in `notGovernedByGameView[]`
   (`ViewportHandler.cpp:668-687`), which is the array whose stated job is to stop the response
   reading as a blanket "the viewport is now clean" claim, and which already carries three entries
   in exactly this voice. A fourth naming vegetation density would have saved this session's
   measurement.

## Same shape as

- `B-set-game-view-does-not-suppress-spline-overlays` — game view not covering what a caller
  assumed. Opposite direction: that is chrome surviving game view, this is content not surviving it.
- `B-game-view-reports-invariant-show-flags` (OPEN, Medium) — the response's flag block does not
  describe what changed. Related, and the reason the third open question above is worth asking, but
  distinct: that ticket is about fields that cannot vary, this is about a visible change no field
  reports.
- `B-showflag-cvar-override-contaminates-capture` (DONE) — a capture dirtier than the response
  admits, closed by measuring and publishing the channel. Same remedy shape as fix item 2.

## Severity

**Medium**, by impact class: *soft blocker* — "doable, but only via a documented workaround". The
workaround is one boolean on a verb the caller is already calling, and it costs nothing beyond
knowing about it.

**Reach modifier declined.** `editor.set_game_view` runs in almost every capture session, which by
the rubric would bump this to High. Declined on a specific ground: the defect distorts a
*comparison baseline*, not a result — any two frames taken with the same game-view state are still
comparable to each other, and every automated flow this board describes holds capture settings
constant across a pair. What breaks is the narrower task of reading absolute vegetation density off
a single game-view frame. That is real, and it is what makes this a bug rather than a doc nit, but
it is not the every-frame corruption the reach bump is meant to price in.

Explicitly **not** rated High alongside `B-capture-open-level-pose-params-photograph-stale-grass`,
filed the same session: there the grass is *entirely* absent and the caller believes they
photographed a pose they did not, so a correct scene reads as broken. Here the frame is a true
picture of the scene rendered with a different flag set, merely thinner than the shipped result the
verb's summary promises.

## History
- `#1-game-view-thins-grass` `OPEN` reporter — Measured live against a running editor: same pose,
  same settings, `meanLuminance` 0.4013 with game view ON versus 0.3769 with it OFF; grass darkens
  the frame, so ON is the one with less grass. Handler attribution re-derived:
  `ViewportHandler.cpp:638` calls `FEditorViewportClient::SetGameView`, which swaps the whole
  `EngineShowFlags` set (`EditorViewportClient.cpp:7221-7249`, saved-set reuse at `:7229-7241`).
  The obvious cause is ruled out: `InstancedGrass` / `InstancedFoliage` / `InstancedStaticMeshes`
  are `SHOWFLAG_ALWAYS_ACCESSIBLE` (`ShowFlagsValues.inl:197`, `:189`, `:187`) and appear nowhere
  in the `FEngineShowFlags` constructor's init-mode disable list (`ShowFlags.h:181`, comment at
  `:390`), so the flag swap does not turn grass off. **Beyond that the mechanism is not
  attributed**; three specific unknowns are listed in the body and no cause is proposed.
