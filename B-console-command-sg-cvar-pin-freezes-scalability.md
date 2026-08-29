---
id: B-console-command-sg-cvar-pin-freezes-scalability
title: "`system.console_command` / `editor.console_command` handed an `sg.*` line pin that group at ECVF_SetByConsole for the rest of the editor session, which permanently outranks the editor's own Scalability panel (ECVF_SetByScalability) — measured: seven groups became unclickable in the UI and it read as the editor force-resetting quality — while both verbs answer `success: true` with nothing about the side effect, no PinWright verb can read a CVar's SetBy priority back, and the correct typed verb `performance.set_scalability` was one call away"
status: IN-REVIEW
severity: High
category: bug
tags: [system, editor, console-command, console_command, scalability, sg-cvars, cvar-priority, ecvf-setbyconsole, ecvf-setbyscalability, silent-side-effect, session-state, user-facing, editor-degradation, set-scalability, typed-verb, readback]
encounters: 1
lastSeen: 2026-08-29T20:10:00+03:00
---

# One console line takes the Scalability panel away from the user until they restart

`system.console_command {command: "sg.FoliageQuality 3"}` does what it was asked. It also, and
undisclosed, raises that CVar's setter priority to `ECVF_SetByConsole` — the highest of the fifteen —
where it stays for the life of the process. The editor's own **Settings → Engine Scalability Settings**
panel writes at `ECVF_SetByScalability`, the second-lowest. From that moment every click the *user*
makes on that group is silently discarded.

Measured on this host, 2026-08-29: seven groups were set this way by an agent, and all seven became
unchangeable in the UI for the rest of the session while the five untouched groups still responded.
The mixed state is what made it read as the editor "force resetting" quality on its own. The user
noticed it before the agent did.

## Mechanism, re-derived at HEAD `6d0e91a3`

**The two verbs do not look at the string.** `system.console_command` registers at
`Plugins/PinWright/Source/PinWright/Private/Handlers/System/SystemControlHandler.cpp:580` (body
`:584-677`) with exactly one parameter, `command` (`:581-583`), and executes it verbatim at `:631`
(`bOk = GEditor->Exec(TargetWorld, *Cmd);`) with a fallback loop over world contexts at `:643`.
`editor.console_command` registers at `Handlers/Editor/EditorCommandHandler.cpp:288` (body `:293-370`),
params `command` (`:289-292`) + `world` (default `editor`), Exec at `:347`. **Neither file contains the
substring `sg.` or `Scalability` anywhere** — no prefix inspection, no warning, no refusal. Both
descriptions even use a CVar as the worked example (`"r.Streaming.PoolSize 2048"`,
`"r.ScreenPercentage 50"`).

**The priorities, and that Console beats Scalability.**
`C:/UE_5.8/Engine/Source/Runtime/Core/Public/HAL/IConsoleManager.h:152` states the ordering is
weak-to-strong; `:159` is `ECVF_SetByScalability = 0x02000000` and `:187` is
`ECVF_SetByConsole = 0x10000000`, the maximum. Scalability is second-lowest of fifteen, above only
`ECVF_SetByConstructor`.

**The lower write is discarded, and only warned about later.**
`C:/UE_5.8/Engine/Source/Runtime/Core/Private/HAL/ConsoleManager.cpp:275-312`,
`FConsoleVariableBase::CanChange`: `:277-280` computes `bRet = NewPri >= OldPri`; `:286-291` builds the
message *"Setting the console variable '%s' with 'SetBy%s' was ignored as it is lower priority than
the previous 'SetBy%s'. Value remains '%s'"*; `:304-308` logs it `Warning` on `LogConsoleManager` in
the default branch; `:311` returns false. Every `Set` path guards on it and no-ops
(`:1178-1181`, `:1369-1373`, and the same guard at `:1517`, `:1562`, `:1773`, `:1938`). Two verbosity
demotions exist and neither applies here: `:293-297` (old priority is an ini/commandline/hotfix source)
and `:298-303` (exactly Scalability-over-DeviceProfile). A Console pin hits the `else` and warns.

**The editor UI is on the losing side.**
`C:/UE_5.8/Engine/Source/Editor/UnrealEd/Private/SScalabilitySettings.cpp:72` —
`Scalability::SetQualityLevels(CachedQualityLevels);`, immediately after the per-group `sg.*` dispatch
at `:60-70` — and again at `:87`, `:113`, `:212`, `:262`. `Scalability::SetQualityLevels`
(`Runtime/Engine/Private/Scalability.cpp:912`, per-group dispatch `:939-950`) writes each group in
`SetQualityLevelCVar` (`:881`) at `:907`:
`CVarToEdit.AsVariable()->Set(DesiredValue, ECVF_SetByScalability);`

So the user's own UI is permanently outranked, and the warning it produces is attributed to *the
click*, not to the console line that caused it — hours earlier, from a different actor.

## The plugin already has the correct verb, and it says so in its own summary

`performance.set_scalability` (`Handlers/Debug/PerformanceHandler.cpp:396`, body `:400-455`) writes at
`:403-406` through `Scalability::SetQualityLevels` — i.e. at exactly the priority the editor UI uses,
which is why it *cannot* create this pin — and reads the groups back at `:432-445`, reporting
`requestedLevel` / `appliedGroups` / `requestedLevelApplied` (`:447-453`). Its registered summary at
`:396` already spells the whole priority model out, including the sentence *"A group already pinned
higher (by a device profile, a config, or a prior console `'sg.<Group> N'` ... at ECVF_SetByConsole) is
therefore NOT overwritten"*.

**The plugin therefore knows the hazard, documents it on the verb that suffers from it, and says
nothing on the two verbs that cause it.** That asymmetry is the defect.

## Nothing in the RPC surface can observe the pin

`system.console.search` returns a `flags` array per row, but `FlagsToStrings`
(`Handlers/System/ConsoleSearchHandler.cpp:23-48`) pushes exactly eight behaviour flags — `Cheat`,
`ReadOnly`, `RenderThreadSafe`, `Scalability`, `ScalabilityGroup`, `Preview`, `ExcludeFromPreview`,
`Unregistered` — and **none of the `ECVF_SetBy*` bits**; the `ECVF_SetByMask` portion
(`IConsoleManager.h:150`) is never surfaced. `currentValue` (`:167`) shows the value, never who set it.
So after the fact there is no verb that can tell a caller — or the next agent in a shared editor — that
a group is pinned, or by what. The only evidence is a `LogConsoleManager: Warning` emitted at the
moment of a *later, failed* write.

## What this costs, and why it is not just an agent's problem

The measured session: seven `sg.*` groups pinned by console, five not. The user's Scalability panel
accepted changes on five groups and silently reverted on seven. From the user's side that is the
application misbehaving — a mixed, self-resetting quality state with no error dialog and no visible
cause. Recovery is an editor restart; the priority does not decay and there is no "unset" path.

Two board tickets currently point callers straight at this:

- **`E-scalability-console-escape-hatch-misleading`** (OPEN, Low) is correct that the aggregate
  `scalability N` routes through `SetQualityLevels` at `ECVF_SetByScalability` and cannot beat a pin —
  and it therefore recommends *"a per-CVar `sg.<Group> N` directly (ECVF_SetByConsole)"* as **"the real
  escape hatch"**. That recommendation is exactly the action documented here. It is not wrong about the
  mechanism; it is silent about the cost, so following it fixes one session and breaks the user's
  Scalability panel for the rest of it.
- **`B-set-scalability-no-sg-update`**'s Workaround prose recommends
  `system.console_command {command: "scalability <N>"}`. That aggregate form is *harmless* on this axis
  (it writes at Scalability priority), so it does not cause the pin — but the sibling ticket already
  notes the prose is wrong about it working at all.

Whoever fixes this should reconcile all three, because two of them currently disagree about whether the
per-CVar console form is advice or a hazard.

## Ask

**Detect a leading `sg.` token (case-insensitive) on `command` in both verbs and stop treating it as an
ordinary line.** In preference order:

1. **Refuse**, with a code such as `SCALABILITY_CVAR_USE_TYPED_VERB`, naming
   `performance.set_scalability` and stating in one sentence that a console set pins the group at
   `ECVF_SetByConsole` above the editor's own Scalability panel for the remainder of the session. This
   is defensible precisely because a correct typed verb exists — the house rule the board already
   applies elsewhere.
2. **With a `force: true` opt-out**, so the deliberate escape hatch
   `E-scalability-console-escape-hatch-misleading` asks for stays reachable, but only by someone who
   typed the word. That single flag serves both tickets: this one stops the accident, that one keeps
   the capability.
3. **If refusal is judged too strong**, then proceed and attach a `warnings[]` entry saying the same
   thing. What is not acceptable is the present `{success: true}` / `{success: true, consumed: true}`
   with no mention, because the side effect outlives the call and is not the caller's to spend.
4. **Docs, in the same commit**: one line on both verbs' registered summaries and on their wiki pages
   pointing `sg.*` at `performance.set_scalability`.

Secondary, separable, and worth its own consideration: **add the `SetBy` priority to
`system.console.search` rows** (`ConsoleSearchHandler.cpp:23-48` already has the `IConsoleObject` in
hand; `GetFlags() & ECVF_SetByMask` plus the engine's own `GetConsoleVariableSetByName` is the whole
implementation). Without it, "is this CVar pinned, and by whom" is unanswerable over the wire — which
is what let the pin go unnoticed for a session. That belongs on `F-console-batch-get-cvar-values` or as
its own ticket rather than being smuggled in here.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as* — `B-ground-probe-hits-hull-not-render`,
`B-niagara-validate-green-while-component-inactive`, `B-mrq-render-result-omits-bitrate-and-size`: the
call succeeds, every number it reports is correct, and the output is wrong because the deciding number
was never reported. This is the variant where the unreported quantity is not part of the *result* at
all but of the *side effect*: the CVar's new setter priority. `editor.console_command`'s response even
carries an honest caution — *"This confirms the command was recognised, not that its effect succeeded —
verify the effect with a read-back verb"* (`EditorCommandHandler.cpp:361-366`) — which is about the
effect the caller wanted, and says nothing about the one they did not.

Also the same family as `B-foliage-writes-vetoed-by-scalability-cvars` and
`B-lighting-writes-vetoed-by-scalability-cvars`, which the board already treats as plugin tickets:
those are "a scalability CVar silently vetoes a PinWright write". This is the mirror image — "a
PinWright call silently pins a scalability CVar" — and it establishes that the priority model is
plugin business in both directions.

## Severity

**High.** Impact class is the rubric's High band: *silent false-success ... the caller trusts a result
that is a lie and builds on it*, in the form where the lie is by omission of a durable side effect. The
requested write does land, so the success is true about what was asked; what is untrue is the implied
scope — the caller believes they set a value for now, and they have taken a control away from the human
using the application, for the session, irreversibly short of a restart. The measured consequence was
not an agent's number going wrong but **the user's editor misbehaving in a way neither party could
attribute**, which is the strongest reading of "builds on it" this rubric has.

**Not Critical.** Nothing crashes, and no asset data is corrupted or lost — the rubric's Critical band
is explicit and this does not reach it. A restart recovers fully.

**Reach modifier declined, with the argument.** `system.console_command` / `editor.console_command` are
the generic escape hatch and appear in a large fraction of sessions, which by the rubric argues for a
bump up. Declined because `sg.*` lines are a small share of console traffic, so the *reach of the
failure* is much narrower than the reach of the method — the same distinction
`E-python-get-editor-property-returns-live-view` drew when it declined its bump. High stands
unmodified; it is already the correct band on impact alone.

## History
- `#1-console-sg-pin-outranks-the-editor-ui` `OPEN` reporter — Observed live on host
  `EAContentExamples58` (UE 5.8) during a vegetation session: seven `sg.*` groups set via
  `system.console_command` became unchangeable in the editor's Scalability panel for the remainder of
  the session while five untouched groups still responded, presenting to the **user** as the editor
  force-resetting quality; recovered only by restart. Chain re-derived at HEAD `6d0e91a3` rather than
  relayed. PinWright side: `system.console_command` (`SystemControlHandler.cpp:580`, body `:584-677`,
  sole param `command` at `:581-583`, Exec `:631`, fallback `:643`, response `:665-675` = `{command,
  success}`) and `editor.console_command` (`EditorCommandHandler.cpp:288`, body `:293-370`, Exec `:347`,
  response `:356-368` with `consumed:true` and a generic effect caution at `:361-366`) both execute the
  string verbatim; **neither file contains `sg.` or `Scalability` at all**. Engine side:
  `ECVF_SetByScalability = 0x02000000` (`IConsoleManager.h:159`) vs `ECVF_SetByConsole = 0x10000000`
  (`:187`), ordering stated `:152`; `FConsoleVariableBase::CanChange` (`ConsoleManager.cpp:275-312`)
  compares at `:277-280`, builds the "was ignored as it is lower priority" message at `:286-291`, logs
  it `Warning` on `LogConsoleManager` at `:304-308` (neither demotion at `:293-297` / `:298-303`
  applies) and returns false at `:311`, with every `Set` guarding on it (`:1178-1181`, `:1369-1373`,
  `:1517`, `:1562`, `:1773`, `:1938`). The editor UI is on the losing side:
  `SScalabilitySettings.cpp:72` calls `Scalability::SetQualityLevels`, which writes each group at
  `ECVF_SetByScalability` in `SetQualityLevelCVar` (`Scalability.cpp:907`, dispatch `:939-950`). The
  correct typed verb already exists and cannot cause the pin — `performance.set_scalability`
  (`PerformanceHandler.cpp:396`, write `:403-406`, readback `:432-445`) uses the same
  `SetQualityLevels` path, and its own summary at `:396` documents the whole priority model. Also
  verified: **no verb can observe the pin** — `system.console.search`'s `flags`
  (`ConsoleSearchHandler.cpp:23-48`) emits eight behaviour flags and none of the `ECVF_SetBy*` bits, so
  `ECVF_SetByMask` is unreachable over the wire. **Board contradiction flagged, not resolved here:**
  `E-scalability-console-escape-hatch-misleading` (OPEN, Low) recommends the per-CVar `sg.<Group> N`
  console form as "the real escape hatch" — mechanically correct, silent about this cost — and
  `B-set-scalability-no-sg-update`'s Workaround prose recommends the aggregate `scalability <N>`, which
  is harmless on this axis; whoever fixes this must reconcile all three. Ask: detect a leading `sg.`
  token in both verbs and refuse with a code naming `performance.set_scalability`, gated by a
  `force: true` opt-out that preserves the deliberate escape hatch the sibling ticket wants; failing
  that, a `warnings[]` entry; plus a docs line on both summaries. Severity High on silent-side-effect
  impact, argued as the omission variant of the High band because the durable consequence lands on the
  user's editor rather than on the agent's numbers; Critical declined (no crash, no data loss, restart
  recovers); reach bump declined because `sg.*` is a small share of console traffic even though the
  verbs themselves are near-universal.
- `#2-refuse-sg-console-sets-with-force-optout` `IN-REVIEW` developer — "Implemented the Ask at
  preference 1+2: both verbs now REFUSE a leading `sg.` token and steer to the typed verb, with a
  `force: true` opt-out. New shared header `Source/PinWright/Private/Handlers/ScalabilityConsoleGuard.h`
  holds the whole rule in one place so the two verbs cannot drift:
  `IsScalabilityGroupLine(CommandLine)` is `CommandLine.TrimStart().StartsWith(TEXT('sg.'),
  ESearchCase::IgnoreCase)` — because `sg.` contains no whitespace, a prefix test on the left-trimmed
  line IS the first-token test, so `r.Foo sg.Bar` (the prefix in a value or a quoted string) does NOT
  match and the aggregate `scalability N` does NOT match, exactly as the ticket requires; and
  `MakeScalabilityTypedVerbRefusal(CommandLine)` builds the one-sentence message naming
  `performance.set_scalability`, both priorities, and the `force:true` escape. Call sites:
  `Handlers/System/SystemControlHandler.cpp` `system.console_command` refuses right after its
  empty-command check (before the `GEditor` guard), and `Handlers/Editor/EditorCommandHandler.cpp`
  `editor.console_command` hoists its `command` read above the `GEditor` guard and refuses there — the
  guard runs ahead of the environment check on purpose, because whether a line pins a group is a fact
  about the string, and it makes the refusal deterministic in a test host with no editor. Both emit
  `SCALABILITY_CVAR_USE_TYPED_VERB`, now registered as `ERR_SCALABILITY_CVAR_USE_TYPED_VERB` in
  `Handlers/ErrorCodes.h`; the call sites deliberately keep the RAW literal because neither file cites
  `ErrorCodes::ERR_` today and the first citation would flip the whole file to 'adopting' and fail
  its existing literals (TestErrorCodeRegistry test 2) — the header entry carries that note. Both verbs
  declare a new optional `force` boolean param, so the dispatcher's UNKNOWN_PARAMS gate accepts it and
  `PinWright.infra.declared_params.HandlersOnlyReadDeclaredParams` stays green. Docs in the same
  change: one paragraph appended to both registered summaries and to `docs/wiki-src/system.md`
  §`system.console_command` and `docs/wiki-src/editor.md` §`editor.console_command`, each pointing
  `sg.*` at `performance.set_scalability` and stating that `scalability N` is unaffected; editor.md's
  standing claim that 'nothing refuses a dangerous one' was corrected to name this one refusal.
  Regression test `Source/PinWright/Private/Tests/EditorOps/TestScalabilityConsoleGuard.cpp`, three
  ids: `PinWright.core.scalability_console_guard.FirstTokenPrefixOnly` (predicate — `sg.X 3`, `SG.X 3`,
  leading whitespace and a bare `sg.X` all match; `scalability 2`, bare `scalability`, `r.Foo sg.Bar`,
  a trailing `sg.` token, `sgfoo 1` and the empty line all do not; refusal text names the typed verb
  and both priorities) plus `PinWright.system.console_command.ScalabilityCvarRefusedWithoutForce` and
  `PinWright.editor.console_command.ScalabilityCvarRefusedWithoutForce`, which drive the real verbs
  through one shared assertion: refused with the code and a message naming
  `performance.set_scalability`, `force: true` NOT stopped by the guard, and the aggregate command not
  refused. **No test can leak a pin, which is load-bearing rather than fastidious:** the refusal cases
  use a group name that does not exist (`sg.__PinWright_NoSuchGroup__ 3`), so if the guard is ever
  reverted the line reaches Exec, finds no console object and sets nothing; the `force` case passes a
  REAL group with NO value, which ProcessUserConsoleInput answers on its `bShowCurrentState` branch
  without calling Set; and the not-refused case passes bare `scalability` (usage print) rather than
  `scalability 2`, which WOULD write all eleven groups and SaveState to the host's editor ini — the
  `scalability 2` spelling is asserted on the predicate instead, where it costs nothing.
  `check_test_ids.py` re-run after adding them: 4784 ids, CLEAN, no dot-prefix collision. NOT
  implemented, per the ticket's own scope line: the `SetBy` priority on `system.console.search` rows —
  filed as `F-console-search-setby-priority` instead. Board contradiction reconciled in the same pass:
  history appended to `E-scalability-console-escape-hatch-misleading` and `B-set-scalability-no-sg-update`
  (statuses unchanged) recording that the per-CVar console form is a hazard with a real cost, and that
  `force: true` is now where the deliberate escape hatch lives. Not compiled or run — the orchestrator
  builds and runs the suite immediately after this change."
