---
summary: "A change to behavior updates the docs that describe it in the same commit; how to find them and check it"
group: docs
tags: [docs, agents, process]
applies-to: [all]
reviewed: 2026-11-01
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
