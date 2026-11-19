---
summary: "Where each kind of knowledge lives so it survives a memory clean-up and travels between machines: git, not model memory"
group: agents
tags: [agents, claude, context, workflow]
applies-to: [all]
reviewed: 2026-11-13
status: draft
---

# Durable knowledge: save it in git

An agent learns things all day: a script it wrote, a rule that held, why a gate failed, how a machine is set up.
Model memory (`~/.claude/projects/.../memory`) and a session scratchpad hold them only on one machine, can be cleaned,
and are not reviewed. Git is shared, diffed, reviewed and backed up. So what an agent builds or learns is saved in git;
memory may point to it but never holds the only copy.

## Where each kind goes

| Kind | Where | Form |
| --- | --- | --- |
| A script or command it wrote | the repo's `scripts/<module>/` plus a `just` recipe | [just-task-runner.md](just-task-runner.md) |
| A project-neutral lesson | HQ `practices/<page>.md` | research first, a Sources section, `just practices-index` |
| What went wrong and why | HQ `docs/incidents/I-<date>-<slug>.md` | factors with evidence ([root-cause-analysis.md](root-cause-analysis.md)) |
| Research that backs a decision | `docs/research/` | dated, with sources |
| A fact about a machine or cloud project | HQ `infra/` | no secrets |
| A rule for one repo | that repo's `AGENTS.md` (short, linking the page) | [agent-instructions.md](agent-instructions.md) |
| A preference of the maintainer | the dotfiles `~/.claude/CLAUDE.md` | committed in `~/dotfiles` |
| A task or follow-up | a backlog item in the repo it concerns | [docs-backlog-changelog.md](docs-backlog-changelog.md) |
| A fact the agent should recall quickly | a memory file, as a pointer or cache of the above | never the only copy |

A scratchpad file counts only once it is committed and pushed.

## When to save

Save as you go, not at the end: after each landed item (the `Lessons saved:` line), after an incident, before a
compaction or a hand-off, and whenever a manual step is repeated (a repeated step is a script not yet written).

## Why

- Memory can be cleaned, is per machine, and nobody reviews it; a lesson held only there is lost with it.
- Another machine, another agent or a fresh session reads the repo, not this session's memory.
- A page in git gets a review date, a Sources section and a place in the index, so it can be found and aged out.

## Checks

A task in memory is a defect (`scripts/docs/requests.py lint-memory`). Further checks are planned in
`B-261113-durable-knowledge-checks` and the migration of existing memory in `B-261113-memory-to-git-migration`.

## Sources

- Observed in HQ and the repos it manages: lessons that existed only in memory files were lost on a clean-up and
  absent on the second machine (2026-11-13, R-2611130034-7f15); no external source.
- Claude Code memory: `MEMORY.md` loads only its first 200 lines or 25 KB, and memory is local to the machine
  (read 2026-11-09): <https://code.claude.com/docs/en/memory>
