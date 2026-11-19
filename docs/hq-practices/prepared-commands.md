---
summary: "Prepared commands for repeated checks (landed, tag, release, gate result): one action, one result line, no invented one-liners"
group: agents
tags: [agents, just, supervision, gate]
applies-to: [all]
reviewed: 2026-11-07
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
- **Wait on the result line, not on a notice.** The "completed" notice of a background launcher says the launcher
  ended, not that the gate passed. Wait with `repos-ship-wait` / `q-gate-result`, or with a monitor that matches only the
  exact result line (`tail -n +1 -F LOG | grep --line-buffered '^exit '`: `+1` so a line written before the monitor
  started is still seen; never a broad `error|FAIL` pattern, it matches crate and test names). A landing needs the
  line itself (`exit 0`, `PASS`, `shipped`) in the log.
- Briefs name the recipe, not the lines inside it. A session that finds itself writing the same pipeline a third time
  adds the recipe (or asks the supervisor to) instead of pasting it again.

## HQ's set

| Recipe | Result |
| --- | --- |
| `just repos-landed <repo> <sha>...` | `landed <sha> on origin/main` or `NOT landed`, exit 1 if any is not |
| `just repos-dod <repo> <ref> [--item B-ID] [--request R-ID] [--handback FILE] [--unlanded]` | one line per Definition of Done item (`ok`/`FAIL`: landed, main pulled, item deleted, request closed, lessons and `Request:` lines, `shas known`: a sha git does not know sends the hand-back back to the worker), exit 1 on a failure; `--unlanded` is the worker's run before the landing (`pushed` instead of landed) |
| `just repos-ship <repo> <branch> [--bg]` | the whole landing in one command: fetch, merge origin's default branch into the branch worktree (refused on a conflict), `machine-wait --for heavy`, the repo gate there (`gate-bg.sh run --all`, the PASS record is kept), on PASS `repos-land` + `repos-finish` + pull: `shipped <branch> as <sha> on origin/main` or `NOT shipped at <merge\|gate\|land\|finish>: <reason>`; `--bg` writes the line to `~/.local/state/hq/ships/` (`just cc-fleet-round` shows it). Prefer it to the hand chain; `repos-land` stays for a branch you gated yourself |
| `just repos-ship-wait <repo> <branch> [--max 20m]` | one bounded wait on a `repos-ship --bg` line (start it with `run_in_background` from the main checkout, `cd ~/dev/hq && just ...`: the landing deletes the branch's worktree and a shell started there dies; the completion notice wakes you, so never "tell me when it's done" and never a sleep loop): the result line, exit 0 on `shipped`, 1 on `NOT shipped` or a timeout (`RUNNING ... (timeout ...)`), 2 when no `--bg` ship was started |
| `just repos-land <repo> <branch> "<merge message>"` | lands a branch: merges it `--no-ff` into origin's default branch in a throwaway worktree, pushes, fast-forwards the main checkout, then finishes the branch: `landed <branch> as <sha> on origin/main, main checkout pulled` plus the `finished` line, or `NOT landed <branch>: <reason>` (conflict, push refused, already landed); run the repo's gate on the branch first, this does not |
| `just repos-finish <repo> <branch> [--worktree PATH]` | after a landing: proves the branch is on origin's default branch (ancestor or empty `git cherry`), then removes its clean worktree, deletes it locally and on origin: `finished <branch>: worktree removed, local deleted, remote deleted` or `NOT finished <branch>: <reason>` (exit 1); never the default branch, `hq-idle/*`, an unlanded branch or a dirty worktree |
| `just machine-test-containers rm <name>` | your own throwaway test container (labelled `hq.test=1`, e.g. started by a test-database helper): `removing <id> <name>`; any other container is `refused` (exit 1), production ones are never touched; `reap` removes the stale ones |
| `just repos-sweep [--apply] [--min-age-hours 24]` | extra worktrees of every clone and HQ: `REMOVE` (clean, nothing outside origin's default branch, not in the fleet ledger, idle over a day) or `KEEP` with the reason (dirty, unmerged, detached with last-commit age, live, young); dry run by default, `--apply` removes through `repos-finish` proofs, the maintainer approves the first apply |
| `just cc-retire <local_id> [--dry-run] [--retro TEXT]` | the four retire checks for one session, one line each (`landed` with requests, `retro`, `waiting` on issues, last turn or children, `record`), then `READY` or `NOT retired ...` (exit 1); on a pass it writes `retired <id>` into the fleet ledger and the closing marker `~/.local/state/hq/sessions/<id>.closing` (also under the transcript id) and prints the exact `archive_session` call, which only the agent can make; `--retro` records the retro line first |
| `just cc-why-idle [--json]` | "why is nothing running?" in one screen, no LLM, read-only: first line `work is blocked by: <causes>` or `nothing blocks work; N idle sessions simply were not nudged`, then one line per cause with its fix: usage (reading age, 429 cooldown, HOLD), machine capacity (why, load, CPU temperature), idle sessions with work left, keeper (last tick, fleet session, last `none:`), landing (shared files, heavy-lock queue, `no room`), pauses (queue, power, mode); exit 0, 2 on a tool error |
| `just repos-tag <repo> <tag>` | `tag <tag> on origin` or `NO tag` |
| `just repos-release <repo> <version>` | checks the version on main, runs the repo's `just release <version> push`, verifies the tag |
| `just repos-land-hq` | pushes HQ main, `in sync with origin/main` or ahead/behind counts |
| `hq repos-commit -m MSG [--add-all]` | commits in the current repo with the commit hooks measured (`metrics.py run --name q-commit`) and cache-aware (a staged tree with a gate PASS on the same base skips them): `COMMITTED <sha>`, or `HOOK FAILED [<hooks>]` followed by every failed hook's whole block (exit 1); never a hand-written commit whose output is cut to its last lines |
| `just q-fix [--no-gate]` | the fixer hooks once over all files (names the ones that changed files), then one `--all` gate: the result line plus every failed hook's whole block; replaces a fail-fast run per fixer (one change took nine) |
| `just cc-clean-own [--apply]` | removes the git-ignored build output (`target`, `node_modules`, `.venv`, `dist`, caches) of the linked worktree you are in; dry run by default, refuses a main checkout and symlinks; the guard's "Recursive rm" ask names it (a `find ... -exec rm` or `xargs rm` names no target) |
| `just machine-kill-own --port N` / `--pattern TEXT` | stops only processes whose cwd is in this worktree (SIGTERM, then SIGKILL), refuses and lists the rest; never `pkill`/`killall` (the guard asks) |
| `just machine-kill-child CLAUDE_PID [--apply]` | a tool command hangs: lists, then with `--apply` stops, the old non-MCP children of that Claude process; never the process itself, MCP servers or foreign processes |
| `just q-gate-bg [--all] [--no-cache] [log]` | the gate with a final `EXIT=<code>` line; start it with the harness's background run. Stops at the first failing hook; `--all` (the landing gate) runs every hook; a clean tree already passed on the same base prints `PASS (cached <sha>)`, `--no-cache` reruns |
| `just q-gate-result [log]` | `PASS`, `PASS (cached <sha>)`, `PASS (dirty)` (the fixers changed files: exit 1, commit and rerun), `FAIL EXIT=1 [<failed hooks>]` `RUNNING`, or `DEAD` (the gate process is gone without an EXIT line, e.g. an OOM kill) |
| `just q-gate-result --wait [--max S] [log]` | blocks until the gate ends (default at most 1800 s, then `RUNNING`, exit 1) and prints the one result line; run it with the harness's background run instead of polling a `RUNNING` line |
| `just q-gate-stop [log]` | `STOPPED gate <pid>` or `NOT RUNNING`: stops only the gate recorded in the log and its children (replaces `pkill -f gate-bg`) |

`q-gate-bg`, `q-gate-result` and `q-gate-stop` are `[no-cd]` recipes: from any repo run `hq q-gate-bg`, `hq q-gate-result` and `hq q-gate-stop` (the
`hq` command runs HQ's recipes from any directory; they gate the repo you are in via its `just q-check`). A repo
needs no copy of the script. Copies already made are removed on the repo's next slice. A machine without HQ (CI, a
clone elsewhere) does not run background gates, so it needs nothing.

`hq q-gate-bg` runs the **main checkout's** `gate-bg.sh`, not a worktree's copy: a branch that changes the gate
script (a new flag) is exercised only by `just q-gate-bg` run inside that worktree; through `hq` an unknown flag is
read as the log path.

A gate that waits for the heavy lock is tied to the `HEAD` it started on: if a commit lands meanwhile it ends
`GATE SUPERSEDED ... HEAD moved` (exit 75) and has to be started again. Commit every code, docs and backlog edit
first, then start `q-gate-bg`; do not commit while it runs. `just machine-wait` can refuse with "the holder is
alive; queue the job": then start `q-gate-bg` directly, it queues itself behind the lock. (Seen
2026-11-05: one wasted gate run of about 20 minutes.)

## Why

- `gate-bg.sh run` is synchronous: it runs the gate and writes `EXIT=` after it. So for a gate started with the
  harness's background run the completion notice does mean the gate ended (only a cached PASS returns at once). The
  notice says that it ended, not how: read the result line (`hq q-gate-result`) and act on it. A turn that ends on a
  finished background gate nobody read is stale (a 6 s lint failure once sat 41 minutes).
- Shorter calls cost fewer tokens on every round, and the plan is the binding limit
  ([autonomous-sessions.md](autonomous-sessions.md)).
- One implementation is fixed once for every session; a pasted line is fixed in none of them.

## Tests that run git: scrub the hook's `GIT_*`

A commit hook runs with `GIT_DIR` and `GIT_INDEX_FILE` set, and the test hooks it starts inherit them. A test that
runs `git init` or `git rev-parse --git-common-dir` then acts on the real repository: `git init` sets its
`core.bare=true` and every worktree command fails ("this operation must be run in a work tree"). Build the test's
environment with `scrubbed_env()` from `scripts/testing/git_env.py` (drops every `GIT_*`, sets a test identity), never
with a hand-written filter. HQ's `scripts/conftest.py` also scrubs every pytest run, and
`just machine-locks check` reports a checkout with `core.bare=true` that has a work tree (`fix` sets it back to false).

## Pinned by

- The scripts print the outcome word and set the exit code; check both on a known landed commit, a bogus one, a
  wrong release version and a missing log.
- The preflight digest lists the recipes, so a supervisor reads them at every start.

## Sources

- HQ implementation: `scripts/repos/ops.sh` (tests: `scripts/repos/test_finish.py`), `scripts/quality/gate-bg.sh`, `just/repos.just`, `just/quality.just`.
- Background-task completion semantics: <https://code.claude.com/docs/en/hooks> (hooks run per tool call, so a prepared
  command is one auditable call).
