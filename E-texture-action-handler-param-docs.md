---
id: E-texture-action-handler-param-docs
title: "`texture.*` action handlers expose no parameter docs in the wiki ('Parameters: none')"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [texture, wiki, discovery, param-spec, action-handler]
---

# `texture.*` action handlers expose no parameter docs in the wiki ('Parameters: none')

The generated wiki page for `texture.set_compression_settings` reads "Parameters: none.", but the handler actually requires `assetPath` and `compressionSettings` and accepts an optional `save` (default `true`). With nothing documented, the agent has to grep the handler source to discover the real parameters before any call can succeed.

The cause is the registration style. These methods are registered through the `REGISTER_TEXTURE_ACTION_HANDLER(Endpoint, SubActionName, SummaryText)` macro defined at `Source/EditorAutomationRpcGateway/Private/Handlers/Material/TextureHandler.cpp:2741`, which expands to `REGISTER_RPC_HANDLER(Endpoint, "Texture", SummaryText, RPC_NO_PARAMS)` — it hardcodes `RPC_NO_PARAMS`, so discovery/wiki has no `FParamSpec` to render and emits "Parameters: none". Each SubAction parses its params manually from `Params` (e.g. `set_compression_settings` at `TextureHandler.cpp:956-986` reads `assetPath`/`compressionSettings`/`save` by hand). Contrast `texture.describe` immediately below at `TextureHandler.cpp:2758-2759`, registered with `REGISTER_RPC_HANDLER(... RPC_PARAMS(RPC_PARAM_REQ("assetPath", ...)))`, which DOES document its params.

This affects the whole family registered via the macro (`TextureHandler.cpp:2747-2794`): `set_texture_group`, `set_lod_bias`, `resize_texture`, `invert`, `desaturate`, `channel_pack`, `combine_textures`, etc. — ~30 texture action handlers, all showing "Parameters: none".

**Workaround:** Grep the matching SubAction block in `TextureHandler.cpp` for its `Get*FieldTextAuth(Params, ...)` / `ValidParams` set to learn the real parameter names before calling.
**Fix:** Give `REGISTER_TEXTURE_ACTION_HANDLER` a parameter-spec argument (forwarded as the macro's 4th `REGISTER_RPC_HANDLER` arg) so each registration declares its real `RPC_PARAMS(...)`, or migrate the family to direct `REGISTER_RPC_HANDLER(... RPC_PARAMS(...))` registrations like `texture.describe`.

## History
- `#1-initial-repro` `OPEN` reporter — `texture.set_compression_settings` wiki page reads "Parameters: none." but handler requires `assetPath`+`compressionSettings` (optional `save`). Cause: `REGISTER_TEXTURE_ACTION_HANDLER` macro (`TextureHandler.cpp:2741`) hardcodes `RPC_NO_PARAMS`; SubActions parse params by hand (`set_compression_settings` at L956-986). Affects the whole macro-registered family (`TextureHandler.cpp:2747-2794`). Contrast `texture.describe` (L2758) which uses `RPC_PARAMS(...)` and documents params correctly.
- `#2-already-fixed` `IN-REVIEW` developer — Already resolved in current source under test (HEAD `08befea`); no code change. Commit `8c530af` ("fix: declare real RPC param specs for texture and nav handlers", 2026-06-15) is an ancestor of HEAD and implements exactly the proposed fix: `REGISTER_TEXTURE_ACTION_HANDLER` is now variadic (`TextureHandler.cpp:2741-2745` → `REGISTER_RPC_HANDLER(Endpoint, "Texture", SummaryText, __VA_ARGS__)`, no `RPC_NO_PARAMS`), and the headline example `texture.set_compression_settings` declares its real spec (`TextureHandler.cpp:2813-2817`: `RPC_PARAM_REQ("assetPath")` + `RPC_PARAM_DEF("compressionSettings", …, "TC_Default")` + `RPC_PARAM_DEF("save", …, "true")`). The whole macro-registered family (`TextureHandler.cpp:2747-2985`) now passes real `RPC_PARAMS(...)`; `git grep RPC_NO_PARAMS` on this file returns 0 matches. Regression-covered by `Private/Tests/TestMaterialHandlers.cpp` (`HasParamSpec` helper + per-handler param assertions, e.g. set_compression_settings / set_texture_group). Tester to verify the texture wiki pages now render the params. Released claim.
