---
id: B-python-pre-gc-pass-heap-fault-root-cause
title: "Python's pre-GC gc.collect pass (OnPreGarbageCollect -> PyGC_Collect) faults on a corrupted interpreter heap; root cause unknown, python.execute only moves the pass"
status: OPEN
severity: Medium
category: bug
tags: [python, crash, garbage-collection, engine-fault, follow-up]
encounters: 2
lastSeen: 2026-09-06T06:03:44Z
rice: [2, 3, 1, 3]
priority: 22
---

# The pre-GC Python pass faults, and nothing has found what corrupts the heap

Follow-up to `B-python-execute-reentrant-gc-crash`, whose first slice is a mitigation only.
`python.execute` disables Python's cyclic collector while a script runs, so the engine's pre-GC hook
(`FPythonScriptPlugin::OnPreGarbageCollect` -> `PyUtil::CollectGarbage()` -> `PyGC_Collect()`)
skips its pass inside the script. The pass then runs at the first collect after the script. Both
recorded deaths faulted inside this pass with the same fault-address shape (`0x00007ffX00007383`).
One was in a live script (`B-python-execute-reentrant-gc-crash` `#1`). The other had no Python frame
on the stack: a Blueprint compile's collect ran seconds after a `python.execute` returned
(`B-python-execute-reentrant-gc-crash` `#7`/`#8`, and
`B-blueprint-compile-gc-kills-editor-via-python-pre-gc-hook`). So moving the pass out of the script
is not evidence that it is safe.

Remaining slices:
1. **Root cause.** Find what leaves a corrupted tracked object in the interpreter heap. The likely
   shape is a GC-tracked object freed without being untracked, or a half-overwritten `PyGC_Head`.
   Candidates: engine wrapper objects bound in a script, and private-scope `sys.modules` restore
   dropping extension modules. This needs a controlled repro on a disposable editor, never a
   shared one. `#8`'s suggested probe: bind UObjects in `python.execute`, then run `obj gc` on an
   idle editor.
2. **Integration repro.** Run `MaterialEditingLibrary.recompile_material` from `python.execute` on
   a material that fails to compile (`#1`'s shape), in a disposable editor. Use it to confirm the
   mitigation before the parent ticket reaches DONE.
3. **Script-side gc control.** A script that calls `gc.enable()` turns the mitigation off for the
   rest of its run. A script that calls `gc.collect()` runs the pass with a live frame. Decide
   whether `python.execute` should report either.

## History
- `#1-split-from-reentrant-gc-crash` `OPEN` developer — Split from
  `B-python-execute-reentrant-gc-crash` `#10`/`#11` (batch 5 review) so the remaining slices
  outlive that ticket's DONE. The `recorder.query` slice was dropped, because `recorder.query` runs
  caller code in a bundled-Python child process and never calls `ExecPythonCommandEx`
  (`RecorderQueryHandler.cpp`). Cross-link: `B-blueprint-compile-gc-kills-editor-via-python-pre-gc-hook`
  is the no-frame instance of the same fault.
