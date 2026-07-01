---
id: F-recorder-producer-integration-guide
title: "Recorder docs cover queries but not host-side telemetry integration (editor-only / proxy pattern)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [recorder, telemetry, docs, wiki-overlay, packaging]
---

# Recorder docs cover queries but not host-side telemetry integration (editor-only / proxy pattern)

The `recorder.*` namespace documents the READ/query surface well (describe_session,
query, get_series, get_state, summarize_change, find_events, list_segments,
list_sessions) via auto-generated method pages in `wiki-generated/recorder.md`. But
there is NO `docs/wiki-src/recorder.md` editorial overlay and NO guide for the
PRODUCER side: how a host project pushes telemetry via `FTelemetryRecorder::LogVariable`
/ `LogEvent` / `RegisterObject` / `StampDomain`, the `EA_TELEMETRY_LOG` guard macro, and
the `IsRecording()` no-op gate (`Source/EditorAutomationRecorder/Public/TelemetryRecorder.h`
lines 25-136).

Critically undocumented: how to integrate the producer calls EDITOR-ONLY so they are
excluded from packaged/shipping builds. The `EditorAutomationRecorder` module is
`"Type": "Editor"` (`EditorAutomationRpcGateway.uplugin` lines 20-21), so a host runtime
module that calls it must either gate the dependency
`if (Target.bBuildEditor) PrivateDependencyModuleNames.Add("EditorAutomationRecorder")`
plus wrap calls in `#if WITH_EDITOR`, or — cleaner — use a proxy: a runtime no-op
interface implemented by an editor-only module so the runtime module never links the
editor recorder at all. The only existing guidance is one sentence in the header class
comment ("host-project modules may link against it", `TelemetryRecorder.h` line 23) —
nothing about packaging, `WITH_EDITOR`, or the proxy pattern.

**Session evidence (replayable):** integrating the recorder into host PDS for per-frame
replay telemetry exposed the gap: (1) `recorder.describe_session` on recent sessions
returned `objects:[], variables:[]` — only events captured, so a producer push was
needed; (2) grepping the host for `FTelemetryRecorder|EA_TELEMETRY_LOG|EditorAutomationRecorder`
returned ZERO host usage to copy from; (3) no `docs/wiki-src/recorder.md` exists; (4) the
`bBuildEditor`-gated dep + `#if WITH_EDITOR` recipe had to be worked out from the header.

**Workaround:** read `TelemetryRecorder.h` and derive the `if (Target.bBuildEditor)`
module dependency + `#if WITH_EDITOR` call gating by hand.
**Fix:** add a `docs/wiki-src/recorder.md` overlay plus a `recorder.integration` topic
page (per the TopicNodes convention) documenting the **plugin-specific** producer API
(LogVariable/LogEvent/RegisterObject/StampDomain, the `EA_TELEMETRY_LOG` guard, the
`IsRecording` no-op gate, and the fact the plugin already owns session lifecycle +
per-tick render-domain stamping so a host only emits values/events) AND the one
plugin-specific packaging constraint: the `EditorAutomationRecorder` module is
`"Type": "Editor"`, so a host runtime module that links it must gate the dependency
`if (Target.bBuildEditor) PrivateDependencyModuleNames.Add("EditorAutomationRecorder")`
and wrap calls in `#if WITH_EDITOR`. The cleaner runtime-no-op-interface + editor-only-impl
**proxy pattern is standard, widely-documented UE practice** (not plugin-specific), so it is
an OPTIONAL one-line aside, not core content — do not gold-plate the page with a full
proxy-class recipe.

## History
- `#1-initial-repro` `OPEN` reporter — Filed after integrating the recorder into host PDS: producer API + editor-only/packaging integration is undocumented (no `docs/wiki-src/recorder.md`; module Type=Editor in the .uplugin; only guidance is the one-line `TelemetryRecorder.h` class comment; `wiki-generated/recorder.md` is query-methods-only). Had to derive the `bBuildEditor`-gated dep + `#if WITH_EDITOR` gating from the header.
- `#2-reword-and-implement` `IN-REVIEW` developer — Reword (adversarial lens): the **Fix:** mandated the runtime-proxy pattern as core content, but that is standard, widely-documented UE editor-only-from-runtime practice, not plugin-specific — rewrote **Fix:** to center on the plugin-specific producer API + the one `bBuildEditor`/`WITH_EDITOR` gate and demote the proxy pattern to an explicitly OPTIONAL aside (no gold-plating). Implemented: added namespace overlay `docs/wiki-src/recorder.md` (producer side: `FTelemetryRecorder` `LogVariable`/`LogEvent`/`RegisterObject`/`StampDomain`, the `EA_TELEMETRY_LOG` guard + `IsRecording` no-op gate, the fact the plugin already owns lifecycle + render-domain stamp so a host only emits, and the `"Type": "Editor"` packaging constraint with the `if (Target.bBuildEditor)` dep + `#if WITH_EDITOR` recipe) and topic page `docs/wiki-src/recorder.integration.md` (full host recipe: producer API table, Case-A editor-module / Case-B runtime-module linking, optional runtime-proxy aside) auto-enrolled into TopicNodes via the wiki discovery walk. Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` — `FWikiHandlerRecorderDocumentsProducerSideTest` renders the `recorder` namespace page and asserts the overlay-exclusive markers `FTelemetryRecorder`/`EA_TELEMETRY_LOG`/`bBuildEditor`/`WITH_EDITOR` (none appear in the recorder query-handler registrations, so the test fails iff the overlay is reverted); `FWikiHandlerRecorderIntegrationTopicResolvesTest` renders the `recorder.integration` topic page (fails as `# Not found:` if the topic file is removed) and asserts the proxy pattern is framed as Optional.
