---
id: H-journal-missing-interruption-terminal-event
title: "A user-interrupted agent leaves only a dangling 'started' in journal.jsonl — no terminal event distinguishes user-stop from crash or session limit"
status: OPEN
severity: Low
workflow: both
category: harness-process
knob: middleware
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets: []
---

# A user-interrupted agent leaves only a dangling 'started' in journal.jsonl — no terminal event distinguishes user-stop from crash or session limit

The test-loop agent was interrupted mid-tool-call; the interruption IS recorded in the agent transcript ('[Request interrupted by user for tool use]'), proving the harness was alive and could have journaled it, yet journal.jsonl ends with type:'started' for that agent and nothing else. Supervision ticks, audit partitioning, and outcome stats cannot tell user-stop from API failure, crash, or session limit — a distinction the standing memory notes say matters (session-limit stops masquerade as other failures). Applies to every interrupted/stopped run; this is a polyskill-runtime journaling gap, not a skill-prompt issue, so no .polyskill file edit target.

## Evidence
- run: wf_510c6c2c-eb5
- agent: journal.jsonl
- quote: "final line: {\"type\":\"started\",...,\"agentId\":\"afb2e32b9a00b6d9f\"} with no matching result"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
Have the polyskill workflow runtime append a terminal journal event (e.g. {type:'aborted', agentId, reason:'user-interrupt'|'transient'|'session-limit'}) whenever an agent ends without a result while the harness is still alive, so audits and supervision can classify dangling 'started' entries without re-reading full transcripts. Repro: interrupt any running fix-workflow agent mid-tool-call (Esc during a build), then `jq -c 'select(.agentId=="<id>")' journal.jsonl` — only a 'started' event exists.

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
