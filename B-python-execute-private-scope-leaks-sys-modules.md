---
id: B-python-execute-private-scope-leaks-sys-modules
title: "python.execute scope:'private' does not isolate sys.modules, so an edited helper module keeps executing its previous bytecode and the call reports success"
status: IN-REVIEW
severity: Medium
category: bug
tags: [python, python-execute, sys-modules, module-cache, silent-noop, scope, docs]
encounters: 1
lastSeen: 2026-08-27T00:00:00Z
---

# An edited helper module silently keeps running its old code across `python.execute` calls

`python.execute` with the default `scope: "private"` isolates the *script's own top-level
namespace*, but `sys.modules` belongs to the embedded interpreter and is process-global. Any
module the script `import`s is therefore cached for the life of the editor. Editing that module
on disk and calling `python.execute` again re-runs the **new** entry script against the **old**
module, with no signal anywhere: `success: true`, no warning, no diagnostic.

This is the `silent-noop` shape — the edit lands on disk, the call reports success, and the
behaviour is the previous version's.

## Why the wiki reading is wrong

`Docs/wiki-src/python.execute.md` describes the parameter as `private (isolated, default)` and
says "`execute_file` calls get an isolated namespace; inline private scripts do not persist
top-level variables in UE console globals". An author factoring a long script into
`entry.py` + `helpers.py` reads "isolated" as "this call starts clean". It does not: only the
entry script's globals are fresh.

## Root cause (source)

`Plugins/PinWright/Source/PinWright/Private/Handlers/System/PythonExecuteHandler.cpp:156-170`
routes a private inline script through a temp file so UE honours
`EPythonFileExecutionScope::Private`, with the in-source comment naming the actual goal:

```cpp
    if (ExecMode == EPythonCommandExecutionMode::ExecuteFile &&
        FileScope == EPythonFileExecutionScope::Private &&
        !IsPythonFileCommand(Code))
    {
        // UE ignores FileExecutionScope for inline strings; a temp script forces
        // the private RunFile path so PIE object wrappers do not persist globally.
```

"so PIE object wrappers do not persist globally" is a narrower guarantee than "isolated".
`EPythonFileExecutionScope::Private` governs the **globals dict handed to the file**; it never
touches `sys.modules`, so imports are shared with every other `python.execute` call in the
session and with the UE Python console.

## Repro (measured live, this session, UE 5.8)

1. Write `gen_common.py` to a directory, containing some function set.
2. Run an entry script that does `sys.path.insert(0, thatDir); import gen_common as G` and
   calls into it. Observe it works.
3. Edit `gen_common.py` on disk — here, adding a new top-level function `seat_instance`.
4. Run the entry script again through `python.execute`, `mode: "execute_file"`,
   `scope: "private"` (the defaults).

Probe run, verbatim log:

```
MODULE was already in sys.modules BEFORE import: True
MODULE file: C:\...\scratchpad\gen_common.py
has seat_instance: False
```

`hasattr(G, 'seat_instance')` is `False` against a file on disk that plainly contains
`def seat_instance(...)`. The module object in memory is the pre-edit one.

The first symptom is worse than the probe, because the traceback actively misdirects. A call
into the stale module raised inside `make_hism`, and Python printed the frame's line number
from the **cached code object** against the source text read from the **current** file:

```
File "...\gen_common.py", line 172, in make_hism
    return unreal.Transform(unreal.Vector(px, py, z), rot, scl), dict(x=px, ...)
                                                                 ^^^^^^^^^^^^^^
AttributeError: 'HierarchicalInstancedStaticMeshComponent' object has no attribute 'is_registered'
```

Line 172 of the current file is inside `seat_instance`, not `make_hism`. A traceback that
quotes one function's source under another function's name is the diagnostic signature of this
defect, and it reads as file corruption rather than a stale import.

## Impact

Any script factored into more than one file. On this level build it made a placement pass run
its previous scatter logic, which spawned an actor holding 217 instances that then had to be
located and destroyed by hand before the pass could be re-run — the level had gained content
that no current source file would have produced. In a shared editor with several agents that is
a level-state change nobody can attribute to a source revision.

It does not take the editor down.

## Workaround

Force the reload explicitly in every entry script:

```python
import gen_common as G
import importlib
importlib.reload(G)
```

## Suggested fix, in preference order

(a) On `scope: "private"`, snapshot `sys.modules` before `ExecPythonCommandEx` and restore it
after, so a private call really is private and repeat runs are reproducible.

(b) Cheaper and still sufficient: document it. State on the `scope` parameter that `private`
covers the entry script's globals only, that `sys.modules` is shared for the life of the
editor, and name `importlib.reload` as the remedy. The current word "isolated" is what makes
the trap invisible.

(c) Orthogonal but related, found in the same run: UE 5.8 Python exposes neither
`register_component()` nor `is_registered()` on `UActorComponent`, so the natural spelling
after building a HISM from `unreal.new_object` is an `AttributeError`. This is the same family
as the `mark_render_state_dirty`-is-a-parameter trap already documented in
`Docs/wiki-src/level-building.instancing-and-scatter.md`; that page is where the missing pair
belongs.

## History
- `#1-sys-modules-snapshot-restore` `IN-REVIEW` developer — "Took route (a) in `Handlers/System/PythonExecuteHandler.cpp`: on the `execute_file` + `scope:"private"` path the handler now runs a snapshot script (`sys.modules` copied onto a stack on `sys`) before `ExecPythonCommandEx` and a restore script after it (drop every module the script added, put back everything it replaced or deleted), so an edited helper module is re-imported on the next call; a scrub that fails to run adds a Warning entry to the response `log` naming `importlib.reload` instead of failing silently. Documented the semantics and its two costs (re-import per call, `sys.path` not restored) on `Docs/wiki-src/python.md` under `### python.execute` plus the `scope` param description, and added (c)'s missing pair — `register_component()` / `is_registered()` carry no UFUNCTION and `AddComponentByClass` is `ScriptNoExport` on 5.8 — to `Docs/wiki-src/level-building.instancing-and-scatter.md`. Regression test `PinWright.python.execute.PrivateScopeReimportsEditedHelperModule` (`Tests/Infra/TestPythonPrivateScopeModuleIsolation.cpp`) writes a GUID-named helper module, runs an entry script through the handler, edits the module on disk and runs it again, asserting the second run reports `cached=False` and the edited value; the counterfactual was measured on UE's bundled CPython 3.11 and reports `cached=True value=FIRST`. Not compiled or run in-editor — build and suite are the verification."
