---
id: E-log-stream-no-readback-method
title: "log namespace describes a subscribe/buffer/read-back loop but ships no read method — docs never state there is NO log.poll/read/pull, forcing a trial-and-error UNKNOWN_ACTION probe"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, log, log-subscribe, log-unsubscribe, wiki, discoverability, readback, placeholder]
---

# The `log` namespace implies a readable capture buffer but exposes no read-back method, and the docs never say so

The `log` namespace registers exactly two methods, `log.subscribe` and
`log.unsubscribe`. Both are intentional placeholders and **honestly
self-document** that streaming is not wired up — the registration summaries
(`LogHandler.cpp:10` "Currently NOT implemented — the call returns
subscribed=false…"; `:25` "currently a no-op since subscribe is unimplemented.
Always returns subscribed=false") render into both the `## Methods` index and
the per-method pages, and the overlay `docs/wiki-src/log.md:3` already flags the
placeholder and points at the two working fallbacks (tail
`Saved/Logs/<ProjectName>.log`; `editor.console_command` with
`Log <Category> <Verbosity>`). So this is **not** a wiki-over-advertises ticket
(cf. `E-ik-rig-family-wiki-advertises-compiled-out-workflow`,
`E-texture-create-wiki-advertises-stub`) — here the docs are honest about the
no-op, and the judge correctly filed nothing.

The residual, distinct PROCESS gap is the **read-back leg of the implied
contract**. The names and summaries describe a subscribe → buffer →
read-back → unsubscribe loop ("streaming editor log lines **back to the
caller**"; `unsubscribe` = "Tear down any in-process log capture device set up
by log.subscribe"), and `PluginState.h:37-39` even carries a
`Get/SetLogCaptureDevice` slot for it. A caller who reads that contract
reasonably looks for the third verb — the one that reads whatever the
subscription buffered — but **no read method exists** (`log.poll`,
`log.read`, `log.pull`, `log.drain` are all absent). Nothing in the namespace
overlay or either method page states that the pair is read-less, so the only
way to discover it is to **guess a method name and get `[UNKNOWN_ACTION]`**.

## What's awkward
- The subscribe/unsubscribe pair advertises a capture buffer ("read back
  whatever the subscription buffered") with no companion verb to read it and
  no docs note that none exists.
- Discovering the absence requires a trial-and-error probe rather than a docs
  read: the only signal is an `UNKNOWN_ACTION` error on a guessed method.
- The two-method namespace lists no third method in its `## Methods` index, but
  that absence reads as "not yet added," not as "there is deliberately no
  read-back path — use the on-disk log instead."

## What the wiki page should do
In `docs/wiki-src/log.md` (the overlay; the edit itself is downstream wiki
process, not this audit's job), add one explicit sentence to the existing
placeholder note: while streaming is unimplemented, the namespace exposes **no
read-back / poll method** (no `log.poll`/`read`/`pull`) — the captured-buffer
language in the `subscribe`/`unsubscribe` summaries is aspirational, and the
working way to read log output today is to tail
`Saved/Logs/<ProjectName>.log` (already named on the page). That converts the
trial-and-error `UNKNOWN_ACTION` probe into a docs read.

## Evidence
Task focus `log` (namespace walk-through), 14 calls, outcome **clean** /
self-reported "Mostly smooth" — the wiki correctly flagged the placeholder and
the fallbacks worked. The lone friction in the note: *"there is no read-back
method (no log.poll/read/pull) for the subscription's claimed buffer, so the
only way to verify capture is to tail the log file directly."* In the call log
this shows as exactly one `is_error`: `log.poll` → `[UNKNOWN_ACTION] Unknown
action: log.poll` (the probe for a read-back method), surrounded by clean
`log.subscribe` (×3, broad + category-scoped, all `subscribed=false`) and
`log.unsubscribe` calls. One avoidable trial-and-error probe on an otherwise
smooth task; the fix is a single overlay sentence. Page to improve:
`docs/wiki-src/log.md`.

## History
- `#2-overlay-no-readback-note` `IN-REVIEW` developer — Added a `## No read-back / poll method` section to the `docs/wiki-src/log.md` overlay (`Plugins/PinWright/Docs/wiki-src/log.md`) stating explicitly that the namespace ships only `log.subscribe`/`log.unsubscribe` and has NO read-back/poll verb (`log.poll`/`read`/`pull`/`drain` all return `[UNKNOWN_ACTION]`), that the "stream log lines back" / "in-process log capture device" wording is aspirational (subscribe always returns `subscribed=false`), and steering the caller to tail `Saved/Logs/<ProjectName>.log` instead of probing a polling verb. This renders onto the `log` namespace page (`##` section, namespace-page-visible, hidden from the root index per the prelude-only root cutoff) and converts the trial-and-error `UNKNOWN_ACTION` probe into a docs read. Regression test `FWikiHandlerLogNamespaceDocumentsNoReadbackTest` (`PinWright.infra.wiki_handler.Namespace.LogDocumentsNoReadback`) in `Plugins/PinWright/Source/PinWright/Private/Tests/Infra/TestWikiHandler.cpp` renders `WikiHandler::RenderPage("log")` and asserts the overlay-exclusive markers (`no read-back`, `log.poll`, `do not probe`) — none of which appear in either `LogHandler.cpp` registration summary — so it fails if the section is reverted. No handler/code behavior changed (docs/overlay only).
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `log` namespace walk-through (14 calls, outcome clean, self-reported "Mostly smooth"; judge filed nothing — correctly, since the no-op is intentional and `docs/wiki-src/log.md:3` + the honest registration summaries at `LogHandler.cpp:10/:25` already document the placeholder and the on-disk-log / `editor.console_command` fallbacks). Distinct PROCESS angle from the accepted `wiki-advertises-*` precedents: the docs here are honest about the no-op, so this is NOT an over-advertisement ticket. The residual gap is the read-back leg of the implied subscribe→buffer→read→unsubscribe contract — the names/summaries advertise a capture buffer ("stream log lines back to the caller"; "in-process log capture device set up by log.subscribe") and `PluginState.h:37-39` reserves a device slot, but there is NO read method (`log.poll`/`read`/`pull` all absent) and no docs note saying so, so the only way to learn the pair is read-less is a trial-and-error guess → the one `is_error` in the log, `log.poll` → `[UNKNOWN_ACTION]`. Fix is a single overlay sentence in `docs/wiki-src/log.md`: state there is no read-back/poll method and point at tailing `Saved/Logs/<ProjectName>.log` (already named on the page). Severity Low — one avoidable probe on an otherwise smooth task. Dedup: no existing `*-log-*` board ticket; the `readback`-tagged tickets (`F-sequencer-track-state-readback`, `F-skeleton-skin-weight-profile-readback`, etc.) are unrelated namespaces.
