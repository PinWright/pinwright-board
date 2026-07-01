---
id: B-join-lan-server-malformed-url
title: "session.join_lan_server glues serverPort to travelOptions with no '?' separator, producing a corrupt connectionURL"
status: IN-REVIEW
severity: Medium
category: bug
tags: [session, lan, travel-url, malformed-output, string-concat]
---

# `session.join_lan_server` synthesizes a malformed `connectionURL` when `travelOptions` lacks a leading `?`

`session.join_lan_server` builds the connection URL it returns by string-concatenating
`host:port` + `travelOptions` + `?Password=...` **without inserting the `?` separator**
that a UE travel URL requires between the `host:port` and the first option. When the
caller's `travelOptions` does not already begin with `?` — which the wiki never says it
must (it documents the param only as "Additional travel options") — the port digits are
glued directly to the first option key, producing a corrupt, unparseable URL.

The handler synthesizes this URL itself and returns it as the `connectionURL` result
field, so the bug is in the tool's own output, not in anything the caller did wrong.
`travelOptions:"bClientReady=1?Region=us-west"` is a valid documented input.

Source (`SessionsHandler.cpp:411`-`:414`):

```cpp
FString ConnectionString = FString::Printf(TEXT("%s:%d"), *ServerAddress, ServerPort);
if (!ServerPassword.IsEmpty())
    TravelOptions += FString::Printf(TEXT("?Password=%s"), *ServerPassword);
FString FullURL = ConnectionString + TravelOptions;   // <- no '?' between port and options
```

`ConnectionString` ends in the port (e.g. `192.168.1.42:7779`); `TravelOptions` is appended
raw. If `TravelOptions` doesn't start with `?`, the concatenation yields `...:7779bClientReady=1...`
— the `7779` and `bClientReady` are fused into one token that no URL parser can split back
into a port and an option. The trailing `?Password=` is only well-formed because that branch
hard-codes its own leading `?`; the user-supplied `travelOptions` gets no such treatment.

**Repro** (live, replay-confirmed via `mcp__editor-automation__call`):
1. `session.join_lan_server {serverAddress:"192.168.1.42", serverPort:7779, serverPassword:"lanparty2026", travelOptions:"bClientReady=1?Region=us-west"}` →
   `{"serverAddress":"192.168.1.42:7779","connectionURL":"192.168.1.42:7779bClientReady=1?Region=us-west?Password=lanparty2026","status":"configured"}`
   — note the corrupt `:7779bClientReady` (port glued to the first option, no `?`).
2. Control: pass `travelOptions` that already starts with `?`:
   `session.join_lan_server {serverAddress:"10.0.0.5", serverPort:7777, travelOptions:"?Region=eu"}` →
   `{"serverAddress":"10.0.0.5:7777","connectionURL":"10.0.0.5:7777?Region=eu","status":"configured"}`
   — well-formed only because the caller happened to prefix `?`. The handler does not
   normalize the separator, so correctness depends on the caller guessing an undocumented
   leading-`?` convention.

**Impact:** the one durable artifact this echo-only handler produces — the `connectionURL`
a caller would copy into `OPEN`/`ClientTravel` to actually connect — is silently corrupt
for the natural option format (`key=value?key=value`). The call returns success, so the
corruption is invisible until the (broken) URL is used downstream.

**Workaround:** always pass `travelOptions` with a leading `?` (e.g. `"?bClientReady=1?Region=us-west"`),
or ignore the returned `connectionURL` and assemble the travel URL yourself.

**Fix:** insert the separator the first option needs. Either normalize: if
`TravelOptions` is non-empty and does not start with `?`, prepend `?` before concatenating;
or build the URL from `host:port` + a `?`-prefixed option list so the password branch and the
user-options branch share the same separator logic. A leading-`?` is the UE travel-URL contract,
so normalizing on the way in also makes the documented `travelOptions` param forgiving of either form.

## History
- `#1-initial-repro` `OPEN` reporter — Surfaced by a LAN co-op client-join task
  (SEED mode, focus `session.join_lan_server`). The agent ran the full flow
  non-error, but `join_lan_server` returned a `connectionURL` of
  `192.168.1.42:7779bClientReady=1?Region=us-west?Password=lanparty2026` — the
  port fused to the first travel option with no `?`. Replay-confirmed live via
  `mcp__editor-automation__call`: the malformed glue reproduces exactly with
  `travelOptions:"bClientReady=1?Region=us-west"`, and a control with
  `travelOptions:"?Region=eu"` (leading `?`) yields a well-formed URL, isolating
  the missing separator. Grounded in source: `SessionsHandler.cpp:411`-`:414`
  concatenates `host:port` + raw `TravelOptions` with no `?` between them (only
  the `?Password=` branch hard-codes its own leading `?`). The wiki
  (`session.join_lan_server.md`) documents `travelOptions` only as "Additional
  travel options" — no leading-`?` requirement stated — so the corrupting input
  is valid per the docs. Distinct from `E-session-info-readback-hardcoded` (OPEN),
  which is about `get_sessions_info` hardcoding `isLANMatch`/`connectedServerAddress`/etc.
  on the *readback* path; this is a malformed *output value the join handler itself
  synthesizes*. No existing board ticket on the join-URL concatenation (ripgrep
  across OPEN + closed; qmd unavailable).
- `#2-fix` `IN-REVIEW` developer — Fixed the root cause in
  `session.join_lan_server` (`SessionsHandler.cpp:411`-`:420`): before
  concatenating, the handler now normalizes user `travelOptions` — if it is
  non-empty and does not already start with `?`, a leading `?` is prepended, so
  the first option is `?`-separated from `host:port` (matching the already-correct
  `?Password=` branch and the UE travel-URL contract; either input form now works).
  `travelOptions:"bClientReady=1?Region=us-west"` + password now yields
  `192.168.1.42:7779?bClientReady=1?Region=us-west?Password=lanparty2026` instead
  of the corrupt `:7779bClientReady` glue. Added regression test
  `EditorAutomationRpcGateway.session.join_lan_server.NormalizesTravelOptionSeparator`
  in `Tests/EditorOps/TestSystemHandlers.cpp` (alongside the existing
  join_lan_server tests): it drives the production handler via
  `InvokeHandlerWithCapture`, asserts the returned `connectionURL` no longer
  contains `:7779bClientReady`, that the first option is reachable as
  `?bClientReady=1`, and that `host:port` is preserved — it fails if the fix is
  reverted. Files: `Private/Handlers/System/SessionsHandler.cpp`,
  `Private/Tests/EditorOps/TestSystemHandlers.cpp`. Not compiled/tested here (later phase).
