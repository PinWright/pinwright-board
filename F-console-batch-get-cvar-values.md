---
id: F-console-batch-get-cvar-values
title: "No exact / batch read of cvar values — verifying N changed cvars costs N substring searches"
status: OPEN
severity: Low
category: feature
tags: [console, cvar, search, verify, batch, read, system]
encounters: 4
lastSeen: 2026-06-25T07:14:35Z
---

# No exact / batch read of cvar values — verifying N changed cvars costs N substring searches

`system.console.search` is the only way to read a cvar's `currentValue`, and it
is a *substring discovery* surface — it takes one `query` string and returns a
ranked list of matches. There is no exact-name read and no multi-name read. So
when an agent already knows the exact names of several cvars and just wants their
current values (the canonical "verify my before/after record" step after a batch
of `system.console_command` sets), it has to fire **one `system.console.search`
per cvar** and pick the matching row out of each result list.

This is the deferred half of `F-search-api-console-commands` (DONE). That
ticket's `#1` proposed both `system.console.search` *and* `system.console.get`
(exact lookup), plus an optional `system.console.read` value-only variant; the
`#2` review deferred `get` / `read` / `list_prefixes` as YAGNI and shipped only
`search`. This ticket re-files the deferred companion against concrete friction
evidence — the verify-loop cost is now observed, not hypothetical.

## What's awkward

Two related shortfalls, both visible in the same task:

1. **Search-as-get for a known name.** To read one cvar whose exact name you
   already hold (`r.Streaming.PoolSize`), you call `system.console.search
   {query:"r.Streaming.PoolSize", kind:"variable"}` and scan the result list —
   using a fuzzy substring engine + ranking + `totalMatches` + `truncated`
   machinery to fetch a single known row. The "search first" framing on
   `docs/wiki-src/system.md:121,135-139` only covers the *discovery* case ("you
   know the rough name but not the exact"); it has no answer for the
   exact-name-readback case, so the agent reuses `search` as a `get`.

2. **No batch read.** Verifying a tuning pass that touched K cvars is K separate
   `search` calls. There is no `system.console.read(names: [...])` /
   `system.console.get(name)` to read one or many known cvars in a single round
   trip. `system.console_command "r.X 2048"` is also pure execute — it does not
   echo the resulting value — so the only way to confirm a set "took" is a
   follow-up read, multiplied per variable.

## What it should do

Add an exact/batch read so a known-name verify costs one call, not N:

```
system.console.get(name: string)
  -> { found: bool, name, kind, currentValue, defaultValue?, help?, flags? }

system.console.read(names: [string, ...])      // batch value-only
  -> { values: { "<name>": { found, currentValue, kind } , ... } }
```

`IConsoleManager::Get().FindConsoleObject(name)` is the exact lookup primitive
(already cited in `F-search-api-console-commands` #1), so this is a thin wrapper
over the same registry `search` already walks. A lighter alternative that also
removes the verify round-trips entirely: have `system.console_command` echo the
resulting `currentValue` for recognized single-cvar `r.X <value>` sets, so the
set call *is* its own confirmation. Any one of these collapses the per-variable
verify loop.

If implemented as docs-only first: `docs/wiki-src/system.md` "Console command
discovery" should at least state that exact-name readback is done by passing the
full name as `query` and reading the first result, and that there is no batch
read — so the N-call cost is a documented expectation rather than a surprise. But
the real fix is the typed read above.

## Evidence

From a `system.console` scalability-tuning fuzz task (story: "verified
before/after record of my scalability tuning"), outcome `clean`, friction
`none` — i.e. nothing *failed*, this is pure step-count friction:

- 14 calls total. The closing **verify phase was 4 separate
  `system.console.search` calls**, one per changed cvar
  (`r.Streaming.PoolSize`, `r.Shadow.MaxResolution`, `r.AntiAliasingMethod`,
  `r.ViewDistanceScale`), purely to read back each `currentValue` — every name
  was already known to the agent (it had just searched and set them).
- The 3 pre-set discovery searches (steps 2,4,6) and the AA-method check
  (step 5) are likewise exact-name reads expressed through the substring
  `search` surface.
- `system.console_command "r.Streaming.PoolSize 2048"` (and the other two sets)
  returned `ok` with no echo of the resulting value, which is *why* a separate
  read phase was needed at all.

Self-report: *"final verification searches confirmed all four currentValues
match the targets."* Four searches to confirm four known names is the friction:
a `system.console.read([...4 names...])` (or a value-echoing
`system.console_command`) would make it one.

This is distinct from `E-console-search-stat-subcommand-blind` (OPEN — that is a
*false-negative* on multi-token `stat <group>` queries; this is *step count* on
exact-name single-cvar reads that all succeed) and from
`F-search-api-console-commands` (DONE — shipped `search` only; explicitly
deferred the `get` / `read` exact/batch companion this ticket evidences).

## History
- `#1-verify-loop-n-searches` `OPEN` reporter — Clean `system.console` tuning task spent its entire verify phase on 4 separate `system.console.search` calls, one per already-known cvar name, just to read back `currentValue` for a before/after record; the 3 pre-set discovery reads are also exact-name lookups expressed through the substring search surface, and `system.console_command` set calls returned no value echo (forcing the read phase). Root cause: `system.console.search` is the only value-read path and is substring-discovery-shaped with a single `query` — no `system.console.get` (exact) and no batch `read([names])`. `F-search-api-console-commands` #1 proposed exactly these companions; its #2 review deferred them as YAGNI and shipped `search` only. Propose `system.console.get(name)` + `system.console.read(names[])` over `IConsoleManager::FindConsoleObject`, or (cheaper) have `system.console_command` echo the resulting `currentValue` for single-cvar sets so the set is self-confirming. Docs fallback: note on `docs/wiki-src/system.md` that exact-name readback reuses `query` and that there is no batch read, so the N-call verify cost is documented rather than surprising.
- `#2-recurrence-rendering-tuning` `OPEN` reporter — Second independent instance from a rendering/perf-tuning struggle audit (story: confirm world + read back 3 tuning cvars; outcome `ergo`, the seed verbs all round-tripped — pure process friction). 16 calls total; after the 3 `system.console_command` sets (`r.ScreenPercentage 75`, `sg.ShadowQuality 2`, `r.Streaming.PoolSize 2000`) the verify phase was again **one `system.console.search` per already-known cvar** — 3 separate read-back searches (`query:"r.ScreenPercentage" limit:1`, `query:"sg.ShadowQuality" limit:2`, `query:"r.Streaming.PoolSize" limit:1`) to confirm each `currentValue`, because the set calls returned no value echo. Confirms the exact step-count this ticket describes on a different cvar set and a different task; a value-echoing `system.console_command` or a `system.console.read([3 names])` would have made the whole verify phase a no-op / one call. (The same task's broad-discovery overflow — `system.console.search "ScreenPercentage"` spilling to file at limit 25/50 — is the separate response-size gap owned by `E-console-search-default-limit-spills`, which the per-finding judge filed; this evidence is only the exact-name read-back step-count half.) No new ticket — same gap, same RPC, same fix family.
- `#3-recurrence-perf-typed-setters` `OPEN` reporter — Third independent instance, and it broadens the "no value-echo" half beyond raw `system.console_command`: a `performance.apply_baseline_settings` struggle audit (story: lay a performance baseline then layer scalability=1 / MaxFPS=60 / screenPercent=80 / VSync / Nanite, then **read the live values back as proof the preset stuck**). Outcome `clean`, friction `none` — pure step-count. 9 calls; the closing verify phase was again **one `system.console.search` per already-known cvar** — 3 separate read-backs (`query:"t.MaxFPS"` →60, `query:"r.ScreenPercentage"` →80, `query:"r.Nanite.ProjectEnabled"` →1) purely to confirm each `currentValue`. The new wrinkle: these cvars were set by the **typed `performance.*` setters** (`set_frame_rate_limit`, `set_resolution_scale`, `configure_nanite`), not by raw `system.console_command`, and *those* setters likewise returned `ok` with no echo of the resulting cvar value — so the value-echo gap is not specific to `console_command`, it spans the whole `performance.*` setter family that drives cvars under the hood. Reinforces the cheaper fix from a wider surface: the typed setters that already know the cvar+value they wrote should echo the resulting `currentValue` in their response (making the set self-confirming and the read phase a no-op), and/or `system.console.read([names])` would collapse the 3 read-backs to one. No new ticket — same gap, same fix family; recorded here because the typed-setter (not just `console_command`) value-echo angle widens the fix's scope.
- `#4-recurrence-playtest-budget` `OPEN` reporter — Fourth independent instance, same typed-`performance.*`-setter value-echo gap, on a low-spec playtest-budget struggle audit (focus `performance.set_frame_rate_limit`; story: apply balanced baseline, cap 60 FPS, scalability Medium, 80% res scale, VSync off, FPS overlay, then **read the cvars back to confirm the budget locked in**). Outcome `tool_bug` (the judge filed the orthogonal `B-set-scalability-no-sg-update`); this entry is the pure step-count process half. 12 calls; the setters all returned a bare `ok:true` with no echo of the cvar they wrote, so every confirmation needed a follow-up read: `set_frame_rate_limit {maxFPS:60}` → then `system.console.search t.MaxFPS` (→60); `set_vsync {enabled:false}` → then `system.console.search r.VSync` (→0); `set_resolution_scale {scale:80}` → then `system.console.search r.ScreenPercentage` (→80); then a second `set_frame_rate_limit {maxFPS:30}` → then re-`system.console.search t.MaxFPS` (→30). That is **4-5 separate read-back searches chasing 4 typed-setter sets**, every cvar name already known to the agent because the setter is what wrote it. Confirms the `#3` finding on a distinct setter set (`set_frame_rate_limit` twice + `set_vsync` + `set_resolution_scale`) and a distinct task; the same fix collapses it — have each `performance.*` setter echo the resulting `currentValue` (self-confirming set, read phase becomes a no-op), and/or `system.console.read([names])` for the multi-cvar confirm. No new ticket — same gap, same RPC family, same fix family.
