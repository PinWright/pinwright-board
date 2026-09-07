---
id: E-add-metasound-node-error-no-hint
title: "NODE_CLASS_NOT_FOUND is a dead-end error — no className hint, no pointer to search_metasound_nodes"
status: OPEN
severity: Low
category: ergonomic
tags: [metasound, audio, authoring, add_metasound_node, error-messages, error-hint, discovery]
blockedBy: [B-add-metasound-node-rejects-registry-classnames]
encounters: 7
costly: 1
lastSeen: 2026-06-24T19:46:41Z
---

# add_metasound_node's NODE_CLASS_NOT_FOUND dead-ends the caller instead of steering it

When `audio.authoring.add_metasound_node` cannot resolve a className, it returns
a flat dead-end error:

> `[NODE_CLASS_NOT_FOUND] Node class '<X>' not found in MetaSound registry`

The message is accurate but offers **no recovery path**: it does not echo any
accepted className, does not list near-match registry entries, and does not name
the sibling RPC (`search_metasound_nodes`) that enumerates valid class names.
Faced with that wall, a caller has no in-band signal of the correct spelling and
falls into a blind guessing spiral.

This is the same ergonomic shape the board has already fixed elsewhere:
[`E-make-struct-error-hint`](E-make-struct-error-hint.md) (DONE — `make<>`
error now suggests the native `Make*` function),
[`E-bpir-createwidget-pin-hint`](E-bpir-createwidget-pin-hint.md) (DONE —
pin-name error now hints the correct `Class:` pin), and
[`B-unknown-params-error-suggests-deleted-question-mark-suffix`](B-unknown-params-error-suggests-deleted-question-mark-suffix.md)
(DONE — param error stopped pointing at a dead discovery form). An accurate
error that doesn't suggest the next step is a recurring friction pattern; this
RPC is the next instance.

## Distinct from the root-cause bug

[`B-add-metasound-node-rejects-registry-classnames`](B-add-metasound-node-rejects-registry-classnames.md)
fixes *why* the right className is rejected (the add resolver and the search
engine speak different className dialects — `Metasound.*` shorthand vs. live
`UE.*` registry keys). This ticket is the **error-ergonomics** angle that
survives that fix: even once the resolver round-trips search output, a caller
who passes a typo'd or genuinely-absent className (e.g. `UE.Sine` instead of
`UE.Sine.Audio`) should get a hint, not a wall. The dead-end error is what
*converted* the underlying resolver bug into an 11-attempt thrash — the caller
could not tell whether the className was wrong, the namespace was wrong, or a
version suffix was needed, so it permuted all of them blindly.

## What it should do

On `NODE_CLASS_NOT_FOUND`, enrich the error with a recovery hint, e.g.:

> `[NODE_CLASS_NOT_FOUND] Node class 'UE.Sine' not found in MetaSound registry.
> Did you mean: UE.Sine.Audio (Generators, v1.1)? Call
> audio.authoring.search_metasound_nodes { query: "Sine" } to list valid
> className values, and pass one back verbatim as nodeClassName.`

Cheapest useful form: run the failed string through the same
`ISearchEngine::FindAllClasses` registry the search RPC walks, surface the top
1–3 near matches (substring/prefix on display name or className) plus the
"call search_metasound_nodes" pointer. Even with zero near matches, the pointer
to the search RPC alone would have collapsed this task's guessing spiral.

## Evidence

From this task's friction note (`audio.authoring.set_metasound_default` focus):
"add_metasound_node returned NODE_CLASS_NOT_FOUND for the exact className that
search_metasound_nodes reports as existing ('UE.Sine.Audio'), for both
documented shorthands, and for ~8 other key formats I tried." Call log:
**11 consecutive `add_metasound_node` failures** (every one a bare
`NODE_CLASS_NOT_FOUND` with no alternative offered) bracketed by **6
`search_metasound_nodes` calls** — the caller kept re-searching to manufacture
the hint the error itself withheld, then guessing spellings off it. Outcome:
step 4 (DSP node wiring) abandoned; the asset saved silent.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the `audio.authoring.set_metasound_default` fuzz task: 11 trial-and-error `add_metasound_node` failures + 6 interleaved `search_metasound_nodes` calls before abandoning step 4. The `NODE_CLASS_NOT_FOUND` error is a dead-end (no near-match list, no pointer to `search_metasound_nodes`), which converted the underlying resolver bug into a blind multi-spelling thrash. Distinct from the judge-filed root-cause bug `B-add-metasound-node-rejects-registry-classnames` (resolver dialect mismatch) — this is the error-ergonomics angle that survives that fix. Follows the established error-hint precedent: `E-make-struct-error-hint`, `E-bpir-createwidget-pin-hint`, `B-unknown-params-error-suggests-deleted-question-mark-suffix` (all DONE).
- `#2-deferred-pending-resolver` `OPEN` developer — DEFERRED pending `B-add-metasound-node-rejects-registry-classnames` reaching DONE. The defect and the precedent are real and the fix is cheap (failure-branch-only edit at `AudioAuthoringHandler.cpp:874`, reusing `Metasound::Frontend::ISearchEngine::Get().FindAllClasses(true)` from `MetaSoundSearchHandler.cpp:50` for near-matches plus a pointer to `search_metasound_nodes`), and the dead-end error site is verified live and untouched in this source tree. BUT this ticket's entire evidentiary basis — the cited 11-attempt thrash on the exact verbatim className (`UE.Sine.Audio`) that search reports as existing — was produced by resolver bug `B-add-metasound-node-rejects-registry-classnames`, whose #2-fix has landed in source (`FMetasoundFrontendClassName::Parse` at `AudioAuthoringHandler.cpp:846-850`, refreshed `UE.*` shorthands at 821-828, `versionMajor`/`versionMinor` at 778-779) but is still IN-REVIEW, not DONE/re-validated. With that resolver fix, `UE.Sine.Audio` now round-trips and the spiral cannot recur; only a genuine-typo nicety (the reporter-authored `UE.Sine` vs `UE.Sine.Audio` example, never observed) survives. Block: defer until `B-add-metasound-node-rejects-registry-classnames` is verified DONE, then re-judge the hint's true value against real (not resolver-induced) friction. No code changed.
- `#3-defer-blocker-still-in-review` `OPEN` developer — DEFER held. Re-checked the blocker `B-add-metasound-node-rejects-registry-classnames`: still `status: IN-REVIEW`, not DONE (per board README:29-30, IN-REVIEW→DONE requires a separate tester, which has not happened). Defect re-confirmed live and untouched in this tree at `AudioAuthoringHandler.cpp:895` (`Ctx.SendError(TEXT("NODE_CLASS_NOT_FOUND"), ...)` — a flat error, no near-match list, no pointer to `search_metasound_nodes`), and the resolver fix is physically present (`FMetasoundFrontendClassName::Parse(ActualClassName, ClassName)` at :868, `versionMajor` driving `AddNodeByClassName` at :876-877). Because that resolver fix already makes `UE.Sine.Audio` round-trip into add, the ticket's 11-attempt evidentiary thrash cannot recur; only the unobserved genuine-typo hint nicety survives, whose true value can only be judged against post-resolver friction. Block: defer until `B-add-metasound-node-rejects-registry-classnames` is verified DONE, then re-judge the hint's value against real (non-resolver-induced) friction. No code changed.
- `#4-defer-blocker-unchanged` `OPEN` developer — DEFER held (third consecutive). Three independent validity lenses (correctness/adversarial/board-historian) agree the defect is real and the fix is cheap but the disposition is DEFER pending the named blocker. Re-verified live in this source tree: blocker `B-add-metasound-node-rejects-registry-classnames` is still `status: IN-REVIEW` (frontmatter line 4; only history entry is `#2-fix` IN-REVIEW, no DONE entry exists) — per board README:48 (`OPEN → DONE is NEVER allowed`) and README:28-30 (developer cannot set DONE unilaterally; a separate tester is mandatory), the defer's lifting condition has not occurred. Defect site still live and untouched at `AudioAuthoringHandler.cpp:895` (flat `NODE_CLASS_NOT_FOUND`, no near-match list, no `search_metasound_nodes` pointer). Resolver fix still physically present: `FMetasoundFrontendClassName::Parse(ActualClassName, ClassName)` at :868 with legacy no-dot fallback at :870, `versionMajor` driving `AddNodeByClassName(ClassName, MajorVersion, ...)` at :877 — so `UE.Sine.Audio` round-trips and the ticket's 11-attempt evidentiary thrash cannot recur; only the unobserved genuine-typo nicety survives, whose value can only be judged against post-resolver friction not yet gathered. Block: defer until `B-add-metasound-node-rejects-registry-classnames` is verified DONE. No code changed.
- `#5-formalize-defer-gate` `OPEN` developer — Recorded the existing defer gate as frontmatter `blockedBy: [B-add-metasound-node-rejects-registry-classnames]` so the fix-loop picker skips this ticket until that blocker reaches DONE/WONTFIX, instead of re-running a full three-lens analysis on it every iteration (board README → "Deferring a ticket"). Disposition unchanged; no code.
- `#6-defer-blocker-still-in-review` `OPEN` developer — DEFER held (fourth consecutive). All three validity lenses concur: correctness confirms the defect is real and the fix sound but the standing defer correct; adversarial and board-historian both vote defer. Re-verified the gate live: blocker `B-add-metasound-node-rejects-registry-classnames` is still `status: IN-REVIEW` (frontmatter line 4), its only history entry `#2-fix` IN-REVIEW with no DONE/WONTFIX transition — per board README:48 (`OPEN → DONE is NEVER allowed`) and README:28-33 (developer cannot self-promote; a separate tester is mandatory), the `blockedBy` gate has not lifted. Defect site still live and untouched at `AudioAuthoringHandler.cpp:895` (flat `Ctx.SendError(TEXT("NODE_CLASS_NOT_FOUND"), ...)`, no near-match list, no `search_metasound_nodes` pointer). The blocker's resolver fix remains physically present (`FMetasoundFrontendClassName::Parse(ActualClassName, ClassName)` at :868, `MajorVersion` driving `AddNodeByClassName(ClassName, MajorVersion, ...)` at :876-877, refreshed `UE.*` shorthands at :842-849), so `UE.Sine.Audio` round-trips and the ticket's 11-attempt evidentiary thrash cannot recur — only the unobserved genuine-typo nicety survives, whose value is re-judgeable only against post-resolver friction not yet gathered. Block: defer until `B-add-metasound-node-rejects-registry-classnames` is verified DONE. No code changed.
- `#7-retriage` `OPEN` triage — Medium→Low: error-hint ergonomic whose real thrash was neutralized by the resolver-fix blocker; only a genuine-typo nicety survives, niche path.
