---
summary: "Lint/type/test gates, heavy-check locking, coverage on changed lines"
group: quality
tags: [gate, hooks]
applies-to: [all]
reviewed: 2026-11-04
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
  the gate fails or exits. Measure before and after (`q-timings`). Example: a `contract` stage whose wall time went from 546 s to
  270 s.
- **The same gate everywhere.** CI runs the same `just q-check`, then
  `git diff --exit-code` (a diff means someone skipped the fixer hooks). It needs
  `fetch-depth: 0` (the changed-line coverage diffs against `origin/main`) and
  `concurrency` with `cancel-in-progress` so a newer push replaces a running gate.
  Example: `repo-template/.github/workflow-drafts/ci.yml`. A tool that
  shells out to platform binaries (uv, git) also runs its tests in a
  `macos-latest` / `windows-latest` matrix job, `fail-fast: false`, so one
  platform's failure does not hide the other's.

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
  errors, next step).
- **Escape hatches need a reason.** A guard script fails a `cast`, a
  `# type: ignore` / `# pyright: ignore`, or a TypeScript `as X` cast with no
  comment on its line or just above. `@ts-ignore` / `@ts-nocheck` are banned.

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

## Every step ran, or the gate is not green

A green summary only counts when every hook ran. Optiomni landed an icon fix before its shellcheck step passed (2026-11-04): the
session's shell had no `shellcheck`, so the step did not run and nothing said so. Rules:

- **A missing tool fails the gate**, it never skips the step quietly. The runner checks each hook's tool up front and
  prints `MISSING: shellcheck` as a failure; the summary line says `N of N hooks ran`, and a landing reads that line.
- **Session shells carry the gate tools**: the setup check (`just setup-check` in HQ) lists them, and a worker's brief
  names the tools it needs. A "skipped" count above 0 on a landing run is a stop, not a note.
- **Land only after a full PASS on the merged tree** (stage 6 of [agent-workflow.md](agent-workflow.md#stages)): an
  earlier green on the branch, or a green with skipped steps, is not it.

## Hooks that run next to other gates

Three lessons from a gate-notification change that failed in a hook while a second worktree's gate ran:

- **A hook runs with git's variables set.** `GIT_DIR`, `GIT_INDEX_FILE`, `GIT_WORK_TREE` and `GIT_PREFIX` are in the
  environment of every hook command, so a test in the hook that calls `git` acts on the repository being committed.
  Tests and the helpers they use clear every `GIT_*` key before spawning git, with a guard test
  ([testing.md](testing.md#protect-real-data)).
- **Format before committing, not in the hook.** A hook that runs a formatter rewrites files after the gate has
  passed, so the commit holds a tree the gate never saw (or the commit fails and the rerun is a new gate). Run the
  fixers first (`Fix, then check` above), stage their result, then commit; a hook that still rewrites something is a
  failure to read, not to retry.
- **No fixed port in a test server.** Two gates in different worktrees both wanted `localhost:5199` and the second
  died with "is already used". Let the hook wrapper take a free port from the OS (listen on port 0), pass it to the
  config in an environment variable, and keep the old default for a manual run. Check it once by starting two runs at
  the same time. Same family as the throwaway-container ports in [testing.md](testing.md#throwaway-service-containers).

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
are no faster and flakier.
Example: 3 workers on 4 cores.

## Checks that parse source by regex

A check that reads source files by pattern (the app name from the i18n file, a version from a manifest) breaks on an
innocent new key that matches the pattern: Optiomni's `project-check` read every indented `name:` line of `en.ts` as the app
name, and a new `familyTree.name` key failed the gate (2026-11-04). Anchor the pattern to the structure (parse the
file, or read one named key), and when it cannot be fixed at once, say in the check's message which lines it read.

## Mutation testing

Out of the gate (slow, uses every core): run it on the core parsers now and
then, through the heavy lock. Procedure: [testing.md](testing.md).

## Tool versions

Pin each tool one way and make a test enforce it: uv `required-version` in the
root `pyproject.toml` is the source (CI follows it), and a test fails when any
other pin (pre-commit hook, Dockerfile, build backend) disagrees. Details:
[tooling-and-deps.md](tooling-and-deps.md).

See also [continuous-improvement.md](continuous-improvement.md): gates fail only on noise-free numbers; slowest-N and quarantine caps.

## Sources

- basedpyright `typeCheckingMode` (`recommended`, `all`), `failOnWarnings`, per-path `strict`:
  <https://docs.basedpyright.com/latest/configuration/config-files/>
- pytest-xdist `-n auto` (physical cores; `-n logical` for logical): <https://pytest-xdist.readthedocs.io/en/stable/distribution.html>
- pytest-testmon selects the tests affected by changes and keeps `.testmondata`: <https://github.com/tarpas/pytest-testmon>
- diff-cover: coverage of changed lines against `origin/main`, `--fail-under`, several reports combined:
  <https://github.com/Bachmann1234/diff_cover>
- GitHub Actions `concurrency` and `cancel-in-progress`: <https://docs.github.com/en/actions/using-jobs/using-concurrency>
- `actions/checkout` `fetch-depth: 0` fetches full history: <https://github.com/actions/checkout>
- Playwright retries and the "flaky" outcome: <https://playwright.dev/docs/test-retries>
- Git hooks export `GIT_DIR` and related variables to hook commands: <https://git-scm.com/docs/githooks> and
  <https://git-scm.com/docs/git#_git_environment_variables>; Playwright `webServer` (`port`/`url`, `reuseExistingServer`):
  <https://playwright.dev/docs/test-webserver>; Node `net.Server.listen(0)` picks an unused port:
  <https://nodejs.org/api/net.html#serverlistenport-host-backlog-callback>
- Gate step skipped for a missing tool: observed in Optiomni (2026-11-04); no external source.
- Observed in HQ's and another managed repo's own gates (`scripts/quality/gate.sh`, the 546 s to
  270 s timing); the stage order, heavy lock, pass record and the thresholds are their own design with no external
  source.
