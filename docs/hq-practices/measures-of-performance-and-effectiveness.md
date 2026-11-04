---
summary: "Measure both: MOP (built right, pass/fail limits) and MOE (right thing, outcome achieved); for backlog items, hand-backs and operator views"
group: agents
tags: [agents, metrics, ux]
applies-to: [all]
reviewed: 2026-11-04
status: stable
---

# Measures of performance and effectiveness

Two questions, asked separately, because a green answer to one says nothing about the other.

| Measure | Question | Shape | Example in HQ |
| --- | --- | --- | --- |
| **MOP**, performance | Did we build or run it right? | Pass/fail limit on a quantity, from a "must" in the spec | gate green, p95 under 300 ms, report parsed without error |
| **MOE**, effectiveness | Did it achieve the outcome it was for? | Result in the real situation, judged against the need | operators fill the daily report with fewer corrections, coverage by 10:00 |
| **MOS**, suitability (optional) | Can it be kept running where it lives? | Reliability, maintainability, support | restore drill passes, upgrade is one command |

MOEs are decomposed into MOPs: an outcome is reached through performances. So the chain runs from an MOE down to the
MOPs that explain it, never the other way: a green MOP does not prove the MOE, and a red MOE points at MOPs to look at.

## The rule

- **Every backlog item names its outcome** (`outcome:` line in the item: what changes for whom, how it is measured,
  when it is checked) next to its gate. No outcome stated: say why (`outcome: none, hygiene`).
- **A hand-back adds one line** beside "Checked in": **Outcome measured:** `yes` (number, source) / `not yet` (check
  date) / `n/a`. A landing is "done" for MOP; it is "worked" only when the MOE is read.
- **A round table keeps a column** for landed items whose outcome is still unmeasured; a monthly pass reads them
  (usage after release, reopened items, rework, operator complaints) and closes or reopens each.
- **Operator views show MOE first** and the MOP under it as the explanation: "is coverage good today?" on top, "which
  parse failed" below, each red outcome linked to its causes.

## Why

- Gate, tests, landed counts and busy sessions are all MOP. A fleet can score perfectly on them while building the wrong
  thing (many landed items, no change in what users do).
- MOEs depend on the real environment, so they are noisier and slower; that is why they are named up front, with a
  check date, instead of being remembered later.
- A measure that nobody can act on is noise: each MOE needs an owner and a threshold at which something changes.

## In JAS Bot (operators)

See the backlog item for the design. Direction: the operator page answers MOE first (share of corps with a complete
daily report, missing reports resolved after a nudge, time from the deadline to full coverage, operator corrections per
report), then the MOP that explains a red value (received and sent on time, parse failures, retries, delivery errors).

## Drafting indicators when data is missing

An operator view is wanted before all its data exists (JAS Bot, 2026-11-04). Build it in layers instead of waiting:

- **An indicator registry first, a dashboard second**: one row per indicator with id, kind (MOP/MOE), one query over
  existing tables, unit, target, the operator question it answers and `data_status`: `available` (computable now),
  `partial` (part of the definition is stored), `needs-capture` (nothing stored yet). The page shows `available` ones
  and lists the rest as "not measured yet, needs X", so a gap is visible, not hidden.
- **Draft the queries from the schema** (a cheap reviewer such as Junie, read-only, [subagents-and-context.md](subagents-and-context.md#junie-cli)),
  then check every table and column name against the schema file and mark the drafts "not run against a database".
  Typical gaps found this way: no stored deadline (use the scheduler's time or a setting), an insert time that is not
  the message time, no per-report "wrong" mark, free-text run summaries that cannot be counted.
- **Validity rules are guesses until the operators confirm them**: what counts as "valid" (funnel order, a coordinate
  pattern) or "incomplete" (what a mark code means) is written as a question list, not shipped as a rule.
- **Capture what is missing now**, as small append-only events (event, subject, time), so history accumulates before
  the screen exists; targets are set after two weeks of real numbers.

Worked example: [the JAS Bot indicator draft](https://github.com/evocaslab/hq/blob/main/docs/reviews/2026-11-04-jasbot-mop-moe-indicators.md), nine indicators,
three `available`, four `partial`, two needing capture.

## Sources

- AcqNotes, "Measures of Effectiveness" and "Test and Evaluation: Measures of Effectiveness" (definitions of MOE, MOP
  and MOS; MOEs decompose into MOPs and MOSs): <https://acqnotes.com/acqnote/careerfields/se-measures-of-effectiveness>,
  <https://acqnotes.com/acqnote/careerfields/te-measures-of-effectiveness>
- Test and evaluation master plan (MOP as pass/fail limits tied to "shall" language; MOE as mission-level results):
  <https://en.wikipedia.org/wiki/Test_and_evaluation_master_plan>
- DAU glossary entries for MOE, MOP and MOS (archived index, wording not checked here):
  <https://webarchive.library.unt.edu/web/20170215181142/https://dap.dau.mil/glossary/pages/2189.aspx>
- Registry and `data_status` layering: observed in the JAS Bot indicator design (2026-11-04), no external source.
- Not checked: INCOSE's own definitions. The "right system / system right" phrasing is a common gloss on verification
  versus validation, not a source quote.
