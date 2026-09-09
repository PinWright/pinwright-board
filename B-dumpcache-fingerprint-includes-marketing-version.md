---
id: B-dumpcache-fingerprint-includes-marketing-version
title: "`.dumpcache.json` folds the .uplugin VersionName into the dumper fingerprint, so any plugin version bump invalidates the entire committed dump mirror and turns the next incremental sweep into a full 22k-asset reload"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset-dump, dump-folder, dumpcache, fingerprint, cache-invalidation, plugin-version, churn]
encounters: 1
costly: 1
lastSeen: 2026-09-09T07:55:00Z
---

# A marketing version string invalidates every cache record in the mirror

The dumper fingerprint stored in each `.dumpcache.json` includes `PluginVersion`, and
`PluginVersion` is the plugin descriptor's marketing `VersionName`:

```
X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Asset\AssetDumpCache.cpp:481-486
    return A.DumpCoreVersion == B.DumpCoreVersion
        && A.EngineVersion   == B.EngineVersion
        && A.PluginVersion   == B.PluginVersion
        && AreAspectVersionsEqual(A.AspectVersions, B.AspectVersions);

AssetDumpCache.cpp:487-491
    FString GetPluginVersion()
    {
        TSharedPtr<IPlugin> Plugin = IPluginManager::Get().FindPlugin(TEXT("PinWright"));
        return Plugin.IsValid() ? Plugin->GetDescriptor().VersionName : TEXT("unknown");
    }
```

`VersionName` is bumped for release/Fab reasons that have nothing to do with the dump
format. Every bump therefore invalidates **every** `.dumpcache.json` in the committed
mirror at once, and the next "incremental" sweep is a full reload of the tree — for
`/Game` that is 22,446 packages, which is exactly the condition that OOM-kills the
editor (`B-dump-folder-sweep-never-gcs-ooms-editor`). The three fields that actually
describe the output — `DumpCoreVersion`, the per-aspect versions, and `EngineVersion` —
already cover every case where a re-dump is genuinely required; `fa755a4f` shows the
intended mechanism working (`properties.json 7->8`, `scs.json 4->5`, `bpir.txt 8->9`,
`dumpCoreVersion 2->3`), which is precisely why the marketing string adds nothing.

Consequences, in order of cost:

1. A routine version bump silently converts a cheap sweep into the most expensive
   operation the plugin has, on a project where that operation currently crashes.
2. The committed mirror churns wholesale — thousands of byte-identical files rewritten,
   because content-identical output still gets a new cache record.
3. The invalidation is indistinguishable from a cache bug at the call site (that half is
   tracked separately as `E-asset-dump-folder-cache-miss-reason-opaque`).

**Fix:** drop `PluginVersion` from `FAssetDumpDumperFingerprint` (and from the written
record, or keep it as an informational field that `AreDumperFingerprintsEqual` ignores).
Fingerprint on dump-core + aspect schema versions and `EngineVersion` only. Any change
that alters dump output already has to bump `DumpCoreVersion` or an aspect version to be
correct — that is the contract to enforce, not the release number. If a belt-and-braces
signal is wanted for unreleased builds, use the plugin binary's build id rather than the
descriptor's `VersionName`, and only in non-shipping configurations.

**Workaround:** none inside the tool. Externally: re-sweep in subtrees after a version
bump and accept the mirror churn.

Related, not duplicates:
- `E-asset-dump-folder-cache-miss-reason-opaque` (OPEN) — same mechanism, but its ask is
  *telemetry* (`staleReasonCounts` in the response) and it explicitly rules the
  invalidation itself "NOT a bug, by design". This ticket disputes that design point;
  the two fixes are independent and both are worth having.
- `B-dumpcache-staleness-loop` (IN-REVIEW) — records that are *never written* (dirty
  self-compile veto, load-failed stubs). This is about records that are written and then
  thrown away.

## History
- `#1-version-bump-forced-a-22k-reload` `OPEN` reporter — Filed from the 2026-09-09 `/Game` sweep on `X:\src\unreal\unreal-fpv-dev` (UE 5.8, plugin `fa755a4f`): the committed mirror was current, but the plugin version had moved since it was written, so `force`-less sweeps had nothing to hit and the full 22,446-asset reload had to be run — which then OOM-killed the editor twice (`B-dump-folder-sweep-never-gcs-ooms-editor`). Source-verified the fingerprint composition and the `VersionName` read at `AssetDumpCache.cpp:481-491`. Marked `costly` because the reload it forced is what consumed the two crashed sweeps; the invalidation itself is cheap only when the tree is small. Distinct from `E-asset-dump-folder-cache-miss-reason-opaque` (which asks for the reason to be *reported*, and calls the behaviour correct) and from `B-dumpcache-staleness-loop` (records never written at all). Filed `Medium`: no wrong data is produced, the cost is wasted editor work on a method that runs in most sessions — but on a `/Game`-sized tree it is the trigger for a crash-class ticket, which is why it is not `Low`.
- `#2-plugin-version-dropped-from-the-fingerprint` `IN-REVIEW` developer — Fixed as proposed, on `origin/master` as `b694c7a4` ("Stop keying the dump cache on the marketing plugin version"). `AreDumperFingerprintsEqual` (`Source/PinWright/Private/Handlers/Asset/AssetDumpCache.cpp`) now compares `DumpCoreVersion`, `EngineVersion` and the per-aspect schema versions only; the `PluginVersion` comparison is gone. `FAssetDumpDumperFingerprint::PluginVersion` (`AssetDumpCache.h`) is kept and still written/read so a record stays self-describing about which build wrote it, and is documented as informational; `force: true` remains the way to re-dump on demand. Rationale recorded in a comment at the comparison site: any change that alters a sidecar must already bump `DumpCoreVersion` or an aspect version to be correct, which is the contract to enforce, whereas the descriptor `VersionName` moves for release reasons that never change output. Test coverage: `PinWright.AssetDumpCache.VersionMismatch` gained an assertion that a fingerprint differing **only** in `PluginVersion` stays fresh (`Source/PinWright/Private/Tests/Utility/TestAssetDumpCache.cpp`); counterfactual — restore the `A.PluginVersion == B.PluginVersion` term and that assertion fails while the rest of the group still passes. Suite: `PinWright.AssetDumpCache.*` all green in the 221-test dump run (216 pass, 5 pre-existing unrelated failures). **Not verified live:** no version bump has been performed since the fix, so the end-to-end "bump the .uplugin and confirm the mirror is not invalidated" check is left to the tester.
