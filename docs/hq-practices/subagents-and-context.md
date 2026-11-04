---
summary: "Delegating to subagents (Claude tiers, Junie) and keeping context lean"
group: agents
tags: [agents, context, cost]
applies-to: [all]
reviewed: 2026-11-04
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
| Local GPU | Checkable text jobs: OCR, extraction, log triage, a second review of a diff; free and offline | `qwen3.5:9b`, `gemma4:12b` ([local-llm.md](https://github.com/evocaslab/hq/blob/main/practices/local-llm.md)) |
| Cheap / small | Searches, boilerplate, small isolated fixes, tests for existing code, extra reviews | Junie Lite |
| Fast | Broad read-only exploration, summarizing, mechanical edits | Claude Haiku |
| Balanced | Everyday implementation, tests, medium refactors | Claude Sonnet |
| Frontier | Hard debugging, architecture, an independent design take | Claude Opus |

Order to reach for them: the local model first when the output can be checked (below), Junie CLI next (cheap scoped work; it spends JetBrains AI credits, see below), Haiku for
broad searches, Sonnet per independent task in its own worktree, Opus only for
hard reasoning or an independent design take.

Primary agent default: frontier model at medium effort; high for hard
debugging or architecture, low for quick questions and git chores.

A requested model is not always the model that runs. The desktop app's `start_session` caps `model` at the caller's
own: a Sonnet supervisor asking for Opus or Fable gets Sonnet, silently (observed 2026-11-01: a "Fable review" session
ran on Sonnet). A higher tier than the supervisor's is reachable only through the Agent tool's `model` (`opus`,
`fable`), whose subagent runs inside the supervisor's turn. Read the model that actually runs from the session note or
the agent's result header, never assume it from the request, and say it when reporting which subagents ran.

## Local model as a tier

Where a repo runs the HQ local LLM (`just llm-*`), use it before a paid tier for jobs whose output is checkable, and say
in the report that a result came from it:

- a second opinion on a diff (`gemma4:12b` found more planted bugs than `qwen3.5:9b`; both raise false alarms, so read
  each finding against the code; two models agreeing is a signal, one alone a hint);
- OCR of table screenshots, messy text into JSON or a table (validate with a schema), summaries and triage of logs and
  gate output (`qwen3.5:9b`).

Never for design, cross-cutting changes or subtle logic. It shares the GPU and some CPU with the checks: look at
`just llm-status` first, skip it while a gate runs, cap the reply length, and skip it when Ollama is down.

## Writing a subagent prompt

Subagents share none of your context. Give: the goal, relevant paths,
constraints, expected output format, and

- for read-only tasks: "do not modify files" (then check `git status`);
- for editing tasks: its own worktree, the fast loop and gate commands, how to
  run heavy checks (detached, never nested in the lock), that it kills only its
  own PIDs, that tests never touch real data dirs, and whether it may land
  (default: report back, the primary agent lands).

Lessons from a long migration run (2026-11-04):

- `isolation: worktree` follows the supervisor's own repo. From HQ, create the target repo's worktree yourself and pass
  its path in the prompt.
- A builder's "waiting on background work" note is not a report. A builder idle for hours after a failed gate is resumed
  with a concrete message, or its branch is taken over.

A code-editing subagent may skip AGENTS.md, so restate the gate rules in its
prompt:

- fast loop while editing, the full gate run detached (`setsid nohup …`) and
  waited on;
- never wrap the gate in the heavy lock (the gate takes it; nesting deadlocks),
  never run several heavy checks at once;
- the land-grade gate (`q-check-land`) for the run right before landing;
- kill only processes it started, by PID, never `pkill` by pattern;
- the repo's e2e rules (session reuse, API setup, no sleeps);
- commands that do not trip the permission guard (a prompt stalls an unattended session): one action per command,
  files created with the Write tool, not a heredoc; named files removed with `git rm <files>`, never `git rm -r` or
  `xargs rm`; long runs with `run_in_background`, not chained after other steps. A "recursive rm" prompt on a
  compound command is a brief problem, not a reason to widen the allow list.

Example: a repo's own `docs/agents/subagents.md`.

Completion contract, for every delegate:

- It ends with its final result as its own message (facts, paths, what it could not check), never only as files or
  a log.
- The delegator never ends its turn while children still run: their results would be orphaned. Wait for them, or
  say plainly that they are pending.
- The delegator owns collecting and verifying what it delegated.

Machine-checkable hand-back (for agents that implement or land work; lessons of two stalled days in one repo):

- run the gate in the foreground, or wait for it in the foreground; no background task left at hand-back;
- edit with the editor tools, not `sed -i` or `perl -pi` on tracked files;
- end the final report with a block the delegator can check by machine: branch, final commit, gate result, whether it
  landed, and where each lesson was saved (a practices commit or an HQ backlog item with `area: practices`; a note in a
  folder outside a repo does not count, it is lost when the session is archived). The delegator checks it (is the commit an ancestor of the default branch, is the primary checkout clean and
  level, are gate processes still running; for example a `just q-landed <commit>` recipe) before telling anyone it is done.
  A report is a claim, not proof;
- price a slice from the session transcripts (`just cc-cost --by branch` in HQ): an equivalent at API list rates,
  not a bill.

A worker that ends its turn while its own background run is still alive delivers no report: tell it to stay in the
turn until the run finishes, and watch its commits rather than assuming. To stop a worker, have it commit its tree as a
pushed WIP and say "gate not run", so nothing is lost and the next owner knows what was never checked.

Verify every subagent's output and diff before reporting or landing it, and
tell the user which subagents ran.

Fact-gathering from vendor pricing or console pages: a fetch tool often returns only the title, because the tables are
filled in by script after load. Do not let the subagent fall back to aggregators and report them as read: have it mark
each number read (V) or not read, then read the rendered page in a browser (open the page, wait, take the text from
the DOM, switch the region or tier selector, read again) and keep the judging. Record the date and URL beside every
number.

Review briefs name the lenses to apply ([review-lenses.md](review-lenses.md)); a planned task is handed over as a
backlog item the worker reads, not as a long prompt ([review-lenses.md](review-lenses.md#backlog-item-as-handoff)).

### Sharpen the brief

Before sending a non-trivial brief, answer four questions and rewrite it: what would a top practitioner of this field
do that the brief does not ask for; what is the rawest material to start from instead of a digest; which methods
and tools fit; what, other than the agent or the supervisor, would show the result is right. The maintainer's own
choices stay open. When the work is more than a few edits, ask for the tool (script, codemod, check) as well as the
result: the tool is what a reviewer can rerun.

### Landing a builder's branch

A builder reports what it ran, not what the gate runs. Before merging its branch (seen in four builder rounds on a
Vue and Rust repo):

- Read `git diff --stat origin/main` for files that do not belong: a builder given a worktree with a `node_modules`
  symlink committed the symlink, and the merge replaced the main checkout's real `node_modules` with a link to
  itself. Tell the builder not to `git add -A`, and keep `node_modules` in `.gitignore` as a path rule that also
  matches a symlink (`node_modules`, not `node_modules/`).
- Run the whole project's type check, not the touched folders: a builder's "clean" `vue-tsc` on its folders missed two
  `exactOptionalPropertyTypes` errors that the gate found.
- Take the hand-back's "Checked in: tests only" at its word and look at the real screen yourself (phone width,
  existing data for a schema upgrade) before reporting the item done; list what stayed unchecked (two devices,
  drag and drop, a canvas that tests mock).
- A worktree with an untracked `node_modules` symlink makes a "worktree is clean" check refuse: remove the link
  before the finish command.

## Naming sessions

A supervisor with six sessions reads their names in a sidebar and in notes, so a name is an address, not a sentence.
Observed 2026-11-01: two sessions in different repos were both titled "adopt ui collapsibles and auth crate", one
carried a hand-added `!`, and a title only said what the first brief asked, not what the session was doing later.

| Field | Rule | Example |
| --- | --- | --- |
| `title` | `<repo>: <verb> <outcome>`, at most 50 characters, repo short name first (it groups and sorts), the repo is always in it | `teleporto: adopt ui and auth` |
| `name` (short handle) | stable `<repo>-<topic>`, lower case, unique among live sessions; the ledger and the notes use it | `teleporto-libs` |
| Not in a title | item ids, branch names, models, state (running, idle, needs you: the app shows them), `!` or other markers | |
| Retire marker | the one exception: `[ARCHIVE]` and a space at the start of the title of a finished session the app would not archive (pinned or in use), so the maintainer sees what to archive by hand | `[ARCHIVE] teleporto: adopt ui and auth` |
| Several sessions for one repo | the topic tells them apart, never a number | `report-bot: remove python writes` |

- Name it when it starts (`title` on `start_session`, the bold title line before the call) and rename it when the work
  changes (`set_session_title`): a session that moved from "adopt" to "fix the build" is retitled, or the supervisor
  retires it and starts a new one.
- Sidebar groups by repo (`create_group`, `move_sessions`) keep a fleet scannable; one group per repo, archived with the
  last session in it.
- A supervisor links a session as `[title](#local_<id>)` with the id from the tool result, so a renamed title never
  breaks the link.

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
- **As a read-only reviewer** (observed 2026-11-04, Sonnet-class model at high effort): `--effort` needs `--model`
  (alone it is refused). One job per background call: a launcher subshell that exits says nothing about its children,
  so start each job with its own `run_in_background` call and wait for its notice. Say "do not modify files" in the
  prompt and run `git status` afterwards: one run left an untracked `.junie/skills/` in the repo (delete it, never
  commit it). Check every finding against the code before acting: one review found a real bug (a removal after an alias
  merge in an op-log fold) next to claims that did not hold. Record the cost in the hand-back: about $0.25 to $0.83
  per job. The same run drafts schema queries well ([measures-of-performance-and-effectiveness.md](measures-of-performance-and-effectiveness.md#drafting-indicators-when-data-is-missing)).
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

## Eager overhead

Some context is loaded on every turn before any work: the `CLAUDE.md` / `AGENTS.md` chain, every skill's name and
description, and the schemas of connected MCP servers. A big roster costs tokens each turn and dilutes attention.

- Look at the native `/context` first: it shows what the session really loads, per category.
- Trim by evidence: shorten long skill descriptions, drop skills and MCP servers nobody used, move rarely needed
  rules out of the always-loaded files into pages that are read on demand.
- A static estimate (sum of description sizes, file sizes) only on request, and labelled an estimate.

## Before every start

A supervisor re-reads these rules and checks the plan pace before each start; a hook keeps that from depending on memory: [supervisor-preflight.md](supervisor-preflight.md).

## Sources

- pstack (MIT, Lauren Tan): <https://github.com/cursor/plugins/tree/main/pstack> (`sharpen, principle-build-the-lever`)
- Subagents run in their own context window and return a summary; a model per subagent (`haiku`, `sonnet`, `opus`);
  `isolation: worktree`: <https://code.claude.com/docs/en/sub-agents>
- Sending the whole context every request, clearing between tasks, compaction focus, `model: haiku` for simple
  subagents, delegating verbose work: <https://code.claude.com/docs/en/costs>
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` (percent of the window; can only lower the threshold):
  <https://code.claude.com/docs/en/env-vars>
- Junie CLI `--model`, `--effort`, `--review` (headless): <https://junie.jetbrains.com/docs/junie-cli.html>; the `--effort` needs `--model` rule, the `.junie/skills/` leftover and the per-job costs are observed in HQ (2026-11-04), not in that page
- Observed in HQ's own use (2026-10; Junie credit costs, tier order, compaction thresholds of 100-130k tokens);
  the `junie` flags `--output-format json`, `--project`, `--session-id` were not found in the pages read.
