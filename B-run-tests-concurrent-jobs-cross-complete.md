---
id: B-run-tests-concurrent-jobs-cross-complete
title: "Concurrent system.run_tests jobs share one automation controller and can change or complete each other's selection and result"
status: OPEN
severity: High
category: bug
tags: [system, automation, jobs, concurrency, global-state, false-success]
encounters: 1
lastSeen: 2026-09-03T23:17:31+03:00
---

# Concurrent `system.run_tests` jobs are not isolated

## What happens

Every exact-name job obtains the process-global controller from
`IAutomationControllerModule::GetAutomationController()`
(`Handlers/System/SystemControlHandler.cpp:176-179`), calls `Controller->Init()`, and binds the
same controller-wide `OnTestsRefreshed` and `OnTestsComplete` delegates at `:181-193`. After
discovery it overwrites the shared filter and enabled-test selection, calls `StopTests`, and starts
its selection at `:249-291`.

The filter/run-all path obtains that same controller at `:526-529` and adds another uncorrelated
`OnTestsComplete` lambda at `:531-542` before executing the console command. There is no active-run
guard or owner token around either path. Two calls can therefore coexist as separate PinWright
jobs while operating one controller: the later call can replace the first selection, and the first
controller completion broadcast can run both delegates. Both job tickets can then terminate from
one shared `ReportsHaveErrors()` value even though only one requested selection finished.

## Why it matters

Multi-client and multi-agent use can produce a green terminal receipt for a test selection that
did not run, or attribute one run's failures to another job. That is a High silent false-success
path, not merely inefficient parallelism.

## What should happen

Give the automation controller one explicit PinWright owner at a time. The simplest fix is an
active-run lease shared by both selection paths: a second `system.run_tests` call fails with a
typed job-conflict error until the owner reaches a terminal state. Bind one-shot delegates to an
owner generation and remove them on every terminal/cancel path. Add a two-call test proving the
second request cannot alter or complete the first.

**Workaround:** Serialize `system.run_tests` calls and wait for the prior job's terminal status.

## Related

- Catalog: `cross-caller-global-state-leak`, `premature-async-success`
- `B-system-run-tests-no-completion-signal` — added job completion but does not isolate concurrent jobs.
- `F-system-run-tests-multiple-names` — introduced the exact-name controller path.

## History
- `#1-shared-controller-cross-completion` `OPEN` reporter — Source-read both selection paths and
  confirmed they share one controller and its multicast completion delegate with no active-owner
  guard. No automation run was started.
