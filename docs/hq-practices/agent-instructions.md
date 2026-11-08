---
summary: "Laying out AGENTS.md / CLAUDE.md, subfolder rules and skills"
group: agents
tags: [agents, claude]
applies-to: [all]
reviewed: 2026-11-07
status: stable
---

# Agent instructions

How to lay out instructions for AI coding agents (Claude Code, Junie, Codex,
Cursor) so they are read, followed, and cheap to load.

## Why not one big file

Every turn re-sends the instructions. A 30+ KB AGENTS.md costs tokens on every
call, dilutes attention (agents skip rules buried in the middle), and becomes a
merge-conflict hotspot. Most of it (how to land, how to write a migration) is
needed rarely.

## Layout

```text
AGENTS.md            short (aim ≤ 6-8 KB): must-follow rules + "read when" link table
CLAUDE.md            @AGENTS.md            (Claude Code reads CLAUDE.md, not AGENTS.md)
docs/agents/*.md     topic files: git/landing, quality gate, style, tooling, subagents
<app>/AGENTS.md      rules only for that subtree (backend, frontend)
<app>/CLAUDE.md      @AGENTS.md
.claude/skills/<name>/SKILL.md   step-by-step how-tos needed rarely
```

- **Root AGENTS.md** holds what an agent must never get wrong: never work on
  `main`, the verification commands, when to land and when to stop, the hard
  style rules (typing, error handling, layering). Everything else is one line
  plus a link.
- **"Read when" table.** The root file maps each task to the one file to read first:

  | Task | Read |
  | --- | --- |
  | Any change (branching, landing, the land menu) | `docs/agents/git-and-landing.md` |
  | Running checks, flaky tests, heavy lock | `docs/agents/quality-gate.md` |
  | Delegating to subagents, context size | `docs/agents/subagents.md` |
  | Web frontend | `web/<app>/AGENTS.md` |
  | Alembic migration | `.claude/skills/db-migration/SKILL.md` |

  Example: a monorepo's root AGENTS.md with this table.
- **Plain Markdown links, not `@imports`, in AGENTS.md.** Junie and most other
  agents read AGENTS.md but do not expand `@path`; they can follow a link by
  reading the file. `@AGENTS.md` belongs only in CLAUDE.md.
- **Subfolder files.** Claude Code loads a nested CLAUDE.md when it touches files
  in that folder. Other agents don't, so the root file says "working in `web/`?
  read `web/AGENTS.md` first".
- **Skills** for procedures: a skill's description loads always, its body only
  when triggered. Good candidates: the full land menu, DB migrations, dependency
  upgrades, mutation testing, release. Link each `SKILL.md` from AGENTS.md so
  non-Claude agents can read it as a plain doc.
- **Topic docs over duplication.** When a rule already has a home doc (e.g.
  API conventions), link it instead of restating it.

## Writing rules

- Imperative, one rule per bullet, the reason in a few words when it is not
  obvious ("each heavy run uses every core").
- Say that read-only questions need no branch and no landing, so an agent does
  not create a worktree to answer "where is X used?".
- Name exact commands (`just q-check`), not descriptions of them.
- State exceptions next to the rule, not in a far-away section.
- Separate "must" from "prefer". Put the musts first.
- When a rule exists to prevent a past incident, keep it short and add a test
  or lint that enforces it; a check beats a sentence.
- A web app's AGENTS.md states its **data mode** in one line, `online-first` (default) or `offline-first`, linking
  [offline-first-sync.md](https://github.com/evocaslab/hq/blob/main/practices/offline-first-sync.md#choosing-the-mode); a repo audit checks that line against the code.
- Review the file when it passes the size target; move details out instead of
  compressing prose into fragments.

## Use the MCP for practices and tasks

A tool that is registered but never named in the instructions is not called: its schema is deferred, so an agent does
not know it exists. HQ's server was connected for a day with an empty call log, because no instruction pointed at it.
Name the tools where the task is named:

- The repo's AGENTS.md has one short rule: "To find the practices or the backlog, call the `hq` MCP tools first
  (`practices_for`, `practices_get`, `practices_search`, `backlog_list`, `backlog_get`); fall back to `just` and the
  files when the server is not connected." The rule is a pointer to [docs/design/hq-mcp.md](https://github.com/evocaslab/hq/blob/main/docs/design/hq-mcp.md)
  in HQ, not a copy of the tool table.
- The "read when" table names the tool beside the file (`practices_get <slug>` instead of opening the page).
- The user-level file (`~/.claude/CLAUDE.md`) says it once for every repo; a repo adds it only when it is not
  reached from there (cloud sessions, other devices, Junie, Codex).
- Writes stay explicit: status changes and closing an item go through the write group only when a session was started
  with it; otherwise edit the file by the repo's backlog rules.
- Measure: `jq -r .tool ~/.local/state/hq/mcp/calls.jsonl | sort | uniq -c`. An empty log after a week means the
  instruction is not read or the server is not worth keeping.

### Make the tools forgiving

Read the call log for failures before adding tools: in HQ's first day 6 of 20 calls failed, all avoidable. Models guess
argument names, ask for records that were deleted, and call a tool by the wrong key.

- Accept the obvious aliases for a field (`id` for `item_id`, `repo` for `repo_or_tags`) and still refuse unknown ones.
- Every invalid-arguments error lists the tool's accepted fields and marks the required ones; a bare "missing field" costs
  a retry.
- A lookup that finds nothing says why it can be empty (finished items are deleted) and names the search tool and the
  nearest matches (shared slug words), so the next call succeeds.
- Add a one-call "session start" tool (plan pace, capacity, top tasks) and a text search before typed wrappers for
  scripts that print only text; a wrapper needs a JSON mode first.
- A script that prints only text still gets a read-only tool when sessions keep asking for it: run it with a timeout,
  cap the output (about 20,000 characters) and validate the arguments (plain repo and date patterns); a JSON mode
  where one exists.

### Design the tools for the agent

- Few, high-impact tools: one tool per task the agent has (a session-start tool, a search), not one per API endpoint;
  overlapping tools make the agent pick wrongly. Consolidate multi-step lookups into one call.
- Offer a concise/detailed switch (a `response_format`-style enum or a `limit`): the default returns what the agent
  needs to decide, the detailed form adds the ids for follow-up calls.
- Fix-it error messages: say what was wrong, the accepted values and what to call next; an opaque code costs a retry.
- Cap the output and say how to narrow it. Claude Code warns when an MCP result passes 10,000 tokens, cuts at 25,000
  tokens by default (`MAX_MCP_OUTPUT_TOKENS` raises it; the warning threshold is fixed) and saves a text result over
  50,000 characters to a file unless the server raises that tool's own limit. Keep a tool well below that: paginate,
  filter or truncate with a sensible default, and let the truncation notice point to a narrower call.

### Roll the rule out and verify it

- Skills and the "read when" tables name the MCP tool first and the `just`/file form as the fallback, so the wording
  does not drift from the rule.
- Check every repo with one grep over `AGENTS.md` for `practices_for`; a repo with live sessions gets the line by
  message, not by a second writer in its checkout. A repo whose own gate fails on unrelated files is reported, not
  forced.
- Run `claude mcp list` once: a server registered at both user and project scope is a duplicate.

### Show the use in the app

- Aggregate the call log in a pure function behind a read-only endpoint: a capped read (last N lines and bytes),
  bad lines skipped and counted, per day, tool, client and repo (from the working folder, a worktree mapped to its
  repo), failures, p50/p95; never arguments or results.
- A missing log is an empty state, not an error; a card on the existing machine page beats a new route.

## Writing a skill

A skill is a procedure an agent loads when the task matches. Rules (Claude Code behaviour, read 2026-11-07; see Sources):

- Three loading levels: the description is always in context (cut at 1,536 characters), the `SKILL.md` body loads
  when the skill is invoked, supporting files load only when the body points to them. Put the trigger ("use when ...")
  in the description, the steps in the body, long tables and examples in supporting files.
- Keep `SKILL.md` under 500 lines. After a compaction the latest invocation of each skill is re-attached, the first
  5,000 tokens of each and 25,000 tokens in all, so a long body is cut where it matters.
- A skill with a side effect (deploy, publish, send) sets `disable-model-invocation: true`: it stays out of the
  model's context and runs only when a person invokes it.
- `context: fork` runs the skill in a subagent without the chat history: its text must stand alone, with every path,
  command and decision it needs.
- `allowed-tools` only pre-approves tools; it does not restrict the rest. Project skills apply it even in an untrusted
  folder, so review the skills of any cloned repo before opening it ([agent-security.md](agent-security.md)).
- A rule that must never bend goes into a hook or a deny rule, not into the skill or the instructions file; prose is
  followed most of the time, a hook every time ([guards.md](guards.md)).
- The "stop doing" list from the Claude Code docs applies here too: no ALL-CAPS emphasis, no long architecture prose;
  the defaults are usually right.
- Trim by evidence: a skill nobody invoked costs its description on every turn
  ([subagents-and-context.md](subagents-and-context.md#eager-overhead)).

## Bind pages by task kind

A repo's binding line of a few pages out of 100+ applicable ones is too thin: sessions open only what the line names
(measured 2026-11-07 on a new web app: 7 of 107 bound, the rest rarely read).

- The repo's AGENTS.md holds a table "task kind → pages to read first" (UI, auth, sync, server, LLM, release and ops),
  under 15 lines, from `practices_for <repo>`.
- The slice or item template has a "Pages to follow" section, and the hand-back ends with "Pages followed:".
- Measure with `just cc-adherence --project <name>` before and after.

## Prepared commands

Repeated checks (landed, tag, release, gate result) are recipes with a one-line result, named in briefs instead of
pasted as pipelines: [prepared-commands.md](prepared-commands.md).

## Global vs repo vs HQ

- `~/.claude/CLAUDE.md`: personal preferences across all repos (language
  feedback, which subagents are available on this machine).
- Repo AGENTS.md: binding rules for that repo, readable without anything else.
- HQ `practices/`: the shared reasoning and longer guides; repos link here.

## Claude memory

Claude Code's auto-memory (`~/.claude/projects/<project>/memory/`) keeps facts
across sessions. Conventions that keep it useful:

- **One fact per file**, with the frontmatter the memory tool writes: `name`,
  `description`, and under `metadata` the `type` (`user | feedback | project |
  reference`), `node_type: memory`, the origin session and `modified`. Keep
  that shape when editing by hand, so the tool keeps reading the file.
- Body: the fact, then a **Why:** line (the reason or incident) and a
  **How to apply:** line (when and how it changes behavior). Feedback and
  project memories need both; they let a later agent judge edge cases.
- Link related memories with `[[file-name]]`.
- `MEMORY.md` is the index, one line per entry: `- [Title](https://github.com/evocaslab/hq/blob/main/practices/file.md) — hook`.
- Repo rules belong in the repo's AGENTS.md, not in memory; memory holds what
  the repo cannot (the user's preferences, context, decisions in progress).

Example: a dotfiles repo with a `claude/memory/<project>/` folder per project.

## Syncing global files and memory

- Keep `~/.claude/CLAUDE.md` and `settings.json` as symlinks into a dotfiles
  repo (private, pushed to a remote); a small `install.sh` restores the links.
- After changing the global CLAUDE.md or writing memories, commit and push from
  the dotfiles repo, so a new machine gets them with `git clone` + `install.sh`.
- Never put credentials, sessions or caches from `~/.claude` in that repo. Its
  `.gitignore` blocks the usual names (`.credentials*`, `*token*`, `*.jsonl`,
  `*.key`, `*.pem`, `.env`, `.env.*`, `*.bak`), and `git status` is checked
  before every commit anyway.
- Memory files are machine-written (frontmatter, one-line paragraphs): exclude
  them from markdownlint with a comment saying why, not by relaxing the rules.
- Personal preferences (e.g. "add an English tip to my replies", which
  subagents exist on this machine) go in the global CLAUDE.md, never in a repo.

## Sources

- Claude Code reads CLAUDE.md, falls back to AGENTS.md only when no CLAUDE.md exists, expands `@path` imports,
  loads nested files on demand, and keeps auto memory in `~/.claude/projects/<project>/memory/` with a `MEMORY.md`
  index: <https://code.claude.com/docs/en/memory>
- Skill descriptions are always in context, the body loads only when the skill is used:
  <https://code.claude.com/docs/en/skills>
- AGENTS.md as a shared format, nested files with the closest one winning, support in Codex, Cursor and Junie:
  <https://agents.md>
- Observed in HQ and the repos it manages (the size target, the MCP call-log failures, the "read when" table, the dotfiles sync); no external source.
- Skill loading levels, description cut at 1,536 characters, 500-line guidance, re-attachment after compaction (5,000
  and 25,000 tokens), `disable-model-invocation`, `context: fork`, `allowed-tools` (read 2026-11-07):
  <https://code.claude.com/docs/en/skills>
- Defaults are usually right, the "stop doing" list, unbendable rules in hooks (read 2026-11-07):
  <https://code.claude.com/docs/en/best-practices>
- Tool design (few high-impact tools, concise/detailed switch, actionable errors, 25,000-token default cap) (read
  2026-11-07): <https://www.anthropic.com/engineering/writing-tools-for-agents>
- MCP output limits (10,000-token warning, 25,000-token default, `MAX_MCP_OUTPUT_TOKENS`, over 50,000 characters saved
  to a file; a server raises one tool's limit with `_meta["anthropic/maxResultSizeChars"]` in its `tools/list` entry,
  up to 500,000 characters, and that value replaces `MAX_MCP_OUTPUT_TOKENS` for the tool's text; images keep the
  token cap) (read 2026-11-07, section "Raise the limit for a specific tool"): <https://code.claude.com/docs/en/mcp>
