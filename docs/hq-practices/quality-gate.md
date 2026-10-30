---
summary: "Lint/type/test gates, heavy-check locking, coverage on changed lines"
group: quality
tags: [gate, hooks]
applies-to: [all]
reviewed: 2026-10-28
status: stable
---

# Quality gate

One command that decides "done": `just q-check` (or the repo's equivalent).

## Shape

- **Fix, then check.** Run the fixers first (formatter, safe lint fixes,
  end-of-file/whitespace hooks), then every check in check mode, so only what a
  fixer cannot fix fails. List the files the fixers changed at the end ("commit
  these") without failing.
- **Unsafe fixes stay out of the gate.** Run them by hand and review the diff.
- **Stop at the first failing stage**, print a per-stage summary
  (time / ok / fail / skipped) and keep each stage's log in a gitignored
  `.quality/logs/`.
- **A fast loop for editing** (`q-fast`): fixers, lint, a fast type checker on
  changed files, only the tests affected by the change (pytest-testmon or
  similar). The full gate still runs before claiming done. Testmon selects by
  Python code a test executed: a change to a widely imported module re-runs
  most tests, and non-Python inputs (SQL, migrations, JSON fixtures, env) may
  not be tracked, so after editing those run their test directory in full. Its
  `.testmondata` is local state: gitignored, per worktree, rebuilt after a big
  branch switch; never with coverage or xdist in the same run. A test that
  passes in the full run but fails when selected alone is coupled to another
  test's state.
- **Two type checkers.** `ty` (Rust) gives quick feedback in the fast loop;
  basedpyright is the authority and runs in the gate and as a pre-push hook.
- **Parallelism differs by stage.** The full gate runs pytest on all cores
  (`-n auto`, pytest-xdist); the testmon fast loop stays single-process.
- **Overlap the light stages; keep the critical path short.** Type checks,
  compiled-language checks and unit tests that need no heavy lock run in the
  background while the frontend checks run; each reports its own time. A slow
  single-core check that nothing later depends on (a contract recorder, a
  generator freshness check) starts with them but is collected only after the
  heavy stages, under `nice`, so the test suites never wait for it. Such a
  check must not rewrite files the tests read meanwhile: it writes into a
  scratch copy and diffs. A background stage is stopped with its children when
  the gate fails or exits. Measure before and after (`q-timings`). Example:
  `report-bot/scripts/quality/gate.sh` (`contract` stage: wall time 546 s ->
  270 s).
- **The same gate everywhere.** CI runs the same `just q-check`, then
  `git diff --exit-code` (a diff means someone skipped the fixer hooks). It needs
  `fetch-depth: 0` (the changed-line coverage diffs against `origin/main`) and
  `concurrency` with `cancel-in-progress` so a newer push replaces a running gate.
  Example: `repo-template/.github/workflow-drafts/ci.yml`. A tool that
  shells out to platform binaries (uv, git) also runs its tests in a
  `macos-latest` / `windows-latest` matrix job, `fail-fast: false`, so one
  platform's failure does not hide the other's. Example: `uv_upgrade/.github/workflows/ci.yml`.

## Strict typing

Fix, don't silence; strictest mode per tool: [lint-strictness.md](lint-strictness.md).

- Python: basedpyright, TypeScript: `strict`.
- **New repo: `typeCheckingMode = "all"`**, and lower single rules to `warning`
  or off in commented regions of `[tool.basedpyright]` (each with a reason).
  Example: `repo-template/pyproject.toml`.
- **Existing repo: the ratchet.** `typeCheckingMode = "recommended"` with
  `failOnWarnings = true`, plus a per-package `strict = [...]` list. A new
  package joins `strict` (with `py.typed`) from day one; old packages join one
  by one when their errors reach 0. Keep a checklist table (package, strict,
  errors, next step). Example: `report-bot/docs/monorepo/TYPING_STRICT.md`.
- **Escape hatches need a reason.** A guard script fails a `cast`, a
  `# type: ignore` / `# pyright: ignore`, or a TypeScript `as X` cast with no
  comment on its line or just above. `@ts-ignore` / `@ts-nocheck` are banned.
  Example: `report-bot/scripts/quality/check_type_hatches.py`.

## Heavy checks share the machine

Full test suites and browser e2e each use every core. Running several at once
(parallel agents) finishes no sooner and makes e2e flaky.

- Wrap heavy stages in a lock script (`heavy-lock.sh <cmd>`) with a FIFO queue,
  so parallel agents wait their turn. One lock per machine, not per repo or clone
  ([machine-capacity.md](https://github.com/evocaslab/hq/blob/main/practices/machine-capacity.md)): HQ's is `scripts/machine/heavy-lock.sh`. Show the holder, stage, worktree and
  expected finish (`q-lock`).
- A landing run may jump the queue (`GATE_PRIORITY=land`).
- If HEAD moves while a stage waits, stop as "superseded" (exit code 75, so
  callers can tell it from a failure) instead of testing stale code.
- Never nest the lock (a gate already holds it) and never kill processes by
  pattern; kill only the PIDs you started.

## Pass record

On a fully green run on a clean tree, record the tree hash in the shared git
dir so every worktree sees it. Landing can skip the gate when the tree to land
has a pass record, or differs from one only in docs (then run just the doc
hooks).

## Coverage on changed lines

Total coverage hides untested new code. Report coverage of the lines changed vs
`origin/main` and fail below a threshold, skipping tiny diffs. One report
covers Python and the frontend together (diff-cover).

- Knobs (env vars): `COVERAGE_DIFF_MIN` (default 80, `0` disables),
  `COVERAGE_DIFF_MIN_LINES` (default 20: smaller diffs skip the check),
  `COVERAGE_DROP_MAX` (points of total coverage; unset means a drop only warns).
- History: a green full gate appends the totals (Python, frontend, date,
  commit, branch) to `<git-common-dir>/quality-coverage/history.tsv`, shared by
  all worktrees and never committed. The report compares them with the last
  entry whose commit is already in `origin/main` and warns on a drop.
- Total coverage is printed but fails nothing.

## Flaky tests

Retry e2e once; a test that passed only on retry is reported as flaky and
written to a FLAKY_TESTS doc with the fix or the reason it stays. Fix flakes by
waiting for a visible state, never with fixed sleeps. Format and e2e rules:
[testing.md](testing.md).

Choose the e2e worker count by measuring, not by guessing: compare wall time and
the sum of test durations per count, and keep the lower count when more workers
are no faster and flakier. Example: `report-bot/docs/monorepo/E2E_SPEED.md`
(3 workers on 4 cores).

## Mutation testing

Out of the gate (slow, uses every core): run it on the core parsers now and
then, through the heavy lock. Procedure: [testing.md](testing.md).

## Tool versions

Pin each tool one way and make a test enforce it: uv `required-version` in the
root `pyproject.toml` is the source (CI follows it), and a test fails when any
other pin (pre-commit hook, Dockerfile, build backend) disagrees. Details:
[tooling-and-deps.md](tooling-and-deps.md).
