---
summary: "Working through one change: end state, data shape first, units that end green, when to stop"
group: agents
tags: [agents, process]
applies-to: [all]
reviewed: 2026-10-28
status: stable
---

# Change discipline

How an agent (or a person) works through one change: in what order, when to stop patching and rethink, and how to
leave the code healthier than it found it. The test rules it relies on: [testing.md](testing.md); the checks:
[quality-gate.md](quality-gate.md), [lint-strictness.md](lint-strictness.md).

## Order of work

Make it work, then make it right. Do not polish code whose behavior is still wrong, and do not call a change done
while it only works.

1. **Name the end state.** Before editing, write down what will be true when the change is done, as something a
   command can check (a test passes, a recipe prints X, a page shows Y). Steps in between may break things when that
   is planned and easy to undo; the end state may not.
2. **Shape the data first.** Settle the types, tables and boundaries before the logic that moves data between them
   ([strict-schemas.md](https://github.com/evocaslab/hq/blob/main/practices/strict-schemas.md), [entity-fields.md](https://github.com/evocaslab/hq/blob/main/practices/entity-fields.md)). Logic written against a vague shape
   is rewritten when the shape settles.
3. **Test first**, then the smallest change that passes ([testing.md](testing.md#test-first-for-behavior-changes)).
4. **Cut the work into units that each end green**: each one compiles, passes the gate and could land alone. Order
   them so a later unit builds on a checked earlier one, not on a guess.
5. **Clean up while the tests pin the behavior** (below).

## Keep the change small

- **One decision, one place.** A rule, limit or mapping lives in one function or constant; a second copy is the
  next bug.
- **Remove before you add.** When a fix seems to need new code, first check whether deleting a special case, a
  flag or a layer makes the problem go away.
- **No unasked refactors.** Clear local code stays as it is; restructure only what the change touches, or file a
  backlog item.

## When fixes keep failing

- **Two failed fixes on one premise: stop.** When two attempts that rest on the same assumption fail the same check,
  write that assumption down in one sentence, then question it (read the code that the assumption is about, run the
  smallest experiment that tests it). No third attempt before that.
- **A new requirement may break the design.** When it only fits through flags, special cases or a second path
  through the same code, sketch the design you would write from scratch with all requirements known. If it differs a
  lot, move towards it in green steps instead of adding the next patch.
- **Fix the cause, not the symptom.** Reproduce first; a guard that hides the failure is not a fix
  ([testing.md](testing.md#test-first-for-behavior-changes)).

## Build a tool for repeated work

When the same edit is needed in many places (a rename across repos, a migration of many files, a report over many
rows), do one instance by hand, check it, then write a script that reproduces exactly that result and run it on the
rest. A tool written before a hand-done example encodes a guess. Keep the script in `scripts/` when it will be
needed again ([just-task-runner.md](just-task-runner.md)).

## Design rules that save later rewrites

- **Illegal states are unrepresentable.** Closed sets are enums or CHECKs, required fields are required, variants
  are separate types ([python-style.md](https://github.com/evocaslab/hq/blob/main/practices/python-style.md#data), [rust-service.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service.md)).
- **Parse at the edge, trust inside.** Input is validated once where it enters (request model, CLI argument, file
  parser) and becomes a typed value; inner code does not re-check it. Business rules are pure functions over those
  values ([python-style.md](https://github.com/evocaslab/hq/blob/main/practices/python-style.md#layering)).
- **Reruns are safe.** A job, migration step, installer or recipe that is killed halfway and run again ends in the
  same state as one clean run ([twelve-factor.md](https://github.com/evocaslab/hq/blob/main/practices/twelve-factor.md), [db-migrations.md](https://github.com/evocaslab/hq/blob/main/practices/db-migrations.md)).
- **One writer per piece of state.** When two processes or tasks could write the same table, file or branch, give
  it one owner by structure (a queue, a lock, one service), not by hoping they never overlap
  ([autonomous-sessions.md](autonomous-sessions.md), [rust-web-api.md](https://github.com/evocaslab/hq/blob/main/practices/rust-web-api.md)).

## Make it right

### Refactoring

Pin the behavior with tests first, change only structure, and show the same tests (and any golden output) still pass
unchanged. A refactor commit has no behavior change and says so; a behavior change in it is a separate commit.

### Code health signals

Measured as trends, not gate stages, after the change works:

| Signal | Rule of thumb | How |
| --- | --- | --- |
| Complexity × missing coverage (CRAP) | below 10 per function: split it, or test it | `CRAP = CC² × (1 − cov)³ + CC`; ruff `C901` plus the coverage report, or `crapper` (trial: `B-261028-crapper-trial`) |
| Near-duplicate functions | structural similarity above about 0.8: merge or extract | review; `dryer` (trial: same item) |
| Surviving mutants | each one that changes behavior gets a test; dead code it reveals is deleted | [testing.md](testing.md#mutation-testing) |

A big `match` or dispatch table scores high on complexity but may be the clearest form; judge it, do not split it
only for the number.

## Commits

Each commit says what changed, why, and how it was checked (the test, the command, the page looked at). Small,
ordered commits, one unit each ([git-worktrees-and-autoland.md](git-worktrees-and-autoland.md#commit-messages)).

## Sources

- Robert C. Martin, [ub-mode](https://github.com/unclebob/ub-mode) (an agent skill; no license, so only the ideas are
  used here, in HQ's words) and its tools [crapper](https://github.com/unclebob/crapper),
  [dryer](https://github.com/unclebob/dryer), [mutator](https://github.com/unclebob/mutator) (0BSD).
- Kent Beck, *Test-Driven Development: By Example*; Martin Fowler, *Refactoring* (2nd ed.).
- Alberto Savoia and Bob Evans, the CRAP metric (2007).
