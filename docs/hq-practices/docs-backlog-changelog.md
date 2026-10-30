---
summary: "Docs, backlog, changelog and decisions laid out so parallel agents don't collide"
group: docs
tags: [docs, backlog]
applies-to: [all]
reviewed: 2026-10-28
status: stable
---

# Docs, backlog, changelog, decisions

A layout where several agents add and close work in parallel without editing
the same lines, and statuses can't silently go stale. `ARCHITECTURE.md` and the diagrams:
[architecture-diagrams.md](architecture-diagrams.md).

## Problem it solves

A single backlog table plus a single progress log grow to hundreds of rows;
every agent edits both, causing merge conflicts, numbering collisions and
"done"/"in progress" rows nobody updates.

## Layout

```text
docs/
  README.md                 the one index of all docs
  backlog/
    README.md               statuses, ID format, how to add/close
    B-YYMMDD-<slug>.md      one file per open item (frontmatter + request text)
  changelog/
    YYYY-MM-DD-<branch>.md  one file per landed branch
  decisions/
    README.md               index table
    D-YYMMDD-<slug>.md      "we chose X over Y because Z", a few lines each
  CHORES.md                 recurring chores: Chore / When / How table
  <topic>/…                 durable topic docs (architecture, API, ops)
```

Example: `auto-joint-lab/report-bot/docs/`. HQ uses the same system for its own
work (`docs/backlog/README.md`, `docs/CHORES.md`, `scripts/docs/backlog.py`),
without a changelog: git history is the record there.

## Backlog

- **IDs are date-slugs** (`B-261023-webauthn`), not sequential numbers, so two
  agents never pick the same ID.
- **One file per open item.** Frontmatter: `id` (= file name), `status`,
  `requested` (date), `area` (from a fixed list in the backlog README), optional
  `legacy` (row number in an old table), `owner: user` (only the maintainer can do
  it: logins, purchases) and, for findings about another repo, `area: review` with
  `repo: <owner>/<repo>`. The body is the request in the
  maintainer's words, then a `## Progress` section (branches, done, left).
- **Statuses:** `idea` (worth doing, not planned), `todo` (ready to pick),
  `design` (needs a design doc or answers), `in-progress` (a branch is on it),
  `review` (waiting for the maintainer), `blocked` (waiting on something
  outside the repo), `later` (postponed). There is no `planned` and no `done`.
- **Done means delete the file** in the branch that finishes the item, and add
  a changelog entry. A partly done item keeps its file; the rest goes under
  Progress.
- **A test or lint guards the backlog**: unknown status, missing fields, an
  id that differs from the file name, or `status: done` fails.
- **Index by command, not by hand**: a recipe prints the open backlog (grouped
  by status, optionally one `--status`). Hand-kept indexes go stale.
- **Maintainer requests are written down as they arrive** (a file per request),
  so they survive context compaction and new sessions.
- **Questions and judgment calls** for the maintainer are GitHub issues with
  the label `maintainer-question`, not item files or a shared questions file:
  [maintainer-questions.md](maintainer-questions.md). Unattended sessions ask
  there instead of stopping.

## Changelog

- **One file per landed branch**, `YYYY-MM-DD-<branch-slug>.md`, written in the
  branch itself before landing, so parallel branches never edit the same file.
- Content: `# Title` with the branch name, then short sections (Domain, API,
  Screens, Tests, Skipped) and a link to the backlog item it closed.
- Fold the month's files into one `YYYY-MM-*.md` archive once a month (a
  chore). Big historic logs get frozen into an archive file.
- **Durable knowledge goes to topic docs**, not into the changelog: if a future
  agent needs it to work, it belongs in the topic doc; the changelog just links.

## Decisions

- One file per decision, `D-YYMMDD-<slug>.md` (date recorded), a few lines.
- Title, then a `Status: accepted · recorded <date>` line; then **Context**, **Decision**, **Consequences**.
- A decision that changes gets a new file; the old one says
  `Status: superseded by D-…`. Never rewrite history.
- `decisions/README.md` has an index table (link, title). Rules agents must
  follow stay in AGENTS.md; the decision explains why. Link records from topic
  docs instead of re-arguing them.

## Chores

- `docs/CHORES.md` is a table: **Chore | When | How** (e.g. "Dependency
  upgrades | weekly | upgrade skill"). Chores are not backlog: never "done".
- When an agent runs one, it notes the date and outcome in the branch's
  changelog entry. Idle agents take a due chore.
- Tracking the last run (built in `report-bot`, `scripts/docs/chores.py`): the
  table gets an `Id` column, the commit that does a chore carries a
  `Chore: <id>` trailer (one per chore), and `just docs-chores` prints each
  chore with the date of its last trailer and whether it is due (`weekly`
  after 7 days, `monthly` after 30; event-driven chores are never due by date).
  HQ instead appends a line per run to a per-machine `~/.local/state/hq/chores.jsonl` (`just docs-chore-done`), so nothing lands in git.

## Docs index

One `docs/README.md` indexes everything (work tracking, products, topic docs);
no second index file. Agents start at the root AGENTS.md, which links it.

## Language

Docs in English; glossaries and places with domain terms in another language
keep both forms.
