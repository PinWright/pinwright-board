---
id: B-set-default-softclass-missing-c-suffix
title: "blueprint.set_default auto-appends _C to TSoftClassPtr paths only under /Game/, so a plugin-mount (/App) Blueprint path is stored bare and unresolvable and reported as success"
status: OPEN
severity: High
category: bug
tags: [blueprint, set-default, tsoftclassptr, soft-class, silent-false-success, docs-mismatch]
encounters: 1
lastSeen: 2026-09-23T18:30:00Z
---

# set_default leaves an unresolvable TSoftClassPtr and reports success

The `blueprint.set_default` wiki ("TSoftClassPtr path auto-resolution", `Saved/PinWright/wiki/blueprint.set_default.md:22-32`) says `/Game/UI/W_Foo` becomes `/Game/UI/W_Foo.W_Foo_C`, and scopes the rule to `/Game/` paths. Observed on a C++ `TSoftClassPtr<UCommonActivatableWidget>` of a widget Blueprint (`W_AppUserPanel.LoginOverlayClass`) with value `/App/App/UI/LobbyAndMenu/Popups/W_LoginOverlay`: the response echoes `value: "/App/App/UI/LobbyAndMenu/Popups/W_LoginOverlay"` with no `warning`, `property.get` shows the same bare path, and `unreal.get_default_object(cls).get_editor_property("LoginOverlayClass")` returns `None` although `W_LoginOverlay_C` is loaded. Six properties were silently set to unresolvable paths. Passing `/App/.../W_LoginOverlay.W_LoginOverlay_C` works. The root is `/App` (a plugin mount), not `/Game`. Source-confirmed at `8748c637`: `Utils/PropertyImport.cpp:996` gates the `_C` append on `StartsWith(TEXT("/Game/"))`, and every other root falls through to `*SoftClassPtr = FSoftObjectPath(ResolvedPath)` with the path unchanged. Not a caller misuse: the `/Game/` scope is an arbitrary restriction (plugin content roots are ordinary Blueprint roots), and the handler neither resolves nor warns when the stored soft path names no UClass.

**Fix:** apply the `_C` resolution for any mounted root (or resolve to the Blueprint's GeneratedClass through the asset registry), and fail when the post-compile soft path does not resolve to a UClass.

**Related:** `B-softclassptr-silent-fail` (DONE) `#2` introduced the `/Game/`-only rule; its later `FClassProperty` handler (for `TSubclassOf`, `PropertyImport.cpp:828-860`) appends `_C` for any non-`/Script/` root, so the fix is to give the soft-class branch the same root-agnostic rule.

## History
- `#1-bare-path-stored` `OPEN` reporter — UE 5.8, host `X:\src\unreal\unreal-fpv-new`, plugin `8748c637`. Caught only by a Python readback; the response looked successful.
