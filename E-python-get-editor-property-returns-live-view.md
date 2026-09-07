---
id: E-python-get-editor-property-returns-live-view
title: "get_editor_property hands back a live reference into the object's own memory, not a snapshot, so the obvious before/after log around a write reports the new value on both sides — a 4-entry-to-1-entry edit logged as '4 -> 4' — and nothing in the python wiki says so"
status: OPEN
severity: Medium
category: ergonomic
tags: [python, python-execute, wiki, wiki-src, docs, get_editor_property, reflection, read-modify-verify, silent-wrong-verification, snapshot, containers, elements, silent-write-loss, landscape-grass]
encounters: 2
costly: 2
lastSeen: 2026-08-29T20:20:00+03:00
---

# The verification half of read-modify-write is aliased to the thing it is meant to verify

`safe-mutation-save.md:24` prescribes *"Verify read-after-write with the same read surface used in
step 1"*, and `:22` names `python.execute` as the writer of last resort. Follow both — capture the
value, write, compare — and from bundled Python the comparison cannot fail, because the "before"
handle is a window onto the object's live memory rather than a copy of what was there.

## Measured

Editing a `UPCGMeshSelectorWeighted`'s `mesh_entries` during the zone D re-speciation of
`PW_VegetationTest` (`Docs/map/vegetation-style-split.md` § Findings 4):

```python
old = sel.get_editor_property('mesh_entries')   # 4 entries
sel.set_editor_property('mesh_entries', new)    # 1 entry
print(len(old), '->', len(sel.get_editor_property('mesh_entries')))
```

printed `4 -> 4` for an edit that went **4 → 1**. The write landed correctly; the log said it had
not happened, and a log written the other way round (asserting the count *changed*) would have said
it had, for a write that did nothing. Either polarity is wrong, which is what makes it worse than a
plain no-op: the pattern reports whatever the current state is, twice, under two labels.

## Mechanism, and it is more general than the case that found it

`get_editor_property` is registered at
`C:/UE_5.8/Engine/Plugins/Experimental/PythonScriptPlugin/Source/PythonScriptPlugin/Private/PyWrapperObject.cpp:1132`,
implemented at `:801`, and dispatches at `:834` into `PyGenUtil::GetPropertyValue`
(`PyGenUtil.cpp:1418`), which reaches `PyUtil::GetPropertyValue` (`PyUtil.cpp:888`). That function
converts with one hardcoded mode:

```cpp
if (!PyConversion::PythonizeProperty_InContainer(InProp, InStructData, 0, PyPropObj, EPyConversionMethod::Reference, InOwnerPyObject))
```
`PyUtil.cpp:906`

`EPyConversionMethod::Reference` is documented in the enum itself as *"Reference the value from the
given owner"* (`PyConversionMethod.h:10-18`). For an array, that resolves in `FPyWrapperArray::Init`
(`PyWrapperArray.cpp:173`) to the branch at `:226-230`:

```cpp
case EPyConversionMethod::Reference:
    PropToUse = PyUtil::FConstArrayPropOnScope::ExternalReference(InProp);
    ArrayInstanceToUse = InValue;
```

`ArrayInstanceToUse = InValue` — the wrapper points at the object's own `FScriptArray`. Contrast the
`Copy` / `Steal` branch at `:191-224`, which mallocs a fresh instance and `CopyCompleteValue`s into
it. So the returned object is not a stale copy, not a lazy copy, and not a copy: it is the property.
`len(old)` reads `Num()` off the live array header, which is why it tracks a later write.

**The generalisation matters more than the case.** `EPyConversionMethod::Reference` is on the
`get_editor_property` path for *every* property type, not for arrays specially. Sets, maps and
structs read the same way (`FPyWrapperSet`, `FPyWrapperMap`, `FPyWrapperStruct` all take the same
conversion mode), so "capture the struct, mutate the object, compare" has the identical hole. The
original finding read this as a struct-`TArray` quirk; it is the default behaviour of the reflection
read.

**And the remedy is mechanism-backed, not folklore.** `.copy()` exists on all four wrappers —
`PyWrapperArray.cpp:1453` (impl `:1225-1233`), `PyWrapperStruct.cpp:1303`, `PyWrapperSet.cpp:1431`,
`PyWrapperMap.cpp:1472` — and the array implementation constructs its result with
`EPyConversionMethod::Copy` (`:1232`), i.e. the malloc-and-`CopyCompleteValue` branch above. So
`.copy()` is a genuine snapshot and not merely a differently-named alias. One caveat worth writing
down with it: `arr.copy()` deep-copies the array's element storage, but iterating the *original*
(`[e for e in old]`) yields element wrappers that still reference it, so a caller keeping elements
rather than the container needs `.copy()` on each element too.

## Why this is a PinWright ticket and not an engine complaint

Nothing above is a PinWright defect and none of it can be fixed in PinWright's code. What PinWright
owns is that it ships `python.execute` as a documented writer (`safe-mutation-save.md:22`), ships a
verification doctrine that says to read back through the same surface (`:24`), and ships a `python`
namespace page (`Docs/wiki-src/python.md`) that already carries exactly this kind of
engine-behaviour warning — § *Marking packages dirty from Python* (`python.md:13`), § *Calls that
crash the editor* (`:60`), § *Calls that freeze the editor* (`:83`). The rule belongs in that list
and is not in it. A caller following PinWright's own guidance is walked into a verification that
cannot fail.

Precedent for filing an engine-Python behaviour here: `E-python-cannot-mark-package-dirty` (DONE,
Medium), filed for the same reason — the advice the plugin gives cannot be carried out as written —
and `B-python-execute-private-scope-leaks-sys-modules` item (c), which put the
`register_component()` / `is_registered()` `AttributeError` pair onto a wiki page rather than into
code.

## Fix

A section on `Docs/wiki-src/python.md`, next to the package-dirtying one, saying: `get_editor_property`
returns a live reference for containers and structs (`PyUtil.cpp:906`), so a before/after comparison
around a write compares a value to itself; snapshot with `.copy()` before writing, and `.copy()`
each element if you keep elements rather than the container. Worth one line on
`safe-mutation-save.md` § *Verification Patterns* too, since that section's rule — read back through
the same surface — is the one this breaks, and it currently lists only RPC surfaces.

A second, optional half worth considering separately: the same trap is why a `python.execute`
before/after cannot substitute for `property.get`, which returns JSON and is therefore a real
snapshot. If the page says that out loud it also nudges callers back onto the typed surface.

## Same shape as

The session's recurring class (`B-foliage-paint-does-no-ground-projection` § *Same shape as*): the
call succeeds, every number it reports is correct, and the output is wrong because the deciding
number was never reported. This is the variant where the caller *does* report the deciding number
and it is the wrong instance of it. Closest sibling on the board is
`B-foliage-remove-empties-ledger-not-component` (OPEN, Critical), where the mutator and its readback
share a blind spot; here the "before" and the "after" share an address.

Project-side note, cross-referenced not duplicated: the host project's `CLAUDE.md` already carries
the sibling rule for asset saves — *"Verify a write against disk, not against the object you just
wrote"*, because `load_asset` returns the already-loaded in-memory object. This is the same failure
one layer down, on a property instead of a package, and neither the host doc nor the plugin wiki
states it in the general form.

## Severity

**Medium.** Impact class is the rubric's Medium band — *doable, but only via a documented workaround
or a source dive*. The workaround (`.copy()`) is one call and it is discoverable **only** by reading
engine Python-plugin source, which is what it took to write the mechanism section above; nothing in
the plugin wiki, the engine's Python docs, or the method's own docstring (*"get the value of any
property visible to the editor"*, `PyWrapperObject.cpp:1132`) hints at it.

**Not Low**, argued rather than asserted. The Low band is *"docs, discoverability, naming"* — a doc
that fails to help. This absent doc lets a caller construct a verification that reports success for
a write that did not happen and failure for one that did, which is a wrong output rather than a
missing convenience. That is the same distinction `B-wiki-hism-recipe-resets-actor-transform` drew
when it filed a wiki defect above the Low band. Filed `E-` rather than `B-` only because there is no
PinWright artefact stating anything false here: the gap is an absence on `python.md`, not a wrong
line on it.

**Not High**: no PinWright verb returns a false field, and the actual writes in the measured case
were all correct — only the log about them was wrong. Nothing was corrupted and nothing needed
redoing.

**Reach modifier declined, deliberately, in the direction that would bump it up.** The rule covers
every container and struct read through `get_editor_property`, which is the single most-used call
in any `python.execute` script, and `python.execute` appears in most non-trivial sessions on this
project. That is a strong case for High-by-reach. It is declined because the trap only bites a
caller who writes a *before/after* comparison — capturing a value, mutating, then comparing to the
capture — and a script that simply reads, writes, and re-reads is unaffected. The reach of the
mechanism is near-universal; the reach of the failure is not. Medium stands.

## History
- `#1-reference-not-snapshot` `OPEN` reporter — Measured during the zone D re-speciation of
  `PW_VegetationTest` (`Docs/map/vegetation-style-split.md` § Findings 4): `old =
  sel.get_editor_property('mesh_entries')` (4 entries) followed by `sel.set_editor_property(
  'mesh_entries', new)` (1 entry) left `len(old)` reporting **4**, so a before/after log printed
  `4 -> 4` for a 4 → 1 edit. The write itself was correct; only the verification was wrong, and it
  is wrong in both polarities — the pattern reports the current state twice under two labels.
  MECHANISM (engine source, UE 5.8 PythonScriptPlugin): `get_editor_property`
  (`PyWrapperObject.cpp:1132`, impl `:801`, dispatch `:834`) → `PyGenUtil::GetPropertyValue`
  (`PyGenUtil.cpp:1418`) → `PyUtil::GetPropertyValue` (`PyUtil.cpp:888`), which converts with a
  hardcoded `EPyConversionMethod::Reference` at `PyUtil.cpp:906`; for an array
  `FPyWrapperArray::Init` (`PyWrapperArray.cpp:173`) then takes the `Reference` branch at
  `:226-230` and sets `ArrayInstanceToUse = InValue`, aliasing the object's own `FScriptArray`,
  against the `Copy`/`Steal` branch at `:191-224` which mallocs and `CopyCompleteValue`s.
  **Finding generalised in the process, and it got stronger:** the original read this as a struct-
  `TArray` quirk, but `Reference` is the mode for the whole `get_editor_property` path, so sets,
  maps and structs alias identically. **Remedy is mechanism-backed:** `.copy()` exists on all four
  wrappers (`PyWrapperArray.cpp:1453`/`:1225-1233`, `PyWrapperStruct.cpp:1303`,
  `PyWrapperSet.cpp:1431`, `PyWrapperMap.cpp:1472`) and the array implementation builds its result
  with `EPyConversionMethod::Copy` at `:1232`, so it is a real snapshot — with the caveat that
  iterating the original still yields element wrappers referencing it, so elements need their own
  `.copy()`. SCOPE: nothing here is fixable in PinWright code; what PinWright owns is that
  `safe-mutation-save.md:24` prescribes read-back verification, `:22` names `python.execute` as a
  writer, and `Docs/wiki-src/python.md` already carries this exact class of engine warning
  (`:13` dirtying, `:60` crashes, `:83` freezes) without this one. Precedents for filing an
  engine-Python behaviour on this board: `E-python-cannot-mark-package-dirty` (DONE, Medium) and
  `B-python-execute-private-scope-leaks-sys-modules` item (c). Asked for: a `python.md` section
  plus a line on `safe-mutation-save.md` § *Verification Patterns*, which currently lists only RPC
  read surfaces. Rated **Medium** on the source-dive band — the workaround is one call and is
  discoverable only from Python-plugin source, and the method's own docstring says nothing
  (`PyWrapperObject.cpp:1132`). Declined **Low** because the absent doc produces a wrong
  verification rather than missing help, the distinction
  `B-wiki-hism-recipe-resets-actor-transform` drew; filed `E-` rather than `B-` because no shipped
  PinWright artefact states anything false — the gap is an absence. Declined **High**: no verb
  returns a false field and the measured writes were all correct. Reach modifier declined in the
  direction that would raise it, with the argument stated: the mechanism is near-universal, but the
  failure only bites the before/after comparison pattern, not a plain read-write-reread.
- `#2-elements-are-copies-the-opposite-polarity` `OPEN` reporter — **Second encounter, from a
  performance-profiling pass on `/Game/Maps/PW_VegetationTest` (`Docs/map/vegetation-performance.md`
  § *Two Python write traps*), and it corrects a load-bearing line in `#1`.** The container is a
  reference; **its elements are not**, and the two failures point in opposite directions.
  **Measured:** editing `ULandscapeGrassType::GrassVarieties` (a `TArray<FGrassVariety>`) with
  `for v in gt.get_editor_property('grass_varieties'): v.set_editor_property(...)` reports success on
  every call, `save_asset` returns `True`, the `.uasset` mtime moves — and the stored value is
  unchanged. Writing elements back by index and re-assigning the whole array with
  `set_editor_property('grass_varieties', arr)` works. **Mechanism, re-derived against UE 5.8
  PythonScriptPlugin source:** `FPyWrapperArray::GetItem` (`PyWrapperArray.cpp:409-432`) calls
  `PyConversion::PythonizeProperty(...)` at **`:426` with three arguments**, silently taking the
  header defaults declared at **`PyConversion.h:226`** — `EPyConversionMethod::Copy` and
  `OwnerPyObj = nullptr`. That single omitted argument is the whole defect, and it is the exact
  argument `PyUtil.cpp:906` (this ticket's `#1` line) passes explicitly one level up. All three
  element surfaces funnel through it: `sq_item` (`PyWrapperArray.cpp:1490` -> `:990-993`),
  `mp_subscript` (`:1499` -> `:1123`), and the iterator, whose hot line `:96` is a bare
  `FPyWrapperArray::GetItem(InSelf->IterInstance, InSelf->IterIndex++)` — so `for v in arr` **is**
  `GetItem`. For a `FStructProperty` inner it routes `PyConversion.cpp:1243-1247` -> `:692-701` ->
  `:1163-1167` into `FPyWrapperStructFactory::CreateInstance` (`PyWrapperTypeRegistry.cpp:525-537`,
  `:536` forcing a fresh wrapper for Copy/Steal) and lands in the **Copy** branch of
  `FPyWrapperStruct::Init` (`PyWrapperStruct.cpp:144-151`: `AllocateStruct` + `InitializeStruct` +
  `CopyScriptStruct`), not the Reference branch at `:153-157`. `v.set_editor_property(...)` then writes
  correctly into that private duplicate (`PyWrapperStruct.cpp:1131` -> `:443-460`, `:459` ->
  `PyGenUtil.cpp:1465` -> `PyUtil.cpp:958`), returns `None`, and the duplicate is freed when the loop
  rebinds `v`. Because `OwnerPyObj` was null the owner context built at `PyConversion.cpp:1086` is
  empty, so `FPyWrapperOwnerContext::BuildChangeNotify` (`PyWrapperOwnerContext.cpp:55-115`) discards
  the notify at `:109-113` — **no `PreEditChange`, no `PostEditChangeProperty`, no `MarkPackageDirty`
  on the owning asset**, which is why nothing anywhere complains. The whole-array re-assign works
  because it goes through `FPyWrapperObject`'s setter where the owner *is* a `UObject`, so
  `BuildChangeNotify` reaches `PyWrapperOwnerContext.cpp:93` and the copy is written back by
  `CopyScriptStruct` at `PyConversion.cpp:929`. Same defaulted 3-arg form at `PyWrapperArray.cpp:746`
  (`Pop`), `:823` (`Sort`), `PyWrapperSet.cpp:127`/`:561`, and `PyWrapperMap.cpp:228`/`:235`/`:256`/
  `:276`/`:848`/`:939`/`:1016`. **Correction to `#1`, and it is the important part of this entry:**
  `#1`'s Fix says to *"`.copy()` each element if you keep elements rather than the container"*, and its
  Mechanism says sets, maps and structs *"read the same way"*. That is right for the top-level
  `get_editor_property` read and **wrong one level in** — elements are already detached copies, so
  `.copy()` on one is a no-op, and the belief it encodes (that iterating yields live references) is
  precisely what makes the doomed `for v in arr: v.set(...)` pattern look sound. The doc section this
  ticket asks for must state **both** halves in one place — the container aliases, its elements do not —
  because a caller who learns only the `#1` half will get this half wrong in the opposite direction, and
  a caller who learns only this half will write a before/after log that cannot fail. **Three premises in
  the incoming report did not survive and are recorded so they do not recur:** (i) *"`PerPlatformInt.default`
  is read-only from Python"* is **false** — `FPerPlatformInt::Default` (`PerPlatformProperties.h:187-188`)
  and `FPerPlatformFloat::Default` (`:233-234`) are both `UPROPERTY(BlueprintReadOnly, EditAnywhere)`, and
  the Python path gates on `PropertyAccessUtil::EditorReadOnlyFlags` (`PyUtil.cpp:927`), not
  `CPF_BlueprintReadOnly`, so `.default` is settable and rebuilding the struct is not the only route;
  (ii) the reported snippet carried an **independent** type error — `FGrassVariety::GrassDensity` is
  `FPerPlatformFloat` (`LandscapeGrassType.h:44-45`), so `unreal.PerPlatformInt(300)` fails the
  `IsChildOf` gate at `PyConversion.cpp:921` and raises `TypeError` even against a correctly-referenced
  element (`unreal.PerPlatformFloat(300.0)` is the correct form); (iii) no UPROPERTY specifier blocks
  anything here — `ULandscapeGrassType::GrassVarieties` is `EditAnywhere`
  (`LandscapeGrassType.h:175-176`). Also worth writing down beside `CLAUDE.md`'s save rule: **the moved
  `.uasset` mtime is not evidence** — `save_asset` rewrote the package whether or not the property
  changed. Severity left at **Medium** and not raised: this half is a silent *write* loss rather than a
  wrong verification, which is a stronger impact, but it remains unfixable in PinWright code and the
  ask is still one doc section — and the workaround (index-assign, then re-assign the array) is one
  extra statement. Raising it would sort a doc edit above shipped-code defects. `encounters` 1 -> 2.
