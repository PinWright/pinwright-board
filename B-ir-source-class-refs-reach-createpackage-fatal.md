---
id: B-ir-source-class-refs-reach-createpackage-fatal
title: "Class refs inside AGIR/CRIR/BPIR source text reach `CreatePackage`'s Fatal — the dispatch gate can never see them"
status: OPEN
severity: High
category: bug
tags: [agir, crir, bpir, compiler, createpackage, editor-crash, path-safety, tokenizer]
blockedBy: [B-createpackage-unvalidated-paths-plugin-wide]
---

# IR source text carries class references past every boundary guard

`B-createpackage-unvalidated-paths-plugin-wide` closes the `CreatePackage` Fatal
by typing path-shaped params `path` / `classref` and refusing `//` at the
dispatch boundary. The IR compilers defeat that by construction: the class
reference is not a parameter, it is **a substring of the `text` parameter**, and
`text` is IR source that must stay typed `string`. The boundary can never see it.

`CreatePackage` logs at **Fatal** on a package name containing `//`
(`UObjectGlobals.cpp:1094-1096`) — not compiled out in any configuration; it ends
the process and every unsaved package in it. Door:
`StaticLoadObjectInternal` → `ResolveName2(..., Create=true)` (`:1427`) →
`CreatePackage` (`:1310`).

## The tokenizers do not filter `/`

Verified by reading each, rather than assumed:

- The IR comment character is **`#`, not `//`**. `FIrTextUtils::StripTrailingComment`
  (`IrCore/IrTextUtils.cpp:154-178`), `FIrTokenizer` (`IrCore/IrTokenizer.cpp:145`)
  and BPIR's own copy (`Compiler/BpirParser.cpp:92-115`) all cut only at an
  unquoted `#`. `/` is special nowhere in the lexer.
- `Unquote` (`AGIR/AGIRCompiler_BlendSpace.cpp:42-49`) strips one surrounding pair
  of `"` and returns the rest **verbatim**. It filters nothing.
- AGIR arg values are pure substring reads (`AGIRParser.cpp:534`, `:594`, `:637`)
  with no character-class check.

So `//` survives the tokenizer intact and arrives at the load.

## Repro shape

```
blend_space `BS` class=/Game//X.X_C {
```

reaches `LoadObject<UClass>` at `AGIRCompiler_BlendSpace.cpp:283` with `//`
intact → `ResolveName2(Create=true)` → `CreatePackage` → **Fatal**.

Quoting is optional. `class=`, `asset=` and `control_enum=` need no quoting at
all; the `call` / `implements` opcodes need the backtick form, because
`DecodeDelimitedToken` accepts every character while bare tokens are held to
`[A-Za-z_][A-Za-z0-9_]*`.

## Sites — seven raw loads

These are raw `LoadObject`, **not** `ResolveUClass`, so the shared-resolver
guards added by the parent wave do not cover them:

| File | Line |
|---|---|
| `AGIRCompiler.cpp` | `:1040`, `:1615` |
| `AGIRCompiler_BlendSpace.cpp` | `:124`, `:283`, `:360` |
| `CRIRCompiler.cpp` | `:1059` |
| `CodeNodeEmitter.cpp` | `:1502` (via `BpirCompiler.cpp:5741`) |

`CodeNodeEmitter.cpp:1502` is the one BPIR site not routing through a guarded
resolver, against 25 that do — so BPIR is nearly clean and this is the outlier.

## Why the parent wave's mechanism does not apply

The parent ticket's whole argument is that guarding N call sites is wrong because
it does not prevent the N+1th, and that declaring the input as a path and
validating once at the boundary is the maintainable answer. Neither half is
available here:

- The boundary sees `text`, an opaque IR document. Typing it `path` is nonsense
  and typing it `classref` would refuse every valid program.
- The class ref is produced by the parser, mid-compile, from caller text.

The validation therefore has to live where the compiler resolves a reference —
which is the one place that knows a token is a class ref. The candidate fix is a
single guarded resolve helper inside the IR compilers, used by all seven sites,
refusing `//` with the compiler's normal diagnostic machinery (line/column) rather
than a Fatal. That keeps it to one chokepoint per compiler rather than seven
scattered guards.

## Related, same wave, different cause

`data_table.set_row_struct` and `data_table.create` resolve `structPath` through
`ResolveRowStruct` (`DataTableAuthoringHandler.cpp:56-59`), which calls
`LoadObject<UScriptStruct>` directly rather than the guarded
`ResolveUScriptStruct`. That one **is** closed by the parent wave, because
`structPath` is a real top-level param and was retyped `path` — but the resolver
asymmetry is worth fixing so the defence is not single-layered.

## History

- `#1-filed-from-plugin-wide-wave` **OPEN** (Reporter) — Found during the
  `B-createpackage-unvalidated-paths-plugin-wide` implementation wave by the
  agent retyping the blueprint/IR cluster, which read the tokenizers to check
  whether `//` survives them. It does. Filed separately because the parent
  wave's dispatch-boundary mechanism structurally cannot reach it. Blocked on the
  parent landing, to avoid conflicting edits in the same compilers.
