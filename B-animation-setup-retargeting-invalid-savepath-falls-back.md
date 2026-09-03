---
id: B-animation-setup-retargeting-invalid-savepath-falls-back
title: "animation.setup_retargeting silently creates output beside each source when the supplied savePath is invalid"
status: OPEN
severity: High
category: bug
tags: [animation, retargeting, path, fallback, wrong-target, persistence]
---

# An invalid requested output directory becomes the source directory

`savePath` is an accepted optional path (`Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/AnimationHandler.cpp:974-983`). If it is neither a valid long package name nor convertible from a filename, the handler silently calls `SavePath.Reset()` (`:1009-1020`). The per-asset loop interprets an empty path as “use the source package directory” (`:1049-1055`), constructs the destination there, duplicates the asset, and marks it dirty (`:1062-1097`). The request still ends in `SendSuccess` (`:1107-1137`).

The dispatcher path type rejects only dangerous `//`, not all invalid package strings (`Plugins/PinWright/Source/PinWright/Private/Handlers/ParamTypeCheck.h:96-109`), so a value that fails both package conversion checks reaches this branch. The mutation lands in a folder the caller did not select; only inspecting returned asset paths after creation reveals the fallback, and a later save can persist it there.

Fail closed with `INVALID_PATH` when a present `savePath` cannot be normalized and validated. Use the existing package-path validation/composition shape before creating any directory or asset; fallback to source folders is valid only when the parameter was omitted. Echo the normalized destination before/with results.

**Workaround:** pass an existing valid long package directory such as `/Game/Animations/Retargeted` and verify every returned path prefix.

## Related

- Catalog: `wrong-target-identity-or-fallback`, `wrong-target-scope-or-identity`
- `B-animation-setup-retargeting-does-not-retarget`

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed invalid-present path reset, source-directory duplicate/mark-dirty fallback, and success response; no editor, build, or test was run.
