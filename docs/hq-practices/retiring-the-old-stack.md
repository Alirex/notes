---
summary: "Replacing a backend piece by piece: freeze the oracle, tag before each removal, test through the new stack, and agent habits that kept the gate honest"
group: agents
tags: [migration, agents, quality-gate, process]
applies-to: [all]
reviewed: 2026-11-01
status: draft
---

# Retiring the old stack

Lessons from moving a Python backend to Rust one route group at a time, with parallel agents. They apply to any
strangler-style rewrite.

## Rule

- **The old code is the oracle until the new code is covered, then freeze it.** Contract tests compare the new stack
  with recorded outputs of the old one. Once a piece is ported, commit the recorded outputs as fixtures, list them in a
  file that a test checks, and stop running the old recorders in the gate. Re-recording is a manual step from a tag.
- **Tag, then remove, in a separate commit.** `pre-remove-<what>` marks the last commit that has the old code; the
  removal commit contains nothing else. Remove only when no caller is left (CLI, scheduler, bot, web API, tests).
- **The gate must run the production path.** An e2e stage that starts only the old stack blocks removing the old
  routes: 13 specs failed the moment four routes went. Run e2e through the new stack (the real edge, the real
  switches) before any removal; skips for specs the new stack cannot serve yet need a `Gate-change:` line and a
  backlog item that removes them.
- **A generated API description survives removals.** Keep a small overlay of retired routes merged into the live
  schema, so the frontend types and contract tests do not change when handlers go.
- **Check the premise of a "port X" item.** Before porting a parser, confirm the module reads the files at all; one
  item was docs only because the module read no DOCX.
- **New work goes to the new stack.** State it as a binding rule in the repo's agent file, with the rare exception
  needing a maintainer's say-so, and brief every worker with it.

## Running agents in parallel

- **One gate per worktree.** Two gates in the same tree collide (shared build output, shared e2e ports) and both fail
  for reasons that say nothing about the branch. Before starting a gate, look for a running one and its worktree.
- **A quiet agent is not a finished agent.** Agents stop "waiting on background work" many times. Read the branch with
  git instead of waiting for the report, and stop duplicate runs by PID.
- **After merging unrelated commits, re-run the fast check; after merging another branch's compiled code, re-run the
  full gate.**
- **Fast checks that look only at changed files surface old debt in touched files.** Expect it; fix it or file it,
  do not skip the check.
- **Suppressions need a reason and removed tests need a `Gate-change:` line** so the integrity check can tell a
  retired test from a weakened one.

## Sources

- Martin Fowler, "Strangler Fig": https://martinfowler.com/bliki/StranglerFigApplication.html
- Feathers, "Working Effectively with Legacy Code", characterization tests (the recorded outputs are these).
- [agent-workflow.md](agent-workflow.md), [quality-gate.md](quality-gate.md), [change-discipline.md](change-discipline.md)
