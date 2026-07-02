---
id: E-describe-metasound-interface-vertices-unflagged
title: "describe_metasound lists the auto-attached MetaSoundSource interface vertices alongside user-added inputs/outputs with no flag distinguishing them, so 'how many outputs did I add' needs manual filtering"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [metasound, audio, authoring, describe_metasound, readback, interface, disambiguation, docs]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# describe_metasound does not flag which graph vertices are auto-created interface members vs user-added

`audio.authoring.describe_metasound` returns the MetaSound graph snapshot
(`MetaSoundDumpBuilder::BuildMetaSoundJson`) including the `rootGraph` interface
inputs/outputs. A `UMetaSoundSource` is created with the standard source
interfaces already attached, so its interface outputs/inputs include built-in
vertices the caller never authored — in this task's `MS_AmbientWind` build the
described graph listed **three built-in interface vertices**
(`UE.Source.OnPlay`, `UE.Source.OneShot.OnFinished`,
`UE.OutputFormat.Mono.Audio:0`) interleaved with the user-added outputs in the
same list, with **no `source` / `isInterfaceMember` / `isBuiltin` boolean** to
tell them apart.

The consequence is that any "confirm my output layout" readback — the documented
verification step (`audio.authoring.md`: "Use `describe_metasound` after wiring
to verify the graph snapshot") — forces the caller to **manually recognize and
subtract the three interface vertices** to answer "exactly one user output named
`Out`?". The caller has to already know the literal names of the auto-attached
source-interface members (`UE.Source.OnPlay`, `UE.Source.OneShot.OnFinished`,
`UE.OutputFormat.Mono.Audio`) to filter them out — knowledge that is not in the
`audio.authoring.metasound_gotchas` page or the `audio.authoring.md` overlay.

This is the *content/labeling* friction in the describe readback, distinct from
the *size* friction tracked by
[`E-describe-metasound-no-compact-mode`](E-describe-metasound-no-compact-mode.md)
(OPEN — the same payload also overflows the 10000-char display limit and spills
to disk). Even with a compact mode, the compacted list would still interleave
interface and user members unflagged; this ticket is the orthogonal "which of
these did I add" question.

## What it should do (downstream fix, not mine)

Either of (a docs touch covers the discoverability half regardless):

- **Structural (preferred):** tag each `rootGraph` interface input/output entry
  with a boolean/source field — e.g. `isInterfaceMember: true` (and ideally the
  owning interface name, `UE.Source` / `UE.OutputFormat.Mono`) for the
  auto-attached vertices, so a caller can count user-added members by filtering
  `isInterfaceMember == false` instead of pattern-matching `UE.*` names. The
  builder already knows the attached interfaces (`list_metasound_interfaces`
  reads them), so the membership is derivable at dump time.
- **Docs:** name and update the `docs/wiki-src/audio.authoring.md` overlay
  (`describe_metasound` section + the `audio.authoring.metasound_gotchas` page)
  to state that a freshly created `UMetaSoundSource` already carries the standard
  source interface, so `describe_metasound` lists `UE.Source.OnPlay`,
  `UE.Source.OneShot.OnFinished`, and `UE.OutputFormat.Mono.Audio` among the
  interface vertices, and a "count my outputs" check must exclude them.

## Evidence

This task (focus `audio.authoring.remove_metasound_output`, build
`MS_AmbientWind`, outcome **clean** — every functional call first-try, judge
filed nothing on the outcome). The task's final verification (story step 8) was
"confirm the final graph has exactly one output named 'Out'" — which the agent
could only satisfy by distinguishing user-added outputs from interface members.
Friction note, verbatim:

> "each describe lists the 3 built-in MetaSoundSource interface vertices
> (UE.Source.OnPlay, UE.Source.OneShot.OnFinished, UE.OutputFormat.Mono.Audio:0)
> alongside user outputs, so 'exactly one output' required distinguishing
> user-added from auto-created interface members."

The call log shows `describe_metasound` invoked three times (after-adds verify,
post-remove verify, final-state confirm); every one carried the unflagged
interface vertices that the caller had to mentally filter to make its
output-count assertion.

Fully recoverable (the caller knew the names this time), hence Low severity — but
every MetaSound-source author who uses the documented `describe_metasound`
verification to confirm an input/output layout pays this manual-filter tax, and a
caller who does NOT know the three built-in vertex names would miscount
silently.

## History
- `#2-fix` `IN-REVIEW` developer — Implemented the preferred structural fix: `MetaSoundDumpBuilder::BuildClassInputJson`/`BuildClassOutputJson` now tag each `rootGraph.interface` input/output with `isInterfaceMember` (and, when true, the owning `interfaceName`). Membership is derived at dump time from the document's attached-interface set (`Doc.Interfaces`) resolved against the registry via `PinWright::MetaSound::FindAllFrontendInterfaces()` — the same source the MSIR decompiler already reads — so a caller filters `isInterfaceMember == false` to count user-added I/O instead of pattern-matching `UE.*` names. Fields are omitted (not guessed) on engines without the interface registry. Since `describe_metasound` and the `asset.dump` `metasound.json` sidecar share this builder, both gain the field together (parity preserved); bumped the `metasound.json` aspect to v2 in `AssetDumpCache.cpp` so stale caches regenerate. Also did the docs half: named the auto-attached source vertices and the `isInterfaceMember == false` count rule in `audio.authoring.md` (describe_metasound) and `audio.authoring.metasound_gotchas.md`. Files: `Source/PinWright/Private/Handlers/Asset/MetaSoundDumpBuilder.cpp`, `Source/PinWright/Private/Handlers/Asset/AssetDumpCache.cpp`, `Docs/wiki-src/audio.authoring.md`, `Docs/wiki-src/audio.authoring.metasound_gotchas.md`. Test: `PinWright.Assets.MetaSound.DumpBuilder.FlagsInterfaceMembers` in `Source/PinWright/Private/Tests/Assets/TestMetaSoundDumpBuilder.cpp` builds a transient Source with `UE.OutputFormat.Mono` attached + a user output, runs `BuildMetaSoundJson`, and asserts the user output is flagged `isInterfaceMember=false` while an auto-attached interface output is flagged true with a non-empty `interfaceName` (fails if the membership fields are dropped).
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the clean `audio.authoring.remove_metasound_output` fuzz task (built MS_AmbientWind: 2 Float inputs + 3 Audio outputs → removed DebugMono+OutRight → renamed OutLeft→Out → validate; every functional call first-try, judge filed nothing on outcome). `describe_metasound` (`MetaSoundDumpBuilder::BuildMetaSoundJson`) lists the auto-attached MetaSoundSource interface vertices (`UE.Source.OnPlay`, `UE.Source.OneShot.OnFinished`, `UE.OutputFormat.Mono.Audio:0`) interleaved with user-added inputs/outputs in `rootGraph`, with no `isInterfaceMember`/`source` flag, so the documented "verify the graph snapshot after wiring" readback forces the caller to manually recognize+subtract the three built-in vertices to answer "exactly one user output named Out?" — and the built-in vertex names are not documented in `audio.authoring.metasound_gotchas` or the `audio.authoring.md` overlay. Distinct from `E-describe-metasound-no-compact-mode` (OPEN — the *size* of this same readback; this is the *labeling* of its members, orthogonal: a compact list would still interleave them unflagged). Proposes either an `isInterfaceMember` boolean (+ owning-interface name) on each interface vertex, or a docs note on `audio.authoring.md` / `audio.authoring.metasound_gotchas` naming the three auto-attached source-interface vertices a "count my outputs" check must exclude. Low severity — recoverable when the caller knows the vertex names; a caller who does not would silently miscount; taxes every MetaSound-source author's routine layout-confirmation readback.
