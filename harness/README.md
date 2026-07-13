---
type: guide
summary: "Harness sub-board — one file per workflow-skill defect under this folder. Scope, frontmatter schema, status rules, and dedup conventions live here."
tags: [harness, workflow-skills, audit, issue-board]
---

# Harness Sub-Board

Living tracker for defects in the **workflow harness itself** — the
`mcp-test-workflow` / `mcp-fix-workflow` / `workflow-log-audit` skills: their
prompts, scripts, supervision, and pipeline logic. **One file per defect** in
this folder, filename = `{id}.md`.

Scope: **harness defects only** — an agent misreading a prompt instruction, a
supervision tick misfiring, a picker choosing badly, a script edge case, a
resume/restart fragility. NOT bugs in the PinWright plugin/MCP under test —
those go on the main board (one level up) as `B-` / `E-` / `F-` tickets.

Filed by:
- the **`workflow-log-audit` skill's file-to-board mode** (launched by the
  workflows' hourly supervision ticks) — the primary producer;
- **in-run workflow agents** that hit a harness defect mid-run;
- **the user** directly.

Applied via `/workflow-log-audit` (default review mode) — fixes are
user-supervised, never made mid-run.

## ID Format

`H-<kebab-slug>.md`, where the slug is a stable 2–6 word kebab-case
description of the defect (e.g. `H-picker-ignores-defer-gate`). `id` matches
the filename stem exactly. Before creating a new file, check that
`harness/{id}.md` doesn't already exist — IDs must be unique across all
statuses (including `DONE` and `WONTFIX`).

## Frontmatter Schema (mandatory on every entry file)

```yaml
---
id: H-picker-ignores-defer-gate
title: "Fix-workflow picker selects tickets whose deferUntil has not passed"
status: OPEN               # OPEN | DONE | WONTFIX
severity: Medium           # High | Medium | Low
workflow: fix              # test | fix | both — which workflow skill the defect lives in
category: coverage-picker  # attempt-behavior | harness-process | prompt-instruction | editor-lifecycle | coverage-picker | ticket-quality | performance-cost | resume-restart | other
knob: prompt               # prompt | tool | middleware — what kind of change fixes it
encounters: 1              # starts 1, +1 per re-observation
lastSeen: 2026-07-13T10:00:00Z   # ISO time of the most recent observation
filedBy: audit@fuzz2       # audit@<host> | <agent-role>@<host> | user
editTargets:               # repo-relative .polyskill/... source paths the fix would touch
  - .polyskill/skills/mcp-fix-workflow/SKILL.md
---
```

Rules:
- `id` matches the filename stem exactly.
- `status` is the single source of truth — filers and the audit filter on it.
- `workflow` names the skill the defect lives in; defects in the audit skill
  itself use `both`.
- `knob` classifies the fix: `prompt` (skill prompt/instruction text), `tool`
  (a script or helper shipped with the skill), `middleware` (harness/runtime
  behavior outside the skill sources).
- `editTargets` may be empty for `middleware` or not-yet-apply-ready items;
  otherwise list every `.polyskill/...` source path the fix would touch.
- `encounters` / `lastSeen` are maintained by filers on every re-observation
  (see [Dedup rules](#dedup-rules-for-filers)).

## Status Rules

```
OPEN → DONE
OPEN → WONTFIX
DONE → OPEN (re-observed by a later audit)
```

- Unlike the main board, **`OPEN → DONE` direct IS allowed** — apply happens
  user-supervised in `/workflow-log-audit` review mode and is verified by
  polyskill regen + syntax check in the same session, so no separate
  `IN-REVIEW` tester pass exists.
- `WONTFIX` = the user rejected the finding. It is **permanent dedup memory**:
  the audit must **never re-file** a finding that matches a `WONTFIX` ticket.
- A `DONE` ticket re-observed by a later audit is **reopened** (status back to
  `OPEN`, `encounters`++, history note `re-observed after DONE`) rather than
  duplicated.
- Tickets are **never deleted**.
- Every status change updates `status:` in frontmatter AND appends a History
  entry.

## Body Template

```markdown
# {Title}

{Description paragraph — what the harness does wrong, what it should do,
impact on the run.}

## Evidence
- run: {run id}
- transcript: {agent transcript file}
- quote: "{verbatim quote from the transcript}"
- report: {narrative report path}

## Proposed edit
{Concrete edit text against the editTargets paths.}

## History
- `#1-initial-observation` OPEN audit@fuzz2 — description
- `#2-re-observed-run-xyz` OPEN audit@fuzz1 — dedup: encounters 1→2
- `#3-applied-in-review` DONE user — applied via /workflow-log-audit, regen verified
```

History follows the main-board convention: **append-only** bullets prefixed
`` `#N-slug` `` where `N` is a file-local monotonic counter (1-based, never
resets) and `slug` is 2–5 kebab words describing the entry. **No dates** in
history entries — git blame supplies timestamps. Never delete or rewrite
prior entries.

## Dedup rules for filers

Before filing, match the new finding against existing tickets:
1. **First by slug/filename** — does an `H-*` file with the same or a
   near-identical slug exist?
2. **Then by `editTargets` + title similarity** — same source files and a
   similar defect description count as the same ticket.

Then act by the match's status:
- **OPEN match** — do not duplicate: `encounters`++, refresh `lastSeen`,
  append a History entry (action `dedup`).
- **DONE match** — **reopen** it (status → `OPEN`, `encounters`++, history
  note `re-observed after DONE`).
- **WONTFIX match** — **skip silently**; never re-file.

**No severity floor** — `Low` findings are filed too.

## Committing

Every create/update goes through `board-commit.ps1` (shipped in each skill
dir; never plain `git` on the board), commit message:

```
board(<host>): harness H-<id> <new|dedup|reopen|DONE|WONTFIX>
```

MARKUP HYGIENE: never leave tool-call XML (`</content>`, `</invoke>`, ...) in
ticket tails — a known recurring corruption on the shared board.
