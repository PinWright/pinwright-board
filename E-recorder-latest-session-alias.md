---
id: E-recorder-latest-session-alias
title: "recorder verbs reject session:\"latest\" — most-recent intent forces an enumerate-first round-trip"
status: OPEN
severity: Low
category: ergonomic
tags: [recorder, ergonomic, session-resolution, docs]
encounters: 1
lastSeen: 2026-06-17T08:48:41Z
---

# recorder verbs reject session:"latest" — most-recent intent forces an enumerate-first round-trip

The single most common recorder intent is "analyse my **last / latest** run"
(post-mortem on the recording that just finished). But the recorder verbs only
accept a concrete session **id** — there is no `latest`/`most-recent`/`newest`
sentinel. An agent that takes the user's wording literally and passes
`session: "latest"` gets `[SESSION_NOT_FOUND] Recording session not found:
'latest'.` from every window-scoped verb, and must back out to
`recorder.list_sessions` to read the newest id (the list is already sorted
newest-first by `RecorderResolver::ListSessions()`) before re-issuing every call.

This is a misuse-then-correct / excessive-steps pattern: the literal-but-wrong
guess costs one failed call per verb before the agent learns it must enumerate
first. The sentinel is exactly the kind of thing other surfaces already accept
(e.g. `actor`/level verbs take symbolic targets), so reaching for it is natural.

Two things would remove the friction:

1. **Ergonomic:** accept a reserved `session: "latest"` (and/or omit `session`
   entirely) on the recorder verbs, resolving to the newest session via the
   existing newest-first `RecorderResolver::ListSessions()` — no client
   round-trip. A reserved literal is unambiguous because real session ids are
   timestamp/GUID-derived base filenames, never the word `latest`.
2. **Docs:** there is **no `docs/wiki-src/recorder.md` overlay** today (the
   recorder wiki is auto-generated; noted on `E-recorder-list-sessions-limit`
   #2). The generated `recorder` index already calls `describe_session` the
   "MANDATORY FIRST CALL" but never says the `session` arg must be a concrete id
   from `list_sessions` and that there is **no** `latest` sentinel. A new
   `docs/wiki-src/recorder.md` overlay should state the resolution contract
   ("`session` is a concrete id from `recorder.list_sessions`, which is sorted
   newest-first; there is no `latest`/`newest` alias — read the first id").

**Workaround:** call `recorder.list_sessions` first, take the first (newest) id,
and pass that concrete id to every subsequent verb.
**Fix:** add a reserved `latest`/omitted-`session` resolution to the newest
session (preferred), and/or document the resolution contract on a new
`docs/wiki-src/recorder.md` overlay.

## History
- `#1-initial-audit` `OPEN` reporter — Process audit of a recorder post-mortem task ("pull up my latest recorder session"). The agent passed `session: "latest"` and hit `[SESSION_NOT_FOUND] Recording session not found: 'latest'.` on SIX consecutive window-scoped verbs (`describe_session`, `list_segments`, `summarize_change`, `get_series`, `find_events`, `get_state`) before recovering via `list_sessions`. Friction note recorded "clean, no retries", but the call-log shows the literal-but-wrong `latest` guess fanning out across every verb — a misuse-then-correct caused by the absence of a most-recent sentinel. (Distinct from `E-recorder-list-sessions-limit`, which is about the list response size, and from the WONTFIX `E-call-noargs-runs-wiki-zero-arg-methods` zero-arg/wiki round-trips. No board ticket covers a `latest` session alias; qmd + ripgrep dedup confirms none.)
