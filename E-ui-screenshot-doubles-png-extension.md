---
id: E-ui-screenshot-doubles-png-extension
title: "ui.screenshot writes a doubled .png.png extension when the filename already ends in .png"
status: OPEN
severity: Low
category: ergonomic
tags: [ui, screenshot, filename, extension, docs]
encounters: 3
lastSeen: 2026-07-02T16:00:00Z
---

# ui.screenshot writes a doubled .png.png extension when filename ends in .png

`ui.screenshot` with `filename: "hud_final_state_ui.png"` writes the file to
`hud_final_state_ui.png.png` — it unconditionally appends `.png` even when the
caller already supplied that extension. The handler returns success, so the
caller's recorded/expected path (`...ui.png`) does not match the actual file on
disk (`...ui.png.png`); a downstream read of the path the caller passed misses
the file.

This is distinct from `B-thumbnail-png-writes-jpeg` (IN-REVIEW), which fixed
the *byte-encoding* mismatch (JPEG bytes in a `.png` file) in `ui.screenshot`'s
encode path. This ticket is about *filename composition* — the extension is
duplicated regardless of the bytes written. The two defects live in different
parts of the same handler and can be fixed independently.

## What it should do

- Append the extension only when the supplied `filename` does not already end
  in it (case-insensitive), so `foo.png` → `foo.png` and `foo` → `foo.png`.
- Echo the actual resolved on-disk path in the success response so the caller
  always knows where the file landed (and can detect any normalization).
- Document the filename/extension handling on the `ui.screenshot` overlay
  (`docs/wiki-src/ui.md`) — whether an extension is required, optional, or
  auto-appended.

## Evidence

From a live-HUD PIE preview task. The attempt agent fell back to `ui.screenshot`
after `editor.screenshot` hung, and noted, verbatim:

> ui.screenshot writes the wrong/doubled extension (file is
> 'hud_final_state_ui.png.png' and the bytes are JPEG, not PNG).

Call log: `ui.screenshot filename hud_final_state_ui.png.png (fallback)` →
ok=true (the success was reported against the doubled-extension path). The JPEG
half is already addressed by `B-thumbnail-png-writes-jpeg`; the doubled
extension is unfiled until now.

**Workaround:** pass a `filename` with no extension, or read back the doubled
path (`<name>.png.png`) instead of the name you passed.

## History
- `#1-initial-audit` `OPEN` reporter — Live-HUD PIE preview task (fallback after editor.screenshot hung): `ui.screenshot {filename:"hud_final_state_ui.png"}` wrote `hud_final_state_ui.png.png` (extension appended even though `.png` was already present) and reported success against the doubled path, so the caller's expected path did not match disk. Distinct from the JPEG-encoding fix in `B-thumbnail-png-writes-jpeg` (filename composition vs byte encoding). Propose appending the extension only when absent, echoing the resolved on-disk path, and documenting filename handling on the `docs/wiki-src/ui.md` `ui.screenshot` overlay.
- `#2-repro-crosshair-hud-task` `OPEN` reporter — Cross-task aggregation: independent reproduction in a crosshair/interaction-prompt PIE HUD task. `ui.screenshot {filename:"hud_crosshair_visible.png"}` → saved as `hud_crosshair_visible.png.png` (same unconditional `.png` append on a name already ending in `.png`); the next two shots passed extensionless filenames (`hud_crosshair_hidden`, `hud_final_state`) and were fine, confirming the doubling is specific to the already-`.png` case. Attempt agent's friction note, verbatim: "Minor extra wart: ui.screenshot appends .png to a filename that already ends in .png, producing hud_crosshair_visible.png.png." Reinforces #1's proposal (append-only-when-absent + echo resolved on-disk path); the caller had to mentally track the doubled path to know where the file landed.
- `#3-additional-tuning-smoke-repro` `OPEN` reporter — Additional evidence (fresh session, today): `ui.screenshot {path:".../scratchpad", filename:"tuning_screen_smoke.png", returnBase64:false}` → response `screenshotPath` = `.../tuning_screen_smoke.png.png`, file confirmed on disk at the doubled path; a second same-session call with `filename:"tuning_screen_improved"` (no extension) produced a correct single `.png`, again isolating the defect to the already-`.png` case. Verified still unfixed in source: `Source/PinWright/Private/Handlers/UI/UiHandler.cpp:135` builds `FPaths::Combine(ScreenshotPath, Filename + TEXT(".png"))` — a blind `Filename + TEXT(".png")` with no case-insensitive "already ends in .png" guard, exactly the spot #1's proposed fix targets. Severity stays Low.
