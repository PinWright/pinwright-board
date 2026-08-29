---
id: E-pcg-graph-class-echo-not-provenance
title: "`pcg.md` says the echoed `graphClass` makes \"a default distinguishable from a requested subclass\", but omitting the parameter and passing `graphClass: \"PCGGraph\"` return byte-identical responses — the echo distinguishes classes, not provenance, and by construction cannot do otherwise"
status: OPEN
severity: Low
category: ergonomic
tags: [pcg, create_graph, docs, wiki, echo, provenance, response-shape, overclaim, doc-defect]
encounters: 2
lastSeen: 2026-08-29T16:45:00+05:00
---

# A documentation overclaim, not an implementation error

`Docs/wiki-src/pcg.md:12` (served verbatim as the `## Conventions` bullet on the generated
`Saved/PinWright/wiki/pcg.md`):

> It is resolved by reflection and gated on `IsChildOf(UPCGGraph)` plus non-abstract, so any
> subclass whose module is loaded works without PinWright linking it; **the resolved class is echoed
> back as `graphClass` so a default is distinguishable from a requested subclass.**

Measured: it is not. `pcg.create_graph` with the parameter omitted and `pcg.create_graph` with
`graphClass: "PCGGraph"` produce **byte-identical** responses — same `graphPath`, same
`graphClass: "/Script/PCG.PCGGraph"`, same everything. Nothing in the payload records that a
parameter was supplied.

## The implementation is right, and that is exactly why the sentence cannot be

This is not a bug to fix in `PCGGraphCreate.cpp`. `F-pcg-create-graph-class-parameter`'s `#2`
records the decision, verbatim:

> Response now echoes `graphClass` read off `Graph->GetClass()->GetPathName()` — **off the created
> object, not off the request.**

`PCGGraphCreate.cpp:132`:

```cpp
Result->SetStringField(TEXT("graphClass"), Graph->GetClass()->GetPathName());
```

Reading the echo off the created object is the correct choice and should stay: it is the one form
that cannot lie about what was built, it survives a resolver that accepts several spellings of one
class (`ResolveUClass` takes both `PCGGraph` and `/Script/PCG.PCGGraph`), and it is what makes the
field trustworthy for the question it *can* answer. But an echo of the created object's class
distinguishes **classes**. Provenance — did the caller ask, or did the default apply — is a property
of the request, and the two coincide only when the requested class differs from the default. The doc
sentence asks a field derived from the object to answer a question about the input, which no
implementation of that field can do.

So the fix is on the documentation side, or in an additional field. It must not be filed as
"`graphClass` is echoed wrong".

## Re-measured on a host where a real subclass exists — the ticket survives, and narrows

`#1` was written on a host with **no** concrete `UPCGGraph` subclass loaded, so the only two calls it
could compare were "omitted" and `"PCGGraph"`. The Procedural Vegetation Editor plugin is now
enabled, `/Script/ProceduralVegetation.ProceduralVegetationGraph` is live, and the third call is
finally possible. Result:

| call | echoed `graphClass` |
|---|---|
| `graphClass` omitted | `/Script/PCG.PCGGraph` |
| `graphClass: "PCGGraph"` | `/Script/PCG.PCGGraph` |
| `graphClass: "ProceduralVegetationGraph"` | `/Script/ProceduralVegetation.ProceduralVegetationGraph` |

So the field **does** separate a requested *subclass* from a default, and rows 1 and 2 remain
byte-identical. That makes `pcg.md:12` **narrowly true and false under its natural reading** — a
distinction worth stating precisely, because a fixer who tests only the subclass case will conclude
the sentence is fine and close this:

- *"a default is distinguishable from a requested **subclass**"* — literally true, and vacuously so:
  a requested subclass differs from the default *by being a different class*, which is the one
  question an object-derived field can answer.
- *"a default is distinguishable from a request"* — false, and this is how the sentence reads, since
  a caller asking "did my `graphClass` take effect?" has no reason to assume they asked for something
  other than the default. Exactly the caller who most needs the answer — someone verifying that a
  parameter they just added to a script is being honoured — is the caller the field cannot serve.

## The provenance bit is already computed, then discarded

`#1` proposed a `graphClassDefaulted` boolean as the more expensive of two options. It is not
expensive. The handler **already branches on the exact bit**, one line before the resolution it
gates, and then throws it away:

`Source/PinWrightPCG/Private/Handlers/PCG/PCGGraphCreate.cpp:47-49`:

```cpp
UClass* GraphClass = UPCGGraph::StaticClass();                                     // :47
const FString GraphClassInput = Ctx.GetString(TEXT("graphClass")).TrimStartAndEnd();  // :48
if (!GraphClassInput.IsEmpty())                                                    // :49
```

`GraphClassInput.IsEmpty()` at `:49` **is** the provenance bit — it is true exactly when the caller
supplied nothing — and the response (`:132`) publishes only `Graph->GetClass()->GetPathName()`. The
fix is one `SetBoolField`, or one `SetStringField` echoing `GraphClassInput` verbatim; no new
computation, no new lookup, no change to the correct object-derived `graphClass`.

**Two mechanism facts that make the emptiness reliable, both worth citing so nobody re-derives them:**

- `Ctx.GetString(TEXT("graphClass"))` at `:48` supplies **no second argument**, so it binds
  `FHandlerContext::GetString`'s defaulted parameter `const FString& Default = TEXT("")`
  (`Source/PinWright/Private/Handlers/HandlerContext.h:79`, definition
  `HandlerContext.cpp:17-24`) and returns `""` for an absent key.
- The `RPC_PARAM_DEF("graphClass", ..., "/Script/PCG.PCGGraph")` default at
  `PCGGraphCreate.cpp:29-34` is **documentation-only and is never applied to the payload**.
  `FParamSpec::Default` (`Source/PinWright/Private/Handlers/ParamSpec.h:20`, populated by the macro
  at `:39-40`) is consumed in exactly one place — wiki rendering,
  `Source/PinWright/Private/Catalog/MarkdownHelpers.cpp:33-35`. Nothing under `Dispatch/` injects it.
  So an omitted key really does reach `:49` as an empty string and the whole resolution block is
  skipped; the class stays the `:47` fallback. Same outcome as a real default would give, by a
  different route — which is why the bit at `:49` is trustworthy and why the schema's default string
  cannot be used to reconstruct provenance after the fact.

## What it should do

Two options, written as alternatives by `#1` and revised to "both" by `#2` — see the paragraph
below them:

1. **Reword `pcg.md:12`.** Drop the provenance clause and say what the field is: *the class the
   created asset actually is, read off the object — so a caller can confirm which subclass was
   instantiated, including when the parameter was omitted.* That is the true and useful statement,
   and it needs no code.
2. **Add a separate field** — `graphClassDefaulted` (bool) or `graphClassRequested` (the raw input
   string) — from `GraphClassInput.IsEmpty()` / `GraphClassInput` at `PCGGraphCreate.cpp:48-49`,
   where the answer is already in hand, already branched on, and free. Then the doc sentence becomes
   true of *that* field, and `graphClass` keeps its honest object-derived meaning.

Option 1 is enough for the defect as filed. **`#2` upgrades the recommendation to "do both".** The
re-measurement above shows the sentence is not simply wrong but *conditionally* right, which is the
worst shape for a doc-only fix: reword it accurately and it becomes a hedge a reader has to parse
("distinguishable when, and only when, the class you asked for is not the default"), while the field
that would make it unconditionally true costs one line at a site the handler already visits.
`graphClassRequested` is the marginally better of the two — it echoes what the caller sent, which
also disambiguates *which spelling* resolved (`ResolveUClass` accepts several), and a boolean does
not.

## Distinct from

- `F-pcg-create-graph-class-parameter` (IN-REVIEW) — the feature ticket that added `graphClass` and
  wrote the `pcg.md` bullet. Its `#2` implementation decision is correct and is not being
  questioned here; its `#3` already forward-references this file. Not merged: that ticket's subject
  is the parameter, this one's is a sentence about the response.
- `E-pcg-set-self-pruning-no-echo` (OPEN, Low) — a `pcg.*` response that echoes **nothing** it
  applied, and notes as an aside that a caller also cannot tell its values from class defaults.
  Same namespace, same response-shape family, opposite failure: there the field is missing, here it
  is present, correct, and described as answering more than it does.
- `E-pcg-add-node-echo-pin-labels` (OPEN, Low, `encounters: 6`) — the third member of that family:
  data the response could carry for free and does not. All three are about what
  `PCGGraphCreate.cpp` / `PCGGraphAuthoring.cpp` put in their `Result` objects, so a fixer opening
  either file should have all three in view; kept separate per the no-umbrella policy.

## Dedup

Board-wide search for `graphClass`, provenance and `create_graph` echo tickets: the only files
touching `pcg.create_graph`'s response are the three above, each distinguished. Nothing owns the
`pcg.md:12` sentence.

## History
- `#1-echo-is-not-provenance` `OPEN` reporter — Measured: omitting `graphClass` and passing `graphClass: "PCGGraph"` return byte-identical `pcg.create_graph` responses, so the field cannot separate a defaulted graph from a requested one. Doc claim quoted and re-derived at `Docs/wiki-src/pcg.md:12` — "the resolved class is echoed back as `graphClass` so a default is distinguishable from a requested subclass" — served verbatim on the generated `Saved/PinWright/wiki/pcg.md`. Filed explicitly as a **doc overclaim, not an implementation error**: `F-pcg-create-graph-class-parameter`'s `#2` states the echo is read "off the created object, not off the request", and the code is `Result->SetStringField(TEXT("graphClass"), Graph->GetClass()->GetPathName())` at `PCGGraphCreate.cpp:132`. That choice is correct — it cannot misreport what was built, and it is stable across the several spellings `ResolveUClass` accepts — and it makes the documented provenance claim unachievable by construction, because provenance is a property of the request and the field is derived from the object. Ask: reword `pcg.md:12` to describe what the field is (the class the asset actually is, confirmable even when the parameter was omitted), or add a separate `graphClassDefaulted` boolean from `GraphClassInput.IsEmpty()` at `PCGGraphCreate.cpp:48-49` where the answer is already free. Cross-linked `E-pcg-set-self-pruning-no-echo` and `E-pcg-add-node-echo-pin-labels` as the same two files' response-shape family. Severity Low: impact class is Low by the rubric's own wording — a docs/accuracy defect, no wrong data returned, no blocked task — and the consequence is bounded because the only case the doc gets wrong is `graphClass: "PCGGraph"` versus omission, where the created asset is identical anyway, so a caller misled by the sentence still builds the right thing. Declining a reach bump in either direction: `pcg.create_graph` is not an every-session verb, and Low is already the floor so the rare-path bump-down has nowhere to go. Not Medium: nothing is blocked and no workaround is needed — the response is correct, only its description is not.
- `#2-stands-with-a-real-subclass-and-the-fix-is-one-line` `OPEN` reporter — **Second encounter; the ticket STANDS and narrows. Status deliberately unchanged.** `#1` could only compare two calls because no concrete `UPCGGraph` subclass was loaded on that host. The Procedural Vegetation Editor plugin is now enabled and `/Script/ProceduralVegetation.ProceduralVegetationGraph` is live, so the missing third call was made: `graphClass: "ProceduralVegetationGraph"` echoes `/Script/ProceduralVegetation.ProceduralVegetationGraph`, while omitted and `"PCGGraph"` still echo `/Script/PCG.PCGGraph` byte-identically. So the field distinguishes **classes** — confirmed live, not inferred — and still never distinguishes **provenance**, exactly as `#1` argued by construction. What this adds is the precise shape of the doc defect: `pcg.md:12` is **narrowly true** (*"a default is distinguishable from a requested **subclass**"* — vacuously, since a requested subclass differs from the default by being a different class) and **false under its natural reading** (*a default is distinguishable from a request*), which is the reading a caller verifying "did my new `graphClass` line take effect?" will take, and that caller is the one the field cannot serve. Recorded because a fixer who tests only the subclass case will read the sentence as accurate and close this.

  **The one-line fix, which `#1` under-rated as the expensive option.** `PCGGraphCreate.cpp:49` already evaluates `GraphClassInput.IsEmpty()` as the branch that gates the whole resolution block, one line after reading the parameter at `:48` — that predicate **is** the provenance bit, and the response at `:132` publishes only `Graph->GetClass()->GetPathName()`. One `SetStringField(TEXT("graphClassRequested"), GraphClassInput)` costs nothing and needs no new lookup. Recommendation upgraded from "option 1 is enough" to **do both**: a doc-only reword now has to encode a conditional ("distinguishable when, and only when, you asked for something other than the default"), which is a hedge rather than a statement, and the field that removes the condition is one line. Preferring the string echo over a `graphClassDefaulted` bool because it also records *which spelling* the caller used, and `ResolveUClass` accepts several.

  **Two mechanism facts verified at HEAD and added to the body, because both are load-bearing for the fix and neither is obvious.** (a) `Ctx.GetString(TEXT("graphClass"))` at `:48` passes no second argument, so it binds the defaulted parameter `const FString& Default = TEXT("")` on `FHandlerContext::GetString` (`Source/PinWright/Private/Handlers/HandlerContext.h:79`, definition `HandlerContext.cpp:17-24`) and yields `""` for an absent key. Stated carefully because the tempting phrasing — "`GetString` takes no default" — is **false**: there is exactly one `GetString`, it has a defaulted `Default` parameter, and there are no overloads; it is the *call site* that supplies none. (b) The declared `RPC_PARAM_DEF` default `/Script/PCG.PCGGraph` (`PCGGraphCreate.cpp:29-34`, the string itself on `:34`) is **documentation-only and never applied**: `FParamSpec::Default` (`Handlers/ParamSpec.h:20`, populated by the macro at `:39-40`) is read in exactly one place, wiki rendering at `Catalog/MarkdownHelpers.cpp:33-35`, and nothing under `Dispatch/` injects it into the payload. So an omitted key genuinely arrives empty and the class stays the `:47` fallback — same outcome a real applied default would give, by a different route. That matters twice over: it is why the `:49` bit is trustworthy, and it is why provenance cannot be reconstructed after the fact from the schema string.

  **Dedup re-run at HEAD, nothing new to merge.** `E-pcg-set-self-pruning-no-echo` and `E-pcg-add-node-echo-pin-labels` remain the two response-shape siblings in these files and are re-confirmed distinct (field absent, and pin labels, respectively). `F-pcg-create-graph-class-parameter` moved to `DONE` this session on the strength of on-disk byte evidence for the subclass; its `#4` forward-references this file and explicitly leaves the doc sentence here rather than absorbing it, so the two do not overlap. No new ticket filed for the doc sentence — this one already owns it, and filing a second would put two authorities on one line of `pcg.md`. **Severity unchanged at Low, re-argued rather than carried over:** still a docs/accuracy defect with no wrong data returned and nothing blocked, and the bounded-consequence argument from `#1` survives intact — the only case the sentence gets wrong is `"PCGGraph"` versus omission, where the created asset is identical either way, so a misled caller still builds the right thing. The new subclass measurement does **not** raise it: it makes the sentence *more* often true, not less. Reach bump declined in both directions for `#1`'s reasons, and Low is the floor besides.
