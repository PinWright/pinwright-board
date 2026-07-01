---
id: F-xml-bp-class-resolution
title: "`widget_import_xml` — BP widget class resolution incomplete"
status: DONE
severity: ""
category: feature
tags: []
---

# `widget_import_xml` — BP widget class resolution incomplete

Some BP widget classes fail to resolve (`W_AnimatedButton`) while others (`W_TutorialIntroBox`, `W_HUD_Common`, `W_PhotoPopup`) work. No clear pattern. `_C` suffix strip implemented but doesn't help.

**Impact:** Must use engine base classes instead of project-specific styled widgets.

## History
- `#1-bp-class-resolve-failed` `OPEN` reporter — W_AnimatedButton failed both with and without _C suffix. Used CommonButtonBase as workaround for photo popup buttons.
- `#2-added-three-resolve-fixes` `IN-REVIEW` developer — Three fixes: (1) Added FindFirstObjectSafe with _C suffix for already-loaded BP classes. (2) Added WaitForCompletion guard before asset registry scan to ensure background discovery is complete. (3) Added diagnostic logging on resolution failure. All fixes are generic, no project-specific paths.
- `#3-verified-animated-button` `DONE` tester — Verified: `mcp__editor_automation__.call path="widget.import_xml" args={...}` with `<W_AnimatedButton Name="TestBtn"/>` on W_McpVerifyTemp resolved successfully. `mcp__editor_automation__.call path="widget.describe" args={...}` confirms class W_AnimatedButton_C instantiated. Previously-failing BP widget class now resolves.
