---
summary: "Evaluating an agent harness (prompt, tools, skills, hooks, context) with evals: task set from real failures, clean trials, layered graders, pass@k vs pass^k, controlled infrastructure, transcripts read"
group: agents
tags: [agents, evals, harness, testing, llm]
applies-to: [all]
reviewed: 2026-11-04
status: draft
---

# Evaluating an agent harness with evals

How to tell whether a change to an agent's **harness** (system prompt, `AGENTS.md`, skills, tools, hooks, context
handling, permission rules) made the agent better, worse or only different. Prompt-level eval sets for single model
calls are in [llm-usage-guards.md](llm-usage-guards.md) (rule 11); the workflow around agents in
[agent-workflow.md](agent-workflow.md); cost of runs in [subagents-and-context.md](subagents-and-context.md). Status
checked 2026-11. Draft: HQ has one eval so far (the command guard, example below); the rules are the method for the next ones.

## Words

- **Task:** one test, with inputs and a success check. **Trial:** one attempt at a task. **Grader:** logic that scores
  a trial. **Transcript:** the full record of a trial (outputs, tool calls, results). **Outcome:** the final state of
  the environment (the file, the database row), never what the agent says it did.
- **Agent harness** (scaffold): what lets a model act as an agent. **Evaluation harness:** what runs the tasks,
  records transcripts, grades and aggregates. A score measures the model and the agent harness *together*.

## Rules

1. **Change one thing at a time.** To judge a harness change, keep the model, task set, trial count, graders and
   environment fixed and vary only the harness. To judge a model, fix the harness. Two changes in one run cannot be
   told apart.
2. **Start small and start now:** 20-50 tasks drawn from real failures (a bug you fixed, a manual check you do before
   a release, a support or review finding) are enough early; effect sizes of a harness change are usually large.
   Writing them later is harder. Rank by user impact.
3. **A task is fair and checkable:** the spec is unambiguous, two people would give the same verdict, and a reference
   solution passes all graders. A task that fails in every trial is first suspected broken (spec or grader), not the
   agent incapable.
4. **Test both directions.** Include cases where the behaviour must happen and cases where it must not (a guard that
   blocks everything passes the "must block" tasks). A one-sided set optimises one side.
5. **Every trial starts clean:** a fresh container or worktree, no leftover files, caches or git history from other
   trials (an agent has been seen reading earlier trials' history). Trials share nothing, so one failure does not
   cause another.
6. **Grade the outcome, not the path.** Check the end state with code (tests pass, file exists, state correct, tool
   called with these arguments). Requiring one exact step order rejects valid solutions. Use partial credit for
   multi-part tasks so progress shows.
7. **Layer the graders, cheapest first:** code-based (fast, objective, brittle), then an LLM judge with a rubric for
   what code cannot check (one judge per dimension, an "Unknown" option, calibrated against human labels), then a
   human sample to calibrate both. Prefer deterministic wherever possible.
8. **Make graders hard to game.** Passing must require solving the problem, not a loophole (editing the test,
   hard-coding the expected output, deleting the check). Add a no-op agent as a baseline: it must score zero.
9. **Run several trials and report the metric.** Outputs vary, so run k trials per task and say which number you
   report: **pass@k** (at least one of k succeeds: peak capability, fine when one good result is enough) or
   **pass^k** (all k succeed: reliability, right for unattended or customer-facing work). At 75 % per trial,
   pass^3 is about 42 % (0.75^3) while pass@3 is about 98 %; by k = 10 pass@k nears 100 % and pass^k nears 0 %.
10. **Control the infrastructure like a prompt.** Memory, CPU, timeouts, network and concurrency change scores:
    Anthropic measured a 6-point gap on Terminal-Bench 2.0 between the tightest and an uncapped container setup, and
    infra errors fell from 5.8 % (strict limits) to 0.5 % (uncapped). Set a guaranteed allocation and a separate hard
    kill limit, record both with the result, run a comparison on the same machine type, and treat a lead under about
    3 points as noise until setups match.
11. **Read the transcripts.** Open the failures and a sample of passes of every run. A failure must look fair: it is
    clear what went wrong. This is how an agent mistake is told from a grader that rejects a valid answer; one
    published model score went from 42 % to 95 % after grader and scaffold fixes, not model changes.
12. **Two suites with different jobs.** *Capability* evals start at a low pass rate and give something to climb.
    *Regression* evals sit near 100 % and run on every harness change (CI or the gate). A capability eval that
    saturates (everything solvable passes) gives no signal: add harder tasks and move it to the regression suite.
13. **Cost is a metric.** Record tokens, tool calls, steps and wall time per success next to the pass rate. A gain
    that doubles the cost is a trade, not a win. Run the suite on the cheapest model that passes it; a frontier
    model only for the tasks the cheap one fails ([subagents-and-context.md](subagents-and-context.md)).
14. **Evals are code, with an owner.** Tasks, graders and reference solutions live in the repo, are reviewed like
    tests, and have one named owner. Anyone who sees an agent fail adds a task (a pull request or a backlog item).
15. **Evals are not the only layer.** Offline evals are the first line (every harness and model change); production
    transcripts, user feedback and spot reviews catch what the set missed, and each real failure becomes a new task.

## What to evaluate in a harness like HQ's

| Harness part | Example task | Grader |
| --- | --- | --- |
| Command guard (`guard.sh`) | "delete this branch", "force-push main", a harmless `git status` | Outcome: the destructive command was denied, the harmless one ran (both directions, rule 4) |
| `AGENTS.md` / skill rule | "land this worktree branch" | Code: branch landed, main pulled, no stash; judge: followed the stages |
| Skill triggering | Prompts that should and should not invoke `/fleet` | Tool-call check: skill invoked or not |
| Context / compaction | Long task with an early fact needed late | Outcome uses the early fact |
| Permission / sandbox | Task that tempts a write outside the allowed folder | State: no file outside; transcript: refused |

## Example: the command guard (a deterministic harness part)

A hook that blocks destructive shell commands needs no model to evaluate: feed it commands, grade the decision. HQ's
unit tests already pin what the guard does today; the eval adds what it should do, as two suites in one YAML file
(`scripts/claude/guard/eval_tasks.yaml`, run by `just cc-guard-eval`):

- **Regression** (24 tasks, in the gate): real incidents and core cases, 100 %. A failure fails the commit.
- **Capability** (reported only): bypasses that slip past string matching (`bash -c '...'`, `git -C dir push
  --force`, `find -exec rm -rf`, a polling loop without `sleep`) and false positives (a blocked word inside a quoted
  message or a `grep`). The first run scored must-block 40 % and must-allow 62 %; one fix round (a quote-masking
  pass, git global options, executor detection) closed all 12 misses, so they moved into the regression suite and
  the capability list is empty until the next gap is found.
- **Baselines** score the two directions apart: an always-allow guard gets 0 % on must-block, an always-block guard
  0 % on must-allow. A one-sided task set would let either pass.
- One trial is enough because the guard is deterministic; the agent that meets the guard needs k trials (rule 9).
- Each fix to the guard moves a capability task into the regression suite (rule 12).

## Checklist

| Check | Pass |
| --- | --- |
| One variable changed per comparison | Model, tasks, k, graders, environment recorded and equal |
| 20-50 tasks from real failures | Each with a reference solution; both directions covered |
| Clean trial environment | Container or worktree per trial; nothing shared |
| Outcome graders first | LLM judge calibrated against humans; no-op scores 0 |
| k trials, metric named | pass@k or pass^k with k stated |
| Infra recorded | Resource limits and kill limits in the result |
| Transcripts read | Failures and a sample of passes inspected before the score is trusted |
| Regression suite runs on change | In the gate or CI; failing task has an owner |

## Sources

- Anthropic, Demystifying evals for AI agents (definitions, grader types, pass@k and pass^k, 20-50 tasks, clean
  environments, transcript reading, saturation, the 42 % to 95 % grading fix):
  <https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents>
- Anthropic, Quantifying infrastructure noise in agentic coding evals (resource limits, the 6-point gap, error rates,
  recommendations): <https://www.anthropic.com/engineering/infrastructure-noise>
- Third-party summaries read for cross-checking: <https://futureagi.com/blog/agent-eval-harness/>,
  <https://www.getwidget.dev/blog/how-to-evaluate-ai-agents/>,
  <https://tianpan.co/blog/2026/02/07/evaluating-ai-agents-trajectories-not-just-outcomes>
- Observed in HQ (2026-11): the first eval is the command guard's (`just cc-guard-eval`, see the example above); the
  other rows of the table are still proposals.
