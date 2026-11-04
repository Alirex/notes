---
summary: "The agent flow made explicit: stages from request to landed, which role (supervisor, worker, reviewer) owns each, and the check between them"
group: agents
tags: [agents, workflow, review]
applies-to: [all]
reviewed: 2026-11-01
explainers: [retire-checks]
status: stable
---

# Agent workflow: stages and roles

See it: [the four retire checks](https://github.com/evocaslab/hq/blob/main/practices/explainers/retire-checks/index.html) (with a short quiz)

HQ's flow is the same for a one-line fix and a rollout; only the size of each stage changes. Roles are lanes by tier,
not personas ([subagents-and-context.md](subagents-and-context.md#pick-the-cheapest-tier-that-works)).

## Roles

| Role | Does | Tier | Never |
| --- | --- | --- | --- |
| Supervisor | Picks work, writes the item, briefs, verifies, lands | Opus (judgment), Sonnet for routine | Edits what another session owns; trusts a report |
| Worker | Implements one item in its own worktree, runs the gate, pushes the branch, reports | Sonnet | Lands; weakens a gate; widens scope |
| Reviewer | Applies named [lenses](review-lenses.md) to the diff, returns findings with `file:line` | Higher tier than the worker, or the local model / Junie for a cheap second read | Edits the code under review |
| Searcher | Facts with `file:line`, no judgment | Local model, Junie, Haiku | Changes files |

## Stages

| # | Stage | Output | Check before the next stage |
| --- | --- | --- | --- |
| 1 | Request | A backlog item, at once, if not finished in the same session | The whole request is in it ("whole requirement" lens) |
| 2 | Preflight | Pace, capacity, overlaps, tier per start ([supervisor-preflight.md](supervisor-preflight.md)) | `/preflight` ran; "slow down" starts nothing |
| 3 | Plan (only if unclear or cross-cutting) | The item's design section: premise, files, done-when | The premise is stated so it can be proven false |
| 4 | Build | A branch with small commits, gate green | The worker's own gate run |
| 5 | Review | Findings per lens | `just q-integrity` on the branch; the supervisor re-runs the gate, a report is a claim |
| 6 | Land | Merged, pushed, main checkout pulled, item deleted | Gate on the merged result, every hook ran, none skipped for a missing tool ([quality-gate.md](quality-gate.md#every-step-ran-or-the-gate-is-not-green)) |
| 7 | Retro | A practice, a guard or a chore change | Reusable lessons go to `practices/`, the repo keeps a short rule ([chore](https://github.com/evocaslab/hq/blob/main/docs/CHORES.md)) |
| 8 | Retire | The session archived, `retired <id>` in the ledger | The four checks of [Retire a session](#retire-a-session) |

After landing, clean up with `just repos-finish <repo> <branch>` (it proves the landing, then removes the clean
worktree and deletes the branch locally and on origin; [prepared-commands.md](prepared-commands.md)), never a
hand-written `worktree remove` / `branch -D` chain.

**Definition of Done** (a worker's hand-back block shows each line; a missing line is "not done"; the reasons are in
[agile-lessons.md](agile-lessons.md)):

- [ ] Gate green on the merged result.
- [ ] Item file deleted in the landing commit.
- [ ] Pushed, landed, and the main checkout pulled.
- [ ] Lessons saved (a `practices/` commit or an `area: practices` item).
- [ ] Docs match the code (README, AGENTS.md, changelog where the repo keeps one).
- [ ] **Checked in:** `real app` / `mock app` / `tests only`, with the data it used (the real log, a fixture, a seed).
      A UI or endpoint change that says `mock app` or `tests only` is flagged by the supervisor before it reports
      done: it is looked at in the real app, or listed as "unchecked in the real app" in the round table. (The MCP stats
      card landed after a check on a mock only; the real log showed it differently.)
- [ ] **Outcome measured:** `yes` (number, source) / `not yet` (check date) / `n/a`: gate green is MOP, the outcome
      is MOE ([measures-of-performance-and-effectiveness.md](measures-of-performance-and-effectiveness.md)).
- [ ] Session retired ([Retire a session](#retire-a-session)).

Small work skips 3 and the separate reviewer but not 5's checks. Questions never stop the flow: a
`maintainer-question` issue, safest reversible choice meanwhile ([autonomous-sessions.md](autonomous-sessions.md)).

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

**Handles.** Every linked session has a stable `<repo>-<topic>` handle (the same as its title's prefix and topic, kept
when the title changes), recorded in the ledger's **handle** column next to its id. Reports, round tables and
hand-offs name sessions by handle, never by id alone, so "ready to archive" and "waiting on" lines stay readable after
a compaction.

What to take from Agile, Kanban and Shape Up on top of this (Definition of Done, WIP limits, cycle time, appetite,
weekly numbers): [agile-lessons.md](agile-lessons.md).

## Guards by stage

| Stage | Guard |
| --- | --- |
| 2 | `PreToolUse` preflight gate ([supervisor-preflight.md](supervisor-preflight.md)) |
| 4 | Destructive-command guard ([agent-security.md](agent-security.md#command-guard)) |
| 5 | `just q-integrity` ([review-lenses.md](review-lenses.md#gate-integrity)), `just q-check` |
| 6 | Landing rules of the repo; push every commit |

Read the gate's exit code (`echo $?`, or the completion notice's status) before a push: a log that scrolled past is not a
pass. A failure is fixed or reported, never pushed over; an unrelated flaky test is re-run once under low load and named
in the report.

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

## A fixed bug is reported again

Before looking for a missed code path, compare times: the commit that fixed it, the build time of the image or binary
that runs, and the container's start time (`docker inspect`, `docker ps`: "Up N hours" keeps serving the old code).
A long-running service only changes when it is rebuilt and restarted, so a landing that does not redeploy leaves
the reported bug in place. Reproduce against the code first (a parser run on the maintainer's files), then against the
running build; say in the report which of the two still fails. A deploy to a machine that is the only one in use is
a production change: it needs the maintainer's instruction, or goes to a `manual-action` issue with the exact commands.

Related, for a migration with a frozen oracle: a new golden needs the update switch of its test (`UPDATE_GOLDENS=1`),
a recorder names the tag that retires its oracle, and a background gate must `cd` into its worktree inside the
command (the launcher shell's directory is not the gate's).

Reading a gate result: a commit made while the gate waits for the lock supersedes it (`EXIT=75`), and a hook
formatter's uncommitted edits or an old `/tmp` log can pass for a result. Check the log's mtime and grep for `GATE OK`,
or use `hq q-gate-result`. Run `git push` and the gate as separate commands: in `git push && gate` a failed push (a
GitHub 500) silently skips the gate. New fixture files also need the goldens list regenerated (`UPDATE_GOLDENS=1`).

## Sources

- gstack sprint (think, plan, build, review, test, ship, reflect): <https://github.com/garrytan/gstack>
- Claude Code subagents and hooks: <https://code.claude.com/docs/en/sub-agents>, <https://code.claude.com/docs/en/hooks>
