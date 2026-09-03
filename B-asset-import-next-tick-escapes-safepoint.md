---
id: B-asset-import-next-tick-escapes-safepoint
title: "asset.import defers ImportAssetsAutomated into FTimerManager's world-tick callback, outside PinWright's safe-point and active-request guards"
status: OPEN
severity: High
category: bug
tags: [asset, import, timer, safe-point, tick, reentrancy, interchange, crash]
---

# `asset.import` moves hazardous import work into `UWorld::Tick`

## What's wrong

`AssetManageHandler.cpp:275-350` returns from the handler after scheduling
`ImportAssetsAutomated` with `GEditor->GetTimerManager()->SetTimerForNextTick`. The dispatcher can
only safe-point and serialize the handler stack; this later callback runs with neither guard.
UE 5.8 calls `GetTimerManager().Tick` from `UWorld::Tick` (`LevelTick.cpp:1811-1817`), and the timer
setting explicitly permits a next-tick timer to run in the same engine tick by default
(`TimerManager.cpp:58-62`). The import then loads factories, creates/replaces packages, compiles
assets, and may pump Interchange while the world is mid-frame—the exact position the safe-point
gate exists to avoid.

Concrete path: an RPC drained during a frame schedules the timer; the timer fires later inside
that or the next `UWorld::Tick`; `ImportAssetsAutomated` re-enters editor asset/package work from
the tick stack. The source comment says the deferral exists to avoid TaskGraph recursion, but it
changes the stack, not the unsafe position.

## What it should do

Keep the asynchronous response token, but launch the import from
`PinWrightSafePoint::RunAtSafePoint`/`DeferToSafePoint` and retain the active request guard through
completion. Add a source ratchet proving no asset import executes directly from a world timer.

## Workaround

Do not call `asset.import` while any world is ticking; there is no in-band way for a caller to
prove that timing.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the crash-pattern catalog; no editor, build, test, or RPC run was performed.
