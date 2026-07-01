---
id: E-asset-list-path-ignored
title: "`asset.list` silently drops top-level `path` when `filter` object is present"
status: DONE
severity: Low
category: ergonomic
tags: []
---

# `asset.list` silently drops top-level `path` when `filter` object is present

`RPC_PARAM_OPT("path", "string", "Package path to list (default /Game)")` advertises top-level `path`, but `AssetManageHandler.cpp:654-665` only reads it inside the `else` branch when the `filter` object is absent. If a caller passes both top-level `path` and `filter`, the top-level `path` is silently discarded and the scan defaults to `/Game`.

**Workaround:** Put the path inside the filter object as `filter.pathStartsWith` or `filter.path`.

**Fix:** After parsing `filter`, if top-level `path` is non-empty AND `filter.path` / `filter.pathStartsWith` are both empty, copy top-level `path` into `PathFilter`. Or deprecate the top-level param in the schema to avoid the ambiguity.

## History
- `#1-top-level-path-silently-dropped` `OPEN` reporter — Surfaced during BPIR auto-formatter investigation. Called `asset.list` with `path="/App/App/UI"` and `filter={"class":"WidgetBlueprint"}`. Got /Game results instead of /App/App/UI results. Re-ran with `filter.pathStartsWith="/App/App/UI"` and got the expected /App/App/UI assets. Top-level `path` was silently dropped when `filter` was present.
- `#2-top-level-path-honored` `IN-REVIEW` developer — `AssetManageHandler::asset.list` now honors top-level `path` whether or not a `filter` object is supplied (top-level `path` is only overridden when `filter.path` / `filter.pathStartsWith` is explicitly set). Covered by `FAssetListTopLevelPathRespectedTest` in `TestAssetHandlers.cpp`.
- `#3-verified-path-with-filter` `DONE` tester — Verified with `asset.list path="/App/App/UI" filter={"class":"WidgetBlueprint"}`. Response contained 3 assets all under `/App/App/UI/...` (e.g., `/App/App/UI/LobbyAndMenu/Elements/W_MouseActivator`), plus subfolders all under `/App/App/UI/...`. Top-level `path` was respected rather than falling back to `/Game`.
