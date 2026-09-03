---
id: B-asset-exists-duplicate-false-negative-in-pie
title: "During PIE, asset.exists answers exists:false for every asset in the project and asset.duplicate refuses with ASSET_NOT_FOUND — the error blames the path, not the play mode"
status: OPEN
severity: High
category: bug
tags: [asset-exists, asset-duplicate, pie, play-mode, false-negative, misleading-error, multi-agent]
encounters: 2
lastSeen: 2026-09-02T20:05:00Z
---

# `asset.exists` says nothing exists and `asset.duplicate` says the source is missing, while PIE is running

With the editor in play mode, the registry-backed existence check that gates both verbs
answers **false for everything** and neither verb mentions PIE.

User-visible symptom, and the string a caller will search for:

```
[ASSET_NOT_FOUND] Source asset not found: /Game/FPS/VFX/Emitters/E_ImpConcrete_Flash
```

## Repro (live, UE 5.8, PinWright port 27145, 2026-09-02 ~19:52-19:57Z)

Another agent held PIE for about five minutes. During that window:

```
asset.exists {assetPath:"/Engine/BasicShapes/Cube"}                        -> {"success":true,"exists":false}
asset.exists {assetPath:"/Game/FPS/VFX/NS_Impact_Concrete"}                -> {"success":true,"exists":false}
asset.exists {assetPath:"/Game/FPS/VFX/Materials/M_FPS_Spark_Add"}         -> {"success":true,"exists":false}
asset.duplicate {sourcePath:"/Game/FPS/VFX/Emitters/E_ImpConcrete_Flash", …}
  -> [ASSET_NOT_FOUND] Source asset not found: /Game/FPS/VFX/Emitters/E_ImpConcrete_Flash
asset.duplicate {sourcePath:"/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst", …}
  -> [ASSET_NOT_FOUND] Source asset not found: /Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst
```

`/Engine/BasicShapes/Cube` is the control: it is not a project asset, it is not one this
session created, and it cannot plausibly be missing. The same object path with `.Cube`
appended fails identically, so it is not a package-vs-object-path spelling problem.

**Three other verbs resolve the very same paths at the very same moment**, which is what
proves the assets are there and the check is what is broken:

- `asset.list {path:"/Game/FPS/VFX/Emitters"}` enumerated 40 assets **including**
  `/Game/FPS/VFX/Emitters/E_ImpConcrete_Flash.E_ImpConcrete_Flash`.
- `niagara.validate {assetPath:"/Game/FPS/VFX/Emitters/E_ImpConcrete_Flash"}` loaded it and
  returned a full script/graph report.
- `niagara.create_emitter` created and **saved** a new emitter to disk in the same window
  (`saved:true`, `existsOnDisk:true`, confirmed 34760 bytes on disk), so not every write
  path is blocked — only the registry-gated ones.

The moment PIE stopped, both verbs answered correctly again with no other change:
`asset.exists {/Engine/BasicShapes/Cube}` -> `exists:true`, and the identical
`asset.duplicate` call succeeded.

Log correlation: `Saved/Logs/EAContentExamples58.log` spammed
`LogUtils: Error: The Editor is currently in a play mode.` throughout the window
(hundreds of lines, e.g. `[2026.09.02-19.56.23:523]`), so the editor **had** the real reason
and it never reached the RPC response.

## Why it matters

- **The error names the wrong thing.** "Source asset not found" sends the caller to check
  spelling, casing, package-vs-object path, and whether a sibling agent deleted their work.
  In this session it cost a full diagnostic detour before the log revealed play mode; a
  caller who trusts the message could reasonably conclude their assets were destroyed and
  start rebuilding them.
- **`asset.exists` reports it as `success:true`,** so it is a *false answer*, not an error —
  nothing in the response marks it as unverified. Anything branching on `exists` takes the
  wrong branch: a create-if-missing helper will happily overwrite an existing asset.
- **It is a multi-agent hazard by construction.** Any agent can start PIE at any time; every
  other agent's registry-gated verbs silently start lying for the duration. This is the same
  class as the already-filed `B-asset-save-pie-failure-reports-pendingflush` and
  `B-asset-save-omits-savestate-pie-block` (`asset.save` presenting a PIE block as a
  retryable throttle) — the save side was reported, the read/duplicate side was not.

## Fix

Detect play mode in the shared existence/resolution helper and report it, rather than
degrading to "not found":

1. `asset.exists` — either return the truthful answer (the package check does not need the
   PIE-blocked path) or, if it genuinely cannot be answered, return an error/`unverified`
   marker instead of `exists:false` under `success:true`. A false negative published as a
   success is the worst of the three options.
2. `asset.duplicate` — refuse with a distinct code, e.g.
   `EDITOR_IN_PLAY_MODE: cannot duplicate while PIE is running; stop play mode and retry`,
   never `ASSET_NOT_FOUND`.
3. Consider surfacing the state through the existing `editor.pie_status` verb in the error
   payload so a caller can act without reading the editor log.

severity rationale: impact=a false negative published under success:true, with an error message that actively misdirects the caller toward "my assets are gone" × reach=every registry-gated verb, for every agent in the shared editor, whenever any one agent starts PIE -> High

## History
- `#1-initial-repro` `OPEN` reporter — Hit while authoring `/Game/FPS/VFX/NS_Impact_Metal` and `NS_Impact_Wood` on UE 5.8 in the EAContentExamples58 checkout, during a ~5 minute PIE window another agent held (19:52-19:57Z). `asset.exists` returned `success:true, exists:false` for `/Engine/BasicShapes/Cube`, `/Game/FPS/VFX/NS_Impact_Concrete` and `/Game/FPS/VFX/Materials/M_FPS_Spark_Add`; `asset.duplicate` refused with `[ASSET_NOT_FOUND] Source asset not found` for both a project emitter and the stock `/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst`. Simultaneously `asset.list` enumerated the same emitter, `niagara.validate` loaded it, and `niagara.create_emitter` created and saved a new emitter to disk — so the assets existed and only the registry-gated check was affected. Both verbs recovered immediately and with no other change once PIE stopped. The editor log carried `LogUtils: Error: The Editor is currently in a play mode.` throughout; none of it reached the RPC response. Sibling tickets on the save side of the same PIE hazard: `B-asset-save-pie-failure-reports-pendingflush`, `B-asset-save-omits-savestate-pie-block`.
- `#2-second-agent-niagara-emitters` `OPEN` reporter — Independent second encounter on EAContentExamples58 (UE 5.8, shared editor, port 27145), authoring Niagara blood emitters. `asset.duplicate {sourcePath:"/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst", destinationPath:"/Game/FPS/VFX/Emitters/E_Blood_Burst"}` returned `[ASSET_NOT_FOUND] Source asset not found: /Niagara/...SimpleSpriteBurst` — twice, and again with the full object-path form `...SimpleSpriteBurst.SimpleSpriteBurst`. The identical call had succeeded four times earlier in the same session while PIE was stopped. Log line at `20.02.00:257` puts `LogUtils: Error: The Editor is currently in a play mode.` in the same millisecond as the `Automation request failed (ASSET_NOT_FOUND)` line, which is the direct evidence tying the two. **Adds one narrowing fact this ticket did not have: `asset.list` is UNAFFECTED.** During the same PIE window `asset.list {path:"/Niagara/DefaultAssets/Templates/Emitters"}` returned all 14 templates with correct object paths, classes and tags, while `asset.exists` answered `exists:false` for `/Niagara/Modules/Collision/Collision`, `/Engine/BasicShapes/Sphere`, and `/Game/FPS/VFX/Emitters/E_Blood_Mist` — the last of which this agent had written and `ls`-verified on disk 90 seconds earlier. So the asset registry itself is intact and correctly populated under PIE; only the existence-check path that `asset.exists` and `asset.duplicate` share returns the false negative. That should localize the fix, and it also gives callers a usable workaround: `asset.list` on the parent folder is a reliable existence probe while PIE is running. Scope note for whoever fixes it: a `/Niagara/` plugin-content source path fails identically to a `/Game/` one, so this is not specific to project content or to the PIE world's `UEDPIE_` package prefixing of `/Game/`. Cross-ref: `B-asset-save-omits-savestate-pie-block` (OPEN) — same PIE root cause, different verb, same shape of defect (the response names neither PIE nor the real outcome).
- `#3-third-witness-asset-search-also-unaffected` `OPEN` reporter — Third witness, same shared editor (EAContentExamples58, UE 5.8, port 27145), authoring Niagara explosion emitters. `encounters` deliberately NOT bumped and `lastSeen` left alone: this falls inside the same PIE window as `#2` (log `LogUtils: Error: The Editor is currently in a play mode.` runs continuously to `20.03.03:201`), so it is the same occurrence with another witness. Five `asset.duplicate {sourcePath:"/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst"}` calls failed `ASSET_NOT_FOUND` at `20.01.09:334`, `20.01.13:795`, `20.02.38:513`, `20.02.50:515` and `20.02.53:203` — each log line immediately preceded by two play-mode lines in the same millisecond — and the full object-path form `SimpleSpriteBurst.SimpleSpriteBurst` failed identically, confirming `#1`'s spelling finding. **Adds one fact this ticket did not have: `asset.search` is unaffected as well as `asset.list`.** At `20.02.42`-`20.02.44`, between the failures at `:38` and `:50`, `asset.search {query:"SimpleSpriteBurst"}` returned the asset with correct name, object path, class `NiagaraEmitter` and packagePath. So both registry query verbs read correctly under PIE while the existence-check path shared by `asset.exists` and `asset.duplicate` returns the false negative — which tightens `#2`'s localization and widens the caller workaround from `asset.list` on the parent folder to `asset.search` by name. Recovery was self-healing and needed no retry logic beyond waiting: the first `asset.exists` after the last play-mode line answered `exists:true` for the same path, and the identical `asset.duplicate` then succeeded. Practical cost to this stream: about two minutes of misdirected diagnosis (checking the mount, both path spellings, and whether a sibling agent had deleted engine content) before the log tied it to PIE — the misdirection `#1`'s severity rationale predicts, reproduced verbatim by a caller who had not read this ticket.
