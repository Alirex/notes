---
summary: "After a sleep, network outage or reset: detect the cause, wait for the API, wake sessions, check unfinished work, clean orphans safely"
group: agents
tags: [agents, machine]
applies-to: [all]
reviewed: 2026-11-09
status: draft
---

# Heal after an interruption

A laptop sleep, an internet outage or a machine reset leaves sessions mid-turn, background jobs dead, locks held by
nothing and half-finished work. `/heal` ([.claude/skills/heal/SKILL.md](https://github.com/evocaslab/hq/blob/main/.claude/skills/heal/SKILL.md)) is the
procedure; this page is the reasoning. Related: [autonomous-sessions.md](autonomous-sessions.md),
[loose-ends skill](https://github.com/evocaslab/hq/blob/main/.claude/skills/heal/SKILL.md#4-unfinished-work).

## Detect the cause by signal

| Cause | Signal |
| --- | --- |
| Sleep/resume | `journalctl -k` line `PM: suspend exit`; logind `PrepareForSleep(false)`; a gap between keeper ticks of 15 min or more (the keeper notes it) |
| Outage | the API probe fails (`curl` to `api.anthropic.com`) while the machine was up; `network-online.target` only covers boot, so probe, never trust it |
| Reset/reboot | uptime shorter than the last tick's age; tmux server gone; `/tmp` files and the heavy-lock holder missing |

Never guess: say which signal was seen. A sleep and an outage can be one event.

## Order

1. **Wait for the API** with a bounded probe and backoff (not a sleep loop: a background command with a completion notice).
2. **Wake sessions.** A Claude Code turn retries a failed request up to 10 times and keeps completed content; a hang
   after a network change can last minutes. Remedy per session: `Esc`, then `continue`; if that fails, restart with
   `claude --resume <id|name>`. Desktop sessions: `/sessions-continue`. tmux autopilots: type the resume prompt into the pane
   (the keeper's `feed_panes` does it for idle ones); a dead pane is respawned from the registry (name, id, folder),
   because tmux-resurrect restores only a conservative program list.
3. **Unfinished work**: `/loose-ends` (git health first after a reset: `git fsck`, empty objects).
4. **Orphans and stale locks**, only after proof: re-read `/proc/<pid>/cmdline` and the start time (field 22 of
   `/proc/<pid>/stat`) before a kill, so a reused pid is never killed; remove a git `*.lock` only when no git process
   runs and the lock is old; `just machine-locks check` then `fix` handles the heavy lock and queue tickets.
5. **Backoff and a circuit breaker**: retries double their wait; after a few failures stop and report, never loop
   (systemd `Restart=`, `RestartSteps=`, `StartLimit*` are the model).
6. **One report table**: session, cause, action, state.

## Boot without network

A service that fetches something at start (uv resolving packages, a model pull) fails when DNS is down at login, and
with a short `Restart=` loop it hits `StartLimitBurst` within a minute and stays `failed` until someone runs
`systemctl --user reset-failed`. `After=network-online.target` does not help a **user** unit: the user manager has no
such target (it shows `not-found`). So give such units a retry budget that outlasts a typical outage: `Restart=on-failure`,
`RestartSec=30`, `StartLimitIntervalSec=900`, `StartLimitBurst=20` (about 10 minutes of tries, then stop and report).
HQ renders this from `boot_retry: true` in `infra/services.yaml`. After an outage, `systemctl --user --failed` lists
the units to `reset-failed` and `start`.

## Never

Kill by name or by a remembered pid; remove a lock a live process may hold; restart a session that waits on the
maintainer; re-run a long gate that is still running; poll in a shell.

## Before the sleep

`just cc-prepare-sleep` (plan) and `--apply` (writes `resume.md` next to the fleet ledger, pauses the queue with reason
`prepare-sleep`, logs `stopped`) are the manual command; `/heal` reads `resume.md` first. It never commits, messages or
kills; saving WIP stays `just cc-power-pause --apply` (private repos only).

Possible automatic hook, **untried** (system settings are the maintainer's). Delay the sleep a few seconds and run the plan
first:

```bash
systemd-inhibit --what=sleep --who=hq --why="prepare-sleep" --mode=delay ~/dev/hq/scripts/claude/prepare-sleep.py --apply
```

or a unit ordered before `sleep.target`:

```ini
# /etc/systemd/system/hq-prepare-sleep.service
[Unit]
Description=HQ prepare for sleep
Before=sleep.target
StopWhenUnneeded=yes
[Service]
Type=oneshot
RemainAfterExit=yes
User=alirex
ExecStart=/home/alirex/dev/hq/scripts/claude/prepare-sleep.py --apply
[Install]
WantedBy=sleep.target
```

A real suspend/resume try (lid close, wake, `/heal` reads `resume.md`) is still to be recorded here.

## Known gaps

Filed in [B-261108-heal-gaps](https://github.com/evocaslab/hq/blob/main/docs/backlog/B-261108-heal-gaps.md): persistent keeper timers, a resume hook, a
connectivity probe, boot-time restart of tmux autopilots, resuming queue jobs after a reset, `/tmp/gate-*.log` cleanup, unlocking proven-orphan agent worktrees (`machine-locks check --only worktree` reports them).

## Sources

Read 2026-11-08:

- systemd `Restart=`, `RestartSteps=`, `RestartMaxDelaySec=`, `StartLimitIntervalSec=`: <https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html>
- systemd sleep hooks and `PrepareForSleep`: <https://www.freedesktop.org/software/systemd/man/latest/systemd-suspend.service.html>, <https://www.freedesktop.org/software/systemd/man/latest/org.freedesktop.login1.html>
- `network-online.target` is a boot concept: <https://systemd.io/NETWORK_ONLINE/>
- Claude Code retries and resume: <https://code.claude.com/docs/en/troubleshooting>, <https://code.claude.com/docs/en/cli-reference>
- Observed in HQ: the 2026-11-08 internet outage (keeper note "woke", report-bot pane resumed by hand); hang reports on
  GitHub (anthropics/claude-code issues about network changes) not re-verified.
