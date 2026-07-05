---
id: F-log-tail-filter-verbosity
title: "log: real log reading - tail with category/regex filters, category list, verbosity get/set"
status: OPEN
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
