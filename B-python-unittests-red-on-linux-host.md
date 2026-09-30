---
id: B-python-unittests-red-on-linux-host
title: "The Content/Python unittest suite has 3 failures on a Linux host: two editor_run_tests tests expect the Windows -Cmd binary and a supervisor test expects a start failure that systemd-run hides"
status: IN-REVIEW
severity: Medium
category: bug
tags: [python, unittest, linux, editor_run_tests, supervisor, systemd-run, test-harness]
encounters: 1
costly: 0
lastSeen: 2026-09-30T00:00:00Z
---

# Python unit tests assume a Windows host

`python -m unittest discover tests` from `Content/Python`, run on the Linux host with the bundled
interpreter (`/sdb-disk/UE_5.8/Engine/Binaries/ThirdParty/Python3/Linux/bin/python3`) on an
unmodified tree (plugin HEAD `2580e7f4`, 2026-09-30): `Ran 366 tests ... FAILED (failures=3,
skipped=4)`.

- `test_mcp_proxy_editor_start.ProxyEditorRunTestsTest.test_windowless_run_launches_the_suite_contract_and_returns_once_tests_start`
  and `...test_headless_run_adds_nullrhi_and_keeps_the_windowless_binary` assert
  `argv[0] == _editor_cmd_from_editor(editor_exe)` (the `UnrealEditor-Cmd.exe` twin), but
  `pinwright_supervisor.suite_executable` returns the plain editor whenever the real `os.name != "nt"`
  (documented Linux behaviour). The tests do not pin the platform.
- `test_pinwright_supervisor.SpawnSupervisedTest.test_start_failure_is_reported` expects
  `RuntimeError` for a non-existent executable, but on a host where `systemd-run --user --scope`
  works the supervisor's `Popen` starts `systemd-run`, which succeeds, so no start failure reaches the
  handoff.

A suite that is always red on Linux (the primary PinWright dev host here) hides new regressions
among known failures. Plugin `CLAUDE.md` still quotes `366 tests (1 skipped)`, a Windows figure.

**Asked for:** pin the platform in the two proxy tests (or assert the per-platform binary), and make
the supervisor test either skip under a working systemd scope or assert the Linux outcome
(`COMMAND_EXIT_NONZERO` from systemd-run's exit).

## History
- `#1-three-reds-on-linux` `OPEN` reporter - Found by a fixer agent's baseline run before touching any Python (plugin HEAD `2580e7f4`, clean `Content/Python`): 366 tests, failures=3 exactly as listed above, reproduced identically after its unrelated change (385 tests, same 3). severity rationale: impact=Low (test-harness false red, no product defect) x reach=every Python suite run on a Linux host, bumped up one -> Medium. costly=0.
- `#2-one-code-fix-one-windows-only-split` `IN-REVIEW` developer - Root causes differed. (a) `test_start_failure_is_reported` exposed a real Linux defect, fixed in production code: under a `systemd-run` scope the supervisor's `Popen` starts `systemd-run`, not the target, so a missing executable handed off as a started child (`started pid N`, plus a misleading `UNCAPPED: scope memory.max never read back ... memory controller is likely not delegated` because the scope died at once) and surfaced only later as `COMMAND_EXIT_NONZERO exit=1` (reproduced on this host with `spawn_supervised([<missing>], kind="command", ...)`). `Content/Python/pinwright_supervisor.py` `_start_child` now checks `shutil.which(argv[0])` on the Linux branch before prefixing and raises `FileNotFoundError`, so the handoff carries the error exactly as on Windows and on the uncapped path; the existing test is the regression test (failed before, passes now, unchanged). (b) The two `ProxyEditorRunTestsTest` failures were a Windows-only assertion inside otherwise platform-neutral tests: production is correct on Linux (plain `UnrealEditor`, documented, pinned by `SuiteExecutableTest.test_linux_windowless_is_plain_binary`). The `argv[0] == UnrealEditor-Cmd` assertion moved out of both into one new test, `test_windowless_runs_use_the_cmd_console_twin` (offscreen + headless), `@unittest.skipUnless(sys.platform == "win32", ...)`: the -Cmd console twin exists only on Windows. The suite-contract test keeps running on Linux; the headless test is renamed `test_headless_run_adds_nullrhi_to_the_offscreen_flags` (it no longer asserts the binary). Result on this Linux host: `Ran 386 tests ... OK (skipped=5)`. Skips reviewed, all legitimate: `test_windowless_runs_use_the_cmd_console_twin` (Windows -Cmd twin), `ResolveEditorExeTest.test_engine_association_last_resort` (the `C:\UE_<ver>` fallback is only added when `os.name == "nt"`, `_association_engine_roots`; Linux resolution via Install.ini has its own tests), `TreeKillSurvivalTest` x2 (WMI `Win32_Process.Create` + `taskkill /T`), `PackageHostVocabularyGuardTests.test_exact_previously_excluded_file_classes_are_rejected` (host-configuration skip: needs the untracked local `scripts/package-fab.local.ps1`, absent on every fresh clone regardless of OS). Plugin `CLAUDE.md`'s `366 tests (1 skipped)` figure left for the manager to update after all fixers land.
