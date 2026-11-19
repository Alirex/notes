---
summary: "Fewer merge conflicts and cheaper gates with many parallel branches: kill generated-file conflicts, regenerate on merge, land through a queue, tier the gate"
group: agents
tags: [agents, git, quality]
applies-to: [all]
reviewed: 2026-11-09
status: draft
---

# Parallel work without merge pain

Many short-lived worktree branches land through "merge main, run the full gate, land". Two costs grow with the number
of branches: conflicts, and gate re-runs after every main move. Related:
[git-worktrees-and-autoland.md](git-worktrees-and-autoland.md) (isolation, fan-out, tree-hash pass),
[quality-gate.md](quality-gate.md). Observed in HQ on 2026-11-09 (facts below are measured, the rest is sourced).

## What actually conflicts

| Class | Observed | Cause |
| --- | --- | --- |
| Committed generated files | 7 of the 15 most-touched files in the last 150 commits (`practices.json`, map, README table, explainers) | every branch regenerates the same aggregate |
| One hot version or index line | `.claude/flow-version.yaml`, `AGENTS.md` | many branches edit the same lines |
| Lockfiles and locale files | optiomni `ru.ts`/`en.ts` 22 touches each, teleporto `pnpm-lock.yaml` 10 | aggregates edited by every feature |
| Hand-written docs | a few backlog items touched 4 times | two writers on one file |

The gate costs: median 143 s for a passing `q-check`, 384 of 965 runs failed, and a ship that loses the push race
re-merges and re-runs the whole gate (`scripts/repos/ops.sh`). Only the heavy lock serialises ships.

## Rules, best first

1. **Do not commit what you can build.** A derived aggregate that nothing needs at checkout (a README table, an HTML
   map) is generated in CI or on demand. Where it must stay committed, go to 2 and 3.
2. **One file per entry, aggregate generated** (the towncrier pattern; backlog items already follow it). Hot single
   lines (a version) are derived from a tag or written by the landing script, never edited on a branch.
3. **Regenerate on merge, never hand-resolve a generated file.** The landing script merges main; on a conflict in a
   declared generated path it takes either side, runs the generator, commits, then gates. A `.gitattributes` merge
   driver can do the same silently: `path merge=regen` plus `[merge "regen"] driver = ...` in the shared
   `.git/config`. There is no built-in `ours` driver; the driver definition is not versioned, so the setup script must
   install it on every machine (worktrees share the main `.git/config`). `union` is unsafe for JSON, HTML and YAML.
4. **Land through a queue that tests the merged result.** Merge-main-then-test proves the branch, not the combination
   (semantic conflicts). A serial lander that merges, gates and pushes one branch at a time is the minimum; GitHub's
   merge queue, Zuul and Mergify test each entry on top of those ahead in parallel and batch entries to share one run.
5. **Tier the gate.** On the branch run the fast tier on changed files (`prek run --from-ref origin/main --to-ref
   HEAD`, fast-fail order); the full gate runs once in the queue. Skip the run when the tree already passed
   (a pass per tree hash).
6. **Declare touched paths at claim time** and refuse overlap between concurrent claims; shared files (config,
   manifests, `AGENTS.md`) get one integrator or a first, separate step.
7. **Small batches.** DORA: few active branches, hours not days, builds of a few minutes. A gate that takes minutes and
   re-runs per main move defeats the small-batch benefit.
8. Affected-only test selection (Nx-style) needs accurate declared inputs and a hermetic build: a later step, not
   a first one.

`git rerere` helps only when the same branch re-merges repeatedly, which short agent branches rarely do.

## Pinning it

A hook that fails when a generated file is stale (HQ's `diagrams-current`, `openapi-stale`) stays; the fix belongs in
the regenerate step, not in a looser hook. Measure after a change: conflicts per week (`git log --merges
--format=%B | grep -c Conflicts:`), ship re-gates per landing, median `q-check` seconds.

## Sources

Read 2026-11-09 (summaries by a fetch model; verify exact setting names before relying on them):

- Git, gitattributes (merge drivers, `union`, `binary`): <https://git-scm.com/docs/gitattributes> (primary)
- Git, rerere: <https://git-scm.com/docs/git-rerere> (primary)
- GitHub, managing a merge queue: <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue> (primary)
- Zuul, gating and dependent pipelines (speculative testing, adaptive window): <https://zuul-ci.org/docs/zuul/latest/gating.html> (primary)
- Mergify merge queue (parallel checks, batches, two-step CI): <https://docs.mergify.com/merge-queue/> (primary)
- Bors, the "not rocket science" rule: <https://bors.tech/essay/2017/02/02/pitch/> (primary, author)
- DORA, trunk-based development: <https://dora.dev/capabilities/trunk-based-development/> (primary)
- Nx affected: <https://nx.dev/docs/features/ci-features/affected>; Bazel remote cache: <https://bazel.build/remote/caching> (primary)
- pre-commit run (`--from-ref/--to-ref`, `fail_fast`): <https://pre-commit.com/#pre-commit-run> (primary)
- towncrier, one fragment per change: <https://towncrier.readthedocs.io/en/stable/index.html> (primary)
- Parallel-agent worktree guides (file-disjoint tasks, serialised merges): <https://augmentcode.com/guides/git-worktrees-parallel-ai-agent-execution> (vendor blog, search summary only)
- Observed in HQ: `metrics.db` runs, `git log` over origin/main, `scripts/repos/ops.sh` (this repo, 2026-11-09).
- Not verified: lockfile merge drivers, `driver = true` as a keep-ours driver (untested), measured conflict reduction
  from any agent-specific technique.
