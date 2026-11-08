---
summary: "A change to behavior updates the docs that describe it in the same commit; how to find them and check it"
group: docs
tags: [docs, agents, process]
applies-to: [all]
reviewed: 2026-11-08
status: stable
---

# Docs match the code

Docs that disagree with the code are worse than none: the next agent or person trusts them. So a change to logic is
not done until the docs that describe it say the new thing, in the same branch.

## Rule

- **Same commit or same branch.** A change to behavior (rules, formats, endpoints, CLI arguments, env vars,
  configuration, schemas, permissions, error codes) updates the docs that describe it before it lands. A follow-up
  "docs later" item is a broken change, not a plan.
- **Find the docs by search, not memory.** Before landing, search the docs for the names the change touches (function,
  endpoint, setting, error code, column) and fix every hit: topic docs, design docs, API reference, README,
  architecture views, decision records, the changelog entry.
- **Decisions stay true.** A decision record that the change overturns gets a new record that supersedes it (or a
  status change), never silent edits to history ([docs-backlog-changelog.md](docs-backlog-changelog.md)).
- **Generated views are regenerated**, not hand-edited, and a hook fails when they are stale
  ([architecture-diagrams.md](architecture-diagrams.md)).
- **State the check in the commit body:** which docs were searched and changed, or "no doc describes this".

## The README is the front door

The README is the one doc every newcomer opens, so it is held to a stricter rule: **always current, in the same
commit** as the change that touches it (a command, flag, config key, recipe, output, UI feature or check).

- **Start with "how do I run it":** a Quick start table (goal, command, what it needs), then a section per way to run
  (CLI, GUI, container), each with copy-paste commands from the repo root and what they write.
- **One place per fact.** Point at the config file and `--help` for the full list instead of copying every option;
  keep deep material in `docs/` and link it. A README that is also the manual drifts.
- **Everything that runs is listed:** the recipes (`just` names), the checks and what they prove, the layout table.
  A new recipe, crate or folder adds its line in the same commit.
- **State what is not obvious:** what two implementations share and where they only agree statistically, which
  parameters exist but are unused, which files are inputs and which are outputs.
- **Try it before landing:** run the Quick start commands once from a clean shell (the smoke profile is enough). A
  command that fails in the README is a failing test.
- **Put the rule where agents read it:** the repo's AGENTS.md says "the README is updated in the same commit" and
  names the Quick start as the front door.
- **Merge, don't fork, a README rewrite:** when a branch rewrites the README, merge it into the next branch that
  touches the config or commands before editing, or the two versions conflict and one goes stale.

## Make it checkable

- Pin the parts that can drift to the code: examples that run in tests, API docs generated from the schema, a test
  that compares a documented list (env vars, error codes, routes) with the real one.
- A review lens "docs": the reviewer names the docs the diff should have touched and checks them
  ([review-lenses.md](review-lenses.md)).
- When a doc is found wrong while working on something else, fix it in place if small, else file a backlog item at
  once.

## Sources

- Write the Docs, "Docs as Code": https://www.writethedocs.org/guide/docs-as-code/
- Software Engineering at Google, ch. 10 "Documentation": https://abseil.io/resources/swe-book/html/ch10.html
