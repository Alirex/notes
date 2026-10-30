---
summary: "Delegating to subagents (Claude tiers, Junie) and keeping context lean"
group: agents
tags: [agents, context, cost]
applies-to: [all]
reviewed: 2026-10-24
status: stable
---

# Subagents and context

## Delegate on judgment

Start subagents without asking when it clearly helps:

- broad searches and fact-gathering where only the conclusion matters;
- independent parallel tasks on different files (one worktree each);
- second opinions: an independent review of a diff or a design.

Keep task decomposition, cross-cutting changes, subtle logic and final
verification in the primary agent. Do small tasks directly.

## Pick the cheapest tier that works

| Tier | Use for | Examples |
| --- | --- | --- |
| Cheap / small | Searches, boilerplate, small isolated fixes, tests for existing code, extra reviews | Junie Lite |
| Fast | Broad read-only exploration, summarizing, mechanical edits | Claude Haiku |
| Balanced | Everyday implementation, tests, medium refactors | Claude Sonnet |
| Frontier | Hard debugging, architecture, an independent design take | Claude Opus |

Order to reach for them: Junie CLI first (cheap scoped work; it spends JetBrains AI credits, see below), Haiku for
broad searches, Sonnet per independent task in its own worktree, Opus only for
hard reasoning or an independent design take.

Primary agent default: frontier model at medium effort; high for hard
debugging or architecture, low for quick questions and git chores.

## Writing a subagent prompt

Subagents share none of your context. Give: the goal, relevant paths,
constraints, expected output format, and

- for read-only tasks: "do not modify files" (then check `git status`);
- for editing tasks: its own worktree, the fast loop and gate commands, how to
  run heavy checks (detached, never nested in the lock), that it kills only its
  own PIDs, that tests never touch real data dirs, and whether it may land
  (default: report back, the primary agent lands).

A code-editing subagent may skip AGENTS.md, so restate the gate rules in its
prompt:

- fast loop while editing, the full gate run detached (`setsid nohup …`) and
  waited on;
- never wrap the gate in the heavy lock (the gate takes it; nesting deadlocks),
  never run several heavy checks at once;
- the land-grade gate (`q-check-land`) for the run right before landing;
- kill only processes it started, by PID, never `pkill` by pattern;
- the repo's e2e rules (session reuse, API setup, no sleeps).

Example: `auto-joint-lab/report-bot/docs/agents/subagents.md`.

Verify every subagent's output and diff before reporting or landing it, and
tell the user which subagents ran.

## Junie CLI

```bash
junie --skip-update-check --output-format json --project <dir> "<task>"
```

JSON output has `sessionId`, `result`, `changes`, `llmUsage`; continue with
`--session-id <id>`. Junie may edit files without asking, so point `--project`
at a dedicated worktree for edits. Its default model is small: keep design and
subtle logic away from it.

- `junie --review` gives a second opinion on a branch before landing.
- `--model <name>` and `--effort low|medium|high` tune cost and depth.
- If `junie` is not on `PATH`, skip delegation.
- **Not free**: Junie spends JetBrains AI credits from the licence's monthly quota (AI Pro, included with the All
  Products Pack: about 10 credits per 30 days, 1 credit ≈ $1). `llmUsage[].cost` in the JSON is the price of a run;
  a small read-only task on the default Qwen model costs well under a cent, so cheap tasks are still fine, but large
  edits or a frontier `--model` drain the quota fast. `just cc-usage` estimates the spend from
  `~/.junie/sessions/*/summary.json`; the exact balance is on the JetBrains account page.
- **Local model**: `install.sh --local-model` runs Junie on a local Qwen3.6-27B (MLX 4-bit, ~35 GB RAM) and is
  **Apple Silicon macOS only**; Linux prints "not supported yet", and an Intel Mac cannot run it. Elsewhere, point
  Junie at a local Ollama with a custom model profile (`--model custom:<profile>`; see
  [local-llm.md](https://github.com/evocaslab/hq/blob/main/practices/local-llm.md)), which costs no credits but is slow (minutes for a one-line fix).

## Context management

Every turn re-sends the whole context: a long one costs more and dilutes
attention; compacting too often loses details.

- Auto-compact around 100-130k tokens on a 200k window; cap around 150-200k
  even on 1M-context models. In Claude Code, `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`
  sets the threshold as a percentage of the window.
- Start fresh between unrelated tasks.
- Compact at natural breakpoints with a focus ("keep the plan, open issues,
  changed files").
- Keep bulky output out: delegate wide reading, trim command output, read file
  ranges.
- Keep long-task state in files (backlog, changelog), not only in chat.
