---
summary: "Running the coding agent unattended (headless, CI, GitHub Actions): flags, permissions, secrets, cost caps, plugins, rollout order"
group: agents
tags: [agents, ci, security, headless]
applies-to: [all]
reviewed: 2026-11-07
status: draft
---

# Headless and CI agents

Running the agent with nobody at the keyboard: a script, a scheduled job, a GitHub Actions workflow. Long interactive
sessions that run unattended are in [autonomous-sessions.md](autonomous-sessions.md); the threat model (prompt
injection, secrets, never-list) is in [agent-security.md](agent-security.md). This page is the runbook for the
non-interactive mode. Claude Code is the example; flags were read 2026-11-07 (see Sources).

## Headless runs

- `claude -p "<task>"` runs once and exits; the exit code is non-zero on failure. `--output-format json` returns the
  result, session id, and `total_cost_usd` (a client-side estimate, not the bill).
- **Use `--bare` in scripts and CI.** It skips auto-discovery of hooks, skills, plugins, MCP servers, auto memory and
  `CLAUDE.md`, so the run is the same on every machine and a teammate's `~/.claude` cannot change it. Pass what you
  need explicitly (`--settings`, `--mcp-config`, `--append-system-prompt`). It does not use a subscription login: set
  `ANTHROPIC_API_KEY`. Hooks do not run in it, so do not use it for a run that depends on a hook as its guard.
- Without `--bare`, a `-p` run executes the project's `.claude/settings.json` hooks and connects the `.mcp.json`
  servers with no trust dialog and no approval prompt, even in a folder you never trusted. Never run it unbare on a
  repo you did not write ([agent-security.md](agent-security.md#mcp-servers-plugins-skills)).
- Permissions: a `-p` run cannot ask. Pre-approve narrowly with `--allowedTools` (`"Bash(git diff *)"`, not bare
  `Bash`), and for locked-down CI add `--permission-mode dontAsk`, which denies every call that would prompt. The
  mode and the allow list decide the run; the prompt text does not.
- Caps: `--max-budget-usd` stops the run (and its background work) at an estimated spend (print mode only; spend can
  pass the cap, so leave headroom); `--max-turns` limits agentic turns (print mode only, no limit by default). In the
  Action put it in `claude_args`. A job with no cap is a bill nobody reads.
- MCP: `--strict-mcp-config` with `--mcp-config` uses only the servers from `--mcp-config` and ignores all other MCP
  configuration (project, user). `--bare` already skips MCP discovery. Check `mcp_server_errors`
  and `plugin_errors` in the `system/init` event: a bad entry is skipped and the run still exits 0.
- The prompt says what success looks like and what to do with the result; the run cannot ask a clarifying question.

## GitHub Actions

`anthropics/claude-code-action@v1` runs the agent in a workflow. Gotchas (read 2026-11-07, full page):

- The triggering user needs write access (`allowed_non_write_users` plus your own `github_token` to allow others;
  `schedule` events skip this check); bot actors are rejected unless listed in `allowed_bots`, which also covers
  scheduled runs attributed to the user who last changed the `cron`; on a public repo a fork PR gets no secrets.
- `id-token: write` is needed for the default GitHub App authentication; `--allowedTools` in `claude_args` is what
  starts the MCP server for inline review comments; a plain-text `prompt` run has no shell or GitHub tools until you
  grant them.
- Keep the API key in GitHub Secrets (or use OIDC to a cloud provider), never in the workflow file or the prompt.
  Give the token the narrowest permissions the job needs ([agent-security.md](agent-security.md)).
- Commits pushed with the default `GITHUB_TOKEN` do not trigger other workflows, so CI will not run on the agent's
  commit unless a different credential is used.
- Scheduled workflows run only from the default branch; on a public repo they are disabled after 60 days without
  activity.
- Cost is Actions minutes plus tokens. Set `--max-turns` in `claude_args`, a workflow timeout and `concurrency` so one
  PR does not start the agent twice
  ([quality-gate.md](quality-gate.md)).
- Treat the PR text, issue body and file contents the agent reads as data from an attacker; the agent's token is what
  they would use.

## Plugins

A plugin bundles skills, agents, hooks and MCP servers as one versioned directory (`.claude-plugin/plugin.json`).

- Use a plugin to share the same setup across several repos or people. Use the repo's own `.claude/` for what only
  one repo needs; it is versioned with the code.
- Descriptions of enabled plugins are in context every turn ([subagents-and-context.md](subagents-and-context.md#eager-overhead)).
- A plugin's hooks and MCP servers run with your privileges: read them before installing and again when they change.

## Rollout order (our order, not a vendor source)

No primary source gives a "30-day rollout" or a "5-phase harness"; those are a course's framing. The pieces exist in
the docs (managed settings, shared permission config in version control, usage monitoring). Our order, each step
only after the one before held for a while:

1. Read-only use: review and explain jobs with `dontAsk`, no write token, `--max-budget-usd` set.
2. Shared config in version control: the repo's `.claude/settings.json` and instructions
   ([agent-instructions.md](agent-instructions.md)), hooks as guards ([guards.md](guards.md)).
3. A narrow write path: a branch-only agent that opens a PR; the gate runs on it like any other
   ([quality-gate.md](quality-gate.md)); a person lands.
4. Spend and result monitoring: cost per run, failures, denials; look at them before widening anything.
5. Widen one thing at a time (more repos, a schedule, auto-land) and measure it against a few tasks
   ([agent-harness-evals.md](agent-harness-evals.md)).

## Sources

- Headless mode: `-p`, `--bare` (skips discovery, needs `ANTHROPIC_API_KEY`, "recommended for scripted calls"),
  `--allowedTools`, `--permission-mode dontAsk`, `--output-format json` with `total_cost_usd`, `--max-budget-usd`,
  `--mcp-config`, `mcp_server_errors` in `system/init`, no trust or approval prompt for unbare `-p` (read 2026-11-07,
  full page): <https://code.claude.com/docs/en/headless>
- `--max-turns`, `--max-budget-usd`, `--strict-mcp-config` entries (read 2026-11-07, page text saved and grepped):
  <https://code.claude.com/docs/en/cli-reference>; `--strict-mcp-config` also on <https://code.claude.com/docs/en/mcp>.
- GitHub Actions gotchas listed above (read 2026-11-07, full page): <https://code.claude.com/docs/en/github-actions>
- Plugins: bundle, `plugin.json`, context cost, privileges (read 2026-11-07, summary): <https://code.claude.com/docs/en/plugins>
- MCP trust and security notes (read 2026-11-07, summary): <https://code.claude.com/docs/en/mcp>,
  <https://code.claude.com/docs/en/security>
- Not found: any primary source for a staged "30-day" or "5-phase" rollout; the order above is HQ's own.
