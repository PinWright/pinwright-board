---
id: B-board-commit-stale-lock-silent-skip
title: "board-commit.ps1 cannot land a commit against a stale .git/index.lock (2 s retry budget) and reports success by exiting 0 when the mutex times out, so tickets silently never reach the board"
status: IN-REVIEW
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

## Fix

`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\scripts\board-commit.ps1`, same rewrite as
`E-board-commit-aborts-on-git-stderr` — which had to land in the same change, because until git's
stderr stopped being fatal none of the lock code was reachable at all. Not committed (plugin repo
left dirty by request). All four requested fixes (a)-(d) are in, but (a) is implemented on a
different premise than the ticket assumes.

**(a) Stale-lock recovery — and a correction that matters.** The obvious guard is to let the
filesystem arbitrate: try to delete the lock and treat a refusal as "a live process holds it".
**That does not work.** Measured 2026-09-03 with a real stalled git (a `pre-commit` hook that
sleeps, so git sits inside the locked section): `Remove-Item` on `.git/index.lock` **succeeded**
while git held the handle. git-for-Windows opens its lock files with `FILE_SHARE_DELETE`, so
Windows never refuses. Any design that keys on delete-refusal is a false guard — worth knowing
before someone writes one.

So age is the whole safety argument, and it is split in two, matching this ticket's own two
encounters:

- **`-StaleLockSec` (default 120)** — the lock's mtime has been frozen this long *and* no `git.exe`
  on the machine names this repo in its command line. Encounter `#1` (0-byte lock, no git
  processes): removed.
- **`-BusyLockSec` (default 600)** — a git process *is* on the repo. Wait, reporting the pid and the
  age each time the state changes, until the lock has been frozen this long; then remove it anyway.
  Encounter `#2` (131072-byte lock, mtime frozen 15 min, live pid 102448): recovered at 10 min.
  This is exactly the "emit the holding pid and the lock's age so a caller can tell *wait* from
  *recover*" that `#2` asked for.

Command-line matching is a stated heuristic, not a guarantee: a git started with the board as its
**working directory** (rather than `git -C <board>`) names nothing and is invisible. That is why
age, not the process list, authorizes removal. Blast radius of a wrong removal is bounded — git's
closing rename of `index.lock` onto `index` fails and that command reports an error; the index is
not rewritten and not corrupted.

`#3`'s zero-byte `next-index-*.lock` crumbs are swept too, but only on the recovery path, so a
normal run never writes to another host's `.git`.

**(b) Real retry budget.** `-LockWaitSec` (default 60) with exponential backoff capped at 2 s plus
jitter, replacing the ~2 s of 5 fixed retries. `-Retries` is kept and now means *minimum attempts*;
giving up requires both to be exhausted. Nothing passes `-Retries` today (grepped the board, the
harness dir and `.polyskill/skills/`), so no caller breaks.

**(c) No `exit 0` on a path that did not commit.** The mutex timeout now prints
`could not acquire the commit lock within <N>s; NOT committed: <files>` and exits 1. Retries
exhausted prints `gave up after <N> attempts / <N>s ...` plus a `NOT committed:` line naming the
ticket, and exits 1. `board path not found` and `no files given` also became exit 1 — they were
silent `exit 0`s and they are caller bugs, not degrades. **One `exit 0` without a commit remains
deliberately:** a board that is not a git repo (the documented custom-`boardPath` degrade). There
is nothing to commit there ever, so it is not a false success; the header documents it.

**(d) Verification is the verdict, not git's exit code.** Before returning, the script confirms each
named file is in `HEAD` (`ls-tree --name-only HEAD -- <file>`), and on the "nothing to commit" path
also that it is clean, and prints the short hash: `board-commit: committed <hash> - <files>`. This
runs in both directions:

- a green `git commit` whose file is somehow not in HEAD → exit 1;
- a **failing** git call whose file *is* in HEAD → exit 0 with
  `git reported an error, but the files ARE on the board: <error>`. This is not theoretical — it
  fired during testing when the lock recovery raced a stalled git: the commit object landed, then
  git failed with `fatal: repository has been updated, but unable to write new index file` (exit
  128). Reporting that as a failure would make an agent re-file a ticket that is on the board.

The dirty check is skipped after our own commit on purpose: a sibling host may edit the same file
the moment the index is released, and that does not unmake the commit.

**Reviewer verification.** No PowerShell test convention exists in the plugin repo, so this is a
manual harness rather than a committed test:
`C:\Users\Alexander\AppData\Local\Temp\claude\X--src-unreal-unreal-fpv-dev\b6c0e260-3e88-4849-a20a-d388c2d0ca65\scratchpad\run-tests.ps1`
(throwaway repo per run; the real board is never touched). **17 checks, green under both
`powershell` (5.1.26100.9168) and `pwsh` (7.6.5).** Covering this ticket:

- **T3** 0-byte lock aged 10 min, no holder → removed, ticket committed, recovery line printed (`#1`).
- **T3b** zero-byte `next-index-*.lock` crumb swept during that recovery (`#3`).
- **T4** real stalled git holding the lock, age below `-BusyLockSec` → waits, then exits **1**,
  naming the pid and the ticket (`#2`, "wait" half).
- **T4b** same stalled git, age past `-BusyLockSec` → lock removed, ticket committed (`#2`,
  "recover" half).
- **T5** retries exhausted → exit 1 with a summary, never silence.
- **T6** a lock that clears mid-wait → commit still lands.
- **T8** mutex already held → exit **1** naming the ticket (was `exit 0` + "skipping commit").
- **T9/T10** non-git board still exits 0; a missing board path now exits 1.

Before/after on the pre-fix script, same scenarios: mutex held → `exit=0`, `inHEAD=''`; stale
0-byte lock → `exit=1` via `NativeCommandError` at line 72, `inHEAD=''`, retry loop never entered.

The board's own `.git` still carries three of `#3`'s crumbs
(`next-index-{103824,66204,94360}.lock`, all 0 bytes, 2026-09-02 22:11-22:28); they were left in
place — the next stale-lock recovery on that repo will sweep them.

## History
- `#1-filed` `OPEN` reporter — Hit while filing two `audio.synth` tickets after synthesizing 20 impact/explosion waves. Every `board-commit.ps1` call over ~6 minutes died on `fatal: Unable to create '.../.git/index.lock': File exists` against a 0-byte lock older than five minutes with zero `git.exe` processes and the script's global mutex free. A 20-attempt wrapper loop (8 s apart, output suppressed) never committed and exited 0; the tickets stayed `??` in `git status` the whole time. Landed them manually as c848e36 (2 files changed, 183 insertions) after removing the stale lock and using a pathspec commit, then verified with `git ls-tree --name-only HEAD` and `git merge-base --is-ancestor c848e36 HEAD`. The severity driver is not the lock but the reporting: the mutex-timeout path does `exit 0` after printing "skipping commit", and the retries-exhausted path falls out of the loop with no `exit 1` and no summary, so an agent that checks the exit code — or that suppresses output while retrying — is told a ticket was filed when it was not. Evidence that this already bit someone else: `B-synth-layer-peak-excludes-layer-gain.md`, filed by a different agent about an hour earlier, appeared in my commit under `create mode 100644`, i.e. it had never been committed at all.
- `#2-second-encounter-nonzero-lock` `OPEN` reporter — Hit again ~40 min later on the same board while appending a second encounter to `B-asset-save-pie-failure-reports-pendingflush` (smoke-grenade emitter stream, EAContentExamples58). 24 `board-commit.ps1` invocations across ~9 minutes all died on `fatal: Unable to create 'X:/src/unreal/.pinwright-board/.git/index.lock': File exists.`, surfaced as a PowerShell `NativeCommandError`. Two details differ from #1 and narrow the diagnosis: the lock was **131072 bytes, not 0**, and its mtime stayed frozen at 22:51 local for the whole 15-minute window, while `tasklist` showed exactly **one live `git.exe` (pid 102448) holding ~2.6 MB** — i.e. a real git process that had stalled mid-index-write rather than a lock orphaned by a dead process. So a stale-lock recovery that keys only on "no git process is running" or "lock is 0 bytes" would not have fired here; it needs an mtime-age threshold too. Net effect matched #1: the ticket edit sits in the board working tree uncommitted and the agent has no sanctioned way to land it (`board-commit.ps1` is the only permitted path, and it has no age-based recovery and no way to report "held by a live but stalled git"). Suggest the fix also emit the holding pid and the lock's age so a caller can tell "wait" from "recover".
- `#3-add-exit-code-ignored-so-the-stale-lock-surfaces-as-a-pathspec-error` `OPEN` reporter — Third encounter, 2026-09-02 ~20:10Z, from the FPS impact-VFX authoring wave (EAContentExamples58). Adds a **second, different failure shape** for the same root cause, and a one-line fix that is not in `#1`/`#2`. A stale `X:\src\unreal\.pinwright-board\.git\index.lock` (mtime 2026-09-02 22:51 local, still present ~17 minutes later, alongside three zero-byte `next-index-*.lock` files from 22:11 / 22:22 / 22:28) made every `git add` inside the script fail. The script pipes `git add`'s output to `Out-Null` and **never reads its exit code** (`& git -C $boardFull add -- $fileArgs 2>&1 | Out-Null`, line ~74), so the failure is invisible and execution continues to the pathspec commit. The commit then reports a *misleading* error naming the files rather than the lock: `board-commit: git commit failed: error: pathspec 'B-asset-exists-duplicate-false-negative-in-pie.md' did not match any file(s) known to git`. That message sends the caller to check filenames and paths — both of which were correct; `git status --porcelain --untracked-files=all` listed both as `??` and `git check-ignore` exited 1 for both, so they were plain untracked files that `add` simply never processed. Four retries at 700 ms produced the identical error, i.e. the retry budget cannot outlast an abandoned lock, which is `#1`'s point reached by a different route. Note the shape difference that matters for a fixer: previously-staged files commit fine while newly-created ones fail, so a partial run can land some tickets and silently drop others in the same invocation. Fixes: (a) check `$LASTEXITCODE` after `git add` and fail loudly; (b) detect an `index.lock` older than a threshold (say 60 s) and report it by name in the error rather than letting it surface as a pathspec complaint; (c) sweep the zero-byte `next-index-*.lock` files, which appear to accumulate one per dead host. Two ticket files created in this session remain uncommitted on disk because of this: `B-asset-exists-duplicate-false-negative-in-pie.md` (new) and an appended History entry on `B-niagara-set-curve-keys-unreachable-module-input-di.md` (another agent's file, also untracked). Both are complete on disk; only the git record is missing. The reporter deliberately did not delete another host's lock file.
