---
summary: "Laying out AGENTS.md / CLAUDE.md, subfolder rules and skills"
group: agents
tags: [agents, claude]
applies-to: [all]
reviewed: 2026-10-25
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

  Example: `auto-joint-lab/report-bot/AGENTS.md`.
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
- Review the file when it passes the size target; move details out instead of
  compressing prose into fragments.

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

Example: `Alirex/claude-dotfiles` under `claude/memory/report-bot/`.

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
