---
id: B-startjob-inline-verbs-reply-running-before-work
title: "Ticketed verbs whose bind delegate finishes inline still reply status:running before the work runs"
status: OPEN
severity: Low
category: bug
tags: [jobs, start-job, transport, honesty, scripting]
encounters: 1
lastSeen: 2026-10-01T00:00:00Z
---

# Inline ticketed verbs reply "running" before their work

`FHandlerContext::StartJob` sends the non-streaming `status:"running"` envelope to the transport
before it invokes the bind delegate. For a verb whose delegate does all of its work inline on the
game thread (it is not a deferral primitive, `docs/rpc-design.md` section 9), the transport can flush
that envelope while the work is still running, so a fast scripted client acts on it before the
verb's output exists. `B-editor-screenshot-returns-before-png-exists` hit this for
`editor.screenshot`; that verb now opts in to `FJobBindArgs::bCompletesInBind`, which makes
StartJob reply after the delegate with the terminal outcome.

Other inline verbs still take the early reply, e.g. `render.capture_ortho_tiles`
(`OrthoTileCaptureHandler.cpp`, documented as synchronous) and the audio analysis/synth verbs
(`AudioAnalysisHandler.cpp`, `AudioSynthGenerateHandler.cpp`). Not audited one by one: each verb
needs a check that its delegate always completes before returning, then `bCompletesInBind = true`
and a reply-shape test. Verbs whose delegate defers (lighting, navigation, MRQ, asset dumps) must
not opt in.

## History
- `#1-found-in-screenshot-fix` `OPEN` developer - Found while fixing B-editor-screenshot-returns-before-png-exists; only editor.screenshot was converted, to keep that change scoped.
