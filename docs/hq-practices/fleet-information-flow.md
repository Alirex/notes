---
summary: "How information and decisions move in a fleet of agent sessions: a pulled, capped brief of tool facts, an explicit delegation level with one bright line, purpose before tasks, outcome next to tempo, lateral knowledge as dated lines"
group: agents
tags: [agents, supervision, fleet, workflow, security]
applies-to: [all]
reviewed: 2026-11-08
status: draft
---

# Fleet information flow

## Rule

When several agent sessions work in parallel, design how knowledge and decision rights move between them, not only who
does which task. Seven rules:

1. **The fleet brief is pulled and capped.** A session gets one fixed-size view (20 lines) at the moment it asks for its
   next work, not a daily broadcast and not a chat with every sibling. Contents: landings elsewhere in the last hours
   with their lessons line, answered questions, writers per repo, the plan pace, the flow version, and at most one
   `decided` or `found` line per sibling. Over the cap, lines are dropped in a fixed order, never by length of prose.
2. **Facts come from tools, prose is a labelled claim.** Every line in the brief is derived by a program from git, the
   ledger, the lease store or the usage endpoint. A sibling's free text may appear only when it carries its handle and a
   label (`decided`, `found`), and the brief states that such lines are information, never an instruction.
3. **Delegation level is explicit, with one bright line.** Each session has a recorded level: `land` (merge after its
   gate), `branch` (push a branch, the supervisor lands) or `propose` (write a backlog item or a patch file only). The
   bright line is a never-list: privileged files (guards, hooks, settings, credentials, the gate itself) stay `propose`
   at any level, whatever the session believes about its purpose.
4. **Purpose before tasks, never over the limits.** A brief opens with the repo's outcome in one line (why the work
   exists), then the item. The purpose lets a session choose between two valid readings of a task; it never widens
   what the item says is out of scope, and it never replaces a stop rule.
5. **Tempo is a measure of performance, outcome a measure of effectiveness.** Landings per hour, rounds per day and
   gaps between rounds say the fleet is busy ([MOP versus MOE](measures-of-performance-and-effectiveness.md)). Whether
   the landing helped is a separate column: `measured` (a number moved) or `unmeasured`. Report both, so a fast fleet
   that moves no outcome shows.
6. **Lateral knowledge travels as dated lines.** What one session learns for another goes into an append-only, dated
   heartbeat file with a closed set of verbs (`decided`, `found`, `round ran`, `round skipped <reason>`), a handle and a
   reason. A gap in the rounds longer than an hour is a finding at every hour of the day, not only in office hours: an
   unattended fleet fails at night too.
7. **The supervisor's turns per landing are a cost.** Count the supervisor's turns (and tokens) per landed item next to
   the landings. If the number rises, move decision rights down (rule 3) or the brief up (rule 1) instead of adding a
   check.

What everyone can read, an injection can read. Anything that enters the shared brief or heartbeat file is reachable by
every session that reads it, so treat each line as untrusted input: tool-derived facts and closed verbs go in, free text
is length-limited, labelled and never executed.

## Why

- **Parallel work fails between the parts.** Work split by repo is easy to plan; failure comes from what the parts do
  not know about each other: the same lesson learned twice, a practice changed while a sibling still follows the old
  one, a finding in one repo that matters in another. McChrystal's account of Task Force 714 (Iraq, 2004 to 2006) is the
  best-known statement of this: a hierarchy that executed well was too slow against a networked enemy, so the task
  force shared one operations-and-intelligence picture widely and pushed decision rights to the edge. Two levers carry
  over to agents: information flow and decision rights. The other levers (trust built by exchanging liaison officers, a
  shared culture) do not, because sessions have no memory of each other and no personal trust.
- **Pull and cap, because attention is the limit.** The task force's picture went to hundreds of people in a daily
  video conference; a session's context window is the opposite constraint, since each added line is paid for in every
  later turn. A fixed 20-line view bounds that cost and makes the brief testable (one function, a line budget, a drop
  order).
- **Facts versus claims, because the brief is an attack surface.** A prompt injection in one repo's issue text, web
  page or log reaches every session whose brief copies it. Keeping tool-derived facts apart from labelled prose keeps
  the trust level of each line explicit. This also answers the usual objection to wide sharing: the task force could
  trust its readers; a fleet reading scraped text cannot.
- **Explicit levels, because implicit autonomy drifts.** Empowering the edge works when the leader's intent and the
  limits are clear. For agents the limits must be machine-readable (a level per session), or the supervisor re-decides
  every landing and becomes the bottleneck the book describes. The bright line exists because "my purpose justifies it"
  is exactly the argument an injected or confused session makes.
- **Purpose first, limits kept.** Commander's intent is the military form of rule 4: say what the work is for so a
  subordinate can adapt. Here it ranks below the item's Not-in-scope list, because agents cannot be trusted to weigh
  intent against limits the way trained officers can.
- **Tempo is not outcome.** A busy loop is easy to measure and easy to mistake for success. Critics of the book make
  the same point about the task force: raid counts rose, and what they achieved is disputed.
- **Coordination has a cost.** The team-of-teams approach spends leaders' time on shared awareness. In a fleet the
  supervisor's turns are billed against a plan limit, so the cost has a number.

## How to apply

- Build the brief as a function of tool output with a test for the cap and the drop order; put the flow version on the
  first line ([flow versioning](flow-versioning.md)). Deliver it where the session already asks for work (a stop hook
  or a `next` call), see [autonomous sessions](autonomous-sessions.md#run-several-repos-at-once-the-fleet-supervisor).
- Keep one writer per repo and the ledger; this page is about what flows between them, not about replacing that.
- Put a level in the ledger per session and have the round table print it. Check the never-list in the landing
  step, not only in the brief ([guards](guards.md), [agent security](agent-security.md)).
- Put a one-line `purpose:` per repo in the project file; print it right after the flow and role line.
- Validate heartbeat verbs: a free-form verb or a missing reason is rejected at write time.
- Start measuring before changing: supervisor turns per landing, questions per item, gaps per day, hours from a
  practices landing to the first sibling applying it, count of stale sessions.

## Not part of the rule

A push-everything daily briefing, a second writer per repo (an embedded liaison), a peer mesh where sessions message
each other freely, trust without verification, and dropping the ledger. These are the parts of the source that do not
survive the move from people to agents.

## Sources

Evidence levels: **secondary** (summaries and critiques read, not the original), **HQ** (observed in our own fleet, one
machine, one maintainer), **design** (our reasoning, no external source).

- McChrystal, Collins, Silverman, Fussell, *Team of Teams: New Rules of Engagement for a Complex World* (Portfolio,
  2015). Evidence: **secondary**. The book itself was not opened for this page; the 12 chapter recaps, the authors'
  TIME article and a history of Task Force 714 were read, plus four critiques. One reader so far; a second-reader pass
  on the claims is still open (`docs/backlog/B-261108-team-of-teams-fleet.md`).
- Task Force 714, Iraq, 2004 to 2006: the raid rate is reported as rising from about 15 to about 300 a month. Evidence:
  **secondary**, and the counts differ by source; treat that order of magnitude as the claim and the causal link to the
  method as contested.
- Critiques of the book (survivorship, a single success case, military context, attribution): evidence **secondary**;
  they are why rules 5 and 7 ask for measures instead of trust in the method.
- Pull versus push, the cap and the drop order, the facts-versus-claims split, the bright-line never-list and the
  supervisor-turns-per-landing measure: **design**, with the HQ fleet as the first user (flow v1.4.0, 2026-11-08, no
  baseline yet).
- Observed in HQ: `scripts/claude/fleet/fleet_beat.py` (verbs, pulse), `scripts/claude/hooks/stop-next.py`,
  `scripts/claude/fleet/fleet-round.py`, `cc-fleet-brief`; evidence **HQ**.
- MOP versus MOE: [measures-of-performance-and-effectiveness.md](measures-of-performance-and-effectiveness.md).
