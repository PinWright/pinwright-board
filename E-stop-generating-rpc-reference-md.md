---
id: E-stop-generating-rpc-reference-md
title: "Stop generating Docs/rpc-method-reference.generated.md on editor startup"
status: DONE
severity: Low
category: ergonomic
tags: [docs, wiki, churn, ergonomic]
---

# Stop generating `Docs/rpc-method-reference.generated.md` on editor startup

`UEditorAutomationRpcGatewaySubsystem::Initialize` writes
`Docs/rpc-method-reference.generated.md` every editor launch via
`Catalog->WriteMarkdownReference(...)`
(`EditorAutomationRpcGatewaySubsystem.cpp:78-95`). The file is committed
to the plugin repo, so every session that opens the editor produces a
dirty working tree on this one file — almost always EOL-only churn
(LF→CRLF on Windows checkouts) with zero content delta. It shows up in
`git status` for every agent and every human, every time, and gets
amended into otherwise-unrelated commits as noise.

The information is already available through two live, lower-friction
channels:

- `call()` / `call("<namespace>")` / `call("<namespace.method>")` — the
  wiki-shaped MCP surface served by `wiki.get`. This is the documented
  agent-facing path (see plugin `CLAUDE.md`, the
  `using-editor-automation` skill, and the `mcp-usage-guide`).
- The RPC discovery protocol — `"?"`, `"namespace?"`,
  `"namespace.method?"` (per `arch.md` and `JsonRpc.h`). Used by clients
  that need machine-readable schemas.

Both regenerate from the same `FToolCatalog` data the markdown writer
reads from, so the committed `.generated.md` is a third copy of the
same source-of-truth that adds nothing the other two don't cover.

## Impact

- Every editor session dirties git on this file. Either it gets
  committed (no-op churn commits, noise in `git log`) or it gets
  stashed/ignored (extra friction on every workflow that touches the
  repo state, including `feedback_ignore_crlf` and `git diff` reviews).
- File is 1,200+ RPCs of generated text — bloats the plugin repo and
  every clone for content that's already in the binary.
- Discovery via `call(...)` is the documented surface; readers
  pointed at a committed `.md` may consume stale state when the editor
  isn't running, then act on outdated schemas.

## Fix

Two viable shapes:

1. **Delete the writer.** Remove the `WriteMarkdownReference` call
   block at `EditorAutomationRpcGatewaySubsystem.cpp:78-95`, delete the
   tracked `Docs/rpc-method-reference.generated.md`, and remove any
   doc cross-references that point at it (audit
   `Plugins/EditorAutomationRpcGateway/**/*.md` for the filename).
   Discovery stays via `wiki.get` and `"?"`. Simplest path; recommended.

2. **Keep the writer, stop tracking.** If there's a reason to retain
   an on-disk copy (offline browsing, IDE indexing), move the output
   under a gitignored path (`Saved/`, `Intermediate/`, or a top-level
   gitignored `.generated/`) and drop the file from the working tree.
   This still keeps a fresh local snapshot per session without
   polluting `git status`.

Prefer (1) — there's no current consumer of the on-disk file that
`call()` or `"?"` doesn't already cover, and the `WriteMarkdownReference`
implementation can stay around if anyone ever wants to re-enable it.

## History
- `#1-filed-from-commit-churn` `OPEN` reporter — Filed after a session where
  the file showed up dirty with EOL-only diff (`git diff -w` empty) again,
  triggering the recurring "how do we commit this" question. Same noise
  on every editor restart; no real content updates between sweeps unless
  RPC methods are added or removed.
- `#2-route-1-deleted-writer` `IN-REVIEW` developer — Implemented Route 1.
  Removed the `Catalog->WriteMarkdownReference(...)` call in
  `UEditorAutomationRpcGatewaySubsystem::Initialize` (and dropped the now-unused
  `Interfaces/IPluginManager.h` include), deleted `FToolCatalog::WriteMarkdownReference`
  declaration in `Catalog/ToolCatalog.h` and its implementation in
  `Catalog/ToolCatalog.cpp` (also dropped the `MarkdownHelpers.h`,
  `Misc/FileHelper.h`, `Misc/Paths.h`, `HAL/FileManager.h`, and
  `Handlers/HandlerRegistration.h` includes which were only used by that
  function; `MarkdownHelpers.h/.cpp` are kept — `WikiHandler` still uses them).
  Deleted the untracked `docs/rpc-method-reference.generated.md` working-tree
  file. Removed the file from `Config/FilterPlugin.ini` and from
  `scripts/release-manifest.json` (both `include` and `requiredFiles` arrays).
  Stripped cross-references from `README.md` (Documentation list),
  `docs/index.md` (Generated And Operational table), `docs/tags.md`
  (deleted the sole `generated-reference` row and removed the entry from
  the `rpc` row), `docs/SCHEMA.md` (directory layout and not-frontmatter
  exemption list), and `docs/arch.md` (rewrote the FToolCatalog bullet
  in the layer diagram to drop the writer mention while preserving the
  70-col ASCII box alignment). Discovery now goes through `call()` /
  `"?"` only.
- `#3-verify-fix` `DONE` tester — Verified: `ToolCatalog.h` no longer declares `WriteMarkdownReference` (file is 16 lines, just `Initialize`/`DispatcherWeak`); `ToolCatalog.cpp` is 11 lines with no writer body; `EditorAutomationRpcGatewaySubsystem.cpp:60-109` has no `WriteMarkdownReference` call and no `IPluginManager.h` include; `git status` shows `D Docs/rpc-method-reference.generated.md`; `Config/FilterPlugin.ini` and `scripts/release-manifest.json` diffs each drop the file entry (manifest drops it from both `include` and `requiredFiles`); `Docs/SCHEMA.md`, `Docs/index.md`, `Docs/tags.md` (both `generated-reference` row and `rpc` row), `Docs/arch.md` (layer-diagram rewrite preserves the 70-col box alignment), and `README.md` no longer reference the file. `grep` for `rpc-method-reference` outside `docs/board/` returns only `Plugins/EditorAutomationRpcGateway/CLAUDE.md:67` (stale architecture-overview sentence — separate scope, not claimed by this ticket).
- `#4-reconcile-status-frontmatter` `DONE` tester — Re-verified: `Docs/rpc-method-reference.generated.md` is absent from working tree and shows as `D` in plugin-repo `git status`; `ToolCatalog.h` (16 lines) and `ToolCatalog.cpp` (11 lines) contain no `WriteMarkdownReference` symbol; `EditorAutomationRpcGatewaySubsystem::Initialize` (lines 48-130) has `Catalog->Initialize(Dispatcher)` at L80 with no writer call after it and no `IPluginManager.h` include. `#3` already reported PASS but left `status: IN-REVIEW` in frontmatter; flipped to `DONE` to match the verified outcome.
