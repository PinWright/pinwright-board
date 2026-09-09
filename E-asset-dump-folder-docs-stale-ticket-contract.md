---
id: E-asset-dump-folder-docs-stale-ticket-contract
title: "asset.dump_folder docs still promise unconditional ticket return; SSE block-and-stream default and wait:false are undocumented"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, wiki, async, jobs, asset, dump-folder, sse, streaming, wait]
encounters: 3
lastSeen: 2026-09-09T07:55:00Z
---

# asset.dump_folder docs still promise unconditional ticket return; SSE block-and-stream default and wait:false are undocumented

`asset.dump_folder`'s handler summary and wiki page unconditionally promise a
fire-and-forget kickoff: *"Returns a ticket; poll system.job_status with the
ticket_id to track progress and completion"*
(`Handlers/Asset/AssetDumpHandler.cpp:2067-2070`). The actual behavior for
streaming-capable MCP clients is block-and-stream, **and that behavior is
correct by design** (maintainer ruling, see history `#2`): the docs are what's
stale.

Observed in a live session: `call {method:"asset.dump_folder",
args:{folderPath:"/App"}}` (8168 assets) held the call open ~12 min and
delivered the **final** payload
`{"rootDir":...,"assetCount":8168,"queued":8165,"dumped":8161,"unchanged":3,"skipCount":4}`
as the terminal frame — no `{status:"running", ticket_id, monitor_path}` ever
arrived. The agent, primed by the wiki's ticket contract, misread this as a
hang/bug and initially filed it as one.

Mechanism (for the doc author): `FHandlerContext::StartJob`
(`Handlers/HandlerContext.cpp:577-617`) branches on
`Sub->IsStreamingRequest(RequestId)` (line 597); on the streaming path it calls
`RegisterStreamingJob` and skips the immediate ticket `SendSuccess` (lines
604-616), resolving the request on the job's terminal event. Streaming is
selected by the transport triple gate — `params._meta.progressToken` +
`Accept: text/event-stream` + `wait != false` — and `wait` defaults true
(`Transport/McpRequestCore.cpp:442-446`). The real `mcp__pinwright__call`
client always satisfies the first two, so every long `StartJob` op
block-and-streams unless the caller passes `wait:false`. The job registry
meanwhile works exactly as documented (`started`/1 Hz `progress`/`completed`
in `Saved/PinWright/jobs.jsonl`).

**Workaround:** pass `args:{wait:false}` to get the immediate
`{status:"running", ticket_id}` return — currently documented nowhere on the
method's page.

**Fix:** update the `asset.dump_folder` summary string
(`AssetDumpHandler.cpp:2067-2070`) and the wiki source page to describe the
conditional contract: streaming clients block-and-stream to completion by
default; `wait:false` forces the immediate ticket return for poll-style flows.
Sweep the other `StartJob`-based long-running methods' pages for the same
unconditional ticket-return phrasing while there.

**Related:** `B-asset-dump-folder-no-completion-signal` (DONE — job-registry
integration, verified intact), `F-long-running-tickets` /
`F-handler-context-startjob` (DONE — established the original ticket-return
contract that block-and-stream now conditionally supersedes),
`B-proxy-blocks-minutes-on-hung-editor` (the stdio proxy's 150 s
`--call-timeout` would sever a multi-minute streamed call if that proxy is in
the path).

## History
- `#1-streaming-kickoff-blocks` `OPEN` reporter — Live session: `asset.dump_folder /App` (8168 assets) did not return a ticket within 120 s; the streaming MCP client held the call open ~12 min and got the final `{rootDir,assetCount,queued,dumped,unchanged,skipCount}` payload instead of `{status:"running",ticket_id}`. Traced to `StartJob` suppressing the immediate ticket `SendSuccess` on the streaming path (`Handlers/HandlerContext.cpp:597-601`) — the block-and-stream default (triple gate, `Transport/McpRequestCore.cpp:442-446`) overriding the method's documented ticket-return contract (`AssetDumpHandler.cpp:2067-2070`). Not a regression of the job registry (`B-asset-dump-folder-no-completion-signal`), which recorded started/progress/completed correctly in jobs.jsonl.
- `#2-reframed-docs-stale` `OPEN` maintainer — Ruling: "sse block is correct there, docs are stale." Reframed from bug `B-asset-dump-folder-kickoff-blocks-until-completion` (deleted before ever being committed) to this doc-drift entry: keep block-and-stream as the streaming default; fix the summary/wiki to document the conditional contract and the `wait:false` escape.
- `#3-additional-app-sweep` `OPEN` reporter — Additional evidence: this session's streaming `asset.dump_folder` calls again blocked by default unless `args.wait:false` was supplied. Current source still implements that contract (`McpRequestCore.cpp:592-597`, `HandlerContext.cpp:588-607`, `mcp_proxy.py:29-36`), while the handler summary says "Returns a ticket" (`AssetDumpHandler.cpp:2280-2283`), `asset.dump-quickstart.md:36` says it "always returns a job ticket", `asset-audit.md:21` directs unconditional polling, and `asset.md:130-132` promises an immediate ticket.
- `#4-document-streaming-wait` `IN-REVIEW` developer — Updated the handler summary plus `asset.md`, `asset.dump-quickstart.md`, and `asset-audit.md` to document the actual conditional contract: streaming MCP blocks and streams progress by default; `wait:false` returns the ticket immediately for explicit polling; non-streaming clients receive the ticket normally.
- `#5-25-minute-block-on-a-game-sweep` `IN-REVIEW` reporter — Third encounter, on the largest tree the verb has; `encounters` 2 -> 3, status unchanged (the doc fix in `#4` is still awaiting a tester). UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`: a streaming `asset.dump_folder {folderPath:"/Game", force:true}` (22,446 assets) held the call open for **~25 minutes** before the editor died under it, keeping the calling agent session parked on one blocking RPC for the whole time. Nothing new about the mechanism — this is `#1`/`#3`'s block-and-stream default, working as the maintainer ruled in `#2`. What the scale adds is that the wiki's guidance should not stop at *documenting* `wait:false`: on a sweep of tens of thousands of assets, blocking is the wrong default for an agent caller and the docs should say so outright — recommend `wait:false` plus `system.job_status` polling above some asset count, and state that a blocked call gives up no diagnostic advantage since the job's progress is on the ticket either way. Concretely relevant here because the sweep in question OOM-kills the editor (`B-dump-folder-sweep-never-gcs-ooms-editor`): an agent holding a blocking call learns that only when the connection drops, whereas a ticket poller sees the progress stall. No source read for this entry; `#4`'s doc changes were not re-inspected, so this is evidence for widening its wording, not a claim that it is wrong.
