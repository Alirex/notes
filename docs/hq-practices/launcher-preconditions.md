---
summary: "A new rule on how sessions start must be proven on every caller of the launcher, including timers and systemd units, or the unattended path breaks first"
group: agents
tags: [agents, supervision, testing]
applies-to: [all]
reviewed: 2026-11-16
status: stable
---

# Launcher preconditions: prove every caller

A launcher (a script that starts a session, service or worker) gets stricter over time: a naming rule, a required
variable, a lock. The interactive caller is run at once and works. The unattended callers (a keeper restart, a timer,
the queue) run hours later, with no environment and nobody watching, and are the first to fail.

## Rules

- List the callers of a launcher in one place and update the list in the commit that adds a precondition.
- Give the launcher a dry mode that checks preconditions and starts nothing; run every caller against it in tests, with
  the caller's real environment (a bare systemd-like one), not a mocked `subprocess.run`.
- An unattended caller never crashes on a refused start: it writes an attention line with the launcher's message, so the
  failure reaches the maintainer's phone.
- "Another supervisor looks alive" is not proof for staying quiet while a restart failed in the last hour.
- A refusal names its blockers (which files, which requests, which owner); a repeated line with no names is not a signal.
- A stuck interactive dialog looks like a hung session: print the dialog text in the attention line before anyone
  restarts the pane.
- A probe, replay or dry run of a guard must have no side effects (a tripwire that a test can trip halts everything).

Case: [I-261116-fleet-session-lost-twice](https://github.com/evocaslab/hq/blob/main/docs/incidents/I-261116-fleet-session-lost-twice.md).

## Sources

- Incident note above (journal of `hq-fleet-keeper.service`, keeper log, commit 7fc22963).
- [guards.md](guards.md) and [autonomous-sessions.md](autonomous-sessions.md) for the keeper and guard design.
