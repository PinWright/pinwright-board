---
id: E-game-framework-info-not-asset-readback
title: "game_framework.get_game_framework_info is a level-spawn snapshot, not an asset read-back — undocumented, so it can't confirm a configured GameMode asset"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, game_framework, readback, discovery, wiki]
---

# `game_framework.get_game_framework_info` is a level-spawn snapshot, not an asset read-back

Every write verb in the `game_framework` namespace
(`set_default_pawn_class`, `set_game_state_class`, `set_player_controller_class`,
`configure_game_rules`, `configure_team_system`, `configure_scoring_system`,
`configure_round_system`, `set_respawn_rules`) takes a **required
`gameModeBlueprint` path** and mutates that **standalone GameMode asset's CDO**
(`GameFrameworkHandler.cpp` lines 280, 292, 304, 316, 328, 399, 473, 550, 617,
753, 813 all declare `RPC_PARAM_REQ("gameModeBlueprint", ...)`).

`game_framework.get_game_framework_info` (the obvious "did it work?" read in the
same namespace) takes **`RPC_NO_PARAMS`** and reads an entirely different thing:
the **active editor level's** `WorldSettings->DefaultGameMode` plus the level's
`APlayerStart` actors (`GameFrameworkHandler.cpp:865-911`). It returns
`{ gameMode, playerStarts[], playerStartCount }` and nothing about the rules /
teams / scoring / rounds / respawn / pawn / state / controller classes that the
write verbs just persisted onto the asset. When the configured asset is not the
level's GameMode override, the verb reports `gameMode: "(default)"` (line 885)
even after a fully successful configure run.

So an agent that follows the natural confirm flow — configure the GameMode
asset, then call the same namespace's `get_*_info` to verify — gets a response
that is **structurally incapable** of confirming its own work: no
`gameModeBlueprint` input, no per-asset CDO output. The two verbs share a
namespace and a noun ("game framework") but operate on different objects (a
standalone asset vs. the live level), and nothing in the method summaries or the
`game_framework` wiki overlay flags the asymmetry.

## Evidence (this task)

A 16-call task wired a full 4v4 CTF GameMode onto
`/Game/Pickups/GameModes/BP_CTFGameMode` via all 8 `game_framework` write verbs
(every call `ok:true`), then hit exactly this friction. From the friction note:

> "get_game_framework_info only returns a wiki file path (not inline content)
> and reports only level-spawn state, so I ... used blueprint.inspect +
> includeProperties (which paged its 33k-char response to a file I had to read)
> to verify the saved CDO."

And the self-report:

> "Final get_game_framework_info still reports gameMode '(default)' because it
> reflects only the live level's spawn state, not the standalone GameMode asset
> — verified via CDO as the success check permits."

The documented confirm-step (story step 10: "read back
get_game_framework_info to confirm the setup") therefore could not confirm
anything; the agent had to recognize the mismatch mid-task and pivot to a
`blueprint.inspect` + `includeProperties` CDO read-back, whose 33k-char response
paged to a file the agent then had to open and read — extra steps and tokens for
a verification the namespace appears to advertise natively.

## What it should do / how to fix (docs-first)

This is primarily a **discovery/docs gap**, not a behavior bug — the level-spawn
snapshot is a legitimate read, it just isn't the per-asset read-back the write
verbs imply. Lowest-cost fix is to document the asymmetry on the
`docs/wiki-src/game_framework.md` overlay (currently a bare 2-sentence prelude
with no `##` sections):

- Add a `## Reading back a configured GameMode asset` section stating plainly
  that `get_game_framework_info` reports the **active level's** spawn state
  (`WorldSettings->DefaultGameMode` + `PlayerStart` actors) and does **not**
  echo the `gameModeBlueprint` asset that the `set_*`/`configure_*` verbs write.
  Point agents to `blueprint.inspect { assetPath, includeProperties: true }`
  (see `F-rpc-blueprint-class-properties`, now DONE) as the supported way to
  confirm the configured CDO values, and note that
  `get_game_framework_info` reporting `gameMode: "(default)"` after a configure
  run is expected, not a failure.

Optional ergonomic follow-up (separate, larger): give
`get_game_framework_info` an optional `gameModeBlueprint` param that, when
supplied, returns the configured class/rule fields from that asset's CDO so a
single namespace verb closes the configure→verify loop. Documenting the current
behavior is the cheap win; the param is a nice-to-have.

**Workaround:** read back configured GameMode values with
`blueprint.inspect { assetPath: "<the gameModeBlueprint>", includeProperties: true }`,
not `get_game_framework_info`.

## History
- `#3-docs-asymmetry-section` `IN-REVIEW` developer — Docs-first fix (the established house pattern for the E-*-readback family). Added a `## Reading back a configured GameMode asset` section to `Docs/wiki-src/game_framework.md` (was a bare 2-sentence prelude, no `##` sections): it states plainly that `get_game_framework_info` (RPC_NO_PARAMS) reports the **active level's** spawn state (`WorldSettings->DefaultGameMode` + `PlayerStart` actors) and does NOT echo the `gameModeBlueprint` asset CDO the `set_*`/`configure_*` write verbs persist; marks a post-configure `gameMode: "(default)"` (or an unrelated level-override GameMode) answer as **expected, not a failure**; and routes the asset-CDO read-back to `blueprint.inspect {assetPath, includeProperties:true}` (the `F-rpc-blueprint-class-properties` sparse CDO diff). No handler code changed — the level-spawn snapshot is a legitimate read, only undocumented. Regression test: `Source/PinWright/Private/Tests/Infra/TestGameFrameworkAssetReadbackDocs.cpp` (`FGameFrameworkAssetReadbackNamespaceDocTest`) renders the `game_framework` namespace page through the live `WikiHandler::RenderPage` path (same entry the HTTP gateway uses, not a copy of the overlay text) and asserts the section markers — section title, the gameModeBlueprint-asset-CDO-vs-level contrast, the `WorldSettings->DefaultGameMode` level-state statement, the `(default)` "expected, not a failure" note, and the `blueprint.inspect`+`includeProperties` route; reverting the overlay section drops the renderer to the prelude only and every assertion fails. Optional `gameModeBlueprint` param on the info verb deliberately left as the separate larger follow-up named in the body.
- `#2-additional-readback-asymmetry` `OPEN` reporter — Additional evidence (independent CTF-mode seed task on `/Game/Multiplayer/Modes/BP_CaptureTheFlagMode`, parent `GameModeBase`). Replay-confirmed: all 6 `game_framework` write verbs (`configure_team_system`/`configure_scoring_system`/`configure_round_system`/`set_default_pawn_class`/`set_player_state_class`) persisted correctly to the asset CDO — `blueprint.inspect { includeProperties:true }` reads back `NumTeams:2`, `Team0_Name:"Red"`, `Team1_Name:"Blue"`, `MaxPlayersPerTeam:8`, `bAutoBalance:true`, `ScoreToWin:3`, `ObjectiveScore:1`, `bTeamScoring:true`, `MaxRounds:5`, `RoundDuration:600`, `RoundTransitionDelay:30`, `WarmupDuration:15`, `bAutoStartRound:true`, `DefaultPawnClass:".../BP_CTFCharacter_C"`, `PlayerStateClass:".../BP_CTFPlayerState_C"` (all `is_overridden_locally:true`). But the task's prescribed verify step — `get_game_framework_info {}` — returned `{"gameFrameworkInfo":{"gameMode":"/Game/Global/Blueprints/CE_Game.CE_Game_C","playerStarts":[...],"playerStartCount":1}}`: only the active level's GameMode (`CE_Game_C`, unrelated to the configured asset) plus player starts, with no `gameModeBlueprint` input and no team/scoring/round/class fields. Confirms the level-vs-asset asymmetry reproduces independently (here the active GameMode is a real BP `CE_Game_C` rather than `"(default)"`, so the answer is even more misleading — it names a *different* GameMode, not an obvious empty sentinel). The configure→verify loop still can't be closed within the namespace; `blueprint.inspect`+`includeProperties` remains the only readback path.
- `#3-confirms-documented-workaround` `OPEN` reporter — Confirming evidence (independent "Last Squad Standing" seed task on `/Game/Multiplayer/Modes/BP_LastSquadStanding`, parent `GameModeBase`, 14 calls all `ok:true`, outcome `clean`). Notable PROCESS angle: this task's story **pre-routed the agent around** the asymmetry — it explicitly instructed "do NOT use get_game_framework_info to confirm ... per the game_framework docs it reports only the active level's spawn state, not this standalone asset's CDO" and told the agent to verify via `blueprint.inspect { includeProperties:true }` instead. Result: the configure→verify loop round-tripped with **zero friction** (`MaxRounds=5`, `RoundDuration=240`, `WarmupDuration=20`, `RoundTransitionDelay=12`, `bAutoStartRound=true`, all `is_overridden_locally:true`), friction note literally "none." This is direct validation that the proposed docs fix works: when the level-vs-asset asymmetry is documented up front and the caller is pointed at `blueprint.inspect includeProperties`, the task is clean — strengthening the case that the cheap docs win (a `## Reading back a configured GameMode asset` section on `docs/wiki-src/game_framework.md`) is sufficient to retire this friction without needing the optional `gameModeBlueprint` param. The only residual cost this run was the inspect response auto-spilling past the 10KB display threshold to the on-disk `HttpResponses` dump (documented behavior, see `E-http-response-spill` DONE / `F-rpc-property-omit-oversized-opt-in` DONE) — not a real obstacle.
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a 16-call CTF GameMode setup task (all `game_framework` write verbs `ok:true`, outcome filed separately as `B-wiki-namespace-underscore-not-found`). Distinct PROCESS angle: the namespace's own `get_game_framework_info` cannot confirm the asset the other 8 verbs configure — it takes `RPC_NO_PARAMS` and reads the level's `WorldSettings->DefaultGameMode` + `PlayerStart`s (`GameFrameworkHandler.cpp:865-911`), while every write verb takes a required `gameModeBlueprint` and writes that standalone asset's CDO. Friction note: agent had to abandon the documented `get_game_framework_info` confirm-step (which reported `gameMode: "(default)"` post-configure) and fall back to `blueprint.inspect`+`includeProperties`, whose 33k-char response paged to a file it had to read. `docs/wiki-src/game_framework.md` is a bare 2-sentence prelude that never documents this level-vs-asset asymmetry. Proposed: add a `## Reading back a configured GameMode asset` section pointing to `blueprint.inspect includeProperties`; optionally add a `gameModeBlueprint` param to the info verb.
- `#4-deathmatch-confirms-docs-and-inherited-default-nuance` `OPEN` reporter — Confirming evidence (independent "arena deathmatch" seed task on `/Game/Gameplay/GameModes/BP_ArenaDeathmatchGameMode`, parent `GameModeBase`, outcome `done`). Validates the live docs fix once more: with the `## Reading back a configured GameMode asset` overlay section in place (rendered overlay re-read this run), the attempt agent never tried to confirm via `get_game_framework_info` — its friction note says the wiki "flagged this clearly so it cost no real confusion." Replay-confirmed all writes persisted to the asset CDO: `blueprint.inspect {includeProperties:true}` reads back `MaxPlayers:12`, `bFriendlyFire:false`, `bAllowRespawn:true`, `RespawnDelay:5`, `bAutoAssignTeams:true`, `NumTeams:2`, `MaxPlayersPerTeam:6`, `Team0_Name:"Red"`, `Team1_Name:"Blue"`, `bAutoBalance:true`, `ScoreToWin:50`, `KillScore:1`, `DeathPenalty:0`, `bTeamScoring:true` (all `is_overridden_locally:true`); and `get_game_framework_info {}` again returned only level-spawn state (`gameFrameworkInfo.gameMode` + `playerStarts`), no asset fields — same asymmetry, no confusion because it was documented. New REFINEMENT for the body's read-back guidance (line 14 names `DefaultPawnClass` as confirmable via `blueprint.inspect`+`includeProperties`): `set_default_pawn_class` set `DefaultPawnClass` to `/Script/Engine.DefaultPawn`, which is **`AGameModeBase`'s inherited engine default** (`GameModeBase.cpp:71` `DefaultPawnClass = ADefaultPawn::StaticClass()`). Because the requested value equals the inherited default, it is NOT a local override, so `blueprint.inspect`'s sparse CDO diff **omits it entirely** — the agent had to fall back to `property.get {objectPath:".../BP_ArenaDeathmatchGameMode.Default__BP_ArenaDeathmatchGameMode_C", propertyName:"DefaultPawnClass"}` → `/Script/Engine.DefaultPawn` (replay-confirmed) to verify it landed. (`property.get` on the `..._C` UClass path correctly returns `[PROPERTY_NOT_FOUND]` — instance props live on the `Default__..._C` CDO, not the UClass.) So the inspect route is silent precisely for the common "make it playable with the stock engine pawn" case; the docs section could add a one-line caveat that a class assignment equal to the parent's inherited default won't appear in the sparse diff and must be read back via `property.get` against the CDO. Not a behavior bug — set/get both correct, value genuinely persisted — only a read-back-discoverability nuance on this same docs ticket.
