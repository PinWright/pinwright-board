---
id: E-property-get-override-state-ignores-boverride-bit
title: "property.get/list isOverridden is a pure value-vs-default compare — reads false for override-flag-gated FPostProcessSettings fields whose value equals the struct default, despite the bOverride_* bit being set, so verifiers spam extra raw-bit read-backs"
status: OPEN
severity: Low
category: ergonomic
tags: [property, property-get, property-list, isOverridden, includeOverrideState, post-process, boverride, readback, verification, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `isOverridden` ignores the `bOverride_*` companion bit, so override-flag-gated PPV fields whose value equals the struct default report `isOverridden:false` even though they genuinely blend

`property.get` / `property.list` compute the `includeOverrideState` field
`isOverridden` as a **pure value-vs-class-default byte compare**:

```cpp
// UtilityPropertyHandler.cpp:1511 (get) and :1693 (list)
const bool bIsOverridden = !Property->Identical(CurrentValuePtr, DefaultValuePtr, PPF_None);
```

For `FPostProcessSettings` (and any UE struct that pairs each value field with a
`bOverride_<Field>` boolean gate) this is the wrong signal. Whether a PPV value
actually blends at composition time is governed by the **`bOverride_<Field>`
companion bit**, not by whether the value differs from the struct default. When
a caller deliberately sets a field to a value that *equals* the struct/cinematic
default and flips its `bOverride_` bit, the field WILL blend — but `Identical`
returns true, so `isOverridden` reports **`false`**. The override-state field
therefore contradicts the actual override status for exactly the fields the
typed `post_process.set_*` setters (see DONE `F-post-process-typed-setters`) are
designed to drive. A verifier who trusts `isOverridden` to confirm "did my PPV
write take" is misled and must fall back to reading the raw `bOverride_*` leaf
bits one by one.

This is the **same value-vs-default comparison limitation** as the DONE bug
`B-asset-dump-properties-spurious-override-on-bp-internal-bools` — but the
opposite symptom (false-negative, not false-positive) on a different surface
(`property.get`/`property.list` `isOverridden`, not `asset.dump`
`is_overridden_locally`) and a different field class (override-flag-gated PPV
struct members, not UMG compiler-managed bools). The asset-dump fix special-cased
three named UUserWidget flags; it did not generalize override-state to consult
`bOverride_*` companions, so this surface still reports the misleading value.

## Evidence (this task — namespace `post_process`, golden-hour grade, outcome clean)

The story's final step required a read-back to "confirm the warm grade, bloom,
Lumen toggles, and motion blur all took effect." The agent applied all six typed
setters successfully, then verified via ~19 `property.get` calls. Friction note
(verbatim):

> "property.get with includeOverrideState reported isOverridden:false for
> BloomMethod/BloomThreshold/DynamicGlobalIlluminationMethod/ReflectionMethod
> despite their values being applied, so I had to do extra read-backs of the raw
> bOverride_* bits (all genuinely true) to confirm the values actually blend —
> the isOverridden field appears not to track the bOverride bit for these PPV
> fields, a minor verification-confidence gap."

The four flagged fields are exactly the ones whose applied value matches the
struct default: `BloomThreshold` → -1 (struct default -1), `BloomMethod` →
`BM_SOG`/Standard (struct default), `DynamicGlobalIlluminationMethod` → `Lumen`
(project/struct default), `ReflectionMethod` → `Lumen` (project/struct default).
For each, `Identical(current, default)` is true → `isOverridden:false`, even
though the agent confirmed `bOverride_BloomMethod` / `bOverride_BloomThreshold` /
`bOverride_DynamicGlobalIlluminationMethod` are all genuinely `true` (the value
blends). The recovery cost was a batch of extra raw `bOverride_*` read-backs
(call log: `property.get bOverride_BloomMethod->true`,
`bOverride_BloomThreshold->true`, `bOverride_DynamicGlobalIlluminationMethod->true`)
that the override-state field should have made unnecessary. All calls succeeded —
pure PROCESS overhead; the seed landed `clean` and the judge filed nothing
(`filed_id` empty).

## What it should do

When the property being inspected sits in a struct that carries a matching
`bOverride_<FieldName>` companion bool, `isOverridden` (and ideally a distinct
`overrideFlagSet` field) should reflect the **companion bit**, not — or in
addition to — the value-vs-default compare. Concretely, in the override-state
emitters at `UtilityPropertyHandler.cpp:1511` and `:1693`: detect a sibling
`bOverride_<Name>` `FBoolProperty` on the same container and, when present, set
`isOverridden` from that bit (an explicitly-flagged value blends regardless of
whether it equals the default). A non-breaking alternative is to keep
`isOverridden` as the value compare but add an `overrideFlagSet` boolean
whenever a `bOverride_*` companion exists, so a verifier has a single field that
answers "will this blend?" without a second read of the raw bit.

**Workaround:** to confirm an override-flag-gated PPV write actually blends, read
the raw companion bit directly — `property.get { propertyName:
"Settings.bOverride_<Field>" }` — rather than trusting `isOverridden`. Each
gated field needs its own bit read.

## Docs angle

The `docs/wiki-src/property.md` overlay documents `isOverridden` (lines ~75/86)
purely as "include `hasDefaultValue` / `isOverridden`" with no caveat. It should
state that for struct fields gated by a `bOverride_<Field>` companion (notably
all of `FPostProcessSettings`), `isOverridden` is a value-vs-default compare and
will read `false` for an applied-but-equals-default value; point readers at the
`Settings.bOverride_<Field>` leaf (or the proposed `overrideFlagSet` field) to
confirm a PPV value actually blends. Cross-link `post_process.md` for the typed
`post_process.set_*` setters whose writes this affects.

## Not a duplicate of

- `B-asset-dump-properties-spurious-override-on-bp-internal-bools` (DONE) — same
  value-vs-default root limitation, but opposite symptom (false-positive) on
  `asset.dump` `is_overridden_locally` for named UMG compiler flags; its fix
  denylisted three flags and did not teach override-state to honor `bOverride_*`.
- `E-lighting-set-ao-exposure-no-echo` (OPEN) — argues the PPV *setters* should
  echo applied values to avoid a `property.get` round-trip; this ticket is about
  the `property.get`/`list` override-state field itself reporting the wrong thing
  once you do read back.
- `F-post-process-typed-setters` (DONE) — added the setters used here; explicitly
  deferred "reading current values back" as out of scope. This is that readback
  surface giving a misleading override signal.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the golden-hour `post_process` grade task (37 calls, outcome clean, judge filed nothing — `filed_id` empty). PROCESS finding: `property.get`/`property.list` `isOverridden` is a pure `!Property->Identical(current, default)` compare (`UtilityPropertyHandler.cpp:1511` get, `:1693` list) that ignores the `bOverride_<Field>` companion bit gating `FPostProcessSettings`. The task set BloomThreshold=-1 / BloomMethod=BM_SOG / DynamicGlobalIlluminationMethod=Lumen / ReflectionMethod=Lumen — all equal to the struct/project default — so each read back `isOverridden:false` despite the genuinely-true `bOverride_*` bits (value WILL blend), forcing a batch of extra raw `bOverride_*` leaf read-backs to confirm the writes. Verbatim friction quoted above. Same value-vs-default limitation as DONE `B-asset-dump-...-spurious-override-on-bp-internal-bools` (inverse symptom, different surface/field class). Fix: have override-state consult the sibling `bOverride_<Name>` FBoolProperty (or add an `overrideFlagSet` field). Docs: `docs/wiki-src/property.md` should caveat `isOverridden` for `bOverride_`-gated struct fields and point at the companion bit. Deduped via ripgrep across OPEN/closed (isOverridden / includeOverrideState / bOverride / override-state): no existing ticket covers the `property.get`-override-state-vs-`bOverride`-bit angle. Severity Low (recoverable via raw-bit read-back; never blocks).
