---
summary: "A backlog item is the spec: how much spec an item needs by size (Done when, Not in scope, Open), who may edit it, and how it is checked at landing"
group: agents
tags: [agents, workflow, backlog, spec]
applies-to: [all]
reviewed: 2026-11-07
status: draft
---

# Spec-driven work: the item is the spec

Spec-driven tools (Spec Kit, Kiro, OpenSpec, BMAD, Tessl) write requirements, design and tasks before code. Our backlog
item already is that file: the request in the maintainer's words, a Plan, a Progress section, an outcome. The research
([note](https://github.com/evocaslab/hq/blob/main/docs/research/spec-driven-development.md)) says the gain is a reviewable contract, the cost is review burden
and decay, and the depth must follow the size. So we add the missing parts and gate them by size, no new pipeline.

## The rule, by size

| Size | The item needs | Read before the brief goes out |
| --- | --- | --- |
| `s` | the request, one paragraph | nothing |
| `m` | + `## Done when`: checkable lines | the supervisor reads it |
| `l` | + `## Plan` (premise, files, steps), `## Not in scope`, `## Open` (questions, or "none") | the supervisor, or a Haiku spec check: every Done-when line testable, no open question, the whole request covered |
| `xl` | split first | n/a |

A Done-when line says how it is shown: a command and its result, a screen, a test name. Behaviour reads "when X, Y".
Docs and infra items use plain checkable lines, not a notation. Keep an item short: about 10 Done-when lines at most; more
means split it.

## Who owns what

- The worker edits Progress. Done when and Not in scope are the supervisor's: a worker that finds one wrong says so in
  Progress; the supervisor edits it with a one-line reason. A worker never weakens a line to pass.
- At review the lens "diff against Done when" applies: each line is met with evidence or "not met (why)" in the hand-back.
- A partial landing rewrites Done when to what is left. A full landing deletes the item; durable facts go to
  practices or the repo's docs, never only the item.

## Verify against the spec

Check the diff against the item with a reader that did not write it: a fresh, read-only subagent (or the local model)
that gets the item (Done when, Not in scope) and the diff, not the writer's reasoning. The prompt: "Report gaps that
affect correctness against this spec; skip style." Without "skip style" a reviewer always finds something, and fixing
every finding adds layers nobody asked for. The output is a list of gaps with `file:line`, each marked met, not met
or unclear; the supervisor decides. This is the "diff against Done when" lens of [review-lenses.md](review-lenses.md)
with the spec as the only input; how specs are written is in [specifications.md](specifications.md).

## Known failure modes of heavy spec-driven tools

One author's hands-on trial of Kiro, Spec Kit and Tessl (opinion, level OP, one person, 2025):

- A small bug became 4 stories and 16 acceptance criteria (Kiro); Spec Kit's output was verbose and overkill for a
  mid-sized feature.
- Agents ignored the spec or over-applied it, and the same spec gave different code on different runs.
- Spec-as-source (the code is generated and never edited) is open, not shown to work (the Radar and Scott Logic's
  Cronk both flag it).

A second trial, a Spec Kit run on a feature the author already knew (read 2026-11-07, one person, level OP): about
2,600 lines of markdown for about 690 lines of code, 3.5 hours of review against about 15 minutes for plain iterative
prompting, a build that did not start, and no path for a bug found at review ("not a spec matter"). Thoughtworks' Radar
(Assess, 2025-11) adds that tool behaviour varies with task size and that long spec files are hard to review. So:

- Write no plan for work the agent already understands (`s` and `m` rows).
- A bug found at review is fixed in code; the item changes only when its Done when was wrong.
- Never back-fill a spec from finished code: it records what exists, not what was meant.
- Spec text several times the size of the code it describes is a sign to cut it.

The tools' own speed claims (hours against minutes) are vendor illustrations and are not evidence. Hence the item is the
spec, at the size it needs (table above): the cost of a spec is review burden and decay, so a small item stays small.
The heavy pipeline (constitution, specify, plan, tasks, analyze) is studied and not taken.

## Why this and not more

- Specs help on new, larger work and hurt on small work; every source warns about review overload and spec decay.
- Checks happen at completion, so ours is the DoD line, not continuous sync tooling.
- Durable context (rules, conventions) stays in AGENTS.md and practices, apart from the task spec.

## Sources

- [Research note](https://github.com/evocaslab/hq/blob/main/docs/research/spec-driven-development.md): the table with each source, date read and evidence level.
- Böckeler, "Understanding Spec-Driven-Development: Kiro, spec-kit, and Tessl", martinfowler.com, 2025-10-15.
- Anthropic, "Effective harnesses for long-running agents", 2025-11-26 (the worker flips `passes` only).
- Claude Code best practices, code.claude.com/docs/en/best-practices (review against a spec, evidence not "tests pass").
- Böckeler's trial of the three tools, level OP (read 2026-11-07): <https://martinfowler.com/articles/exploring-gen-ai/sdd-3-tools.html>
- Spec Kit's phases, Kiro's `requirements.md`, `design.md`, `tasks.md` (read 2026-11-07):
  <https://github.com/github/spec-kit/blob/main/spec-driven.md>, <https://kiro.dev/docs/specs/>
- Fresh-subagent review of the diff against `SPEC.md`, "report gaps affecting correctness, skip style" (read
  2026-11-07): <https://code.claude.com/docs/en/best-practices>
- Thoughtworks Technology Radar, "Spec-driven development", Assess, 2025-11-05 (read 2026-11-07, level OP, vendor view):
  <https://www.thoughtworks.com/radar/techniques/spec-driven-development>
- Eberhardt, "Putting Spec Kit through its paces", Scott Logic, 2025-11-26 (read 2026-11-07, one-person trial; his
  timings do not fully add up, so the 10x is approximate):
  <https://blog.scottlogic.com/2025/11/26/putting-spec-kit-through-its-paces-radical-idea-or-reinvented-waterfall.html>
- Cronk, "The specification renaissance", Scott Logic, 2025-12-15 (opinion, no data; the "no back-filling" rule):
  <https://blog.scottlogic.com/2025/12/15/the-specification-renaissance-skills-and-mindset-for-spec-driven-development.html>
- Netflix: no engineering-blog post on coding-agent workflows found. A second-hand recap of an ex-Netflix talk (research,
  plan, implement with a human check on the research; hearsay, not used for a rule):
  <https://velagao.substack.com/p/no-more-vibe-coding-practice-from>
