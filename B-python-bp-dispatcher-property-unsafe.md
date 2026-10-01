---
id: B-python-bp-dispatcher-property-unsafe
title: "python.execute: reading a Blueprint event-dispatcher property via get_editor_property gives a value whose is_bound() reports False while the dispatcher is bound, and dir() on that value SIGSEGVs the editor; the python wiki's 'Calls that crash the editor' list does not mention it"
status: OPEN
severity: High
category: bug
tags: [python, python.execute, delegate, multicast, event-dispatcher, crash, silent-wrong-data, docs]
encounters: 1
costly: 1
lastSeen: 2026-10-01T07:16:00Z
---

# Blueprint dispatcher properties are unsafe to read from python.execute

UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv` (PDS), plugin `1044f6de`, visible editor, listen PIE
with 1 client. The target was a live `W_RaceOnlineResultsForPilotFrame_C` widget whose Blueprint event
dispatchers `OnShowResults` / `OnClose` / `OnDeactivated` (FMulticastInlineDelegateProperty) are bound by
another widget (`W_RaceResultsHandler.ShowOnlineRaceFrameForPilot` → `bind_dispatcher`).

1. **Wrong data, silently.** `widget.get_editor_property('OnShowResults').is_bound()` returned `False` for
   all three dispatchers, both right after the bind and later. In the same state a real OS click on the
   button that calls `OnShowResults` *did* reach the bound handler (the standings table opened), so the
   dispatcher was bound. The `False` sent the investigation the wrong way for a whole session (I reported
   "dispatchers unbound" as the cause of a dead button).
2. **Crash.** `dir(widget.get_editor_property('OnShowResults'))` killed the editor:

```
LogPinWrightSafePoint: Running 'python.execute' (id=01a0f651-2153-762c-8c67-97f299669405) inline ...
LogCore: === Critical error: ===
Unhandled Exception: SIGSEGV: invalid attempt to write memory at address 0x0000000000000000
libpython3.11.so.1.0!UnknownFunction(0x25179c)
libpython3.11.so.1.0!PyObject_Vectorcall(+0x88)
...
libUnrealEditor-PinWright.so!AutoHandler_332_(FHandlerContext&)::$_0::operator()() const [.../PythonExecuteHandler.cpp:255]
LogCore: FUnixPlatformMisc::RequestExit(1, GenericPlatformmallocCrash::Malloc.OutOfMemory)
```

The faulting code is the engine's Python plugin, not PinWright. PinWright owns `python.execute` and the
`python` wiki page, whose "Calls that crash the editor" section is the agent's only protection, and it does
not mention delegate properties.

## Expected

- `python.md` "Calls that crash the editor": add Blueprint event-dispatcher (multicast inline delegate)
  properties read with `get_editor_property` — never introspect (`dir`, `repr` of members) the returned
  value, and never trust its `is_bound()`.
- Better: a typed read verb (or `property.get` support) that reports a multicast delegate's real
  invocation list (bound object + function name per entry) from C++, so binding state can be checked
  without Python.

**Workaround:** verify dispatcher binding by behaviour (real click + read-back of the resulting UI/log), not
from Python.

## History
- `#1-dispatcher-isbound-false-and-dir-crash` `OPEN` reporter - Filed from a PDS race-results-button repro, UE 5.8 Linux, plugin `1044f6de`. `is_bound()` returned False on bound dispatchers (proved bound by a working real click), and `dir()` on the same value SIGSEGV'd the editor (`Saved/Crashes/crashinfo-PDS-pid-2216420-*`). Costly: one editor crash and restart plus a wrong root-cause claim in an earlier report.
