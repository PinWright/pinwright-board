---
id: B-general-purpose-host-vocabulary
title: "Shipped docs and render tests still contain host-project vocabulary"
status: IN-REVIEW
severity: High
category: bug
tags: [general-purpose, portability, documentation, tests]
encounters: 1
---

# Shipped docs and render tests still contain host-project vocabulary

The plugin is the deliverable, but existing material outside the three-verb repair still names
one host project's factions, units, content root, and asset paths. The affected deliverable files
are `Docs/engine-research-2026-08-plugin-pass.md`, `Docs/pwmodel-design.md`,
`Docs/pwmodel-format.md`, `Docs/plans/floating-geometry-audit.md`, and
`Source/PinWright/Private/Tests/Render/TestAnimationCaptureHandlers.cpp`.

The prose can be made generic directly. The render test needs a plugin-owned or synthetic fixture;
renaming its strings alone would hide the dependency rather than remove it. The packaging guard,
its documentation, and the repository instructions also quote forbidden words in order to detect
or prohibit them and are not the leak described here.

## History
- `#1-existing-portability-leak-found` `OPEN` reporter — A whole-deliverable scan found host-specific prose in four docs and hard-coded host assets in one render test; left separate from the three reproduced verb fixes because the test needs a real general-purpose fixture.
- `#2-generic-docs-synthetic-render-and-repository-guard` `IN-REVIEW` developer — Replaced all four documentation leaks with neutral examples. The render regression now uses an engine-shipped neutral skinned mesh plus a runtime-registered synthetic 36-frame animation, so it no longer needs project content or a subtle canned animation. Root cause in `scripts/package-fab.ps1`: the vocabulary scan ran only on the release stage, while the manifest excludes the four main `Docs/` files and every `Private/Tests` tree. Added a repository-level scan with narrow guard/instruction exemptions and a failure fixture that writes the exact five formerly missed path classes and requires every path to be rejected. Verified the bundled-Python guard test (1/1), a clean live repository probe, full build `Result: Succeeded`, and scoped automation `659/659` succeeded with zero failures or skips.
