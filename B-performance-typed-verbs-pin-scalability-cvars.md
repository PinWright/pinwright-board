---
id: B-performance-typed-verbs-pin-scalability-cvars
title: "Third door to the same scalability pin, and the only one a well-behaved caller takes: performance.apply_baseline_settings and performance.configure_texture_streaming write seven ECVF_Scalability cvars through a bare CVar->Set(), which defaults to ECVF_SetByCode — six priority levels above the ECVF_SetByScalability the editor's own Scalability panel writes at — while apply_baseline_settings' response asserts scalabilityGroupsChanged:false and the sg.-prefix guard never sees these verbs at all"
status: OPEN
severity: High
category: bug
tags: [performance, scalability, cvar-priority, ecvf-setbycode, ecvf-setbyscalability, apply_baseline_settings, configure_texture_streaming, silent-side-effect, session-state, user-facing, editor-degradation, guard-gap, typed-verb]
encounters: 1
lastSeen: 2026-08-30T17:40:00+03:00
---

# The guard watches the console; these two verbs walk past it

Two tickets already cover this pin through the **console**:
`B-console-command-sg-cvar-pin-freezes-scalability` (the `sg.<Group> N` spelling, which its guard
refuses) and `B-console-member-cvar-pin-freezes-scalability` (the member cvars, which it does not).
This is the **third door**, and it is the one that matters most to an end user: **it needs no console
string at all.** Two ordinary typed verbs in the `performance.*` namespace do it, and the guard is
in a different file that neither of them includes.

The consequence is the inversion of the guard's own advice. `ScalabilityConsoleGuard`'s refusal text
tells the caller to *"use `performance.set_scalability` instead"* — steering them off the console and
onto typed verbs. Two of the typed verbs in that same namespace inflict the pin themselves.

## The mechanism, re-derived at plugin HEAD `ef8a1f1b` and engine `C:/UE_5.8`

`IConsoleVariable::Set` is reached through the templated helper
`Set(T Value, EConsoleVariableFlags Flags = ECVF_SetByCode, FName Tag = NAME_None)`
(`IConsoleManager.h:766`). **A bare `CVar->Set(x)` therefore writes at `ECVF_SetByCode`.**

    ECVF_SetByScalability = 0x02000000   IConsoleManager.h:159   <- the editor's panel
    ECVF_SetByCode        = 0x0E000000   IConsoleManager.h:183   <- a bare CVar->Set(x)
    ECVF_SetByConsole     = 0x10000000   IConsoleManager.h:187   <- the two console tickets

`FConsoleVariableBase::CanChange` (`Engine/Source/Runtime/Core/Private/HAL/ConsoleManager.cpp:275`)
is `bool bRet = NewPri >= OldPri;` at `:280`. `Scalability::SetQualityLevels` — the path the
editor's **Settings > Engine Scalability Settings** panel writes through — sets at
`ECVF_SetByScalability`, so once one of these cvars sits at `ECVF_SetByCode` **every later change
the human user makes to the owning group is discarded for the rest of the editor session**, logged
as a `LogConsoleManager` Warning by the `else` branch at `ConsoleManager.cpp:305-309`. `SetByCode`
is lower than `SetByConsole`, so this door is one notch less severe than the other two — and still
six levels above the panel, which is all that is needed.

## The two verbs and what they write

**`performance.configure_texture_streaming`** (registered `Handlers/Debug/PerformanceHandler.cpp:558`)

| cvar | write | `ECVF_Scalability`? | group |
|---|---|---|---|
| `r.Streaming.PoolSize` | `:572`, gated on `poolSize` being present at `:568` | **yes** — `Runtime/Engine/Private/Streaming/TextureStreamingHelpers.cpp:120`, flags `:123` | `TextureQuality` (`BaseScalability.ini:749,760,771,782,793`) |
| `r.TextureStreaming` | `:591`, unconditional | no — `TextureStreamingHelpers.cpp:105`, flags `:110` are `ECVF_Default \| ECVF_RenderThreadSafe` | — |

**`performance.apply_baseline_settings`** (registered `:939`) writes seven cvars per profile through
one lambda, `SetCVar` at `:949-956`, whose write is the bare `CVar->Set(Value)` at `:953`:

| cvar | `ECVF_Scalability`? | group |
|---|---|---|
| `r.VSync` | **yes** — `ConsoleManager.cpp:4330`, flags `:4334` | **no ini row** — see below |
| `r.MotionBlurQuality` | **yes** — `ConsoleManager.cpp:4126`, flags `:4130` | `PostProcessQuality` |
| `r.DepthOfFieldQuality` | **yes** — `ConsoleManager.cpp:4173`, flags `:4181` | `PostProcessQuality` |
| `r.BloomQuality` | **yes** — `ConsoleManager.cpp:4082`, flags `:4091` | `PostProcessQuality` |
| `r.ShadowQuality` | **yes** — `ConsoleManager.cpp:4119`, flags `:4123` | `ShadowQuality` |
| `r.MaxAnisotropy` | **yes** — `ConsoleManager.cpp:4361`, flags `:4364` | `TextureQuality` |
| `r.AllowHDR` | no — `Runtime/RenderCore/Private/RenderCore.cpp:322`, flags `:327` are `ECVF_ReadOnly` | — |

**Seven `ECVF_Scalability` cvars across the two verbs, not six**, and the discrepancy is instructive.
`B-console-member-cvar-pin-freezes-scalability` § *Scope note* counted five on
`apply_baseline_settings` by matching against `BaseScalability.ini`. `r.VSync` is the sixth: it
carries `ECVF_Scalability` on its declaration and has **no row in `BaseScalability.ini`** — exactly
the `r.ScreenPercentage` case that same ticket already identified when it wrote *"Refuse on the flag,
not on a hardcoded list … any list built by scanning `BaseScalability.ini` misses it."* The ini scan
undercounts here by precisely the case the sibling ticket predicted. Test the flag.

## The aggravator: `apply_baseline_settings` asserts the opposite

`Resp->SetBoolField(TEXT("scalabilityGroupsChanged"), false)` at `:996`, introduced by the comment at
`:992-995` to be *"machine-checkable"* so a caller *"cannot read a success here as 'the sg.\* groups
were reset'."* The field is **literally true** — no `sg.*` value moved, and
`Scalability::SetQualityLevels` was never called — and **materially false**: after this call six
scalability cvars, spanning three groups, are pinned above the priority the Scalability panel writes
at, and the user's panel will silently stop working for those groups until the editor restarts. A
caller who reads the field for reassurance gets the reassurance and the damage.

That is what makes this a `B-`, not an `E-`: the response does not merely omit the side effect, it
publishes a machine-readable field that a caller will reasonably read as its absence.

## The guard cannot see these verbs

`ScalabilityConsoleGuard::IsScalabilityGroupLine` (`Handlers/ScalabilityConsoleGuard.h:38-41`) is a
prefix test on a **command line string**. It is included in exactly two files —
`Handlers/System/SystemControlHandler.cpp:6` (call site `:611`) and
`Handlers/Editor/EditorCommandHandler.cpp:13` (call site `:302`). **`PerformanceHandler.cpp` does not
include it**; `grep ScalabilityConsoleGuard` over that file returns zero hits. There is no string to
test here, so no widening of that predicate — including the flag-based widening
`B-console-member-cvar-pin-freezes-scalability` asks for — reaches this path. Different file,
different fix, same defect.

## Fix — the correct pattern is already in this tree

`Handlers/Render/PreviewViewportCaptureUtils.cpp` solves exactly this problem for
`r.ViewDistanceScale`: `FScopedViewDistanceScale` reads the cvar's **current** SetBy with
`CVar->GetFlags() & ECVF_SetByMask` and writes back at that same priority (`:161-162`), restoring at
`:174`. Its comment at `:139-142` states the reason in as many words — *"Setting at ECVF_SetByCode
would leave the variable pinned at Code priority after the [capture]"*. So the plugin has already
reasoned this through once and shipped the right shape; these two verbs predate it.

Asked, in order of value:

1. **Write at the cvar's existing priority.** Replace both `Set(x)` sites (`:572`, `:953`) with the
   `GetFlags() & ECVF_SetByMask` read-then-write of `PreviewViewportCaptureUtils.cpp:161-162`. This
   is the whole fix for the durability half and it is two lines. Lift the pattern into a shared
   helper rather than copying it a third time — the same bare `CVar->Set()` shape recurs elsewhere
   in `PerformanceHandler.cpp` (`:1011`, `:1043`, `:1049` lambdas) and a fixer should sweep the file.
2. **Correct or remove `scalabilityGroupsChanged`** (`:996`). If (1) lands, the honest field is
   something like `scalabilityCVarsPinned: false` alongside it; if (1) does not land, the field as
   written is worse than nothing and should say which scalability-flagged cvars the call wrote and
   at what priority.
3. **Report requested-vs-effective.** `appliedCVars` (`:991`) is built from the *requested* value at
   `:954` and never re-reads the cvar, so a write that `CanChange` discarded — the mirror-image
   failure, when something already sits at a higher priority — is reported as applied. One
   `CVar->GetInt()` after the set closes it.

## Severity: High

Impact class **High** on two independent counts of the same band. It is a **silent side effect that
degrades the editor for the remainder of the session** with nothing in the response naming it; and
`scalabilityGroupsChanged: false` is **silent wrong data on a normal path** — a caller reads it,
trusts it, and builds on it. Either count alone reaches High; together they are why this sits in the
same band as its two console siblings rather than below them.

**Critical declined.** No crash, and no asset data is written or lost — nothing here touches a
package. The damaged artefact is editor session state, recoverable by a restart. The restart cost is
real (it is why the editor was restarted today) but it is not the Critical band.

**Medium declined.** Medium is the soft-blocker band, reached via a documented workaround. There is
no workaround at all from inside the tool surface: nothing in PinWright reports a cvar's SetBy
priority to the caller, so a caller cannot even *detect* the pin, let alone undo it — `Unset` is not
exposed and re-setting at a lower priority is what `CanChange` refuses.

**Reach modifier declined in both directions.** No bump up: neither verb runs in almost every
session. No bump down to "rare edge path", and this is the argument to read if you are re-rating it:
these are not obscure verbs, they are **the path the guard's own refusal text steers callers onto**,
and `apply_baseline_settings` is what an agent reaches for at the start of a performance pass — the
same moment the editor's quality settings matter most.

## Not observed — stated plainly, because a stronger claim was available and does not hold

Relayed into this ticket's brief as *"the failure that silently froze the editor's Scalability panel
today"*. **That is not established for this door and is not claimed here.** Checked directly in the
running editor's log (`Saved/Logs/EAContentExamples58.log`, opened 17:15:53 after the restart):
zero occurrences of the `CanChange` discard message *"was ignored as it is lower priority"*, and
zero occurrences of `configure_texture_streaming` or `apply_baseline_settings` — **neither verb has
been called this session.** Today's observed pin was measured through the **console** door and
belongs to `B-console-member-cvar-pin-freezes-scalability`, whose `LastSetBy: Console` readings are
its own evidence.

So: the mechanism here is **source-derived and not RPC-verified**. It was deliberately not
reproduced live, because reproducing it means pinning six scalability cvars in an editor several
agents are sharing — inflicting the defect to prove it. A fixer wanting the live confirmation
should take it in a disposable editor: read `r.MaxAnisotropy`'s SetBy, call
`performance.apply_baseline_settings`, read it again, then move the Scalability panel's Texture
Quality slider and watch the warning appear.

## Distinct from

- **`B-console-command-sg-cvar-pin-freezes-scalability`** (IN-REVIEW, High) — the `sg.<Group> N`
  console line, at `ECVF_SetByConsole`. Door 1. Its fix is the guard.
- **`B-console-member-cvar-pin-freezes-scalability`** (OPEN, High) — a member cvar on a console
  line, also at `ECVF_SetByConsole`, which that guard's `sg.` prefix test does not match. Door 2.
  Its § *Scope note* recorded this finding and explicitly declined to file it here, saying it *"wants
  its own ticket"*; this is that ticket. **Its `encounters` and `lastSeen` are deliberately
  untouched** — that ticket is about a verb passing the caller's string through, this is about a verb
  doing it to the caller unasked, and merging them would hide that the fix sites are in three
  different files.
- **`E-baseline-no-sg-setall`** (IN-REVIEW, Low) — the same verb, the opposite direction: its docs
  once claimed it was *"equivalent to a fresh sg.\* setall"* when it touches no `sg.*` group. That
  is a documentation overclaim about what the verb does **not** do; this is an unreported side
  effect of what it **does**. The two interact and a fixer should read both: whoever closes
  `E-baseline-no-sg-setall` by making the verb actually drive `SetQualityLevels` must not do it with
  a bare `Set()`.
- **`B-set-aa-invalid-method-silent-noop`**, **`B-lighting-configure-shadows-echoes-unmeasured`**,
  **`E-perf-wp-configure-readback-thin`** — the other three board tickets mentioning
  `ECVF_SetByCode`. All are about a *write not taking effect* or *not being read back*; none is about
  the write's priority outliving the call.

## Same shape as

Cross-linked into the session's recurring class (`B-foliage-paint-does-no-ground-projection`
§ *Same shape as*) for the `scalabilityGroupsChanged` half only: the call succeeds, the number it
reports is correct, and the caller is misled because the deciding fact was never reported. The
durable-pin half is not a member of that class — it is a side effect, not a missing readback — and
is instead the third instance of the **one defect, three doors** shape this session established
across the two console tickets.

## Dedup

Searched the board for `apply_baseline_settings`, `configure_texture_streaming`, `ECVF_SetByCode`,
`SetByScalability`, `scalabilityGroupsChanged`, `r.MaxAnisotropy`, `r.Streaming.PoolSize` and every
`*scalability*` and `*console*` filename. Eight files mention the two verbs and five mention
`ECVF_SetByCode`; all are distinguished above. **Nothing on the board says a typed verb creates the
scalability pin** — the only prior record is the § *Scope note* that asked for this file.

## Provenance

Source re-derived at plugin HEAD `ef8a1f1b`, clean working tree. The running editor is the 13:32
build, reported as plugin commit `d8f1bc32`;
`git diff d8f1bc32..ef8a1f1b -- Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp
Source/PinWright/Private/Handlers/ScalabilityConsoleGuard.h` is **empty**, so both files are
byte-identical between the build and HEAD and every line cited here applies to the running binary.
Engine citations opened in `C:/UE_5.8`. Log evidence read from the live
`Saved/Logs/EAContentExamples58.log`.

## History
- `#1-typed-verbs-pin-above-the-panel` `OPEN` reporter — Third route to the scalability pin the two console tickets cover, and the one that needs no console string. `performance.configure_texture_streaming` (`Handlers/Debug/PerformanceHandler.cpp:558`) and `performance.apply_baseline_settings` (`:939`) write scalability cvars through a bare `CVar->Set(x)` — `:572` and the `SetCVar` lambda's `:953` — and the templated helper at `IConsoleManager.h:766` defaults its flags to `ECVF_SetByCode = 0x0E000000` (`:183`), six levels above the `ECVF_SetByScalability = 0x02000000` (`:159`) that `Scalability::SetQualityLevels` and therefore the editor's own Settings > Engine Scalability Settings panel write at, so `FConsoleVariableBase::CanChange` (`ConsoleManager.cpp:275`, `bRet = NewPri >= OldPri` at `:280`) discards every later panel change to the owning group for the session, logging a Warning via the `else` at `:305-309`. **Corrected the relayed count while re-deriving: seven `ECVF_Scalability` cvars across the two verbs, not six.** `apply_baseline_settings` writes six, not five — the scope note on `B-console-member-cvar-pin-freezes-scalability` matched against `BaseScalability.ini` and so missed `r.VSync`, which carries `ECVF_Scalability` on its declaration (`ConsoleManager.cpp:4330`, flags `:4334`) with **no ini row** — precisely the `r.ScreenPercentage` case that same ticket identified when it wrote *"Refuse on the flag, not on a hardcoded list."* Every membership re-derived from the declaration rather than the ini: `r.MotionBlurQuality` (`:4126`/`:4130`), `r.DepthOfFieldQuality` (`:4173`/`:4181`), `r.BloomQuality` (`:4082`/`:4091`), `r.ShadowQuality` (`:4119`/`:4123`), `r.MaxAnisotropy` (`:4361`/`:4364`), `r.Streaming.PoolSize` (`TextureStreamingHelpers.cpp:120`/`:123`); and the two non-members named so a fixer does not over-scope — `r.TextureStreaming` is `ECVF_Default` (`TextureStreamingHelpers.cpp:105`/`:110`) and `r.AllowHDR` is `ECVF_ReadOnly` (`RenderCore.cpp:322`/`:327`). **The aggravator:** `apply_baseline_settings` publishes `scalabilityGroupsChanged: false` at `:996` — added by the comment at `:992-995` to be *"machine-checkable"* — which is literally true (no `sg.*` value moved) and materially false (six scalability cvars across three groups are now pinned above the panel), so the response actively asserts the absence of the side effect it just caused. **The guard cannot reach this path:** `ScalabilityConsoleGuard::IsScalabilityGroupLine` (`ScalabilityConsoleGuard.h:38-41`) is a prefix test on a command-line string, included only by `SystemControlHandler.cpp:6` (`:611`) and `EditorCommandHandler.cpp:13` (`:302`); `grep ScalabilityConsoleGuard PerformanceHandler.cpp` returns zero hits and there is no string here to test, so even the flag-based widening the sibling ticket asks for does not reach it. Fix asked in three parts, with the correct pattern already in the tree: `FScopedViewDistanceScale` (`PreviewViewportCaptureUtils.cpp:161-162`, restore `:174`, rationale `:139-142`) reads `GetFlags() & ECVF_SetByMask` and writes back at that same priority — lift it to a shared helper and sweep the file, since `:1011`/`:1043`/`:1049` carry the same bare-`Set` shape; correct or remove `scalabilityGroupsChanged`; and re-read the cvar after the set so `appliedCVars` (`:991`, built from the requested value at `:954`) stops reporting a discarded write as applied. **Stated as not observed:** the relayed claim that this is what froze the Scalability panel today does **not** hold for this door — the live log (`Saved/Logs/EAContentExamples58.log`, opened 17:15:53) contains zero `"was ignored as it is lower priority"` lines and zero calls to either verb this session, so today's pin was through the console door and belongs to the sibling ticket. Source-derived, **not RPC-verified, and deliberately not reproduced**: reproducing it means pinning six cvars in an editor several agents share, i.e. inflicting the defect to prove it; the live check is written out for a disposable editor. Dedup: searched `apply_baseline_settings`, `configure_texture_streaming`, `ECVF_SetByCode`, `SetByScalability`, `scalabilityGroupsChanged`, `r.MaxAnisotropy`, `r.Streaming.PoolSize` and every `*scalability*`/`*console*` filename; the two console tickets, `E-baseline-no-sg-setall` and the three other `ECVF_SetByCode` tickets are distinguished in § *Distinct from*, and nothing on the board says a typed verb creates this pin. Filed as its own ticket exactly as `B-console-member-cvar-pin-freezes-scalability` § *Scope note* asked, with **that ticket's `encounters` and `lastSeen` left untouched** — three doors, three files, one defect, and merging them would hide that the fix sites differ. Severity High on two independent counts of the band: a silent session-durable degradation of the user's editor with nothing in the response naming it, and a machine-readable field asserting its absence. Critical declined — no crash, nothing written to a package, and the damaged artefact is session state a restart recovers. Medium declined — no workaround exists inside the tool surface at all, since nothing in PinWright reports a cvar's SetBy priority, so a caller cannot even detect the pin. Reach declined in both directions: neither verb is every-session, but neither is a rare edge path — they are the path the guard's own refusal text steers callers onto.
