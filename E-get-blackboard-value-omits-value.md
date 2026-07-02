---
id: E-get-blackboard-value-omits-value
title: "`ai.get_blackboard_value` reports only keyType + instanceSynced, not the stored default — the verb's name promises a value its readback never carries; document the property.get route"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [ai, blackboard, get_blackboard_value, set_blackboard_value, default-value, readback, round-trip, naming, docs]
---

# `ai.get_blackboard_value` is named like a value getter but returns no value

`ai.set_blackboard_value` is documented as "Set a default value on a Blackboard
key" and genuinely writes the per-key default (`DefaultValue` / `bDefaultValue`
on each `UBlackboardKeyType_*`). The obvious symmetric read-back — and the
verb whose name (`get_blackboard_value`) and method summary
("Get information about a Blackboard key") both promise the value — is
`ai.get_blackboard_value`. But that verb **never reads the stored default
value**. Its result carries only `{blackboardPath, keyName, keyType,
instanceSynced}`. There is no field for the value the sibling setter just wrote,
for any key type. So the natural "did my default land?" check against the
matching getter is structurally impossible, and the caller is pushed onto a
lower-level `property.get` of the raw `Keys[i].KeyType.DefaultValue` /
`bDefaultValue` array to confirm a value the `*_value` verb pair appears to own.

A thin payload from a verb literally named `get_blackboard_value`, sitting next
to `set_blackboard_value`, reads as "the value is missing / unset," not "this
verb does not report values" — a misleading-by-naming ergonomic gap, the same
shape as `E-get-ai-info-no-perception-readback` (a namespace's own `get_*`
verb can't confirm what its `set_*`/`configure_*` verb wrote), but on the
blackboard-key value surface rather than the controller surface.

## Replay evidence (this task, BB_StealthGuard, REPLAY-CONFIRMED at HEAD)

Set defaults on the simple keys, then queried them back via the named getter:

- `ai.set_blackboard_value {blackboardPath:/Game/AI/Blackboards/BB_StealthGuard, keyName:AlertLevel, value:"0.0"}` → `valueSet:true`
- `ai.set_blackboard_value {... keyName:GuardName, value:"Sentry"}` → `valueSet:true`
- `ai.get_blackboard_value {blackboardPath:/Game/AI/Blackboards/BB_StealthGuard, keyName:AlertLevel}` →
  `{"blackboardPath":"/Game/AI/Blackboards/BB_StealthGuard","keyName":"AlertLevel","keyType":"BlackboardKeyType_Float","instanceSynced":false}`
- `ai.get_blackboard_value {... keyName:HasLineOfSight}` →
  `{"blackboardPath":"/Game/AI/Blackboards/BB_StealthGuard","keyName":"HasLineOfSight","keyType":"BlackboardKeyType_Bool","instanceSynced":false}`
- `ai.get_blackboard_value {... keyName:GuardName}` →
  `{"blackboardPath":"/Game/AI/Blackboards/BB_StealthGuard","keyName":"GuardName","keyType":"BlackboardKeyType_Name","instanceSynced":false}`

In every case the response has **no value/default field at all** — yet the value
is genuinely stored: `property.get {objectPath:/Game/AI/Blackboards/BB_StealthGuard.BB_StealthGuard, propertyName:Keys}`
returns `Keys[6].KeyType.DefaultValue == "Sentry"` for GuardName (Int/Float/Bool
defaults that equal their type-default 0/false are simply omitted by the UE
serializer, which is itself why a value the getter *did* report would be useful).
So the data is present; only the named getter hides it.

## Root cause

`AIHandler.cpp` `ai.get_blackboard_value` (REGISTER at ~line 3221) loops
`BBData->Keys`, captures only `Key.bInstanceSynced` and
`Key.KeyType->GetClass()->GetName()`, and emits exactly four fields
(`blackboardPath`, `keyName`, `keyType`, `instanceSynced`). It never casts
`Key.KeyType` to the concrete `UBlackboardKeyType_*` to read the
`DefaultValue` / `bDefaultValue` — even though the sibling
`ai.set_blackboard_value` handler (~line 3102) does exactly that cast-ladder to
**write** those same fields. The getter just doesn't mirror the setter's
type switch on the read side.

## What it should do / how to fix (docs-first)

This is the same readback-thinness shape as `E-get-ai-info-no-perception-readback`,
which was resolved **docs-first** (`#5`: the `ai.md` overlay note shipped; the
`get_ai_info` code enrichment was carved out as the optional follow-up). Take the
same scope here — the cheap, near-zero-risk win is to document the limitation, not
to expand the version-gated key-type cast-ladder in a 3,400-line handler for a
rare edge path that already has a working route:

- Add a `### ai.get_blackboard_value` H3 to `docs/wiki-src/ai.md` (surfaces when an
  agent calls `call("ai.get_blackboard_value")`) stating plainly that the verb
  reports only `{blackboardPath, keyName, keyType, instanceSynced}` and never the
  stored default. A thin payload means "this verb does not report the value," not
  "the default is unset" — do not retry it to confirm a `set_blackboard_value`
  write landed. Point at the supported read-back: `property.get
  {objectPath:"<bb>.<bb>", propertyName:"Keys"}` → index `Keys[i].KeyType.DefaultValue`
  (scalar / Name / String) or `Keys[i].KeyType.bDefaultValue` (Bool). Note the
  UE serializer omits values equal to their type-default (Int/Float `0`, Bool
  `false`), so an absent `DefaultValue` there means "still the type default," not
  "unreadable."

**Optional ergonomic follow-up (separate, larger — not this ticket):** enrich the
`get_blackboard_value` handler to mirror `set_blackboard_value`'s cast-ladder on
the read side — `Cast<>` to each `UBlackboardKeyType_Bool/Int/Float/Vector/Rotator/Name/String`
under the same `UE_VERSION_NEWER_THAN_OR_EQUAL(5,5,0)` `#if` and emit the stored
`DefaultValue` / `bDefaultValue` (plus `bUseDefaultValue` for Vector/Rotator) as a
`value` field, with Object/Class/Enum keys emitting their `BaseClass` / enum type —
so a single `ai` verb closes the set→get loop. Documenting the current behavior is
the cheap win; the enrichment is the nice-to-have.

**Workaround:** read the per-key default via
`property.get {objectPath:"<bb>.<bb>", propertyName:"Keys"}` and index
`Keys[i].KeyType.DefaultValue` (scalar/Name/String) or `.bDefaultValue` (Bool);
type-default values (Int/Float 0, Bool false) are omitted by the serializer.

## History
- `#1-initial-repro` `OPEN` reporter — Struggle audit of the BB_StealthGuard stealth-guard blackboard task (seed `ai.create_blackboard`; the friction is in a neighbor, `ai.get_blackboard_value`). Task succeeded (all calls `ok:true`); friction is a misleading-by-naming readback gap. REPLAY-CONFIRMED at HEAD: `ai.set_blackboard_value` writes per-key defaults (`valueSet:true`), but the symmetric, identically-named `ai.get_blackboard_value` returns only `{blackboardPath, keyName, keyType, instanceSynced}` and NO value/default field for any key type — verbatim `{"blackboardPath":"/Game/AI/Blackboards/BB_StealthGuard","keyName":"AlertLevel","keyType":"BlackboardKeyType_Float","instanceSynced":false}`. The value IS stored (proven via `property.get {propertyName:Keys}` → `Keys[6].KeyType.DefaultValue=="Sentry"`), so the named getter hides genuinely-set state, forcing the attempt agent onto the raw `Keys[].KeyType.DefaultValue`/`bDefaultValue` property route to confirm the "default 0.0/false" success check (friction note verbatim: *"ai.get_blackboard_value reports keyType + instanceSynced but NOT the stored default value, so the success check's 'default 0.0/false' can't be confirmed from that verb — had to read Keys[i].KeyType.DefaultValue/bDefaultValue via property.get"*). Root cause: `AIHandler.cpp` `ai.get_blackboard_value` (~3221) emits only 4 fields and never casts `Key.KeyType` to the concrete `UBlackboardKeyType_*` to read the default, even though the sibling `ai.set_blackboard_value` (~3102) uses exactly that cast-ladder to write it. Same readback-thinness shape as `E-get-ai-info-no-perception-readback`, on the blackboard-key value surface. (The task's second reported friction — `ai.get_ai_info` controllerPath returning only `{controllerClass}` and never the assigned blackboard — is already tracked by `E-get-ai-info-no-perception-readback` `#2`/`#6`; not re-filed here.)
- `#2-reword-and-docs-fix` `IN-REVIEW` developer — REWORDED to a docs-first scope and implemented it. The defect is real and present at HEAD (verified: `ai.get_blackboard_value` at `AIHandler.cpp:3221` emits exactly `{blackboardPath, keyName, keyType, instanceSynced}` at `AIHandler.cpp:3276-3280` and never casts `Key.KeyType` to read the default, while the sibling `ai.set_blackboard_value` at `AIHandler.cpp:3102` runs the full Bool/Int/Float/Vector/Rotator/Name/String cast-ladder at `3148-3184` to WRITE it). But this is the SAME readback-thinness shape as `E-get-ai-info-no-perception-readback`, which the board resolved DOCS-FIRST (`#5`: shipped the `ai.md` overlay note, carved out the `get_ai_info` code enrichment as the optional follow-up) — and the stored default IS already readable today via `property.get {propertyName:Keys}` → `Keys[i].KeyType.DefaultValue`/`bDefaultValue` (proven in `#1`: `Keys[6].KeyType.DefaultValue=="Sentry"`). So the consistent scope is the docs steer, not a version-gated cast-ladder rewrite for a rare edge path that already has a working route. Severity kept Low: base impact is the Medium "readback omits a field and forces a fallback," but the reach modifier bumps it down one for a rare edge path (blackboard default set/get is not an every-session method). Changes: added a `### ai.get_blackboard_value` H3 to `Docs/wiki-src/ai.md` (surfaces when an agent calls `call("ai.get_blackboard_value")`) stating the verb reports only `{blackboardPath, keyName, keyType, instanceSynced}` and NO default, that a thin payload means "verb doesn't report the value" (don't retry to confirm a write), and pointing at the `property.get {objectPath:"<bb>.<bb>", propertyName:"Keys"}` → `Keys[i].KeyType.DefaultValue` (scalar/Name/String) / `bDefaultValue` (Bool) route, with the serializer-omits-type-defaults caveat. Demoted the cast-ladder getter enrichment to a carved-out optional follow-up in the body. Regression test added in `Source/PinWright/Private/Tests/Infra/TestWikiHandler.cpp` (`FWikiHandlerGetBlackboardValueDocumentsThinReadbackTest`, `...wiki_handler.MethodPage.GetBlackboardValueDocumentsThinReadback`): drives production `WikiHandler::RenderPage("ai.get_blackboard_value")` and asserts the rendered page contains the overlay-exclusive markers `property.get`, `DefaultValue`, and `bDefaultValue` (none appear in the auto summary "Get information about a Blackboard key" or the `blackboardPath`/`keyName` param list — verified by grep of the registration block) — it fails iff the overlay H3 is reverted. Did not compile/run (later phase).
