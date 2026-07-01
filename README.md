---
type: guide
summary: "MCP issue board — one file per issue under this folder. Workflow, frontmatter schema, and status rules live here."
tags: [bpir, widget-xml, bugs, feature-request, ergonomics, issue-board]
---

# MCP Issue Board

Living tracker for bugs, missing features, and ergonomic improvements in the
EditorAutomation MCP plugin. **One file per issue** in this folder, filename
= `{id}.md`. New issues are created as new files; status changes edit the
existing file's frontmatter and append to its History section.

Scope: **MCP tool issues only** — BPIR compiler bugs, `widget_import_xml`
problems, property resolution failures, missing tool features, ergonomic
gaps. NOT game-level bugs.

## Workflow

```
OPEN → IN-REVIEW → DONE
            ↓
          OPEN (returned with failure reason)
```

Three roles:
- **Reporter agent** — files issues with `OPEN` status.
- **Developer agent** — implements fix, moves to `IN-REVIEW` with
  implementation notes. **Cannot set `DONE` unilaterally** — verification
  by a separate tester is mandatory.
- **Tester agent** — verifies fix, moves to `DONE` or returns to `OPEN` with
  failure details. The user is the default tester; an AI agent can act as
  tester **only when the user explicitly asks it to verify** ("check if
  fixed", "see if it works", "test this", etc.). In that case the agent
  runs the live verification and updates `DONE` / `OPEN` based on the
  observed result.

## Status Rules

| Status | Who sets it | Required comment |
|--------|-------------|------------------|
| `OPEN` | Reporter or Tester | Problem description. If returned: why it failed and repro steps |
| `IN-REVIEW` | Developer | How it was fixed: "Changed X in Y.cpp to do Z" |
| `DONE` | Tester | Test performed and result |
| `WONTFIX` | Developer | Why it won't be fixed |

Rules:
- **OPEN → DONE is NEVER allowed.** IN-REVIEW is mandatory.
- Every status change MUST update `status:` in frontmatter AND append a
  History entry with new status, agent role, and comment.
- Every history entry is prefixed with `` `#N-slug` `` where `N` is a
  monotonic, file-local counter (1-based, never resets, keeps incrementing
  across status cycles) and `slug` is a short kebab-case descriptor
  (2–5 words) of what the entry is about. The `#N` prefix guarantees
  each bullet is textually unique so Edit-tool appends land at the true
  end of the History section; the slug makes cross-references in prose
  readable (`see #3-returned-widget-bp-case`).
- Never delete history entries — append only.
- No dates in history entries — git blame provides timestamps.

## Deferring a ticket

A defer keeps a ticket `OPEN` but records a **machine-checkable gate** so the
fix loop's picker skips it until the gate lifts — instead of re-running a full
analysis on it every iteration. Set one or both frontmatter fields:

- `blockedBy: [<id>, ...]` — the ticket(s) whose resolution unblocks this one.
  Released when **every** listed ticket reaches `DONE` or `WONTFIX`. Use this
  whenever another ticket gates the work; it may list several.
- `deferUntil: <YYYY-MM-DD>` — a deliberate review-by date, for a gate that is
  **not** a ticket (waiting on field evidence, an upstream/engine change, a
  human decision). Released when the date is reached. Never a placeholder or
  arbitrary default — pick a date you can justify.

A deferred ticket becomes eligible again as soon as **either** gate lifts
(blockers all cleared, or the date reached — whichever is first). A defer that
can name neither a real blocking ticket nor a justifiable date is not a defer:
use `WONTFIX` instead of parking it ungated. Drop the fields once the ticket
leaves the deferred state.

## Claiming a ticket

Several fix hosts (`fuzz1`..`fuzz4`) work this board in parallel. To stop two
hosts from picking the same ticket, the picker takes a **lease** the moment it
selects one — the ticket **stays `OPEN`** but gets two frontmatter fields:

- `claimedBy: <hostId>` — the host working it (e.g. `fuzz3`).
- `claimedAt: <ISO-8601>` — when the claim was taken.

A picker **skips** an `OPEN` ticket that carries a *fresh* claim by another host
(`claimedAt` within the last **4h**). A claim **4h or older** is treated as
**stale** (the host likely died) and is reclaimable — the new picker just
overwrites it. A host's own claim never blocks it. The lease is **removed on
every terminal transition** (→`IN-REVIEW` on success, `WONTFIX`, `DEFER`, or a
return to `OPEN` on abandon), so it never lingers.

The lease is best-effort, not a hard lock: if two hosts claim simultaneously the
loser yields on a re-read, and any residual double-work is caught when the second
host finds the fix already published (it resolves to **ALREADY-FIXED**: the
defect is already gone from current source, so no new code — flip `OPEN→IN-REVIEW`
noting it is already resolved).

## ID Format

Kebab-case slug IDs prefixed by category:
- `B-descriptive-slug` — bug
- `F-descriptive-slug` — feature request
- `E-descriptive-slug` — ergonomic improvement

Keep slugs short (2-4 words). Before creating a new file, check that
`board/{id}.md` doesn't already exist — IDs must be unique across all
statuses (including `DONE` and `WONTFIX`).

## Frontmatter Schema (mandatory on every entry file)

```yaml
---
id: B-enum-raw-integers
title: "Enum values stored as raw integers on non-byte pins"
status: DONE               # OPEN | IN-REVIEW | DONE | WONTFIX
severity: Critical         # Critical | High | Medium | Low
category: bug              # bug | feature | ergonomic
tags: []                   # optional; list of short keywords
blockedBy: []              # optional; deferral-only — ticket ids gating this OPEN ticket
deferUntil: 2026-07-21     # optional; deferral-only — YYYY-MM-DD review-by date
claimedBy: fuzz3           # optional; lease-only — host currently working this OPEN ticket
claimedAt: 2026-06-21T14:30:00Z  # optional; lease-only — ISO time the claim was taken
encounters: 3              # optional; times this bug was observed (seeded 1, +1 per dedup-append). Same-severity tiebreak; absent = 1
lastSeen: 2026-06-27T09:15:00Z   # optional; ISO time of the most recent observation
---
```

Rules:
- `id` matches the filename stem exactly.
- `category` is derived from the ID prefix (`B`→bug, `F`→feature,
  `E`→ergonomic).
- `status` is the single source of truth — skills filter on this field.
- `tags` is optional. May be empty.
- `blockedBy` / `deferUntil` are optional and present **only while an OPEN
  ticket is deferred** — see [Deferring a ticket](#deferring-a-ticket). Omit
  them otherwise.
- `claimedBy` / `claimedAt` are optional and present **only while a host is
  actively working an OPEN ticket** — see [Claiming a ticket](#claiming-a-ticket).
  Removed on every terminal transition.
- `encounters` / `lastSeen` are optional and maintained by the test workflow.
  `encounters` is seeded `1` when the ticket is first filed and incremented by
  `1` on every dedup-append (additional-evidence, regression, or revival); an
  absent field is treated as `1`. `lastSeen` is the ISO-8601 timestamp of the
  most recent observation, refreshed on each append. (`lastSeen` is allowed in
  frontmatter — the "no dates in history" rule governs history entries only, and
  `claimedAt` is already an ISO frontmatter timestamp.) `encounters` is a
  same-severity work-ordering tiebreak, **never** a severity input — see
  [Severity Levels](#severity-levels).

### Severity Levels

`severity` is the work-ordering key: the fix picker ranks OPEN tickets by
severity (Critical > High > Medium > Low) and works the top band first, so a
mis-rated ticket is worked in the wrong order. Choose it by **impact class**,
then adjust for **reach**. Never default to `Low`.

Impact class:
- **Critical**: editor crash, or a write that corrupts or loses asset data.
- **High**: silent false-success, or silent wrong / stale / hardcoded data on a
  normal path (the caller trusts a result that is a lie and builds on it).
- **High or Medium**: hard blocker with no workaround (a stub, a missing verb,
  or rejecting valid input), so a reasonable task is impossible.
- **Medium**: soft blocker. Doable, but only via a documented workaround, a
  source dive, or many extra calls; or a readback omits a field and forces a
  fallback.
- **Low**: pure friction. Docs, discoverability, naming, a response spill that
  only forces a `Read`, or cosmetic.

Reach modifier: if the affected method runs in almost every session, bump up one
level; if it is a rare edge path, bump down one. A `Low`-impact gap on an
every-session method outranks a `High`-impact gap on a method nobody hits.

Tiebreak: within one severity band the picker orders OPEN tickets by
`encounters` (descending), then filename. `encounters` only breaks ties between
equally-severe tickets — it measures how often the fuzzer re-hits a bug, not user
impact, so it **never** bumps severity. Severity stays impact × reach; a
high-`encounters` ticket is still worked after every more-severe ticket.

## Body Template

```markdown
# {Title}

{Description paragraphs — what the tool does wrong, what it should do, any
repro steps, impact.}

**Workaround:** {optional}
**Fix:** {optional proposed or implemented approach}

## History
- `#1-initial-repro` `OPEN` reporter — description
- `#2-changed-x-in-y` `IN-REVIEW` developer — "Changed X in Y to do Z"
- `#3-verified-on-asset-bar` `DONE` tester — "Verified: test description and result"
- `#4-returned-still-fails` `OPEN` tester — "Returned: still fails because X. Repro: ..."
```

## Creating a New Entry

1. Choose an ID: `{B|F|E}-{2-4-word-slug}`.
2. Verify `board/{id}.md` does not exist.
3. Write the file with full frontmatter and at least one History entry
   (`#1-{slug}` `OPEN` reporter).

## Changing an Entry's Status

1. Edit `board/{id}.md`.
2. Update the `status:` field in frontmatter.
3. Append one History entry to the `## History` section, prefixed with
   `` `#{N+1}-{slug}` `` where `N` is the current highest number in the
   section. Don't modify any existing entry — just emit a new row in the
   correct format at the end of the section.
4. Never delete or rewrite prior history entries.

## See also
- [bpir-language-reference](../wiki/bpir.md)
- [Blueprint wiki insertion sections](../wiki/blueprint.md#blueprintinsert_bpir_at_node)
- [bpir-test-matrix](../bpir-test-matrix.md)
- [mcp-usage-guide](../wiki/wiki.md)
