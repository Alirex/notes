---
summary: "Keep sessions working: what each backlog state (todo, idea, design, blocked, later) lets an agent do, research and design passes that turn ideas into todo items, an idle order of work and an idle policy, and which role and model tier does what"
group: agents
tags: [agents, autopilot, backlog, workflow]
applies-to: [all]
reviewed: 2026-11-12
status: draft
---

# Continuous work: every state is somebody's next step

## Rule

A backlog state is not a reason to skip an item; it says **which kind of pass** the item needs next. `todo` gets built,
`idea` gets a research pass, `design` gets a design pass, `blocked` gets a check for the answer, `later` gets built
when nothing else is left. An idle session works down one fixed order (below) and never ends its turn while anything
in it is pickable; when it truly cannot go on it logs `idle waiting <reason>` (or `idle capped <reason>`), not
`stopped`.

## Why

Measured on 2026-11-12 (`just machine-metrics fleet-flow`, branch `worktree-agent-ab4c8d25ab1dc0358`): sessions were
idle with pickable work in 90 % of the covered minutes; 31 % of the `stopped` lines named the next item instead of
doing it, 32 % were wake caps, and the keeper closed panes with work left. Autopilot step 2 said "skip `idea`,
`design`, `blocked`, `later`", so 35 `idea` and 10 `design` items (and 130+ items that mention research) were nobody's
work. Schedulers solved this long ago: an idle worker **pulls** (work stealing), and a pull system keeps an explicit
"ready" queue and a policy per state (Kanban). A research spike is how agile teams make an unclear item ready.

## States and what they let an agent do

| State | Pass | Who, which tier | Ends in |
| --- | --- | --- | --- |
| `todo` | build | worker or builder-subagent, Sonnet | landed, file deleted |
| `idea` | **research pass**: read-only, timeboxed | researcher: local model, Junie or Haiku; Sonnet when it needs judgment over code | a spec in the item and `status: todo`, or a drop |
| `design` | **design pass**: options and a recommendation | designer: Opus (Fable only on the maintainer's request) | a `maintainer-question` with a default; `todo` (default easy to undo) or `design` + `blocked_by: <repo>#N` |
| `blocked` | check the issue or condition named in `blocked_by` | any session, no model work | unblocked when answered; else listed once in `waiting` |
| `later` | build, only when nothing above is pickable | as `todo` | as `todo` |
| `owner: user` | none (only the maintainer can do it) | nobody | listed once |

### Research pass (`idea` to `todo`)

- Timebox: one pass, about 30 tool calls or 30 minutes; a pass is finished when the box ends, whatever it learned (a
  second pass is a second, named step). Read-only on code and on other repos.
- Output, written into the item by the session that ran the pass (the researcher returns text; it changes no files):
  a one-paragraph premise, a `## Done when` with checkable lines, `size:`, `priority:` and, for `l`, `outcome:`
  ([spec-driven-work.md](spec-driven-work.md)). Then `status: todo` with an Update line `research pass <date>: ...`.
- Or a drop: the item is a duplicate, already done (cite the sha), or not worth it; delete the file with the reason in
  the commit, or set `later` with the reason when it may matter again.
- An idea that needs a decision, not facts, becomes `design` with the question written down.
- Do not research and build in the same pass: the new `todo` goes back into the queue and is picked by priority.

### Design pass (`design`)

- A designer writes 2 to 4 options with costs, a recommendation, and what changes for each. It files one
  `maintainer-question` issue with the recommendation as the default ([maintainer-questions.md](maintainer-questions.md))
  and never stops for the answer.
- Default easy to undo: write the spec on the default (as a research pass) and set `todo`; the answer may revise it.
  Hard to undo (data, deletion, public names, money): keep `design`, add `ready: false` and `blocked_by: <repo>#N`.
- Opus is for design only; on a `cc-usage` "SLOW DOWN" or a fast plan pace the design pass waits, and a Sonnet
  session may still draft the options and the question.

### Blocked and later

- `blocked` on a person: skipped, but named once per round in the `idle waiting` line with its issue; each Orient checks
  the issue, and an answer makes it first in line (answered questions come first). The check is
  `just docs-item-link sweep` (HQ), run by a session in a worktree at its Orient step: it clears `blocked_by` of every
  item whose issue is closed and adds a dated Progress note with the answer; the session commits and lands that change
  with its next item. No timer runs it, because it edits tracked files and nobody would commit the result.
- `later`: picked only when the idle order below has nothing else; the pick says `later item, nothing else pickable`.

## Idle order of work

A session with a free turn takes the first that applies, every time:

1. Answered maintainer questions; its own unlanded branches (land them).
2. `in-progress` items of a session that is gone.
3. `todo` items by priority (autopilot step 2).
4. Chores that are due.
5. **Research passes** on `idea` items, highest priority then oldest; at most 3 per session per round, so research does
   not crowd out building.
6. **Design passes** on `design` items without an open question (plan pace allowing; else draft only).
7. Optimizations (autopilot step 2, item 5) and generators (`just docs-backlog-gen run`, apply when the dry run is clean).
   When 1 to 6 are empty and the last `/improvement-review` is over 24 h old, run it measure-only (scores and a table,
   no priority changes; [continuous-improvement.md](continuous-improvement.md#deciding-what-to-improve)).
8. `later` items.
9. Nothing: log `idle waiting <reason>` and schedule a wake-up, or `stopped done` after the proof of
   [autonomous-sessions.md](autonomous-sessions.md#stop-reasons).

## Idle policy

- Never end a turn while 1 to 8 has a pickable entry. If you can name the next item, you can pick it: "NEXT item
  named" is not a stop reason.
- A cap (wake cap, plan, RAM, landing slot) is `idle capped` or `idle waiting` (the
  [stop reasons](autonomous-sessions.md#stop-reasons)), never `stopped`: keep the pane, schedule the wake-up, resume.
- Log a pass as `picked <id> research pass` or `picked <id> design pass`, so the round and the metrics see it.
- A `stopped` line needs reason `done` and the skipped list (`skipped N: <id>=<why>`); a skip reason of `idea`,
  `design` or `later` alone is not valid any more.
- WIP: one pass at a time per session (build, research or design), at most 3 builders per repo
  ([autonomous-sessions.md](autonomous-sessions.md)); a blocked item still counts as open work in the round table.

## Roles

| Role | May do | Tier | Never |
| --- | --- | --- | --- |
| `autopilot` | the whole idle order for its repo; writes research and design results into items | Sonnet | widen a scope the maintainer set; stop with pickable work |
| `coordinator` | as autopilot, and hands passes to researchers, designers and builders | Sonnet, Opus for its own design | land two branches at once |
| `researcher` | research passes, read-only, returns the spec text | local model, Junie, Haiku | change files; decide a design |
| `designer` | design passes, files the question | Opus; Fable only on request | build; wait for the answer |
| `builder-subagent` | builds one `todo` in a worktree, pushes, hands back | Sonnet | pick other items; land without `land: self` |
| `keeper` | sees idle sessions with only `idea`/`design` work and nudges a research or design pass | none (no LLM) | start work itself |

The roles table in [agent-workflow.md](agent-workflow.md#roles) stays the source for the other roles.

## Sources

- Blumofe, Leiserson, "Scheduling multithreaded computations by work stealing", J. ACM 46(5), 1999 (idle workers pull
  work; near-optimal time): <https://people.cs.umass.edu/~emery/classes/cmpsci691s-fall2004/papers/blumofe95cilk.pdf>
- The Kanban Guide (pull when WIP drops below the limit; explicit policies per state; read 2026-11-12):
  <https://kanbanguides.org/english/>
- Microsoft Engineering Playbook, Definition of Ready (a checklist, not a gate; read 2026-11-12):
  <https://microsoft.github.io/code-with-engineering-playbook/agile-development/team-agreements/definition-of-ready/>
- Mike Cohn, "Spikes" (a timeboxed research item that makes a story ready; read 2026-11-12, practitioner):
  <https://www.mountaingoatsoftware.com/blog/spikes>; origin in XP: <https://en.wikipedia.org/wiki/Spike_(software_development)>
- Anthropic, "How we built our multi-agent research system" (effort scaled to the task, about 3 to 10 tool calls for a
  simple fact-finding pass; a stronger lead with cheaper subagents): <https://www.anthropic.com/engineering/multi-agent-research-system>
- HQ evidence: `scripts/machine/profile/fleet_flow.py` (branch above), [fleet efficiency
  research](../docs/research/2026-11-08-fleet-efficiency.md), [roles research](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-08-roles-for-agents.md),
  backlog counts on 2026-11-12 (496 todo, 35 idea, 10 design, 17 blocked, 5 later).
