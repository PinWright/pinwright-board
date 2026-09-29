---
id: B-recorder-drain-writeevent-fname-segv
title: "Editor SIGSEGV in `FNdjsonSessionWriter::WriteEvent` (`FName::ToString` on a corrupt FName) during the recorder drain tick right after a PIE map travel"
status: OPEN
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
