---
id: H-test-loop-unreviewed-behavior-expansion
title: "Escalated reviewer concerns force the test loop to implement new production behavior that no adversarial reviewer ever sees"
status: OPEN
severity: Medium
workflow: fix
category: harness-process
knob: middleware
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# Escalated reviewer concerns force the test loop to implement new production behavior that no adversarial reviewer ever sees

The diff-review fixer escalated concern-1 as 'a behavior expansion (unload path, plus a result-contract decision)'; the test-loop prompt converts every escalated item into a MUST-fix, so the test-loop agent implemented a force-unload path (UnloadPackages with bUnloadDirtyPackages=true), a changed helper signature, and a new unloadedCount response field AFTER both context-isolated adversarial reviewers had run. The commit phase publishes testResult.touched directly (workflow js ~line 772) — the 'no production behavior ships without adversarial review' invariant breaks precisely for the least-supervised changes, and this same run's earlier review pass caught a real bug the author's green test missed. Recurs whenever the fixer escalates an expansion (which it is told to do). Fix ADDS review coverage, consistent with the quality-over-cost preference.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a2899fc97170a5c5c.jsonl
- quote: "a behavior expansion (unload path, plus a result-contract decision on how unloads count) requiring its own reverted-add fixture"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
Insert after the test-loop call (anchor verified at line 753: `const testResult = await agentOrRetry(() => agent(testLoopPrompt(ticket.id, impl2.testAdded, reviewResult), { model: runModel, schema: testSchema, label: \`test-loop:${ticket.id}\` }), 5);`):
+    // Re-review gate: the test loop may implement escalated concerns = NEW production behavior
+    // that the two diff reviewers never saw. If it touched non-test production files and went
+    // CLEAN, run ONE more adversarial review + fixer pass over the now-current diff, then
+    // re-enter the test loop (its SALVAGE CHECK parses the existing green log when nothing changed).
+    const isProd = (p) => p && !/[\\\/]Tests[\\\/]/i.test(p);
+    if (testResult && testResult.finalState === 'CLEAN' && (testResult.touched || []).some(isProd)) {
+        phase('Diff-Review');
+        const postReview = await agentOrRetry(() => agent(advReviewPrompt(3, ticket), { model: runModel, label: `diff-review:post-test:${ticket.id}`, phase: 'Diff-Review' }));
+        const postApply = await agentOrRetry(() => agent(reviewApplyPrompt(ticket.id, [postReview]), { model: runModel, label: `review:apply:post-test:${ticket.id}`, phase: 'Diff-Review' }));
+        phase('Test');
+        const testResult2 = await agentOrRetry(() => agent(testLoopPrompt(ticket.id, impl2.testAdded, postApply), { model: runModel, schema: testSchema, label: `test-loop2:${ticket.id}` }), 5);
+        if (testResult2) Object.assign(testResult, testResult2, { touched: [...new Set([...(testResult.touched || []), ...(testResult2.touched || [])])] });
+    }
(Adapt helper names advReviewPrompt/reviewApplyPrompt/phase to the file's actual identifiers when applying.)

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
