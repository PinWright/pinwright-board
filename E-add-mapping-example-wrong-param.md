---
id: E-add-mapping-example-wrong-param
title: "input.add_mapping wiki code example uses 'imcPath' but the param is 'contextPath' — copy-pasting the example errors"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [input, enhanced-input, wiki, docs, param-name]
---

# `input.add_mapping` wiki example contradicts its own Parameters list

The wiki page for `input.add_mapping` documents the IMC argument two ways
that disagree with each other:

- The **Parameters** list (and the actual handler) require `contextPath`:
  > `contextPath` (`string`, required): Object path to the UInputMappingContext asset.
- The **code Example** in the same page uses `imcPath` instead:
  ```
  call("input.add_mapping", {
    "imcPath": "/Game/Input/IMC_Default.IMC_Default",
    "actionPath": "/Game/Input/IA_Jump.IA_Jump",
    "key": "SpaceBar"
  })
  ```

A caller who copy-pastes the documented example (the natural thing to do)
hits a hard error, because `imcPath` is not the real parameter name. The
example is the highest-trust part of the page and it is wrong.

**Why it matters:** the example is verbatim broken. The only signal the
caller gets is `[MISSING_REQUIRED_PARAM] Missing required parameter
'contextPath'`, which does not mention `imcPath` at all — so the caller has
to notice the doc's example key differs from the doc's own Parameters list
to recover. Worked around easily here, but it is a guaranteed first-try
failure for anyone following the example.

**Fix:** change the `imcPath` key in the `input.add_mapping` code example
(in `docs/wiki-src/input.md`, which generates the wiki page) to
`contextPath` so the example matches the Parameters list and the handler.

## Repro

1. `input.create_input_mapping_context({name: "IMC_Replay", path: "/Game/Input"})`
   → success.
2. `input.create_input_action({name: "IA_ReplayMove", path: "/Game/Input"})`
   → success.
3. Replay the documented example shape:
   `input.add_mapping({imcPath: "/Game/Input/IMC_Replay.IMC_Replay",
   actionPath: "/Game/Input/IA_ReplayMove.IA_ReplayMove", key: "SpaceBar"})`
   → `[MISSING_REQUIRED_PARAM] Missing required parameter 'contextPath'
   (type: string)`.
4. Same call with the Parameters-list key `contextPath` instead of `imcPath`
   → succeeds (returns `contextPath`/`actionPath`/`key` echo plus
   `contextVerification` and `actionVerification`).

## History
- `#1-initial-repro` `OPEN` reporter — Realism-mode Enhanced-Input authoring task; the attempt agent flagged the doc contradiction as friction. Replayed live: the wiki `input.add_mapping` code Example uses key `imcPath`, but the handler rejects it with `[MISSING_REQUIRED_PARAM] Missing required parameter 'contextPath'`; the same call with `contextPath` (the Parameters-list name) succeeds. Confirmed self-contradiction between the page's Example and its Parameters list. No existing input/add_mapping ticket on the board (ripgrep clean for `imcPath` and `add_mapping`). Fix: correct the example key in `docs/wiki-src/input.md`.
- `#2-fix` `IN-REVIEW` developer — All three validity lenses (correctness/adversarial/historian) confirmed: handler reads only `contextPath` (`Source/EditorAutomationRpcGateway/Private/Handlers/Input/InputHandler.cpp:102,107`), no `imcPath` alias exists anywhere in Source, and the overlay example used `imcPath` (`Docs/wiki-src/input.md:22`). Fix: changed the `input.add_mapping` code-Example key `imcPath` -> `contextPath` in `Docs/wiki-src/input.md` so the example matches the auto-generated Parameters list and the handler. No code change (docs-only ergonomic defect). Regression test: added `EditorAutomationRpcGateway.infra.wiki_handler.MethodPage.AddMappingExampleParam` in `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` — drives `WikiHandler::RenderPage("input.add_mapping")` (the real overlay render path via `WikiOverlay::LoadMethodSection`) and asserts the page contains `contextPath` and does NOT contain `imcPath`; it would fail if the example key were reverted. Followed the established overlay-edit + render-assert precedent (`FWikiHandlerAddSwitchDocumentsIfTest`, `E-add-switch-is-comparator-if-undocumented`).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. No body citation was rewritten here. 2 citations sit in history rows and are left verbatim per the append-only rule. The one citation is in history row `#2` and stays verbatim. Its map is a file move plus a line repair: `Handlers/Input/InputHandler.cpp:102,107` → `Source/PinWright/Private/Handlers/Input/InputHandler.cpp:136` (`RPC_PARAM_REQ("contextPath")`) and `:141` (`Ctx.GetString(TEXT("contextPath"))`), in the `input.add_mapping` handler registered at `:134`; `:102-107` at HEAD belongs to `input.create_context`. The row's load-bearing negative was re-verified and still holds — `imcPath` occurs tree-wide only in `Tests/Infra/TestWikiHandler.cpp:188-215`, which asserts its absence. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
