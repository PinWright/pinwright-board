---
id: E-ensure-exists-create-branch-omits-created-flag
title: "blueprint.ensure_exists returns divergent shapes across its two branches — the create branch omits the created/changed flag the already-exists branch carries"
status: OPEN
severity: Low
category: ergonomic
tags: [create-vs-exists-shape, blueprint, ensure_exists, idempotency, result-misreport, docs]
encounters: 1
lastSeen: 2026-07-02T13:53:36.1750763+03:00
---

# `blueprint.ensure_exists` reports "did I just create this?" inconsistently across its two branches

`blueprint.ensure_exists` is documented as the idempotent existence guard you
put "at the top of provisioning scripts". Its whole job is to tell a caller
whether the asset was freshly created or already existed — the exact fact an
idempotent script branches on. But the two branches return **different payload
shapes**, and the create branch is the one missing the signal:

- **CREATE branch** (asset did not exist, freshly made) →
  `{"path":..., "assetPath":..., "saved":true, "assetName":"BP_HealthPotion",
  "existsAfter":true, "assetClass":"Blueprint"}` — **no `created` and no
  `changed` field.**
- **ALREADY-EXISTS branch** (idempotent re-invoke) →
  `{"exists":true, "created":false, "blueprintPath":...}` — carries `created`
  (and uses `blueprintPath`, not `path`, as the key).

So a caller writing the natural idempotent guard `if (resp.created) {...}`
cannot branch uniformly: on the already-exists path `created:false` is present,
but on the create path there is no `created:true` to read — the caller has to
*infer* creation from `existsAfter:true`+`saved:true`. The path key itself also
differs (`path` on create vs `blueprintPath` on already-exists), so even a
simple `resp.path` readback is branch-dependent. This is a pure ergonomic /
discoverability defect: the operation succeeds and nothing is corrupted, but the
method's own documented purpose (report create-vs-reuse) is only machine-readable
on one of its two branches.

**Severity rationale: impact=ergonomic-consistency/discoverability (correct
result, trivial `existsAfter` fallback, not a silent lie or corruption) ×
reach=common-but-not-every-session (top-of-provisioning-script guard) -> Low.**
Matches the sibling create-vs-modify signal ticket `E-set-row-create-reports-updated`
(also Low/Medium-class result-label inconsistency on a create-if-missing path).

**Fix (either):** (a) make both branches return a stable `created` boolean
(`created:true` on the create branch, `created:false` on the already-exists
branch) plus a single consistent path key, so an idempotent guard can branch on
`resp.created` regardless of outcome; or (b) if the shape divergence is
intentional, document on the `blueprint.ensure_exists` wiki page
(`docs/wiki-src/blueprint.md`) that the create branch omits `created` and the
caller must read `existsAfter`/`saved` to detect a fresh create, and that the
path key differs between branches (`path` vs `blueprintPath`).

## Verbatim repro (from the Attempt trace, focus method `blueprint.ensure_exists`)

Blueprint: `/Game/Pickups/BP_HealthPotion` (Actor).

1. First `blueprint.ensure_exists` (create=true, path absent) →
   `{"path":"/Game/Pickups/BP_HealthPotion", "assetPath":..., "saved":true,
   "assetName":"BP_HealthPotion", "existsAfter":true, "assetClass":"Blueprint"}`
   — **no `created` field.** The attempt agent SAY on this response: *"The
   response doesn't explicitly say `created:true/false`, but `existsAfter:true`
   and `saved:true`."*
2. Idempotency-recheck `blueprint.ensure_exists` (same path, now exists) →
   `{"exists":true, "created":false, "blueprintPath":...}` — carries `created`.

The two responses for the same method describe the same fact (was it created?)
with different keys, and only the already-exists branch exposes it as a boolean.

Distinct from `B-ensure-exists-create-missing-name` (that was the create dispatch
hard-failing with a foreign `MISSING_REQUIRED_PARAM 'name'` error — a *broken*
create path, now IN-REVIEW/fixed). This ticket is about the *shape* the
now-working create branch returns: it succeeds but omits the `created` flag that
its sibling branch reports. Also distinct from `E-scs-add-component-not-idempotent`
(that is a hard-error-on-re-add non-idempotency in a different verb); this is a
response-shape inconsistency within one idempotent method's two success branches.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the `blueprint.ensure_exists`-focus idempotent collectible-pickup provisioning task on `/Game/Pickups/BP_HealthPotion`. CallAnalyzer (transcript `agent-a0ec3e029805cc502.jsonl`, 22 calls, outcome=done) flagged the focus method's two branches returning divergent shapes: create branch payload `{path, assetPath, saved, assetName, existsAfter, assetClass}` with NO `created`/`changed` field, vs already-exists branch `{exists:true, created:false, blueprintPath}`. Transcript-confirmed: the create response literally lacks `created`, and the attempt agent's own SAY noted it had to infer creation from `existsAfter`/`saved` because "the response doesn't explicitly say created:true/false". Filed E-/Low (docs-tagged): correct result, trivial fallback, but a branch-shape inconsistency in the method whose documented purpose is exactly to report create-vs-reuse. Not filed by the per-finding judge (whose `filed_id` was E-scs-add-component-not-idempotent, a different verb/pattern).
