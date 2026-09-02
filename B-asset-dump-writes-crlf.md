---
id: B-asset-dump-writes-crlf
title: "Asset-dump sidecars are written with platform-native CRLF, so the committed mirror is OS-dependent and every git pass over it emits one normalization warning per file"
status: OPEN
severity: Medium
category: bug
tags: [asset-dump, dump-folder, determinism, crlf, line-endings, git, tooling-noise]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---

# Asset-dump sidecars are written with platform-native CRLF

Every text sidecar the dumper writes (`meta.json`, `properties.json`, `bpir.txt`,
`tree.xml`, …) lands on disk with **CRLF** on Windows. The bytes come from
`SortedJsonWriter` / the BPIR emitters through
`FFileHelper::SaveStringToFile` (`Source/PinWright/Private/Utils/AssetDumpWriter.cpp:128`
for the text batch, `:265` for the transactional overload), and UE's
`TPrettyJsonPrintPolicy` terminates lines with `LINE_TERMINATOR`, which is
`\r\n` on Windows and `\n` elsewhere. Nothing on the write path normalizes.

Measured on the PDS mirror (2026-09-02, 30,807 dump dirs):
`asset-dumps/App/App/B_DroneExperience/meta.json` — 338 bytes, CR 15, LF 15,
CRLF 15; `properties.json` and `bpir.txt` samples are the same shape.

## Two consequences

**1 — the output is not platform-deterministic.** The same asset dumped on
Windows and on Linux/macOS produces different bytes for identical content. That
contradicts the tree's whole premise: the dump root's own seeded `CLAUDE.md`
calls it derived data whose "diffs between sweeps are meaningful", and the board
already carries a determinism programme for this tree
(`B-asset-dump-tier3-determinism-backlog`, `B-sortedjson-not-enforced-on-disk-writes`,
`B-asset-dump-preview-png-nondeterministic`). Line endings are the one axis none
of them covers.

**2 — the dumper fights the git scaffold it seeds itself.**
`AssetDumpWriter::EnsureDumpRootScaffold` (`AssetDumpWriter.cpp:113`) seeds a
`.gitattributes` reading:

```
# Machine-generated dump tree: bytes on disk are canonical; never normalize line endings.
* -text
```

so the plugin's own default is "commit whatever bytes the platform produced" —
which makes the mirror's committed content OS-dependent by design. The PDS host
overrode that file to the opposite policy (`* text=auto eol=lf linguist-generated=true`,
in the mirror since outer-repo commit `92d87494fe`). With `core.autocrlf=true` and
that LF pin, git normalizes on the way in, so the **committed** mirror does not
churn — but every `git add` / `git diff` over a freshly re-dumped subtree emits one
`CRLF will be replaced by LF` line per file.

Session evidence (2026-09-02): after a forced full re-dump of `/Game` + `/App`,
`git diff --stat asset-dumps/` produced roughly **6.4 MB** of normalization
warnings and was unusable for reading the diff; characterising the change had to
fall back to `git status --porcelain -uall`. A single-asset re-dump of
`W_LyraFrontEnd` re-emitted `meta.json` and `properties.json` with only the line
endings changed.

Note the seeded default makes this worse, not better: on a host that keeps
`* -text`, the CRLF bytes are committed verbatim, and the first sweep run from a
non-Windows machine rewrites every file in the mirror.

**Workaround:** pin `* text=auto eol=lf` in the dump root's `.gitattributes` (as
PDS did) and read diffs through `git status --porcelain -uall` instead of
`git diff --stat`. Neither stops the per-file warnings.

**Fix:** normalize to `\n` on the dump write path — a single
`Content.ReplaceInline(TEXT("\r\n"), TEXT("\n"))` before each
`SaveStringToFile` in `AssetDumpWriter.cpp` covers every text aspect at one
chokepoint, and makes the on-disk bytes identical on every platform. Then change
the seeded `.gitattributes` from `* -text` to an LF pin so the scaffold agrees
with the writer (`TestAssetDumpWriter.cpp:714` asserts the current `* -text`
string and must be updated with it). Every aspect's serialized bytes change, so
bump the `GetAspectVersion` table entries (`AssetDumpCache.cpp`) in the same
commit — a one-off full re-dump, after which diffs stay clean.

## See also
- `E-stop-generating-rpc-reference-md` (DONE) — same family: a committed
  plugin-generated file churning on EOL every session, resolved by deleting the
  writer. This one cannot be deleted; the mirror is the product.
- `E-asset-dump-folder-cache-miss-reason-opaque` — its `#1` already records
  "mass CRLF-warning churn in git" as a side effect, without attributing it.

## History
- `#1-initial-repro` `OPEN` reporter — Dump sidecars are written CRLF on Windows via `FFileHelper::SaveStringToFile` (`AssetDumpWriter.cpp:128`, `:265`) with `TPrettyJsonPrintPolicy`'s platform `LINE_TERMINATOR` and no normalization; measured `asset-dumps/App/App/B_DroneExperience/meta.json` = 338 B, 15 CRLF, same for `properties.json`/`bpir.txt`. Two effects: output is not platform-deterministic (same asset, different bytes on Windows vs Linux), and the writer contradicts the `.gitattributes` the dumper itself seeds (`* -text`, `AssetDumpWriter.cpp:113`), which the PDS host overrode to `* text=auto eol=lf` (`92d87494fe`). With the LF pin the committed mirror does not churn, but every git pass emits one `CRLF will be replaced by LF` per file: a forced full re-dump of /Game + /App on 2026-09-02 made `git diff --stat asset-dumps/` unusable behind ~6.4 MB of warnings, forcing a fallback to `git status --porcelain -uall`. severity rationale: impact=Medium — no wrong dump data and the sweep succeeds, but reading the resulting diff is doable only via a documented workaround, and the platform-dependent bytes are a determinism hole in a tree whose contract is byte-stable diffable output × reach=asset.dump/dump_folder are common but a full mirror refresh is not every session, so no modifier -> Medium. Fix: normalize to `\n` at the two `SaveStringToFile` chokepoints, change the seeded `.gitattributes` to an LF pin (and `TestAssetDumpWriter.cpp:714` with it), bump every `GetAspectVersion` entry in the same commit.
- `#2-still-present-at-upstream-head` `OPEN` reporter — Re-verified against plugin HEAD `347826a6` after the 398-commit pull from `b16f0f2b`. **Defect unchanged**; status stays `OPEN`/`Medium`, `encounters` unchanged (source re-read). Line numbers shifted, mechanics did not. The three text-write chokepoints all still hand `FFileHelper::SaveStringToFile` a string carrying whatever `LINE_TERMINATOR` produced: the scaffold write (`Source/PinWright/Private/Utils/AssetDumpWriter.cpp:129`), the transactional `WriteFileAtomic` (`:292-294`), and the batch path `AddPrepared(File.Name, EncodeUtf8(StripTrailingLineWhitespace(File.Content)))` (`:377`). The only content transform on that path is `StripTrailingLineWhitespace` (`:195-219`), which chops spaces/tabs *before* a `\r` or `\n` and then appends the terminator character unchanged (`:199-211`) — it explicitly preserves CR, so it is not a normalizer and adding one is still unimplemented. The seeded scaffold is byte-identical: `EnsureDumpRootScaffold`'s `ScaffoldFiles` table still emits `.gitattributes` = `"# Machine-generated dump tree: bytes on disk are canonical; never normalize line endings.\n" "* -text\n"` (`:114-116`), create-if-missing at `:121-124`. The test still pins the old string: `TestAssetDumpWriter.cpp:741-742` asserts `GitattributesContent.Contains(TEXT("* -text"))` (ticket cited `:714`). Two upstream commits in the pulled range mention line endings and neither touches this path — `f1642031` "Declare the line-ending and binary policy the index already has" and `b88bb24e` "Pin the MCP instructions file to LF newlines" both edit only the **plugin repo's own** `.gitattributes`, not the dump writer or the seeded dump-root scaffold. `git log -S'\r\n' -- Utils/AssetDumpWriter.cpp` is empty. Fix as filed still applies, with the three chokepoints above (not two) and the corrected test line.
