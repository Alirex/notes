---
summary: "Test-first, fixtures, guard tests, e2e, flaky tests, parity tests, throwaway service containers"
group: quality
tags: [tests]
applies-to: [all]
reviewed: 2026-10-30
status: stable
---

# Testing

How tests are set up so they are fast, safe for real data, and catch more than
line coverage. The gate that runs them: [quality-gate.md](quality-gate.md).
Examples are from `report-bot` (`conftest.py`, `pyproject.toml`, `docs/monorepo/`).

## Test-first for behavior changes

- A bug fix starts with a test that reproduces the bug; a new behavior starts with its test. Run it and see it fail
  for the expected reason before changing the code; a test that never failed proves nothing.
- Then the smallest change that makes it pass, then refactor while it stays green.
- A test drives the code the way its user does (an HTTP call, a CLI run, a public function) and checks what that
  user sees. A test that would still pass with the function bodies emptied is deleted, not kept for coverage;
  mutation testing (below) finds them mechanically.
- When an honest test is impractical (real timing, an external service), say so in the commit message; do not
  write a test that only exercises mocks.
- A guard that only hides a crash (an extra `try`, a `None` check, a skipped test) is not a fix: reproduce it,
  then fix the cause ([lint-strictness.md](lint-strictness.md)).
- The order of work around it (end state, small green units, when to stop patching):
  [change-discipline.md](change-discipline.md).

## Pytest setup

- `addopts = "--strict-markers"`: a typo in a marker fails instead of silently
  selecting nothing. Declare every marker with a description.
- A `slow` marker for tests that take seconds (real migrations); skip with
  `-m "not slow"` (note: this disables testmon selection).
- `tmp_path_retention_policy = "failed"`: keep temp dirs of failed tests only
  (database copies add up to ~1 GB per passing run otherwise).
- `testpaths` lists the test dirs, so collection skips `reference/`, `temp/`,
  build output.
- Coverage: `branch = true`, `source` = the package src dirs, `omit` migration
  revisions, reports under `.quality/`; exclude `if TYPE_CHECKING:`,
  `raise NotImplementedError` and bare `...` bodies.

## Protect real data

- A root `conftest.py` sets the data/cache/log env overrides and redirects the
  `platformdirs` defaults to a throwaway session dir in `pytest_configure`.
- It also wraps the path resolvers so a path inside the real platform dir raises
  a dedicated error (`RealDataDirError`) instead of touching it. This catches
  stragglers such as a worker thread that outlives its test.
- Delete the throwaway dir in `pytest_unconfigure`.
- **Tests that spawn `git` strip git's own variables first.** A git hook
  (pre-commit, pre-push, prek) exports `GIT_DIR`, `GIT_INDEX_FILE`,
  `GIT_WORK_TREE`, `GIT_PREFIX` and more to every process it starts, and a
  child `git` obeys them over its working directory. A suite run by a commit
  hook that does `git init` or `git config` in a temp dir then edits the
  repository being committed to (it once set `core.bare = true` on a real
  clone). The fixture helper and the code under test build the subprocess env
  without any `GIT_*` key (`{k: v for k, v in os.environ.items() if not
  k.startswith("GIT_")}`, `env_remove` in Rust) and set only what they mean
  (`GIT_CONFIG_GLOBAL=/dev/null`, `GIT_TERMINAL_PROMPT=0`, `HOME` to the temp
  dir). A guard test runs the helper with `GIT_DIR` pointing at a second temp
  repository and asserts nothing there changed.

## Fast database tests

- Run migrations once per session into a template DB (built under a temporary
  name, renamed when complete, behind a lock). Each test's DB is a file copy of
  the template.
- A `real_migrations` marker keeps the real upgrade for the few tests about
  migrations themselves (also mark them `slow`).
- Cheap password hashing: swap the production hasher (Argon2 allocates 64 MB per
  hash) for its cheapest profile; keep the same algorithm.
- SQLite without fsync in tests: an `Engine` `connect` listener runs
  `PRAGMA synchronous=OFF`.

- PostgreSQL and other servers: [throwaway service containers](#throwaway-service-containers), a database per
  test cloned from a migrated template.

## Throwaway service containers

Integration tests run against the real service (PostgreSQL, Redis, a routing engine, an S3 stand-in) in a
container the test run starts and owns, never against a shared or production instance, and never against a
look-alike (SQLite in place of PostgreSQL). Same image and major version as production. `report-bot` since
2026-10-26: 6,862 pytest tests in 33 s on one container with xdist, a template clone per test.

| Choice | Rule | Why |
| --- | --- | --- |
| Scope | **One container per test run** (session), shared by the xdist workers and by every stage of the gate (pytest, e2e, Rust) through one URL variable; never one per test | A start costs 1–3 s; a database clone costs milliseconds |
| Isolation | **A database per test, cloned from a migrated template** (`CREATE DATABASE t_<id> TEMPLATE tpl`); a transaction rollback only for pure query tests | Rollback never reaches a real `COMMIT` (see below) |
| Reuse | `<APP>__TEST_DATABASE_URL` names a server to reuse; nothing is started then | A dev server, a CI service container, a gate that shares one server |
| Speed | Data dir on a sized tmpfs (`--tmpfs /var/lib/postgresql:rw,size=2g`) plus `--memory` above it, so a runaway test hits a limit, not the host; `fsync=off`, `synchronous_commit=off`, `full_page_writes=off`, `initdb --no-sync`; `max_connections` above the parallelism | Durability means nothing for a database that dies with the run |
| Ports | Publish on `127.0.0.1` with a random host port (`-p 127.0.0.1::5432`), read it back with `docker port` | Parallel runs and worktrees never collide; nothing listens on the LAN |
| Name | `<app>-test-<service>-<random>`; distinct from every production container | `docker stop` on a test name can never hit `jasbot-postgres` |
| Image | Pin tag **and digest** (`postgres:18-alpine@sha256:…`), read from the compose file the updater already watches, or matched by a Renovate regex manager (Dependabot does not see an image in a code constant) | A tag moves; a digest is the bytes the last green run used ([supply-chain-security.md](supply-chain-security.md)) |
| Labels | `hq.test=1`, `hq.test.owner=<boot-id>/<pid>`, `hq.test.host=<hostname>`, `hq.test.expires=<unix time>` (start + the longest sane run), plus the app's own label. The boot id is Linux's `/proc/sys/kernel/random/boot_id` stripped, macOS's `sec` from `sysctl -n kern.boottime`. The pid is the **long-lived owner** (the pytest controller's `os.getpid()`, the gate's `$$`, e2e global setup's `process.pid`), passed to the start helper as `--owner-pid`, never the helper CLI's own pid (`uv run` does not exec, so even its parent is uv) | A reaper can tell a live run's container from a leftover without asking anyone |
| Cleanup | `--rm`, stop in the session teardown and in the gate's EXIT trap, **plus a watcher that outlives the run** (below) | Teardown and traps never run on SIGKILL, an OOM kill or a killed terminal |
| Leftovers | Each start first reaps this app's stale containers; a `just <prefix>-test-containers` recipe (`ls`, `reap`) for the rest | So leftovers can't pile up (`report-bot`, 2026-10-27: five, one with 73 orphaned databases) |
| No Docker | No URL and no `docker`: **fail** with the fix in the message (start Docker, or set the URL). A suite where only a few tests need the service may `skip` those with that reason locally, never in CI (`CI` set: fail) | A silent skip turns a missing service into a green run ([lint-strictness.md](lint-strictness.md)) |
| CI | The same code path: prefer Docker on the runner (the same `docker run` and flags); a `services:` container cannot take `-c` server flags or a tmpfs, so use it only through the reuse variable when the defaults are fine | One way to get a server, so CI tests what runs locally |

### Cleanup that survives a kill

`--rm` removes a container once it exits (or the daemon exits), and the run's own teardown stops it, but neither
happens when the test process is killed: a `kill -9`, the OOM killer, a timed-out agent command, a closed tmux pane. The container then
runs until someone notices. Three layers, each cheap:

1. **In-process teardown**: `pytest_unconfigure` (controller only, not xdist workers), `Drop` in Rust, the gate's
   `trap … EXIT` plus `trap 'exit 143' TERM INT` so a signal still runs the EXIT trap. Start the container in the
   shell that owns the trap, never inside a `$(…)` subshell, whose variables the trap can't see.
2. **A watcher**: right after `docker run`, start a small detached process in its own session (`setsid`,
   `start_new_session=True`) that waits for the owner pid to exit and then runs `docker stop`. Give it `stdin`, `stdout` and `stderr` on
   `/dev/null` and close the other fds, or a caller reading the helper's output (`$(…)`, `execFileSync`) waits for
   the watcher forever; keep `DOCKER_HOST` and `DOCKER_CONTEXT` in its environment; and let it stop the container
   at `hq.test.expires` too, since a pid can be reused. It survives the kill
   of the test process and its process group. This is what testcontainers' Ryuk does, but on the host: Ryuk is a
   container with the Docker socket mounted, which [host-security.md](https://github.com/evocaslab/hq/blob/main/practices/host-security.md) rules out on our hosts, and
   it is not there in Rust's `testcontainers` (it removes containers on `Drop`, and on SIGINT/SIGTERM/SIGQUIT only
   with its `watchdog` feature; SIGKILL leaks).
3. **A sweep by label**: on every start, and on demand, remove containers whose owner pid is gone, whose owner
   ran before the last boot, or that are past `hq.test.expires` (a hard ceiling, even with a live owner). Owner and
   boot are judged only when `hq.test.host` is this machine (a shared or remote daemon); otherwise, and without an
   owner label, by expiry and age alone.
   HQ's `just machine-test-containers ls|reap` does this for every repo on the machine
   (`scripts/machine/test-containers.py`).

### Databases per test

- **Template database.** Migrate once into a template, then `CREATE DATABASE t_<id> TEMPLATE tpl` per test (or
  per worker). The clone fails while anyone is connected to the template: close every connection to it and never
  test on it directly. A clone copies files, so keep the template schema-only plus a minimal seed. Keep the default
  `STRATEGY = WAL_LOG`, the efficient one for a small template; `FILE_COPY` forces a checkpoint before and after
  each clone. Prior art: IntegreSQL (a pool of ready clones per template, behind a small HTTP API).
- **Drop what the test created** when it ends, so a long run does not fill the server; a killed run's databases go
  with its container.
- **Rollback per test or a database per test.** Rolling back a transaction is cheapest but never reaches a real
  `COMMIT`: it hides commit, deferred constraint and transaction-boundary bugs and breaks code that opens its own
  connections or threads. Use it only for pure query tests; anything about transactions gets a cloned database.
  No reliable speed numbers exist for this trade-off: measure on your schema.
- **Per-test schemas** (`SET search_path`) are a third option for tools that cannot switch databases; they share
  one catalog and leak through anything schema-qualified, so prefer databases.
- **Rust (sqlx).** `#[sqlx::test]` gives each test a fresh database and applies `migrations/`
  (`migrations = false` to skip, `fixtures("name")` for SQL fixtures, run in the given order, so order them by
  foreign key). It needs a superuser `DATABASE_URL` and the `migrate` feature; a failed test keeps its database
  for debugging. All test pools share the server's `max_connections`, so keep it above the test parallelism. When
  the migrations are slow, clone a template instead (`report-bot`: `JASBOT_TEST_TEMPLATE_URL`) and keep one test
  that runs the real migrations.

### Libraries

testcontainers (Python, Rust, Go, Java, Node) gives the same container-per-run with a fluent API. Use it when it
fits, with Ryuk off on hosts that forbid the Docker socket in containers (`TESTCONTAINERS_RYUK_DISABLED=true`) and
the watcher and sweep above in its place. A plain `docker run` helper of about 100 lines (`report-bot`:
`packages/jas-bot/src/jas_bot/db/testing.py`) is just as good and has no daemon-side parts; either way, keep the
rules in the table.

### Tests that pin it

- The helper's own tests: the stale rule (owner gone, earlier boot, expired, no owner label by age, another host;
  a live owner is kept until its expiry), the run flags (tmpfs, fsync off, loopback port, labels), and a missing `docker` failing with
  the fix in the message. HQ: `scripts/machine/test_test_containers.py`.
- A guard test that the test server URL is never the production one (port, container name, database name).
- A kill test: start the run from a child process, `kill -9` the **owner** (the pytest or gate pid named in
  the label, not the `uv` wrapper), assert the container is gone within a few seconds (the watcher).

Sources: [docker run `--rm`, `--tmpfs`](https://docs.docker.com/reference/cli/docker/container/run/),
[tmpfs mounts](https://docs.docker.com/engine/storage/tmpfs/) (counts against the memory limit),
[`docker ps` filters](https://docs.docker.com/reference/cli/docker/container/ls/),
[`docker container prune`](https://docs.docker.com/reference/cli/docker/container/prune/) (stopped only, `label` and
`until` filters), [pull by digest](https://docs.docker.com/reference/cli/docker/image/pull/);
PostgreSQL 18 [WAL settings](https://www.postgresql.org/docs/18/runtime-config-wal.html) (`fsync`,
`synchronous_commit`, `full_page_writes`), [CREATE DATABASE](https://www.postgresql.org/docs/18/sql-createdatabase.html)
(template, `STRATEGY`), [initdb `--no-sync`](https://www.postgresql.org/docs/18/app-initdb.html);
[Ryuk](https://github.com/testcontainers/moby-ryuk) (`RYUK_RECONNECTION_TIMEOUT` 10 s, mounts the Docker socket),
[testcontainers configuration](https://golang.testcontainers.org/features/configuration/) (`TESTCONTAINERS_RYUK_DISABLED`),
[testcontainers-python](https://testcontainers-python.readthedocs.io/en/latest/modules/postgres/README.html) (Ryuk on by
default), [testcontainers-rs](https://docs.rs/testcontainers/latest/testcontainers/) (`Drop`, `watchdog` feature),
[IntegreSQL](https://github.com/allaboutapps/integresql),
[GitHub Actions PostgreSQL service](https://docs.github.com/en/actions/using-containerized-services/creating-postgresql-service-containers),
[pytest-xdist `worker_id` / `testrun_uid`](https://pytest-xdist.readthedocs.io/en/stable/how-to.html).
Examples: `report-bot` `packages/jas-bot/src/jas_bot/db/testing.py`, root `conftest.py`, `scripts/quality/gate.sh`;
HQ `scripts/machine/test-containers.py`.

## Guard tests over prose

A rule written only in a doc gets forgotten; a test that walks the code does not.
Prefer a small test that fails with the offending names. Kinds used in
`report-bot`:

- **Route table walk (auth).** Read the live route table; every route must be in
  a `PUBLIC` or `SETUP` allow-list or reject an anonymous / no-2FA session.
  (`apps/jasbot-web/tests/test_two_factor_api.py`)
- **OpenAPI shape.** Every `operationId` is `<tag>_<route_name>`; every path sits
  under the API prefix. (`test_routing.py`)
- **Nav and permission sync.** Frontend nav items and router routes declare the
  same permission; every permission-gated page has a nav item.
  (`web/jasbot/src/nav/items.spec.ts`)
- **Icon registry.** Every icon name found in the sources is registered, via a
  raw-text glob of the source files. (`web/jasbot/src/plugins/icons.spec.ts`)
- **Backlog files.** Front matter and ids of the backlog dir are valid
  (`tests/test_docs_backlog.py`; see [docs-backlog-changelog.md](docs-backlog-changelog.md)).
- **Tool pins.** Every uv pin equals `required-version`
  (`tests/test_uv_version_pins.py`).
- **Migration guardrails.** One linear head, contiguous numbers matching revision
  ids, migrations import only alembic/sqlalchemy/stdlib, and models match the
  migrations (`compare_metadata`). (`test_migration_guardrails.py`)
  The comparison (and `alembic check`) sees only what the models declare. It
  sees tables, columns, nullability, types, plain indexes and foreign keys;
  server defaults only with `compare_server_default=True`; it does not see
  renames (drop + add), anonymous constraints, CHECK expressions or partial
  index predicates. So: declare every index and constraint in the model
  (`__table_args__`, `Index(..., unique=True, postgresql_where=text(...))`),
  never only in a migration or a comment (`report-bot` lost two partial unique
  indexes that way in a new PostgreSQL baseline; a review caught it), and name
  them all through a naming convention on `MetaData`. Prove the check works with
  a negative test (change a temp copy of the metadata, expect a diff). A second
  line of defence on PostgreSQL: compare `pg_indexes` / `pg_constraint` of the
  migrated database with one created from the models (normalize the predicate
  text first, `pg_get_expr` rewrites it).
- **Escape hatches.** The type-hatch checker has its own tests
  (`tests/test_check_type_hatches.py`).
- **Ratchet counts** (designed, not yet in use there): the number of old call
  sites may only go down; the baseline is in the test and each step lowers it.
  See the async migration in [python-style.md](https://github.com/evocaslab/hq/blob/main/practices/python-style.md#async).

## Time, concurrency and flaky tests

- **Inject the clock.** Code that waits, expires, retries or schedules takes a clock (a parameter, a trait, a
  `now()` seam) that tests set and advance; a test never `sleep`s for real time to see a backoff, a lease expiry
  or a session timeout ([jobs-and-integrations.md](https://github.com/evocaslab/hq/blob/main/practices/jobs-and-integrations.md)). Date and zone rules:
  [datetime-timezones.md](https://github.com/evocaslab/hq/blob/main/practices/datetime-timezones.md).
- **Concurrency is tested by firing the race, not by hoping.** Start N tasks at a barrier against one database,
  assert one winner and no duplicate rows ([database-integrity.md](https://github.com/evocaslab/hq/blob/main/practices/database-integrity.md)); run it against the real
  engine (PostgreSQL container, SQLite file), since in-memory fakes hide locking.
- **Seed or fix every source of randomness** (ids, shuffles, jitter) that an assertion depends on; log the seed
  on failure.
- **A flaky test is a bug, not noise.** On a failure that passes on rerun: do not just rerun. Find the shared
  state, clock or ordering, fix it, or quarantine it with a backlog item and a date; the gate never retries
  silently. Slow password hashing, ports and temp paths are the usual causes: use unique ports and per-test dirs.
- **Pin the contract.** The generated OpenAPI file is committed and a hook fails when it is stale; a response-shape
  test per endpoint catches a field that disappears for a client ([web-api.md](https://github.com/evocaslab/hq/blob/main/practices/web-api.md)).

## E2E (Playwright)

- **One worker, one backend.** Each worker starts its own API with a throwaway
  data dir and ports from a fixed table; it also serves the prebuilt frontend
  (built once in global setup), so no dev server runs. Import `test` / `expect`
  from the shared fixture file, not the library.
- **Order.** Files run in parallel; tests of one file run in order and may build
  on each other, never on other files.
- **Sign in once per worker.** Keep the session cookies, put them into new
  contexts, verify with the `me` endpoint, and sign in through the UI again when
  the session is gone. The sign-in specs still drive the form.
- **Seed through the API**, not clicks, in specs that only need the data. Keep
  one UI test of the import screen itself.
- **No sleeps.** Wait for a visible state. A deliberate delay of a mocked
  response is the only fixed wait.
- **Retrying menu helper.** A row menu can close while the list reloads under it
  and swallow the click. Use one helper that repeats "open menu, click item"
  until the action's result (a dialog) is visible.
- **Quote search phrases** when a test searches for text that other tests' data
  could also match in the same worker.
- **Never assert on shared aggregates.** A count, total, badge or title prefix
  changes with other files' data in the same worker: `report-bot`'s page-title
  test failed as `(2) Головна · JAS Bot` once another file created two pending
  items. Match only the part the test owns (a regex with an optional prefix,
  `toContainText`), or give the test its own scope (user, project) so counters
  start at zero. Prefer unique data per test (a suffix from `testInfo.testId`)
  over cleaning the database between tests.
- **Find order dependence early.** Now and then run one worker with
  `--repeat-each=2`, or the files in reverse order; a test that passes alone and
  fails after another is coupled through data. Do not lower the worker count to
  hide it.
- Per-worker ports, data dirs and users come from `testInfo.parallelIndex`
  (stable across worker restarts), not `workerIndex`.
- Retry once; a pass on retry is reported as flaky (see below). Speed and worker
  count: [quality-gate.md](quality-gate.md#flaky-tests).

### FLAKY_TESTS doc

A table per state, one row per test, removed when fixed:

| Test | Symptom | Suspected cause |
| --- | --- | --- |
| `[phone] file.spec.ts:22` | what the failure looked like, and when | cause, or "Not investigated" |

A second table "Fixed" lists test, cause, fix. Also write down how to read a
failure: the page snapshot (`error-context.md`), screenshot, trace, per-worker
API log, and how to run one test alone.

## Parity tests against an old implementation

When a tool is rewritten (another language, a new framework), the old
implementation is the oracle until it is deleted; ported unit tests alone miss
what nobody wrote down.

- One test runs both implementations on identical fixtures in a temp dir, with
  the same fake external tool first on `PATH` (a script that logs its calls,
  plays the one call that matters by copying a prepared file into place, and
  fails on request).
- One record holds everything observable, so a single equality compares it all:
  exit code, stdout, log lines, resulting files byte for byte (and which files
  are gone), the sequence of external calls. Temp paths are replaced by a
  placeholder first. A scenario that produced no output fails.
- Scenarios: each flag, each failure path (a failing external call, the
  rollback), each odd input shape from the old fixtures. Intended differences
  are listed in the design doc with a reason each; everything else is a bug.
- Without the oracle installed the test skips with a note, so the new side
  builds in a plain checkout; the gate installs the oracle first. While both
  exist, a behaviour change lands in both with a parity scenario.
- For a web API the oracle is a recorder that seeds a throwaway database and
  stores the old app's answers as fixtures, plus the committed OpenAPI document
  as the schema: [rust-web-api.md](https://github.com/evocaslab/hq/blob/main/practices/rust-web-api.md#porting-an-existing-api-strangler-fig).
  The Rust CLI version: [rust-service.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service.md#porting-an-existing-tool).

## Data migration verification

For a one-shot move of production data (another database engine, a schema
rewrite). `report-bot`'s SQLite → PostgreSQL cutover, 2026-10-26: 81,554 rows in
61 tables, a dry run and the real run gave the same counts and checksums.

- **Dry run on a read-only copy** of the production file or dump, into a
  throwaway instance (never the production target), with the real migration
  code; time it and write the numbers down.
- **Counts are not enough.** Compare per table the row count and an
  order-independent checksum of canonical row text (timestamps in UTC, numbers
  in one scale, booleans one way) so a row with a wrong value fails too. The
  copy tool does this itself and stops on the first value it cannot convert,
  naming table, row and column.
- **Pass/fail before you start:** write the pass criteria and the rollback
  trigger down, keep writes off until verification passed, and keep the source
  read-only and untouched (rollback = old release + the kept source).
- **One checker for both runs**, so the sign-off compares the same numbers.
- **Restore drill:** take the first backup of the new store right after the
  copy and restore it into scratch, timed: [backup-and-restore.md](https://github.com/evocaslab/hq/blob/main/practices/backup-and-restore.md).
- Check the source is complete before copying: a SQLite file in WAL mode keeps
  committed data in `-wal` until a checkpoint, so copy with the online backup
  API or check that `-wal` is empty.

## Frontend unit tests

Vitest with `pool: 'vmForks'`: jsdom is created once per worker, not once per
spec file (a ~20 s run became ~6 s).

## Coverage-guided passes

Total coverage hides weak spots. List the least-covered modules that matter
(domain services, parsers, API routes, frontend API clients), add tests for one
capability per branch, and record the new percentage per module in the backlog
item. Verify the claim for a module with the package-level `--cov` of the gate;
`--cov=<one module>` can behave differently.

## Mutation testing

Not in the gate (slow, all cores): run it through the heavy lock on the core
parsers monthly or after a parser change. Tool in `report-bot`: mutmut,
configured in `[tool.mutmut]`.

1. Mutate one module (about a minute) or all configured modules.
2. List the survivors (a mutant is a code change no test noticed); show one
   mutant's diff when needed.
3. For each survivor that changes observable behavior, add a test that fails on
   the mutant: field values, boundaries, sheet choice, message content.
4. Ignore equivalent mutants (no observable change: message text, flags like
   `read_only=True`, default values). Do not chase every survivor.
5. After changing tests or code, delete the mutants cache.
6. Commit the tests on a `test/mutants-<area>` branch; land after the gate.

See also [database-integrity.md](https://github.com/evocaslab/hq/blob/main/practices/database-integrity.md) and [database-performance.md](https://github.com/evocaslab/hq/blob/main/practices/database-performance.md): concurrency, query-count and schema-guard tests.
