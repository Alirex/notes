---
summary: "The ordered verification layers from type check to production monitoring, where each runs in HQ, and the typical agent mistake each one catches"
group: quality
tags: [gate, testing, agents]
applies-to: [all]
reviewed: 2026-11-09
status: draft
check: "none: a map of other pages' checks, each of which has its own check"
---

# Verification layers

An agent's "done" is a claim. Each layer below catches a different kind of wrong claim, and a cheaper layer earlier
means fewer expensive failures later. This page is the map: what runs where, and what each layer is for. The reasons
and the rules live in the linked pages.

## The layers, cheapest first

| Layer | Where it runs in HQ | The agent mistake it catches |
| --- | --- | --- |
| 1. Type check | the editor loop and the fast loop; `basedpyright` and `cargo clippy` in `just q-check` ([quality-gate.md](quality-gate.md#strict-typing)) | An invented API: a method, field or argument that does not exist; a wrong signature; an optional value used as present |
| 2. Lint and format | prek hooks on commit (`ruff`, `shellcheck`, `markdownlint`, `format-justfile`, `gitleaks`); fixers run first in `just q-check` | A silenced lint: `noqa`, `type: ignore`, `#[allow]` instead of a fix. The suppression needs a reason on the line, and `just q-integrity` reads the branch diff for it ([review-lenses.md](review-lenses.md#gate-integrity)) |
| 3. Unit tests | `just q-check` (the repo's test stage); the fast loop runs only the affected ones ([tdd.md](tdd.md)) | A deleted or weakened test (a skip, a removed assertion, a longer timeout) to get green: `q-integrity` fails it unless a commit carries a `Gate-change:` reason; a test that never failed first is not evidence; mutation on changed lines finds tests that pass whatever the code does |
| 4. Integration | the same gate, tests that use the real database, files or process | A mock that agrees with the code and not with the dependency; a migration or config that works only on an empty setup |
| 5. End to end | the repo's e2e stage, in the gate when the repo has one; looked at on the real screen for UI work ([subagents-and-context.md](subagents-and-context.md#landing-a-builders-branch)) | A flow that works in each part and breaks across them; a UI that renders in the test and not at phone width |
| 6. The gate | `just q-check`, run detached with `hq q-gate-bg` / `hq q-gate-result` ([quality-gate.md](quality-gate.md)) | Stale docs and stale generated files (`practices-current`, `diagrams-current`, `backlog` hooks fail when the source moved and the copy did not); a step that was skipped but reported green ("every step ran or the gate is not green") |
| 7. Landing proof | `just repos-landed <repo> <sha>`, `repos-dod`, `repos-ship` (merge the default branch in, run the gate, land, finish, pull), `repos-finish` ([prepared-commands.md](prepared-commands.md)) | A "landed" claim without proof: a commit that is not an ancestor of the default branch, a dirty or behind main checkout, a worker that reports a branch that was never pushed |
| 8. CI | in the managed repos, the same `just q-check`, then `git diff --exit-code` ([quality-gate.md](quality-gate.md#shape)); HQ itself has no CI workflows, its gate runs on the machine | A green local run that skipped the fixer hooks, or that depends on a local file, tool version or cache |
| 9. Post-deploy smoke | a route smoke from the public URL after `up` (`just <repo>-smoke-routes`, [deploy-parity.md](https://github.com/evocaslab/hq/blob/main/practices/deploy-parity.md)), and the smoke stack in the repo's deploy ([deploy-and-operations.md](https://github.com/evocaslab/hq/blob/main/practices/deploy-and-operations.md)) | A route gone but health green: tests ran against the new build while prod still ran the old unit, so a removed route answers 404 or 405 while `/healthz` is fine |
| 10. Production monitoring | readiness and heartbeat on the status route, logs and alerts ([observability-and-incidents.md](https://github.com/evocaslab/hq/blob/main/practices/observability-and-incidents.md)); on this machine the weekly `hq-outdated.timer` report and `cc-watch` for sessions | Drift after a green deploy: a bot whose connection dropped while the process stays up, a job that stopped, a dependency that became vulnerable |

## How to use the map

- **Put the check at the cheapest layer that can catch it.** An invented API is a type error, not an e2e failure; a
  removed route is a one-time static route-table diff plus a smoke, not a feature test.
- **A mistake that got through twice moves down the table.** If prose said "do not silence lints" and an agent did,
  the check moves into a hook or `q-integrity`, as [guards.md](guards.md) and
  [agent-instructions.md](agent-instructions.md#which-mechanism-for-which-job) say.
- **Report the layer.** A hand-back says which layers ran ("Checked in: tests only" means layers 1-3), and the
  supervisor re-runs layers 6-7 itself; a worker's "gate passed" is a claim ([review-lenses.md](review-lenses.md)).
- **Layers 9-10 are not optional for a deploy.** Green layers 1-8 test the build, not what runs. A change that removes
  or moves a route, a job or a unit needs layer 9 in the same change.
- **A layer that has never failed is suspect** as much as one that always fails: look at when it last caught
  something before trusting its silence.

## Sources

- Observed in HQ: the hooks and recipes named in the table are those in `.pre-commit-config.yaml`, `just/*.just` and
  `AGENTS.md`; the failures are incidents or reviews recorded in the linked pages (the deploy-parity route incident,
  the gate-integrity heuristic, "landed" claims checked by `repos-landed`). No external source for the table itself.
- Hooks run at fixed lifecycle points and a blocking hook stops the action (read 2026-11-09):
  <https://code.claude.com/docs/en/hooks>
- Permission rules and the sandbox enforce what prose cannot (read 2026-11-09):
  <https://code.claude.com/docs/en/permissions>
- Linked pages: [tdd.md](tdd.md), [quality-gate.md](quality-gate.md), [review-lenses.md](review-lenses.md),
  [deploy-parity.md](https://github.com/evocaslab/hq/blob/main/practices/deploy-parity.md), [observability-and-incidents.md](https://github.com/evocaslab/hq/blob/main/practices/observability-and-incidents.md).
