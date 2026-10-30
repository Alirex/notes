---
summary: "Long unattended sessions: /autopilot, questions instead of stopping, cc-watch"
group: agents
tags: [agents, autopilot]
applies-to: [all]
reviewed: 2026-10-28
status: stable
---

# Autonomous sessions

Long agent sessions that work a repo's backlog without the maintainer watching: how to start them, keep them
useful, and collect their questions ([maintainer-questions.md](maintainer-questions.md)). Builds on [git-worktrees-and-autoland.md](git-worktrees-and-autoland.md)
(landing) and [subagents-and-context.md](subagents-and-context.md) (delegating).

## Start: one shorthand, every repo

The instructions live in one user skill, `/autopilot` (`claude/skills/autopilot/SKILL.md` in
`Alirex/claude-dotfiles`, linked to `~/.claude/skills/` by its `install.sh`), so every repo and machine gets the
same behaviour and an improvement lands once. The repo's AGENTS.md still decides how to branch, gate, land and file
things; the skill only says what to work on and when to stop.

```text
/autopilot                              everything, until nothing is left
/autopilot area:web, max 3 items        a focus and a count
/autopilot until 18:00 no-land          a time limit; push branches without landing
```

What the skill asks for, and why:

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
has room for.

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

- **Keep the supervisor alive from outside.** A Claude session cannot sleep: "every 30 min" holds only while its turn
  runs, and any turn that ends (a report, a summary) waits for a human. A timer outside the session (HQ:
  `just cc-fleet-keep install`, every 30 min) reads the supervisor's state and the capacity: idle with pickable work
  after a grace → type a resume prompt into its pane; at most 3 nudges without a new heartbeat line, then only
  report; idle and older than 8 h, or `stopped` over an hour ago with work back → restart it for a fresh context
  (only at an idle point, never mid-turn); nothing when the plan has 0 slots. Starting a missing supervisor is
  opt-in (`--start`).

HQ commands: `just machine-capacity` (`--json`) is the overview and verdict; `just machine-queue` the background
queue; `just cc-watch` and `just cc-usage` (`json` has the pace, verdict and chart series; knobs `CC_PACE_QUIET`,
`CC_PACE_QUIET_WEIGHT`) the supervision and pace inputs; `scripts/claude/claude-tmux.sh -C <repo dir>
start autopilot --permission-mode auto --model sonnet "/autopilot max 5 items"` starts a repo's autopilot; `/fleet` runs the loop.
