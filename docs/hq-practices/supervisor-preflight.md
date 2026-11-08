---
summary: "A supervisor re-reads its standing rules before every start; a hook re-injects a digest after each compaction"
group: agents
tags: [agents, context, cost, supervision]
applies-to: [all]
reviewed: 2026-11-01
status: stable
---

# Supervisor preflight

## Rule

A session that starts other sessions, workers, workflows or subagents runs a preflight before each start: re-read the
rules from their source files, check the plan pace and machine capacity, set the cheapest model that does the job,
check the brief and the overlap with running sessions. In HQ this is the `/preflight` skill; its digest of standing
rules is re-injected by a hook, so the check does not rest on the supervisor remembering it.

## Why

Observed 2026-10-31 in one long HQ session (compacted several times): four sessions were started without
`just cc-usage` while the week stood at 88 % and "a bit fast", no brief named Sonnet for implementation, and the
subagent rules were re-read only after the maintainer asked. All rules existed on paper. A summary keeps the task,
not the standing rules, and a long context weighs recent turns over a rule read hours ago.

## Trigger options

| Option | Fires | Cost | Verdict |
| --- | --- | --- | --- |
| Rule in `CLAUDE.md` only | loaded at start, summarized away | 0 | what failed |
| `SessionStart` hook, matcher `compact` (verified: a headless `/compact` fires `SessionStart:compact`) (and startup, resume, clear) | after every compaction and new session; plain stdout becomes context | the digest, about 450 tokens | chosen: the drift came after compactions |
| `PreToolUse` hook on session-starting tools, `Workflow`, `Agent` | right before the start; `additionalContext` carries the live pace | about 80 tokens per start | chosen: the moment of the mistake |
| `PostCompact` / `PreCompact` hook that saves the summary's key facts | after or before compaction; no context output, side effects only | a file to keep in sync | not done: every fact a supervisor loses (ledger sessions with model and branch, claimed items, open requests of the session, stale flow sessions, pending branches, deadlines) already lives in a file or git and is re-derived by the `compact` digest (`preflight.sh digest`, at most 40 lines, no LLM); a saved summary would be a second copy that can go stale |
| Blocking `PreToolUse` (deny until `/preflight` ran) | session starts and `Workflow` | one retry when forgotten | chosen for `start_session`, `hand_off_to_session`, `Workflow`: `/preflight` ends with `preflight.sh mark`, valid 30 min; `Agent` stays a reminder (too frequent). The gate checks twice: the mark (30 min) and the machine's room (`capacity.py --json`, cached 60 s): `start_session` and `Workflow` are denied when room is 0 or memory is strained, because a mark younger than 30 min says nothing about the machine now (2026-11-04: a worker started inside the window, load 86, swap 44 GB). Every gate call shows the room line; unreadable capacity allows with a note |

Chosen: the first, third and fifth together, both in the project's `.claude/settings.json` (`scripts/claude/preflight.sh`).
User-level (`~/dotfiles/claude/settings.json`) would also cover supervisor sessions outside HQ; not done: the
script and digest live in HQ and are HQ-specific (asked in a `maintainer-question`). Hook text is worded as facts, not commands: a system-style imperative can trigger the
model's prompt-injection defenses.

## One source of truth

`.claude/preflight-digest.md` is a list of one-line pointers, each `[path § Heading]` to the page that holds the
rule. `scripts/claude/preflight.sh check` (prek hook `preflight-digest`) fails when a referenced file or heading is
gone, so a rename cannot leave the digest pointing at nothing. Change a rule in its page, never in the digest.

## Other rules supervisors forget

Push every commit right away; pull main after landing; a backlog item for every unfinished request; questions as
`maintainer-question` issues and manual steps as `manual-action` issues; link every session mentioned; never poll;
tell the maintainer which subagents ran. They are in the digest. Two more from 2026-11-01:

- A session start caps `model` at the supervisor's own model; a higher tier runs only as an Agent-tool subagent. Check
  the session's real model in its note ([subagents-and-context.md](subagents-and-context.md#pick-the-cheapest-tier-that-works)).
- After hours a supervisor compacts and forgets its pending branches and deadlines: it keeps a ledger file and
  re-reads it after each compaction ([autonomous-sessions.md](autonomous-sessions.md#run-several-repos-at-once-the-fleet-supervisor)).
  The `compact`/`resume` digest prints that state itself (R-2611080415-196a): live ledger sessions with repo, branch and
  model, claimed items, branches not yet landed (`fleet-round --offline --brief`), this session's open requests (the
  count and the newest four, with the `docs-requests check --session` command), the first line of
  `cc-flow status --stale`, and the deadlines. Baseline before this change (`just cc-adherence`, 7 days, 104 compacted
  sessions, rules preflight-before-start, model-set, no-sleep-loops): 0.64 misses per active hour before a
  compaction, 0.62 after, miss rate 0.005 before and 0.008 after; measure again after a week.

## Sources

- Hooks reference (SessionStart matchers `startup|resume|clear|compact`, plain stdout as context; PreToolUse
  `hookSpecificOutput.additionalContext`; PostCompact has no decision control): <https://code.claude.com/docs/en/hooks>
- Skills: <https://code.claude.com/docs/en/skills>
- Observed in HQ (2026-10-31 session); see [subagents-and-context.md](subagents-and-context.md) and
  [autonomous-sessions.md](autonomous-sessions.md).
