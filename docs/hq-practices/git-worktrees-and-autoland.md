---
summary: "Branching, parallel agents, landing work on main"
group: agents
tags: [git, worktrees]
applies-to: [all]
reviewed: 2026-10-28
explainers: [landing-claims]
status: stable
---

# Git worktrees and auto-land

See it: [a landing claim against git](https://github.com/evocaslab/hq/blob/main/practices/explainers/landing-claims/index.html) (with a short quiz)

A workflow for one maintainer with several agents working in parallel.

## Isolation

- Never edit or commit on `main`. Each task gets a branch in its own worktree.
- Keep worktrees inside the clone (`.worktrees/<short-name>`, gitignored). IDEs
  and agent sandboxes trust the workspace root; sibling `../x` checkouts trigger
  permission prompts.

```bash
git fetch origin
git worktree add -b feat/<short> .worktrees/<short> origin/main
```

- Branch prefixes: `feat/`, `fix/`, `chore/`, `docs/`, `test/`; short kebab-case.
- One worktree per independent task; never two agents on the same files.

## Push every commit

Every commit is pushed right away (WIP included) to the work branch, so
progress survives a dead machine or container. `main` gets work only when it is
finished. Never leave local `main` ahead of `origin/main`.

## Auto-land by default

When the task is done and the full gate passed, the agent lands without asking:

1. `git fetch origin`, merge `origin/main` into the work branch (merge, not
   rebase, unless asked; merging keeps pushed history intact).
2. In the primary checkout: `git checkout main && git pull`, merge the work
   branch, `git push origin main`. When the primary checkout holds other
   sessions' uncommitted work, push from the worktree instead
   (`git push origin HEAD:main`), then still update the primary checkout:
   `git -C <primary> pull --ff-only`. Its dev servers serve those files, so
   skipping this leaves the running app stale. Never stash or reset there; if
   the fast-forward is refused, report it.
3. Remove the worktree, delete the merged branch locally and on the remote.
4. Report the commit subject + short SHA and which checks ran.

If the push is rejected because another agent landed first: fetch, merge again,
re-run the checks the new commits affect, retry.

Land only the tree that passed. The gate runs for minutes and `main` moves meanwhile: fetch right before the merge and
check `git merge-base --is-ancestor origin/main HEAD`. If it fails, merge again and re-gate; a landing that pulled in
commits nobody gated is a guess, and the only fix is a gate on `main` afterwards (observed once: the check passed, but
only after the push).

Only a full gate result lands a branch, never a local spot check: a tool missing in the agent's shell (shellcheck was not
on its PATH) means run the gate, which has it, not skip the check. A landing before the gate's last verdict puts a
rejected commit on `main` (Optiomni, 2026-11-04: `just repos-land` landed a script that failed shellcheck, a follow-up
had to land). Run one gate per worktree at a time. A branch that is behind `main` shows the newer main files as deleted
in `git diff --stat origin/main`: merge `origin/main` before judging a diff or gating.

A worktree's `node_modules` symlink (made to share the install) must stay untracked. A builder that runs `git add -A`
commits it, and a worktree with the untracked symlink makes `repos-land` print "NOT finished": check
`git ls-files web/node_modules` on a builder's branch, `git rm --cached` it, and remove the symlink before
`repos-finish`.

A merge conflict in a generated file (API revision counter and hash, schema, a version fixture) is never merged by
hand: take the other side, run the generator, commit the result. Formatters that run as the gate's first stage can
rewrite files and leave the pass unrecorded ("uncommitted changes"): commit the rewrite and run the gate once more.

## When to stop and ask instead

Show a numbered menu (PR / merge / push branch only / stop) when:

- the user asked to review first, asked for a PR, or said "don't land";
- the gate failed, was skipped, or a flaky test could not be confirmed;
- merging `main` needed non-trivial conflict resolution;
- the change is hard to undo or reaches outside the repo: data migrations or
  deletions, secrets, CI/deploy config, major dependency bumps, force pushes;
- the result only partly does what was asked, or contains a judgment call.

A numbered menu lets the user answer with one character from a phone.

When stopping, end the reply with a run command from the **feature worktree**
(`cd <repo>/.worktrees/<short> && <run>`) so the user can try the change; never
point at the primary checkout for unmerged work. After landing, give the run
command from the **primary** checkout instead. Then the menu:

```text
Ready to land — reply with a number:
1. Commit + merge latest main into work branch + push + open PR (leave open)
2. Commit + merge latest main into work branch + merge into main + update primary checkout + clean worktree
3. Commit + push the work branch only (no merge, no PR)
4. Stop (leave uncommitted / unpushed as-is)
```

Do nothing (no commit, push, PR, merge) until the user replies.

| Reply | Agent does |
| --- | --- |
| 1 | commit, fetch, merge `origin/main`, push the branch, `gh pr create`, report the URL; then update local `main` in the primary checkout (does not merge the PR) |
| 2 | commit, fetch, merge `origin/main`, merge into `main` in the primary checkout, push, remove the worktree, delete the branch; no PR |
| 3 | commit and push the work branch; no merge of main, no PR |
| 4 | nothing further |

Synonyms count: "PR" / "open PR" = 1; "merge" / "ship it" / "merge and clean" = 2;
"commit only" = 3; "stop" / "don't land" = 4. Without `gh`, option 1 pushes the
branch and gives the GitHub compare URL.

Docs-only changes need just the pre-commit hooks on the changed files, not the
full gate.

Narrow exceptions: read-only questions need no branch; "commit on this branch"
means 3; a full custom git sequence from the user wins over auto-land and the
menu.

Where a land on `main` would redeploy a live service, deploy from a fast-forward-only `prod` branch instead, so
auto-land stays safe: [releases-and-versions.md](releases-and-versions.md#a-prod-branch-for-auto-deployed-services).

Production hosts: if a dev machine doubles as production, restarts and
env/compose edits there are production changes; treat them as risky and use the
menu, not auto-land.

## Commit messages

- Conventional Commits: `type(scope): lowercase subject`; types `feat`, `fix`,
  `docs`, `test`, `chore`, `style`; scope optional (`feat(web): …`).
- The body says what changed and why, when the subject is not enough, and how it was checked (the test,
  the command, the page looked at): [change-discipline.md](change-discipline.md#commits).
- Merges of main read `Merge origin/main into <branch>`.
- Agent commits end with the `Co-Authored-By:` trailer for the model.

## Housekeeping

- After each land (and at session start) delete agent branches already merged
  into `origin/main` and prune stale worktrees; list unmerged old ones to the
  user instead of deleting them. A `git-cleanup` recipe with `--dry-run` makes
  this safe to run often.

## Skipping a re-run of the gate

Record a pass per git *tree* (not commit) in the shared git dir
(`$(git rev-parse --git-common-dir)/quality-passed/<tree>`). A merge that
produces an already-passed tree, or differs only in docs, needs no new gate.
See [quality-gate.md](quality-gate.md).

## Worktree location and cargo

A worktree created inside another repo's tree (for example HQ's `.claude/worktrees/`) sits under that repo's cargo
workspace discovery: `cargo` in a different Rust repo's worktree finds the wrong workspace root and fails. Create a
worktree for repo X inside X (or beside it), never under a different Rust repo.

## Sources

- Observed in HQ's backlog v2 work (2026-11-03): a worktree under HQ's `.claude/worktrees/` broke cargo for a
  second Rust repo.
- Observed in the JAS Bot passkey landing (2026-11-01): a generated-file conflict, a formatter rewrite after the gate
  and a `main` that moved during a seven-minute gate.
- Observed in Optiomni's family tree and icon landings (2026-11-04): a landing before the gate verdict, a committed
  `node_modules` symlink, a branch behind `main` read as deleted files.
- `git worktree add -b`, linked worktrees, `remove`, `prune`: <https://git-scm.com/docs/git-worktree>
- `git rev-parse --git-common-dir` (the shared git dir for the gate-pass records): <https://git-scm.com/docs/git-rev-parse>
- Commit message format `type(scope): description`, body and footers: <https://www.conventionalcommits.org/en/v1.0.0/>
- Observed in HQ's own repos and sessions (auto-land, push-every-commit, the numbered menu); no external source for
  those workflow rules.
