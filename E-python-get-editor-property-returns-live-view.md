---
id: E-python-get-editor-property-returns-live-view
title: "get_editor_property hands back a live reference into the object's own memory, not a snapshot, so the obvious before/after log around a write reports the new value on both sides — a 4-entry-to-1-entry edit logged as '4 -> 4' — and nothing in the python wiki says so"
status: OPEN
severity: Medium
category: ergonomic
tags: [python, python-execute, wiki, wiki-src, docs, get_editor_property, reflection, read-modify-verify, silent-wrong-verification, snapshot, containers]
encounters: 1
lastSeen: 2026-08-29T18:00:00+05:00
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
