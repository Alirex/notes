---
summary: "Finding why something went wrong or runs slow, idle or wasteful: a timeline from logs, sized contributing factors with evidence per line, the constraint, fixes ranked by strength, automation of the largest factor and a re-check on the symptom"
group: quality
tags: [incidents, agents, evidence, workflow]
applies-to: [all]
reviewed: 2026-11-16
status: draft
---

# Root cause analysis: factors, not one cause

When work stalls, a gate fails the same way twice, a guard prompt repeats or a landing goes bad, "why?" deserves
more than a guess and less than a ceremony. This page is the method. It extends the post-incident note in
[observability-and-incidents.md](https://github.com/evocaslab/hq/blob/main/practices/observability-and-incidents.md#incidents) (services) and the Retro stage in
[agent-retire.md](agent-retire.md#periodic-self-analysis) (the agent flow).

"5 Whys" is the well-known tool for this. Keep its habit (do not stop at the first answer, look at the real thing),
drop its shape (one chain, five steps, one root cause).

## The rule

| # | Rule | Why |
| --- | --- | --- |
| 1 | **Triggers are fixed in advance**: ready work and nothing running for over an hour; the same failure, denial or prompt three times; a landing reverted or main red; a claim that did not match git; lost work; any "why did X not happen?" from the maintainer. A single red gate is normal work, not an analysis. | Criteria set before the event keep the analysis from depending on mood [5] |
| 2 | **Timeline first, from records**: logs, ledgers, `git log`, hook output, usage snapshots, with timestamps. Start at the first trigger, not at the moment someone noticed; explain every long gap. | The gap between trigger and detection is itself a finding [7] |
| 3 | **Factors as a branching list, never one chain.** Each "why" may have several answers; keep all of them. Expect several independent factors that had to coincide. | A single chain tends "to simplify complex problems and limit understanding of how processes fail" [2]; assuming one root cause leads to fixing symptoms and "having the same accident over and over" [4] |
| 4 | **Every line cites evidence**: a log line, `file:line`, or a command and its output. A line without evidence is labelled `hypothesis` and gets a check or stays labelled. No factor from memory or from "it probably". | Each "why" has many possible answers and the analyst picks one, guided by what they already look for [3]; models attributing a multi-agent failure named the right agent in about half the cases and the right step in 14 % [9] |
| 5 | **Ask "how", not only "why"; never stop at an actor.** "The agent forgot", "the supervisor did not check" and "the maintainer was away" are starting points: what did that actor see at the time, which rule, prompt, default or tool output made the action reasonable? Stop a branch at something that can be changed (a guard, a script, a default, a rule's wording), or mark it `accepted` with a reason. | "Why" drifts to "who" [3]; "should have, could have" is hindsight and explains nothing [4] [6]; when the answer is human error, ask why the error was possible [7] |
| 6 | **Three questions per factor**: what made it possible, what would have shown it sooner, what would have recovered from it without a person. | Detection and recovery are separate fixes from prevention [5] [7] |
| 7 | **Fixes are ranked by strength**: `eliminate` (the situation can no longer arise) > `automate` (a guard, hook or script acts) > `detect` (a check, alert or status line shows it) > `document` (a practice sentence, a skill line). Every factor gets a fix or `accepted`. An analysis whose fixes are all `document` says why nothing stronger fits. | Controls that need no ongoing human effort are the effective ones [8]; changing behaviour is less reliable than changing automated systems [5] |
| 8 | **Every fix is a backlog item at once**, with the strength in its text, and the note links it. At the next Retro: did the symptom return? If so, the analysis is reopened, not repeated from zero. | "A postmortem without subsequent action is indistinguishable from no postmortem" [5]; an analysis needs feedback on whether its changes were effective [4] |
| 9 | **No depth number.** Three levels or seven: a branch ends by rule 5, not by counting. | The "five" was how many the original example took [1] |
| 10 | **Size the loss and every factor in the symptom's own unit** (idle session-hours, minutes waited, landings not made, plan percent), each marked `measured` (a record), `estimated` (how derived) or `unknown`. A count of things (rows, worktrees, panes) is not a size until it is tied to the symptom. | Without a unit, the fix goes to the factor that is easiest to count or to prove; toil work starts by choosing a unit of human effort and tracking it "before, during, and after" [11] |
| 11 | **The first build is for the largest measured factor**, at the strongest strength that fits, and it lands in the same piece of work when it is size s or m. A hypothesis that looks largest gets a measurement first, not a build. | At least one action on the root cause is the minimum a postmortem owes [13]; time saved must be "proportional to the time invested" [11] |
| 12 | **The re-check signal is a command, a number and a threshold on the symptom**, not on the fix ("idle sessions with pickable work: 0 on three daily readings", not "the sweep ran"). A signal that will be read more than twice becomes a guard: a timer check, a status line or a hook that fails. | Activity is not effect ([continuous-improvement.md](continuous-improvement.md#assess-the-effect-not-the-activity)); a manual re-check is itself toil [10] |

Blameless applies to agents as to people: an agent that "ignored a rule" is a finding about where the rule lives,
how long the context was, and what the guard allowed.

## The template

Small case (one or two factors, one fix): a `## Causes` section in the backlog item that fixes it. Larger (three or
more factors, or fixes across several items): a note `docs/incidents/I-YYMMDD-<slug>.md` in the repo it concerns, the
same folder and id the service incidents use.

```text
# <symptom, as observed, one line> (<date>)
Trigger: <which trigger from rule 1>. Impact: <what was lost, as a number with its unit and measured | estimated>.
Request: <R-id or item>. Analysed by: <handle, model>. Confidence: high | medium | low.
Cause: <tag, tag>   (kebab-case class of the cause; a tag in two notes is a repeat)

## Timeline (UTC or one zone, from records)
- <time> <event> - <source: log path, command, sha>
- <time> first trigger ... <time> first noticed (gap: <duration>, why)

## Contributing factors (a branching list; every line has evidence or says hypothesis)
1. <factor> (<size in the Impact unit>, measured | estimated | unknown) - <evidence>
   - possible because: <deeper factor> - <evidence>
     - possible because: <something changeable> - <evidence>
   - not seen sooner because: <...> - <evidence | hypothesis>
   - not recovered automatically because: <...> - <evidence>
2. <independent factor> - <evidence>
How they combined: <two lines: which factors had to coincide>
Constraint: <for a slow or idle symptom: the one step with a queue before it and idle capacity after it, and its rate>

## What worked (kept the damage small)

## Fixes
| Factor | Fix | Strength | Item |
| --- | --- | --- | --- |
| 1 | <...> | eliminate / automate / detect / document | B-... |
| 2 | accepted: <reason> | - | - |

## Not checked
<records not read, hypotheses left open>

## Re-check: <date> (<command>: <number> <threshold>, on the symptom; baseline today: <number>)
```

The `Cause:` tag classes the factors; `just docs-incident groups` lists notes per tag and marks a tag seen in two or
more notes as `REPEATED`: the Retro turns each repeat into one backlog item (the same cause twice means the first fix
was too weak) and a note's factors are a source for generated backlog items.

Fifteen minutes for a small case, an hour for a note. Longer means the records are missing: that is the first
factor, and its fix is a `detect` item.

## Where it fits

| Situation | Starts from | Records to read first |
| --- | --- | --- |
| Work stalled, sessions idle | the `cc-why-idle` screen (the current blockers), then this page for how they arose | usage snapshots, the fleet ledger, capacity history, session last turns |
| A gate fails the same way again | the gate log | `just machine-metrics runs --steps`, the diff, earlier runs |
| A guard prompt or denial repeats | `/guard-care` | the guard's ask and deny log |
| After sleep, an outage or a reset | `/heal` (cause by signal first) | the keeper's notes, journals |
| A bad landing, a claim that did not match git | the landing check | `git log`, `git ls-remote`, the hand-back |
| A service incident | [observability-and-incidents.md](https://github.com/evocaslab/hq/blob/main/practices/observability-and-incidents.md#incidents) | the runbook, service logs |

A status command answers "what blocks now"; this method answers "how did we get here". The weekly Retro reads the
open notes' Re-check lines.

The shape, from a stall on 2026-11-11 (factor 1 is measured in its backlog item; the others stay hypotheses until
cited): (1) a usage reading 114 minutes old produced "HOLD" at 42 % used, possible because many callers polled a
rate-limited endpoint; (2) idle sessions with work left were not nudged; (3) landings refused for lack of gate room
were never retried; (4) branches sharing generated files landed one at a time. One chain would have ended at
factor 1 and left three in place.

## Slow, idle or wasteful: from factors to automation

An event (a bad landing, lost work) has a moment; "the fleet is mostly idle" or "the gate is slow" is a rate. The
rules above still hold; rules 10 to 12 and the constraint line are what a rate adds. The `/rca` skill runs this loop.

### The checklist

1. **Symptom and unit.** One line as observed, the unit, today's number and where it comes from. No number: the
   first fix is the record that gives one (a `detect` item), and the analysis says `unknown`.
2. **Same symptom before?** `just docs-incident list`, then search the notes for the symptom, not only for the
   `Cause:` tag. Found: reopen that note and ask why its fixes did not hold.
3. **Timeline and factors** by rules 2 to 6, each factor with a size (rule 10).
4. **Constraint.** Walk the flow from picked item to landed commit and find the step with a queue before it and
   idle capacity after it. Give its rate (landings per day, gates per hour, sends per round).
5. **Pick one factor**: the largest measured one at or before the constraint. Say why the next is smaller.
6. **Fix by strength** (rule 7): can the step go away? If not, automate it; build it now when small (rule 11).
7. **Re-check signal** on the symptom (rule 12), with today's baseline, and a guard when it will be read again.
8. **Record**: the note, `just docs-incident check`, one item per remaining fix, the lesson line.
9. **At the re-check date**: read the signal. Held: status `verified`. Returned: reopen, do not start from zero.

### What flow theory changes here

- **Busy is not the goal; landed work is.** A system's output is set by one constraint; time saved elsewhere changes
  little, and the other steps are subordinated to it [14]. So an idle session is a symptom to read, not a number to
  drive to zero: starting more sessions while landing is the constraint only adds unlanded branches.
- **Lead time = work in progress / throughput** in a steady state, whatever the distributions [15]. More branches in
  flight at the same landing rate means each waits longer, conflicts more and is rebased more. A cap on unlanded work
  is therefore right, and the fix for a full cap is help for the lane, not a higher cap.
- **A serial resource needs slack.** Queue length grows much faster than utilisation as a single server nears full
  use; the popular "past 80 %" curve is a simplification with no time unit [16], so use the shape, not the number.
  The gate seats and the lander are such servers: never plan them at 100 %.
- **Toil is the test for "automate"**: manual, repetitive, automatable, with no lasting value, growing with the
  size of the system [10]. The first option is to reject the task, the second to change the system so it needs no
  action; a script that copies the human steps is third [11] [12]. Automation that halts and says why is safer
  than automation that guesses [12].
- **A budget turns a mood into a trigger.** An error budget stops other work when it is spent and makes the analysis
  mandatory [13]. Here: the re-check threshold of the last note is the budget; a reading over it reopens the note
  and puts its largest fix ahead of new work.

### Specific to agent fleets

- **Who can act is a serial resource too.** One sender with the desktop tools, one lander, one plan quota, one
  maintainer: agents wait on these far more than on CPU. List them when looking for the constraint.
- **Every wait has an owner and an expiry.** A pause, a "waiting" heartbeat, an answered question and a lock are
  all waits; one without an expiry or an owner who is woken becomes the stall.
- **Agents retry.** A failure a person would retry once, an agent fleet retries until the lane is full; a landing
  needs an explicit attempt budget and then an item, not another try [17] (a practitioner's argument, not a study).
- **Noise hides blockers.** Stale rows in a status screen are a factor in their own right: they raise the time to
  see the real blocker. They are rarely the constraint.

### Worked examples (the same symptom, three notes in six days)

| | [I-261111](https://github.com/evocaslab/hq/blob/main/docs/incidents/I-261111-work-did-not-continue.md) | [I-261115](https://github.com/evocaslab/hq/blob/main/docs/incidents/I-261115-session-limits-static-and-opaque.md) | [I-261116](https://github.com/evocaslab/hq/blob/main/docs/incidents/I-261116-stale-idle-sessions.md) |
| --- | --- | --- | --- |
| Symptom | 13 to 19 sessions idle, 700 items ready | a machine with 50 GB free took no work; 839 pickable | idle sessions with 648 pickable items |
| Loss | about 7 hours (measured from ticks) | one machine-day; 14 % of the plan projected unused | "hours not measured" |
| Measured factors | usage reading 114 min old gave HOLD at 42 % used; ships failed "no room" and stayed failed; 117 branch pairs sharing files | seat total 6 against measured room 48; 19 idle sessions counted as seats | 27 finished worktrees, 20 of 28 blocked rows stale, 22 sessions flagged idle with work, 2 answered questions |
| Guessed (labelled) | no inbox; the age of the reading was not shown | why the numbers were hand-set; where the count is read | a `slow_down` wait that nothing wakes |
| Constraint | admission (the HOLD), then the landing lane (`room=false` on every tick) | the plan slot (`limits.plan: 1`), shown as "full (19 sessions)" | not named; factor 3 (no sender, a paused queue) is the one on the path of the symptom |
| Automation built | only the timer polls usage (eliminate); merge driver; Stop-hook nudge | adaptive seats (eliminate), on a branch | `repos-sweep --idle-locks`; `ship_proof` without a sidecar (28 blocked rows became 8) |
| Re-check signal | empty | placement names a binding reason | blocked rows at most 10, REMOVE rows at most 5 |

What the three show together:

- **The automation followed what could be counted and proved.** In I-261116 the two landed fixes remove noise
  (factors 1 and 2) and uncover the 8 real blockers; the factor that carries the symptom (3) has three open items,
  and the re-check signal does not mention idle sessions. Later the same day `cc-why-idle` still said `blocked by:
  pauses`, with the two paused queue kinds the note names. Rules 10 and 12 are for this.
- **Three fixes in five hours cannot be told apart.** I-261111 lists "whether the nudge ended the plateau" as not
  checked. One signal per fix, or one fix at a time, when the fixes are not urgent.
- **Naming the binding limit is half the analysis.** I-261115 found that the screen named a session count while the
  plan slot bound; every refusal that names its limit saves the next timeline.
- **A repeat hides behind new tags.** The three notes carry 13 cause tags and share none, so
  `just docs-incident groups` does not join them, although the symptom is the same. Step 2 searches by symptom.

### Who does what

| Question | Goes to |
| --- | --- |
| "Why is it slow, idle or wasteful?" (cause unknown, several steps involved) | `/rca`, this page |
| "What is the largest cost in this one area?" (gate, tests, GPU, memory, plan; the measure exists) | `/optimize`, [continuous-improvement.md](continuous-improvement.md#the-loop) |
| "Which of the open items first?" (weekly, across the backlog) | `/improvement-review` |
| "What blocks right now?" | `just cc-why-idle`, `just machine-capacity` |
| After a sleep, an outage or a reset | `/heal` |

`/rca` writes the note and builds one fix; its other fixes are items that `/improvement-review` ranks. `/optimize`
that finds a cost with no single owner (the wait is between steps) hands over to `/rca` and does not guess.

## What 5 Whys is good for

A quick first pass on a simple, mechanical fault in one component, and the habit: the first answer is a symptom; go
and see the real thing [1]. Its critics
grant it teaching value and no more [2]. Use it as one branch of the list above, under the evidence rule.

## Pitfalls

- Stopping at the first factor that has an easy fix.
- A tidy story written after the fact: the timeline is evidence, the narrative is not.
- Filing a model's answer to "why did this fail": it is a list of hypotheses to check [9].
- Fixes that are all "add a line to AGENTS.md": the weakest class.

## Studied and not taken

- **5 Whys as the method** (single chain, fixed depth): kept as a first-pass habit only.
- **Fishbone (Ishikawa) diagram**: a drawing is not diffable or greppable; the nested list carries the same
  branching. Reopen when a group analyses together at a board.
- **Fault tree analysis**: its probabilities need failure rates this setup does not have; the AND idea is kept as
  the "how they combined" line. Reopen for a safety or money-critical service.
- **CAST / STAMP**: models the whole control structure; the most thorough method studied and, by own estimate, hours
  to days per case. Taken: missing feedback as a factor, no stopping at an actor. Reopen for a data-loss or
  security incident.
- **Facilitated debriefings and Learning Teams**: built for groups of people recalling an event; here transcripts
  are the record. Taken: "how" questions.
- **Kepner-Tregoe problem analysis**: proprietary material, not opened. Reopen if a fault resists the timeline.
- **Amazon COE as a full document**: its detection and mitigation questions are taken (rule 6), its review process
  is not.
- **A model that writes the analysis**: about 53 % agent-level and 14 % step-level accuracy in the one benchmark
  read [9]. Reopen when a benchmark shows it usable.

## Sources

Research note with evidence levels and what was not opened:
[2026-11-11-five-whys-and-alternatives.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-11-five-whys-and-alternatives.md).

1 to 9 read 2026-11-11; 10 to 17 read 2026-11-16 for the section on slow, idle or wasteful systems.

1. Ohno's "Toyota Production System" (1988) and Toyota's own page were not opened; quoted through Mark Graban,
   "Even Ohno's Classic '5 Whys' Example Deserves Another Why": <https://www.leanblog.org/2026/04/ohno-5-whys-actually-seven/>
2. Alan J. Card, "The problem with '5 whys'", BMJ Quality & Safety 26(8), 2017, doi:10.1136/bmjqs-2016-005849; full
   text not opened (403), read through the AHRQ PSNet summary: <https://psnet.ahrq.gov/issue/problem-5-whys>
3. John Allspaw, "The Infinite Hows (or, the Dangers of the Five Whys)":
   <https://www.kitchensoap.com/2014/11/14/the-infinite-hows-or-the-dangers-of-the-five-whys/>
4. Nancy Leveson, MIT 16.63J lecture notes, "Analyzing Accidents and Incidents with CAST" (2016):
   <https://ocw.mit.edu/courses/16-63j-system-safety-spring-2016/d2501c7b1ce13719f38812e6d249dbee_MIT16_63JS16_LecNotes6.pdf>
5. Google, SRE Book and SRE Workbook, "Postmortem Culture": <https://sre.google/sre-book/postmortem-culture/>,
   <https://sre.google/workbook/postmortem-culture/>
6. Allspaw, Evans, Schauenberg, "Debriefing Facilitation Guide" (Etsy, 2016):
   <https://extfiles.etsy.com/DebriefingFacilitationGuide.pdf>
7. AWS, "Why you should develop a correction of error (COE)":
   <https://aws.amazon.com/blogs/mt/why-you-should-develop-a-correction-of-error-coe/>
8. NIOSH, "About Hierarchy of Controls": <https://www.cdc.gov/niosh/hierarchy-of-controls/about/index.html>
9. Zhang et al., "Which Agent Causes Task Failures and When?", arXiv 2505.00212 (2025), abstract only:
   <https://arxiv.org/abs/2505.00212>
10. Google, SRE Book, "Eliminating Toil" (the definition and its six attributes):
    <https://sre.google/sre-book/eliminating-toil/>
11. Google, SRE Workbook, "Eliminating Toil" (unit of measure, tracking before, during and after, rejecting toil
    first, time saved against time invested): <https://sre.google/workbook/eliminating-toil/>
12. Google, SRE Book, "The Evolution of Automation at Google" (a design that needs no automation is better than
    automation; automation that fails at scale): <https://sre.google/sre-book/automation-at-google/>
13. Google, SRE Workbook, "Example Error Budget Policy" (work stops when the budget is spent; a postmortem with at
    least one top-priority action): <https://sre.google/workbook/error-budget-policy/>
14. Theory of Constraints Institute, the five focusing steps: <https://www.tocinstitute.org/five-focusing-steps.html>
    (read 2026-11-13 for [continuous-improvement.md](continuous-improvement.md); Goldratt's "The Goal" not opened)
15. Little's law, L = λW, proved by John Little in 1961; read through the encyclopedia article, the paper not
    opened: <https://en.wikipedia.org/wiki/Little%27s_law>
16. Gene Kim, "Kanbans and DevOps: Resource Guide for The Phoenix Project (Part 2)" (wait against percent busy; the
    author says he does not know the curve's exact derivation): <https://itrevolution.com/?p=68061>
17. Tian Pan, "The merge queue is the new bottleneck" (a practitioner post; its throughput figures are the author's
    arithmetic, not measurements): <https://tianpan.co/blog/2026/07/02/the-merge-queue-is-the-new-bottleneck>
