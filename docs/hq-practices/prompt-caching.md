---
summary: "Prompt caching for agent work: how the provider cache and its TTLs behave, what breaks it, idle wakes and subagents, batch with a warm-up, local-model prefix reuse, and measuring the hit share"
group: agents
tags: [agents, claude, cost, llm, performance]
applies-to: [all]
reviewed: 2026-11-15
status: draft
---

# Prompt caching: keep the prefix, measure the hit

A long agent session sends its whole context on every turn. The provider caches the longest unchanged prefix and bills a
read at a fraction of the input price (0.1x, 0.05x on Opus 5.5 and Sonnet 5.5), so what keeps the prefix unchanged and
warm decides much of the plan use. A broken cache raises no error: the only sign is the usage fields. Research and the
HQ audit behind this page: [docs/research/2026-11-15-llm-cache-optimisation.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-15-llm-cache-optimisation.md).

## How the cache behaves (Anthropic, checked 2026-11-15)

- The prefix is matched in the order tools, system, messages. A change invalidates that level and all later ones.
- A prefix under 512 tokens on current models (up to 4,096 on older) is silently not cached.
- Writes cost 1.25x (5 min TTL) or 2x (1 h TTL), reads 0.1x or 0.05x; a read refreshes the timer.
- Claude Code on a subscription gives the main conversation a 1 h TTL; subagents, workflows, forks and compaction get
  5 min. Scope is one machine and directory, so separate worktrees never share a hit.
- Breaks it: `/model`, an effort change, fast mode, compaction, an upgrade, MCP or plugin changes with tool search off.
  Keeps it: editing CLAUDE.md, skills or hooks, plan mode, spawning a subagent (which itself starts cold).
- An idle gap beyond the TTL re-processes the whole context at the write price.
- Other providers differ (OpenAI automatic from 1,024 tokens, Gemini implicit, DeepSeek automatic; local servers reuse a
  prefix per slot, Ollama's reuse is undocumented): measure before relying on it.

## Rules

1. Stable content first, volatile content last; append to a session, never edit earlier turns; keep the tool list fixed
   and serialisation deterministic. This holds for briefs and `SendMessage` notes too: Flow, role and rules first, the
   item text last.
2. Judge a prompt, hook or brief change by the cache hit share, not by feel: `just cc-cost` shows per group the hit
   share (read / (read + write + input)) and the tokens re-cached after an idle gap over 1 h.
3. Do not wake a cold large session. Idle past 1 h with a large context: retire it and start fresh from a brief;
   compact only while warm, at a task boundary.
4. A subagent that waits on a gate over 5 min hands back first (its 5 min TTL expires and every wake rewrites its
   context); the other fix is a trial of `subagentPromptCacheTtl: "1h"`.
5. Pin model and effort at session start; run another model as a subagent instead of switching.
6. Bulk calls through `scripts/llm/llm-batch.py` use the 1 h TTL, one warm-up request before the batch, the read factor
   of the model in the cost estimate, and a shared prefix of at least 512 tokens (the script warns below that).
7. Local model: `llm_calls.py` records `prompt_eval_seconds`; move per-call text (date, file name) behind the fixed
   instruction only when two calls on one prompt show the reuse in that field.
8. Keep always-loaded text (instruction files, memory index, digest) to binding rules: every byte is paid on each
   fresh session and each cold wake.

## Not confirmed

How the plan limit weights cache reads and writes; whether worktrees share the system layer; Ollama prefix reuse; the
Gemini and DeepSeek defaults. A shared KV cache server (vLLM/LMCache, SGLang HiCache) pays only with several clients on
long shared prefixes, which HQ does not have.

An app that calls a provider API itself (no managed repo does today, 2026-11-19) gets its own audit: the MIT skill
[audit-prompt-caching](https://github.com/sernote/audit-prompt-caching) traces misses through request code, tool
registries, usage fields and routing, and reports a stable-prefix byte count; it makes no savings claim. Read it and run
it when a repo adds its first LLM call; HQ's own sessions are measured by `cc-cost` (hit share 96 to 98 % on every day
since 2026-11-14), so there is nothing to audit here.

## Sources

- [Anthropic prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Anthropic batch processing](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Claude Code prompt caching](https://code.claude.com/docs/en/prompt-caching.md)
- [Claude Code costs](https://code.claude.com/docs/en/costs)
- [OpenAI prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching)
- [Gemini caching](https://aistudio.google.com/docs/caching)
- [DeepSeek KV cache](https://api-docs.deepseek.com/guides/kv_cache)
- [llama.cpp server](https://raw.githubusercontent.com/ggml-org/llama.cpp/master/tools/server/README.md)
- [Ollama FAQ](https://docs.ollama.com/faq)
- [vLLM automatic prefix caching](https://docs.vllm.ai/en/latest/features/automatic_prefix_caching/)
- [SGLang HiCache](https://docs.sglang.io/docs/advanced_features/hicache_design)
