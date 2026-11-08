---
summary: "uv workspace, pre-commit hooks, tool pins, dependency upgrades, technical debt"
group: quality
tags: [deps, hooks]
applies-to: [all]
reviewed: 2026-11-01
status: stable
---

# Tooling and dependencies

Example: a repo template (the starting point) and a mature repo that grew from it.

## uv workspace

- uv for environments and dependencies; no framework by default; keep usage
  intentional.
- The workspace **root holds only tooling** (no installable `[project]`; uv's workspace page calls the root "also a
  workspace member", so run `uv sync` on the pinned uv to confirm a tooling-only root still works):
  `members = ["packages/*"]`, the dev dependency group, and tool config. Code
  lives in members (libraries and apps alike).
- `[tool.uv] required-version = ">=X.Y.Z"` makes older uv refuse to run; CI
  (`setup-uv`) installs that version. It is the single source of the uv version.
- A pin-bump script moves every pin together (root `required-version`, the
  uv-pre-commit hook rev, the Dockerfile image tag, the `uv_build` bound of each
  member), upgrades the local uv and relocks. A test fails when pins disagree
  (see [testing.md](testing.md#guard-tests-over-prose)). Use `perl -pi` in it
  ([just-task-runner.md](just-task-runner.md)).

## Dependency groups

- Comment each dependency with why it is there.
- Group by purpose in comment regions: `# [typing]-[BEGIN]` ...
  `# [typing]-[END]` (also `testing`, `linting`, `task`, `extra_code_quality`).
  The same `[name]-[BEGIN]/[END]` style marks regions of tool config, e.g.
  `[disable_explicit]` and `[temporarily_lower_the_level]` in basedpyright.
- Dev tools (`ruff`, `basedpyright`, `ty`, `pytest`, `pytest-xdist`,
  `pytest-cov`, `prek`, `rust-just`) go in the `dev` group.
- Extras for occasional checks: `vulture` (dead code) and `radon` (complexity).

## Hooks (prek)

prek runs the `pre-commit` config. `prek install` sets up both stages
(`default_install_hook_types: [pre-commit, pre-push]`; default stage is
pre-commit).

- Cheap fixers and checks on commit: `uv-lock`, merge-conflict markers,
  trailing whitespace, end-of-file, YAML/TOML/JSON syntax, ruff check and format.
- `check-added-large-files` with `--maxkb=1024`: 1 MB leaves room for icon packs
  while catching accidental dumps.
- `shellcheck` through the `shellcheck-py` wheel (no system install).
- `check-executables-have-shebangs` and `check-shebang-scripts-are-executable`:
  a script under `scripts/` that lost its executable bit fails in a recipe, not
  in review.
- A justfile formatter hook (`format-justfile`).
- `pre-commit-update` keeps hook revs current. A gate that runs fixers itself
  may skip it. Where it runs on commit it fails the commit whenever a newer rev exists (and rewrites
  `.pre-commit-config.yaml`): land the bump as its own `chore:` commit first, never inside a feature commit, and never
  skip the hook (seen in `evocaslab/ui`, 2026-11).
- `markdownlint` with structural rules only (`.markdownlint.yaml` turns off line
  length, inline HTML and the like), so existing docs pass.
- Repo-specific checks are `local` hooks (`language: system`) running a script:
  e.g. backlog files valid, nothing private tracked.
- Install prek once per machine (`uv tool install prek`, or as a dev dependency in
  a uv project) and `prek install` once per clone; a `setup-hooks` recipe does both.
  `--overwrite` replaces hooks left by the old `pre-commit`.
- **basedpyright on pre-push**, not on every commit: a local hook,
  `uv run basedpyright`, `pass_filenames: false`, `require_serial: true`.

- Security hooks (secret scanning, zizmor) and the dependency audit: [supply-chain-security.md](supply-chain-security.md).

## Gitignore

Besides the usual Python/IDE entries, ignore `.quality/` (gate logs and coverage
reports) and `.worktrees/` (agent worktrees kept inside the clone, see
[git-worktrees-and-autoland.md](git-worktrees-and-autoland.md)).

## Dependency upgrades

- Weekly chore: `uv lock --upgrade && uv sync` (and `pnpm update` for a
  frontend), then the full gate, on a work branch in its own worktree with a
  clean tree. A script wraps it: `--dry-run` (show what moves), `--latest`,
  `--commit` (only if the gate passed), `--skip-gate`. It never pushes and
  refuses to run on `main`.
- Minor and patch bumps with a green gate auto-land
  ([git-worktrees-and-autoland.md](git-worktrees-and-autoland.md)).
- `--latest` (majors, ignoring ranges) never auto-lands: show the land menu with
  what changed and why.
- Updates respect a release cooldown and are followed by an audit
  ([supply-chain-security.md](supply-chain-security.md#cooldown-minimum-release-age)).
- Read the lockfile diff, and the changelogs of packages that change visible
  behavior (web framework, ORM, UI kit, bundler, e2e runner).

## Technical debt file

Every lint or type exception (ruff per-file ignores, lowered type rules) is
listed in a `TECHNICAL_DEBT.md`: affected files, rules, what has to be done to
remove it, priority. Intentional exceptions say so. Remove one exception per
month (a recurring chore, see [docs-backlog-changelog.md](docs-backlog-changelog.md))
so the repo moves toward strict rules with no per-file overrides.
Example: a `TECHNICAL_DEBT.md` listing the open overrides. Typing ratchet:
[quality-gate.md](quality-gate.md#strict-typing).

## Sources

- uv `required-version` makes an older uv refuse to run: <https://docs.astral.sh/uv/reference/settings/#required-version>
- uv workspace `members` globs: <https://docs.astral.sh/uv/concepts/projects/workspaces/>
- prek as a drop-in, pre-commit compatible hook manager installed with `uv tool install prek`: <https://github.com/j178/prek>
- Hook definitions (`check-added-large-files --maxkb`, `check-executables-have-shebangs`,
  `check-shebang-scripts-are-executable`, `check-merge-conflict`, `end-of-file-fixer`, `trailing-whitespace`):
  <https://github.com/pre-commit/pre-commit-hooks>
- `default_install_hook_types`, `pass_filenames`, `require_serial`, `install --overwrite` and `--hook-type`:
  <https://pre-commit.com/>
- Observed in a repo template and the mature repo built from it; no external source for the pin-bump script, dependency groups comments,
  weekly upgrade chore or `TECHNICAL_DEBT.md`.
