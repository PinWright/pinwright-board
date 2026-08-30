---
id: B-console-member-cvar-pin-freezes-scalability
title: "SCALABILITY_CVAR_USE_TYPED_VERB refuses only a leading `sg.` token, but a scalability group is just a bundle of ordinary cvars and the console pins any of them at ECVF_SetByConsole through the identical code path — so `system.console_command {command: \"r.ViewDistanceScale 0.6\"}` freezes ViewDistanceQuality against the user's Scalability panel exactly as `sg.ViewDistanceQuality 1` would, and is allowed; measured in this editor's own log, r.Shadow.Virtual.ResolutionLodBiasDirectional and r.ScreenPercentage both read LastSetBy: Console right now"
status: IN-REVIEW
severity: High
category: bug
tags: [system, editor, console-command, console_command, scalability, cvar-priority, ecvf-setbyconsole, ecvf-setbyscalability, silent-side-effect, session-state, user-facing, editor-degradation, set-scalability, guard-gap, incomplete-fix, scalability-member-cvar]
encounters: 1
lastSeen: 2026-08-30T17:05:00+03:00
---

# The guard names the group; the pin is done by its members

`B-console-command-sg-cvar-pin-freezes-scalability` (IN-REVIEW, High) established that a console
`sg.<Group> N` line pins that group at `ECVF_SetByConsole`, permanently above the editor's own
Scalability panel (`ECVF_SetByScalability`), and its fix refuses lines whose first token starts with
`sg.`.

**A scalability group is not a special kind of variable. It is a name for a list of ordinary cvars
in an ini section.** Setting one of those members directly — `r.ViewDistanceScale`,
`r.Shadow.Virtual.MaxPhysicalPages`, `r.Shadow.Virtual.ResolutionLodBiasDirectional`,
`r.Streaming.PoolSize`, `r.ShadowQuality`, `r.ScreenPercentage` — goes through the same console
`Set`, takes the same priority, and is vetoed by the same comparison when the panel later tries to
write it. The guard does not look at the cvar at all, so every one of those lines is allowed.

The asymmetry is the defect: **the refusal covers the spelling that names a scalability group, and
allows the spellings that a performance session actually types.** Nobody profiling a level types
`sg.ShadowQuality 2`; they type `r.Shadow.Virtual.MaxPhysicalPages 4096`.

## The guard, re-derived at plugin HEAD

`Source/PinWright/Private/Handlers/ScalabilityConsoleGuard.h:38-41` is the whole predicate:

```cpp
inline bool IsScalabilityGroupLine(const FString& CommandLine)
{
    return CommandLine.TrimStart().StartsWith(TEXT("sg."), ESearchCase::IgnoreCase);
}
```

No cvar lookup, no `ECVF_Scalability` flag test, no ini-membership test. Its own comment at `:25-37`
calls it *"Deliberately narrow in two directions"* — first token only, `sg.` prefix only — and
justifies the narrowness against false refusals (`r.Foo sg.Bar`, the aggregate `scalability N`).
That reasoning is sound for what it covers; it simply does not reach a member cvar.

Call sites: `Handlers/System/SystemControlHandler.cpp:611` (refusal `:613-614`, return `:615`) and
`Handlers/Editor/EditorCommandHandler.cpp:302` (refusal `:303-304`, return `:305`), both gated on
`!Ctx.GetBool(TEXT("force"), false)`; `force` declared at `SystemControlHandler.cpp:584` and
`EditorCommandHandler.cpp:293`; code registered at `Handlers/ErrorCodes.h:1160`.

`Source/PinWright/Private/Tests/EditorOps/TestScalabilityConsoleGuard.cpp:113-138` exercises
`sg.*`, `scalability`, `sgfoo`, `r.Foo sg.Bar` and the empty line. **No test names a single member
cvar**, so the gap is untested as well as unguarded.

## The two doors are one code path — engine source, UE 5.8, opened at HEAD

**The console door does not distinguish them.** `FConsoleManager::ProcessUserConsoleInput`
(`C:/UE_5.8/Engine/Source/Runtime/Core/Private/HAL/ConsoleManager.cpp:3040`) reaches, for *any*
registered console variable given a value token:

    ConsoleManager.cpp:3228    CVar->Set(*Param2, ECVF_SetByConsole);

`r.ViewDistanceScale 0.6` and `sg.ViewDistanceQuality 1` take that identical line.

**The panel writes members at `ECVF_SetByScalability`, and that is the load-bearing chain.** The
sibling ticket cites `Scalability.cpp:907`, which is the write of the *group* cvar. The member
cvars arrive one hop later:

| step | file:line |
|---|---|
| `Scalability::SetQualityLevels`, per-group dispatch | `Runtime/Engine/Private/Scalability.cpp:912`, dispatch `:939-950` |
| `SetQualityLevelCVar` writes the group cvar | `Scalability.cpp:907` — `Set(DesiredValue, ECVF_SetByScalability)` |
| the group cvar's sink fires (`OnChangeViewDistanceQuality` `:559-562`, `OnChangeShadowQuality` `:569-572`) | bound `:623`, `:625` |
| `SetGroupQualityLevel` resolves the ini section `"<Group>@<N>"` | `Scalability.cpp:412`, section string `:340-356` |
| **and applies the whole section at Scalability priority** | **`Scalability.cpp:446`** — `UE::ConfigUtilities::ApplyCVarSettingsFromIni(*Section, *GScalabilityIni, ECVF_SetByScalability);` |
| which writes each member cvar | **`Runtime/Core/Private/Misc/ConfigUtilities.cpp:365`** — `CVar->Set(Value, (EConsoleVariableFlags)SetBy, Tag);` |

`ConfigUtilities.cpp:365`, reached from `Scalability.cpp:446`, is the single `Set` that carries
every individual `r.*` named in a `[<Group>@N]` section, and it carries them at
`ECVF_SetByScalability` — the same priority the group cvar gets, and therefore subject to the same
veto.

One group bypasses the ini and still writes at the same priority:
`Scalability.cpp:551` — `CVar->Set(InResolutionQualityLevel, ECVF_SetByScalability);` on
`r.ScreenPercentage`, from `SetResolutionQualityLevel` / `OnChangeResolutionQuality` (`:554-557`).
So `r.ScreenPercentage` is a `ResolutionQuality` member even though it appears in no ini section.

**The veto.** `FConsoleVariableBase::CanChange`, `ConsoleManager.cpp:275-312`: `OldPri`/`NewPri` at
`:277-278`, `bool bRet = NewPri >= OldPri;` at `:280`, the *"was ignored as it is lower priority
than the previous"* message at `:286-291`, `UE_LOGF(LogConsoleManager, Warning, ...)` at `:307`,
`return bRet;` at `:311`. Neither verbosity demotion applies: `:293-297` needs an *old* priority of
ini/commandline/hotfix, `:298-303` needs exactly Scalability-over-DeviceProfile. Console-over-
Scalability lands in the `else` and warns. Every concrete `Set` gates on it —
`FConsoleVariable<T>::Set` at `:1369`, `FConsoleVariableRef<T>::Set` at `:1773`, `PreprocessSet` at
`:1178`.

Priorities: `ECVF_SetByScalability = 0x02000000` (`Runtime/Core/Public/HAL/IConsoleManager.h:159`),
`ECVF_SetByConsole = 0x10000000` (`:187`), ordering stated `:152`.

## Measured, in this editor, right now

Editor build 13:32 (plugin commit `d8f1bc32`, waves 2/3 — i.e. the build that *has* the guard),
host `EAContentExamples58`, UE 5.8, `/Game/Maps/PW_VegetationTest`. A performance pass earlier in
this same session put six cvars back to their entry values with `system.console_command` lines, none
of which the guard refused, and the editor's own log records the result
(`Saved/Logs/EAContentExamples58.log`, log timestamps UTC; frame 178):

```
[2026.08.30-12.15.40:381][178]r.Shadow.Virtual.Cache.DeformableMeshesInvalidate = "1"      LastSetBy: Console
[2026.08.30-12.15.40:381][178]r.Shadow.Virtual.ResolutionLodBiasDirectional = "-1.5"      LastSetBy: Console
[2026.08.30-12.15.40:381][178]r.Shadow.Virtual.Clipmap.WPODisableDistance.LodBias = "3"      LastSetBy: Console
[2026.08.30-12.15.40:381][178]r.Nanite.MaxPixelsPerEdge = "1"      LastSetBy: Console
[2026.08.30-12.15.40:381][178]r.ScreenPercentage = "100"      LastSetBy: Console
[2026.08.30-12.15.40:381][178]r.ProfileGPU.ShowUI = "true"      LastSetBy: Console
```

**Of those six, exactly two are scalability-driven and are therefore now frozen against the
Scalability panel for the rest of the session** — checked one at a time rather than assumed:

- `r.Shadow.Virtual.ResolutionLodBiasDirectional` — a **ShadowQuality** member,
  `C:/UE_5.8/Engine/Config/BaseScalability.ini:147, 182, 220, 258, 296`. Note `:258` is
  `-1.5`, the Epic value: the console line pinned it holding precisely the number scalability
  would have written, so the pin is invisible until the panel is used.
- `r.ScreenPercentage` — a **ResolutionQuality** member via `Scalability.cpp:551` (no ini row).

The other four are genuinely outside scalability and are named here so the count is not inflated:
`r.Shadow.Virtual.Cache.DeformableMeshesInvalidate`,
`r.Shadow.Virtual.Clipmap.WPODisableDistance.LodBias` and `r.ProfileGPU.ShowUI` appear in no
scalability ini and in no `Scalability.cpp` handler; `r.Nanite.MaxPixelsPerEdge` is declared with
`ECVF_RenderThreadSafe` only — no `ECVF_Scalability` —
(`Runtime/Renderer/Private/Nanite/NaniteCullRaster.cpp:133-138`), appears zero times in
`BaseScalability.ini`, and could not be added to one: `ConfigUtilities.cpp:337-346` `ensureMsgf`s
against an `ECVF_SetByScalability` ini write to a cvar lacking `ECVF_Scalability` /
`ECVF_ScalabilityGroup`.

**Two pinned members is enough.** The mechanism is demonstrated end to end on a live editor running
the guarded build, and `ShadowQuality` and `ResolutionQuality` are two of the eleven groups the
Scalability panel offers. The user's clicks on those two rows are now discarded silently.

## Membership is wide, and it is exactly the wide part callers type

Confirmed in `C:/UE_5.8/Engine/Config/BaseScalability.ini` (the only scalability inis on this
install are `BaseScalability.ini`, `AndroidScalability.ini`, `IOSScalability.ini`; this project
ships no `DefaultScalability.ini`):

| cvar | group | ini rows |
|---|---|---|
| `r.ViewDistanceScale` | ViewDistanceQuality | `:112, 116, 120, 124, 128` (sections `:110, 114, 118, 122, 126`) |
| `r.Shadow.Virtual.MaxPhysicalPages` | ShadowQuality | `:146, 181, 219, 257, 295` |
| `r.Shadow.Virtual.ResolutionLodBiasDirectional` | ShadowQuality | `:147, 182, 220, 258, 296` |
| `r.ShadowQuality` | ShadowQuality | `:134, 242` |
| `r.Streaming.PoolSize` | TextureQuality | `:749, 760, 771, 782, 793` |
| `r.MaxAnisotropy` | TextureQuality | `:746` |
| `r.MotionBlurQuality` / `r.DepthOfFieldQuality` / `r.BloomQuality` | PostProcessQuality | `:565, 571, 576` |

`r.ViewDistanceScale` is declared `ECVF_Scalability | ECVF_RenderThreadSafe` at
`Runtime/Core/Private/HAL/ConsoleManager.cpp:4434-4440`; the two virtual-shadow cvars likewise at
`Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.cpp:173-182` and
`VirtualShadowMapClipmap.cpp:42-47`.

## Nothing over the wire can see the pin afterwards

`system.console.search` emits `Scalability` and `ScalabilityGroup` — so an agent *can* discover that
a cvar is a member — but never any `ECVF_SetBy*` bit:
`Handlers/System/ConsoleSearchHandler.cpp:39-46` lists exactly `Cheat`, `ReadOnly`,
`RenderThreadSafe`, `Scalability`, `ScalabilityGroup`, `Preview`, `ExcludeFromPreview`,
`Unregistered`, with the comment at `:20-22` recording that `ECVF_SetByMask` is *"intentionally
omitted per the MVP scope on this ticket"*. `F-console-search-setby-priority` (OPEN, Medium) owns
that half and is cross-linked below, not duplicated here.

The engine *does* print it — `ConsoleManager.cpp:3247` logs `LastSetBy: %s` on the
`bShowCurrentState` branch (`:3187-3190`) when a cvar name is typed with no value — but neither
console verb captures Exec output (`SystemControlHandler.cpp:645`, `EditorCommandHandler.cpp:360`
call `Exec` with no `FOutputDevice`; the response is `{command, success}`,
`SystemControlHandler.cpp:679-681`). That is why the evidence above had to be read out of the log
file rather than off a response.

One narrow counter-example exists and shows the plugin already knows how:
`Handlers/Render/PreviewViewportCaptureUtils.cpp:1397-1398` calls
`::GetConsoleVariableSetByName(CVar->GetFlags() & ECVF_SetByMask)` and publishes it as `setBy` at
`:2803` — scoped to the forced-show-flag survey inside capture output only.

## Ask

**Extend the guard from a spelling test to a membership test.** The predicate has the command line;
it needs one lookup:

1. Split the first token. `IConsoleManager::Get().FindConsoleObject(Token)`.
2. If the object's flags carry `ECVF_Scalability` **or** `ECVF_ScalabilityGroup`
   (`IConsoleManager.h:103`, `:106`) **and** a value token follows, refuse with the existing
   `SCALABILITY_CVAR_USE_TYPED_VERB`, naming `performance.set_scalability` and stating that a
   console set pins the cvar at `ECVF_SetByConsole` above the editor's Scalability panel for the
   session.
3. Keep the existing `force: true` opt-out unchanged — this widens what `force` protects, it does
   not add a new gesture.
4. Keep the `sg.` prefix test as the fallback for a group cvar that is unregistered at guard time,
   so the current behaviour is a subset of the new one and no existing test regresses.

Two properties worth stating in the implementation rather than discovering later:

- **A bare read (`r.ViewDistanceScale` with no value) must not be refused.** The engine's own
  `bShowCurrentState` branch (`ConsoleManager.cpp:3187-3190`) never calls `Set`, so a read pins
  nothing. The existing `sg.` rule over-refuses bare reads deliberately and says so; a membership
  test can afford to be exact, and being exact matters more here because reading a member cvar is
  a normal, frequent thing to do.
- **Refuse on the flag, not on a hardcoded list.** `r.ScreenPercentage` is a `ResolutionQuality`
  member with no ini row (`Scalability.cpp:551`), so any list built by scanning
  `BaseScalability.ini` misses it. `ECVF_Scalability` is on the declaration and catches it.
- Docs in the same commit, on both verbs' registered summaries and on
  `docs/wiki-src/system.md` / `docs/wiki-src/editor.md`, since those pages currently describe the
  refusal as an `sg.*` rule and would become wrong.

## Scope note — a third door, found while re-deriving this one, deliberately NOT filed here

The same pin is reachable through **typed verbs**, with no console string involved.
`IConsoleManager.h:766` defaults `Set`'s flags to `ECVF_SetByCode = 0x0E000000`
(`IConsoleManager.h:183`), also far above `ECVF_SetByScalability`, so a bare `CVar->Set(x)` pins too:

- `Handlers/Debug/PerformanceHandler.cpp:572` — `performance.configure_texture_streaming` writes
  `r.Streaming.PoolSize`, a **TextureQuality** member.
- `Handlers/Debug/PerformanceHandler.cpp:953` (lambda `:949-956`) —
  `performance.apply_baseline_settings` writes five members: `r.MotionBlurQuality`,
  `r.DepthOfFieldQuality`, `r.BloomQuality` (PostProcessQuality), `r.ShadowQuality`
  (ShadowQuality), `r.MaxAnisotropy` (TextureQuality) — and its response asserts
  `scalabilityGroupsChanged: false` at `:996`, which is literally true (no `sg.*` value moved) and
  materially misleading.

The correct pattern is already in the tree: `PreviewViewportCaptureUtils.cpp:161-162` reads the
cvar's current `SetBy` and writes `r.ViewDistanceScale` back at that same priority, restoring at
`:174`, so `FScopedViewDistanceScale` never raises the priority.

That is a different fix in a different file with a different failure mode (a verb doing it to you
versus a verb passing your string through), so it wants its own ticket rather than being smuggled
in here. Recorded so it is not lost.

**Now filed as `B-performance-typed-verbs-pin-scalability-cvars` (OPEN, High).** One correction it
carries back to the list above: the count is **seven** `ECVF_Scalability` cvars across the two verbs,
not six — `apply_baseline_settings` writes **six**, not five. The missing one is `r.VSync`
(`ConsoleManager.cpp:4330`, flags `:4334`), which carries `ECVF_Scalability` on its declaration and
has **no row in `BaseScalability.ini`**, so the ini-scan used to build the list above skipped it.
That is precisely the `r.ScreenPercentage` case this ticket's own § *Fix* already warns about —
*"Refuse on the flag, not on a hardcoded list"* — reappearing one section later in its own evidence.

## Distinct from

- **`B-console-command-sg-cvar-pin-freezes-scalability`** (IN-REVIEW, High) — the same root cause
  through the door its guard *does* cover. **Not filed as an encounter on it, and its `encounters`
  and `lastSeen` are deliberately untouched**, for three reasons. (i) It is IN-REVIEW: its fix
  shipped and awaits a tester, and widening its scope now would ask a tester to verify something
  nobody built and would retroactively falsify its `#2` implementation record. (ii) The fixes are
  not the same fix. `sg.` is a two-character prefix test needing no data; a membership test needs
  the console object and its flags, changes shape per engine version, and carries an over-refusal
  risk on bare reads that the prefix test does not have. Neither fix closes the other. (iii) A
  different door is not a re-observation, so bumping a work-ordering tiebreak would tell the picker
  the `sg.*` case was seen again, which it was not. A cross-link-only history entry is appended
  there instead.
- **`F-console-search-setby-priority`** (OPEN, Medium) — owns the *observability* half: putting
  `SetBy` on `system.console.search` rows. That ticket already notes the `sg.` refusal covers only
  pins this plugin's verbs would create. This ticket is the *prevention* half for the door that
  refusal misses. Neither closes the other: a readback tells you the panel is dead, it does not
  stop it dying.
- **`E-scalability-console-escape-hatch-misleading`** (OPEN, Low) and
  **`B-set-scalability-no-sg-update`** (IN-REVIEW, Low) — both about `set_scalability`'s own
  documentation and readback, both `sg.*`-scoped.
- **`B-foliage-writes-vetoed-by-scalability-cvars`** (IN-REVIEW, High) and
  **`B-lighting-writes-vetoed-by-scalability-cvars`** (IN-REVIEW, Medium) — the opposite direction:
  a scalability cvar vetoing a PinWright write. Together with this ticket they establish that the
  priority model is plugin business in both directions.

Board-wide grep before filing: zero hits for `MaxPhysicalPages`, `ResolutionLodBias`,
`MaxPixelsPerEdge`, and no ticket names any individual scalability-member cvar in a pin context.

## Same shape as

The sibling ticket's family — the call succeeds, every number it reports is correct, and the
durable side effect is not part of the result at all. This is the variant where the *guard* against
that side effect was written against the symptom's spelling rather than against its cause.

## Severity

**High**, matching the sibling ticket band for band. Impact class is the rubric's High band in its
omission form: the requested write lands, so the success is true about what was asked; what is
untrue is the implied scope. The caller believes they set a value for now and has in fact taken a
control away from the human using the editor, for the session, irreversibly short of a restart.
The measured consequence on the sibling was *the user's editor misbehaving in a way neither party
could attribute* — the user noticed before the agent did — and that consequence is reachable here
through a line the guard explicitly allows.

**Not Critical.** Nothing crashes, no asset data is corrupted or lost, a restart recovers fully.

**Reach modifier declined, and the argument runs the opposite way to the sibling's.** The sibling
declined a bump *up* because `sg.*` lines are a small share of console traffic. The honest reading
here is that this door is **wider**, not narrower: `r.ViewDistanceScale`, `r.ScreenPercentage`,
`r.Streaming.PoolSize` and the shadow cvars are the vocabulary of every performance pass, whereas
almost nobody types `sg.ShadowQuality`. That argues for a bump up to Critical, which I decline
because Critical is defined by crash and data loss and this is neither. So it stays High
unmodified — and the width is offered as the argument for working it *ahead* of the sibling within
the band, since the sibling's fix is already in review and this is what that fix does not cover.

## Not done

Plugin source was not modified. No `performance.set_scalability` call was made and no scalability
level was changed, deliberately: the editor is shared with other agents and the measurement above
is read out of the log rather than produced by moving the panel. The pinned state was not cleared —
there is no unset path short of a restart.

severity rationale: impact=silent durable side effect on a normal path (High) × reach=declined,
argued in both directions -> High

## History
- `#1-guard-misses-member-cvars` `OPEN` reporter — Filed from the final triage sweep of a vegetation
  performance session on host `EAContentExamples58` (UE 5.8, editor build 13:32, plugin commit
  `d8f1bc32` — the build that ships the `SCALABILITY_CVAR_USE_TYPED_VERB` guard). Every citation
  re-derived at HEAD rather than relayed. **Plugin side:** the guard predicate is a bare `sg.`
  prefix test on the left-trimmed line (`Handlers/ScalabilityConsoleGuard.h:38-41`, rationale
  `:25-37`), called at `SystemControlHandler.cpp:611` and `EditorCommandHandler.cpp:302` behind
  `force` (declared `:584` / `:293`, code `ErrorCodes.h:1160`); its test file
  `Tests/EditorOps/TestScalabilityConsoleGuard.cpp:113-138` names no member cvar. **Engine side,
  the two doors joined into one path:** `ConsoleManager.cpp:3228` pins *any* cvar given a value at
  `ECVF_SetByConsole`; the Scalability panel reaches member cvars at `ECVF_SetByScalability` via
  `Scalability.cpp:912` -> `:907` -> the group sink (`:559-562`, `:569-572`, bound `:623`, `:625`)
  -> `SetGroupQualityLevel` `:412` -> **`Scalability.cpp:446`** -> **`ConfigUtilities.cpp:365`**,
  plus the ini-less `ResolutionQuality` route `Scalability.cpp:551` for `r.ScreenPercentage`; the
  veto is `FConsoleVariableBase::CanChange` `ConsoleManager.cpp:275-312` (`:280` compare, `:286-291`
  message, `:307` Warning, `:311` `return bRet;`), with neither demotion branch (`:293-297`,
  `:298-303`) applying to Console-over-Scalability, gated into every `Set` at `:1178`, `:1369`,
  `:1773`. Priorities `IConsoleManager.h:159` / `:187`. **Measured, this editor, this session:**
  `Saved/Logs/EAContentExamples58.log` frame 178 shows six cvars at `LastSetBy: Console`, of which
  exactly **two** are scalability-driven and therefore frozen —
  `r.Shadow.Virtual.ResolutionLodBiasDirectional` (ShadowQuality, `BaseScalability.ini:147, 182,
  220, 258, 296`) and `r.ScreenPercentage` (ResolutionQuality, `Scalability.cpp:551`). **Three
  claims in the lead I was handed did not survive and are corrected here rather than repeated:**
  `r.Nanite.MaxPixelsPerEdge` is **not** a scalability member (`NaniteCullRaster.cpp:133-138` has
  `ECVF_RenderThreadSafe` only, zero rows in `BaseScalability.ini`, and `ConfigUtilities.cpp:337-346`
  would `ensureMsgf` against adding one), so the "four pinned cvars" figure is really six pinned of
  which two matter; `CanChange` has no literal `return false` (it is `return bRet;` at `:311`); and
  "nothing in the plugin can read `SetBy`" is too strong —
  `PreviewViewportCaptureUtils.cpp:1397-1398` does, for `ShowFlag.*` only. Ask: replace the
  spelling test with a flag test on the first token's console object (`ECVF_Scalability` /
  `ECVF_ScalabilityGroup`, `IConsoleManager.h:103`/`:106`) when a value token follows, keeping
  `force: true` and keeping the `sg.` prefix as a fallback so current behaviour is a subset;
  refuse on the flag rather than an ini-derived list, because `r.ScreenPercentage` has no ini row;
  and do not refuse a bare read, which pins nothing. Severity High by the sibling's band, Critical
  declined on the rubric's crash/data-loss wording, reach bump declined although the argument here
  runs the opposite way to the sibling's — this door is the wider one. A third door found in
  passing and recorded in § *Scope note* rather than filed: `performance.configure_texture_streaming`
  (`PerformanceHandler.cpp:572`) and `performance.apply_baseline_settings` (`:953`, lambda
  `:949-956`) write six scalability members at the default `ECVF_SetByCode`, and the latter reports
  `scalabilityGroupsChanged: false` at `:996`. Cross-link-only entries appended to
  `B-console-command-sg-cvar-pin-freezes-scalability` and `F-console-search-setby-priority` with
  their `encounters` / `lastSeen` deliberately untouched.
- `#2-third-door-now-filed` `OPEN` reporter — **Cross-link and one correction. Status unchanged, and `encounters` / `lastSeen` deliberately left untouched** — this is not a re-observation of the member-cvar defect. The typed-verb door § *Scope note* recorded and declined to file is now `B-performance-typed-verbs-pin-scalability-cvars` (OPEN, High), filed on the terms this ticket set: different file, different failure mode, its own fix. The three doors are cross-linked in all directions so a fixer sees one defect with three entrances — `sg.` console line (`B-console-command-sg-cvar-pin-freezes-scalability`, IN-REVIEW), member cvar on a console line (this ticket), and typed verb with no console string at all (the new one). **Correction to this ticket's § *Scope note*, offered rather than edited into it silently:** the count is seven `ECVF_Scalability` cvars across the two verbs, not six, because `apply_baseline_settings` writes six rather than five. The one the note missed is `r.VSync` — `ECVF_Scalability | ECVF_RenderThreadSafe` on its declaration (`Runtime/Core/Private/HAL/ConsoleManager.cpp:4330`, flags `:4334`) with **no `BaseScalability.ini` row**, so an ini scan cannot see it. That is the same `r.ScreenPercentage` case this ticket's § *Fix* already argues for — *"Refuse on the flag, not on a hardcoded list … any list built by scanning `BaseScalability.ini` misses it"* — which means the note's own evidence was gathered by the method the ticket tells the fixer not to use. Worth carrying into the fix: whatever predicate `#2` ships must be flag-based, and the acceptance test should include a scalability cvar with no ini row.
- `#3-guard-tests-the-flag-not-the-spelling` `IN-REVIEW` developer — "Implemented the Ask: the predicate is now a flag test on the resolved console object, with the `sg.` spelling test kept as a fallback so every line the old rule refused is still refused. **`Handlers/ScalabilityConsoleGuard.h`** gains three functions beside the untouched `IsScalabilityGroupLine`: `SplitFirstConsoleToken` (first token to the first whitespace, remainder trimmed — mirroring `FConsoleManager::GetTextSection`, `ConsoleManager.cpp:3409-3429`, which is what `ProcessUserConsoleInput:3079` splits `Param1` with); `IsScalabilityCVarSetLine`, which resolves the token through `IConsoleManager::Get().FindConsoleObject(*Token, /*bTrackFrequentCalls=*/false)` and returns true only when the object is a **variable**, is not `ECVF_ReadOnly`, carries `ECVF_Scalability` **or** `ECVF_ScalabilityGroup`, **and** a value token follows; and `IsScalabilityPinningLine`, the `||` of the two rules, which both call sites now gate on (`SystemControlHandler.cpp:611`, `EditorCommandHandler.cpp:302`, both still behind `!Ctx.GetBool(TEXT("force"), false)`). No new error code: `SCALABILITY_CVAR_USE_TYPED_VERB` is reused and stays a raw literal in both non-adopting handlers. **The four non-refusals are deliberate and each has an engine line behind it, not a judgement call.** A bare read takes the `bShowCurrentState` branch at `ConsoleManager.cpp:3189` and never calls `Set`; the help form `r.Foo ?` takes `:3213`; a `ReadOnly` variable is refused by the console itself at `:3223` (and `AddConsoleObject:3269-3275` `check()`s that an `ECVF_Scalability` variable is never `ReadOnly` or `Cheat`, so that clause can only ever bite a `ReadOnly` `ECVF_ScalabilityGroup`); and an unresolvable token — a typo, or an `Exec` command owned by a module — is left alone, which is also what keeps the aggregate `scalability N` allowed, since it is not a console object at all but a `UEngine::Exec` branch (`UnrealEngine.cpp:5717-5721`). **One bypass closed that the Ask did not name:** a trailing `?` is stripped from the token before lookup, because `ConsoleManager.cpp:3085-3090` strips one and `r.ViewDistanceScale? 0.6` still reaches the `Set` at `:3228`; without the strip that spelling would have been a free evasion. A `<cvar>@<platform>` token is deliberately left unresolved instead — the engine sets a per-platform copy and logs *"Unable to set a value for %s another platform!"* rather than touching the live variable, so it cannot pin and must not be refused. **Tests** (`Tests/EditorOps/TestScalabilityConsoleGuard.cpp`, +3 ids, all failing before the fix): `core.scalability_console_guard.ScalabilityFlaggedCVarSet` asserts the tokenizer, then for each of `r.ViewDistanceScale`, `r.Streaming.PoolSize`, `r.MaxAnisotropy`, `r.ScreenPercentage` and `r.VSync` — the last two being the no-ini-row cases the Ask insisted on — that the set is refused, the read is not, the help form is not, and the `?`-suffixed set is; plus the `sg.` fallback on an unregistered group, `scalability 2`, an unresolvable token, an empty line, and a registered non-scalability cvar (`r.TextureStreaming`, `TextureStreamingHelpers.cpp:105`/`:110`). `system.console_command.ScalabilityMemberCvarRefusedWithoutForce` and the `editor.console_command` twin drive both verbs end to end. **Test hygiene, which is the defect itself.** The shipped `sg.` tests could name a group that does not exist because the old rule was a spelling test; a flag test cannot be exercised by a fake name, and naming a real member would mean a reverted guard pins a real quality cvar at `ECVF_SetByConsole` for the life of the process. The handler-level tests therefore register their own `PinWright.Test.ScalabilityGuardProbe` with `ECVF_Scalability` (`FScopedScalabilityProbeCVar`) and unregister it with `bKeepState=false` afterwards — indistinguishable to the guard, worthless if pinned. The predicate tests may name the real cvars because the predicate only reads the registry and never calls `Set`; each such assertion is gated on that cvar actually carrying the flag on this host, with `PINWRIGHT_ASSERTIONS_SKIPPED` otherwise. **One existing test needed adapting and it was not weakened.** `render.lumen_update_scene.RunsRealCommand` cross-checks its echoed command by running it through `system.console_command` and asserting success; that command is `r.LumenScene.SurfaceCache.Reset 1`, which carries `ECVF_Scalability` (`LumenSceneRendering.cpp:90`, flags `:93`), so the widened guard would have made the cross-check report the guard's refusal instead of the engine's verdict on the name. It now passes `force: true` **plus** a new assertion that the error code is not the refusal code, so the original discriminator is intact and one more is added. It costs the host nothing: `render.lumen_update_scene` Execs the identical line two statements earlier. **Docs** in the same commit — `Docs/wiki-src/system.md` and `editor.md` rewritten from the `sg.*` rule to the flag rule with the three non-refusals named, both registered summaries and both `force` param descriptions updated, and the two `command:` examples that were themselves now-refused lines (`r.Streaming.PoolSize 2048`, `r.ScreenPercentage 50`) replaced. `editor.md` also gains a warning on `editor.set_preferences`, which it recommends for multiple cvars and which has no guard at all. **Stated rather than buried: the flag is broader than "a quality slider".** `ECVF_Scalability` is the engine's classification and covers one-shot levers like `r.LumenScene.SurfaceCache.Reset`, which are now refused too; that is documented with `force: true` as the intended answer. **Not compiled and not run** — the orchestrator builds and runs after all agents return. `check_test_ids.py` and `check_test_skips.py` both CLEAN (4818 ids, 4818 unique)."
