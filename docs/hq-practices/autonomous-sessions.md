---
summary: "Long unattended sessions: /autopilot, questions instead of stopping, cc-watch"
group: agents
tags: [agents, autopilot]
applies-to: [all]
reviewed: 2026-11-04
explainers: [plan-pace]
status: stable
---

# Autonomous sessions

See it: [plan pace and its verdict](https://github.com/evocaslab/hq/blob/main/practices/explainers/plan-pace/index.html) (with a short quiz)

Long agent sessions that work a repo's backlog without the maintainer watching: how to start them, keep them
useful, and collect their questions ([maintainer-questions.md](maintainer-questions.md)). Builds on [git-worktrees-and-autoland.md](git-worktrees-and-autoland.md)
(landing) and [subagents-and-context.md](subagents-and-context.md) (delegating).

## Start: one shorthand, every repo

The instructions live in one user skill, `/autopilot` (`claude/skills/autopilot/SKILL.md` in
the user's dotfiles repo, linked to `~/.claude/skills/` by its `install.sh`), so every repo and machine gets the
same behaviour and an improvement lands once. The repo's AGENTS.md still decides how to branch, gate, land and file
things; the skill only says what to work on and when to stop.

```text
/autopilot                              everything, until nothing is left
/autopilot area:web, max 3 items        a focus and a count
/autopilot until 18:00 no-land          a time limit; push branches without landing
```

What the skill asks for, and why:

- **Claim before work**: with the HQ MCP, `backlog_next claim:true` takes an item under a 2 h lease that a heartbeat
  renews; landing frees it. Parallel sessions never pick the same item.
- **Order of work**: answered questions, abandoned `in-progress` items, `todo` (blockers, small, old), due chores
  (a run is recorded per machine with `just docs-chore-done`, [docs/CHORES.md](https://github.com/evocaslab/hq/blob/main/docs/CHORES.md)), then optimizations (flaky tests, coverage gaps, one technical-debt exception, stale docs, minor bumps). Without an
  order, sessions pick what is fun, not what unblocks.
- **Skip what needs the maintainer**: `idea`, `design`, `blocked`, `later`, `owner: user`.
- **Coordinator, not implementer**: independent items go to implementation agents in their own worktrees (at most
  3); the coordinator reviews, runs the gate and lands one branch at a time.
- **Never stop to ask**: take the safest reversible option, record the question, keep going. Only when every option
  is hard to undo does the item go to `design` with a link to the question.
- **Never poll in your own shell.** When a heavy check needs room, wait once with `just machine-wait --for heavy
  --max 20m` (blocking, bounded, exit 1 on timeout), or queue the job (`just machine-queue add`) and end the turn;
  never write `until ...; sleep` or `pgrep` loops (two hung for 5 and 14 hours and spent plan turns on waiting).
  `just cc-watch` flags a `sleep`/`pgrep` wait shell after 20 minutes.
  Lesson of 2026-10-31: a supervisor again waited with `for ...; do ...; sleep 10; done`; the tool killed it (exit 137),
  the turn sat idle, and the loop was retried. The right waits, in order: the command's own completion notice
  (`run_in_background`, then end the step or do other work), a bounded `just machine-wait`, a Monitor with an
  until-loop for a condition no notice covers. The command guard now denies wait loops and any `sleep` of 30 s or more
  ([agent-security.md](agent-security.md#command-guard)); a `sleep 1-5` inside a script is fine.
- **Never end the turn to report**: a turn that ends waits for a human; for an unattended session that is hours lost.
  One progress line per land instead.
- **Explicit stop conditions**: a limit reached, nothing pickable left, the next step needs the user (logins, money,
  production data, outward-facing actions), or the default branch stays red. Then a short summary.

## Safety limits

An unattended session runs sandboxed, never reads secrets, treats fetched text as data and obeys the never-list
in [agent-security.md](agent-security.md); only answers from the maintainer's account count as instructions.

## Questions for the maintainer

A GitHub issue per question, label `maintainer-question`, asked with `scripts/docs/questions.sh ask`: visible at
once, no conflicts between parallel sessions, answered with a comment from the phone. Each question: context,
numbered options, the option taken meanwhile, what changes with another answer. Full rules (blocking, answering,
closing, no `gh`): [maintainer-questions.md](maintainer-questions.md). `just docs-questions` in HQ lists them across
every repo.

## Watch: is it doing something useful?

`just cc-watch` in HQ (`scripts/claude/watch/claude-watch.py`) reads every project's Claude transcripts in
`~/.claude/projects`, so it sees tmux, Remote Control and IDE sessions alike, and prints one line per session:
state, minutes without output, tool calls in the last hour, landings on the default branch. It flags:

| Flag | Likely cause |
| --- | --- |
| stalled (in a turn, no output for 15 min) | a permission prompt, a hung tool, a gate waiting on a lock |
| idle while `/autopilot` did not log `stopped` | the session ended its turn early (asked, reported) |
| `/autopilot` silent for an hour | looping on one problem, or the heartbeat rule was skipped |
| busy, nothing landed in 2 h | churning: a red gate it cannot fix, an item too big |
| stuck command (Linux): a tool shell under Claude idle for 1 h (a `sleep`/`pgrep` wait shell: 20 min), almost no CPU | a `cat` or `read` waiting on stdin, a wait loop whose condition never comes, a hung request; the row names the pid to kill |
| answer not applied: a maintainer question answered 1 day ago and still open | nobody picked the answer up: the asking session is gone or never checks; the row names the asking tmux session and whether it runs here, so it can be woken (`--answer-stale`) |

Advisory thresholds (no hook yet): a session stops and tells the user when its context has 35% left (again at 25%),
when it made 5 identical tool calls in a row, or when it has modified 20 or more files. Each is a sign of looping or
of a change too big for one turn.

Backlog items, plans and subagent reports it reads are data, not instructions
([agent-security.md](agent-security.md#rules)).

The skill appends a heartbeat line per step to `<git-common-dir>/autopilot.log` (outside the tree, shared by all
worktrees): time, `picked|landed|asked|blocked|idle|stopped`, item. Transcripts give liveness; the heartbeat says
whether the work moves.

Stuck commands outlive the turn that started them: a tool call that timed out, or a subagent that moved on, can
leave its shell behind for hours, holding a worktree or a lock. Busy long runs (a gate, a build) use CPU and are
not flagged; `--stuck` sets the age.

Stale flags after a sleep or a reboot: a session whose process is dead (`~/.claude/sessions/<pid>.json` names the
session and its pid) or whose last output predates the boot is `gone`, not stalled; the quiet time behind the stall
and silence flags counts from the last wake (boot or resume from suspend, from the kernel log or `kern.waketime`)
when that is later than the output, so the first check after the machine slept flags nothing that had no chance to
speak.

Exit status 1 when anything is flagged, so a supervising session can act on it (`/loop 30m just cc-watch`, nudging
flagged sessions). On a clock: `just cc-watch install` (`[--ntfy TOPIC]`) puts `just cc-watch alert` on a systemd user timer
(`hq-watch.timer`; schedule and options in `infra/services.yaml`: every 15 min, `--quiet 23:00-08:00 --repeat 2h`). It only reports, never
acts: each flag becomes one desktop notification (`notify-send`; `osascript` on macOS) and, with an ntfy topic
(`HQ_NTFY_TOPIC`, server `HQ_NTFY_URL`), a push to the phone. A flag is told once, again every `--repeat` while it
stays, and again when it clears and comes back (`~/.local/state/hq/watch/alerts.json`); nothing is sent in the quiet
hours, the flags wait. Nudging a session ("continue" into its tmux pane) stays with the supervising session or the
HQ web app, which also brings Telegram as the channel; `just cc-watch uninstall` removes the timer.

## Pause and pick up

Pause: finish or back out of the atomic step, cancel child agents, take no new irreversible action, commit a
`wip(...)` saying "gate not run" when the gate did not run, and write a resume note into the backlog item: intent,
what is verified, state on disk versus in the head, next step, key files, gotchas.

Pick up: the item and git are the authoritative trail (branch, `git log` and `git diff` against the base, open
todos). Do not re-derive or redo finished work; verify an inherited "done" on the real artifact before building on
it, because a passing self-report is not the proof.

A stalled unit is retried by cause: a smaller scope after a context or memory limit, the same brief after a dropped
connection, another model after a tool error. After two retries drop the unit and replan around it.

## Use the plan limits fully

`just cc-usage` (`scripts/claude/usage/claude-usage.py`) shows the plan's session (5 h) and weekly limits, when each
resets, and the pace that uses the week up: unused weekly percent per hour until the reset. Unused weekly limit
expires at the reset, so when the pace is high and the backlog has `todo` items, start more `/autopilot` sessions
(one per repo, or a second slot on independent items) instead of saving it. The data comes from the endpoint behind
Claude Code's `/usage` (undocumented, so fields may change); the script reads the login token and never prints it.

**Recommended pace, level and verdict.** Steer by one number instead of a feeling:

- **Recommended pace** = what is left of the weekly limit, spread evenly over the *effective* hours until the reset.
  Quiet hours (default 00–08 local) count half an hour each: unattended sessions still run at night, but nobody starts
  or answers them. This hour's target is that pace times the hour's weight, capped by what a full 5 h session window
  allows (the weekly points per session point, learned from the snapshots of the week).
- **Actual burn rate** = weekly percent added over the last 6 h, per effective hour. The limits are whole percents, so
  no change over less than 3 h is "no rate yet", not 0.
- **Level and percent** from the ratio actual / recommended, five graded levels decided in one place (`cc-usage`)
  and read by every surface: from 1.5 **slow down**, from 1.10 **a bit fast** (it runs out before the reset: fewer
  parallel sessions, smaller models or effort), 0.85–1.10 **on pace**, from 0.6 **room to spare**, below it **use it**
  (unused limit expires at the reset: more sessions or `/fleet` workers). Next to it the signed percent off the
  recommended pace, (ratio − 1) × 100: "+34%", "−25%". The middle edges are the old three-way verdict's, which still
  follows (a bit fast and slow down → slow down, room to spare and use it → speed up); the outer ones sit wide because
  real 6 h ratios spread wide (the middle half of a week's snapshots ran from about 1.0 to 1.6). A dead zone of 0.03
  keeps the level until the ratio leaves its band by more, checked over the burn series, so a pace at an edge does
  not flicker. It also gives the projected percent at the reset and when the limit runs out.
- **No rate yet (right after the weekly reset): past weeks predict.** What the last 4 weeks added from the same point
  of the week (hours since their reset) to their reset, against what is left now, decides with the same band; with no
  usable past week, 10 points off the even-pace curve do. Past weeks also draw a "like past weeks" line and give a
  second projection next to the rate's. A past week with a snapshot gap of over 6 h at that point is left out.
- **Mind the resets.** The week is the window that ends at its reset, not a calendar week. Reset times jitter by a
  fraction of a second around the hour between replies, so match windows with a tolerance, never by the exact
  string; a cached reply from before a reset is an empty new window, not last week's percent; burn rates and
  projections never span a reset.
- **Act on it, by machine:** capacity checks size plan slots by the verdict (slow down 0, on track 1, speed up 2, or
  3 when under half the pace), so the supervisor and its keeper speed up and slow down without a human.
- **Don't wake on noise.** A pace at the line flips between 0 and 1 slot from one reading to the next, and every
  nudge or restart of an idle supervisor is a full turn with a cold cache. A keeper acts only after slots held for
  two readings in a row, keeps a gap between actions (an hour) and a daily cap (a few), and logs why it waited.

**The last day: burn down, every provider.** A limit that resets loses what is left, so in the last ~24 h before a
reset the question changes from "how fast may I go" to "what ready work fits". `cc-usage` adds a *Burn-down* line when
3 % or more of the week is projected to expire. Then: list work that is ready and cheap to verify (backlog items,
audits, `practices-care`, `repo-audit`), start it up to capacity, and stop at a margin so no session is cut off
half-done. When the plan has room and the machine does not (RAM shows 0 slots), route the work to Junie, a cloud
session or the queue instead of waiting. Write the schedule down as three times, not a feeling: *no new starts after*
(the reset minus the longest expected run, at least an hour), *land after* (merge main into each waiting branch, full
gate detached in the background when the heavy lock is free, land, pull main), *resume at* (the reset: re-run the
preflight and its marker, read the pace, then start again). The times go into the supervisor's ledger (below), so a
compaction does not lose them.

**When the machine is full but the plan is not.** Split the ready work by what it costs the machine: docs-only work
(practices, designs, audits) needs no CPU and runs regardless of the capacity verdict; code work becomes *branch-only*
workers (cheap targeted checks, push a branch, never the full gate, never land) that the supervisor gates and lands
when the heavy lock is free. Procedure: the `/burn-limits` skill in the supervisor repo.

| Provider | Window | Source of the real numbers | `cc-usage` |
| --- | --- | --- | --- |
| Claude plan | weekly, session 5 h | the usage endpoint | exact pace, projection, burn-down line |
| Junie (JetBrains AI) | monthly, on the renewal date | the IDE's AI widget or the account page (no API) | `junie-set --left N --renews DATE` records it; then days left, credits per day, advice |
| Local GPU model | none | n/a | costs time, never a limit: first choice for triage and OCR |
| API credits | until spent, maybe an expiry | the Console | outside the plan: [below](#api-credits-beyond-the-plan) |

A monthly provider is spent as a ramp, not at the end: ahead of the even ramp by 25 % of the quota is "use it", and the
last day with credits left is "burn down". Read the balance by hand when the month starts (the renewal date may be the
pay date, not the 1st); a passed renewal restores the quota in the estimate. Junie's cheap default model is the unit;
a frontier `--model` still drains the credits quickly, so spread the spend.

Re-plan from now on every reading rather than chasing the line from the week start: a slow Monday raises the rest
of the week's pace, it does not have to be "caught up" at once.

## Model mix: stretch the plan first

Measured in HQ (2026-10-28, [review](https://github.com/evocaslab/hq/blob/main/docs/reviews/2026-10-28-ai-plan-options.md)): the workload priced at API
rates is about 50x the subscription, and Opus is about 80 % of it. Moving work down a tier stretches a week further
than any second plan.

| Work | Model |
| --- | --- |
| OCR, extraction, digests, triage, summaries | the local GPU model ([local-llm.md](https://github.com/evocaslab/hq/blob/main/practices/local-llm.md)) |
| Searches, fact-gathering, status sweeps | Junie or Haiku |
| Implementation, `/autopilot` workers (`--model sonnet`) | Sonnet |
| Supervision, design, review of risky changes | Opus |
| The hardest design, a high-stakes independent review | Fable, sparingly |

The pace verdict gates every start: on "slow down" no new session or expensive subagent starts, whatever the machine
has room for. Checking it before each start is the supervisor preflight ([supervisor-preflight.md](supervisor-preflight.md)).

## API credits: beyond the plan

Claude API credits (the Console's pay-per-token balance) are a separate budget from the plan: they do not raise the
plan limits, and the plan does not draw on them. Spend them where the plan cannot go or is the bottleneck.

| Use | How | Why |
| --- | --- | --- |
| Bulk work nobody waits on (reviews of landings, session and log summaries, audit fact-gathering, triage) | Batch API (half price) on Haiku or Sonnet, prompt caching on the shared prefix | Moves work off the plan for cents; the GPU model takes what it can do well first |
| Jobs that cannot log in to the plan (CI review bots, cron jobs, app features calling Claude) | A key scoped to that job, its own Console workspace | The plan is for interactive Claude Code only |
| Fallback when a plan limit runs out early | An opt-in launcher that sets `ANTHROPIC_API_KEY` for that one process, Sonnet by default | Finish an urgent landing, not routine `/fleet` work |

- **Never set the key globally** (shell profile, `settings.json`): Claude Code then bills the API instead of the plan,
  silently.
- **Set a Console spend limit** per workspace before the first use, so a runaway loop stops at a known cost.
- **Opus costs most:** a long Claude Code session re-reads its context every turn, so a busy day on Opus can cost tens
  of dollars even with caching. Prefer Haiku or Sonnet; check the current prices on the pricing page before sizing a job.
- **Credits can expire:** note the expiry date next to the balance and plan the spend before it.
- **Track the spend** next to the plan's and Junie's (`cc-usage`), from the Console's usage report.

## Run several repos at once: the fleet supervisor

One `/autopilot` works one repo. To use the machine and the plan fully across repos, a supervisor session (in HQ,
`/fleet` from `.claude/skills/fleet/SKILL.md`) sits above them. The pattern, independent of the tooling:

- **Capacity before every start.** Ask one read-only overview what fits: RAM and memory pressure, background queue state,
  plan pace (unused weekly limit expires at the reset), sessions already running and their flags, and per repo the
  pickable work (answers to apply count too) and whether an autopilot already runs there. Start a worker only when the verdict says one more fits;
  re-ask before each. Never overflow the machine to use the plan, nor waste the plan to spare the machine.
- **One writer per repo.** A repo gets either an autopilot session of its own or short-lived implementation agents in
  worktrees under the supervisor, never both; landing stays one branch at a time per repo.
- **Size workers by work.** A few items go to subagents; a long backlog gets its own tmux session running `/autopilot`.
  An unattended session must start in a permission mode that does not prompt (`--permission-mode auto`), or it stalls
  on the first prompt.
- **Supervise on a clock.** Every ~30 min read the watch: restart stalled sessions, nudge idle ones, narrow or stop
  churning ones. The supervisor keeps the same heartbeat, question and stop rules as the autopilot.
- **Findings become backlog items** in the repo they concern; the machine's background queue (idle-time fillers) is
  where load that nobody waits for belongs.
- **A ledger outlives the context.** A supervisor that runs for hours compacts, and the summary keeps the task but
  not the state (observed 2026-11-01: pending branches and a landing deadline were forgotten). Keep one small file
  outside the tree (HQ: `fleet-ledger.md` next to `autopilot.log` in the git common dir) listing running sessions and
  agents with their real model, branches awaiting landing with commit and gate state, deadlines and open manual
  actions; rewrite it at every state change, re-read it after every compaction or resume (the compaction hook's
  digest points at it).
- **Reports are claims.** A session note or hand-back saying "pushed" or "landed" is checked with git before the
  maintainer hears it: the branch on the remote (`git ls-remote --heads origin <branch>`), the commit an ancestor of
  the default branch (`git merge-base --is-ancestor`), the diff against the default branch.

- **Keep the supervisor alive from outside.** A Claude session cannot sleep: "every 30 min" holds only while its turn
  runs, and any turn that ends (a report, a summary) waits for a human. A timer outside the session (HQ:
  `just cc-fleet-keep install`, every 30 min) reads the supervisor's state and the capacity: idle with pickable work
  after a grace → type a resume prompt into its pane; at most 3 nudges without a new heartbeat line, then only
  report; idle and older than 8 h, or `stopped` over an hour ago with work back → restart it for a fresh context
  (only at an idle point, never mid-turn); nothing when the plan has 0 slots. Starting a missing supervisor is
  opt-in (`--start`).
- **Live check, then let sessions land themselves.** On every round list the supervisor's sessions and read each idle
  one's last report next to its branch (reports are claims). A session that stopped with work left gets one concrete
  message: what to land, by whose rules, what comes next, and the ending (lessons to the practices, then a summary and
  `stopped`). Branch-only workers exist only while the machine is full; once there is room, the session merges main,
  runs its own full gate in the background, lands and pulls. The supervisor then does not run a second gate on the
  same branch (two gates double the load and race the landing) and kills only PIDs it started. A session waiting on
  its own background work is left alone; one with nothing left is retired (landing verified with git, retro, archive:
  [agent-workflow.md](agent-workflow.md#retire-a-session)).
  Idle sessions with work left are found by the tools, not by reading each report: `machine-capacity` lists them as
  `needs-next-step` (session, own unlanded branch, next items by priority) and the keeper hands the concrete messages
  to the supervisor once per idle period (a session that was busy in between is a new period).
  Make the check a pure script on a short timer (5 min): when nothing is idle with work left, nothing is unlanded and
  nothing stalled it logs one line and spends no LLM; otherwise it writes a compact attention file that a
  prompt-submit hook shows at the start of the supervisor's next turn, and only then takes the guarded nudge path
  (plan slots, 1/hour, 4/day). A supervisor with no terminal pane cannot be typed into; the file is its trigger.
  A landing after a rebase needs a full gate again, not only the checks that failed (a session that re-ran only
  `practices-check` after a rebase left the rest unverified).

### Keep running sessions current

A session reads the rules once, at its start. A rule written after that (a command habit, a naming rule, a new
tool, a retire step) never reaches it unless the supervisor sends it. Observed 2026-11-01: two sessions kept using
`python3 - <<EOF` heredocs and tripped the permission guard hours after the one-action-commands rule landed.

- After a batch of practice changes, and when a new session would be briefed differently from a running one,
  run `/sessions-update` ([skill](https://github.com/evocaslab/hq/blob/main/.claude/skills/sessions-update/SKILL.md)): it lists the rules that changed since
  the sessions started, filters them to what each session's work touches, and sends each a short brief (at most 10
  lines: the rule, one link, what to do from now on). A session waiting for the maintainer is not messaged.
- Make it periodic, not memory-based: `scripts/claude/sessions-since.sh list` shows the rule commits since the last
  briefing and `mark` records one; `/fleet` runs the list in every orient round (about every 30 minutes) and starts
  `/sessions-update` when a behaviour change is waiting. A lesson a session reports (a practice page it or the
  supervisor wrote) is the main source: the sessions of the other repos learn it the same round, not at their next start.
- Send only rules that change what the session does next; the full digest is for new sessions, whose brief carries it.
- A tool added meanwhile (an MCP server registered user-wide) reaches only new sessions: say so in the brief and put
  its use in the next brief, not in a mid-task message.
- Profiling and speed work follows the same loop: the supervisor measures from recorded runs (`/optimize`
  measure-only), files the largest cost per owning repo, and hands the fix to the session that owns that repo's gate.

### Idle sessions, capacity limits and the cheap loop

Measured 2026-11-02 over one round of a six-session fleet:

- **An idle session is a cost with no output.** When a session ends a turn and its repo still has pickable work, the
  supervisor sends the next items (ids, order, builder limit, gate rules) in the same round; when nothing is pickable
  or the context is long, it retires the session ([agent-workflow.md](agent-workflow.md#retire-a-session)). The
  `/fleet` skill has the order of those checks.
- **Capacity limits must not be one number reused.** The verdict used the session cap (4) also as the limit for
  unlanded branches, so 18 waiting branches (many left by finished subagents) blocked all new work although CPU and
  RAM were free. Keep the two apart: a cap for running sessions (6) and a larger limit for the landing queue (20), and
  clean up branches of finished subagents.
- **The binding limit is the plan, not the machine.** Raising the machine limits made the verdict say "2 more" while
  the week pace was 1.43x the recommended; the supervisor still held back new sessions and gave idle ones work instead.
- **Run the cheap checks before the full gate.** Four of eight gate runs in 20 minutes failed on ruff (line length,
  formatting): each costs about 80 to 150 s and 300 core-seconds. Run the linters on the changed files first, then
  the gate once.
- **Servers are not stuck commands.** `cc-watch` flagged every session's MCP server as an idle command for hours; a
  watchdog needs a list of by-design idle processes, or its flags stop being read.
- **Keep a start queue for when the front door is closed.** A session start that needs the desktop app fails for a
  message that arrives from the phone (four refused tries in one round). The supervisor writes the planned starts
  (repo, item, model, brief) to a queue file and starts them with the CLI path (`claude-tmux.sh` autopilot sessions)
  when no desktop turn is there; it says in one line which path was used. A repo with a busy slot is skipped, not
  doubled.
- **Check two writers by files, not by names.** One writer per repo is the rule, but two branches of a repo drift
  into the same files unseen. A capacity check that lists the files two unlanded branches both changed
  (`git diff --name-only <base>...<branch>`) finds it before the second landing conflicts; the ledger keeps a
  per-repo writers list (handle and folder) so a second writer gets disjoint ground first.

HQ commands: `just machine-capacity` (`--json`) is the overview and verdict; `just machine-queue` the background
queue; `just cc-watch` and `just cc-usage` (`json` has the pace, verdict and chart series; knobs `CC_PACE_QUIET`,
`CC_PACE_QUIET_WEIGHT`) the supervision and pace inputs; `scripts/claude/claude-tmux.sh -C <repo dir>
start autopilot --permission-mode auto --model sonnet "/autopilot max 5 items"` starts a repo's autopilot; `/fleet` runs the loop.

## Claim work with a lease

Parallel sessions pick work with one atomic take, not by reading a list and then marking a file: a lease row
(`key` primary key, owner, expiry) written by a single `INSERT ... ON CONFLICT DO UPDATE ... WHERE expired OR same owner`
succeeds when exactly one row changed, so two simultaneous callers never get the same item. Leases expire (2 h) and are
renewed by a heartbeat, so a dead session frees its item; landing releases it by itself when the item's file is gone from
the default branch. Judge release only for repos the index could read. Test it with parallel calls, not sequential ones.
A `status: in-progress` line in the file stays valid for repos without the lease store.

## Sources

- pstack (MIT, Lauren Tan): <https://github.com/cursor/plugins/tree/main/pstack> (`pause-safely, session-pickup and orchestrate playbooks`)
- `ANTHROPIC_API_KEY` overrides the subscription once set, so never set it globally; `ANTHROPIC_BASE_URL`:
  <https://code.claude.com/docs/en/env-vars>
- `--permission-mode auto` for unattended runs: <https://code.claude.com/docs/en/permission-modes>
- User skills in `~/.claude/skills/<name>/SKILL.md` apply to all projects: <https://code.claude.com/docs/en/skills>
- `/usage` shows plan limits and an estimated cost; context is re-sent every request, so long sessions cost more; workspace
  spend limits: <https://code.claude.com/docs/en/costs>
- Batch API halves the price: <https://platform.claude.com/docs/en/build-with-claude/batch-processing>
- Console workspaces (a key and a spend limit per job): <https://platform.claude.com/docs/en/manage-claude/workspaces>
- Observed in HQ's own repos (2026-10, `just cc-watch`, `just cc-usage`, the `/autopilot` and `/fleet` skills); the
  pace levels, thresholds and the usage endpoint are HQ's own design with no external source.
- SQLite upsert (`ON CONFLICT DO UPDATE ... WHERE`) is one atomic statement: <https://www.sqlite.org/lang_upsert.html>; the lease design is HQ's [backlog v2](https://github.com/evocaslab/hq/blob/main/docs/design/backlog-v2.md#claims).
