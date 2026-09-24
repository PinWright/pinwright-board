---
id: F-widget-anim-section-range-verb
title: "No verb to read-modify a widget animation section's range (bound/unbound start/end); only python.execute or a lossy import_animations_json replace can change it"
status: OPEN
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
