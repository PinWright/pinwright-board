---
id: E-board-commit-aborts-on-git-stderr
title: "board-commit.ps1 aborts on git's STDERR instead of using its own retry loop — a CRLF warning or a contended index.lock kills the commit and the ticket silently stays unstaged"
status: OPEN
severity: Medium
category: ergonomic
tags: [board, board-commit, scripts, powershell, git, index-lock, nativecommanderror, tooling]
encounters: 2
lastSeen: 2026-09-02T19:55:00Z
---

# `board-commit.ps1` turns git's stderr into a terminating error, so its retry loop never runs

## Symptom

`Plugins/PinWright/scripts/board-commit.ps1` failed to commit a new ticket on eight
consecutive invocations across ~20 minutes, for two different reasons, and in both
cases the script never reached the retry logic it already contains.

Call:

```
powershell -File X:\src\unreal\EAContentExamples58\Plugins\PinWright\scripts\board-commit.ps1 `
  -Board X:\src\unreal\.pinwright-board `
  -Files X:\src\unreal\.pinwright-board\B-level-load-dirty-world-memory-leak-fatal.md `
  -Message "Track B-level-load-dirty-world-memory-leak-fatal"
```

First failure — a line-ending warning:

```
git.exe : warning: in the working copy of 'B-level-load-dirty-world-memory-leak-fatal.md',
LF will be replaced by CRLF the next time Git touches it
At ...\board-commit.ps1:72 char:9
+         & git -C $boardFull add -- $fileArgs 2>&1 | Out-Null
+         ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (warning: in the... Git touches it:String)
                              [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
```

Exit code 1, nothing committed. The file **was** staged (`git status --porcelain`
showed `A `), so the caller who reads only the exit code concludes the ticket was not
filed while the index says otherwise.

Second failure — a contended index:

```
git.exe : fatal: Unable to create 'X:/src/unreal/.pinwright-board/.git/index.lock': File exists.
```

Same `NativeCommandError` shape, same abort at line 72, seven retries in a row over
15 minutes, every one dying on the first `git add` rather than entering the loop.

## What I expected

The script already handles both cases. Line 72-79:

```powershell
for ($i = 0; $i -lt $Retries; $i++) {
    & git -C $boardFull add -- $fileArgs 2>&1 | Out-Null
    $out = (& git -C $boardFull commit -m $Message -- $fileArgs 2>&1) | Out-String
    if ($LASTEXITCODE -eq 0) { $committed = $true; break }
    ...
    if ($out -match 'index\.lock|cannot lock ref|...') {
        Start-Sleep -Milliseconds (200 + (Get-Random -Maximum 400)); continue
    }
```

The `index.lock` branch is exactly right for the second failure. It never executes.

## Root cause

`git` writes warnings and fatals to **stderr**. Under Windows PowerShell 5.1 (which
`powershell -File` selects), `2>&1` on a native command converts stderr lines into
`ErrorRecord`s, and with the default `$ErrorActionPreference` those become a
**terminating** `NativeCommandError` that unwinds the script. So the abort happens
*inside* the `git add` on line 72, before `$LASTEXITCODE` is read and before the retry
branch can match. Setting `$ErrorActionPreference='Continue'` or `'SilentlyContinue'`
around the invocation does not help either, because the conversion happens at the
native-command boundary rather than at the cmdlet error stream.

Two independent triggers, both routine on this board:

1. **CRLF warning** — the board has no `.gitattributes` and `core.autocrlf` is on, so
   every LF-authored ticket produces the warning on its first `git add`. Any agent
   writing a ticket from a tool that emits LF hits this on its very first commit.
2. **`index.lock` contention** — three hosts share this board and commit constantly.

## Suggested fix

Any one of these closes it; the first is the smallest:

- Redirect stderr to a file or to `$null` at the native level rather than merging it
  into the PowerShell error stream: `& git ... 2>$null` for `add`, and capture the
  commit output with `--porcelain`/`$LASTEXITCODE` instead of `2>&1 | Out-String`.
- Wrap each native call in `try { } catch { }` **and** read `$LASTEXITCODE` after, so
  a `NativeCommandError` becomes a loop iteration rather than an exit.
- Move the retry test off the captured text and onto `$LASTEXITCODE` plus a probe of
  `.git/index.lock`, and keep retrying while the lock file exists and is younger than
  a threshold.

Independently: **add a `.gitattributes` to the board** (`*.md text eol=lf` or
`* -text`) so the CRLF warning stops being emitted at all. That alone removes the
more common of the two triggers.

## Workaround

Re-run the script until another host's commit happens to sweep the staged file in.
That is what happened here: my ticket was eventually committed by a *different*
agent's `board-commit` run (`904d070`, which recorded it under that agent's own commit
message). The ticket landed, but the commit message and authorship are wrong, and
nothing in my session could tell that it had.

## History

- `#1-filed` `OPEN` reporter — Hit while filing `B-level-load-dirty-world-memory-leak-fatal` during the FPS VFX build on UE 5.8 / EAContentExamples58. Eight invocations over ~20 minutes, all exit 1, all aborting at `board-commit.ps1:72` with `FullyQualifiedErrorId : NativeCommandError` — the first on git's CRLF warning, the next seven on `fatal: Unable to create '.../.git/index.lock': File exists.` while other hosts committed. The script's own `index.lock` retry branch (line 79) is correct and simply unreachable, because the native-command stderr merge raises a terminating error inside the `git add` on line 72 before `$LASTEXITCODE` is ever consulted; `$ErrorActionPreference='Continue'` and `'SilentlyContinue'` around the invocation both failed to suppress it. Verified the file really was staged the whole time (`git status --porcelain` -> `A `), so the failure is invisible to a caller that trusts the exit code. It was finally committed as a side effect of another agent's run, under that agent's message. Severity Medium: no data loss and a workaround exists, but every agent on this board is told to file tickets at the moment of discovery, and this makes the very first filing of a session fail in a way that reads as "the board rejected my ticket". Suggested a `.gitattributes` on the board as the cheap half of the fix — it removes the CRLF trigger outright.
- `#2-backlog-of-eight-and-pathspec-cannot-sweep` `OPEN` reporter — Reproduced on EAContentExamples58 while filing against the shared board from a third agent. Both triggers fired in one sitting: seven consecutive `index.lock` aborts, then, once the lock cleared, the CRLF warning abort — same terminating `NativeCommandError` at `board-commit.ps1:72`, same staged-but-uncommitted `A ` outcome. Two facts this adds beyond `#1`: **(a) the backlog is not one ticket, it is eight.** `git diff --cached --name-only --diff-filter=A` on the board currently lists eight new tickets staged and never committed — `B-niagara-module-input-cannot-link-particle-attribute`, `B-niagara-module-input-dotted-subinput-silent-noop`, `B-niagara-module-input-enum-display-name-rejected`, `B-niagara-module-input-unqualified-subinput-pin`, `B-niagara-static-switch-bool-writes-zero-on-int-switch`, `E-niagara-module-input-enum-value-rejects-label`, `F-metasound-no-render-to-pcm`, and `B-screenshot-designer-leaves-designer-open-compile-crash`. That last one is **Critical** and carries three independent witnesses who each believe they filed it; none of them is in `HEAD` (`git cat-file -e HEAD:<file>` fails) and none has reached the backup remote. `git ls-files | wc -l` = 1606 vs `git ls-tree -r --name-only HEAD | wc -l` = 1598 confirms the count. **(b) The workaround in this ticket does not generalise, and the ticket's own design note says why.** `#1` records being rescued when "another agent's `board-commit` run swept the staged file in" (`904d070`). It cannot normally do that: the script commits by **pathspec** (`git commit -m $Message -- $fileArgs`, line 77) precisely so a run never records a sibling host's in-flight edit, so only a run naming that exact path can commit it. A ticket whose author's own runs all abort therefore stays staged indefinitely — which is what the eight-file backlog is. Anyone relying on "re-run until someone sweeps it" is relying on a sibling happening to name the same file. Severity left at `Medium`: impact class is unchanged (nothing is lost — the ticket text is on disk and staged, and a mutex-guarded manual `git add` + pathspec `git commit` lands it), and per the board README `encounters` is a work-ordering tiebreak, never a severity input. But the reach is wider than `#1` could see: this is not "the first filing of a session fails", it is "an entire session's filings accumulate unpublished, including Criticals, until someone inspects the index by hand". The `.gitattributes` half-fix `#1` proposes is still absent from the board, so trigger 1 still fires on every LF-authored ticket — mine hit it on the first `git add`.
