---
id: F-universal-agent-compiler
title: "Universal agent compiler: single-source skills + subagents for Claude and Codex"
status: DONE
severity: Medium
category: feature
tags: [skills, agents, tooling, codex, claude]
---

# Universal agent compiler

Skills and subagents previously lived in three parallel forms — real Claude `.md`, hand-rewritten Codex `.toml`, and NTFS junctions stitching the two together. The result drifted: the Codex `wave-worker.toml` body diverged from the Claude `wave-worker.md` and every edit had to be applied twice.

New `Plugins/EditorAutomationRpcGateway/.universal-agent/` tree owns the source. Authors write one Claude-flavored `.md` per skill/subagent with optional `<claude>` / `<codex>` macro blocks; `build.py` compiles per-target outputs from `build.toml`:

- Claude: `.universal-agent/skills/<name>/` → `<repo>/.claude/skills/<name>/`, `.universal-agent/agents/<name>.md` (+ sibling folder) → `<repo>/.claude/agents/<name>.md` (+ folder).
- Codex: same skill tree → `<repo>/.agents/skills/<name>/` with frontmatter shrunk to `name` + `description`; subagent `.md` → `<repo>/.codex/agents/<name>.toml` with body wrapped in `developer_instructions`, `model` dropped, `mcpServers` → `mcp_servers`, `effort` → `model_reasoning_effort`. Other Claude-only fields drop with a warning.

Macros (`<claude>…</claude>`, `<codex>…</codex>`) are processed only inside `.md` files; non-`.md` bundled assets copy byte-for-byte. Macros also support per-target frontmatter blocks at the top of a file when frontmatter must differ.

## Scope of this ticket

Milestone covers the existing five skills (`mcp-audit`, `mcp-review`, `mcp-sprint`, `mcp-test-loop`, `writing-wave-plan`) and the `wave-worker` subagent with its seven `phase-*.md` files. Other skills (`shortcut-audit`, `qmd`, `swagger-scan`, `ui-edit`, `wiki-sync`, etc.) and `*-workspace` eval scaffolding are deliberately out of scope.

## What landed

- New files: `.universal-agent/{build.toml, build.py, README.md, skills/<5>, agents/wave-worker.md, agents/wave-worker/phase-*.md}`.
- Migrated source: four `mcp-*` skill folders moved into `.universal-agent/skills/` (in-plugin `git mv`); `writing-wave-plan/`, `wave-worker.md`, and the seven `phase-*.md` files moved cross-repo from outer `.claude/{skills,agents}/` into `.universal-agent/`; outer `.codex/agents/wave-worker.toml` deleted (its hand-written "Codex compatibility note" paragraph intentionally not carried over).
- Junctions cleared: outer `.claude/skills/mcp-{audit,review,sprint,test-loop}`, outer `.codex/agents/wave-worker/`, plus the preexisting `.agents/skills/*` symlinks that pointed back at `.claude/skills/*` (they were causing the Codex output pass to clobber the Claude output).
- Gitignore updates: outer ignores all build outputs (`.claude/skills/mcp-*`, `.claude/skills/writing-wave-plan`, `.claude/agents/wave-worker*`, `.codex/agents/wave-worker.toml`; `.agents/` was already covered). Plugin ignores `.universal-agent/__pycache__/`.
- Build verified deterministic (second run writes 0 files), claude outputs byte-identical to source modulo CRLF→LF normalization, codex skills equivalent to Claude (no Claude-only frontmatter fields in the milestone set), generated `wave-worker.toml` parses cleanly with `name` / `description` / `developer_instructions`.

## What still needs verification

- Codex CLI: confirm the five skills + `wave-worker` subagent enumerate.
- Codex app: skills appear in the picker, `wave-worker` loads.
- Wave-worker round-trip: dispatch against a trivial task in both Claude Code and the Codex app; confirm matching phase-file flow.

## How to run

From `Plugins/EditorAutomationRpcGateway/.universal-agent/`:

```powershell
npx --yes polyskill                  # all targets
npx --yes polyskill --target claude  # one target
npx --yes polyskill --target claude --target codex
```

Build is overwrite-only — unchanged files are not rewritten, orphans from deleted sources are not auto-cleaned. Remove stale outputs by hand.

## Out of scope (explicit)

- Cursor / Gemini target kinds (architecture leaves room).
- Skills outside the milestone set; `*-workspace` eval scaffolding.
- Slash commands (`commands/`).
- Reverse compilation (Codex → source), watch mode, sidecar tracking, `--clean`.
- Macros inside non-`.md` files.
- Negation tags (`<except>` / `<not-claude>`).

## History
- `#1-initial-spec` `OPEN` reporter — Filed to track the universal-agent compiler work specified in `~/.claude/plans/lucky-munching-scott.md`. Replaces the prior three-form layout (Claude `.md`, hand-edited Codex `.toml`, junctions) that had already drifted on `wave-worker.toml` vs `wave-worker.md`.
- `#2-implemented-milestone` `IN-REVIEW` developer — Built `.universal-agent/build.{py,toml}` + README, migrated the five milestone skills and the `wave-worker` subagent into the source tree, removed obsolete junctions and the hand-written `wave-worker.toml`, updated outer + plugin `.gitignore` to mark build outputs as ignored. Build runs deterministic; claude/codex outputs both render cleanly on disk. Macro processor exercised live with a `<codex>` / `<claude>` round-trip. Approval to delete the preexisting `.agents/skills/*` symlinks pointing back at `.claude/skills/*` confirmed by user (they were causing the Codex output pass to clobber the Claude output before deletion). Pending: live verification in Codex CLI + Codex app + a wave-worker dispatch through both harnesses on a trivial task.
- `#3-verify-build-outputs` `DONE` tester — Verified: ran `uv run python -m build` from `.universal-agent/` twice — second run wrote 0 files (deterministic). Source tree has all five skills + `wave-worker.md` + `wave-worker/phase-{explore,fix,implement,quality-review,resolve,shortcut-audit,spec-review}.md`. Outputs landed at `.claude/skills/{mcp-audit,mcp-review,mcp-sprint,mcp-test-loop,writing-wave-plan}`, `.claude/agents/wave-worker{,.md}`, `.agents/skills/<five>`, `.codex/agents/wave-worker.toml`. Generated `wave-worker.toml` parses with Python `tomllib` and has `name` + `description` + `developer_instructions = """..."""` with `model` dropped as specified. Claude vs Codex `mcp-audit/SKILL.md` frontmatter identical (no Claude-only fields in milestone set, matching ticket claim). Outer `.gitignore` lists all five expected build-output paths (lines 91–98). Live Codex CLI/app enumeration is out of scope for an MCP-driven verifier — outside this harness's reach.
- `#4-migrated-to-polyskill` `DONE` developer — Replaced the in-tree `build.py` + `build.toml` with [polyskill](https://github.com/SSS135/polyskill), a standalone zero-dependency Node/`npx` compiler (public repo, MIT). polyskill output verified byte-identical to the Python tool via `diff -r` on both synthetic fixtures and this plugin's real skill sources (42 files, clean). Removed `build.py`, `build.toml`, and the `.universal-agent/__pycache__/` gitignore line; added `.universal-agent/polyskill.config.json` (source `.`, out `../../..`, claude+codex targets); updated the `.universal-agent/README.md` run/layout/output sections. Run with `npx --yes github:SSS135/polyskill` from `.universal-agent/` (switch to `npx polyskill@<version>` once published to npm).
