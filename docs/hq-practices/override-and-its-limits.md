---
summary: "Override and its limits: what a maintainer override lifts (WIP, cap, seats, plan pace) and what it always keeps"
group: agents
tags: [agents, fleet, modes, safety]
applies-to: [all]
reviewed: 2026-11-12
status: draft
---

# Override and its limits

A maintainer who says "ignore the pace, work freely" wants more work to run, not fewer checks. The override is a pin
that lifts the limits which protect the plan's pace and the queue, and keeps the limits which protect the machine,
the data and the plan's hard edge. The modes are listed in
[autonomous-sessions.md](autonomous-sessions.md#operating-modes) and
[fleet-modes](https://github.com/evocaslab/hq/blob/main/docs/design/fleet-modes.md).

## Research first: why some limits must survive

- Safety interlocks are not bypassed by an operator's "go": a limit that prevents damage (out of memory, an exhausted
  quota, an unreadable sensor) stays, while a limit that only paces the work can be waived by the person who owns the
  work. The same split appears in "fail-safe" design and in privilege rules: a waiver covers policy, never invariants.
- A wrong waiver costs differently: a lifted pace costs plan budget the owner chose to spend; a lifted RAM limit kills
  running sessions and loses their work.
- An unknown reading is not a green reading: when the capacity or the plan usage cannot be read, the safe answer is to
  hold, whatever the mode.

## What an override lifts

| Limit | Meaning | Under override |
| --- | --- | --- |
| plan pace | the weekly or 6 h pace verdict (slow down = 0 slots) | a warning only |
| `wip` | unlanded branches count against the session cap | lifted |
| `cap` | the session cap (`MAX_SESSIONS`) | lifted |
| `seats` | reserved gate and class seats | lifted, shared seats may be used |
| model ceiling | the tier table | none (unlike `free`) |

## What an override keeps

- **RAM and swap.** The memory room for new sessions and its strain verdict.
- **Hard stops.** Week at 95 % or more, session at 90 % or more, under 1 h to the weekly reset, a full machine.
- **Unknown capacity or plan usage.** No reading, no start for code work.
- **Writer checks.** Two writers on the same paths, the claim lease and the landing lane stay in force: an override
  starts more work, it does not let two sessions edit the same files.
- **The end.** A pin has an end (default the end of the local day, never later than the weekly reset).

## Rules

1. Only the maintainer's words pin an override, and the pin records the reason and the request id.
2. A tool that applies the mode reads it once from the mode command, not from a second file, so every consumer
   (capacity, keeper, preflight, usage) lifts and keeps the same things.
3. When a limit is lifted, the verdict still prints it as an advisory line, so the maintainer sees what is being spent.
4. A new limit is classified when it is added: pacing (liftable) or invariant (kept). An unclassified limit is kept.

## Sources

- Fail-safe and safety interlocks (an operator override does not bypass the protective function):
  <https://en.wikipedia.org/wiki/Interlock_(engineering)>
- Principle of least privilege (a grant is scoped to the policy, not the invariants):
  <https://csrc.nist.gov/glossary/term/least_privilege>
- Observed in HQ (2026-11-12): `scripts/claude/mode/mode.py` (hard stops as an overlay) and `scripts/machine/capacity.py`
  (`mode_limits`, limits `plan`, `cap`, `wip`, `ram`, `hard_stop`); the lift/keep split is HQ's own design.
