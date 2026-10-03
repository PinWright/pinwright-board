---
id: B-fixup-redirectors-double-counts-deleted
title: "asset.fixup_redirectors and asset.bulk_delete report redirectorsFixed/redirectorsDeleted as ObjectTools::DeleteObjects' object count, which counts each redirector AND its package, so 8 redirectors report as 16 ('Fixed 16 of 8 redirectors')"
status: OPEN
severity: Medium
category: bug
tags: [asset, fixup-redirectors, bulk-delete, redirector, objecttools, wrong-count, response-honesty]
encounters: 1
lastSeen: 2026-09-30T12:00:00+05:00
rice: [2, 2, 1, 1]
priority: 33
---

`Utils/RedirectorFixupPolicy.cpp` step 8 adds both the `UObjectRedirector` (`ObjectsToDelete.AddUnique(PackagedRedirector)`, :430) and, when the package holds no real asset, the `UPackage` itself (`ObjectsToDelete.AddUnique(RedirectorPackage)`, :442) to one array. `Result.RedirectorsDeleted = ObjectTools::DeleteObjects(ObjectsToDelete, false)` (:451) then counts objects, which comes to 2 per ordinary redirector package. The weak-pointer-measured `DeletedRedirectorPackages` (:453-461) is correct, so the response contradicts itself.

Consumers of the inflated count:
- `asset.fixup_redirectors`: `redirectorsFixed` (`Handlers/Asset/AssetWorkflowHandler.cpp:320`) and the message `"Fixed %d of %d redirectors"` (:323-325).
- `RedirectorFixupPolicy::AddReport` → `redirectorsDeleted` (:477), which reaches both `asset.fixup_redirectors` (:321) and `asset.bulk_delete` (:1174).

The test doesn't catch it because `Tests/Assets/TestRedirectorFixupPolicy.cpp:157` only asserts `RedirectorsDeleted >= 1`.

Observed: `asset.fixup_redirectors {"directoryPath":"/Game/References"}` after consolidating 8 assets returned `redirectorsFound:8, redirectorsFixed:16, redirectorsDeleted:16`, 8 entries in `redirectorsDeletedPaths`, and message "Fixed 16 of 8 redirectors".

**Workaround:** Ignore `redirectorsFixed`/`redirectorsDeleted` and use `redirectorsDeletedPaths.length` (packages actually removed, measured after the delete).

**Fix:** Stop assigning the `DeleteObjects` return value to `RedirectorsDeleted`. Derive it from the measured identity instead: count the `DeleteCandidates` whose weak pointer stopped resolving, or use `DeletedRedirectorPackages.Num()` if the unit is packages. Halving the count is wrong, because a package that also holds a real asset contributes only its redirector. Tighten the test at :157 to `TestEqual(RedirectorsDeleted, <expected>)` and to equality with `DeletedRedirectorPackages.Num()` for the single-redirector-per-package case.

## History

- `#1-double-counted-deletes` `OPEN` reporter — Verified from source: DeleteObjects counts the redirector plus its UPackage (RedirectorFixupPolicy.cpp:430/:442/:451), which inflates `redirectorsFixed`/`redirectorsDeleted` 2x in asset.fixup_redirectors and asset.bulk_delete. Seen live as "Fixed 16 of 8 redirectors" on /Game/References.
