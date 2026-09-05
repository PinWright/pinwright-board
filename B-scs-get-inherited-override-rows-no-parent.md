---
id: B-scs-get-inherited-override-rows-no-parent
title: "blueprint.scs.get emits inherited-override rows with no parent field at all, and the root's child_count undercounts them — the hierarchy is still not reconstructable on a BP that overrides inherited components"
status: OPEN
severity: Medium
category: bug
tags: [blueprint, scs, blueprint-scs-get, hierarchy, child_count, parent, inherited-override, ich, readback, weapons]
encounters: 1
lastSeen: 2026-09-05T00:00:00Z
---

# The tree is emitted as rows that cannot be reassembled into a tree

`blueprint.scs.get` returns a flat row per component and expects the caller to rebuild the hierarchy
from `parent` links, cross-checked against each row's `child_count`. On a Blueprint whose components
come from the inheritable-component handler, those two signals disagree and one of them is absent.

## What was called

```
blueprint.scs.get {blueprintPath: "/Game/FPS/Weapons/BP/BP_Weapon_AR"}
```

## What came back — measured

- `WeaponRoot` reports **`child_count: 1`**.
- `MagazineMesh` carries an explicit **`parent: "WeaponRoot"`**.
- `WeaponMesh` is also under `WeaponRoot`.

So two components sit under a root that says it has one child. And separately:

- every row with **`source: "inherited-override"`** carries **no `parent` field at all** — not
  `parent: null`, not `parent: ""`, the key is absent.

A caller cannot answer "what is `WeaponMesh` attached to?" from this response, and cannot trust
`child_count` to tell it when it has found all of a node's children.

## Why this is not the local-SCS ticket

`B-scs-get-local-child-parent-link-missing` (IN-REVIEW, High) covers the case where **locally
authored** children carry no `parent` because UE leaves `ParentComponentOrVariableName` as `None`
for them; its fix derives the link by walking each node's `GetChildNodes()`. That derivation runs
over the SCS node array. The rows here come from a **different producer**: the ICH iteration added by
`F-dump-ich-overrides` (DONE), which walks `Blueprint->GetInheritableComponentHandler()->Records`
and emits `source: "inherited-override"` entries built from `Record.ComponentKey.GetSCSVariableName()`.
Those records are not SCS nodes, so a `GetChildNodes()`-based derivation does not reach them, and a
`child_count` computed from the local SCS array does not count them.

That also explains the `1`-vs-2 arithmetic without needing a source read: the root counts the local
child it owns and not the inherited-override sibling attached to it. **Stated as the reading that
fits the numbers, not as a source claim** — no plugin source was opened for this ticket and no
`file:line` is claimed.

## What is asked for

1. **Emit `parent` on `inherited-override` rows**, resolved the same way the local rows resolve it —
   from the ICH record's component key against the parent Blueprint's SCS. A row that says which
   component it overrides but not where it hangs is half a row.
2. **Make `child_count` count every child the response emits**, from whichever producer, or drop the
   field. A count that is authoritative for one source and silently partial for another is worse than
   no count, because it reads as a completeness check.
3. **Or, decisively: emit `children: [names…]`** on each row. `B-scs-get-local-child-parent-link-missing`
   already floats this as its alternative; it is strictly better here, because it makes the tree
   expressible top-down regardless of which producer contributed a row, and it makes `child_count`
   redundant rather than contradictory.
4. Whatever lands must reach `scs.txt` too — that emitter rebuilds its nested `children {}` blocks
   purely from `parent`, so inherited-override components render flat today.

## Severity

**Medium**, soft blocker. The information is recoverable — a caller can dump the parent Blueprint and
diff, or read the component transforms — but only through extra calls and a source dive, and the
`child_count` disagreement is the kind of thing a caller checks *in order to* trust the readback. Not
High because the rows themselves are correct and the contradiction is visible in the same response,
so it misleads rather than lies silently.

## Related

- `B-scs-get-local-child-parent-link-missing` (IN-REVIEW, High) — the same defect on the local-SCS
  producer; its fix is the pattern to follow and does not cover these rows. **A fixer should do both
  at once**, and the `children: []` option satisfies both tickets in one edit.
- `F-dump-ich-overrides` (DONE) — added the `source: "inherited-override"` rows this ticket says are
  incomplete. The rows landed with properties but without a hierarchy link.
- `B-asset-dump-scs-omits-inherited-parent` (DONE) — the earlier inherited-node case, where children
  referenced a parent absent from the file; this is the mirror image, where the parent link itself is
  absent from the row.
- `E-scs-blueprintpath-no-path-alias` (OPEN, Low) — **the parameter half of the same session's
  friction on this verb**: `blueprint.scs.get` rejects `assetPath`/`path` with
  `MISSING_REQUIRED_PARAM: blueprintPath` while `blueprint.decompile`,
  `blueprint.graph.find_orphaned_nodes` and `asset.dump` all accepted `assetPath` in the same
  session. Evidence appended there rather than restated here.
- `E-scs-get-diff-only-undocumented`, `E-actor-get-components-bp-cdo-omits-scs` — neighbouring gaps in
  what this verb reports.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 3. `blueprint.scs.get {blueprintPath:"/Game/FPS/Weapons/BP/BP_Weapon_AR"}` returns a hierarchy that is internally inconsistent in two ways: `WeaponRoot` reports **`child_count: 1`** while both `MagazineMesh` (carrying an explicit `parent:"WeaponRoot"`) and `WeaponMesh` sit under it; and every row with **`source:"inherited-override"` carries no `parent` field at all** — the key is absent, not null. The tree is therefore not reconstructable from the response, and `child_count` cannot be used as the completeness check a caller reaches for it as. **Distinct from `B-scs-get-local-child-parent-link-missing` (IN-REVIEW):** that ticket's `GetChildNodes()`-based derivation runs over the SCS node array, while these rows come from the ICH iteration added by `F-dump-ich-overrides` (DONE) over `GetInheritableComponentHandler()->Records`, which are not SCS nodes — so the shipped derivation does not reach them, and a `child_count` computed from the local array does not count them. That reading also accounts for the 1-vs-2 arithmetic (the root counts the local child, not the inherited-override sibling) — **stated as the reading that fits the numbers, not a source claim; no plugin source was opened and no `file:line` is claimed.** Ask, in order: emit `parent` on inherited-override rows resolved against the parent BP's SCS; make `child_count` count every child the response emits or drop it; better, emit `children: [names…]` so the tree is expressible top-down regardless of producer — which also satisfies the sibling ticket's own stated alternative in one edit; and carry whatever lands into `scs.txt`, whose nested `children {}` blocks are rebuilt purely from `parent` and so render inherited-override components flat today. Severity **Medium** on the soft-blocker band: recoverable via extra calls and a parent-BP diff, and the contradiction is visible inside the same response, so it misleads rather than lies silently. The parameter-alias half of the same session's friction on this verb (`assetPath`/`path` rejected with `MISSING_REQUIRED_PARAM: blueprintPath` while three sibling verbs accepted `assetPath` in the same session) is appended to `E-scs-blueprintpath-no-path-alias` rather than restated here.
