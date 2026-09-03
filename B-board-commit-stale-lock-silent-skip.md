---
id: B-board-commit-stale-lock-silent-skip
title: "board-commit.ps1 cannot land a commit against a stale .git/index.lock (2 s retry budget) and reports success by exiting 0 when the mutex times out, so tickets silently never reach the board"
status: OPEN
severity: High
category: bug
tags: [board, tooling, board-commit, git, index-lock, silent-failure, concurrency]
encounters: 2
---

# A stale `index.lock` makes `board-commit.ps1` drop tickets without failing

## Symptom

`board-commit.ps1` returns without committing, and two of its three exit paths do not report
that. A ticket file is left untracked in the board working tree while the calling agent believes
it is on the board.

Observed over roughly six minutes: every invocation failed with

```
fatal: Unable to create 'X:/src/unreal/.pinwright-board/.git/index.lock': File exists.
```

against a **0-byte** `index.lock` that persisted for more than five minutes while
`Get-Process git` returned **zero** processes and the script's own
`Global\pinwright-board-commit` mutex was free (the script had acquired it — it got as far as
`git add`). That is a stale lock, not contention.

## Why the retry loop cannot cover it

The contention retry is sized for a transient race, not a stale file:

```powershell
for ($i = 0; $i -lt $Retries; $i++) {          # $Retries = 5
    & git -C $boardFull add -- $fileArgs 2>&1 | Out-Null
    $out = (& git -C $boardFull commit ... ) | Out-String
    ...
    if ($out -match 'index\.lock|cannot lock ref|...') {
        Start-Sleep -Milliseconds (200 + (Get-Random -Maximum 400)); continue
    }
```

Five attempts at 200-600 ms is a **~2 second** total budget. Nothing in the script inspects the
lock's age or whether any `git` process actually holds it, so a lock left behind by a git process
that died mid-write is permanently fatal to every future call on that machine.

## The silent paths

1. **Mutex timeout** — `if (-not $held) { Write-Output '... skipping commit'; exit 0 }`. Exit code
   0 with nothing committed. A caller checking `$LASTEXITCODE` is told it succeeded.
2. **Retries exhausted** — the `for` loop ends without setting `$committed`, control falls through
   `finally`, and the script ends. No `exit 1`, no summary line: the only evidence is git's raw
   stderr, which a caller redirecting output (`*>$null`, common in retry wrappers) never sees.

Only path 3, a git error that does not match the lock regex, does `exit 1` with a message.

## Impact

`High`. The board is the project's memory for plugin defects, and this makes filing *look* like it
worked while the ticket exists only as an untracked file in a working tree with no remote of its
own. In this session `B-synth-layer-peak-excludes-layer-gain.md` was committed for the first time
by an unrelated agent's later `git commit`, under `create mode 100644` — meaning it had sat
untracked since it was filed, roughly an hour earlier, by an agent that had every reason to
believe it was on the board. At the time of writing `git status --porcelain` on the board lists
45 entries, so the exposure is not one file.

**Workaround:** never trust the exit code. After calling the script, verify with
`git -C <board> ls-tree --name-only HEAD -- <file>` and treat empty output as failure. To land a
ticket when the lock is stale, confirm no `git` process is running, delete `.git/index.lock`, and
commit with a **pathspec** commit (`git commit -m ... -- <files>`), which preserves the script's
one genuinely important safety property: only the named files are recorded, so a sibling host's
staged edit for a different ticket is never swept in.

**Fix:** (a) treat a lock as stale when no `git` process holds it and its mtime exceeds a
threshold, and remove it before retrying; (b) raise the retry budget well past a couple of
seconds, with backoff; (c) make both silent paths `exit 1` with a message naming the ticket that
did not land, so a caller cannot mistake "skipped" for "committed"; (d) optionally verify
`ls-tree HEAD` before returning success, since the script already knows the pathspec.

## History
- `#1-filed` `OPEN` reporter — Hit while filing two `audio.synth` tickets after synthesizing 20 impact/explosion waves. Every `board-commit.ps1` call over ~6 minutes died on `fatal: Unable to create '.../.git/index.lock': File exists` against a 0-byte lock older than five minutes with zero `git.exe` processes and the script's global mutex free. A 20-attempt wrapper loop (8 s apart, output suppressed) never committed and exited 0; the tickets stayed `??` in `git status` the whole time. Landed them manually as c848e36 (2 files changed, 183 insertions) after removing the stale lock and using a pathspec commit, then verified with `git ls-tree --name-only HEAD` and `git merge-base --is-ancestor c848e36 HEAD`. The severity driver is not the lock but the reporting: the mutex-timeout path does `exit 0` after printing "skipping commit", and the retries-exhausted path falls out of the loop with no `exit 1` and no summary, so an agent that checks the exit code — or that suppresses output while retrying — is told a ticket was filed when it was not. Evidence that this already bit someone else: `B-synth-layer-peak-excludes-layer-gain.md`, filed by a different agent about an hour earlier, appeared in my commit under `create mode 100644`, i.e. it had never been committed at all.
- `#2-second-encounter-nonzero-lock` `OPEN` reporter — Hit again ~40 min later on the same board while appending a second encounter to `B-asset-save-pie-failure-reports-pendingflush` (smoke-grenade emitter stream, EAContentExamples58). 24 `board-commit.ps1` invocations across ~9 minutes all died on `fatal: Unable to create 'X:/src/unreal/.pinwright-board/.git/index.lock': File exists.`, surfaced as a PowerShell `NativeCommandError`. Two details differ from #1 and narrow the diagnosis: the lock was **131072 bytes, not 0**, and its mtime stayed frozen at 22:51 local for the whole 15-minute window, while `tasklist` showed exactly **one live `git.exe` (pid 102448) holding ~2.6 MB** — i.e. a real git process that had stalled mid-index-write rather than a lock orphaned by a dead process. So a stale-lock recovery that keys only on "no git process is running" or "lock is 0 bytes" would not have fired here; it needs an mtime-age threshold too. Net effect matched #1: the ticket edit sits in the board working tree uncommitted and the agent has no sanctioned way to land it (`board-commit.ps1` is the only permitted path, and it has no age-based recovery and no way to report "held by a live but stalled git"). Suggest the fix also emit the holding pid and the lock's age so a caller can tell "wait" from "recover".
- `#3-add-exit-code-ignored-so-the-stale-lock-surfaces-as-a-pathspec-error` `OPEN` reporter — Third encounter, 2026-09-02 ~20:10Z, from the FPS impact-VFX authoring wave (EAContentExamples58). Adds a **second, different failure shape** for the same root cause, and a one-line fix that is not in `#1`/`#2`. A stale `X:\src\unreal\.pinwright-board\.git\index.lock` (mtime 2026-09-02 22:51 local, still present ~17 minutes later, alongside three zero-byte `next-index-*.lock` files from 22:11 / 22:22 / 22:28) made every `git add` inside the script fail. The script pipes `git add`'s output to `Out-Null` and **never reads its exit code** (`& git -C $boardFull add -- $fileArgs 2>&1 | Out-Null`, line ~74), so the failure is invisible and execution continues to the pathspec commit. The commit then reports a *misleading* error naming the files rather than the lock: `board-commit: git commit failed: error: pathspec 'B-asset-exists-duplicate-false-negative-in-pie.md' did not match any file(s) known to git`. That message sends the caller to check filenames and paths — both of which were correct; `git status --porcelain --untracked-files=all` listed both as `??` and `git check-ignore` exited 1 for both, so they were plain untracked files that `add` simply never processed. Four retries at 700 ms produced the identical error, i.e. the retry budget cannot outlast an abandoned lock, which is `#1`'s point reached by a different route. Note the shape difference that matters for a fixer: previously-staged files commit fine while newly-created ones fail, so a partial run can land some tickets and silently drop others in the same invocation. Fixes: (a) check `$LASTEXITCODE` after `git add` and fail loudly; (b) detect an `index.lock` older than a threshold (say 60 s) and report it by name in the error rather than letting it surface as a pathspec complaint; (c) sweep the zero-byte `next-index-*.lock` files, which appear to accumulate one per dead host. Two ticket files created in this session remain uncommitted on disk because of this: `B-asset-exists-duplicate-false-negative-in-pie.md` (new) and an appended History entry on `B-niagara-set-curve-keys-unreachable-module-input-di.md` (another agent's file, also untracked). Both are complete on disk; only the git record is missing. The reporter deliberately did not delete another host's lock file.
