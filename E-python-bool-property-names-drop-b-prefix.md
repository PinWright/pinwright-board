---
id: E-python-bool-property-names-drop-b-prefix
title: "Unreal's Python bindings strip the leading b from every boolean property name, so bOverride_ColorSaturation is override_color_saturation and the C++ spelling the wiki prints in its property.set examples is a name that does not exist on the Python side — the same string is correct on one PinWright surface and unresolvable on the other, and no page says so"
status: OPEN
severity: Low
category: ergonomic
tags: [python, python-execute, wiki, wiki-src, docs, post-process, boverride-foot-gun, property-naming, reflection, silent-noop-adjacent]
encounters: 1
lastSeen: 2026-08-29T18:00:00+05:00
---

# One property, two correct spellings, and the wiki only prints one of them

From bundled UE 5.8 Python, `set_editor_property('b_override_color_saturation', True)` raises
*"Failed to find property"*. The reachable name is `override_color_saturation`. The `b` is not
snake-cased to `b_`; it is deleted.

That is a rule, not a quirk, and it applies to every boolean `UPROPERTY` in the engine — `bEnabled`
→ `enabled`, `bVisible` → `visible`, `bOverride_*` → `override_*`.

## Mechanism

`PyGenUtil::PythonizePropertyName`
(`C:/UE_5.8/Engine/Plugins/Experimental/PythonScriptPlugin/Source/PythonScriptPlugin/Private/PyGenUtil.cpp:1951`),
reached for every property at `:3301`:

```cpp
// Strip the "b" prefix from bool names
if (InName.Len() - NameOffset >= 2 && InName[NameOffset] == TEXT('b') && FChar::IsUpper(InName[NameOffset + 1]))
{
    NameOffset += 1;
```
`PyGenUtil.cpp:1957-1962`

The loop also strips a leading `In` (`:1965-1968`) and repeats until neither prefix matches, then
snake-cases the remainder. So `bOverride_ColorSaturation`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Classes/Engine/Scene.h:725-726`) yields
`override_color_saturation`, and `b_override_color_saturation` is a string the generator can never
produce.

## Why it is worth a doc line rather than a shrug

**The loud failure is fine. Its neighbour is not.** *"Failed to find property"* is the right
behaviour and it is what makes this Low. But the shape of the mistake matters: a caller writing a
post-process override is writing a **pair** — the value and its `bOverride_` flag — and the pair has
an asymmetric failure. Get the flag's name wrong and it throws; wrap that in a `try/except`, or
notice the throw and drop the line, and the value write still succeeds with the override still
`false`. That is the documented silent no-op the plugin already warns about for the MCP path:

> Every value in `FPostProcessSettings` (~150 fields) is paired with a `bOverride_*` boolean.
> **Writing the value without also flipping the override silently no-ops**
> — `Plugins/PinWright/Docs/wiki-src/post_process.md:9`

So the plugin knows the hazard, states it well, and states it only in terms of `property.set`.

**And the wiki prints the spelling that fails.** `Docs/wiki-src/water.md:104` shows it inside a
`property.set`-shaped payload — `"bOverride_SceneColorTint": true,` — and four more pages name
`bOverride_*` flags in prose as things to read or set: `material.authoring.md:209` and `:214`,
`render.capture-exposure.md:98`, `render.preview-scene-rig.md:116`, `widget.md:245`. Every one of
those is correct: `property.set` resolves against `FProperty` names, so the C++ spelling is the
right one on that surface. The problem is that the same string is unresolvable from
`python.execute` and nothing marks the boundary. A caller copying a working
`bOverride_SceneColorTint` out of `water.md` into a Python block gets an exception and no clue that
the string was surface-specific rather than wrong.

**One page was checked and does not carry it**, recorded so nobody re-derives the list:
`render.capture-subjects.md` has no `bOverride_` occurrence, and `water.md:95` is prose about
`FPostProcessSettings` nesting rather than a flag.

**The `python` page is the one with the gap.** `Docs/wiki-src/python.md` already collects exactly
this class of engine-binding trap — § *Marking packages dirty from Python* (`:13`), § *Calls that
crash the editor* (`:60`), § *Calls that freeze the editor* (`:83`) — and has no naming section.
`safe-mutation-save.md` mentions `bOverride_` nowhere.

## Measured

Look-dev polish over `PW_VegetationTest` (`Docs/map/vegetation-polish.md` § 1), applying a
post-process rig: `set_editor_property('b_override_color_saturation', True)` raised *"Failed to
find property"*; `override_color_saturation` worked.

Worth recording why Python was in play at all, since it bears on the reach argument below. The six
typed setters from `F-post-process-typed-setters` (DONE) cover colour grading, bloom, Lumen GI,
Lumen reflections, anti-aliasing and motion blur, and flip the flags for the caller — the pass used
them where they applied. It fell back to Python for the two knobs no typed setter covers,
`lumen_diffuse_color_boost` and `indirect_lighting_intensity`, which is the correct fallback and the
exact situation where the naming rule bites.

## Fix

A short naming section on `Docs/wiki-src/python.md`, alongside the existing engine-behaviour
sections: bindings strip a leading `b` before an uppercase letter and a leading `In`
(`PyGenUtil.cpp:1957-1968`), so `bOverride_X` is `override_x` from Python and `bOverride_X` from
`property.set`; and never swallow a `Failed to find property` on an override flag, because the value
write beside it will succeed and no-op.

One cross-reference line on `Docs/wiki-src/post_process.md`, where the foot-gun is already
described, saying the flag's Python spelling differs — that page is where a caller looking for
override flags actually lands.

## Related

- `F-post-process-typed-setters` (DONE, Low) — the ticket that fixed this foot-gun on the MCP side
  by pairing every value write with its `bOverride_*` flip, and produced `post_process.md`. Nothing
  it did is in question; this is the surface it did not cover, for the knobs it did not cover.
- `E-python-get-editor-property-returns-live-view` (OPEN, Medium) — the sibling `python.md` gap
  found in the same session, and the more expensive one. Both want a section on the same page and
  should be written together.
- `E-python-cannot-mark-package-dirty` (DONE, Medium) and
  `B-python-execute-private-scope-leaks-sys-modules` (IN-REVIEW, Medium) item (c) — the precedents
  for filing an engine-Python behaviour here when it invalidates advice the plugin gives.

## Severity

**Low.** Impact class is the rubric's Low band, verbatim: *"pure friction. Docs, discoverability,
naming"*. This is literally naming. The failure is loud, immediate, and self-correcting once the
rule is known; nothing is written wrongly and nothing is lost.

**Reach modifier declined, and this is the part worth arguing** — the rule covers every boolean
property in the engine and `python.execute` appears in most non-trivial sessions on this project,
which reads like the rubric's *"runs in almost every session, bump up one level"*. It is declined
because the bump is meant for a gap that *bites* every session, and this one bites only a caller who
guesses `b_`-prefixed names. Most callers reach a bool through a typed verb or copy a working
spelling from an example on the correct surface. The population that hits this is specifically
"reaching for a `bOverride_*` flag from Python", which is small precisely because the typed setters
landed.

**Considered and rejected: filing the silent-no-op sibling as its own Medium.** The value-without-flag
write is a genuine silent no-op, and this board rates those higher. It is not filed that way because
the no-op requires the caller to swallow an exception the engine raised — the plugin's surface
behaved correctly and the caller discarded the signal. What PinWright owns is the missing warning,
which is the Low-band item filed here.

## History
- `#1-b-prefix-stripped-from-python-names` `OPEN` reporter — Measured during the look-dev polish
  pass over `PW_VegetationTest` (`Docs/map/vegetation-polish.md` § 1):
  `set_editor_property('b_override_color_saturation', True)` raised *"Failed to find property"*
  while `override_color_saturation` worked. MECHANISM (engine source, UE 5.8 PythonScriptPlugin):
  `PyGenUtil::PythonizePropertyName` (`PyGenUtil.cpp:1951`, entry point `:3301`) strips a leading
  `b` followed by an uppercase letter at `:1957-1962` and a leading `In` at `:1965-1968`, looping
  until neither matches, before snake-casing — so `bOverride_ColorSaturation` (`Scene.h:725-726`)
  becomes `override_color_saturation` and the `b_`-prefixed form is a name the generator can never
  emit. The rule applies to every boolean `UPROPERTY`, not to post-process specially. WHY IT IS A
  TICKET AND NOT A SHRUG: the failure is loud and correct, but the flag is half of a pair, and
  swallowing the exception leaves the value write succeeding with the override still false — the
  documented silent no-op that `Docs/wiki-src/post_process.md:9` already warns about, in terms of
  `property.set` only. Compounding it, the wiki prints the C++ spelling: `water.md:104` inside a
  `property.set`-shaped payload (`"bOverride_SceneColorTint": true,`), plus four pages naming
  `bOverride_*` flags in prose (`material.authoring.md:209`, `:214`,
  `render.capture-exposure.md:98`, `render.preview-scene-rig.md:116`, `widget.md:245`) — all
  correct for `property.set`, which resolves `FProperty` names, and unresolvable from
  `python.execute`, with no page marking the boundary. Every citation in this entry was resolved
  against the file before filing; two candidates from the first sweep did **not** survive and are
  named in the body so nobody re-derives them (`render.capture-subjects.md` carries no
  `bOverride_` at all, and `water.md:95` is prose about `FPostProcessSettings` nesting).
  `Docs/wiki-src/python.md` already collects this class of trap (`:13` dirtying, `:60` crashes,
  `:83` freezes) and has no naming section; `safe-mutation-save.md` mentions `bOverride_` nowhere.
  Context for reach: the pass used the typed `post_process.set_*` setters from
  `F-post-process-typed-setters` (DONE) where they applied and fell back to Python only for
  `lumen_diffuse_color_boost` and `indirect_lighting_intensity`, which no typed setter covers — the
  correct fallback, and exactly where the naming rule bites. Asked for: a naming section on
  `python.md` (strip rules, plus "never swallow a Failed to find property on an override flag") and
  a cross-reference line on `post_process.md`. Rated **Low** — this is naming, the failure is loud
  and self-correcting. Reach modifier **declined with the argument stated**: the rule is universal
  and `python.execute` is common, which reads like the every-session bump, but the bump is for gaps
  that bite every session and this one bites only a caller guessing `b_`-prefixed names from Python,
  a population the typed setters made small. Also considered and rejected: filing the
  value-without-flag no-op as a separate Medium, since that no-op requires the caller to discard an
  exception the engine correctly raised — what PinWright owns is the missing warning, filed here.
  Should be written together with `E-python-get-editor-property-returns-live-view` (OPEN, Medium),
  the sibling gap on the same page from the same session.
