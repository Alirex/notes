---
summary: "Delegating to subagents (Claude tiers, Junie) and keeping context lean"
group: agents
tags: [agents, context, cost]
applies-to: [all]
reviewed: 2026-11-09
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

### Delegate by default

Standing rule (R-2611191636-86da): a task that is wide reading, a search, a design pass, or implementation in files
other than the supervisor's current ones goes to the cheapest fitting tier (Junie or Haiku for reading and searches,
Sonnet for implementation, Opus only for judgment) in the same turn it is recognised, not after the supervisor has
read it all itself. The supervisor reviews the result and lands. Say which tier ran when reporting.

Stays with the supervisor: task decomposition, a change to the file it is editing right now, cross-cutting or subtle
logic, verification of a child's claims, and landing. A one-command lookup is not worth a subagent.

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

In HQ the idle GPU also keeps a code map of every managed clone (the `code-map` filler): one paragraph per source file,
one page per folder. Haiku and Explore subagents run `just llm-digest map-show <repo> [folder]` first to pick the files
worth opening, then read those files; the summaries are hints that can be stale until the file's next refresh, never a
spec. `just machine-queue report` shows how many map pages were opened.

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

- run the gate detached (`hq q-gate-bg`), wait for it with `hq q-gate-result --wait`, and hand back after PASS; no
  background task left at hand-back;
- edit with the editor tools, not `sed -i` or `perl -pi` on tracked files;
- end the final report with a block the delegator can check by machine: branch, final commit, gate result, whether it
  landed, and where each lesson was saved (a practices commit or an HQ backlog item with `area: practices`; a note in a
  folder outside a repo does not count, it is lost when the session is archived). The delegator checks it (is the commit an ancestor of the default branch, is the primary checkout clean and
  level, are gate processes still running; for example HQ's `just repos-landed <repo> <sha>`) before telling anyone it is done.
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
backlog item the worker reads, not as a long prompt ([review-lenses.md](review-lenses.md#backlog-item-as-brief)).

### Sharpen the brief

Before sending a non-trivial brief, answer four questions and rewrite it: what would a top practitioner of this field
do that the brief does not ask for; what is the rawest material to start from instead of a digest; which methods
and tools fit; what, other than the agent or the supervisor, would show the result is right. The maintainer's own
choices stay open. When the work is more than a few edits, ask for the tool (script, codemod, check) as well as the
result: the tool is what a reviewer can rerun.

### Landing a builder's branch

A builder reports what it ran, not what the gate runs. Before merging its branch (seen in four builder rounds on a
Vue and Rust repo):

- Read `git diff --stat origin/main` for files that do not belong (a committed `node_modules` symlink is the usual
  one: [the rule and the repair](git-worktrees-and-autoland.md#a-shared-install-symlink-stays-untracked)).
  Tell the builder to stage files by name, not `git add -A`.
- Run the whole project's type check, not the touched folders: a builder's "clean" `vue-tsc` on its folders missed two
  `exactOptionalPropertyTypes` errors that the gate found.
- Take the hand-back's "Checked in: tests only" at its word and look at the real screen yourself (phone width,
  existing data for a schema upgrade) before reporting the item done; list what stayed unchecked (two devices,
  drag and drop, a canvas that tests mock).

## Naming sessions

A supervisor with six sessions reads their names in a sidebar and in notes, so a name is an address, not a sentence.
Observed 2026-11-01: two sessions in different repos were both titled "adopt ui collapsibles and auth crate", one
carried a hand-added `!`, and a title only said what the first brief asked, not what the session was doing later.

| Field | Rule | Example |
| --- | --- | --- |
| `title` (public, in the sidebar) | `<repo>: <verb> <outcome>`, one topic (a session that works through a backlog is titled by its current item), at most 50 characters, repo short name first (it groups and sorts), the repo is always in it | `teleporto: adopt ui and auth` |
| `name` = **handle** (stable id for tools and reports) | `<repo>-<topic>`, lower case, `[a-z0-9-]`, at most 24 characters, topic never `work`, `main` or a number, unique among live sessions; the ledger and the notes use it; it cannot change after the start | `teleporto-libs` |
| Not in a title | item ids, branch names, models, state (running, idle, needs you: the app shows them), `!` or other markers | |
| Retire marker | the one exception: `[ARCHIVE]` and a space at the start of the title of a finished session the app would not archive (pinned or in use), so the maintainer sees what to archive by hand | `[ARCHIVE] teleporto: adopt ui and auth` |
| Several sessions for one repo | the topic tells them apart, never a number | `report-bot: remove python writes` |

- Two words only: **title** and **handle** (not name, slug or label). `just cc-names-check --handle H --title T` checks both (no LLM, exit 1 on a break).
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

## Hub and spoke, or peer mesh

Hub and spoke is the default: the caller briefs a subagent, the subagent works in its own window and returns to the
caller only; subagents do not talk to each other. A peer mesh (Claude Code Agent Teams: teammates message each
other and share a task list) is the exception.

- A spoke returns a short summary, 1-2k tokens, with `file:line` references; the hub keeps the conclusions and does
  not paste the spoke's tool output into its own context.
- Do not fan out coding work that shares files or context, has many dependencies between steps, or has little to
  run in parallel. Most coding tasks are like that; one agent, or a chain of spokes, is cheaper and safer.
- Delegating does not make an answer better. A fresh reviewer helps because it has not seen your reasoning.
- Size by the task: one agent for a fact, 2-4 for a comparison, more only with a written brief per agent (see
  [research-flow.md](research-flow.md)). Vague briefs made agents repeat each other's work.

| | Hub and spoke (subagents) | Peer mesh (Agent Teams) |
| --- | --- | --- |
| Talk to each other | No, only back to the caller | Yes, plus a shared task list |
| Worth it for | Wide reads, searches, a bounded build in a worktree, a fresh review | Research and review with competing views, debugging with competing hypotheses |
| Not worth it for | Work that needs the caller's whole context | Write-heavy work, same-file edits, small tasks |
| Rules from the source | Nesting depth 3 and 20 concurrent by default | Experimental, off by default; each teammate owns different files; 3-5 teammates, 5-6 tasks each; the spawn prompt carries all context; start with review or research |
| Cost | Tens of thousands of tokens inside the spoke, 1-2k back | More tokens again: every teammate is a full session |

Cost, level MR (the vendor's own internal measurement, not replicated): agents used about 4 times the tokens of a
chat and a multi-agent system about 15 times; token use alone explained about 80 % of the variance on one benchmark;
an Opus lead with Sonnet workers beat a single Opus agent by 90.2 % on an internal research eval (see Sources). That result is for
breadth-first research; the same post names shared context, many dependencies and most coding tasks as a poor fit.

## Context management

Every turn re-sends the whole context: a long one costs more and dilutes
attention; compacting too often loses details. Claude Code compacts on its own near the limit; the levers are yours
before it does (read 2026-11-09, see Sources).

- Look first: `/context` shows the live breakdown by category with suggestions, `/memory` opens the loaded files.
- Start fresh between unrelated tasks (`/clear`): old conversation crowds out the files needed next and costs tokens on
  every message.
- Compact at a natural breakpoint with a focus, `/compact focus on the auth bug fix`: the summary keeps what you name
  instead of what the automatic pass guesses. `/rewind` then "Summarize from here" compacts only part of the chat.
  `/autocompact <tokens>` sets how full the window gets before the automatic pass; the default thresholds per model
  are on the model-config page, not fixed numbers to copy here.
- What a compaction keeps: the system prompt, the project-root CLAUDE.md and unscoped rules (re-read from disk), auto
  memory, skill descriptions, plan-mode plan, up to five recently read files (a file over 5,000 tokens comes back as a
  path only), and the output of `SessionStart` hooks matching `compact`. What it summarizes away: the chat, context a
  hook added earlier, and path-scoped rules and nested CLAUDE.md files (they reload only when a matching file is
  touched again). Invoked skill bodies are re-attached, 5,000 tokens each and 25,000 in all, oldest dropped first and
  the start of a file kept, so the key instruction goes first in a `SKILL.md`.
  A rule that must hold after a compaction therefore lives in the root file, or is re-injected by a `SessionStart`
  hook (HQ's `preflight.sh digest` does this on `compact`).
- A 1M-token window (offered for several current models; availability is on the model-config page) lets a session run longer; it
  compacts the same way and every turn still re-sends the whole context, so the cost grows with it. A bigger window is
  not a reason to skip the levers above.
- Keep bulky output out: delegate wide reading to a subagent (its reads stay in its own window; a 1-2k summary comes
  back), trim command output, read file ranges.
- Keep long-task state in files (backlog, changelog), not only in chat.

## A repeated brief becomes a definition

When the same inline brief (role, tool limits, output shape) is pasted into several Agent calls, move it into
`.claude/agents/<name>.md`: a Markdown file whose body is the system prompt. Identity comes from `name`, not the file
name; project definitions (`.claude/agents/`, committed) beat user ones (`~/.claude/agents/`) of the same name.
Fields (all camelCase; unknown ones are ignored):

| Field | Meaning |
| --- | --- |
| `name`, `description` | Required. The description tells the caller when to delegate; keep it short, it is in context every turn |
| `tools`, `disallowedTools` | Allowlist and denylist (the denylist applies first); omitted `tools` inherits everything |
| `model` | `haiku`, `sonnet`, `opus`, `fable`, a full id or `inherit`; the per-call `model` of the Agent tool wins over it |
| `permissionMode` | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan`; the caller's mode can override it |
| `maxTurns` | Stops the agent after N turns; the output is marked partial and can be resumed |
| `isolation` | `worktree`: a temporary worktree cut from the default branch, removed when nothing changed |
| `memory` | `user`, `project` or `local`: a persistent `MEMORY.md` (first 200 lines or 25 KB loaded) and read/write tools |
| `skills`, `mcpServers`, `hooks`, `effort`, `background`, `omitClaudeMd` | Preloaded skills, own MCP servers and hooks, effort level, always background, skip CLAUDE.md |

Convert a brief when it was used three times, has a stable tool list and a fixed return shape. Keep a one-off brief
inline. A definition fixes the role, not the task: the task still goes in the call. A read-only fact-gatherer, as in
HQ's `.claude/agents/researcher.md`:

```markdown
---
name: researcher
description: Read-only fact-gathering over the repo: where or how X is used, what a file says. Returns facts with file:line references, never judges or edits.
tools: Read, Grep, Glob
disallowedTools: Write, Edit
model: haiku
permissionMode: plan
maxTurns: 25
---

You gather facts for a caller that keeps the judging. Do not modify files. Search first, read only the ranges you
need. Return at most 30 lines, each fact with `path:line`; mark what you could not find or read; end with what you
searched.
```

Observed in HQ: the built-in `Explore` type already covers broad read-only searches; the definition adds HQ's return
shape (file:line, "not read" marks) and a cheap fixed model. A definition is not enforcement of the caller's checks:
verify its output as for any subagent, and the `SubagentStop` evidence hook in `.claude/settings.json` matches only
`general-purpose|worker|reviewer`.

## A counselor before a stop

A session that ends its turn with work left, or commits to a hard choice, asks a stronger model that has no stake in the
session's own story: HQ's read-only `counselor` agent (`.claude/agents/counselor.md`, model Opus).

- **Enforced, not remembered**: measured 2026-11-19, a counselor and a reviewer agent existed and were called 0 times
  in the HQ transcripts (designer 15, researcher 21, general-purpose 135): a rule the model must remember is followed
  about 0 %, a hook-enforced one 97-100 % (`just cc-adherence`). So the Stop hook, on the last block of a pull-loop
  session with a pickable item, refuses the turn end until the transcript shows a consult since the last maintainer
  message (`Agent` with `subagent_type` counselor, or `cc-counsel ask|log`), once; a wait on own work, no pickable item
  and a closing session keep their behaviour.
- **When**: before writing `stopped` or ending a turn while `backlog_next` still has an item; a design choice in a brief
  of size l; before a landing that carries a `Gate-change:`.
- **How**: `just cc-counsel ask --cwd .` builds the brief (transcript tail, top pickable item, branch state, under 4 KB),
  runs the counselor headless (`claude -p --agent counselor`, `--max-budget-usd 1`, 300 s timeout, model from the agent
  file), prints the answer's first line (`CONTINUE: <step>`, `STOP-OK: <reason>` or `ASK: <question>`) and logs the
  verdict itself. If the headless call gives no verdict (offline, no auth, timeout) it prints the brief and the Agent
  fallback: `Agent subagent_type counselor` with that brief, then `just cc-counsel log --verdict <v> --session <id>
  --item <id> --cost-note <T>`. Optional `--lens reliability|simplicity`.
- **Lenses**: at most two, named in the brief (`reliability`, `simplicity`); never a third.
- **Cost**: one Opus read of 4 KB plus a few tool calls per consult; write the plan percent in `--cost-note` (`cc-cost`).
- **Trial** ([B-261119-advisor-for-supervisor-and-workers](https://github.com/evocaslab/hq/blob/main/docs/backlog/B-261119-advisor-for-supervisor-and-workers.md)):
  the first 10 stops with work left must consult (the hook makes it so); `just cc-counsel stats` shows the verdicts,
  consults per day and how many `stop-ok` still had a pickable item.
  Decision rule: adopt as a hook when `CONTINUE` fires on the stops where work was left and the cost per consult is small
  against a session's day; adopt for fleet supervisors only when it helps there and not on workers; drop with the numbers
  when `stop-ok` mostly agrees with the plain stop.

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

- Advisor tool (beta, API): a faster executor model consults a stronger advisor mid-run; read 2026-11-19:
  <https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool>. Claude Code cannot call this API tool,
  so HQ uses a subagent at fixed moments (the counselor).
- pstack (MIT, Lauren Tan): <https://github.com/cursor/plugins/tree/main/pstack> (`sharpen, principle-build-the-lever`)
- Subagents run in their own context window and return a summary; a model per subagent (`haiku`, `sonnet`, `opus`);
  `isolation: worktree`: <https://code.claude.com/docs/en/sub-agents>
- Sending the whole context every request, clearing between tasks, compaction focus, `model: haiku` for simple
  subagents, delegating verbose work: <https://code.claude.com/docs/en/costs>
- `/context`, `/compact <focus>`, `/rewind`, `/autocompact`, what survives a compaction, skill-body budgets, 1M windows
  (read 2026-11-09): <https://code.claude.com/docs/en/context-window>
- Subagent file format: fields `name`, `description`, `tools`, `disallowedTools`, `model`, `permissionMode`, `maxTurns`,
  `isolation`, `memory` and the rest, locations and priority, a read-only example (read 2026-11-09):
  <https://code.claude.com/docs/en/sub-agents>
- Skill bodies after a compaction (5,000 and 25,000 tokens) (read 2026-11-09): <https://code.claude.com/docs/en/skills>
- The earlier "100-130k auto-compact" lines and `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` were dropped 2026-11-09: the context
  page now documents `/autocompact` and per-model defaults instead; no current source for fixed numbers.
- Junie CLI `--model`, `--effort`, `--review` (headless): <https://junie.jetbrains.com/docs/junie-cli.html>; the `--effort` needs `--model` rule, the `.junie/skills/` leftover and the per-job costs are observed in HQ (2026-11-04), not in that page
- Observed in HQ's own use (2026-10; Junie credit costs, tier order);
  the `junie` flags `--output-format json`, `--project`, `--session-id` were not found in the pages read.
- Context as a finite attention budget; just-in-time retrieval, compaction, notes outside the window, subagents that
  use tens of thousands of tokens and return 1-2k (read 2026-11-07): <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- Multi-agent research system: 4x and 15x tokens, 90.2 % on the internal eval (MR, vendor, not replicated), 80 % of
  variance from tokens, poor fit for shared context and most coding tasks, early failures of too many subagents and
  vague briefs (read 2026-11-07): <https://www.anthropic.com/engineering/multi-agent-research-system>
- Agent Teams (experimental): mesh and shared task list, 3-5 teammates, own files per teammate, start with research or
  review (read 2026-11-07): <https://code.claude.com/docs/en/agent-teams>
- Subagents return to the caller; nesting depth and concurrency limits (read 2026-11-07; the numbers are the
  defaults named there, re-check): <https://code.claude.com/docs/en/sub-agents>
