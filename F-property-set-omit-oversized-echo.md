---
id: F-property-set-omit-oversized-echo
title: "property.set has no way to suppress its value echo — writing 4 mesh entries on a PCG node returned 16.5 KB and 6 returned 24.8 KB, because each FPCGSoftISMComponentDescriptor serialises its whole BodyInstance"
status: OPEN
severity: Low
category: feature
tags: [property, property-set, oversized, response-size, spill, projection, pcg, write-echo, follow-on]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The oversized-omission machinery exists, is opt-in, and the write path cannot reach it

`property.set` unconditionally re-exports the property it just wrote and puts it in the response
(`Handlers/Utility/UtilityPropertyHandler.cpp:1264-1266`):

```cpp
if (TSharedPtr<FJsonValue> CurrentValue = ExportPropertyToJsonValue(TargetContainer, Property))
{
    ResultPayload->SetField(TEXT("value"), CurrentValue);
}
```

That is the raw exporter, not the omission-aware wrapper. Its registered params are `objectPath`,
`propertyName`, `value`, `markDirty` (`:1020-1026`) — there is no `omitOversized`, and no other lever
that trims the echo.

Measured this session, writing mesh entries on a PCG node: **4 entries → 16.5 KB; 6 entries →
24.8 KB.** The payload is not the meshes. `UPCGMeshSelectorWeighted::MeshEntries` is a
`TArray<FPCGMeshSelectorWeightedEntry>` (`PCGMeshSelectorWeighted.h:100`), each entry holding a
`FPCGSoftISMComponentDescriptor Descriptor` (`:36`). That struct derives from
`FSoftISMComponentDescriptor` (`PCGISMDescriptor.h:13`), whose base carries
`FBodyInstance BodyInstance` (`Runtime/Engine/Public/ISMPartition/ISMComponentDescriptor.h:86`) —
a full physics body serialised per entry, per write. Only the 10,000-char spill guard keeps the verb
usable at all: past it the response becomes a file reference and the caller pays a `Read`.

## This is the mirror of a DONE ticket, and it must rebut that ticket's scope sentence

`F-rpc-property-omit-oversized-opt-in` (**DONE**, Low) added the opt-in `omitOversized` to
`property.get` and `property.list`. Its scope sentence, verbatim:

> `container.array.get` is intentionally excluded — it's per-element by index and cannot accidentally
> dump a mega-array. `container.array.append` / `set` / `clear` are writers and irrelevant. Scope is
> `property.get` and `property.list` only.

That sentence is right about the array writers and wrong by omission about `property.set`. It rules
the write direction out on the premise that writers do not emit payloads worth trimming — true of
`container.array.append` / `set` / `clear`, which do not echo the container. `property.set` **does**
echo, at `:1264-1266`, and `FPCGSoftISMComponentDescriptor` is the counterexample the sentence did
not anticipate: a write whose echo is bigger than most reads.

**Do not reopen the DONE ticket.** This is the follow-on it did not know it needed. Its verification
(`#3`) is sound and the shipped behaviour is correct for the two verbs it covers.

## Fix: reuse the landed machinery, do not build new machinery

The helpers the DONE ticket named still exist, and the write path already has a wrapper sitting one
screen above the site that needs it:

- `IsKnownOversizedProperty(const FProperty*)` — `Utils/PropertyExport.cpp:1234-1271`
- `BuildOmissionPlaceholder(FProperty*, const void*, const FOmissionReason&)` —
  `Utils/PropertyExport.cpp:1275`
- `ExportPropertyToJsonValueWithOversizedOmission(void*, FProperty*, bool)` —
  `Handlers/Utility/UtilityPropertyHandler.cpp:1000-1015`, the exact three-line wrapper
  `property.set` needs and does not call. `property.get` uses it at `:1733`/`:1743`;
  `property.list` at `:1923`/`:1925`.

So the code change is an `omitOversized` param on `property.set` and swapping the call at `:1264`
for the wrapper — the same opt-in default (`false`) the DONE ticket chose, for the same reason.

**A cite correction the fixer needs.** The DONE ticket places these helpers at
`PropertyUtils.cpp:1812-1884`. At this checkout's HEAD they live in `Utils/PropertyExport.cpp`
(declared in `PropertyExport.h:62`, `:67`); `PropertyUtils.cpp` no longer holds them. Callers are
`UtilityPropertyHandler.cpp:1007-1010`, `PinWright_SCSHandlers.cpp:264-267`, and
`PropertyExport.cpp:554`, `:588-592`, `:1419`, `:1466-1469`.

## Adding the entry: the existing three, and why a fourth row is not the whole answer

The current allow-list, verbatim from `PropertyExport.cpp:1244-1261`, is three entries built once
into a `TMap<UClass*, TMap<FName, FOmissionReason>>`:

| owner class | property | reason string |
|---|---|---|
| `/Script/Synthesis.AudioImpulseResponse` | `ImpulseResponse` | `TArray<float>` — "audio samples, %d elements" |
| `/Script/Engine.BodySetup` | `AggGeom` | `FKAggregateGeom` — "physics geometry (convex/tri-mesh sub-arrays)" |
| `/Script/Engine.InstancedStaticMeshComponent` | `PerInstanceSMData` | `TArray<FInstancedStaticMeshInstanceData>` — "per-instance transforms, %d elements" |

The table keys on `Property->GetOwnerClass()` + `GetFName()` (`:1234-1240`, `:1264-1271`). That has a
consequence worth stating plainly, because the obvious phrasing of this request does not fit it:
**`FPCGSoftISMComponentDescriptor` cannot be "added to the allow-list" as a type.** The table
recognises `(class, property name)` pairs, not struct types wherever they appear.

Two things follow, and a fix should do both:

1. **The row is expressible and should be added**, because the property that blows up is a real
   UCLASS-owned UPROPERTY: `/Script/PCG.PCGMeshSelectorWeighted` + `MeshEntries`. That closes the
   measured case.
2. **The row does not generalise, and this shape will recur.** Any struct embedding an
   `FBodyInstance` — every `FISMComponentDescriptor` variant, and PCG has several — reproduces it
   under a different owner and a different property name, needing a new hand-written row each time.
   The durable version is a type-aware or size-measured rule: recognise `FBodyInstance` (and
   descriptor structs) by struct type wherever they are reached, or gate on measured serialised size
   rather than on a name table. That is a bigger change than this ticket asks for and is offered as
   the direction, not the requirement.

## The effective budget is smaller than 10,000

`E-spill-threshold-measured-post-wrap` (OPEN, Medium) is why the numbers above spill so readily. The
`10000` threshold is measured on the **wrapped** MCP ToolResult, which carries the payload twice —
JSON-escaped inside `content[0].text` and verbatim in `structuredContent` — so a bare handler result
over roughly **4,250 characters** spills, an amplification measured at ~2.35x. Against that real
ceiling, a 4-entry write at 16.5 KB is not marginally over: it is ~4x the effective budget, and a
6-entry write ~6x. Every such write spills, every time.

## Remedy family

`E-asset-list-no-projection-spills` (IN-REVIEW, Low) is the landed template for this shape — the
`namesOnly` / `fields` per-row projection, mirroring `actor.list`'s already-shipped lever
(`E-actor-list-no-limit-spills`). Siblings still open: `E-blueprint-list-no-projection-spills`
(OPEN, Low), `E-get-node-details-batch-no-projection-spills` (IN-REVIEW, Low),
`E-inspect-object-no-projection-spills` (WONTFIX, Low). The whole family is "the reader has no lever
to drop its heavy field". This ticket is the same family on the **write** side, where the heavy field
is the echo of what was just written and the lever already exists two verbs over.

## Severity

**Impact = Low, by the rubric's own words.** Low covers *"a response spill that only forces a
`Read`"*, which is precisely this: the write lands, `applied` is truthful, the value is fully
recoverable from the spill file, and nothing downstream is wrong. No band above Low is reachable —
nothing is corrupted (not Critical), nothing is false (not High), and no task is blocked or forced
through a workaround (not Medium; the spill *is* the mechanism working).

**Does "write echo" rather than "read echo" change the band? Argued, and the answer is no.** The
honest case for a bump is that a read spill is paid when you ask for the data, while a write spill is
paid on **every mutation in an authoring loop** — nobody reads `MeshEntries` 110 times, but a session
that writes it repeatedly pays the spill on each write, and the caller never wanted the payload at
all. That is a real asymmetry and it is why this is filed rather than shrugged off. It still does not
clear Medium: the rubric's Medium needs a soft *blocker* — a documented workaround, a source dive, or
many extra calls — and this costs neither extra calls nor a workaround, only a larger response per
call already being made.

**Reach modifier: bump-up declined, and named.** `property.set` runs in almost every session, which
by the letter of the modifier would carry Low → Medium. Declined because the *oversized* case does
not: it needs a property in the small set of struct-array monsters (PCG descriptors, ISM per-instance
data, impulse responses), so the affected path is narrow even though its host verb is universal.
Bumping on the verb's frequency rather than the gap's would put this ahead of
`E-spill-threshold-measured-post-wrap` (Medium), which is the ticket that actually makes every
structured response worse.

**Net: Low**, matching every sibling in the projection-spill family and matching the DONE ticket it
follows on from.

## Same shape as

- `F-rpc-property-omit-oversized-opt-in` (DONE, Low) — the read-side half; this is the write-side
  follow-on, and the rebuttal of its scope sentence is in the body above.
- `E-asset-list-no-projection-spills` (IN-REVIEW, Low) — the landed remedy template.
- `E-blueprint-list-no-projection-spills` (OPEN, Low) / `E-get-node-details-batch-no-projection-spills`
  (IN-REVIEW, Low) — open siblings, same lever, different verbs.

## Related

- `E-spill-threshold-measured-post-wrap` (OPEN, Medium) — why the effective inline budget is ~4,250
  chars rather than 10,000, which is what turns a 16.5 KB echo into a guaranteed spill.
- `F-property-set-batch` (filed this session, Medium) — the other half of the same authoring loop.
  The two compound: a batch write without an echo lever would return one response carrying every
  oversized echo in the bag.
- `F-pcg-set-node-property` (OPEN, High) — the PCG authoring path these writes were made on.

## History
- `#1-write-echo-blows-up-on-pcg-descriptors` `OPEN` reporter — Measured on host project EAContentExamples58 while authoring a PCG graph: setting 4 mesh entries on a PCG node through `property.set` returned 16.5 KB, 6 entries returned 24.8 KB. Cause traced to source, not guessed: `UPCGMeshSelectorWeighted::MeshEntries` is `TArray<FPCGMeshSelectorWeightedEntry>` (`PCGMeshSelectorWeighted.h:100`), each entry carrying `FPCGSoftISMComponentDescriptor Descriptor` (`:36`), which derives from `FSoftISMComponentDescriptor` (`PCGISMDescriptor.h:13`) whose base holds `FBodyInstance BodyInstance` (`ISMComponentDescriptor.h:86`) — a whole physics body serialised per entry, per write. `property.set` echoes it unconditionally through the RAW exporter at `UtilityPropertyHandler.cpp:1264-1266` and declares no `omitOversized` param (`:1020-1026`); only the 10,000-char spill guard keeps the verb usable. This is the mirror of `F-rpc-property-omit-oversized-opt-in` (DONE, Low), whose scope sentence — "`container.array.append` / `set` / `clear` are writers and irrelevant. Scope is `property.get` and `property.list` only." — is verified verbatim and is right about the array writers but ruled out the write direction on a premise `property.set` breaks: those writers do not echo, and `property.set` does. Filed as a FOLLOW-ON, not a reopen. Fix is to reuse the landed machinery: `IsKnownOversizedProperty` / `BuildOmissionPlaceholder` (`Utils/PropertyExport.cpp:1234-1271`, `:1275`) via the existing wrapper `ExportPropertyToJsonValueWithOversizedOmission` (`UtilityPropertyHandler.cpp:1000-1015`) that `property.get` (`:1733`, `:1743`) and `property.list` (`:1923`, `:1925`) already call and `property.set` does not. **Cite correction for the fixer**: the DONE ticket places those helpers at `PropertyUtils.cpp:1812-1884`; at HEAD they are in `Utils/PropertyExport.cpp` (declared `PropertyExport.h:62`, `:67`). **Allow-list note**: the existing three entries are `/Script/Synthesis.AudioImpulseResponse::ImpulseResponse`, `/Script/Engine.BodySetup::AggGeom` and `/Script/Engine.InstancedStaticMeshComponent::PerInstanceSMData` (`PropertyExport.cpp:1244-1261`), and the table keys on `(GetOwnerClass, GetFName)` — so `FPCGSoftISMComponentDescriptor` cannot be added as a TYPE; the expressible row is `/Script/PCG.PCGMeshSelectorWeighted` + `MeshEntries`, which closes the measured case but does not generalise to the other `FBodyInstance`-carrying descriptor structs, for which a type-aware or size-measured rule is the durable answer. Effective budget context from `E-spill-threshold-measured-post-wrap` (OPEN, Medium): the 10,000 gate is measured post-wrap at ~2.35x, so the real bare-result ceiling is ~4,250 chars and a 4-entry write is ~4x over it — every such write spills. Severity Low: the rubric's "a response spill that only forces a `Read`" fits exactly; the write-vs-read asymmetry (paid on every mutation in an authoring loop, and the caller never wanted the payload) was argued and does not reach Medium, which needs a soft blocker; the every-session reach bump-up on `property.set` is declined because the oversized case is confined to a narrow set of struct-array properties even though its host verb is universal.
