---
summary: "One recipe vocabulary in every repo: core names, groups, customising, distribution, drift check, aliases"
group: quality
tags: [just, recipes, repo-template, conventions]
applies-to: [all]
reviewed: 2026-11-01
status: draft
---

# Unified just recipe vocabulary

[just-task-runner.md](just-task-runner.md) says how a justfile is laid out; this page fixes **which recipes exist under which name**, so
`just q-check` or `just setup-check` means the same in every repo and an agent never has to read a justfile first.

## State found

On 2026-11-01 six repos used the same module split but five different names for the same action (tests: `q-test`, `w-test`, `web-test`, none; web: `w-` vs `web-`; docker: `dk-`, `docker-`, `app-`) and only two shared the modules byte for byte; the per-repo table is in [the survey](https://github.com/evocaslab/hq/blob/main/docs/reviews/2026-11-01-just-recipe-survey.md).

## Core vocabulary

Every repo provides these names, in the group shown, with the meaning below. Names follow what most repos already use; new are `q-test`, `dev-run` and `dev-build`, and `q-lint`/`q-types` where a repo lacks them.

| Group | Recipe | Guarantee |
| --- | --- | --- |
| quality | `q-check` | The repo's **one** gate: fixers, then every check and test; exit 0 only if all pass; records a pass for a clean tree (`q-passed`). Same scope as CI. |
| quality | `q-fast` | The edit loop: fixers and checks on what changed, no full test run. Never a substitute for `q-check`. |
| quality | `q-test` | Tests only, no fixers, no rewriting of files; exit code = test result. |
| quality | `q-lint` | Formatter plus linter with fixers (may rewrite files; review `git diff`). There is no separate `q-fmt`: every repo's formatter runs inside `q-lint`. |
| quality | `q-types` | The authoritative type check, read-only. |
| quality | `q-passed`, `q-integrity`, `q-secrets` | Exit 0 if the clean tree passed `q-check`; flag a diff that weakens a gate; scan history for secrets. |
| setup | `setup-all`, `setup-check`, `setup-hooks`, `setup-just`, `setup-shortcuts`, `setup-upgrade` | See [just-task-runner.md](just-task-runner.md#setup-recipes); `setup-check` is read-only and exits 1 when a required item is missing; all idempotent. |
| docs | `docs-backlog`, `docs-backlog-check`, `docs-questions`, `docs-questions-ask`, `docs-actions`, `docs-actions-add`, `docs-chores` | Backlog, maintainer questions and manual actions; checks exit 1 on a bad item. |
| claude | `cc-1`, `cc-2`, `cc-up`, `cc-down`, `cc-ls`, `cc-rc`, `cc-rc-down`, `cc-rc-autostart` (+ `-off`, `-ls`) | Claude Code sessions in tmux. |
| git | `git-cleanup` | Branch housekeeping; dry run unless told otherwise. |
| release | `release-check` | Read-only: is the tree releasable (version, changelog, clean tree)? Publishing recipes are repo-specific and take `confirm` ([outward-facing recipes](just-task-runner.md#outward-facing-recipes)). |
| dev | `dev-run` | Start the repo's app for development; delegates to the repo's own recipe (`web-dev`, `app-dev`, `rust-serve`). |
| dev | `dev-build` | Build what the repo ships; delegates (`web-build`, `dk-build`, …). |

Rules for every core recipe: the name and group are fixed; it never prompts; extra arguments are passed through
(`*args`); it is cheap to call twice. A repo with no such area keeps the name and prints `n/a: <reason>` with exit 0
(an explicit stub, never a missing recipe), so a script written for one repo runs in all.

### Conditional vocabulary

Names fixed for repos that have the area (not required elsewhere): `db-upgrade`, `db-revision`, `db-check`; `dk-up`, `dk-down`, `dk-logs`,
`dk-build`, `dk-update`, `dk-autoupdate*` (prefix `dk-`, group `docker`, never `docker-` or `app-`); `web-dev`, `web-build`, `web-test`, `web-check`,
`web-install` (prefix `web-`, group `web`, never `w-`); `g-` desktop GUI, `rust-` Rust tooling, `vm-` dev VM, `as-` assets, `llm-`. The
typical-prefix list in [just-task-runner.md](just-task-runner.md#layout) is amended to match.

## Customising

- **Own modules** with their own prefix and group (`machine-`, `practices-`, `app-`): free, listed in AGENTS.md's table.
- **Core recipe delegates**: `dev-run` and `dev-build` live in the repo's `just/dev.just` and call the specific recipe; `q-*` in
  `just/quality.just` call `scripts/quality/gate.sh`.
- **Extra recipes** in a core prefix (`q-clippy`, `q-mutants`, `release-tag`) are fine; they never reuse a core name with another meaning.
- **Never**: rename or regroup a core recipe, change its guarantee, give `q-check` a different scope from CI, or make `q-fast` pass where
  `q-check` would fail on the same tree *without* saying so in its comment.
- Shared modules are not edited in a repo (next section); a different behaviour goes into the repo's own module.

## Distribution

The template repo (`repo-template`) is the source ([library layout](shared-libraries.md#library-repo-layout); B-261101-library-repo-template adds the variants).

| Files | Owner |
| --- | --- |
| `just/{claude,git,docs,setup,release}.just`, `scripts/{claude,git,docs,setup,release}/` | **Shared**: copied byte for byte, never edited in the repo |
| `justfile`, `just/quality.just`, `just/dev.just`, `scripts/quality/gate.sh` | **Skeleton**: the repo owns the content, the template defines the core names |
| everything else | the repo |

A repo records its base in `.repo-template.yaml` (`ref:` template tag or commit, `shared:` the file list). `scripts/quality/recipe-vocabulary.py`
(a uv script from the template) runs two checks, both in the `local` prek hook `just-vocabulary` and as `just q-vocabulary`:

1. **Vocabulary**: `just --dump --dump-format json` against `just/core-recipes.yaml` (name → group, from the template): fails when a core
   recipe is missing or its group differs, naming each one.
2. **Drift**: the shared files' hashes against the template at `ref:`; reports a changed or missing shared file. `--update` copies the template's
   files in and bumps `ref:`; it never commits.

HQ reports across the registry with `just repos-vocabulary` (read-only; the same two checks run in each clone), so a template change shows as a list of
repos to sync. This is the same idea as `practices-sync`, applied to tooling instead of pages.

## Migration with aliases

A rename keeps the old name for one release as a `just` alias in `just/aliases.just` (imported by `justfile`):

```just
# remove after 2026-12-01 (renamed to web-dev)
alias w-dev := web-dev
```

- Alias, update README/AGENTS/docs/scripts/CI in the same change, then delete the alias at its date; the vocabulary check ignores aliases
  and warns on one past its `remove after` date.
- Agents' notes and memory are fixed in the same change (grep `.claude/`, `AGENTS.md`, `docs/`, hooks, systemd units, `package.json`).
- Aliases are not core recipes: a core name must be a real recipe.

## Sources

- just manual, `alias`, `import`, `[group]`, `--dump --dump-format json`: <https://just.systems/man/en/>
- Observed: the justfiles of the managed repos on 2026-11-01 (`just --summary`, file hashes): the survey above.
- The core names, the `n/a` stub and the drift check have no external source; they follow the repos' existing majority.
