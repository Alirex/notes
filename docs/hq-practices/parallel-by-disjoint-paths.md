---
summary: "Parallel work by disjoint paths: items that touch different files run at once; the limits that stop it and how to lift them"
group: agents
tags: [agents, parallel, worktrees, landing]
applies-to: [all]
reviewed: 2026-11-12
status: draft
---

# Parallel work by disjoint paths

Two pieces of work can run at the same time when they cannot disturb each other. The test that is cheap to check is
the set of paths each one changes. "One writer per repo" is a safe default, but it wastes the machine when most items
touch different folders.

## Research first: why work conflicts

- A merge conflict needs two branches that change the same lines of the same file. Branches that change disjoint files
  merge without a conflict, and a textual merge of disjoint hunks is the case Git handles by design.
- Parallelism is bounded by the shared part of the work (Amdahl's law): if every branch must pass one full-repo gate and
  land one at a time, more writers only lengthen the queue at the gate and the landing lane.
- Limiting work in progress (Kanban WIP limits) keeps the queue short: unlanded branches are inventory, and a pile of
  them blocks new starts and grows merge risk.
- Generated files (indexes, API schemas, README tables) are the usual false conflict: independent items rewrite the
  same generated file, so disjoint work still collides ([generated-files.md](generated-files.md)).

## Rules

1. **Declare what an item touches.** An item names its paths (files, folders or a context) in `touches:`. Derive it from
   the item text or from the first diff of the worker; a wrong guess is corrected on the first overlap check.
2. **A writer lease is per path, not per repo.** Two items run together when their `touches` sets are disjoint. A claim
   (`backlog_next claim:true`) refuses an item whose paths overlap a running one, and says which one.
3. **Plan the set, not the next item.** A round picks the largest set of pickable items with pairwise disjoint paths,
   by priority, and prints why each of the rest waits.
4. **Take the generated files out of the way.** Regenerate on land (merge driver or a pre-land step) so independent
   branches never conflict on them; the path check ignores generated files.
5. **Shared lanes stay narrow on purpose.** The gate seats and the landing lane are machine-wide: disjoint paths let
   many sessions build, but landings still go one at a time or as one batch
   ([agent-landing.md](agent-landing.md)). Scope the gate by changed paths so a seat frees sooner.
6. **Machine limits still hold.** Disjoint paths do not create RAM: the RAM room, the session cap and the hard stops of
   the mode apply ([override-and-its-limits.md](override-and-its-limits.md)). Temperature is measured, not a limit.
7. **Inside one session**, read-heavy work goes to parallel read-only subagents, implementation to workers in their own
   worktrees ([subagents-and-context.md](subagents-and-context.md)).

## Checks

- Two items with disjoint `touches` in one repo are both running and both land; a test covers the overlap check.
- A round prints its independent set and the reason for each waiting item.
- A branch that edits a path outside its declared `touches` is reported as a writer clash, not silently landed.

## Sources

- Git merge behavior (three-way merge, conflicts only where both sides changed the same region):
  <https://git-scm.com/docs/git-merge#_how_conflicts_are_presented>
- Amdahl's law (speed-up is limited by the serial part): <https://en.wikipedia.org/wiki/Amdahl%27s_law>
- Kanban WIP limits: <https://kanbanguides.org/english/>
- Observed in HQ (2026-11-12, backlog `B-261112-parallel-by-disjoint-paths`): one writer per repo, two gate seats,
  generated files that made independent branches conflict, and unlanded branches that blocked every start.
