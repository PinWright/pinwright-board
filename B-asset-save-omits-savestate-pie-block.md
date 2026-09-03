---
id: B-asset-save-omits-savestate-pie-block
title: "asset.save returns saved:false + pendingFlush:true with NO saveState when PIE blocks the write, so the caller cannot tell 'deferred, retry with force' from 'failed, retrying is pointless' — the exact distinction the wiki tells them to read"
status: OPEN
severity: High
category: bug
tags: [asset, save, savestate, pendingFlush, pie, diagnostics, false-deferral, error-payload, niagara, multi-agent]
encounters: 5
lastSeen: 2026-09-03T08:20:00+05:00
mergedFrom: [B-asset-save-omits-savestate-and-pie-cause]
---

# `asset.save` drops `saveState`, so a PIE-blocked failure is indistinguishable from a throttle

## Symptom

With PIE running in the shared editor, `asset.save {assetPath, force:true}` on a dirty
Niagara emitter returns:

```json
{"assetPath":"/Game/FPS/VFX/Emitters/E_Blood_Mist","package":"/Game/FPS/VFX/Emitters/E_Blood_Mist",
 "saved":false,"sizeBytes":83736,"pendingFlush":true}
```

There is **no `saveState` field** — and no `saveDetail`, no `saveRequested`, no `pieActive`.
`sizeBytes` reports the size of the *stale* file already on disk, which reads as reassuring.
`asset.is_dirty` on the same asset returns `isDirty:true`, so the caller can prove the edit is
real and unwritten, but not *why*.

## Why the missing field is the whole bug

`safe-mutation-save.md` § "Save States" makes `saveState` the sole basis for the retry decision:

> `pendingFlush` means "requested and not durable". It does **not** mean flushing will work —
> only `deferred` and `notRequested` are fixed by a flush. **Read `saveState` before retrying.**

The verb never emits the field the protocol is built on. Its own page tells the caller to do
the thing that cannot work here:

> When the write is throttled (`saved:false` + `pendingFlush` + `saveState: "deferred"`), an
> `editor.save_all` — or `asset.save {force: true}` — is then needed to flush it.

A caller matching on the two fields they *can* see (`saved:false` + `pendingFlush:true`)
concludes "deferred" and retries `force:true` forever. The plugin already knows better —
the **log** carries the correct verdict on the very same call:

```
LogUtils: Error: The Editor is currently in a play mode.
LogPinWrightSubsystem: Warning: SaveLoadedAssetThrottled: failed to save '/Game/FPS/VFX/Emitters/E_ImpactFlesh_Spray.E_ImpactFlesh_Spray'
LogPinWrightSubsystem: Warning: SaveAssetToDiskReportingPresence: '...' is NOT durable after this save.
  state=failed outcome=Failed forced=true dirtyBefore=true existedBefore=false sizeBefore=-1 sizeAfter=-1 ...
  The save was attempted and produced no durable revision; a flush will not help until the cause is cleared.
```

`state=failed` and "a flush will not help" are computed and written to the log, then **dropped
on the way into the RPC response**. Note also `SaveLoadedAssetThrottled` naming the *throttle*
path while the actual outcome is `failed` — the handler is reporting through the throttle
channel regardless of cause, which is plausibly where the field is lost.

`safe-mutation-save` anticipates this exact hole: *"A response carrying `saveRequested` but no
`saveState` comes from a handler not yet threaded through."* `asset.save` is that handler, and
it is the single-package writer the same document names as the preferred save for every
`niagara.*` / `material.authoring.*` / `property.set` mutation — so the gap sits on the most
recommended path.

## Repro

1. Have PIE running (any agent; `ui.stop_play` not required to observe).
2. `asset.duplicate` a stock emitter to `/Game/FPS/VFX/Emitters/<X>`; `asset.save` it (succeeds —
   this is pre-PIE or an unlocked package).
3. Mutate it: `niagara.set_static_switch`, `niagara.set_module_input`, `niagara.add_module`.
4. `asset.save {force:true}` → `saved:false, pendingFlush:true`, no `saveState`.
5. `asset.is_dirty` → `isDirty:true`. Log shows `state=failed`.

## What should happen

1. `asset.save` emits `saveState` (here `failed`) and `saveDetail` on every response — the field
   `safe-mutation-save` § "Save States" already specifies and that the handler already computes
   for the log line.
2. Include the environmental cause when it is known: `pieActive:true` / `reason:"BlockedByPie"`.
   `B-editor-save-all-pie-diagnostic` (DONE) asked for exactly this on `editor.save_all` and the
   fix was not carried to `asset.save`.
3. Do not report a `failed` outcome through `SaveLoadedAssetThrottled`; the name asserts a cause
   the code has already ruled out.
4. Do not return the stale on-disk `sizeBytes` on a failed save without a field saying the write
   did not happen — a plausible non-zero size is the most misleading part of the payload.

**Workaround:** after any `saved:false`, call `asset.is_dirty`; if still dirty, grep the editor
log for `SaveAssetToDiskReportingPresence` and read `state=` there. Do not retry `force:true`.

severity rationale: impact=silent unbounded data loss (agents author for many minutes into memory
believing a retry will flush, and lose it on the next crash — this shared editor crashed once
today at 19:44:57) x reach=the documented single-package save path for every asset-authoring verb
-> High.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8, shared multi-agent editor, port 27145) authoring Niagara blood emitters. `E_Blood_Mist` was fully authored in memory (4 static switches, 10 module inputs, 4 added modules, 2 `move_module`, 1 renderer material) and `niagara.compile` returned `status:"completed"`, but `asset.save {force:true}` returned `saved:false, pendingFlush:true, sizeBytes:83736` twice with no `saveState`; `ls` confirmed the `.uasset` still held the 83736-byte pre-edit floor. `asset.is_dirty` returned `isDirty:true`. Cause found only by grepping `Saved/Logs/EAContentExamples58.log`: `LogUtils: Error: The Editor is currently in a play mode.` plus `SaveAssetToDiskReportingPresence ... state=failed outcome=Failed forced=true ... a flush will not help until the cause is cleared` — a concurrent agent's `E_ImpactFlesh_Spray` was hitting the same wall in the same seconds. Cross-refs: `B-editor-save-all-pie-diagnostic` (DONE) requested the same diagnostic for `editor.save_all` and it was never extended to `asset.save`; `E-asset-save-force-no-clean-check` (OPEN) covers defensive force-save over-calling, which this defect directly causes. No plugin source read; diagnosis is from the RPC responses, the wiki pages `asset.save.md` / `safe-mutation-save.md`, and the log lines quoted above.
- `#2-second-agent-same-window-and-the-recommended-verb-argument` `OPEN` reporter — Merged in from `B-asset-save-omits-savestate-and-pie-cause`, which a second VFX builder filed independently for this same defect in the same PIE window (19:52-19:57Z); the two agents were authoring different emitters and neither could see the other's ticket because neither existed yet. That file has been deleted and its evidence folded here. What it adds beyond `#1`:
  - **A different `sizeBytes` on the same failure.** `#1` observed `sizeBytes:83736`, the size of the stale file already on disk. This agent observed `sizeBytes:0` on `E_ImpactFlesh_Spray`, whose package did not yet exist (`existedBefore=false`, `sizeBefore=-1`). So the field is neither a written size nor a consistent sentinel — it is whatever the pre-existing file happened to be — which makes it actively misleading in both directions rather than merely uninformative.
  - **Three consecutive byte-identical retries.** `asset.save {force:true}` was called three times on `E_ImpactFlesh_Spray` and returned exactly `{saved:false, sizeBytes:0, pendingFlush:true}` each time, with `asset.is_dirty` confirming `isDirty:true` throughout. Per `safe-mutation-save.md` a `failed` save is not flushable, so all three were guaranteed no-ops the response gave no way to predict — the wasted-retry cost of the missing field, measured.
  - **Why the existing PIE fix does not reach this verb, argued.** `B-editor-save-all-pie-diagnostic` (DONE) added `pieActive` / `editorMode` / `failedAssets[]` with a `BlockedByPie` reason to `editor.save_all` only. But `safe-mutation-save.md` tells callers to *prefer* `asset.save` as the narrow single-package writer, and in a shared multi-agent editor it is the only save that is safe to call at all, because `editor.save_all` flushes every other agent's dirty packages. The diagnostic therefore landed on the blunt verb and not on the recommended one, which is why this gap survived a fix that looks like it should have closed it.
  - **A concrete reuse for the fix:** thread the already-computed classification out through `saveState` + `saveDetail` and add `pieActive` / `editorMode` alongside, reusing `EditorSaveAllDiagnostic::ClassifyFailureReason` from the `B-editor-save-all-pie-diagnostic` fix so both save verbs answer in one vocabulary.
  - **An extra cross-ref:** `E-compile-bpir-no-persistence-field` (OPEN) is the same family — a writer whose response understates persistence — on a different verb.
  - **An in-band-observability point `#1` does not make:** an agent authoring assets has no in-band way to learn that another agent started PIE, so it keeps authoring into memory against a save path guaranteed to fail. The prescribed workaround (grep the editor log for `SaveAssetToDiskReportingPresence`) requires filesystem access to `Saved/Logs/`, which an agent driving the editor purely over MCP does not have — for that caller the defect has no workaround at all.
  No status change; severity stays High and the two filings agree on it by different routes (`#1` from data-loss impact, the merged one from reach across every authoring session).
- `#3-mid-repair-stall-across-two-streams` `OPEN` reporter — Third encounter, on the FPS Niagara VFX repair stream in the same shared editor, and the one that shows the cost is not a lost asset but a **stalled multi-step repair**. Context: I was walking 13 emitters through a fixed per-emitter sequence (`remove_module` → `set_module_input` x2 → `niagara.compile {force,wait}` → `asset.save {force:true}` → `remove_emitter` + `add_emitter` on the owning system → compile → save), a sequence adopted precisely because three editor-killing crashes today had shown that batching loses work. Nine emitters and eight systems went through it and are on disk (mtimes 23:51–23:56 local, verified with `ls`). The tenth, `/Game/FPS/VFX/Emitters/E_Explosion_Fireball`, returned `{"saved":false,"sizeBytes":108204,"pendingFlush":true}` to three consecutive `asset.save {force:true}` calls, with no `saveState` and no mention of PIE. `sizeBytes` was the stale on-disk size, matching `#2`'s point on this ticket's sibling. The log carried `LogUtils: Error: The Editor is currently in a play mode.` + `SaveAssetToDiskReportingPresence ... state=failed outcome=Failed forced=true ... stampBefore == stampAfter`. Cause: an unrelated stream had entered PIE on `/Game/FPS/Test/T_AI` at 20:56:58Z, legitimately and under the project's world-lock protocol.

  What is new here, and what I would put in the fix's motivation: **while any agent holds PIE, `asset.save` converts every asset write in the shared editor into a silent no-op that only the editor log reveals.** Not a slow write, not a queued write — a no-op reported as a queued write. Three consequences this sighting makes concrete. (1) It is *cross-stream by construction*: the blocked writer and the PIE holder are different agents doing unrelated, individually correct work, so nothing in the blocked agent's own actions hints at the cause, and `pendingFlush` actively points away from it. (2) It is *time-varying*, so an identical call succeeds and then fails minutes apart on the same asset — nine identical saves succeeded immediately before this one failed. (3) For any workflow that saves per step in order to be crash-safe, a mid-sequence false `pendingFlush` is worse than an error: the honest response is to stop and hold one dirty asset, but the payload's plain reading is "queued, carry on", which would have had me push three more emitters into memory in a session that has already lost a full wave of Niagara systems to a crash. The verb had the truth (`state=failed`) and answered with the opposite signal. Ask is unchanged and is this ticket's existing one — emit `saveState` on the failure path and name PIE as the blocker (`editor.save_all` already returns `reason:"BlockedByPie"`, and `editor.pie_status` already knows the map) — with one addition: since `force:true` is documented as the remedy for `pendingFlush` and does **not** bypass PIE, a `pendingFlush` that `force` cannot clear should not be spelled `pendingFlush` at all. Disk proof was `ls` mtime only; `asset.reload` is now forbidden project-wide after it crashed the editor earlier today (`B-asset-reload-access-violation-kills-editor`), which removes the one verb that would otherwise confirm a save round-trips.
- `#4-flag-is-not-even-consistent-within-one-batch` `OPEN` reporter — Fourth encounter, ENV stream, same shared editor, another agent (`UI-critic`) holding PIE with `"pie": true` in the world lock. New datum: **the `pendingSave` flag is not a reliable indicator even when it is present, because it is not applied consistently to identical calls in the same second.**

  Four back-to-back `material.authoring.set_material_instance_parameters {save: true}` calls on four material instances in one folder, issued together, all four with `existsOnDisk: true` and a full `applied[]` list and an empty `failed[]`:
  ```
  MI_ENV_Concrete_Barrier   applied 3   failed []   pendingSave: true
  MI_ENV_Concrete_Worn      applied 3   failed []   (no pendingSave key at all)
  MI_ENV_Concrete_Floor     applied 3   failed []   pendingSave: true
  MI_ENV_Sandbag            applied 1   failed []   pendingSave: true
  ```
  **None of the four reached disk.** Verified the way this project verifies every write — `ls` the `.uasset` mtime, not a read-back of the in-memory object:
  ```
  MI_ENV_Concrete_Barrier   05:30:26      MI_ENV_Concrete_Worn   05:30:23
  MI_ENV_Concrete_Floor     05:30:17      MI_ENV_Sandbag         04:34:26
  wall clock at the check    05:35:00
  ```
  So `MI_ENV_Concrete_Worn` reported a clean success with no persistence caveat of any kind and wrote nothing. An agent that trusted the flag as a filter — retry the ones marked pending, accept the ones that are not — would have shipped that one silently stale. That is worse than `#1`'s uniform-but-uninformative `pendingFlush`, because an inconsistent flag actively invites the wrong filter.

  It also widens the reach recorded in `#2`: this is `material.authoring.set_material_instance_parameters`, not `asset.save`, so the whole authoring family that saves on the caller's behalf carries the defect and each spells the caveat differently or not at all.

  **What the response needed to say** and did not, on all four: that a PIE session was active and the write was therefore not attempted. The editor knows; `editor.status` reports `inPie` in the same process. Any verb that writes should consult it before claiming a save.

  **Workaround in force on this build:** treat `save: true` as advisory during any period another stream may hold PIE; poll `editor.status` for `inPie: false`, re-issue the save, and confirm against the `.uasset` mtime on disk before believing it. `encounters` 3 -> 4, `lastSeen` refreshed.
- `#5-compile_material-is-the-third-verb-in-the-family` `OPEN` reporter — Fifth encounter, FPS VFX stream, same shared editor, `UI` holding PIE on `/Game/FPS/Test/T_UI` (log: `PIE: Created PIE world ... UEDPIE_0_T_UI` at 03:12:23Z, no teardown for the next several minutes). Two things this sighting adds.

  **`material.authoring.compile_material {save:true}` carries the defect too, and it is the worst-placed of the three**, because the same payload that hides the failed save is otherwise unambiguously green:
  ```json
  {"compileSucceeded":true,"compileStatus":"completed","compiledWithErrors":false,"compileErrors":[],
   "consumerRefresh":{"measured":true,"complete":true},
   "saveRequested":true,"saved":false,"pendingFlush":true}
  ```
  Every field a caller is told to branch on says success; the two that matter for durability are the generic pair. `M_FPS_Smoke_Lit` (a master material with a new `AmbientBoost` scalar wired to `EmissiveColor`) stayed at its pre-edit 23582 bytes / `stampBefore == stampAfter == 2026-09-03T02:19:28Z` through that call and four subsequent `asset.save {force:true}` calls. So the family is now three verbs deep — `asset.save` (`#1`-`#3`), `material.authoring.set_material_instance_parameters` (`#4`), `material.authoring.compile_material` (here) — each spelling the same non-durability differently, and none naming PIE.

  **The prescribed workaround does not survive an undeclared PIE.** `#4`'s remedy is to read the project world lock and poll for `inPie:false`. Here `Saved/PinWright/fps/WORLD_LOCK.json` **did not exist** — the PIE holder never declared it — so the lock said "free" while every save in the editor was a no-op. The only in-band signal that worked was `editor.pie_status`, which answered correctly and specifically (`inPie:true, count:1, mapName:"T_UI", worldPath:"/Game/FPS/Test/UEDPIE_0_T_UI.T_UI"`) at the same moment `asset.save` was reporting an unattributed `pendingFlush`. That is the whole ask restated as a two-line fix: the process that refuses the write already holds the answer one RPC away, and returns it to anyone who asks directly.

  One further datum for `#4`'s inconsistency point, reproduced on a different folder: of three `set_material_instance_parameters` calls issued back to back during this PIE window, `MI_FPS_Smoke_Grenade` and `MI_FPS_Smoke_GrenadeWisp` carried `pendingSave:true` and `MI_FPS_Smoke_GrenadeCore` carried no `pendingSave` key — same verb, same second, same PIE block, same outcome on disk.

  Also confirmed, as a boundary on the impact: an `asset.save {force:true}` issued in the ~70 s gap between two other streams' PIE sessions (`T_Player` teardown 03:11:14Z, `T_UI` start 03:12:23Z) wrote normally — `MI_FPS_Glass_Dust` 11369 -> 11808 bytes at 03:11:33Z — after two identical calls seconds earlier had returned `state=failed`. Time-varying exactly as `#3` describes. `encounters` 4 -> 5, `lastSeen` refreshed.
