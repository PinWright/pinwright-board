---
id: E-montage-section-autochain-undocumented
title: "animation.authoring montage docs never state that add_montage_section auto-chains sections (sets prev.NextSectionName) — callers do a defensive get_animation_info + redundant link_sections to discover it"
status: OPEN
severity: Low
category: ergonomic
tags: [animation, anim-montage, authoring, add-montage-section, link-sections, next-section, docs, wiki]
encounters: 1
lastSeen: 2026-06-24T06:10:13Z
---

# Montage section auto-chaining is undocumented, so callers over-step to verify it

`animation.authoring.add_montage_section` **already chains sections sequentially**:
adding a section auto-links the previous section's `NextSectionName` to the new
one (when the previous section had no link yet). Neither the handler description
nor the wiki overlay says so, and the wiki Notes instead present
`link_sections` as the way to "chain sections" — so a caller building a normal
linear section flow (`A -> B -> C`) cannot tell whether the sequential
`add_montage_section` calls already linked the chain or whether explicit
`link_sections` is required. The safe response is to over-step: inspect with
`get_animation_info` and then issue `link_sections` calls that are in fact
redundant.

## Source-confirmed: add_montage_section auto-chains

`AnimationAuthoringHandler_Sequence.cpp` `add_montage_section` (:1189) is a thin
wrapper over the engine call:

```cpp
int32 SectionIndex = Montage->AddAnimCompositeSection(FName(*SectionName), StartTime);
```

`UAnimMontage::AddAnimCompositeSection` (UE 5.7
`Engine/Source/Runtime/Engine/Private/Animation/AnimMontage.cpp:348-362`) links
the previous section to the new one automatically:

```cpp
NewSection.Link(this, StartTime);
int32 NewSectionIndex = CompositeSections.Add(NewSection);

// when first added, just make sure to link previous one to add me as next if previous one doesn't have any link
// it's confusing first time when you add this data
int32 PrevSectionIndex = NewSectionIndex-1;
if ( CompositeSections.IsValidIndex(PrevSectionIndex) )
{
    if (CompositeSections[PrevSectionIndex].NextSectionName == NAME_None)
    {
        CompositeSections[PrevSectionIndex].NextSectionName = InSectionName;
    }
}
```

The engine's own comment ("it's confusing first time when you add this data")
underscores that this behavior is non-obvious — exactly the discoverability gap
this ticket is about. So after `add_montage_section Windup`, `... Strike`,
`... Recovery` the chain is *already* `Default -> Windup -> Strike -> Recovery`
(Recovery left as the natural end) without any `link_sections` call. Explicit
`link_sections Windup->Strike` / `Strike->Recovery` only re-assert links that
already exist.

## Process friction (the PROCESS angle — distinct from the snake_case ticket)

Struggle audit of the Echo sword-combo montage task (namespace
animation.authoring, realism mode / no seed, outcome `ergo`; the wiki snake_case
half is the judge-filed `E-montage-wiki-example-snake-case-params`). This is the
separate over-stepping angle in the same workflow: the call log shows the agent
did **one extra inspect + two redundant links** specifically to resolve the
chaining uncertainty —

1. `get_animation_info` — args summary *"inspect section state pre-link"* (a
   defensive read inserted between the section adds and the links, solely to see
   the chain state).
2. `link_sections` — `Windup->Strike`.
3. `link_sections` — `Strike->Recovery`.

Agent friction note (verbatim): *"references add_montage_slot/set_section_timing
steps that turned out unnecessary — ... add_montage_section auto-chains sections
sequentially, so the explicit link_sections calls were belt-and-suspenders
confirmation rather than strictly required."* The agent reached the correct
mental model (auto-chaining) only by inspecting the asset; the docs neither
confirm it up front nor scope when `link_sections` is actually needed (only for
*non-linear* / branching flows, or to override the auto-set link). Net: 3 calls
on this task that a single documented sentence would have removed.

This is distinct from the existing montage tickets:
- `E-montage-wiki-example-snake-case-params` — wrong *param spellings*
  (snake_case / phantom `end_time`) in the same Notes block; this ticket is the
  missing *auto-chain contract*, a different sentence to add.
- `F-montage-section-remove-rename` / `B-create-montage-duplicate-default-slot`
  — missing lifecycle verbs / duplicate seeded slot; this is purely a docs gap
  on linking behavior, no code change to the verbs themselves.

## What it should do

Downstream wiki edit (not this audit's job) on the overlay page
`docs/wiki-src/animation.authoring.md`, `create_montage` section (and ideally the
`add_montage_section` handler description), add an explicit auto-chain note:

- State that `add_montage_section` adds sections in time order and **auto-links
  the previous section's `NextSectionName` to the newly-added one when the
  previous section has no next link yet** — so a linear `A -> B -> C` flow needs
  no `link_sections` calls; the sequential adds already chain it.
- Re-scope `link_sections` to its real job: setting a section's `NextSectionName`
  to a **non-sequential** target (branching/looping flows) or **overriding** the
  auto-set link — not as a required step for ordinary linear chaining.
- Optional: note that the auto-link only fills an *empty* `NextSectionName`, so
  re-ordering / re-pointing an existing link still needs `link_sections`.

Once written, the documented build sequence stops teaching callers to issue (and
verify) redundant `link_sections` calls for the common linear case.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle (process) audit of the Echo
  sword-combo montage task (`M_Echo_SwordCombo`, namespace animation.authoring,
  realism mode / no seed, outcome `ergo`; snake_case half judge-filed as
  `E-montage-wiki-example-snake-case-params`). Distinct over-stepping angle:
  `add_montage_section` auto-chains (source-confirmed
  `AnimMontage.cpp:348-362` sets `prev.NextSectionName` to the new section when
  empty; handler `AnimationAuthoringHandler_Sequence.cpp:1189` wraps it
  unchanged), but neither the handler description nor the wiki overlay says so,
  and the wiki presents `link_sections` as the chaining step. Call log shows the
  agent inserted a defensive `get_animation_info` ("inspect section state
  pre-link") plus two `link_sections` calls (`Windup->Strike`, `Strike->Recovery`)
  that the friction note calls *"belt-and-suspenders confirmation rather than
  strictly required"* — 3 extra calls a one-sentence doc note would remove.
  Tagged docs; downstream wiki overlay edit names the page
  `docs/wiki-src/animation.authoring.md` (create_montage section). Distinct from
  the snake_case param-spelling ticket and from the
  remove/duplicate-slot lifecycle tickets.
