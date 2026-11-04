---
summary: "What to take from Agile, Kanban and Shape Up for a flow of one maintainer and agents: Definition of Done, WIP limits, cycle time, appetite, weekly metrics"
group: agents
tags: [agents, workflow, process, kanban]
applies-to: [all]
reviewed: 2026-11-01
status: draft
---

# Agile ideas for a maintainer-plus-agents flow

The flow in [agent-workflow.md](agent-workflow.md) already has a backlog, a review, a retro and a daily round
([autonomous-sessions.md](autonomous-sessions.md)). This page lists the ideas from Scrum, Kanban and Shape Up that add
something, and the ones that do not fit.

## Take

| Idea | Rule here |
| --- | --- |
| **Definition of Done**, one list per kind of work | Code: gate green on the merged result, landed, default branch pulled, item file deleted, changelog and docs updated, lessons placed. Practice or doc: index regenerated, links checked, `reviewed` date set. A worker's hand-back block shows each line; a missing line is "not done" |
| **WIP limit** | At most one writer per repo, at most the machine's verdict in sessions overall, and an idle session is retired, not parked. A new start waits while a finished branch awaits landing |
| **Cycle time** | Measure request date to landing date per item (the backlog file has `requested`; the landing commit has the date). Watch the median and the oldest open item, not an average |
| **Appetite** (Shape Up) | Beside `size`, an item may say how much it is worth: a time and plan budget. A worker that reaches it stops, commits a pushed WIP, says "gate not run" if so, and reports what is left |
| **Cost of delay** | `due:` and `depends_on:` feed the tie-break of the backlog order ([backlog-priority design](https://github.com/evocaslab/hq/blob/main/docs/design/backlog-priority.md)); small first stays as the WSJF-style tie-break |
| **Weekly review** | Once a week, in the Retro chore: items landed, median cycle time, items bounced by review, plan spent per landed item, and one change for the worst number |

Why WIP limits: by Little's Law, average cycle time is average work in progress divided by throughput. Throughput is
hard to decree, WIP is not, so lowering WIP lowers cycle time. A limit set too high creates no pressure to finish.

## Leave

- Story points: `s | m | l | xl` is enough, and agents estimate badly in hours.
- Fixed sprints and daily meetings: the work is continuous; the round every 30 minutes is the stand-up.
- Velocity targets: they reward splitting items, not finishing them.

See also [continuous-improvement.md](continuous-improvement.md): the flow numbers (lead time, deploy frequency, change fail rate) as a trend, never a target.

## Sources

- Cycle time and WIP, Little's Law: <https://www.scrum.org/node/75346>
- Limits that are too high do not create tension to finish: <https://prokanban.org/blog/dont-just-limit-wip-optimize-it>
- Taskwarrior urgency coefficients (due, blocking, age): <https://taskwarrior.org/docs/urgency/>
- WSJF, cost of delay divided by job size: <https://framework.scaledagile.com/wsjf/>
- Not fetched, cited from memory, check before relying on them: Scrum Guide <https://scrumguides.org/scrum-guide.html>
  (Definition of Done, events); Shape Up, appetite <https://basecamp.com/shapeup>.
