---
id: E-bpir-select-literal-index-wildcard
title: "A select driven by a literal index leaves the Index pin PC_Wildcard, because a literal creates no connection and nothing triggers the pin-type notification"
status: WONTFIX
severity: Medium
category: ergonomic
tags: [bpir, select, index-pin, wildcard, gap-23, bpir-test-matrix]
encounters: 1
lastSeen: 2026-08-27
---

# A literal index never types the index pin

`select(Index: true, ...)` leaves the Index pin at `PC_Wildcard`. A literal creates no connection, and
PinWright's wiring never triggers `NotifyPinConnectionListChanged` -- `UEdGraphSchema::TryCreateConnection`
only calls `MakeLinkTo` -- so the node never learns the index's type.

Fixing it means changing `IndexPinType`, which is private and reachable only through
`ChangePinType` / `PinTypeChanged`, both of which force a node reconstruct mid-wiring and invalidate
the pin pointers the emit pass has cached.

**This fails loudly, not silently**, which is why it is filed as ergonomic rather than a bug: the
compile reports an error rather than producing a wrong graph. It is logged as gap 23 in
`docs/bpir-test-matrix.md`.

**Workaround:** drive the index from a variable or an expression rather than a literal.

## History
- `#1-logged-as-gap-23` `OPEN` reporter -- Named by the agent fixing `B-bpir-select-literal-text-lost`
  while establishing why `GetOptionPins()` returns nothing on a freshly emitted enum-backed select
  (it keys off `IndexPinType`, which `SetEnum` leaves at its wildcard default). Source-level claim.
- `#2-ungated-park-closed-per-readme` `WONTFIX` developer — "Closed per this board's own rule: a park that can name neither a blocking ticket nor a justifiable deferUntil date is not a defer, it is a WONTFIX. This ticket has neither. It proposes no fix and instead documents why the obvious one is unsafe: IndexPinType is private and reachable only through ChangePinType / PinTypeChanged, both of which force a node reconstruct mid-wiring and invalidate the pin pointers the emit pass has cached. The defect also fails LOUDLY at BPIR compile rather than shipping wrong data, so the status quo is tolerable. A real fix needs a deferred type-fixup pass that re-resolves pins by name after reconstruct; reopen when someone commits to that design."
