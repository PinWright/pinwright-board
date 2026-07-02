---
id: E-respawn-delay-dual-verb
title: "respawnDelay is settable from BOTH game_framework.configure_game_rules and set_respawn_rules (writes the same RespawnDelay var, different category) with no ownership guidance — agents double-set it"
status: OPEN
severity: Low
category: ergonomic
tags: [shared-param-no-ownership, docs, game_framework, configure_game_rules, set_respawn_rules, respawnDelay, discovery, wiki]
encounters: 1
lastSeen: 2026-07-01T23:45:31+03:00
---

# `respawnDelay` straddles two `game_framework` write verbs with no canonical owner

A GameMode's respawn configuration is split across two `game_framework` write
verbs that BOTH expose a `respawnDelay` param, and neither wiki page says which
is canonical:

- **`game_framework.configure_game_rules`** (`GameFrameworkHandler.cpp:325`) —
  params include `bAllowRespawn` (the on/off switch, `:331`) **and**
  `respawnDelay` (`:332`). The delay branch writes the Blueprint variable
  `RespawnDelay` under category **"Game Rules"** (`:369-371`).
- **`game_framework.set_respawn_rules`** (`GameFrameworkHandler.cpp:750`) —
  params `respawnDelay` (`:754`) + `maxLives` (`:755`). Its delay branch writes
  the **same** Blueprint variable `RespawnDelay`, but under category
  **"Respawn"** (`:774-776`).

So `RespawnDelay` is a single field reachable from two verbs. There is no
`respawnDelay` ownership note on either method page or the `game_framework`
overlay, so an agent facing "respawn 5s / unlimited lives" — which naturally
routes to both verbs (the on/off + delay live on `configure_game_rules`, the
`maxLives` on `set_respawn_rules`) — has no signal that `respawnDelay` is shared
and defensively sets it in both. Two subtle wrinkles compound the confusion:

1. **Redundant write.** Both calls write the identical `RespawnDelay` value, so
   one of the two writes is dead work.
2. **Category drift by call order.** The two verbs file the *same* variable into
   *different* Blueprint categories ("Game Rules" vs "Respawn") via
   `AddBlueprintVariable`, so which category `RespawnDelay` ends up in depends on
   which verb ran last — a non-obvious, order-dependent side effect for a field
   the caller thinks it set once.

This is the overlapping-capability / no-docs-reconciliation friction family
(cf. `E-geometry-auto-uv-redundant-with-unwrap-uv`,
`E-asset-search-vs-search-assets-overlap`,
`E-anim-blueprint-create-two-methods-discovery`) — here at the *shared-parameter*
granularity rather than whole-verb duplication: adjacent verbs in one namespace
overlap on a field and the overlay never names the owner, so the caller pays a
redundant call.

## Evidence (this task)

Arena-deathmatch GameMode build on `/Game/Arena/BP_ArenaDeathmatchGameMode`
(parent `GameModeBase`, 13 calls, all `ok:true`, outcome `clean`). The
CallAnalyzer trace shows `respawnDelay` set in BOTH verbs, back-to-back:

- `game_framework.configure_game_rules { maxPlayers:16, friendlyFire:false, allowRespawn:true, respawnDelay:5 }` (trace line 263)
- `game_framework.set_respawn_rules { respawnDelay:5, maxLives:-1 }` (trace line 265)

`blueprint.inspect { includeProperties:true }` confirmed the single
`RespawnDelay=5` persisted correctly (both writes landed the same value), so the
outcome was fully correct — this is pure PROCESS redundancy, not a data bug. The
attempt's own friction note reads "none"; the double-set surfaced only in the
call-trace analysis (a HUNCH the CallAnalyzer escalated to Audit), which is why
it is filed here as a discoverability/ergonomic gap rather than an outcome
defect.

## What it should do / how to fix (docs-first, NAMES the overlay pages)

Docs-first, matching the house pattern for this family. The overlay edit is a
downstream wiki process; naming the pages and the change is the deliverable:

- On `docs/wiki-src/game_framework.md` (and the two method pages
  `game_framework.configure_game_rules.md` /
  `game_framework.set_respawn_rules.md`), add a short "respawn config" note that:
  - states `respawnDelay` writes the **shared `RespawnDelay`** Blueprint variable
    and names ONE canonical verb for it — recommend
    **`set_respawn_rules`** (the dedicated respawn verb, which also owns
    `maxLives`), and note that `configure_game_rules.respawnDelay` writes the
    *same* variable (so setting both is redundant and the last writer decides the
    Blueprint category);
  - keeps `bAllowRespawn` (enable/disable) on `configure_game_rules` and the
    respawn timing/lives detail on `set_respawn_rules`, so the caller sets each
    field once from one place.
- Optional structural follow-up (out of scope for this docs ticket): drop
  `respawnDelay` from `configure_game_rules` (leaving only `bAllowRespawn`
  there), or have both verbs file `RespawnDelay` under one consistent category so
  the order-dependent categorization can't happen.

**Workaround:** set `respawnDelay` from `set_respawn_rules` only (alongside
`maxLives`); leave it off `configure_game_rules` (use that verb for
`bAllowRespawn` and the non-respawn rules). Both write the same value, so setting
it in one place is sufficient.

severity rationale: impact=docs/discoverability (redundant call + order-dependent
category, values persist correctly) × reach=game_framework GameMode setup is an
occasional, not every-session, path -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the arena-deathmatch GameMode setup (13 calls, all `ok:true`, outcome `clean`; the read-back asymmetry angle already tracked on `E-game-framework-info-not-asset-readback`). Distinct PROCESS angle: `respawnDelay` is a param on BOTH `game_framework.configure_game_rules` (`GameFrameworkHandler.cpp:332`, writes `RespawnDelay` under category "Game Rules" at `:369`) and `game_framework.set_respawn_rules` (`:754`, writes the same `RespawnDelay` under category "Respawn" at `:774`), with no ownership note on either wiki page. The agent double-set `respawnDelay:5` in both verbs back-to-back (trace lines 263, 265) to be safe; the value persisted correctly (verified via `blueprint.inspect includeProperties`), so this is redundant-call friction, not a data bug — the shared field also causes an order-dependent Blueprint category ("Game Rules" vs "Respawn"). CallAnalyzer flagged it as a HUNCH for Audit (low/docs). Proposed: name the canonical owner (`set_respawn_rules`) for `respawnDelay` on `docs/wiki-src/game_framework.md` + both method pages, and note `configure_game_rules.respawnDelay` writes the same variable redundantly; optional structural follow-up to drop the duplicate param or unify the category.
