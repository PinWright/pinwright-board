---
id: E-render-nanite-rebuild-async-poll-undocumented
title: "render wiki overlay never documents the nanite_rebuild_mesh async ticket → system.job_status poll pattern"
status: OPEN
severity: Low
category: ergonomic
tags: [render, nanite, nanite-rebuild, docs, async, jobs, job_status, discoverability]
encounters: 2
lastSeen: 2026-06-24T19:46:41Z
---

# render wiki overlay never documents the nanite_rebuild_mesh async ticket → system.job_status poll pattern

`render.nanite_rebuild_mesh` (also exposed as `asset.nanite_rebuild_mesh`) is an
async-job handler: per the DONE bug `B-render-nanite-rebuild-mesh-no-completion-signal`
it now `Ctx.StartJob()`s, wraps the Nanite build in an `AsyncTask`, and returns a
job **ticket synchronously** (not a rebuild result); the real
`{naniteEnabled:true, rebuilt:true}` disposition is only knowable by separately
polling `system.job_status` with that ticket until it reaches a terminal
`completed`. But the render namespace wiki overlay that should teach a caller this
two-call contract — `docs/wiki-src/render.md` — documents only the two capture
verbs (`capture_asset_preview`, `capture_open_level`) and says **nothing** about
`nanite_rebuild_mesh` at all, let alone its async/ticket/poll pattern
(`grep -niE "async|ticket|poll|job_status|nanite_rebuild" docs/wiki-src/render.md`
= no matches; the file is 88 lines and `nanite_rebuild_mesh` / `create_render_target`
are both absent).

The consequence is a discoverability gap: a caller who needs the Nanite rebuild to
have actually completed before the next step (here: re-capturing the mesh to
"eyeball that the Nanite version still looks correct") has to **self-discover** that
`nanite_rebuild_mesh` returns a ticket and that `system.job_status` is how you wait
for it. If they instead treat the synchronous response as "done" and recapture
immediately, they could capture a mesh whose rebuild had not finished — a silent
correctness trap, not just an ergonomic one.

This is the direct **render-namespace** analog of the already-filed
`E-pipeline-run-ubt-async-poll-undocumented` (pipeline overlay / `run_ubt`) and
`E-performance-run-benchmark-async-poll-undocumented` (performance overlay /
`run_benchmark`): same friction, same root pattern — an async ticket→`system.job_status`
contract that the namespace overlay never surfaces — but a different overlay page
(`render.md`) and a different async handler (`nanite_rebuild_mesh`), so it is a
distinct docs edit. It is also distinct from:

- `B-render-nanite-rebuild-mesh-no-completion-signal` (DONE) — that bug *added* the
  completion signal (the job + `CompleteJob`); the signal now exists, but nothing
  in the render overlay tells a caller it exists or how to consume it. The
  discoverability gap survives that fix because the async response shape is the
  permanent design.
- `E-static-mesh-describe-doc-promises-nanite` (OPEN, judge-filed for this same task)
  — that is the orthogonal readback angle (`static_mesh.describe`'s doc advertises a
  "Nanite state" field the response omits, forcing a `properties.json` detour). This
  ticket is the *async-discoverability* angle on the **mutator** verb, on a different
  overlay page (`render.md` vs `static_mesh.md`).

**What it should do:** The `docs/wiki-src/render.md` overlay should document, for
`render.nanite_rebuild_mesh` (and add the verb to the page at all — plus
`create_render_target`, also currently absent), that the call is fire-and-forget: it
returns a ticket synchronously (NOT a rebuild result), and callers who need the
rebuild to have finished (or who want the `{naniteEnabled, rebuilt}` payload) must
poll `system.job_status {ticket_id}` until a terminal `status`. Cross-link the
`system.job_status` doc and the `F-long-running-tickets` job model so the two-call
pattern is discoverable from the render page a catalog/thumbnail author starts on.
Mirror the wording already proposed in `E-pipeline-run-ubt-async-poll-undocumented`
and `E-performance-run-benchmark-async-poll-undocumented`.

**Workaround:** After `render.nanite_rebuild_mesh`, poll
`system.job_status {ticket_id}` (from the kickoff response) until the status is
terminal before doing anything that depends on the rebuild having finished (e.g. a
re-capture of the same mesh); treat the synchronous ticket as "accepted", never as
"the rebuild finished".

## Process friction this caused (this task)

A product-catalog thumbnails task (seed `render`) ran a clean 13-call sequence — three
`render.capture_asset_preview` shots (Cube/Cylinder/Sphere), `render.nanite_rebuild_mesh`
on `/Engine/BasicShapes/Cube.Cube`, a `system.job_status` poll, a Nanite re-capture, a
`render.create_render_target`, and three verification reads (`texture.describe`,
`static_mesh.describe`) — all `ok=true`, no retries/crashes (a clean "ergo" outcome).
The Nanite step's async-ness was pure process friction: quoting the friction note,
*"render.nanite_rebuild_mesh is async (returns a job ticket), so I had to read the
system.job_status wiki and poll once to confirm completion rather than getting an
inline result."* The agent had to self-discover the poll contract — nothing in the
render overlay pointed there — specifically because step 5 re-captures the cube to
verify the Nanite version, which depends on the rebuild having actually finished.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean product-catalog thumbnails task (seed `render`, 13 calls, all `ok=true`, no retries/crashes, outcome "ergo"). Friction note: "render.nanite_rebuild_mesh is async (returns a job ticket), so I had to read the system.job_status wiki and poll once to confirm completion rather than getting an inline result." Verified `docs/wiki-src/render.md` (88 lines) documents only `capture_asset_preview` + `capture_open_level`; `nanite_rebuild_mesh` and `create_render_target` are absent, and `grep -niE "async|ticket|poll|job_status|nanite_rebuild"` = no matches — the async ticket→`system.job_status` poll contract is undocumented in the render overlay, so a thumbnail author must self-discover the poll, a silent correctness trap since step 5 re-captures the cube to verify the rebuild took. Direct render-namespace analog of `E-pipeline-run-ubt-async-poll-undocumented` and `E-performance-run-benchmark-async-poll-undocumented` (distinct overlay page + handler); distinct from the DONE bug `B-render-nanite-rebuild-mesh-no-completion-signal` (signal now exists but is undocumented) and from the judge-filed `E-static-mesh-describe-doc-promises-nanite` (the orthogonal readback-verb over-promise angle). Ergonomic/docs, not an outcome bug — every call in the task succeeded. Fix: document the async/ticket/poll contract on `docs/wiki-src/render.md`, cross-link `system.job_status` and `F-long-running-tickets`.
- `#2-evidence-marketing-thumbnails-task` `OPEN` reporter — Second clean-outcome occurrence (seed `render`, "marketing product-shot thumbnails" task, 7 calls all `ok=true`, no retries/crashes, outcome "clean"). Call sequence: 1 wiki-nav, `render.nanite_rebuild_mesh` (Cube enable Nanite), `system.job_status` (poll the ticket), three `render.capture_asset_preview` (hero 1600x1200 / angle2 1024x1024 yaw45 / ortho-front 1024x1024), `render.create_render_target` (CatalogPreviewRT 512x512). The friction note again pins the *only* wrinkle to this exact gap, quoting: "the only wrinkle was render.nanite_rebuild_mesh returning an async job ticket rather than a synchronous result, which required one extra system.job_status poll (the wiki for that method does document the async pattern, so discovery was smooth)." Notable confirmation: the reporter rated discovery "smooth" **only because they fell through to the `system.job_status` method's own wiki page** — the render overlay still pointed them nowhere. Re-verified at audit time: `docs/wiki-src/render.md` unchanged at 88 lines, `grep -niE "async|ticket|poll|job_status|nanite_rebuild|create_render_target"` still = no matches; `nanite_rebuild_mesh` and `create_render_target` both still absent from the page the catalog author starts on. Same root gap, second task — aggregating evidence rather than re-filing. The extra `system.job_status` call is the one excess step in an otherwise minimal 7-call sequence.
