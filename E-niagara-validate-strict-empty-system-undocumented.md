---
id: E-niagara-validate-strict-empty-system-undocumented
title: "niagara.validate strict level: wiki doesn't say it escalates NO_EMITTERS/NO_RENDERERS to errors, so strict-on-an-empty-system always hard-errors with no doc warning"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, validate, strict, docs, empty-system, discovery]
encounters: 3
lastSeen: 2026-07-02T17:06:00.3561033+03:00
---

# `niagara.validate level:strict` escalation is undocumented; strict on a deliberately-empty system always errors

`niagara.validate` accepts `level: basic | strict`, but neither the
generated `niagara.validate` page nor the `niagara.md` overlay states
**what strict actually changes**. Under strict, three normally-`warning`
codes are promoted to hard `error`:
`NO_EMITTERS`, `DISABLED_EMITTER`, `NO_RENDERERS`
(`NiagaraInspectHandler.cpp` `NormalizeValidationSeverity`, lines 85-95;
the `NO_EMITTERS` issue itself is raised in
`NiagaraDumpBuilder.cpp:1500-1508`).

The wiki page only says:

```
- level (string, optional): Validation level: basic or strict. Defaults basic.
```

No list of what strict promotes, and — more importantly — no note that a
system produced by `niagara.create_system` (which the `niagara.md`
overlay correctly documents as starting with **no emitter handles**) will
**always** fail strict validate with `NO_EMITTERS` as an error, regardless
of what the agent was actually trying to validate.

## Why this is friction (process, not the save bug)

This task's plan (focus `niagara.rename_parameter`) built an empty
`FX_Projectile` system, did the renames, then ran step 10:
*"Validate the system to make sure the renames didn't introduce dangling
references or errors (niagara.validate, level strict)."* On an
emitter-less system that strict call is **guaranteed** to return a hard
`NO_EMITTERS` error that has nothing to do with the renames the agent was
validating. The agent had to manually disambiguate the result, recording
in the call log: `strict error NO_EMITTERS (inherent to empty system; no
old-name refs)`. That is interpretive friction — the agent had to reason
out, with no doc support, that the strict error was an inherent
empty-system artifact and not a rename regression. A less careful agent
could read the hard error as "the renames broke something" and chase a
non-bug, or conversely dismiss a real strict error as "probably just the
empty-system thing."

The fix is a downstream wiki edit (not mine): on the `niagara.validate`
H3 in `docs/wiki-src/niagara.md` (or a short `## Validation levels`
section on that overlay), document (a) the exact codes strict escalates
(`NO_EMITTERS`, `DISABLED_EMITTER`, `NO_RENDERERS`), and (b) the steer
that strict validate on a freshly-created emitter-less system always
errors on `NO_EMITTERS` by design — use `basic` (or add an emitter
first) when validating an in-progress/empty system for *other* issues
such as dangling parameter references after a rename. The overlay already
warns that "a freshly-created system without emitters renders nothing";
this is the validation-side corollary of that same note.

## Evidence

- Friction note for the task: *"none - all calls succeeded first try"* —
  i.e. no tool error, but the agent still had to hand-annotate the strict
  `NO_EMITTERS` result as inherent. The friction is interpretive, not a
  failed call.
- Call log: step 10 `niagara.validate FX_Projectile level=strict` →
  `ok:true` with `error_text: "strict error NO_EMITTERS (inherent to
  empty system; no old-name refs)"`. 1 call, no retry — the cost was
  reasoning, not retries.
- Source confirms the escalation is intentional and tested
  (`TestNiagaraCanonicalHandlers.cpp` `FNiagaraValidateStrictNoEmittersErrorTest`),
  which is exactly why it should be documented rather than treated as a
  bug.

## Cross-ref

- `B-niagara-save-no-disk-write` (OPEN) — the OUTCOME bug filed by the
  per-finding judge for this same task (save reports `saved:true` but
  writes nothing). This ticket is the distinct PROCESS/docs angle on
  step 10's strict-validate, not the save defect.
- `niagara.md` overlay already documents the empty-system-renders-nothing
  gotcha (`docs/wiki-src/niagara.md`, "Workflow gotcha"); this is the
  validate-level corollary that's currently missing.
- Wiki page to improve: `docs/wiki-src/niagara.md` (the
  `### niagara.validate` H3 / a `## Validation levels` section).

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the `niagara.rename_parameter` task. The plan's step 10 ran `niagara.validate level:strict` on the emitter-less `FX_Projectile` that step 1 created; strict escalates `NO_EMITTERS` from warning to error (`NiagaraInspectHandler.cpp` NormalizeValidationSeverity, `NiagaraDumpBuilder.cpp:1500-1508`), so the call is guaranteed to hard-error independent of the renames being validated. The agent had to manually annotate the result as "inherent to empty system; no old-name refs" — interpretive friction with no doc support (1 call, no retry, no tool error). The `niagara.validate` wiki page documents `level: basic or strict` but never lists which codes strict promotes nor warns that strict-on-an-empty-system always errors. Proposed downstream wiki fix: on the `niagara.validate` H3 in `docs/wiki-src/niagara.md`, list the escalated codes (NO_EMITTERS/DISABLED_EMITTER/NO_RENDERERS) and steer agents to `basic` (or add an emitter first) when validating an in-progress empty system for other issues. Distinct from the judge's `B-niagara-save-no-disk-write` (save outcome). Dedup: ripgrep + qmd found no existing validate-strict docs ticket; `F-rpc-niagara-inspect-standalone-script` is a different scope, and the strict behavior is intentional and tested, so docs (not a fix) is the right lever.
- `#3-additional-no-renderers-populated-single-emitter` `OPEN` reporter — Additional evidence: NEW ANGLE on the `NO_RENDERERS` escalation this ticket already names but had never demonstrated (prior repros #1/#2 were all `NO_EMITTERS` on a deliberately-empty system). Replayed live against `mcp__pinwright__call` a "campfire" build (`/Game/VFX/ReplayCampfire`): `niagara.create_system` NS_ReplayCampfire, two `niagara.create_emitter` (Flames, Embers), two `niagara.add_emitter`, `niagara.remove_emitter {emitter:"Embers", compile:true, save:true}` (the seed — worked cleanly: `removed:true, remainingEmitters:1`), leaving a POPULATED single-emitter system whose sole emitter is a blank `create_emitter` template with `renderers:[]`. `niagara.validate {level:"strict"}` on that system returned `valid:false` with `errors:[{severity:"error", code:"NO_RENDERERS", message:"Emitter has no renderers.", emitter:"Flames"}]` — so a perfectly-valid single-emitter system built entirely from the niagara authoring RPCs hard-fails strict validate purely because the stock blank emitter ships no renderer, exactly the `NO_RENDERERS` escalation branch of `NiagaraInspectHandler.cpp` `NormalizeValidationSeverity` (lines 101-107) that this ticket documents. The agent's only fix was a realistic authoring step (`niagara.add_renderer` a sprite → re-validate → `valid:true, errors:[]`). This confirms the docs-note should call out `NO_RENDERERS` on a freshly-`create_emitter`'d emitter — not just `NO_EMITTERS` on an empty system — as a strict escalation that always fires until a renderer is added; the two triggers share the same undocumented-escalation root. Supplementary observation for the wiki note: within the SAME strict-validate response, the reused dumper block reports the identical finding as `compile.issues[].severity: "warning"` while the top-level normalized `issues`/`errors` report it as `severity: "error"` (the top-level applies `NormalizeValidationSeverity`; the nested `compile.issues` carries the raw pre-normalization dumper severity). The top-level fields are authoritative and correct, so this is a documentation/layering nuance rather than a separate bug (kin to `E-niagara-validate-compile-state-uninitialized-undocumented` #2's "inner compile.valid:false is benign pre-compile noise"), but a reader diffing the two blocks sees warning-vs-error for one code and should be told which layer is authoritative. Seed `niagara.remove_emitter` itself is clean; culprit is `niagara.validate` strict escalation. 7 niagara RPCs, all first-try; the only interpretive/authoring cost was the NO_RENDERERS strict escalation.
- `#2-basic-level-corroboration` `OPEN` reporter — Second struggle-audit of a `niagara.rename_parameter` task (focus `niagara.rename_parameter`, system `/Game/FuzzVFX/NS_MuzzleFlash`) corroborates the same docs gap at the **basic** level. This run's plan step 7 ran `niagara.validate level:basic` (not strict) on the emitter-less system that step 1 created; basic does NOT escalate, so the call returned `ok:true` with `errors:[]` — but the agent still had to hand-annotate the result in self_report as *"validate basic returned errors:[] (only a NO_EMITTERS warning, expected for an empty system)."* Same interpretive friction (the agent had to reason, with no doc support, that the NO_EMITTERS warning is an inherent empty-system artifact and not a rename regression), same root cause (`niagara.create_system` produces an emitter-less system that always emits NO_EMITTERS — warning at basic, escalated to error at strict per `NiagaraInspectHandler.cpp` NormalizeValidationSeverity), same downstream fix lever (`docs/wiki-src/niagara.md` validate H3 / `## Validation levels` section). 9 RPCs, all first-try, zero retries; friction note literally *"none"* — confirming this is interpretive/docs friction invisible in the call-log, not a tool error. Strengthens the case that the proposed wiki note should state NO_EMITTERS is expected on any freshly-created emitter-less system at BOTH levels (warning at basic, hard error at strict), not just steer strict→basic.
