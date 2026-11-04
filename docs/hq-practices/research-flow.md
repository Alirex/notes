---
summary: "How agents research a question: brief, source tiers, evidence labels, second reader, model per stage, stop rule, note in the repo, not-taken register"
group: agents
tags: [research, docs, subagents, evidence, decisions]
applies-to: [all]
reviewed: 2026-11-04
status: draft
---

# Research flow

Research is work that feeds a decision: which practice to write, which tool to adopt, which design to take. It
produces a note that someone else (or the same agent a month later) can check and reuse. This page says how; the
health-specific rules stay in [supplement-evidence-research.md](supplement-evidence-research.md).

## The rule

| Step | Rule |
| --- | --- |
| 1. Brief | One paragraph before the first search: the question, the decision it feeds, what is in and out of scope, the budget (subagents, tool calls, wall time) and the deadline. Check the not-taken register (below) and `practices_search` first: a settled question is not reopened without new facts. |
| 2. Sources | Tiers, best first: primary (specs, standards, papers, the code itself) > official docs of the tool or platform > maintained code (recent commits, the tool's own tests) > blog posts, vendor pages, search snippets, model memory. Every claim carries a URL that was opened or a `file:line`; a snippet or a fact from memory is marked `(not opened)`. |
| 3. Evidence level | Each load-bearing claim gets `strong` / `moderate` / `weak` / `none`, the scheme of the supplement page mapped to technical sources: `strong` = spec or primary source read in full, or the behaviour reproduced locally; `moderate` = official docs or maintained code, read; `weak` = blog, abstract, snippet, one example repo; `none` = a vendor's claim about its own product, or memory. |
| 4. Models per stage | Gathering (wide reads, greps, "where is X used"): Junie or Haiku, several in parallel with disjoint scopes. Reading and drafting the note: Sonnet. Judging (what to take, the trade-off): the primary agent, Opus at most. Fable only when the call is contested or high-stakes and the maintainer or a skill asks for it. |
| 5. Second reader | A cheaper model than the writer (Haiku, the local model, or Junie `--review`) re-opens the cited source of each load-bearing claim and answers: does the link open, is it on topic, does the passage support the claim. A claim that fails is downgraded or removed, never silently kept. |
| 6. Stop rule | Stop when two consecutive sources add nothing to the decision, when the brief's budget is used, or when the top-tier source for the question has been read. Defaults when the brief sets none: a fact lookup is one agent and 3-10 tool calls; a comparison 2-4 subagents with 10-15 calls each; more only for a design question across several systems, with a written brief per subagent. |
| 7. The note | Lives in the repo: `docs/research/<date>-<slug>.md` (create the folder on first use), or, when the research only backs a practice page, in that page's Sources. Never only in a session-notes or scratch folder. Shape below. |
| 8. Not-taken register | Each option studied and rejected gets one line in the note's "Studied and not taken" section and, when it affects the whole repo, one line in the repo's register (in HQ: `docs/design/practices-system.md`, "Studied and not taken"). A line says what, why not, when, and what would reopen it. |
| 9. Freshness | Every source line carries the date it was read. The note states a re-check age by volatility: prices, model names and quotas in days; tool versions and APIs in months; specs and papers in years. An older note is cited as `(stale: re-check)`. |
| 10. Hand-back | States what was opened (URLs), what was not and why (403, paywall, out of budget), which claims come from memory, the model that actually ran, and a confidence line: `high` (strong sources, second reader passed), `medium` (moderate sources or no second reader yet), `low` (weak sources, draft). |

## The note's shape

```text
# <question> (<date read>)
Decision it feeds: <backlog item, practice page or design doc>
Scope: in / out. Budget used: <agents, calls, time>. Confidence: high | medium | low.

## Answer (3-10 lines)
## Findings
- <claim> - <level> - <URL or file:line>, read <date>
## Studied and not taken
- <option> - <why not> - <date> - reopen when <condition>
## Not opened / not found
## Re-check: <age> (<reason>)
## Sources (numbered, URL, date read)
```

A research note is not a design doc: it reports what the sources say and what was not found. The decision goes into
the backlog item, design page or practice that the brief named, with a link back to the note.

## Why

**Brief first, because effort does not size itself.** Early versions of a multi-agent research system spawned dozens
of subagents for simple questions until scaling rules went into the prompt (one agent with 3-10 calls for a fact,
2-4 subagents for a comparison) [1]. Each subagent needs an objective, an output format, the tools and sources to use
and clear boundaries, or subagents duplicate each other's work [1]. The budget is also the plan-pace control:
multi-agent research used about 15 times the tokens of a chat [1].

**A URL per claim and a second reader, because citations look better than they are.** An audit of deep-research agents
found the strongest models keep link validity above 94 % and relevance above 80 %, yet only 39-77 % factual accuracy
of the cited sources [2]. The second-reader check is that audit's pipeline: fetch each URL, ask whether it opens, is on
topic and supports the specific fact [2]. The first supplement research run showed the same locally: a journal name
from memory was wrong and regulators' pages answered 403 to automated fetches [3].

**Source tiers, because search ranks for SEO, not truth.** Early research agents preferred SEO-optimised content farms
to authoritative sources until source-quality heuristics were added to the prompts [1]. The tiers are those heuristics
written down once.

**Evidence labels reused.** The four levels come from the supplement page's simplification of GRADE certainty [3]; one
vocabulary across health and technical notes keeps reviewers on one scale.

**The note in the repo, because a scratch folder is where research goes to be redone.** Notes outside the repo are not
synced, not found by search tools and not linked from what they fed, so the next session researches again. The
not-taken register is the ADR idea of recording rejected options: without it a later reader accepts or reverses a
decision blindly [4]; MADR keeps every considered option with pros and cons and dates the record [5].

**Freshness, because sources move at different speeds.** A price is wrong within days, an API within months, a spec
within years; the supplement page already dates every price for that reason [3].

**Hand-back with what was not opened, because silence hides gaps.** A reviewer cannot tell an unopened source from a
confirmed one [3]; the confidence line gives the supervisor one word to decide whether to act or send it round again.

## Pitfalls seen

- A subagent "confirms" a hypothesis it was handed; brief it with the question, not the expected answer.
- A query URL (search page, API call) is not a source: cite the landing page a human would read [3].
- Own arithmetic, benchmarks or a local reproduction are useful evidence, labelled as own work with the command.
- Two subagents reading the same page waste budget: give disjoint scopes.
- A wait for slow subagents is a completion notice, never a sleep loop.

## Sources

1. Anthropic, "How we built our multi-agent research system" (scaling rules, subagent briefing, source-quality
   heuristics, 15x tokens), read 2026-11-04: <https://www.anthropic.com/engineering/multi-agent-research-system>
2. "Cited but Not Verified: Parsing and Evaluating Source Attribution in LLM Deep Research Agents", arXiv 2605.06635
   (link, relevance and fact-check pipeline; 39-77 % factual accuracy), abstract read 2026-11-04:
   <https://arxiv.org/abs/2605.06635>
3. This repo's [supplement-evidence-research.md](supplement-evidence-research.md) (evidence levels, "Not verified",
   dated prices, landing-page citations), read 2026-11-04.
4. Michael Nygard, "Documenting Architecture Decisions" (blind acceptance versus blind reversal), read 2026-11-04:
   <https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions>
5. MADR, "Markdown Any Decision Records" (considered options, status and date), read 2026-11-04:
   <https://adr.github.io/madr/>
