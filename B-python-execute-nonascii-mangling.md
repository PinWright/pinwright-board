---
id: B-python-execute-nonascii-mangling
title: "python.execute corrupts non-ASCII chars in inline code (temp-file encoding path)"
status: WONTFIX
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
- `#2-mechanism-disproven-wontfix` `WONTFIX` developer — Mechanism refuted against verified UE 5.7 source; the temp-file round-trip is provably lossless for `×` (U+00D7), so the ticket's own static-trace suspicion was correct and the empirical anecdote is not attributable to this path. WRITE: `SaveStringToFile(Code, ...)` AutoDetect emits UTF-16LE+BOM for non-ANSI (FileHelper.cpp:681,697-702). READ: `RunFile` does NOT feed bytes to Python's parser — it reads via BOM-aware `LoadFileToString` (PythonScriptPlugin.cpp:1822), decoding the UTF-16LE+BOM back to the exact TCHAR. COMPILE: `EvalString(*FileStr,...)` → `Py_CompileString(TCHAR_TO_UTF8(...))` (PythonScriptPlugin.cpp:1864,1728), with the explicit comment that they avoid `PyRun_File` — so the UTF-16 bytes never reach a tokenizer that could choke. No stage mangles non-ASCII. The proposed `ForceUTF8WithoutBOM` only changes on-disk bytes; `LoadFileToString` decodes UTF-16LE+BOM and UTF-8 to the identical FString, so re-running `×` would behave identically — a fix that cannot alter the symptom. Basis: single unconfirmed encounter (`encounters: 1`), ticket itself flags "re-confirm before fixing"; JSON transport already delivers `×` correctly (reporter's own `blueprint.scs.get` note); zero-code workarounds exist (`chr(0xXXXX)`, or `mode: execute_statement`/`scope: public` bypass the temp write via `RunString`). No duplicate/regression/conflict per historian. If a variable-isolated re-repro ever surfaces it points to a DIFFERENT stage → new ticket. No code changed.
