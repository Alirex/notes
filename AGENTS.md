# Instructions for AI agents

Personal notes in Markdown (`notes/`, saved answers in `saved/`), indexed in [README.md](README.md). No code, no
tests: the only checks are the prek hooks.

General practices live in HQ: [evocaslab/hq `practices/`](https://github.com/evocaslab/hq/tree/main/practices)
(local clone `~/dev/hq/practices`). This file is this repo's binding version; it wins where the two differ.

## Rules

- Never edit on `main`: `git fetch origin && git worktree add -b <type>/<short> .worktrees/<short> origin/main`
  (`docs/`, `chore/`, `fix/`). Push every commit right away; land when the hooks pass: merge `origin/main` into the
  branch, merge into `main` in the primary checkout, push, remove the worktree. Read-only questions need no branch.
  Details: [HQ git-worktrees-and-autoland](https://github.com/evocaslab/hq/blob/main/practices/git-worktrees-and-autoland.md).
- The gate is `prek run --all-files` (setup: `prek install`, see [README.md](README.md#dev)). markdownlint needs
  Node.js on PATH.
- Markdown is formatted by prettier and checked by markdownlint with the default rules; `.markdownlint.yaml` turns a
  rule off only with a reason on its line. Fix the finding, don't silence it:
  [HQ lint-strictness](https://github.com/evocaslab/hq/blob/main/practices/lint-strictness.md).
- A new note gets a row in the README index (`notes/<slug>/<lang>.md`, `ukr` or `eng`).
- Commits are Conventional Commits (`docs:` for notes, `chore:` for tooling), lowercase subject.
- `docs/hq-practices/` is a generated copy of the HQ practices that apply here (`just practices-sync notes` in HQ). Never
  edit it; change the page in HQ and sync again. Prettier skips it.
