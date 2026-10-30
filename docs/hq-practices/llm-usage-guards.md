---
summary: "Bounded LLM calls: max_tokens, timeouts, input caps, budgets, usage table, cache, fallback"
group: security
tags: [llm, cost]
applies-to: [all]
reviewed: 2026-10-30
status: stable
---

# LLM usage guards

Keeps calls to a language model bounded in cost, time and blast radius. Prompt injection and tool safety are in
[agent-security.md](agent-security.md); local models in [local-llm.md](https://github.com/evocaslab/hq/blob/main/practices/local-llm.md); retries and timeouts in
[jobs-and-integrations.md](https://github.com/evocaslab/hq/blob/main/practices/jobs-and-integrations.md); personal data sent out in
[privacy-and-personal-data.md](https://github.com/evocaslab/hq/blob/main/practices/privacy-and-personal-data.md). Status checked 2026-10.

## Rules

1. **Every call is bounded:** `max_tokens` set, a request timeout, and a cap on prompt size (truncate or refuse
   with a visible flag, never silently drop the tail).
2. **Every job is bounded:** a maximum number of calls per run, a per-user and per-day budget (tokens or money),
   and a hard stop when the budget is used up. Over-budget answers `429`/a clear message, not a surprise bill.
3. **Spend is measured.** Each call logs model, input and output tokens, duration and the job/user id to a table
   that a status page sums per day. An alert fires at 80 % of the budget.
4. **Cache what repeats.** Key = hash of model + prompt version + input; store the result with a TTL and
   `last_used_at`, purge by age ([privacy-and-personal-data.md](https://github.com/evocaslab/hq/blob/main/practices/privacy-and-personal-data.md)).
5. **Choose the cheapest model that passes the task's eval.** A small fixed evaluation set per prompt version
   (inputs and expected properties) is run when the prompt or model changes; the choice and date are recorded.
6. **Failure has a plan.** On provider error, `429` or timeout: bounded retry with jitter, then a fallback
   (second provider or local model) or a visible "unavailable" state; the job stays queued, not lost.
7. **Output is untrusted.** Parse into a schema, validate, and never execute or render it raw
   ([web-app-security.md](https://github.com/evocaslab/hq/blob/main/practices/web-app-security.md), rule 3).
8. **Keys are scoped.** One key per app and environment with a provider-side spend limit; never in the repo.

## Checklist

| Check | Pass |
| --- | --- |
| `max_tokens`, timeout, input cap | Set at the one client wrapper |
| Per-run, per-user, per-day caps | Enforced and tested |
| Usage table and dashboard | Tokens and cost per day visible |
| Cache | Keyed by prompt version, purged |
| Provider outage | Fallback or queued, tested with a failing stub |

## Why

- One loop bug or one pasted book can spend a month's budget in an hour; the caps turn it into an error.

## Tests that pin it

- Stub provider: a call without `max_tokens` fails the wrapper's test; the Nth call in a run is refused; an
  over-budget user gets `429`; an outage test queues the job.

## Examples

- report-bot (the only LLM user): `max_tokens` 8192 and 4096 for OCR, timeouts 300 s, input cap 24,000 characters,
  10 reviews per run, result cache (`llm/client.py`, `report_review/`). Gaps: no token or cost accounting, no
  per-user or daily cap, no rate limit, no fallback provider.

## Sources

- Anthropic API errors, rate limits and usage: <https://docs.anthropic.com/en/api/>
- OWASP Top 10 for LLM Applications (LLM04 model denial of service, LLM10 unbounded consumption):
  <https://genai.owasp.org/llm-top-10/>
