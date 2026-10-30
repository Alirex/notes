# HQ practices (generated copy)

Synced from <https://github.com/evocaslab/hq/blob/main/practices> by `just practices-sync`; do not edit here, change HQ's page and sync again. Pinned commit and hashes: `.hq-practices.json`.

- [Agent instructions](agent-instructions.md): Laying out AGENTS.md / CLAUDE.md, subfolder rules and skills
- [Agent security](agent-security.md): Coding agents: prompt injection, secrets out of reach, sandbox, MCP trust, never-list
- [Architecture docs and diagrams](architecture-diagrams.md): ARCHITECTURE.md and diagrams: hand-written views, generated views with an is-current hook
- [Autonomous sessions](autonomous-sessions.md): Long unattended sessions: /autopilot, questions instead of stopping, cc-watch
- [Change discipline](change-discipline.md): Working through one change: end state, data shape first, units that end green, when to stop
- [Docs, backlog, changelog, decisions](docs-backlog-changelog.md): Docs, backlog, changelog and decisions laid out so parallel agents don't collide
- [Git worktrees and auto-land](git-worktrees-and-autoland.md): Branching, parallel agents, landing work on main
- [.gitignore and IDE settings](gitignore-and-ide.md): What .gitignore lists and which JetBrains .idea files to commit
- [just task runner](just-task-runner.md): Organizing just recipes and helper scripts
- [Strict checks: fix, don't silence](lint-strictness.md): Strictest linters and type checkers; fix findings, never silence them
- [LLM usage guards](llm-usage-guards.md): Bounded LLM calls: max_tokens, timeouts, input caps, budgets, usage table, cache, fallback
- [Questions for the maintainer](maintainer-questions.md): Asking the maintainer through issues, defaults, blocking, manual-action issues
- [Quality gate](quality-gate.md): Lint/type/test gates, heavy-check locking, coverage on changed lines
- [Releases, versions and rollback](releases-and-versions.md): Release recipe, SemVer or deploy tags, version stamp, update notice, rollback
- [Subagents and context](subagents-and-context.md): Delegating to subagents (Claude tiers, Junie) and keeping context lean
- [Supply-chain and repository security](supply-chain-security.md): Secret scanning, dependency audit and cooldown, Actions hardening, repo settings
- [Testing](testing.md): Test-first, fixtures, guard tests, e2e, flaky tests, parity tests, throwaway service containers
- [Tooling and dependencies](tooling-and-deps.md): uv workspace, pre-commit hooks, tool pins, dependency upgrades, technical debt
