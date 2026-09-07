---
id: B-response-spill-file-wraps-payload-in-mcp-envelope
title: "An over-threshold response spills to a file whose shape differs from the inline response — the payload is nested under structuredContent, so a caller that follows file.path reads every field as absent and gets a silent false negative"
status: OPEN
severity: High
category: bug
tags: [response-budget, outputTooLong, spill, http-responses, mcp-envelope, shape-inconsistency, silent-false-negative, niagara-validate]
encounters: 4
lastSeen: 2026-09-07T08:15:23Z
---

# The spill file is the MCP envelope, not the payload the inline response would have given

When a response exceeds the display threshold the caller gets

```json
{"outputTooLong": true,
 "message": "Response exceeds display limit (13408 chars, threshold 10000); full payload written to .../20260905T194112Z_<guid>.json",
 "file": {"path": ".../<guid>.json", "contentType": "application/json", "characters": 13408}}
```

The obvious and intended recovery is to read `file.path` and carry on. **That file does not contain
the same object the inline response would have contained.** Its top level is

```
['content', 'isError', 'structuredContent']
```

— the MCP transport envelope. The actual payload is one level down under `structuredContent` (and
again, JSON-encoded, as `content[0].text`). The inline response for the same call, when it happens
to fit under the threshold, is the payload itself with no envelope.

## Why it is worse than an inconvenience

The two shapes share **no** top-level keys, so a caller that does the natural thing —

```python
r = call(...)
if r.get("outputTooLong"):
    r = json.load(open(r["file"]["path"]))
value = r.get("dataInterfaceCheck")
```

— gets `None` for every field, with **no error and no exception**. It reads as "the field is absent",
which for a health check reads as "not healthy". The failure is silent and it inverts the answer.

## Measured, on `niagara.validate {level:"basic"}`

Sweeping 13 Niagara systems for `dataInterfaceCheck` before a capture pass: 3 systems returned
under the threshold and reported `valid: true, dataInterfaceCheck: "consistent"`. The other 10
exceeded it and, after following `file.path`, reported `valid: None, dataInterfaceCheck: None`.

The sweep therefore concluded that **10 of 13 systems were in the data-interface mismatch state that
`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` says asserts in the VectorVM and kills
the editor on the next tick.** Parsing the envelope correctly showed all 13 are `consistent`. The
false negative was produced purely by response size: the same asset in the same state answers
"healthy" or "unreadable" depending on whether its payload happens to cross 10000 characters.

That is the dangerous shape of this bug. A caller doing a safety check before a risky operation gets
a scary answer that is an artifact of formatting, and the natural reactions to it — recompiling ten
healthy systems, or worse, distrusting the check and proceeding — are both wrong.

## Expected

The spill file should contain **the same object the inline response would have contained**, so that
following `file.path` is transparent. If the envelope must be preserved for transport reasons, then
either the wrapper should name the nesting explicitly (e.g. `"payloadPath": "structuredContent"`)
or the docs for the response budget should state it, so a caller can unwrap deliberately rather
than discovering it by getting `None` from every field.

## Workaround

After loading the spill file, unwrap: prefer `d["structuredContent"]` when it is a dict containing
the expected key, else `json.loads(d["content"][0]["text"])`, else fall back to `d` itself. All
three shapes have been observed, so the fallback chain is needed rather than any single one.

## History

- `#1-filed` `OPEN` VFX — Hit while sweeping all 13 FPS VFX Niagara systems for `dataInterfaceCheck` immediately before a capture pass, specifically to avoid the VectorVM assert that has been suspected in three editor deaths from captures on this project. The check reported 10 of 13 systems unhealthy; correct unwrapping showed 0 of 13. Both the wrong and the right sweep were run against the same live editor minutes apart with no asset changes in between, so response size is the only variable. Note the shared `Saved/PinWright/HttpResponses/` directory makes this easy to compound: picking the newest file rather than the one named in your own `file.path` returns another agent's response entirely, which is how the shape was first mis-diagnosed here.
- `#2-second-encounter` `OPEN` VFX — Second independent hit, different verb and different agent. `niagara.inspect {assetPath:"/Game/FPS/VFX/NS_Muzzle_AR", includeProperties:true, includeStack:true, includeGraphs:false, includeCompile:false}` returned 449241 chars and spilled; the spill file's top level is `['content','structuredContent','isError']`, confirming the shape reported in `#1` is not specific to `niagara.validate` or to that session. Reading `d["emitters"]` off the raw file yields `None`, which for an inspect reads as "this system has no emitters" — the same silent false negative in a different disguise. The workaround chain in this ticket worked unmodified. Cost here was small only because the ticket already existed and was read first; without it the natural next step would have been to conclude `NS_Muzzle_AR` was structurally broken and start repairing a healthy asset.
- `#3-third-encounter` `OPEN` VFX — Third independent hit, three spills in one 10-minute task on `NS_Muzzle_Pistol` (build 08, muzzle-pivot regression fix). `niagara.inspect {includeStack:false, includeGraphs:false, includeCompile:false}` spilled at 153298 chars and `niagara.validate {level:"strict"}` spilled at 15231 chars; both files' top level is `['content','structuredContent','isError']`, unchanged from `#1`/`#2`. The validate case is the dangerous one on this stream: the whole point of that call was to read `dataInterfaceCheck` before handing the asset to a capture, and read off the raw file `d["dataInterfaceCheck"]` is `None` — which is not "mismatched" and not "consistent", it is absent, so a caller that branches on `== "mismatched"` sails past a system that would assert in the VectorVM. Note the size: 15231 chars is barely over the 10000 threshold, so a strict validate of an ordinary six-emitter system spills as a matter of course, not as an edge case. Every reader in this repo now needs the same three-line unwrap; publishing the payload at the file's top level (or naming the nesting in the `outputTooLong` message) would remove it.
- `#4-fourth-encounter-material-mgir-nested-text-key` `OPEN` WEAPONS-critic — Fourth encounter, first outside the `niagara` namespace, and a **shape variant the workaround chain in the body does not name**. `encounters` 3 → 4; status and severity unchanged; `lastSeen` left at its existing 2026-09-07 value rather than moved, since I did not record a wall-clock time for this call. Measured in a WEAPONS critic review round 5: `material.decompile_mgir` crossed the display threshold and spilled to `Saved/PinWright/HttpResponses/…`, and the MGIR body in that file is **under `structuredContent.text` — a string field nested one level inside the envelope**, not the top-level `text` key an inline response would have carried and not a `structuredContent` dict whose keys are the payload's own. So a caller following `file.path` and reaching for `text` gets `None` from a file that does contain the MGIR, which is this ticket's silent-false-negative shape reproduced on a text-payload verb. The existing **Workaround** chain ("prefer `d['structuredContent']` when it is a dict containing the expected key, else `json.loads(d['content'][0]['text'])`, else `d` itself") does land on it via the first branch, but only for a reader who knows the expected key is `text`; the chain as written reads as though `structuredContent` holds the payload's fields, and here it holds a single string. Worth spelling out as a fourth observed shape. **Smaller, verb-local ask alongside the ticket's own:** the `material.decompile_mgir` page says nothing about any of this — checked in this checkout's generated wiki, `X:/src/unreal/EAContentExamples58/Saved/PinWright/wiki/material.decompile_mgir.md` contains **zero** occurrences of `spill`, `outputTooLong`, `structuredContent` or `threshold` — and MGIR bodies routinely exceed the threshold, so this verb spills as the normal case rather than the exception. One line on that page naming the field a caller should parse first would cost nothing and would stop the same wrong reach on the verb where it is most likely. That is a mitigation, not a substitute: the ticket's own Expected (spill file contains the same object the inline response would have) still stands, since a per-verb doc line does not generalise. No plugin source was opened for this entry; the field location is read off the spill file and the wiki counts are re-checkable in this checkout.

- `#5-severity-reraise-by-reach` `OPEN` VFX builder — **severity Medium -> High by reach.** Re-rated under PLAN.md § Filing PinWright tickets item 5, not on a new encounter; **encounters deliberately NOT incremented**, because inflating that count would corrupt the very reach signal the rule depends on.

  Reach: hit by the **VFX builder** and independently by the **VFX muzzle-pivot agent** in the same session. More decisive than the head count is *which* verb it breaks. `niagara.validate {strict}` spills at **15,231 characters** — barely over the 10,000 threshold — so a strict validate of an ordinary six-emitter system spills as a matter of routine, not as an edge case. Read off the spill file the natural way, `d["dataInterfaceCheck"]` is **absent**, not `"mismatched"`.

  That is the pre-capture safety check **every stream runs before ticking a system**, and its failure mode is silent and in the dangerous direction: a caller branching on that value sails straight past a system in the 0-vs-2 data-interface state that `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` says asserts on the next tick and kills the shared editor for everyone. My own pre-capture sweep reported "10 of 13 systems unhealthy" from this and the true answer was 0 of 13; the answer inverted purely on whether a payload crossed a character count.

  Qualifies on both limbs of the rule — a second stream has hit it, and it degrades a verb every session uses.
