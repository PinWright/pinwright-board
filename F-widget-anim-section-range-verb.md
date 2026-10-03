---
id: F-widget-anim-section-range-verb
title: "No verb to read-modify a widget animation section's range (bound/unbound start/end); only python.execute or a lossy import_animations_json replace can change it"
status: IN-REVIEW
severity: Medium
category: feature
tags: [widget, animation, umg, moviescene, section-range, missing-verb]
encounters: 1
lastSeen: 2026-09-24T07:25:00Z
---

# No verb to change a widget animation section's range

A UMG animation whose sections end exactly at the last key (`[0, N)` with the
final key at frame `N`) and whose playback range ends at `N+1` never evaluates
the final key: the player's last tick evaluates at frame `N`, which is outside
the exclusive section end, so the widget keeps whatever pose the previous
in-range tick produced (frame-rate dependent: at 10 fps the first tick overshoots
and the widget never moves at all). The fix is to make the section open-ended
(`TRange::All()`, the UMG default for new sections) or extend its end. PinWright
can read the range (`widget.export_animations_json` -> `sections[].range`) but has
no verb to change it:

- `widget.*` animation verbs cover create / track / keyframe / playback range /
  loop / speed; nothing edits a section's range.
- `sequencer.set_sub_section_range` is for level-sequence sub-sections.
- `widget.import_animations_json mode:replace` deletes and re-creates the whole
  `UWidgetAnimation` (new object, new binding GUIDs, unsupported track types and
  event metadata dropped), which is far too destructive for a range tweak, and
  its range reader cannot express an unbounded range anyway (see
  `B-widget-anim-json-unbounded-range-imports-empty`).

**Workaround (used):** `python.execute` on the BP's animation subobject.
`WidgetBlueprint.Animations` is protected from Python (`Property 'Animations' ...
is protected and cannot be read`), so resolve it by path:
`unreal.find_object(None, bp.get_path_name() + ':<AnimName>')`, then
`sequence.get_bindings()[i].get_tracks()[j].get_sections()[k].set_start_frame_bounded(False)`
/ `set_end_frame_bounded(False)`, mark dirty, `blueprint.compile`, `asset.save`.

**Gotcha found while verifying:** editing the compiled-class copy
(`<BP>_C:<Anim>_INST`) in memory during PIE changes the section object but the
running player keeps evaluating stale compiled data; toggling a range back and
forth there produced wrong poses until the BP was recompiled. A verb should
edit the BP's animation and require/perform the compile.

**Proposed:** `widget.set_animation_section_range { widgetPath, animationName,
widgetName?, trackType?/propertyName?, sectionIndex?, start?: number|"unbounded",
end?: number|"unbounded", units: "ticks"|"displayFrames"|"seconds" }`, echoing
before/after ranges, plus an audit mode that lists sections whose exclusive end
is at or before their last key while the playback range extends past it (that
pattern was found on 70 widgets / 193 sections in the host project).

## History
- `#1-missing-section-range-verb` `OPEN` reporter — Needed to open-end two RenderTransform sections of a UMG hover animation (sections [0,6000) at 60000 tick/s, playback [0,6001), final key at 6000). No verb exists; used python.execute via find_object on `<BP path>:<Anim>` because `Animations` is protected. Verified in PIE with detailed UMG logging: last tick evaluates at frame 6000, so the unfixed section never applies the final key.
- `#2-section-range-verbs` `IN-REVIEW` developer — Added `widget.get_animation_section_ranges` (read-only; per section `range` in the export_animations_json shape, `keyCount`/`firstKeyFrame`/`lastKeyFrame`, `keysInsideSection`, `locked`, `compiledRange`/`compiledCopyMatches` against the compiled `<Anim>_INST` copy, top-level `compileRequired`) and `widget.set_animation_section_range` (filters `widgetName`/`propertyName`/`sectionIndex`; `start`/`end` each a number or `"unbounded"`; `units` ticks (default) | displayFrames | seconds; all matched sections validated before any write -> `INVALID_PARAMETER` for an empty range / fractional ticks / bad units, new `SECTION_READ_ONLY` for a locked section, `SECTION_NOT_FOUND` for no match; one `FScopedTransaction`; per-section `before`/read-back `range`/`applied`/`changed`, `VERIFICATION_FAILED` if a section did not take the value; `MarkBlueprintAsModified` + mark-dirty save report). The verb does NOT compile (a direct `CompileBlueprint` in a handler is ratcheted by the tick-safety contract and needs the reinstancing guard); instead it measures the compiled copy and reports `compileRequired`, which addresses the reporter's stale `_INST` gotcha. Not done: the project-wide audit sweep; `keysInsideSection:false` is the per-animation signal. Files: `Source/PinWright/Private/Handlers/UI/WidgetAnimationSectionRangeHandler.cpp` (new), `Source/PinWright/Private/Tests/Widget/TestWidgetAnimationSectionRange.cpp` (new), `Handlers/ErrorCodes.h` (+`ERR_SECTION_READ_ONLY`), `docs/wiki-src/widget.md` (two H3 sections), `CHANGELOG.md`. Tests: `PinWright.widget.animation_section_range.OpenEndReachesFinalKeyAfterCompile`, `PinWright.widget.animation_section_range.UnitsAndRefusals`.
- `#3-review-fixes` `IN-REVIEW` developer — Applied review ad8f3e5b0f2a13074 NITs in `WidgetAnimationSectionRangeHandler.cpp`: (1) the pre-write gate now refuses `SECTION_READ_ONLY` for `IsLocked() || MovieScene->IsReadOnly()` (the same condition `UMovieSceneSection::TryModify` checks), so a read-only MovieScene no longer falls through to a silent `SetRange` no-op and `VERIFICATION_FAILED`; (2) `MarkBlueprintAsModified` + save now run whenever `changedCount > 0`, before the `VERIFICATION_FAILED` return; (3) `trackName` and the `propertyName` track-name match use `UObject::GetName()` instead of `GetTrackName()` (NAME_None for non-property tracks); (4) each written section echoes `requested` (target range in ticks after unit conversion). `widget.md` now documents `trackName`, `requested`, the read-only MovieScene refusal, that `keysInsideSection`/`firstKeyFrame`/`lastKeyFrame` are omitted when `keyCount` is 0, and that `compiledCopyMatches:false` without `compiledRange` means no matching compiled copy. `UnitsAndRefusals` gains a read-only-MovieScene refusal case and a `trackName == track object name` assertion (both fail if reverted). Tests: `PinWright.widget.animation_section_range.OpenEndReachesFinalKeyAfterCompile`, `PinWright.widget.animation_section_range.UnitsAndRefusals`. Not yet built or run. Re-review NITs: ErrorCodes.h comment covers read-only MovieScene; VERIFICATION_FAILED message drops the read-only hint and its payload now carries compileRequired + save report; UnitsAndRefusals asserts propertyName matches the track object name and the `requested.endFrame` echo for seconds.
- `#4-linux-verification` `IN-REVIEW` tester — Passed non-skipped in run3/full: `PinWright.widget.animation_section_range.OpenEndReachesFinalKeyAfterCompile`, `PinWright.widget.animation_section_range.UnitsAndRefusals`, `PinWright.widget.animation_json.UnboundedSectionRangeRoundTrip`. Demonstrated: `widget.set_animation_section_range` sets bounded/unbounded start/end in ticks/displayFrames/seconds, with filters, before/after echo, typed refusals and the compiled `_INST` staleness reported as `compileRequired`. The open-ended section reaches its final key after compile. Remains: the ticket's proposed audit mode (list sections whose exclusive end is at or before their last key) was left out (#2); `keysInsideSection` is the per-animation signal only. The verb also reports, rather than performs, the compile. A human must accept dropping the audit sweep or file it as a follow-up.
