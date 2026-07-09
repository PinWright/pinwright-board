---
id: F-log-tail-filter-verbosity
title: "log: real log reading - tail with category/regex filters, category list, verbosity get/set"
status: WONTFIX
severity: High
category: feature
tags: [log, perception, diagnostics, parity-ue58]
---

# log: real log reading - tail with category/regex filters, category list, verbosity get/set

The entire `log` namespace is a no-op stub: `log.subscribe` returns `{subscribed:false, "Log streaming is not yet implemented."}` and `log.unsubscribe` tears down a device that was never created (`Source\PinWright\Private\Handlers\Debug\LogHandler.cpp`, 43 lines total). No other namespace reads logs; the wiki tells agents to go read `Saved/Logs/<Project>.log` off disk themselves. Log access is the single perception area where UE 5.8's built-in toolset beats PinWright.

UE 5.8 parity evidence: `LogsToolset` (4 tools, `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\EditorToolset\Source\...\LogsToolset.h`): `GetLogEntries` (tails the session log with category + regex filters), `GetLogCategories`, `GetVerbosity`/`SetVerbosity`.

Related (scope beyond, do not duplicate): `E-log-stream-no-readback-method` (IN-REVIEW) covers the bare "no read method" gap. This ticket is the full parity surface: filters, category enumeration, verbosity control, structured entries.

Proposed scope:
- `log.tail(count, category?, pattern?, minVerbosity?)` - structured entries {time, category, verbosity, message} from an in-memory ring buffer (FOutputDevice capture) with disk-log fallback.
- `log.list_categories()`.
- `log.get_verbosity(category)` / `log.set_verbosity(category, level)`.
- Rework or remove the dead subscribe/unsubscribe pair to match whatever lands (align with the IN-REVIEW ticket outcome).

Acceptance: UE_LOG a marker at a known category/verbosity, `log.tail` with category filter and with regex filter both return it; set_verbosity suppresses/reveals as expected.

## History
- `#1-log-namespace-stub` `OPEN` reporter — log namespace is a 43-line no-op stub; agents must read Saved/Logs off disk. Epic 5.8 LogsToolset (tail+filters+verbosity) is the parity target and Epic's only perception win; scope extends E-log-stream-no-readback-method (IN-REVIEW) with filters/categories/verbosity.
- `#2-wontfix-redundant-existing-paths` `WONTFIX` developer — Declined per adversarial objection, verified against synced source (plugin @ HEAD aee3a1a). Defect premise is TRUE (log namespace is a 43-line no-op stub — `LogHandler.cpp` registers only `log.subscribe`/`log.unsubscribe`, no read verb; `PluginState.h:66-67,113` hold a `LogCaptureDevice` slot subscribe never fills, only Reset at `PluginState.cpp:99`), but this is speculative capability-expansion, not observed friction, and every proposed verb already has a working, DOCUMENTED path: (1) `log.tail` ↔ tail `Saved/Logs/<Project>.log`, the on-disk file E-log-stream-no-readback-method's shipped overlay (`Docs/wiki-src/log.md:5-7`) already steers callers to; (2) `list_categories`/`get_verbosity`/`set_verbosity` ↔ `editor.console_command` (`EditorCommandHandler.cpp:283-308`, `GEditor->Exec`) with UE's `Log list` / `Log <Cat>` / `Log <Cat> <Verbosity>`. The only OBSERVED friction (E's 14-call task, self-rated "clean/Mostly smooth") was one avoidable `log.poll`→UNKNOWN_ACTION probe, ALREADY docs-fixed by E (IN-REVIEW). Motivation is parity with UE 5.8 `LogsToolset`, an EXPERIMENTAL engine toolset (ticket's own path: `.../Experimental/Toolsets/...`) — gold-plating, not a needed capability. Real cost/risk: the proposed in-memory ring buffer reuses `FMcpOutputCapture`, whose `Serialize` does an UNLOCKED `Lines.Add` (`LogUtils.cpp:12`) — a persistent `GLog` device receives `Serialize` from arbitrary threads while handlers read on the game thread (data race the current scoped usage avoids); `set_verbosity` also mutates global log-category state other tooling/tests depend on. Implementing would additionally force rewriting E's IN-REVIEW overlay and deleting its `FWikiHandlerLogNamespaceDocumentsNoReadbackTest` (`TestWikiHandler.cpp:1353-1378`, hard-asserts "no read-back"/"log.poll"/"do not probe") in the same commit. The red test (`Tests/EditorOps/TestLogTail.cpp`) reproduced:true only confirms the feature is unbuilt (a feature-gap trivially "reproduces"), not that it should be built; removed the stray untracked red test since the feature is declined. Not a duplicate of E (E is the Low docs note; F was the full parity surface) — E already covers the real need.
