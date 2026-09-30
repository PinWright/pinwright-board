---
id: B-recorder-drain-writeevent-fname-segv
title: "Editor SIGSEGV in `FNdjsonSessionWriter::WriteEvent` (`FName::ToString` on a corrupt FName) during the recorder drain tick right after a PIE map travel"
status: IN-REVIEW
severity: High
category: bug
tags: [recorder, journal, crash, fname, pie, map-travel]
encounters: 1
lastSeen: 2026-09-29T19:46:23Z
---

# Recorder drain crashes the editor after a PIE map load

Fresh visible editor (`-SKIPCOMPILE`), `editor.play {}` on `L_Core`, then
`editor.console_command {command: "App.Launch DA_ChemicalMap_Race_00 mapmode=EditMap", world: "pie:0"}`.
The map loaded (`LoadMap(/Game/Maps/L_PDS_ChemicalPlant)` took 20.5 s), about 2.5 s later the editor died:

```
Unhandled Exception: SIGSEGV: invalid attempt to read memory at address 0x0000000002015173
  FName::ToString(FString&) const                  NameTypes.h:328
  FName::ToString() const                          UnrealNames.cpp:3608
  FNdjsonSessionWriter::WriteEvent(...)            PinWrightRecorder
  FJournalRecorder::DrainAndFlush()                JournalSession.cpp:155
  FRecorderLifecycle::OnDrainTick(float)           RecorderLifecycle.cpp:90
  FTSTicker::Tick
```

The FName passed to `WriteEvent` (`Key` or `Name`) has a garbage index, so the queued event record was
freed/overwritten between enqueue and drain, or was enqueued with an uninitialised FName. The crash came
on the first drain after the map travel, while the new world was still finishing BeginPlay.

No agent call touched the recorder; the only RPCs were `editor.play` and the console command above (which
also raised the known `B-console-command-world-precondition-ensure`).

Evidence: `Saved/Crashes/crashinfo-PDS-pid-3794727-01A0EEB439A07E4587B47574E97179C5`, `Saved/Logs/PDS.log`
(backed up on the next start). Host `/sdb-disk/src/unreal/unreal-fpv` (Linux), UE 5.8, plugin `8fcc0b2a`.

**Impact:** lost editor session (~3 min restart); no work lost here.

## History
- `#1-segv-after-app-launch` `OPEN` reporter - First sighting, as above.
- `#2-enqueue-rejects-corrupt-fname` `IN-REVIEW` developer - The recorder holds no pointers or views. Every `FRecordMsg` field is copied by value (`JournalTypes.h`), the MPSC queue has one consumer, and every drain runs on the game thread. So the corrupt key reached `WriteEvent` exactly as the producer handed it over. The crash line in plugin `8fcc0b2a` is `NdjsonSessionWriter.cpp:169` = `Key.ToString()`. A read at `0x2015173` means a name-pool block pointer of null plus an offset, so the key's entry id was past the last allocated block, which is what `GetFName()` on a freed UObject returns. The likely producer is in the host: an async-load `StateFunc` that captures a raw `this` and runs `KeyFor(this)` after map travel GC'd its object. The evidence: the session journal has no `menu:reached` before travel, and the menu input mode switched on after `LoadMap` finished. This is inferred from the log and not reproduced. It is filed on the host project's own board. Fix: `JournalRecorder.cpp` now checks every FName at the producer boundary (`RegisterObject(FName)`, `Append`, `LogEvent` key / name / prop keys) with `FName::IsValid()` on both the comparison id and the display id. A message with an out-of-pool id is dropped with `LogJournalRecorder: Warning: Journal: dropped <event|value|object registration> '<name>': it carries a corrupt FName ...`, which names the producer's event or tag. The drain never resolves such a name. A garbage id that happens to fall inside the pool is not detectable this way. Test: `PinWright.recorder.writer.CorruptNameRejectedAtEnqueue` (`Tests/Recorder/TestJournalCorruptName.cpp`) enqueues an out-of-pool FName as event key, prop key, value key and catalog key, plus one valid event. It then drains and asserts that only the valid event reached the file. Unfixed, the drain segfaults in `WriteObject`/`WriteEvent`. Doc: `docs/wiki-src/recorder.integration.md`.
