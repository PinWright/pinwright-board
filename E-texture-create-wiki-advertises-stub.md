---
id: E-texture-create-wiki-advertises-stub
title: "texture namespace prelude over-promises 'Create … texture arrays / volume/cube textures' with no namespace-page caveat that the multi-slice create family is a NOT_IMPLEMENTED stub"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [texture, wiki, docs, discovery, create, stub]
---

# texture namespace prelude over-promises the multi-slice create family

The `texture` namespace overview (`docs/wiki-src/texture.md:3`) opens by listing
"**Create** … render targets, **texture arrays**, **volume/cube textures**, …"
as a flat capability claim. A reader scanning the namespace prelude sees an
unqualified "Create" promise for arrays / volume / cube, even though those three
verbs (`texture.create_texture_array` / `create_cube_texture` /
`create_volume_texture`) build no asset.

## What is already true (so this ticket is now narrow)

Since the sibling C++ ticket `B-texture-create-placeholder-fake-success` landed
its fail-loud fix, the create family is **already largely self-documenting** —
the original premise of this ticket (a fake-success no-op + "documents no create
method, no param spec, no caveat") no longer holds:

- The three create verbs return `[NOT_IMPLEMENTED]` with an actionable workaround
  hint (`TextureHandler.cpp:2648-2682`), not a fabricated success — the
  multi-call diagnostic hunt the original report described no longer reproduces.
- Each create verb IS registered, so it appears in the auto-generated `## Methods`
  index on the namespace page, AND its registration summary already begins
  "Returns NOT_IMPLEMENTED: …" with the workaround (`TextureHandler.cpp:2944` /
  `:2949` / `:2956`). `WikiHandler` renders that summary into both the `## Methods`
  index (`WikiHandler.cpp:401-404`) and the per-method page (`:414`).
- The verbs declare param specs, each marked "(unused; method is not
  implemented)" (`TextureHandler.cpp:2946-2962`).

So a caller who navigates to the create method already hits an explicit
NOT_IMPLEMENTED caveat + workaround. The only residual gap is the **namespace
prelude itself**, which a reader scanning the overview reads before drilling into
any method — and it still presents create-for-arrays/volume/cube as a working
capability with no caveat at that level.

## Distinct from neighbours

- `B-texture-create-placeholder-fake-success` (IN-REVIEW) is the C++ honesty
  fix; this is the docs-overlay companion it explicitly deferred. It does not
  cover the prelude over-promise.
- `E-texture-action-handler-param-docs` (IN-REVIEW) is the `set_*` family's
  "Parameters: none" gap — orthogonal.

This is the same shape as the accepted overlay precedents
`E-ik-rig-family-wiki-advertises-compiled-out-workflow`,
`E-level-structure-wp-wiki-advertises-dead-end`, and
`E-set-transition-rules-wiki-overstates-rule-authoring`: the wiki overview
over-advertises a capability the runtime does not deliver; the fix is a
`docs/wiki-src/` overlay edit.

## What the wiki page should do

In `docs/wiki-src/texture.md`:
- Soften the prelude so it does not list arrays / volume / cube under an
  unqualified "Create" — keep the genuine create surface (render targets,
  procedural patterns).
- Add a namespace-page-visible `## ` section that states the multi-slice create
  family is NOT_IMPLEMENTED (builds no asset), points at the working
  import/assemble workarounds, and clarifies that `texture.describe` reading
  those classes is genuine (only the create verbs are stubs). Keep it below the
  prelude so it renders on the namespace page but not the root index.

## History
- `#1-initial-audit` `OPEN` reporter — Process/docs audit of task focus `texture.create_texture_array`. The wiki overview (`wiki-src/texture.md:3`) advertises "Create … texture arrays, volume/cube textures" and the describe section (`:7`) lists those classes, but the page documents NO create method and carries NO stub warning, so wiki navigation steered the caller straight into the fake-success no-op. Friction: after the fake success the caller burned `describe`+`asset.exists`+`asset.list`+a duplicate `asset.exists` recheck and finally a `TextureHandler.cpp:2683-2702` source-dive to confirm the no-op. Distinct from `B-texture-create-placeholder-fake-success` (C++ fake success) — this is the docs advertising the trap, and survives even the "fail-loud" fix of that bug. Not covered by `E-texture-action-handler-param-docs` (that is about the `set_*` family's "Parameters: none"). Names the page to edit: `docs/wiki-src/texture.md`.
- `#2-reword-and-overlay-caveat` `IN-REVIEW` developer — Reworded down to current reality: `B-texture-create-placeholder-fake-success`'s fail-loud fix already landed, so the create verbs now return `[NOT_IMPLEMENTED]` (`TextureHandler.cpp:2648-2682`) and their registration summaries ("Returns NOT_IMPLEMENTED: …", `TextureHandler.cpp:2944/2949/2956`) auto-render into the `## Methods` index and per-method pages — the "documents no create method / no param spec / no caveat / fake-success" premise and the multi-call friction repro are stale. Residual fixed: severity Medium→Low; the only live gap was the namespace prelude over-promising the multi-slice create family. Edited `docs/wiki-src/texture.md` — softened the prelude (dropped the unqualified "Create … texture arrays, volume/cube textures"; kept render targets + procedural patterns) and added a namespace-page-visible `## Create coverage` section flagging the create_cube/volume/array family NOT_IMPLEMENTED with the import/assemble workarounds and a note that `texture.describe`'s coverage of those classes is genuine. Regression test: `FWikiHandlerTextureDocumentsCreateStubsTest` in `Tests/Infra/TestWikiHandler.cpp` renders the `texture` namespace page via `WikiHandler::RenderPage` and asserts the overlay-exclusive markers `## Create coverage` and "describe coverage is genuine" (neither appears in any handler registration summary, so it fails iff the overlay section is reverted). No C++ behavior changed. Removed the `claimedBy`/`claimedAt` lease.
