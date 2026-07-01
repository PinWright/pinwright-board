---
id: B-asset-dump-bpir-stale-generic-emit
title: "asset-dump bpir.txt serves stale non-recompilable generic-emit tokens (Resolve_Soft_Reference) — Jun-14 emit change didn't bump the bpir.txt aspect version"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset-dump, bpir, decompiler, aspect-version, stale-cache, convertasset, generic-emit]
encounters: 1
lastSeen: 2026-06-26T08:40:42Z
---

# asset-dump bpir.txt serves stale, non-recompilable generic-emit labels

The asset-dump cache's `bpir.txt` and live `blueprint.decompile` share **one**
emit path — `AssetDumpBuilder::BuildBpirText` (`Utils/AssetDumpBuilder.cpp:178-200`)
calls the same `FBpirDecompiler::DecompileGraph` as the live handler. There is no
separate dumper emitter, so the divergence is pure **cache staleness**, not a fork.

On 2026-06-14, commit `05c1f3bc` changed `FBpirTextEmitter::EmitPureNode` so any
pure `UK2Node` subclass with no specific handler routes to `EmitGenericNode` →
`call K2Node_<ClassName>(...)` (`Decompiler/BpirTextEmitter.cpp:2465-2477`; emit at
`:2523,:2572`), replacing the old title-derived fallback (`:2480-2505`, e.g.
"Resolve Soft Reference" → `Resolve_Soft_Reference`, "Equal (Enum)" → `Equal`).
The new `K2Node_` form recompiles (it routes through `EmitGenericK2NodeInstruction`);
the old title token does not — the resolver rejects it with
`Unresolved function: 'Resolve_Soft_Reference'`. That commit did **not** bump the
`bpir.txt` aspect version — it stayed `3` in `Handlers/Asset/AssetDumpCache.cpp:724`
before and after (verified by diffing `05c1f3bc^` vs `05c1f3bc`). So caches dumped
before Jun-14 record `bpir.txt: 3` (verified in the `W_LyraFrontEnd/.dumpcache.json`)
and the freshness check serves their stale `Resolve_Soft_Reference`/`Equal` bytes as
"fresh". An agent following the docs ("prefer reading the cache") copies that BPIR
into `compile_bpir`/`insert_bpir_at_node` and hits a hard compile failure; only a
re-run of live `blueprint.decompile` yields the recompilable
`K2Node_ConvertAsset`/`K2Node_EnumEquality`.

`Format_Text` is **not** this bug — it is the dedicated FormatText emitter's current,
recompilable output (the same cache already shows the NSLOCTEXT-synthesized form per
DONE `B-bpir-format-text-bare-string-roundtrip-break`); the live-`K2Node_FormatText`
claim for it is mistaken. Distinct from `B-bpir-exec-target-leads-pure-node` (OPEN), a
separate compiler exec-wiring bug that already uses the correct `call K2Node_ConvertAsset`
form.

**Workaround:** Re-dump (`asset.dump force=true`) or read live `blueprint.decompile`
instead of the cache — both emit the recompilable `call K2Node_<ClassName>(...)` form.
**Fix:** Bump `bpir.txt` 3 → 4 in the `Versions` table of `AssetDumpCache.cpp` (~line
724) so all pre-Jun-14 caches regenerate with the current emit — the aspect-version
bump that `05c1f3bc` omitted (plugin CLAUDE.md "Aspect Version Bumping"). Optional
follow-up: the title fallback at `BpirTextEmitter.cpp:2480-2505` still emits
non-recompilable tokens for the residual **non**-UK2Node pure-node case — narrow, out
of scope here.

## History
- `#1-initial-repro` `OPEN` reporter — Cached `Saved/EditorAutomation/asset-dumps/App/App/UI/LobbyAndMenu/W_LyraFrontEnd/bpir.txt:313` emits `%n28: object<W_LoginOverlay_C> = call Resolve_Soft_Reference(Input: ...)` and `:264` `call Equal(A: %n2, B: EServerType::Demo)`; feeding `Resolve_Soft_Reference` to `blueprint.insert_bpir_at_node` failed with `[COMPILE_FAILED] Unresolved function: 'Resolve_Soft_Reference'`, while `K2Node_ConvertAsset` parsed past that line. Verified in source: asset-dump and live decompile share `FBpirDecompiler::DecompileGraph` (`AssetDumpBuilder.cpp:182,193`); current `EmitPureNode` routes pure UK2Nodes to `EmitGenericNode` (`BpirTextEmitter.cpp:2474-2476`) emitting `call <Node->GetClass()->GetName()>` (`:2523,:2572`) = `K2Node_ConvertAsset`; the old title fallback (`:2480-2505`) produced `Resolve_Soft_Reference`. Route-to-generic landed in commit `05c1f3bc` (2026-06-14) which left `{ TEXT("bpir.txt"), 3 }` unchanged (`AssetDumpCache.cpp:724`), and the on-disk `.dumpcache.json` records `bpir.txt: 3` with the stale token — so the freshness check serves the un-recompilable form. `Equal` is the same root cause (title "Equal (Enum)" → strip parens → `Equal`); `Format_Text` is unrelated (dedicated, recompilable emitter). Root cause is the missing aspect-version bump; fix is `bpir.txt` 3 → 4. Distinct from `B-bpir-exec-target-leads-pure-node` (compiler exec-wiring on the already-correct `K2Node_ConvertAsset` form).
- `#2-fix` `IN-REVIEW` developer — Root-cause fix as scoped: bumped the `bpir.txt` aspect version `3 → 4` in the `Versions` table of `Handlers/Asset/AssetDumpCache.cpp:725` — the bump commit `05c1f3bc` omitted when it routed pure UK2Nodes through `EmitGenericNode`. Now `GetAspectVersion("bpir.txt")` returns 4, so `MakeCurrentAspectVersions`/`AreDumperFingerprintsEqual` mark every pre-Jun-14 cache (recorded `bpir.txt:3`) as stale → it re-dumps with the recompilable `call K2Node_<Class>(...)` emit instead of serving `Resolve_Soft_Reference`/`Equal`. Regression test `PinWright.AssetDumpCache.BpirAspectVersion` (`Tests/Utility/TestAssetDumpCache.cpp`) asserts the explicit/current aspect version is 4 and that a record carrying `bpir.txt:3` reads STALE through production `IsCacheFresh` — it fails if the bump is reverted to 3. Out-of-scope residual non-UK2Node title fallback left untouched per the ticket.
