---
id: E-show-stats-no-category-pointer-to-performance
title: "editor.show_stats is hardcoded FPS+Unit with no category param and its docs steer to the raw console hatch, not the typed performance.show_stats {category} — a 'show GPU too' intent splits across namespaces and ends in a redundant Stat None"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, editor, performance, show_stats, hide_stats, stat, gpu, cross-namespace, discoverability]
encounters: 1
lastSeen: 2026-06-24T06:10:13Z
---

# `editor.show_stats` has no category and points at the console hatch instead of `performance.show_stats {category}`, splitting a multi-stat intent across namespaces

A "turn on FPS+Unit, and also GPU" profiling intent has a clean, single-verb
path — `performance.show_stats {category}` toggles **any** stat group by name
(`unit`, `fps`, `gpu`, `scenerendering`, …) and is the parameterized verb a
multi-stat task wants. But an agent that starts in the `editor.*` namespace (the
natural home for "the editor viewport") lands on `editor.show_stats`, which is:

- **hardcoded to exactly `Stat FPS` + `Stat Unit`** with `RPC_NO_PARAMS`
  (`EditorCommandHandler.cpp:645` — `GEditor->Exec("Stat FPS")` then
  `Exec("Stat Unit")`, no category argument); and
- documented to reach any other stat via the **raw console escape hatch** — its
  own handler summary says *"For other named stats, issue editor.console_command
  with 'Stat <Name>'"*, and the `editor.md` overlay lists `editor.show_stats`
  only in the one-line "Stats / capture" bullet (no per-method `###` section)
  with **no pointer** to the typed `performance.show_stats {category}` sibling.

So an agent following the `editor.*` docs to add a GPU overlay does NOT discover
the typed verb; it falls back to `editor.console_command "Stat GPU"`. The
namespace then splits across the intent: FPS+Unit via `editor.show_stats`, GPU
via the console hatch. The cleanup side then mismatches too — `editor.hide_stats`
issues a single `Stat None` (`EditorCommandHandler.cpp:684`), which already
clears **every** active stat HUD including the console-added GPU one, but because
the surface is split the agent can't tell `hide_stats` covers the
console-enabled stat and tacks on a **redundant** `editor.console_command
"Stat None"` to be safe — `Stat None` issued twice for one clear.

The capability is present and every call succeeds; this is purely a
discoverability/ergonomic gap in how the `editor.*` stat verbs advertise their
limits and their typed partner.

## What's awkward

- `editor.show_stats` quietly enables only two stats but its name implies "show
  the stats", so an agent batching "FPS + Unit + GPU" reasonably expects it to
  cover GPU and learns otherwise only by reading the handler summary.
- That summary then routes the agent to the **raw console** (`editor.console_command
  "Stat <Name>"`) rather than the typed, injection-sanitized
  `performance.show_stats {category}` that already exists for exactly this — so
  the discoverable path is the lower-level one.
- The split surface (`editor.show_stats` = FPS+Unit only / `editor.hide_stats`
  = total `Stat None`) gives no signal that `hide_stats` clears
  console-added stats too, inviting a redundant explicit `Stat None` on cleanup.
- A *sibling* profiling task (see `E-performance-run-benchmark-async-poll-undocumented`,
  step sequence `show_fps → show_stats unit → show_stats gpu → …`) used the
  clean `performance.show_stats {category}` verb throughout — proving the
  parameterized path is the intended one and that `editor.*`-rooted tasks are
  the ones that miss it.

## What it should do (docs-only; no code change required)

Add a `### editor.show_stats` (and matching `### editor.hide_stats`) section to
`docs/wiki-src/editor.md`:

1. State plainly that `editor.show_stats` is a **fixed FPS+Unit convenience** (no
   parameters) and that `editor.hide_stats` issues `Stat None`, which clears
   **all** stat HUDs — including any enabled via `editor.console_command "Stat
   <Name>"` or `performance.show_stats`. So after enabling extra stats, a single
   `editor.hide_stats` is sufficient; do not also issue `Stat None`.
2. For **any other stat group** (GPU, scenerendering, chaos, …), point to the
   typed `performance.show_stats {category}` (toggles `stat <category>`,
   sanitized) as the first choice, and `editor.console_command "Stat <Name>"`
   only as the raw fallback. Mirror the pointer the other direction is already
   getting: `insights.stat-companions.md:36` already names
   `performance.show_stats` as "the lighter call" — the `editor.*` overlay should
   carry the same pointer.

A cheaper-still alternative (code, out of scope) would be to give
`editor.show_stats` an optional `categories` array so the FPS+Unit+GPU intent is
one call — but the docs pointer to the existing `performance.show_stats
{category}` is the minimal fix.

## Evidence (this task — on-screen profiling pass, seed `editor.show_stats`)

15-call clean run; self-reported friction "none". The PROCESS cost shows in the
call log:

- step 3 (FPS+Unit): `editor.show_stats {}` — hardcoded FPS+Unit, fine.
- step 4 (GPU): `editor.console_command "Stat GPU"` — fell back to the raw
  console hatch because `editor.show_stats` can't add GPU and the docs point
  here, not at `performance.show_stats {category:"gpu"}`.
- step 8 (clear): `editor.hide_stats {}` (`Stat None`) **then**
  `editor.console_command "Stat None"` — a redundant second `Stat None`;
  `editor.hide_stats` had already cleared everything (GPU included). One logical
  "clear" became two calls because the split surface gave no signal that
  `hide_stats` covers the console-added stat.

So a single-intent "show FPS+Unit+GPU, then clear" became
`show_stats` + `console_command Stat GPU` … + `hide_stats` + redundant
`console_command Stat None` — three stat-toggle calls and one wasted clear,
where `performance.show_stats unit/gpu` + a single `hide_stats` would do it,
all because the `editor.*` overlay never points at the typed sibling.

Distinct from `E-console-search-stat-subcommand-blind` (IN-REVIEW; that is about
`system.console.search` false-negativing a `stat unit` lookup — discovery of a
console *name*, not the `editor.show_stats`↔`performance.show_stats` cross-pointer)
and from `E-performance-run-benchmark-async-poll-undocumented` (the async
`run_benchmark` poll contract — same family of sibling task, different friction).

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean 15-call on-screen profiling-pass task (seed `editor.show_stats`; outcome clean; self-reported friction "none"). The PROCESS angle the self-report missed: `editor.show_stats` is hardcoded `Stat FPS`+`Stat Unit` with `RPC_NO_PARAMS` (`EditorCommandHandler.cpp:645`) and its docs steer to the raw `editor.console_command "Stat <Name>"` hatch (handler summary; `editor.md` "Stats / capture" bullet, no `###` section) rather than the typed parameterized sibling `performance.show_stats {category}` (`PerformanceHandler.cpp:94`, takes `category` e.g. `gpu`/`unit`/`fps`). Consequence in the call log: GPU enabled via the console hatch (step 4) instead of `performance.show_stats {category:"gpu"}`, and a **redundant** `editor.console_command "Stat None"` after `editor.hide_stats` (step 8) — `hide_stats` already issues `Stat None` (`:684`) which clears all HUDs incl. the console-added GPU, but the split surface gave no signal of that. Dedup: ripgrep over OPEN+closed (qmd unavailable) — no ticket covers the `editor.show_stats`→`performance.show_stats` cross-pointer or the redundant-clear; `E-console-search-stat-subcommand-blind` (console-name search false-negative) and `E-performance-run-benchmark-async-poll-undocumented` (async poll, sibling task) are distinct. Proposed: docs-only — add `### editor.show_stats`/`### editor.hide_stats` sections to `docs/wiki-src/editor.md` stating the fixed FPS+Unit scope, that `hide_stats`'s `Stat None` clears ALL stats (no redundant `Stat None` needed), and pointing to `performance.show_stats {category}` (then console hatch as fallback) for any other stat group; mirrors the pointer `insights.stat-companions.md:36` already gives the other direction.
