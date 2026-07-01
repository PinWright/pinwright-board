---
id: B-container-set-fname-lookup-broken
title: "container.set.contains/remove miss FName elements — element-match loop lacks an FNameProperty branch"
status: IN-REVIEW
severity: High
category: bug
tags: [container, set, fname, membership]
---

# container.set.contains/remove miss FName elements (element-match loop lacks an FNameProperty branch)

`container.set.add` correctly stores elements into a `TSet<FName>` property
(`setSize` grows 1→2→3, dedupe is a correct no-op), and `container.set.clear`
correctly empties it. But the two **membership-lookup** verbs on the same set —
`container.set.contains` and `container.set.remove` — cannot find any of the
elements that `add` just stored:

- `container.set.contains` returns `contains:false` for **every** genuinely
  present member.
- `container.set.remove` rejects a present member with the documented error
  `[ELEMENT_NOT_FOUND] Element not found in set.`

`property.get` on the same property is the source of truth and proves the
elements are present, so the defect is isolated to the contains/remove lookup
path. **Root cause (verified in source):** these two handlers do NOT do any
hash lookup — each linearly scans the set's elements and compares every element
against the requested value with a per-type `CastField` chain
(`UtilityPropertyHandler.cpp` contains-loop and remove-loop). Those chains
handle only `FStrProperty` / `FIntProperty` / `FFloatProperty` — there is **no
`FNameProperty` branch**, unlike `container.set.add`, which coerces the JSON
string to an `FName` and inserts it. So for a `TSet<FName>` no element ever
matches: `remove`'s `bMatch` never becomes true (falling through to
`ELEMENT_NOT_FOUND`) and `contains`'s `bContains` stays false. It is a simple
add-vs-lookup type-coverage asymmetry, not a hashing/key-construction mismatch.
The result is a silent wrong answer from `contains` and a wrong-but-loud
rejection from `remove` on valid input.

This makes incremental, unique-membership curation of a set property impossible:
a user cannot verify membership or remove a single entry; the only working path
is `add` + `clear` + re-sending the whole container via `property.set`.

**Repro** (verbatim, replayed against a live editor; target is a real reflected
`TSet<FName>` in installed engine source):

```
# 1. add three FName elements — all succeed, set grows
container.set.add  {objectPath:"/Script/Engine.Default__AssetManagerSettings", propertyName:"MetaDataTagsForAssetRegistry", value:"FuzzTag_Alpha"}  -> {setSize:1}
container.set.add  {... value:"FuzzTag_Beta"}   -> {setSize:2}
container.set.add  {... value:"FuzzTag_Gamma"}  -> {setSize:3}

# 2. source of truth: all three are present
property.get  {objectPath:"/Script/Engine.Default__AssetManagerSettings", propertyName:"MetaDataTagsForAssetRegistry"}
   -> {value:["FuzzTag_Alpha","FuzzTag_Beta","FuzzTag_Gamma"], existsAfter:true}

# 3. BUG: contains reports false for a present member
container.set.contains  {... value:"FuzzTag_Beta"}
   -> {contains:false}            # WRONG — Beta is present per property.get

# 4. BUG: remove rejects a present member
container.set.remove    {... value:"FuzzTag_Gamma"}
   -> ERROR [ELEMENT_NOT_FOUND] Element not found in set.   # WRONG — Gamma is present

# control: add/clear work correctly
container.set.add    {... value:"FuzzTag_Beta"}   -> {setSize:3}            # correct dedupe no-op
container.set.clear  {...}                          -> {previousSize:3, newSize:0}
```

**Impact:** Two of the four set verbs are non-functional for FName-keyed sets
(and likely any non-trivially-hashed element type that `add` and lookup encode
differently). `contains` returning a confident `false` is especially dangerous
because it is a silent wrong result, not an error.

**Fix:** Add the missing `FNameProperty` branch to BOTH the
`container.set.contains` and `container.set.remove` element-match loops,
mirroring `container.set.add`'s FName coercion: read the element as
`*reinterpret_cast<const FName*>(ElemPtr)` and compare it against
`FName(*ValueField->AsString())` (linear value comparison — the same scan the
existing FStr/FInt/FFloat branches use, NOT a hash lookup). Add a regression
test that adds an FName element to a reflected `TSet<FName>` then asserts
`contains`==true and `remove` succeeds (and that an absent element still reports
`contains`==false / `ELEMENT_NOT_FOUND`).

## History
- `#1-initial-repro` `OPEN` reporter — Replayed against live editor on `/Script/Engine.Default__AssetManagerSettings.MetaDataTagsForAssetRegistry` (`TSet<FName>`). `add` 1→2→3 and `clear` 3→0 work and `property.get` confirms `[Alpha,Beta,Gamma]` present, but `container.set.contains` returns `false` for present members Alpha/Beta/Gamma and `container.set.remove` fails `[ELEMENT_NOT_FOUND]` on present member Gamma. FName-keyed contains/remove lookup is broken; add/clear use a different (correct) element path.
- `#2-reword-and-fix-fname-branch` `IN-REVIEW` developer — Reworded: the cause is NOT a hash/key-construction mismatch; the contains/remove element-match loops linearly scan and compare per-type, and simply lacked an `FNameProperty` branch (only FStr/FInt/FFloat), unlike `set.add` which coerces to FName. Fix: added the missing `FNameProperty` else-if to BOTH loops in `Source/EditorAutomationRpcGateway/Private/Handlers/Utility/UtilityPropertyHandler.cpp` (container.set.remove and container.set.contains), reading `*reinterpret_cast<const FName*>(ElemPtr)` and comparing to `FName(*ValueField->AsString())` — linear value comparison mirroring set.add. Regression test added: `Source/EditorAutomationRpcGateway/Private/Tests/Utility/TestContainerSetFNameLookup.cpp` (+ fixture `TestContainerSetFNameHost.h` hosting a `TSet<FName>`) drives the real `container.set.contains`/`container.set.remove` handlers end-to-end against a transient reflected FName set: asserts a present member reports `contains==true` and `remove` succeeds and actually shrinks the set, while an absent member reports `contains==false` / `ELEMENT_NOT_FOUND`. Reverting either branch fails the test. Not compiled/run here (later phase verifies).
