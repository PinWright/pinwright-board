---
id: B-python-execute-nonascii-mangling
title: "python.execute corrupts non-ASCII chars in inline code (temp-file encoding path)"
status: OPEN
severity: Medium
category: bug
tags: [python, encoding, temp-file, non-ascii]
encounters: 1
lastSeen: 2026-06-30T00:00:00Z
---

# python.execute corrupts non-ASCII chars in inline code (temp-file encoding path)

In default `execute_file` + `private` mode, the handler writes the inline `code`
to a temp `.py` (`Intermediate/PinWright/Python/InlinePython_*.py`) and runs that
file — `PythonExecuteHandler.cpp:156-164`, written via
`FFileHelper::SaveStringToFile(Code, *TempScriptPath)` (line 157) with the
**default** `EEncodingOptions::AutoDetect`, not a forced UTF-8.

Empirically (reporter): a literal `×` (U+00D7) inside
`unreal.load_asset('/App/.../Tower_7×6')` failed to load until rebuilt as
`chr(0x00d7)`. The same `×` path passed to C++ handlers (e.g. `blueprint.scs.get`)
over JSON works, so the JSON transport is fine and the corruption is specific to
this temp-file path. Many gate assets use `×` (`Launch_Gate_7×6`, `Gate_5×5`, …).

**Mechanism is unconfirmed against UE 5.6 source.** AutoDetect writes non-ASCII as
UTF-16LE+BOM (`FileHelper.cpp:681,695-703`; `IsPureAnsi` `>0x7f`, `CString.h:111`),
and `RunFile` reads it back BOM-aware (`BufferToString`, `FileHelper.cpp:149-158`)
then compiles `Py_CompileString(TCHAR_TO_UTF8(...))` (`PythonScriptPlugin.cpp:1755,
1797,1661`) — a lossless round-trip on paper. So the exact corrupting stage is not
pinned; trust the live repro over the static trace and re-confirm before "fixing".

**Workaround:** build non-ASCII chars with `chr(0xXXXX)` inside the script.
**Fix:** write the temp `.py` with `EEncodingOptions::ForceUTF8WithoutBOM` (Python's
canonical source encoding; removes the AutoDetect UTF-16 branch), then re-run the
original `×` repro to verify it actually resolves the symptom.

## History
- `#1-initial-repro` `OPEN` reporter — `python.execute {code: "...load_asset('/App/.../Tower_7×6')..."}` mangled `×`; `chr(0x00d7)` workaround loaded the asset. Handler writes inline code to a temp .py via SaveStringToFile/AutoDetect (PythonExecuteHandler.cpp:157); UE 5.6 write→read→compile trace round-trips losslessly, so the corrupting stage is unconfirmed — needs a live re-repro. Proposed fix: ForceUTF8WithoutBOM on the temp write.
