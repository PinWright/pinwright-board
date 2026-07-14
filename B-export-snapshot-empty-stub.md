---
id: B-export-snapshot-empty-stub
title: "environment.build.export_snapshot reports success but writes an empty stub (no actors); import_snapshot restores nothing"
status: IN-REVIEW
severity: High
category: bug
tags: [environment, snapshot, export, import, silent-success, no-effect, round-trip]
---

# `environment.build.export_snapshot` writes a content-free stub and reports success

`environment.build.export_snapshot` returns `{"exportPath":...,"message":"Snapshot
exported","success":true}` and writes a real file to disk, but the file body
contains **only** `{timestamp, type}` — none of the level's environment state.
No terrain, sky sphere, fog volume, sky atmosphere, volumetric cloud,
directional/sky light, or time-of-day value is captured. The handler never
enumerates a single actor from the world; it serializes a hard-coded two-field
object and saves that.

Its documented counterpart `environment.build.import_snapshot` is the symmetric
half of the same no-op: it loads the JSON, parses it, **echoes it straight back**
as the `snapshot` field, and reports `{"message":"Snapshot imported","success":true}`
— without applying anything to the world (no spawn, no property set). So the
entire export → import "environment snapshot" round-trip is hollow: the namespace
page (`environment.md`) advertises `call("environment.build")` "for generated
environment assets **or snapshots**", but a caller who exports a look and later
imports it to restore it gets `success:true` both times and **zero** restoration.
This is a silent success-with-no-effect: the operation that the call name,
message, and `success:true` all promise (persist the environment so it can be
restored) does not happen, and nothing in either response signals that.

Severity High: this is the only environment persist/restore facility in the
namespace, and both directions silently succeed while losing 100% of the
payload. An agent told to "save this look so I can restore it later" gets a green
checkmark and a JSON file that can never reproduce the scene.

## Verbatim repro (live, replay-confirmed against `mcp__editor-automation__call`)

Level is populated (an `actor.list` on the same world returned a 121,942-char
payload of actors).

1. `environment.build.export_snapshot` `{"path":"Saved/EnvSnapshots/oracle_replay.json"}`
   → `{"exportPath":"/Saved/EnvSnapshots/oracle_replay.json","message":"Snapshot exported","success":true}`
2. On-disk file `Saved/EnvSnapshots/oracle_replay.json` — **entire contents**:
   ```json
   {
   	"timestamp": "2026.06.19-03.14.54",
   	"type": "environment_snapshot"
   }
   ```
   No actors, no environment data at all.
3. `environment.build.import_snapshot` `{"path":"Saved/EnvSnapshots/oracle_replay.json"}`
   → `{"snapshot":{"timestamp":"2026.06.19-03.14.54","type":"environment_snapshot"},"message":"Snapshot imported","success":true}`
   — `success:true` "imported", but the world is unchanged because the file holds
   nothing to import.

## Root cause

`Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp`

Export handler (`environment.build.export_snapshot`, lines ~156-170) builds the
snapshot from two literal fields and never touches the world:

```cpp
TSharedPtr<FJsonObject> Snapshot = MakeShared<FJsonObject>();
Snapshot->SetStringField(TEXT("timestamp"), FDateTime::UtcNow().ToString());
Snapshot->SetStringField(TEXT("type"), TEXT("environment_snapshot"));
// ... serialize + SaveStringToFile ...
Resp->SetStringField(TEXT("message"), TEXT("Snapshot exported"));
Resp->SetBoolField(TEXT("success"), true);
```

Import handler (`environment.build.import_snapshot`, lines ~223-235) deserializes
and echoes, applying nothing:

```cpp
// ... LoadFileToString + Deserialize into SnapshotObj ...
Resp->SetObjectField(TEXT("snapshot"), SnapshotObj.ToSharedRef());
Resp->SetStringField(TEXT("message"), TEXT("Snapshot imported"));
Resp->SetBoolField(TEXT("success"), true);
```

Both handlers are stubs: there is no code path that walks the environment actors
(terrain / sky / fog / atmosphere / cloud / lights) on export, nor any that
re-creates or mutates them on import.

**Workaround:** do not rely on export/import_snapshot to persist a look. Capture
the environment via `actor.list` + per-actor `actor.describe` / `property.get`
into your own JSON, and rebuild on restore with the typed spawn verbs
(`environment.build.create_*`, `environment.spawn_*`) + `property.set`.

**Fix:** make export actually serialize the environment — enumerate the relevant
actors in the active world (terrain meshes, SkySphere, fog volumes, SkyAtmosphere,
VolumetricCloud, Directional/SkyLight, time-of-day) and write their class, label,
transform, and key component properties; make import re-apply them. At minimum, if
real serialization is out of scope, stop reporting `success:true` / "Snapshot
exported" for a body that contains no environment data — return a clear "not
implemented" / empty-snapshot error so the no-op is not disguised as success.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live via `mcp__editor-automation__call` on a populated level (an `actor.list` returned 121,942 chars of actors). `environment.build.export_snapshot {path:"Saved/EnvSnapshots/oracle_replay.json"}` → `{"exportPath":"/Saved/EnvSnapshots/oracle_replay.json","message":"Snapshot exported","success":true}`, but the on-disk file body is exactly `{"timestamp":"2026.06.19-03.14.54","type":"environment_snapshot"}` — no actors, no environment state. `environment.build.import_snapshot` on that file → `{"snapshot":{timestamp,type},"message":"Snapshot imported","success":true}` while changing nothing. Root cause: both handlers in `EnvironmentHandler.cpp` are stubs — export serializes two literal fields and never enumerates the world; import deserializes and echoes without applying. The documented export/import environment-snapshot round-trip (per `environment.md`, "environment.build ... for generated environment assets or snapshots") is a hollow no-op that silently reports success in both directions. Surfaced by the canyon-overlook greybox task (seed `environment.build.create_procedural_terrain`, which itself worked); culprit is `export_snapshot`.
- `#2-stubs-return-not-implemented` `IN-REVIEW` developer — Applied the minimum-safe honest-failure fix matching the accepted silent-success-stub precedent (B-material-stub-handlers-silent-success DONE, B-input-trigger-modifier-stub-silent-success IN-REVIEW): real environment serialization is out of scope, so both stubs now fail loud instead of disguising the no-op as success. In `EnvironmentHandler.cpp`, `environment.build.export_snapshot` now returns `SendError("NOT_IMPLEMENTED", ...)` before writing anything (so no content-free `{timestamp,type}` file is created), and `environment.build.import_snapshot` keeps its LOAD_FAILED/PARSE_FAILED gates but then returns `SendError("NOT_IMPLEMENTED", ...)` instead of echoing the parsed object back with `success:true`. Both error messages point callers at the documented workaround (capture via `actor.list`+`property.get`, rebuild via the typed `environment.build.create_*`/`environment.spawn_*` verbs + `property.set`). The legacy cross-dispatcher (`environment.build` → `export_snapshot`/`import_snapshot`, EnvironmentHandler.cpp:~688-695) forwards to these same two handlers, so both reach paths are covered. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp`. Test: new `Source/EditorAutomationRpcGateway/Private/Tests/Environment/TestEnvironmentSnapshotStubsNotImplemented.cpp` invokes both handlers via `InvokeHandlerWithCapture` and asserts `bSuccess==false` and `ErrorCode=="NOT_IMPLEMENTED"`; the export case additionally asserts no stub file was written, and the import case writes a real parseable fixture file first so it gets past the LOAD/PARSE gates and exercises the actual apply-step fix (not a LOAD_FAILED short-circuit). Not compiled/tested here — left for the test phase.
- `#3-additional-fix-confirmed-live` `IN-REVIEW` reporter — Additional evidence: the `#2` honest-failure fix is confirmed behaving as intended on the live editor. An evening-blockout greybox task naturally reached `export_snapshot {path:"Saved/EnvSnapshots/evening_blockout.json"}` and got the new clean error (no longer the old silent `success:true` + content-free file): `[NOT_IMPLEMENTED] environment.build.export_snapshot is not implemented: it does not enumerate or serialize the level's environment actors, so the snapshot cannot restore the scene. To persist a look, capture the environment with actor.list + per-actor property.get and rebuild on restore with the typed spawn verbs (environment.build.create_*, environment.spawn_*) + property.set.` Oracle replay-confirmed the same string verbatim via `mcp__editor-automation__call` `environment.build.export_snapshot {path:"Saved/EnvSnapshots/oracle_replay2.json"}`, and verified **no** `oracle_replay2.json` file was written (the pre-write guard holds). The task agent then followed the prescribed `actor.list`+`property.get` workaround to capture the look itself. One residual sub-note for whoever closes this: the auto-generated wiki page `environment.build.export_snapshot.md` still reads "Export an environment snapshot to a JSON file" with a required `path` param and no hint the handler is a NOT_IMPLEMENTED stub — callers discover the stub only at call time. Not filed separately (it is the same export_snapshot stub, in-progress here, and the runtime error is honest/actionable); a wiki-src overlay note could pre-warn but is optional and within this ticket's scope.
- `#4-additional-golden-hour-reconfirm` `IN-REVIEW` reporter — Additional evidence: the `#2` honest-failure fix remains stable across an independent task (golden-hour/dusk sky-rig setup). Oracle replay-confirmed verbatim via `mcp__editor-automation__call` `environment.build.export_snapshot {path:"Saved/EnvSnapshots/DuskPreset.json"}` → `[NOT_IMPLEMENTED] environment.build.export_snapshot is not implemented: it does not enumerate or serialize the level's environment actors, so the snapshot cannot restore the scene. To persist a look, capture the environment with actor.list + per-actor property.get and rebuild on restore with the typed spawn verbs (environment.build.create_*, environment.spawn_*) + property.set.` (identical string to `#3`); verified **no** `Saved/EnvSnapshots/DuskPreset.json` was written (pre-write guard holds). The wiki pages still advertise both verbs without a stub warning (`environment.build.export_snapshot.md` "Export an environment snapshot to a JSON file" / `environment.build.import_snapshot.md` "Import an environment snapshot from a JSON file") — same residual wiki sub-note already captured in `#3`, no new ticket. No new defect; fix confirmed behaving as designed.
- `#5-additional-morning-rig-both-dirs-reconfirm` `IN-REVIEW` reporter — Additional evidence: the `#2` honest-failure fix reconfirmed across another independent task (clean outdoor mid-morning daylight rig: sky sphere + time_of_day=10.5 + directional sun + sky light + sun/skylight intensity, then a production lightmap bake — all of which succeeded — culminating in the snapshot save). This task naturally exercised **both** directions in one flow, unlike the export-only reconfirms in `#3`/`#4`. Oracle replay-confirmed both verbatim via `mcp__editor-automation__call`: `environment.build.export_snapshot {path:"Saved/EnvSnapshots/morning.json"}` → `[NOT_IMPLEMENTED] environment.build.export_snapshot is not implemented: it does not enumerate or serialize the level's environment actors, so the snapshot cannot restore the scene. To persist a look, capture the environment with actor.list + per-actor property.get and rebuild on restore with the typed spawn verbs (environment.build.create_*, environment.spawn_*) + property.set.` (identical to `#3`/`#4`), and `environment.build.import_snapshot {path:"Saved/EnvSnapshots/morning.json"}` → `[NOT_IMPLEMENTED] environment.build.import_snapshot is not implemented: it parses the file but applies nothing to the world (no spawn, no property set), so it cannot restore an environment. Rebuild the look manually with the typed spawn verbs (environment.build.create_*, environment.spawn_*) + property.set.`. Surfaced by the `environment.build.bake_lightmap` seed (which itself succeeded — async job ticket started); culprit is the `export_snapshot`/`import_snapshot` stub pair. The wiki pages still advertise both verbs without a stub warning — same residual wiki sub-note as `#3`/`#4`, no new ticket. No new defect; fix confirmed behaving as designed.
- `#6-methods-removed-in-cull` `IN-REVIEW` reporter — Note (superseded by removal): both `environment.build.export_snapshot` and `environment.build.import_snapshot` were removed entirely in the RPC cull recorded in [`E-rpc-cull-151-record`](E-rpc-cull-151-record.md) (they are 2 of the 55 stubs). The `#2` honest-failure (NOT_IMPLEMENTED) fix is therefore moot — the methods no longer exist — and the dedicated regression test (`Tests/Environment/TestEnvironmentSnapshotStubsNotImplemented.cpp`) plus the related cases in `Tests/World/TestEnvironmentHandlers.cpp` and the wiki-coherence assertions in `Tests/Infra/TestWikiHandler.cpp` were deleted with the handlers; the legacy `environment.build` cross-dispatch branch and the `docs/wiki-src/environment.md` overlay were trimmed too. The residual wiki sub-note from `#3`/`#4` is resolved by removal. Recorded for traceability.
