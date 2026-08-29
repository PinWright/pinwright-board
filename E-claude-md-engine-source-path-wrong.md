---
id: E-claude-md-engine-source-path-wrong
title: "The outer PDS repo's CLAUDE.md sends agents to C:\\Program Files\\Epic Games\\UE_5.8\\Engine\\Source for engine source; that directory does not exist on this host and the engine is at C:\\UE_5.8\\Engine\\Source"
status: OPEN
severity: Medium
category: ergonomic
tags: [docs, agent-facing, engine-source, host-environment, wrong-path, outer-repo]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# One wrong path in the file every agent reads first

`X:\src\unreal\unreal-fpv-new\CLAUDE.md:189` states:

```
- Unreal Engine 5.8 (engine source at `C:\Program Files\Epic Games\UE_5.8\Engine\Source\`)
```

That directory does not exist on this host. The engine source is at
**`C:\UE_5.8\Engine\Source`** — verified: `ls C:\UE_5.8\Engine\Source` lists
`Developer/ Editor/ Programs/ Runtime/ ThirdParty/ UnrealEditor.Target.cs UnrealGame.Target.cs`;
`ls "C:\Program Files\Epic Games\UE_5.8\Engine\Source"` fails.

An agent that needs to read engine source — to check an API signature, confirm what a UE call
does, or resolve which overload UE 5.8 actually ships — reaches for the documented path first,
gets a not-found, and then has to go hunting. Several agents lost time to this in a single
session. It is one line, and it is the file loaded into every session on this project before
any other context.

The correct path is already used consistently everywhere else that matters: this board's own
tickets cite `C:\UE_5.8\Engine\...` (e.g. `F-sequencer-controlrig-track` cites
`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\...`). Only the top-level `CLAUDE.md` is wrong.

## Scope note, stated plainly

This board's README scopes it to "MCP tool issues only [...] NOT game-level bugs." A wrong path
in the outer repo's `CLAUDE.md` is neither an MCP tool defect nor a game-level bug — it is a
host-documentation defect. It is filed here because the population it costs is **the agents
working this plugin**, and there is no other tracker they read. A maintainer who disagrees
should `WONTFIX` it on scope rather than let it sit; that is a cheaper outcome than the file
staying wrong.

## The fix is one line, and this ticket does not perform it

The file lives **outside** the plugin (`X:\src\unreal\unreal-fpv-new\CLAUDE.md`, the outer PDS
repo) and is read concurrently by other sessions, so it was deliberately not edited when this
ticket was filed. Change line 189 to:

```
- Unreal Engine 5.8 (engine source at `C:\UE_5.8\Engine\Source\`)
```

Two things worth doing in the same edit:

1. **Say it is host-specific.** The path is an install location, not a project fact, so it will
   be wrong again on the next machine. A parenthetical ("on this host; check
   `<engine>\Engine\Source` wherever UE 5.8 is installed") is cheaper than the next agent
   repeating this.
2. **Sweep for siblings.** `grep -rn "Program Files.*Epic Games.*UE_5"` over the outer repo's
   `*.md` also hits `docs/tester-setup.md:38` (a Russian-language installer instruction naming
   `C:\Program Files\Epic Games\UE_5.6` as the launcher *default*) and three
   `Plugins/BpGeneratorUltimate/` how-to files using `UE_5.4` paths as generic examples. Those
   three are illustrative and correct as written; `tester-setup.md` describes the launcher's
   default rather than this host and is also fine. **Only `CLAUDE.md:189` asserts a fact about
   this host, and only it is wrong.** Recorded so a fixer does not "fix" the four innocent
   ones.

## Severity

**Medium.** Impact class is Low by the rubric — pure friction, docs, no behaviour is wrong and
nothing is blocked. The reach bump is applied and is the whole argument for Medium: engine
source reads are a near-constant in this plugin's work (this board's tickets cite engine
files routinely, and several agents hit this in one session), the file is loaded into *every*
session on this project, and the failure mode is a dead end rather than a wrong answer — the
agent knows it failed but not where to go. The rubric's own line applies directly: "A
`Low`-impact gap on an every-session method outranks a `High`-impact gap on a method nobody
hits."

## History
- `#1-wrong-engine-source-path` `OPEN` reporter — Verified on this host, not inferred: `ls /c/UE_5.8/Engine/Source` succeeds and lists the expected six entries; `ls "/c/Program Files/Epic Games/UE_5.8/Engine/Source"` fails. The claim is at `X:\src\unreal\unreal-fpv-new\CLAUDE.md:189`, in the "Key Technologies" list of the Project Overview section. **The file was deliberately NOT edited** — it is outside the plugin, in the outer PDS repo, and other sessions read it concurrently; this ticket describes the change instead. Swept the outer repo for sibling occurrences (`grep -rn "Program Files.*Epic Games.*UE_5" --include=*.md`): four other hits, all correct as written (`docs/tester-setup.md:38` describes the Epic launcher's *default* install location for 5.6, and three `Plugins/BpGeneratorUltimate/` how-to files use `UE_5.4` paths as generic illustrative examples) — recorded above so a fixer does not change them. Filed with an explicit scope caveat: this board is scoped to MCP tool issues, and a host-doc defect in the outer repo is arguably outside it; filed anyway because the agents it costs are this plugin's, and `WONTFIX` on scope is a legitimate and cheap disposition. Severity Medium: Low impact class with the rubric's reach bump applied, argued above rather than assumed.
