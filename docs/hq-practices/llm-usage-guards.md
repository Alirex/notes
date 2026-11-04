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

9. **Context overflow is handled, not hoped away.** Count tokens before sending (the provider's token-counting
   endpoint), and when history or documents would not fit: summarise or compact older turns, clear old tool results,
   or chunk the input. Check `stop_reason` as well as errors: a reply cut at `max_tokens` or at the context window
   is flagged to the user, never shown as complete. The API answers `400` "prompt is too long" when the input alone
   exceeds the window and `413` past the request-size limit.
10. **Retry only what can succeed.** `429`, `500`, `529` and timeouts: backoff with jitter (the official SDKs retry
    twice by default); a `429` from a monthly spend cap has no `retry-after` and fails until the limit is raised, and
    a `400` or `402` billing error is not retried. Fall back (another model of the same provider first, then another
    provider or a local model) only for a task whose eval set passed on the fallback.
11. **The eval set is a gate artefact:** 20-50 real, anonymised inputs per prompt with checks that can fail (schema
    valid, facts present, forbidden content absent); run on a prompt or model change, score recorded in the PR.
    The `/claude-api` skill's eval flow can scaffold it.

## Checklist

| Check | Pass |
| --- | --- |
| `max_tokens`, timeout, input cap | Set at the one client wrapper |
| Per-run, per-user, per-day caps | Enforced and tested |
| Usage table and dashboard | Tokens and cost per day visible |
| Cache | Keyed by prompt version, purged |
| Context overflow | Token count before send; over-long input compacted or refused with a message |
| Eval set | Runs on prompt/model change; score in the PR |
| Provider spend limit | Set in the provider console per workspace/key; alert at 80 % |
| Provider outage | Fallback or queued, tested with a failing stub |

## Feature UX

When a person uses the AI feature, not only a batch job:

- Mark output as AI-made. A suggestion that changes data is previewed first and applied on a click, can be edited,
  rejected or undone ([web-ux.md](https://github.com/evocaslab/hq/blob/main/practices/web-ux.md)); a sent question can be edited and re-asked.
- A rating (useful or not, with an optional reason) and a "report a problem" control store the verdict with the
  prompt and model version, so the eval set grows from real cases.
- On failure, keep the user's input and any steps done, say what failed in plain words, and offer the manual path.
  A multi-step flow saves progress after each step.
- A slow call shows progress or streams text; a stop button cancels it and the budget spend is still recorded.
- Let the user pick what the app can promise to honour (language, answer length); say what data is sent to the
  provider ([privacy-and-personal-data.md](https://github.com/evocaslab/hq/blob/main/practices/privacy-and-personal-data.md)).

## Why

- One loop bug or one pasted book can spend a month's budget in an hour; the caps turn it into an error.

## Tests that pin it

- Stub provider: a call without `max_tokens` fails the wrapper's test; the Nth call in a run is refused; an
  over-budget user gets `429`; an outage test queues the job.

## Examples

- A service that is the only LLM user: `max_tokens` 8192 and 4096 for OCR, timeouts 300 s, input cap 24,000
  characters, 10 reviews per run, a result cache. Typical gaps: no token or cost accounting, no
  per-user or daily cap, no rate limit, no fallback provider.

## Sources

- Claude API errors (400/413/429/500/529, retries, spend-limit 400/429): <https://platform.claude.com/docs/en/api/errors>
- Claude context windows (overflow behaviour, compaction, context editing, token counting):
  <https://platform.claude.com/docs/en/build-with-claude/context-windows>
- Anthropic API errors, rate limits and usage: <https://docs.anthropic.com/en/api/>
- OWASP Top 10 for LLM Applications (LLM04 model denial of service, LLM10 unbounded consumption):
  <https://genai.owasp.org/llm-top-10/>
- Feature UX (preview, edit, undo, feedback, graceful failure): Microsoft Guidelines for Human-AI Interaction,
  <https://www.microsoft.com/en-us/haxtoolkit/ai-guidelines/>; Google People + AI Guidebook, <https://pair.withgoogle.com/guidebook/>
