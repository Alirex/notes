---
summary: "Name the kind of thing you run - an activity (debug, test, experiment, dry-run), a place (lab, sandbox, staging) or a rollout step (pilot, canary) - and never reuse one word for two; how items, commits and hand-backs phrase them"
group: agents
tags: [vocabulary, testing, deploy, experiments, glossary]
applies-to: [all]
reviewed: 2026-11-08
status: draft
---

# Environments and trials

## Rule

Before you write "test", "lab" or "pilot" in an item, a commit or a hand-back, decide what kind of thing it is:

- an **activity** (something you do): debug, test, experiment, dry-run;
- a **place** (where it runs): lab, sandbox, staging, production;
- a **rollout step** (who gets the real thing first): pilot, canary.

Use the word of that kind and never one word for two kinds. "Tested in staging" is an activity in a place; "a test
environment" says nothing. A **mode** (burn mode) is none of these: a standing setting for the whole fleet. The
definitions are in [the glossary](https://github.com/evocaslab/hq/blob/main/practices/glossary.md#running-things-and-what-hq-is-made-of); this page is the reasoning.

## The table

| Word | Kind | Repeatable? | Exposure | One line |
| --- | --- | --- | --- | --- |
| debug | activity | no, one task | fake or real | find why one thing misbehaves; leave nothing behind |
| test | activity | yes, every gate | fake | check against a stated expectation, pass or fail |
| experiment | activity | no, time-boxed | fake or real | try an idea with a hypothesis and a measured outcome |
| dry-run | activity | yes | real inputs, no effect | the real path, nothing changed, print what would change |
| lab | place | yes | fake | where trials run on purpose, no uptime promise |
| sandbox | place | yes, resettable | fake data, dead side effects | safe to break and reset |
| staging | place | yes | production-shaped data | a release is checked here before it ships |
| pilot | rollout step | no | real, few users | one repo, session or user first; decide to widen or stop |
| canary | rollout step | per release | real, small share | new beside old, compared on a signal, widened or rolled back |
| mode | standing setting | n/a | real | changes how all work runs until switched off |

Three questions sort a new word: is it something you do, a place, or a step of a rollout (kind); can you run it again
unchanged (repeatable); can it hurt a real user or real data (exposure). If two words answer the same three, one of
them is redundant.

## Mode, experiment, debug

These three get confused most:

| | Scope | Lifetime | Ends with |
| --- | --- | --- | --- |
| **mode** | the whole fleet | until switched off | a command (`cc-burn off`) |
| **experiment** | one question | time-boxed | a measured outcome: keep or drop |
| **debug** | one task | minutes to hours | the cause found, the temporary state removed |

An experiment that never ends is a mode; a debug state that stays is a bug.

## Dry-run is `plan`

HQ's tools say `plan` for a dry-run and `--apply` for the real run (`machine-cleanup plan`). Keep that in tools that
have it; a flag already named `--dry-run` in an external tool stays. In prose say "dry-run" for the activity and name
the recipe.

## How to phrase it

- **Backlog items**: put the kind in the verb. "Pilot X in repo Y, widen if the gate stays green" is a rollout step with
  a decision; "experiment: does Z cut gate time, measure with `q-check`" has a hypothesis and an outcome. Do not file
  "test X" for a trial of an idea; that is an experiment.
- **Commits**: say what ran and where: "checked in staging", "dry-run showed 3 deletions". A trailer or subject that
  says only "tested" tells the reader nothing about the place.
- **Hand-backs**: the Definition of Done line says `real` (the actual tool or service ran) or `tests only`. Add the
  place when it was not production: "real, in the lab", "tests only, in the sandbox".
- **Cycle time** is request to landing; **lead time** is request to production. Do not use one for the other.

## Framework, library, platform

The same rule of one word per thing applies to what HQ is made of. A **library** is what you call (`practices/`); a
**framework** calls you and names its required parts (the agent flow, one bundle with one semver); a **platform** is the
self-service set around both (recipes, guard, MCP, digest, timers). HQ as a whole is a platform; do not call it a
framework. See the research note below for the evidence and the not-taken options.

## Checks

The glossary check (`just practices-check`) fails on a stale generated glossary, a duplicate id or a broken `see`. The
`avoid` rows of these terms appear in the renamed-terms table; a lint over the avoided words in prose is still open
(see [practices-system.md](https://github.com/evocaslab/hq/blob/main/docs/design/practices-system.md#glossary)).

## Sources

Read 2026-11-08; no external source was opened for this page.

- [docs/research/2026-11-08-versioning-and-framework.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-08-versioning-and-framework.md):
  library, framework and platform definitions with their sources (Fowler, Johnson and Foote, Scrum Guide, CNCF).
- [agile-lessons.md](agile-lessons.md) (cycle time), [continuous-improvement.md](continuous-improvement.md) (lead time),
  [deploy-and-operations.md](https://github.com/evocaslab/hq/blob/main/practices/deploy-and-operations.md) and [deploy-parity.md](https://github.com/evocaslab/hq/blob/main/practices/deploy-parity.md) (canary, staging),
  [access-control-roles.md](https://github.com/evocaslab/hq/blob/main/practices/access-control-roles.md) (sandbox), [autonomous-sessions.md](autonomous-sessions.md) (burn mode).
- The maintainer's question of 2026-11-08 (request R-2611081617-10fe).
