---
id: B-bpir-warn-marker-not-newline-separated
title: "asset.dump's bpir.txt emits `}# BPIR_WARN: …` on one line — the graph's closing brace and the warning share a line, so a line-oriented consumer loses both"
status: OPEN
severity: Medium
category: bug
tags: [asset-dump, bpir, bpir-txt, BPIR_WARN, text-emitter, newline, parseability, weapons]
encounters: 1
lastSeen: 2026-09-05T00:00:00Z
---

# The graph terminator and the warning are the same line, and neither survives a line filter

`bpir.txt` is a line-oriented text sidecar: a graph body ends with `}` on its own line, and a
diagnostic is a `#`-prefixed comment line. In at least one emit path those two are concatenated
without a separator.

## What was called

```
asset.dump {assetPath: "/Game/FPS/Weapons/BP/BP_WeaponBase"}
```

## What came back — measured

`bpir.txt:335`:

```
}# BPIR_WARN: <warning text>
```

Deterministic — reproduced on `BP_WeaponBase` on every dump of the asset this round.

## Why it matters

Both halves are lost to the tools that read this file:

- A consumer counting graph terminators (`grep -c '^}'`, or any brace-matched split into per-graph
  blocks) does not see this `}` — so the graph it closes runs on into the next one, and every
  subsequent block boundary in the file is off by one.
- A consumer collecting diagnostics (`grep '^# BPIR_WARN'`, the natural pattern given every other
  marker line starts at column 0) does not see this warning — so a dump with warnings audits as
  clean.

The two failures are independent, and the second is the more dangerous: the warning block is the
mechanism several other BPIR tickets rely on as their fallback remedy (`B-bpir-disabled-nodes-emitted-as-live`
ask 3, `B-bpir-orphan-warning-diagnostics-lossy`), so a marker that hides from an anchored grep
undermines fixes that have already shipped.

## What is asked for

1. Terminate the graph body with a newline before emitting any warning marker, so every
   `# BPIR_WARN:` / `# BPIR_ERROR:` line starts at column 0 like the rest.
2. A regression test asserting that no line in a generated `bpir.txt` contains `# BPIR_` at a
   non-zero column — this catches the whole class rather than this one site.

## Root cause — guess, no source read taken

The warning-marker emitter (`AssetDumpBuilder::FormatBpirWarningMarkers`, named by
`E-asset-dump-bpir-rename-failed-to-warn`'s fix, called from `BuildBpirText`) most likely appends its
marker string to a buffer whose preceding graph body was written without a trailing newline — the
markers themselves are almost certainly fine, and the missing `\n` belongs to the body writer just
before the call. **Inference from the emitted text; no plugin source was opened for this ticket and
no `file:line` in the plugin is claimed.** The only citation here is the output line,
`bpir.txt:335`.

## Severity

**Medium.** Soft blocker: a consumer can recover with an unanchored `grep 'BPIR_WARN'` once it knows,
but nothing in the file advertises that the anchored form is unsafe, and the natural reading is that
the dump has no warnings. It is not High because the text is present and correct — only its
placement is wrong, so no information is destroyed, and the failure is visible the moment anyone
looks at the raw line.

## Related

- `E-asset-dump-bpir-rename-failed-to-warn` (DONE) — introduced the `BPIR_WARN:` / `BPIR_ERROR:`
  split this ticket is about the formatting of. Its own verification grepped for the marker text
  rather than for an anchored line, which is why this would not have been caught there.
- `B-asset-dump-writes-crlf` — the other line-ending defect in the same sidecar family; a fixer
  touching the dump text writers should read both.
- `B-bpir-orphan-warning-diagnostics-lossy` (DONE), `B-bpir-disabled-nodes-emitted-as-live` (OPEN) —
  both depend on the warning block being reliably greppable.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 3. `asset.dump` on `/Game/FPS/Weapons/BP/BP_WeaponBase` writes `bpir.txt` whose line **335** reads `}# BPIR_WARN: …` — the graph body's closing brace and the warning comment on one line with no newline between them. Deterministic across repeated dumps of the asset. Both halves are lost to line-oriented consumers: a brace-matched or `grep -c '^}'` split does not see the terminator, so every later graph boundary in the file shifts; and `grep '^# BPIR_WARN'` — the anchored form the rest of the file justifies, since every other marker starts at column 0 — does not see the warning, so a dump with warnings audits clean. The second failure undermines shipped work: the warnings block is the fallback remedy several other BPIR tickets rely on. Ask: emit a newline terminating the graph body before any marker, plus a regression test that no generated `bpir.txt` contains `# BPIR_` at a non-zero column (catching the class, not the site). Root cause is a **guess**: the marker emitter `FormatBpirWarningMarkers` is probably fine and the missing `\n` belongs to the body writer immediately before it — inferred from the emitted text, no plugin source opened, no plugin `file:line` claimed; the only citation is the output line `bpir.txt:335`. Severity **Medium** on the soft-blocker band — recoverable with an unanchored grep once known, but nothing warns that the anchored form is unsafe; not High because no information is destroyed, only misplaced.
