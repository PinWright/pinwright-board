---
id: B-board-commit-stale-lock-silent-skip
title: "board-commit.ps1 cannot land a commit against a stale .git/index.lock (2 s retry budget) and reports success by exiting 0 when the mutex times out, so tickets silently never reach the board"
status: OPEN
severity: High
category: bug
tags: [board, tooling, board-commit, git, index-lock, silent-failure, concurrency]
encounters: 1
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
