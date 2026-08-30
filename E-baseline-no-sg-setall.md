---
id: E-baseline-no-sg-setall
title: "performance.apply_baseline_settings claims 'equivalent to a fresh sg.* setall' but touches zero sg.* groups"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [performance, scalability, baseline, doc-mismatch, result-misreport]
---

# `performance.apply_baseline_settings` is documented as an `sg.*` setall but never touches `sg.*`

The handler/wiki description for `performance.apply_baseline_settings` states it is
a **"One-shot configuration — equivalent to a fresh 'sg.*' setall."** That phrasing
is concretely misleading: the implementation sets only seven `r.*` render CVars
(`r.VSync`, `r.AllowHDR`, `r.MotionBlurQuality`, `r.DepthOfFieldQuality`,
`r.BloomQuality`, `r.ShadowQuality`, `r.MaxAnisotropy`) and **never writes a single
`sg.*` scalability group** — it does not call `Scalability::SetQualityLevels`, does
not issue `sg.* <n>` / `scalability <n>`, and does not touch any `sg.ViewDistanceQuality`,
`sg.ShadowQuality`, `sg.EffectsQuality`, `sg.TextureQuality`, etc. A caller who reads
"equivalent to a fresh sg.* setall" and runs this to reset the scalability groups to a
known mid-quality baseline (the exact task that surfaced this) gets a silent no-op on
every `sg.*` group, with a clean success response and no indication the groups were
left untouched.

The response compounds the friction: it is a bare echo of the input,
`{"profile":"balanced"}`, with no applied-CVar report. So neither the description nor
the result tells the caller what actually changed — and what the description *promises*
changed (the `sg.*` groups) is precisely what does **not** change.

This is the giveaway asymmetry against the sibling `performance.set_scalability`, which
DOES drive the `sg.*` groups (`Scalability::SetFromSingleQualityLevel` →
`SetQualityLevels`). To get the documented "fresh sg.* setall" behaviour the caller must
ignore `apply_baseline_settings` and call `set_scalability` instead — exactly the
workaround the description's wording discourages.

Root cause (`Source/.../Handlers/Debug/PerformanceHandler.cpp`, the
`performance.apply_baseline_settings` handler): every branch (`performance` / `quality` /
`balanced`) calls only the local `SetCVar(...)` lambda on `r.*` names; there is no
`sg.*` write and no `Scalability::*` call anywhere in the handler. The registered
description string ("equivalent to a fresh 'sg.*' setall") describes behaviour the code
does not implement.

## Verbatim repro (live, replay-confirmed against `mcp__editor-automation__call`)

1. `performance.set_scalability` `{"level":0}` → `{"message":"Scalability set"}` — establishes a distinct before; `sg.*` groups now read 0.
2. `system.console.search` `{"query":"sg.ShadowQuality"}` → `currentValue:"0"` (also `sg.ViewDistanceQuality`=0, `sg.EffectsQuality`=0).
3. `performance.apply_baseline_settings` `{"profile":"balanced"}` → **`{"profile":"balanced"}`** (bare echo, no applied-CVar report).
4. `system.console.search` `{"query":"sg.ShadowQuality"}` → still `currentValue:"0"`; `sg.ViewDistanceQuality` → still `0`; `sg.EffectsQuality` → still `0`.

So the quotable mismatch: the description says **"equivalent to a fresh 'sg.*' setall"**,
yet after the call every `sg.*` group sits exactly where it was (`0 → 0`, and likewise
`3 → 3` from a default baseline). The `r.*` CVars the handler actually sets (e.g.
`r.ShadowQuality`=3 for `balanced`) are real, but they are not the `sg.*` groups the
docs promise and the result never reports them.

**Impact:** an agent doing a perf-baseline pass that relies on the documented "sg.*
setall" to normalize scalability groups will believe the groups were reset when they
were not, silently profiling at whatever stale `sg.*` state was already live. Low
severity — the call is not destructive and `set_scalability` covers the real need — but
the description actively misdirects.

**Workaround:** to reset the `sg.*` scalability groups, call
`performance.set_scalability {level: N}` (which does drive `sg.*`); treat
`apply_baseline_settings` as an `r.*`-only convenience bundle.

**Fix:** make the description honest — either (a) reword to "sets a curated bundle of
`r.*` render CVars (does not change `sg.*` scalability groups; use
`performance.set_scalability` for those)", or (b) actually drive the `sg.*` groups in
the handler (e.g. `Scalability::SetQualityLevels` per profile) so it lives up to the
"fresh sg.* setall" claim. Either way, return an applied-CVar report in the response
instead of a bare `{"profile":...}` echo so the caller can see what changed.

## History
- `#1-initial-repro` `OPEN` reporter — `performance.apply_baseline_settings {"profile":"balanced"}` returns a bare `{"profile":"balanced"}` echo and leaves every `sg.*` group unchanged (replay-confirmed live: `sg.ShadowQuality`/`sg.ViewDistanceQuality`/`sg.EffectsQuality` stayed `0` across the call after a `set_scalability {level:0}` before-state), despite the registered description claiming it is "equivalent to a fresh 'sg.*' setall". Source confirms the handler (`PerformanceHandler.cpp`, `apply_baseline_settings`) only writes seven `r.*` CVars and never calls `Scalability::*` / writes any `sg.*`. Sibling `performance.set_scalability` is the only RPC that drives `sg.*`. Ergonomic: description concretely misleads + bare-echo result misreports nothing applied.
- `#2-doc-honesty-and-applied-report` `IN-REVIEW` developer — Applied fix option (a): made the description honest and added an applied-CVar report. In `Source/EditorAutomationRpcGateway/Private/Handlers/Debug/PerformanceHandler.cpp` (`performance.apply_baseline_settings`, formerly ~line 513): (1) rewrote the registered summary to drop the false "equivalent to a fresh 'sg.*' setall" claim — it now states the handler sets a curated bundle of `r.*` render CVars, explicitly says it does NOT change `sg.*` scalability groups, and points sg.* users at `performance.set_scalability`; (2) the response is no longer a bare `{"profile":...}` echo — it now carries `appliedCVars` (array of `{cvar, value}` for each `r.*` actually set) plus a machine-checkable `scalabilityGroupsChanged:false` flag spelling out the sg.* no-op (the corrected description and the appliedCVars report already convey the r.*-only scope in prose, so no redundant prose `note` is added). Did NOT take option (b) (driving `sg.*` per profile) — adversarial lens flagged it as scope-creep that risks surprising existing `r.*`-only callers; `set_scalability` already covers the real sg.* need. Regression test added: `EditorAutomationRpcGateway.performance.apply_baseline_settings.HonestDescriptionAndAppliedReport` in `Source/EditorAutomationRpcGateway/Private/Tests/EditorOps/TestDebugHandlers.cpp` — asserts the registered summary no longer says "setall"/claims an sg.* equivalence, and that the live response carries a non-empty `appliedCVars` report containing `r.ShadowQuality`; it would fail if the misleading phrasing or the bare echo were restored.
- `#3-already-fixed-reconcile` `IN-REVIEW` fuzz3-supervisor — Board/origin reconciliation. The fix for this ticket (GO — fix option (a): honest description + applied-CVar report) was implemented, committed, and pushed to origin/master as `4d253f4` ([tests:CLEAN]) at 2026-06-23 03:42, but the OPEN→IN-REVIEW flip and claim-release did not persist, leaving the ticket OPEN with a stale fuzz3 lease (claimedAt 2026-06-23T03:23). Verified `4d253f4` is present in origin/master; flipping OPEN→IN-REVIEW and clearing the stale claim to match the already-landed fix. No code change.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
