---
summary: "Named checklists a reviewer applies to a diff (roles as lenses, not personas), gate integrity, and the backlog item as handoff"
group: agents
tags: [agents, review, gate]
applies-to: [all]
reviewed: 2026-11-04
status: stable
---

# Review lenses

A "role" is useful when it is a checklist, not a persona. A reviewer (a subagent on a higher tier than the worker, the
local model, Junie `--review`, or the supervisor) gets the diff plus the names of the lenses to apply. Which tier does
the review: [subagents-and-context.md](subagents-and-context.md#pick-the-cheapest-tier-that-works). The stages and who
owns them: [agent-workflow.md](agent-workflow.md).

## Lenses

| Lens | The question | Typical finding |
| --- | --- | --- |
| Attack the premise | What assumption does this change rest on, and how would it be proven false? | Two fixes in a row on one wrong assumption; state it falsifiably and test it first |
| Whole requirement | Does the diff cover every sentence of the request, not the easiest one? | A flag added to the CLI but not to the web path |
| Minimal diff | Is every changed line needed for the request? | Drive-by renames and reformatting that hide the real change |
| Explain every number | Where does each figure (limit, timeout, size, count) come from: command, sample, noise? | A magic `300` with no measurement |
| Idempotency | What happens if this runs twice, or halfway and again? | A migration or job that duplicates rows |
| Single writer | Who else writes this state (file, table, branch, service)? | Two sessions in one repo, two copies of a service meant to run once |
| Search before building | Does the repo, a practice page or a standard tool already do this? | A new helper next to an existing one |
| Blast radius | What breaks beyond the diff, and which one fact makes it safe? | A wire format or column another service reads |
| Structure | Did the change add a branch, flag, wrapper or cast where a shape (table, enum, state machine) or the existing helper was due, or push a file past about 1,000 lines? | A third boolean that must stay in sync with the other two |
| UI craft | Does the screen show generated tells, or a direction that was decided? | See [ui-craft-audit.md](https://github.com/evocaslab/hq/blob/main/practices/ui-craft-audit.md) |
| Gate integrity | Did the change make a check easier to pass instead of the code correct? | See below |

Name the lenses in the review brief ("apply: attack the premise, idempotency, gate integrity") and ask for findings
with `file:line` and a verdict per lens, "no finding" included. A finding is a claim: the supervisor checks it against
the code before acting ([subagents-and-context.md](subagents-and-context.md#writing-a-subagent-prompt)).

A structure finding is filed as a backlog item or a comment, not applied as an unasked refactor
([change-discipline.md](change-discipline.md#keep-the-change-small)). Report a few high-conviction findings, structural
ones first, and drop nits when a structural problem exists; a rename does not fix a structural problem.

## Evidence levels

A claim of "safe" or "done" names how it was shown, lowest to highest:

1. Said so (no check).
2. Pointed at the line (`file:line`).
3. Walked the failing case through the code and showed it cannot reach the bad state.
4. Ran a script or test against the real code.
5. Reproduced it in the running app.

A hand-back states the level per claim, next to "Checked in: real app / mock app / tests only"
([agent-workflow.md](agent-workflow.md)). The verdict is verified, not verified or inconclusive; inconclusive is reported
as not verified, never as a pass. A reviewer drops any finding it did not run or trace.

## Fan-out review

To review many repos independently (worked in HQ, 2026-11-01): one read-only frontier reviewer per repo or pair,
started together, each with the lenses, a fixed report shape (Blocker / Should fix / Nit with `path:line`, a
Not-checked list, project-neutral practice lessons; under 60 lines, a time cap) and the order to say what it skipped.
The supervisor spot-checks two references per report, files one backlog item per repo, sends *all* practice lessons to
*one* writer (parallel writers on the same pages collide), gives the fixes to branch-only workers, and gates and lands
them itself. Before starting several candidates or reviewers on one question, write the three to six criteria the
result is judged by; if the answers diverge wildly, the brief was under-specified: reframe and rerun instead of
averaging. Several candidates multiply plan spend, so use it only for one-way doors. A reviewer on a frontier model must be started through a tool that honours the model request; a
session-start tool that caps the model at the caller's own runs the "frontier review" on the caller's model
([subagents-and-context.md](subagents-and-context.md#pick-the-cheapest-tier-that-works)).

## Gate integrity

Never make a gate pass by weakening it. Without a human-recorded reason, a change must not:

- skip, delete or `xfail` a test, or add `only`/`fixme`;
- remove assertions or test cases;
- raise a timeout or retry count to hide slowness or flakiness;
- use `--no-verify`, `continue-on-error`, `|| true` in a gate;
- add a lint or type suppression (`noqa`, `type: ignore`, `eslint-disable`, `#[allow]`) without a reason on the line, or
  loosen a lint level ([guards.md](guards.md)).

A child agent's "tests pass" is a claim: the parent re-runs the gate itself, and reports "done, unverified" when it
could not. The machine check is HQ's `just q-integrity` (`scripts/quality/check_gate_integrity.py`): it reads the
branch diff against the merge base and fails on the patterns above unless a commit in the branch carries a
`Gate-change: <reason>` line. It is a heuristic over added and removed lines: a false hit is answered with the reason
line, a miss is caught by the reviewer. Run it on a worker's branch before landing. Related: the
[command guard](agent-security.md#command-guard) blocks `--no-verify` at the shell.

## Backlog item as handoff

Between a planning step and its workers, the hand-over is a file, not a long prompt: a backlog item (or a design doc
linked from it) that says goal, files, constraints, "done when" and the gate. The brief then references it ("do
`B-261031-x`; read it first") and stays short. The file survives a context compaction, a restart and a second worker;
a prompt does not. The item is the contract: the worker updates its Progress section, the supervisor reads that and the
diff, never the worker's chat. See [backlog/README.md](https://github.com/evocaslab/hq/blob/main/docs/backlog/README.md) for the format.

## Sources

- pstack (MIT): <https://github.com/cursor/plugins/tree/main/pstack> (`blast-radius, interrogate, arena, principle-explain-the-number`)
- Cursor plugins, `cursor-team-kit/skills/thermo-nuclear-code-quality-review` (no license found; only the ideas are reused):
  <https://github.com/cursor/plugins/tree/main/cursor-team-kit>
- gstack (MIT): <https://github.com/garrytan/gstack> (README: roles as slash commands; `ETHOS.md`: the principles the
  lenses come from; `careful/SKILL.md`: the guard)
- Claude Code hooks: <https://code.claude.com/docs/en/hooks>
- Claude Code subagents: <https://code.claude.com/docs/en/sub-agents>
