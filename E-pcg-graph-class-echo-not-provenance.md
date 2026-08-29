---
id: E-pcg-graph-class-echo-not-provenance
title: "`pcg.md` says the echoed `graphClass` makes \"a default distinguishable from a requested subclass\", but omitting the parameter and passing `graphClass: \"PCGGraph\"` return byte-identical responses — the echo distinguishes classes, not provenance, and by construction cannot do otherwise"
status: OPEN
severity: Low
category: ergonomic
tags: [pcg, create_graph, docs, wiki, echo, provenance, response-shape, overclaim, doc-defect]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
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

## What it should do

Either, not both:

1. **Reword `pcg.md:12`.** Drop the provenance clause and say what the field is: *the class the
   created asset actually is, read off the object — so a caller can confirm which subclass was
   instantiated, including when the parameter was omitted.* That is the true and useful statement,
   and it needs no code.
2. **Add a separate boolean** — `graphClassDefaulted` (or `graphClassRequested`) — set from
   `GraphClassInput.IsEmpty()` at `PCGGraphCreate.cpp:48-49`, where the answer is already in hand
   and free. Then the doc sentence becomes true of *that* field, and `graphClass` keeps its honest
   object-derived meaning.

Option 1 is enough for the defect as filed. Option 2 is worth it only if some caller genuinely needs
provenance; nothing on the board asks for it today, so this ticket does not argue for it.

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
