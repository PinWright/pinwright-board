---
id: B-asset-import-next-tick-escapes-safepoint
title: "asset.import defers ImportAssetsAutomated into FTimerManager's world-tick callback, outside PinWright's safe-point and active-request guards"
status: IN-REVIEW
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

## Fix

Verdict: **PARTLY TRUE** (`valid-bug-wrong-fix`). The bare callback did escape the active-request
serialization and unattended scopes, but the reported `UWorld::TimerManager` / world-tick premise
was false. `GEditor` is `UEditorEngine`; its timer is ticked by `UEditorEngine::Tick` before editor
worlds, rather than by `UWorld::Tick`. The defect was lost request ownership and modal-suppression
scope across a detached asynchronous continuation, not execution from a world timer.

`Dispatch/SafePoint.h` now exposes `PinWrightSafePoint::DeferRequestToSafePoint`, which always takes
one core-ticker safe-point hop, creates the safe-point responder/token, retains the dispatcher's
active request through the callback, and uses the existing standalone fallback. `asset.import` in
`Handlers/Asset/AssetManageHandler.cpp` uses that helper and responds through
`FSafePointResponder`, retaining serialization and unattended mode through import, rename,
verification, and response. `Dispatch/ScopedUnattendedRpc.h` records that explicit continuation
coverage. `Docs/wiki-src/asset.md` and defect-backlog D-74 now describe the retained asynchronous
contract.

`PinWright.asset.import.SafePointContinuationRetainsRequestScope` in
`Tests/Assets/TestAssetImportSafePoint.cpp` covers non-inline execution, retained processing and
queue order, safe-point and unattended state, deferred response routing, guard release, a source
ratchet against `SetTimerForNextTick` / `GetTimerManager`, and continued absence from the
tick-unsafe method table.

Deliberately unchanged: the existing overwrite default, collision refusal, complete multi-output
response, canonical paths, persistence reporting, primary-only rename behavior, and rename-failure
payload. No tick-unsafe table entry was added because handler-entry gating cannot retain detached
work. This is a source-only implementation; no editor, build, automation, or RPC run was performed.

## History
- `#1-pattern-scan` `OPEN` reporter — Source-only confirmation from the crash-pattern catalog; no editor, build, test, or RPC run was performed.
- `#2-retained-import-continuation` `IN-REVIEW` developer — Replaced `asset.import`'s bare editor-timer callback with `PinWrightSafePoint::DeferRequestToSafePoint`, retaining the active-request and unattended scopes through import, rename, verification, and response; added `PinWright.asset.import.SafePointContinuationRetainsRequestScope` with a source ratchet against `SetTimerForNextTick`. Source-only implementation; no editor, build, automation, or RPC run was performed.
