---
summary: "What .gitignore lists and which JetBrains .idea files to commit"
group: quality
tags: [git, ide]
applies-to: [all]
reviewed: 2026-10-24
status: stable
---

# .gitignore and IDE settings

The maintainer's longer reasoning on IDE settings (in Ukrainian): [Alirex/notes:
ignore_or_not_ide_config](https://github.com/Alirex/notes/blob/main/notes/ignore_or_not_ide_config/ukr.md).

## .gitignore

- **List what this repo's tools generate, grouped by tool** (Python, tests and coverage, linters, Node, Rust,
  Docker, secrets, agents, IDE, OS), each group with a one-line comment. Not the full GitHub Python template: its
  Django, Flask, Scrapy and Celery lines hide nothing here and its broad ones (`lib/`, `var/`, `parts/`) can hide
  real source folders later.
- A folder with its own toolchain keeps its own `.gitignore` (`web/<app>/`, `temp/`).
- Secrets: `.env` and `.env.*`, then `!.env.example`, so the documented example stays tracked.
- Agents: `.worktrees/`, `.claude/worktrees/`, `.claude/settings.local.json`; the shared `.claude/settings.json`
  and skills stay tracked.
- **Checks** after any change: `git ls-files -ci --exclude-standard` must print nothing (no tracked file is hidden);
  when trimming rules, save `git status --ignored --porcelain -uall` before and run it through
  `git check-ignore --no-index --stdin -v -n` after, so nothing that was ignored turns up as untracked.

## JetBrains `.idea/`

**Keep it selectively**: a fresh clone (new machine, reinstall, a second checkout) then opens with the same
layout and plugins, with less to set up again by hand. Ignore all, then allow the shared files:

```gitignore
.idea/**
!.idea/*.iml
!.idea/modules.xml
!.idea/misc.xml
!.idea/vcs.xml
!.idea/ruff.xml
!.idea/MypyPlugin.xml
!.idea/codeStyles/
!.idea/codeStyles/**
!.idea/runConfigurations/
!.idea/runConfigurations/**
!.idea/inspectionProfiles/
!.idea/inspectionProfiles/Project_Default.xml
```

| Keep | Why |
| --- | --- |
| `*.iml`, `modules.xml` | source, test and excluded folders ("Mark directory as"), modules of a uv workspace |
| `misc.xml` | the project interpreter; name it so it works on every machine (`uv (<repo>)`, or a `~/…` path) |
| `vcs.xml` | the Git root mapping |
| plugin settings (`ruff.xml`, `MypyPlugin.xml`, …) | the same linting in every clone; use `$PROJECT_DIR$` and `$USER_HOME$`, never absolute paths |
| `codeStyles/`, `runConfigurations/` | shared formatting beyond `.editorconfig`; run and debug setups (mark them "Store as project file") |
| `inspectionProfiles/Project_Default.xml` | only when customized and used (`profiles_settings.xml` without `USE_PROJECT_PROFILE=false`) |

| Leave out | Why |
| --- | --- |
| `workspace.xml`, `shelf/`, `tasks.xml`, `usage.statistics.xml` | personal state, changes on every click |
| `dataSources*`, `httpRequests/`, `queries/` | may hold credentials and local data |
| AI-plugin state (`copilot*.xml`, …) | migration flags of one machine's plugin |

- Commit the settings early, on the default branch: switching to a branch without them confuses the IDE.
- Read every `.idea` file before its first commit: plugins can store tokens or absolute paths.
- Pre-commit hooks exclude `\.idea/.*`: the IDE owns the formatting of those files.
