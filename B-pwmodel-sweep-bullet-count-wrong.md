---
id: B-pwmodel-sweep-bullet-count-wrong
title: "pwmodel-format.md says 'Six things about sweep and extrude_along_spline' above a list of seven bullets, so a reader who stops counting at six skips one of the two ops' documented behaviours"
status: OPEN
severity: Low
category: bug
tags: [docs, pwmodel, sweep, extrude-along-spline, off-by-one, trivial]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# Off-by-one in a counted list

`X:\src\unreal\unreal-fpv-new\Plugins\PinWright\Docs\pwmodel-format.md:451` reads:

> Six things about `sweep` and `extrude_along_spline` that the names do not say:

Seven top-level bullets follow, at lines 453, 461, 470, 478, 490, 493 and 528. No `##`/`###`
heading and no fenced block breaks the list, so all seven belong to it.

The seven, by their bold lead-ins:

1. The profile lands in each frame's local Y-Z plane; the sweep advances along the frame's
   local +X.
2. A frame whose local +X is PERPENDICULAR to the segment leaving it sweeps nothing.
3. `cap` works, and `extrude_along_spline` loops only when the path returns to its start.
4. They are modifiers and they APPEND.
5. Without `profile=` the cross-section is a CIRCLE sized from the accumulated mesh's bounding
   box.
6. `scale_start` / `scale_end` interpolate LINEARLY.
7. They take `material=`, and untagged they INHERIT the slot of the geometry they append onto.

A counted lead-in is a contract with the reader: it tells them when to stop. Six of these are
non-obvious behaviours a caller gets wrong by default — bullets 2 and 3 each describe a way to
get a silently degenerate mesh — so a reader who counts to six and moves on loses a real one.

**Fix:** change "Six things" to "Seven things" at `Docs/pwmodel-format.md:451`. Alternatively
drop the count ("Things about `sweep` and `extrude_along_spline` that the names do not say:"),
which cannot go stale the next time a bullet is added — this is at least the second bullet
added since the count was written.

## Provenance and timing

Pre-existing at `HEAD`. Note the file currently carries **uncommitted changes**
(`git diff --stat HEAD -- Docs/pwmodel-format.md` → 64 insertions, 28 deletions) from another
agent's in-flight work, which is plausibly where the seventh bullet came from. A fixer must
re-check the count against the tree at the time of the fix rather than trusting the number
here, and must not revert the surrounding edits.

## Severity

**Low.** Pure friction, docs, cosmetic-adjacent; nothing behaves wrongly and the content is
all present and correct. Reach bump not applied — `pwmodel-format.md` is a reference for one
authoring format, not an every-session read. This is exactly the rubric's Low band and there is
no argument for more.

Filed rather than fixed in passing because the tree is mid-verification and this file is being
edited concurrently; a one-word edit is not worth a merge conflict on a 92-line diff.

## History
- `#1-seven-bullets-under-six` `OPEN` reporter — Verified by reading the file: the sentence is at `Docs/pwmodel-format.md:451` and seven `- **` top-level bullets follow at offsets +2, +10, +19, +27, +39, +42 and +77, with no `#` heading and no fence between them (checked explicitly, since a heading would have split the list and made the count right). Pre-existing at `HEAD`. **Not fixed in this pass by instruction and by choice** — the file has 64 insertions / 28 deletions uncommitted from another agent's concurrent work, so a fixer must re-count against the tree at fix time rather than trusting this ticket's number, and must not revert the surrounding edits. Severity Low with no reach bump, argued above.
