---
summary: "Retiring a session (the four checks, retro, archive) and the periodic self-analysis of the supervisor"
group: agents
tags: [agents, workflow]
applies-to: [all]
reviewed: 2026-11-11
status: stable
---

# Agent retire: end a session and look back

Part of [agent-workflow.md](agent-workflow.md#stages); the picture is the Atlas `retire` view in `docs/architecture/workflow/`.

## Retire a session

A session whose work is done is archived with its retro, not left idle (idle sessions hold memory and clutter the list).
The supervisor does it on every live-check round, for sessions it started or the maintainer named:

1. **Landed.** With git, not the session's note: the branch is on `origin/<default>` (or has no commits ahead), the
   worktree is clean, nothing is unpushed. Anything left: nudge the session instead
   (`/sessions-continue`); a session that "needs the user" is listed, not touched. **Requests too:**
   `just docs-requests check --session <id>` (the same before a compaction and a hand-off) lists the maintainer's
   requests that have no item, landing or mark; each gets an item (`request: R-...`), a `Request: R-...` commit trailer
   or `requests.py mark <id> answered|dropped|merged`. Open ones refuse the retire from 2026-11-09 (record-only capture
   first; `ENFORCE_FROM` in `scripts/docs/requests.py`).
2. **Retro.** The reusable, project-neutral lessons from its hand-back go to `practices/` (research first, the
   repo keeps a short rule; research follows [research-flow.md](research-flow.md)) or, when the supervisor cannot write them now, an `area: practices` backlog item. Repo-specific
   ones go to that repo's AGENTS.md.
3. **Archive.** One path, in this order: a session that passed checks 1 and 2 is archived with `archive_session`
   (one call per session) when the maintainer's standing allow rule for it exists (the `manual-action` issue
   "allow archive_session in settings"; the auto-mode classifier denies the call without it). Until then, or when the
   app refuses (pinned, in use, Remote Control client, background task), the supervisor writes a **ready to archive**
   line in the ledger (handle, id, landed commit, retro done) and lists it **once** in the round table; the maintainer
   clears it in one click and the supervisor then removes the line. Never retitle with `[ARCHIVE]` (the marker
   vanishes from the sidebar once the maintainer closes the session by hand) and never force a session with live work.
4. **Record.** `retired <id>` in `autopilot.log` and the ledger (the ledger row's state becomes `retired`).

**One command.** `just cc-retire <local_id>` (`--dry-run` to only check; `--retro TEXT` records the retro first) runs the
four checks for one session, prints one line each and a verdict, refuses a session that waits on the maintainer (an
open `maintainer-question` or `manual-action` issue naming it, a last turn that ended asking, a child that waits) and
lists why. On a pass it writes `retired <id>` into the fleet ledger and the closing marker
`~/.local/state/hq/sessions/<id>.closing` (a Stop hook reads it), then prints the `archive_session` call; the agent makes
that call, the script cannot. A session retires itself the same way once its work is landed.

**Automatic detection.** The 5-minute no-LLM tick (`fleet-keeper.py tick`) runs the four checks as code
(`fleet-round.py retire --idle 30m`): a session another session started that passes all four, idle 30 minutes with no
running gate, gets a **ready to archive** ledger line (once) and a line in the attention file, which wakes the
supervisor. The call itself stays with the supervisor: a timer cannot call the desktop app's `archive_session`, and the
app still refuses pinned or in-use sessions. A session the maintainer started is noted, never touched. A failing check
is not forced (the keeper's one nudge per idle period stays); a hard failure such as `branch not found` is noted once
after 24 hours so the supervisor files a backlog item.

**Handles.** Every linked session has a stable `<repo>-<topic>` handle (the same as its title's prefix and topic, kept
when the title changes), recorded in the ledger's **handle** column next to its id. Reports, round tables and
hand-offs name sessions by handle, never by id alone, so "ready to archive" and "waiting on" lines stay readable after
a compaction.

What to take from Agile, Kanban and Shape Up on top of this (Definition of Done, WIP limits, cycle time, appetite,
weekly numbers): [agile-lessons.md](agile-lessons.md).

## Periodic self-analysis

The retro stage also looks at the supervisor's own work, not only at the landings. Run it every few hours of a
long run (at the 30-minute orient round when a milestone passed) and in the weekly Retro chore: six measured
questions, each answered from data (`just cc-usage`, `just cc-cost`, `just cc-watch`, `git log`, the ledger), then
one change per finding (a guard, a skill line, a practice sentence), never only a note. The loop and the per-landing
numbers are in [continuous-improvement.md](continuous-improvement.md).

| Question | Where to look | Typical fix |
| --- | --- | --- |
| Where did the plan go (model, project, session)? | `just cc-cost`, `just cc-usage` | smaller model or effort for that kind of work |
| What waited (idle sessions, gates, a full machine)? | `just cc-watch`, `just machine-capacity` | live check and a concrete message; one gate per branch |
| What was redone (gate reruns, rebases, a second reviewer of the same diff)? | `git log`, gate logs | one owner per step; full gate again after every rebase |
| Which claims did not match git (a note says landed or pushed)? | `git ls-remote`, `merge-base --is-ancestor` | check before reporting, as in the fleet skill |
| Which block repeated (a classifier denial, a 429, a wait loop)? | session notes, hook output | route it to the actor that can do it (the supervisor lands what a session may not push) |
| Was it the right thing, not only done right (MOE, not MOP)? | landed items' `outcome:` lines, items reopened, usage after release | measure the outcome, or fix a measure nobody can read ([MOP/MOE](measures-of-performance-and-effectiveness.md)) |

Findings that repeat go to a practice page; one-offs go to the repo's backlog. Say in the report what changed since
the last analysis, not only what was found.

## Sources

- Split out of [agent-workflow.md](agent-workflow.md), which keeps the sources of the whole flow; the Atlas `retire` view is in `docs/architecture/workflow/`.
- Claude Code subagents: <https://code.claude.com/docs/en/sub-agents>
