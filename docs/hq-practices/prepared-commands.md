---
summary: "Prepared commands for repeated checks (landed, tag, release, gate result): one action, one result line, no invented one-liners"
group: agents
tags: [agents, just, commands, supervisor, gate]
applies-to: [all]
reviewed: 2026-11-04
status: draft
---

# Prepared commands

An agent that rebuilds the same compound shell line every time makes the same small mistakes every time: a wrapper
exit read as the gate's result, a forgotten `fetch` before `merge-base`, a heredoc the guard asks about, a tag
checked with the wrong ref. A repeated check becomes a recipe once, with a one-line result, and is called by name
afterwards.

## The rule

- Anything an agent runs more than twice in the same shape (a landing check, a release, a gate in the background) is a
  `just` recipe over a script, not a line in the brief or in memory
  ([just-task-runner.md](just-task-runner.md), [just-recipe-vocabulary.md](just-recipe-vocabulary.md)).
- **One action, one result line.** The line says the outcome in words an agent cannot misread (`landed`, `NOT landed`,
  `PASS`, `FAIL <hooks>`, `RUNNING`) and the exit code agrees with it. A long log stays in a file.
- **Verify, don't trust.** A recipe that reports a landing or a release checks the remote itself (`fetch`, then
  `merge-base --is-ancestor` or `ls-remote`), never a session's note
  ([agent-workflow.md](agent-workflow.md#stages)).
- **One command, no chains.** A prepared command passes the guard as one action; the compound lines it replaces are
  what trips permission prompts ([guards.md](guards.md), [agent-instructions.md](agent-instructions.md)).
- The landing cleanup is one of them: after a landing run `just repos-finish`, never a hand-written
  `git worktree remove --force …; git branch -D …; git push origin --delete …` chain. The guard rightly asks on
  `git branch -D` (it cannot know the branch is merged); the recipe proves the landing itself, so nothing needs asking.
- Briefs name the recipe, not the lines inside it. A session that finds itself writing the same pipeline a third time
  adds the recipe (or asks the supervisor to) instead of pasting it again.

## HQ's set

| Recipe | Result |
| --- | --- |
| `just repos-landed <repo> <sha>...` | `landed <sha> on origin/main` or `NOT landed`, exit 1 if any is not |
| `just repos-land <repo> <branch> "<merge message>"` | lands a branch: merges it `--no-ff` into origin's default branch in a throwaway worktree, pushes, fast-forwards the main checkout, then finishes the branch: `landed <branch> as <sha> on origin/main, main checkout pulled` plus the `finished` line, or `NOT landed <branch>: <reason>` (conflict, push refused, already landed); run the repo's gate on the branch first, this does not |
| `just repos-finish <repo> <branch> [--worktree PATH]` | after a landing: proves the branch is on origin's default branch (ancestor or empty `git cherry`), then removes its clean worktree, deletes it locally and on origin: `finished <branch>: worktree removed, local deleted, remote deleted` or `NOT finished <branch>: <reason>` (exit 1); never the default branch, `hq-idle/*`, an unlanded branch or a dirty worktree |
| `just machine-test-containers rm <name>` | your own throwaway test container (labelled `hq.test=1`, e.g. started by a test-database helper): `removing <id> <name>`; any other container is `refused` (exit 1), production ones are never touched; `reap` removes the stale ones |
| `just repos-tag <repo> <tag>` | `tag <tag> on origin` or `NO tag` |
| `just repos-release <repo> <version>` | checks the version on main, runs the repo's `just release <version> push`, verifies the tag |
| `just repos-land-hq` | pushes HQ main, `in sync with origin/main` or ahead/behind counts |
| `just machine-kill-own --port N` / `--pattern TEXT` | stops only processes whose cwd is in this worktree (SIGTERM, then SIGKILL), refuses and lists the rest; never `pkill`/`killall` (the guard asks) |
| `just q-gate-bg [log]` | the gate with a final `EXIT=<code>` line; start it with the harness's background run |
| `just q-gate-result [log]` | `PASS`, `FAIL EXIT=1 [<failed hooks>]` `RUNNING`, or `DEAD` (the gate process is gone without an EXIT line, e.g. an OOM kill) |
| `just q-gate-stop [log]` | `STOPPED gate <pid>` or `NOT RUNNING`: stops only the gate recorded in the log and its children (replaces `pkill -f gate-bg`) |

`q-gate-bg`, `q-gate-result` and `q-gate-stop` are `[no-cd]` recipes: from any repo run `hq q-gate-bg`, `hq q-gate-result` and `hq q-gate-stop` (the
`hq` command runs HQ's recipes from any directory; they gate the repo you are in via its `just q-check`). A repo
needs no copy of the script. Copies already made are removed on the repo's next slice. A machine without HQ (CI, a
clone elsewhere) does not run background gates, so it needs nothing.

## Why

- A background command's completion notice reports the wrapper, not the gate: only the `EXIT=` line says whether it
  passed. The recipe writes that line and reads it back.
- Shorter calls cost fewer tokens on every round, and the plan is the binding limit
  ([autonomous-sessions.md](autonomous-sessions.md)).
- One implementation is fixed once for every session; a pasted line is fixed in none of them.

## Pinned by

- The scripts print the outcome word and set the exit code; check both on a known landed commit, a bogus one, a
  wrong release version and a missing log.
- The preflight digest lists the recipes, so a supervisor reads them at every start.

## Sources

- HQ implementation: `scripts/repos/ops.sh` (tests: `scripts/repos/test_finish.py`), `scripts/quality/gate-bg.sh`, `just/repos.just`, `just/quality.just`.
- Background-task completion semantics: <https://code.claude.com/docs/en/hooks> (hooks run per tool call, so a prepared
  command is one auditable call).
