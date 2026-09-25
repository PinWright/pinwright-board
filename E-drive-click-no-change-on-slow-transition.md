---
id: E-drive-click-no-change-on-slow-transition
title: "drive.click reports no_change_within_budget for a successful click whose screen transition starts after the 500ms quiet budget — the outcome cannot be told apart from a click that genuinely did nothing without an expensive follow-up observe"
status: OPEN
severity: Medium
category: ergonomic
tags: [drive, drive.click, settle, quiet-budget, changed-false, no-change-within-budget, ambiguous-outcome]
encounters: 3
lastSeen: 2026-09-25T09:07:00Z
---

# drive.click reports `no_change_within_budget` when a successful click's screen transition begins after the quiet budget

## What's wrong
Clicking a main-menu button that navigates to a new screen returned
`{"outcome":"no_change_within_budget","changed":false,"settled":false,"elapsed_ms":619,"ticks":13}`
— yet the click succeeded: the very next `drive.observe` showed the destination
screen (the multiplayer rooms list). The click was NOT a no-op; the settle loop
just closed its observation window before the navigation became visible.

Mechanism (verified from source, not value-blindness — see the cross-reference
below): the settle loop derives its change signal from the shape-only fingerprint
(`DriveSettleDriver.cpp:68` `bChangedSinceBaseline = (Fingerprint != Baseline)`,
computed per engine frame at `:52`/`:66`), and `StepSettle` only latches
`State.bHasChanged` once that fingerprint diverges from the baseline
(`DriveSettleDecision.cpp:19`). While `bHasChanged` is still false, once real
elapsed time crosses `QuietBudgetMs` the loop returns via the never-changed branch
`EDriveSettleOutcome::NoChangeWithinBudget` with `bChanged=false`
(`DriveSettleDecision.cpp:83-88`). The default `QuietBudgetMs` is **500**
(`DriveTypes.h:211`). The reported `elapsed_ms:619` is the first tick past that 500ms
gate (tick 13 of a ~21 fps editor), which proves the fingerprint never diverged
across those 13 ticks — i.e. the destination screen's widgets were not yet built
(menu animation / deferred navigation / async widget or level load) when the quiet
window closed. The transition's **onset** is later than the 500ms quiet budget, so
the loop exits before it can ever see the change.

Note this is the QUIET budget firing, not the settle budget. Had the transition
merely been *slow* (started but not stabilized within budget), `bHasChanged` would
be true and the loop would run to `SettleBudgetMs=1500` (`DriveTypes.h:213`) and
return `timeout` with `changed:true`. The `no_change_within_budget` / `changed:false`
result specifically means "nothing changed in the first 500ms", which for a delayed
navigation is a false negative.

The core ergonomic defect: **the exact same outcome string is returned for a click
that genuinely does nothing.** In the same session a disabled quick-host button
returned `no_change_within_budget` twice (`elapsed_ms` 514/504) and the UI really
had not changed. An agent therefore cannot distinguish "click landed, UI still
transitioning" from "click did nothing" from the action response alone — it must
issue a follow-up `drive.observe`, which at 200-450 KB per observe is expensive and
turns every navigating click into a two-call sequence.

The information to disambiguate already exists inside the handler but is never
surfaced: the pre-action resolve confirms the target was Found + visible + enabled
before injecting (`DriveActionCommon.cpp:196-227`), and `Inject()` returns a bool
whose failure is a distinct `ERR_INPUT_FAILED` (`:253-258`). So "a real, actionable
widget was hit and synthetic input was dispatched" is known — but the response
(`:274-285`) carries only `outcome / changed / settled / condition_met / elapsed_ms /
ticks / diff`, with no field saying the input was dispatched or which widget received
it. There is also an existing escape hatch the outcome never hints at: passing a
`wait_for` condition switches the loop to the 5000ms `WaitForTimeoutMs`
(`DriveActionCommon.cpp:264-268`, `DriveTypes.h:215`) with a real condition check,
which would ride out a delayed transition — but nothing steers the caller to it.

Secondary discovery friction (same session): `settle_ms` was rejected with
`UNKNOWN_PARAMS`; the valid parameter is `settle_budget_ms` (`DriveActionHandlers.cpp:50`,
alongside `quiet_budget_ms` `:49`). The dispatcher's unknown-param error does already
enumerate the valid params (`RpcDispatcher.cpp:136-138`), so this is self-correcting
on the next call — minor, noted for completeness.

## Relationship to `E-drive-settle-changed-false-on-value-edit`
Same symptom string (`no_change_within_budget` / `changed:false` on a successful
action) and the same shared settle driver, but a **different root cause**, so this is
filed separately rather than merged:
- That ticket: a value-only text edit that the *shape-only* fingerprint cannot see by
  design (content is intentionally excluded from `Compute`). No budget change helps;
  its fix is a value-aware one-shot `Diff` + a doc correction — which does nothing for
  this case.
- This ticket: a genuine *shape* change (a full screen swap) that the fingerprint WOULD
  catch, but whose onset falls after the 500ms quiet window. A larger/adaptive budget,
  an input-dispatched signal, or a `wait_for` hint fixes this; the value-aware `Diff`
  fix is irrelevant here.

The shared thread is the meta-gap both expose: `changed:false` is ambiguous between
"the action worked" and "the action did nothing", and the caller pays an extra observe
to tell them apart. A fix that surfaces input-dispatch + hit-widget separately from the
settle signal would improve both.

## What it should do
Pick one (or combine):
- **Surface input dispatch separately from settle** (preferred, also helps the
  value-edit ticket): add e.g. `input_dispatched:true` and the hit widget's
  handle/type to the action response, sourced from the already-known pre-resolve +
  `Inject()` result. Then `changed:false` means "settle window saw no shape change",
  while a distinct field confirms the click actually hit an enabled widget — the agent
  no longer needs a follow-up observe just to learn whether the click landed.
- **Hint the escape hatch in the outcome**: when `no_change_within_budget` is returned
  for an action that dispatched input, include a hint that a delayed transition may be
  in flight and `wait_for` (5s window) or a larger `settle_budget_ms`/`quiet_budget_ms`
  can confirm it.
- **Bump/adapt the default quiet budget for full-screen navigations** — least
  preferred: 500ms is deliberately small to keep quiet reporting snappy, and a blanket
  bump slows every no-op click; only worth it if the first two options are declined.

## Verbatim repro
1. In PIE at the main menu, `drive.click` the МУЛЬТИПЛЕЕР (multiplayer) button ->
   `{"outcome":"no_change_within_budget","changed":false,"settled":false,"elapsed_ms":619,"ticks":13,...}`.
2. `drive.observe` (game surface) -> the multiplayer rooms screen is now shown, i.e.
   the click DID navigate; step 1's `changed:false` was a false negative.
3. Contrast: `drive.click` a disabled quick-host button -> `no_change_within_budget`
   twice (`elapsed_ms` 514/504) with the UI genuinely unchanged — the identical outcome
   string for a true no-op, which is why steps 1 and 3 are indistinguishable without
   step 2's observe.

## Guilty source (ground truth, read verbatim)
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveSettleDecision.cpp:19`
  — `State.bHasChanged |= bChangedSinceBaseline;` only latches once the fingerprint
  diverges; `:83-88` — the never-changed branch returns
  `EDriveSettleOutcome::NoChangeWithinBudget` with `bChanged=false` once
  `ElapsedMs >= Config.QuietBudgetMs`.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveSettleDriver.cpp:52,66,68`
  — per-frame (`:39-43`, 0.0f ticker) fingerprint compute + `bChangedSinceBaseline`.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveTypes.h:211,213,215`
  — `QuietBudgetMs = 500`, `SettleBudgetMs = 1500`, `WaitForTimeoutMs = 5000` defaults.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveActionCommon.cpp:196-227`
  — pre-action resolve confirms the target is Found + visible + enabled; `:253-258`
  `Inject()` returns bool (`ERR_INPUT_FAILED` on failure); `:264-268` the `wait_for`
  path swaps to `WaitForTimeoutMs`; `:274-285` the response carries only
  `outcome/changed/settled/condition_met/elapsed_ms/ticks/diff` — no input-dispatched
  or hit-widget field.
- `Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveActionHandlers.cpp:49-50`
  — valid settle params are `quiet_budget_ms` / `settle_budget_ms` (not `settle_ms`);
  `Plugins/PinWright/Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:136-138`
  — the `UNKNOWN_PARAMS` error already lists the valid params.

severity rationale: impact=misleading result field on a normal path (`changed:false`
after a successful navigating click) that is indistinguishable from a true no-op, but
the click works, the change round-trips via a follow-up `drive.observe`, and it is a
safe-direction false-negative (under-claims success) with a documented workaround
(observe / wait_for) × reach=`drive.click` navigation is an every-session verb ->
Medium. (Matches how the sibling value-edit false-negative was rated.)

## History
- `#1-initial-repro` `OPEN` reporter — Filed: `drive.click` on the main-menu
  МУЛЬТИПЛЕЕР button returned `no_change_within_budget` / `changed:false`
  (`elapsed_ms:619`, `ticks:13`) yet the very next `drive.observe` showed the
  destination multiplayer screen. Root cause is timing, not value-blindness: the
  shape-only fingerprint (`DriveSettleDriver.cpp:68`) never diverged in those 13 ticks
  because the navigation's onset fell after the 500ms `QuietBudgetMs`
  (`DriveTypes.h:211`), so the loop exited via the never-changed branch
  (`DriveSettleDecision.cpp:83-88`). The identical outcome is returned for a genuine
  no-op (disabled quick-host button, `elapsed_ms` 514/504), so an agent cannot tell
  "click landed, UI transitioning" from "click did nothing" without an expensive
  (200-450 KB) follow-up `drive.observe`. Input-dispatch success + hit widget are
  already known inside the handler (`DriveActionCommon.cpp:196-227,253-258`) but not
  surfaced in the response (`:274-285`). Distinct from
  `E-drive-settle-changed-false-on-value-edit` (that is the shape-only fingerprint
  being value-blind; its value-aware-`Diff` fix does nothing here) — cross-referenced;
  a fix surfacing input-dispatch separately from the settle signal would help both.
  Secondary friction: `settle_ms` -> `UNKNOWN_PARAMS`; valid name is `settle_budget_ms`
  (`DriveActionHandlers.cpp:50`), though the error already lists valid params
  (`RpcDispatcher.cpp:136`).
- `#2-diff-contradicts-outcome` `OPEN` reporter — Second sighting (UE 5.8, PDS PIE, school-computer verification). Two `drive.click` calls returned `outcome:"no_change_within_budget"`, `changed:false`, `settled:false` while the SAME response's one-shot `diff` reported the transition: (a) closing the auto-opened Message Log tab (`surface:"editor_chrome"`, tab close `SButton`) -> `elapsed_ms:502`, `diff.disappeared_count:72`, and `drive.list_windows` right after showed the window gone; (b) `W_SchoolNameLogin/AnonymousLoginButton` in PIE -> `elapsed_ms:553`, `diff.appeared_count:228`, `disappeared_count:284` (login overlay dismissed, main menu shown). New evidence for the fix: the response is self-contradictory, so the caller needs no follow-up observe to see the false negative, and the handler could derive `changed` (or at least a warning) from its own non-empty final diff when the settle loop exits via the never-changed branch.
- `#3-drive-type-diff-contradicts-outcome` `OPEN` reporter - Third sighting (UE 5.8, PDS PIE, school name sign-in). `drive.type {handle:"FirstNameBox", text:"Иван"}` returned `outcome:"no_change_within_budget"`, `changed:false`, `settled:false` with `diff.changed_count:2` (`FirstNameBox`, `.../W_SchoolNameLogin/SEditableText[0]`), and the text was in fact entered (the follow-up sign-in succeeded as «Иван Петров»). The next `drive.type` into `LastNameBox` reported `settled_changed`. Same self-contradiction as `#2`, now on `drive.type`.
