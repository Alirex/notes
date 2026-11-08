---
summary: "Version the agent flow (briefs, hand-back block, retire checks, round): a semver file with a changelog, a per-session registry of role and version told vs seen, stale detection that briefs only the sessions behind"
group: agents
tags: [agents, supervision, workflow, versioning]
applies-to: [all]
reviewed: 2026-11-08
status: draft
---

# Flow versioning

## Rule

The rules a session follows (how it pulls work, what its hand-back shows, how it is retired, what the supervisor's
round checks) are one versioned artefact: a file with a semver-like `version` and a changelog, one entry per behaviour
change with its level. Every brief starts with the version it was written for and the session's role; every hand-back
echoes the version. A registry records per session who it is (role, model, manager, surface, machine) and which
version it was *told* and has *seen*. The supervisor briefs or restarts only the sessions that are **stale**: a major
or minor change lies between what they follow and the current version.

## Why

A session reads the rules once, at its start. A rule written after that reaches it only when someone sends it, and
"someone remembers" does not scale: with one global "last briefed" marker the supervisor either re-briefs everyone
(tokens, interruptions) or nobody (observed 2026-11-01: sessions kept a habit a rule had forbidden hours earlier). A
version per session turns "who is behind?" into a lookup and "what do I send?" into a changelog slice. The same number
on the hand-back makes the round table show, without a model turn, which sessions run an old flow, and a stale one can
be restarted instead of nudged when the gap is large (the maintainer allowed it).

## The version file

One file next to the digest it stamps (in HQ `.claude/flow-version.yaml`), machine-readable so the SessionStart hook,
the round table and the CLI read the same number:

- `version`: `MAJOR.MINOR.PATCH`.
- `files`: the flow files (digest, the fleet and preflight skills, the workflow practices). A commit that touches one
  without a bump is reported (`sessions-since.sh list`) and is the cue to bump.
- `changelog`, newest first; each entry `version`, `date`, `level`, `note` (the behaviour change, in one paragraph)
  and `files`.

Levels, from the running session's point of view:

| Level | Meaning | What the supervisor does |
| --- | --- | --- |
| major | a rule a session follows changed or went away | re-brief every running session (or restart it) |
| minor | additive: a new tool, command, line or step | brief the stale sessions with the slice; new sessions get it from the digest |
| patch | wording, links, examples | nothing to send |

Only the `bump` command writes the file (it validates order, dates, the top entry), so the version cannot drift from
the changelog.

## Told vs seen

Two facts per session, both with a time, in a registry file written atomically under a lock:

- **told**: a brief carried this version (a start brief, a message brief). Recorded by a hook on the message and
  start tools when the text contains `Flow: vX.Y.Z`, so the supervisor does not have to remember a bookkeeping step.
- **seen**: the session confirmed this version: the digest a SessionStart hook printed to it (recorded by that hook,
  for every new session and after every compaction), or the `Flow: vX.Y.Z` line of its hand-back (read by the round).

A session's known version is `seen`, else `told`. `told` without `seen` is a claim the supervisor made about the
session, not the session's state; the round prefers the hand-back line. One session may have several ids (a
transcript id and an app id): the registry joins them so one row shows both facts.

## The registry

Each record (`schema: 1`, every field has a default, readers ignore unknown fields, the writer keeps them):

| Field | Meaning |
| --- | --- |
| `role` | one of **supervisor** (starts and briefs sessions, lands their branches), **coordinator** (one repo's coordinator of builders), **worker** (builds one item or branch, reports back), **autopilot** (pulls a repo's backlog on its own), **reviewer** (read-only review by lenses), **researcher** (read-only research, a note as output), **builder-subagent** (an Agent-tool subagent editing in a worktree), **keeper** (a timer or no-LLM tick); `detail` is free text |
| `model`, `handle`, `repo`, `kind` | the model it runs on, its ledger handle, `hq` or `owner/repo`, `session` or `subagent` |
| `managed_by` | the supervisors or coordinators that briefed it, oldest first (several managers over a long life) |
| `surface`, `machine` | `desktop`, `tmux-cli`, `remote-control`, `cloud` or `subagent`; the machine id or hostname |
| `cwd`, `branch`, `pid`, `tmux` | where it works, when known |
| `started`, `last_seen` | first record; last confirmation |
| `flow_told`, `flow_told_at`, `flow_seen`, `flow_seen_at` | the two facts above |

Briefs carry `Role: <role>` next to `Flow: vX.Y.Z`, so the hook registers the role with the version; the hooks also
fill surface (a tmux environment, a desktop environment, a start from the desktop app), machine, cwd and branch. The
file is for other tools too (a fleet page in a web app, the round table): they read it, never write it.

## Stale detection

`status` lists every session with told, seen, what it is behind by (`1 major, 2 minor`) and `stale` (a major or minor
between its known version and the current one; a patch gap is not stale). `diff <from> [to]` prints the changelog
slice to put in the brief. The round table appends `flow vX.Y.Z` to each session line, `stale (...; restart to pick
up flow vY)` when behind, and counts the stale and unknown ones in its footer; a stale flag is a recommendation, the
restart stays the supervisor's call. The `/sessions-update` flow becomes: bump if flow files changed without one, list
the stale sessions, send each the slice (filtered to its work), and the hook records `told`; the global marker stays
as the fallback for rule changes that were never bumped.

## Sources

- Semantic Versioning 2.0.0, the meaning of major, minor and patch: <https://semver.org/>
- Keep a Changelog, one entry per change in human words, newest first: <https://keepachangelog.com/en/1.1.0/>
- Claude Code hooks (SessionStart input carries `session_id` and `cwd`; PostToolUse carries `tool_input` and
  `tool_response`): <https://code.claude.com/docs/en/hooks>
- Observed in HQ: [autonomous-sessions.md](autonomous-sessions.md#keep-running-sessions-current) (2026-11-01),
  [supervisor-preflight.md](supervisor-preflight.md); implementation `scripts/claude/flow/flow-version.py`
  (tests `test_flow_version.py`), `scripts/claude/preflight.sh digest`, `scripts/claude/sessions-since.sh`,
  `scripts/claude/fleet/fleet-round.py`.
