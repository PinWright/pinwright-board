---
id: B-supervisor-protocol-skew-unexplained
title: "An MCP proxy started before the supervisor changed launches the new script with the old argv and gets only 'exited without a handoff'"
status: IN-REVIEW
severity: Medium
category: bug
tags: [proxy, supervisor, editor-build, protocol, stale-proxy, gap-analysis-2026-09-28]
encounters: 1
costly: 1
lastSeen: 2026-09-29T00:00:00Z
---

# A stale MCP proxy fails supervised launches without saying why

The MCP proxy imports `pinwright_supervisor` once and keeps it in memory, but launches
`pinwright_supervisor.py` from disk. After `B-supervisor-dies-with-mcp-client` changed the exchange
(protocol 1: `--supervise`, spec on stdin, handoff on stdout; protocol 2: `--supervise <spec.json>`,
handoff file), a proxy loaded before that edit launched the new script with the old argv. The new
`main()` refused with the generic "internal to the PinWright MCP proxy and has no command line"
line and exit 2, and `editor_build` surfaced only
`BUILD_START_FAILED ... supervisor pid 43848 exited without a handoff`
(`Saved/PinWright/builds/8cbaef64c7b34d999963c6694af0dd13/build.log.supervisor.log`). Every proxy
older than the on-disk supervisor, in any session on the machine, breaks the same way.

**Fix:** `PROTOCOL_VERSION` in the spec, checked by `main()`; a skew exits `EXIT_VERSION_MISMATCH`
(3) with a `SUPERVISOR_VERSION_MISMATCH` line naming both versions and the fix (restart the MCP
server, `/mcp` reconnect), and the proxy reports it as that error code.

## History
- `#1-stale-proxy-bare-refusal` `OPEN` reporter — A proxy holding the pre-WMI supervisor in memory launched the new script with the protocol-1 argv; the build failed as "exited without a handoff" with no reason in the tool result and only the generic refusal in the log.
- `#2-protocol-version-handshake` `IN-REVIEW` developer — `pinwright_supervisor.py`: `PROTOCOL_VERSION = 2`, `VERSION_MISMATCH`, `EXIT_VERSION_MISMATCH = 3`, `SupervisorVersionMismatch`. `spawn_supervised` writes `protocolVersion` into the spec. `main()`: the protocol-1 argv (`--supervise` alone) now writes the mismatch text to stderr (the old caller's supervisor.log) and a `{"error", "code"}` JSON line to stdout (the old caller's handoff, so its tool result reads `supervised start failed: SUPERVISOR_VERSION_MISMATCH: ... Restart the MCP server ...`), exit 3; a spec whose `protocolVersion` differs logs the line, writes a mismatch handoff and exits 3. A spec with no `protocolVersion` is accepted as protocol 2 (the spec-file exchange one revision earlier, unchanged), so saving this file did not break proxies loaded from the 19:46 revision, including the one running the current full suite. Caller side: a mismatch handoff raises `SupervisorVersionMismatch`; a supervisor that dies without a handoff now names its log's last line in the error, and raises the typed error when that line carries the marker (the exit code of a WMI-created process is not observable after it exits, so the handoff and the log are the channels). `mcp_proxy.py` maps `SupervisorVersionMismatch` to error `SUPERVISOR_VERSION_MISMATCH` (text starts with the code) in `editor_start`, `editor_run_tests` and `editor_build`. Tests: a real protocol-1 caller (spec on stdin, stdout handoff) against the new script, an incompatible spec version refused by name, a spec without the field still accepted, mismatch handoff and log marker raise the typed error, other early deaths name the log's last line, `editor_build` and `editor_start` return the typed code. Staged and tested in a copy first; applied after the live files matched the pre-change hashes. Python suite 362 OK (1 skipped). Other sessions' proxies started ~17:33 still hold pre-WMI modules: until restarted they now get the clear mismatch error instead of the bare failure.
