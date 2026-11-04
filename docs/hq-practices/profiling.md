---
summary: "Profile before optimizing: the tool per layer (py-spy, flamegraph/samply, Vitest slow tests, EXPLAIN, DevTools), reading flame graphs, slow tests, low-load repeat runs, gate speed budget"
group: quality
tags: [performance, tests, tooling]
applies-to: [all]
reviewed: 2026-11-04
status: draft
---

# Profiling

How to find where time goes before changing code, and how to keep a gate fast. A guess about the slow part is wrong
often enough that no optimization starts without a measurement. The gate itself:
[quality-gate.md](quality-gate.md); the tests that feed it: [testing.md](testing.md).

## The tool per layer

Pick the tool that matches the layer where the time is spent. When unsure, start at the outermost layer (a timer
around the command) and go down.

### Python: py-spy

A sampling profiler that attaches from outside, so the code needs no change.

- `py-spy record -o profile.svg -- python script.py` runs a command and writes a flame graph (SVG by default;
  `--format` selects speedscope or raw data). `py-spy record -o profile.svg --pid <pid>` attaches to a running
  process. For a test, run the test runner as the command.
- `py-spy top --pid <pid>` is a live view of the hottest functions; `py-spy dump --pid <pid>` prints the current
  stack of every thread, the quick answer to "what is it stuck on".
- Options worth knowing: `--native` includes C, C++ and Cython frames; `--idle` includes threads waiting (off by
  default, so a program that mostly waits looks empty without it); `--subprocesses` follows children; `--gil` keeps
  only the thread holding the GIL.
- Attaching to a PID on Linux usually needs root (the `ptrace_scope` sysctl); launching the program through py-spy
  generally does not. In Docker the container needs `--cap-add SYS_PTRACE`.
- Wall-clock time and CPU time differ: a flame graph of only on-CPU samples hides waiting on a database, network or
  lock. If total run time is far above CPU time, profile with `--idle` or look at the database layer.

### Rust: cargo flamegraph and samply

- Profile a release build with debug info, or the stacks are poor. Either set `debug = true` in `[profile.release]`
  (and in `[profile.bench]` for tests and benchmarks, which use the bench profile), or use
  `CARGO_PROFILE_RELEASE_DEBUG=true` for one run. A separate `profiling` profile that inherits `release` keeps the
  shipped build unchanged.
- `cargo flamegraph` runs `cargo run --release` by default and writes `flamegraph.svg`; select a target with
  `--bin`, `--example`, `--test` or `--bench`, and pass program arguments after `--`. Unit tests:
  `cargo flamegraph --unit-test -- path::to::test`.
- On Linux it uses `perf`. Without root, lower `perf_event_paranoid` (or pass `--root`). With the lld or mold
  linker add `-Wl,--no-rosegment`, or stacks can be inaccurate.
- `samply record ./target/profiling/app args` runs the binary, records stacks, and opens the Firefox Profiler on a
  local server: flame graph, call tree, timeline, and per-line sample counts in the source view. Prefer it when the
  program has several threads or when the timeline matters; prefer `cargo flamegraph` for a single quick SVG.

### JavaScript tests: Vitest

- `slowTestThreshold` (default 300 ms; also `--slow-test-threshold=<ms>`) marks tests and suites that took longer
  as slow in the results. Set it to a value that fits the project so the slow list is short enough to read.
- Run the full suite once, sort by duration, and work from the top. A test that is slow for its own reasons
  (a real animation, a timer) is a candidate for fake timers; one slow only in a group points at shared setup.

### PostgreSQL: EXPLAIN and pg_stat_statements

- `EXPLAIN (ANALYZE, BUFFERS) <query>` runs the query and reports actual time, actual row counts and block
  activity (hits, reads, dirtied, written) per plan node, next to the planner's estimates.
  - Read it from the innermost, most-indented nodes outward. The first thing to check is the estimated against the
    actual row count: a big gap means stale or missing statistics and usually explains a bad plan (`ANALYZE` the
    table, or add an index or extended statistics).
  - Times and rows in a node are per-execution averages; multiply by `loops` for the total. A cheap inner scan with
    `loops=100000` is the cost.
  - `ANALYZE` really executes the statement, so for `INSERT`, `UPDATE`, `DELETE` wrap it:
    `BEGIN; EXPLAIN (ANALYZE, BUFFERS) ...; ROLLBACK;`.
  - Timing itself has overhead on hosts with a slow clock (`pg_test_timing` measures it), and network transfer is
    not included in the reported time.
  - Run it on data of production size. A plan on a hundred rows says nothing about a hundred million.
- `pg_stat_statements` answers "which queries cost the most in total", which a single `EXPLAIN` cannot. It needs
  `shared_preload_libraries = 'pg_stat_statements'` (a restart) and `CREATE EXTENSION pg_stat_statements` in each
  database that should have the view. Sort by `total_exec_time` to find the worst offenders (many cheap calls often
  beat one slow call), then by `mean_exec_time` for the slow single ones. Compare `calls`, `rows` and the block hit
  ratio (`shared_blks_hit` against `shared_blks_read`). Constants are normalized, so one entry covers every value of
  the same query. `pg_stat_statements_reset()` starts a clean window before a measurement.

### Browser: DevTools and Lighthouse

- DevTools Performance panel: enable screenshots, set CPU throttling (4x for a mid device, 20x for a low-end
  phone; a developer machine hides problems), press Record, do the action, Stop. In the Main track, wide bars are
  long events, vertical stacking is the call stack, and a red triangle marks a long task.
- Lighthouse (DevTools tab, command line, Node module for CI, or PageSpeed Insights) scores a page load and lists
  First Contentful Paint, Speed Index, Total Blocking Time, Largest Contentful Paint and Cumulative Layout Shift.
  The DevTools tab can audit pages behind a login. Use it to find what to look at, then use the Performance panel
  to find which code is responsible.

## Reading a flame graph

- The horizontal axis is not time. Frames are merged stacks sorted by name; width is the share of samples that
  included the frame. A frame twice as wide was in twice as many samples. Colors carry no meaning in the classic
  layout. A chart with time on the x-axis is a flame chart (the browser panel above), a different thing.
- The vertical axis is stack depth: the callers are below (or above, in an inverted "icicle" layout), the function
  that was running is at the top edge.
- Look for plateaus: wide frames with a flat top mean the function itself burns the time. A wide frame with many
  thin frames on top is only a caller; descend into the children instead.
- Use the search box: it shows the percentage of all samples that include a term, which turns "is JSON parsing a
  problem" into a number.
- Sampling loses order and short events. A function that runs for 1 ms a thousand times shows as one wide frame,
  which is right for cost but says nothing about when it ran.
- Before acting, ask whether the wide frame is avoidable work (repeated parsing, an N+1 loop, rebuilding a client
  per call), needed work done slowly, or waiting. Fix avoidable work first: it is usually the cheapest and the
  largest.
- Compare two graphs from before and after rather than reading one in isolation; a differential view or a side by
  side of the same frame is the proof that a change worked.

## A slow test is often a fixture problem

When one test or file is slow, profile it before touching the code under test.

- Time is usually in setup: creating a database, running migrations, seeding rows, starting a server or a
  container, building a large object, importing heavy modules. The test body is often milliseconds.
- Signs: many tests in one file are each slow by about the same amount; the first test in a session is far slower
  than the rest; the profile's widest frame is in a fixture, a migration or a fixture factory.
- Fixes in order: share the expensive setup at a wider scope (per session or per worker) and reset state cheaply
  (a transaction rolled back, a template database copy, truncating tables); build the minimum rows a test needs;
  replace a real clock or sleep with a fake; move the one genuinely slow case behind a marker so the quick loop can
  skip it. See [testing.md](testing.md) for the database patterns.
- A test that is slow because the product code is slow is a product finding; the fix goes in the code and the test
  stays as the regression check.

## Judging a run

A single timing is an anecdote. Shared machines, other builds, disk caches and CPU frequency scaling move a result
by tens of percent.

- Judge a speed result only at low load. Check the load average or the running jobs first; if other heavy work is
  on the machine, wait or do not draw a conclusion.
- Run at least twice, three or more when the difference is small. The first run warms caches (disk, package
  imports, compiled outputs, test selection), so compare warm runs with warm runs and cold with cold.
- Report the spread, not only the best: "159 s and 161 s" is a result, "159 s" once is not. Keep the same command,
  the same data, the same machine for before and after.
- A profile is useful even under load (it shows proportions), a timing is not. Do not claim a speedup from a
  profile; claim it from repeated timings.

## Before you report a number

A number goes into a report or a commit message only after these checks:

1. Write the sentence you would ship, and the result that would change a decision.
2. Name the limiter from a profile or counters: why is it not twice as good? No limiter named means inconclusive.
3. Check the arithmetic bound: removing a piece that takes 10 % of the run makes it at most about 11 % faster.
4. Count errors and non-success inside the timed region; a fast failure is not a speedup.
5. Confirm the work happened (rows written, bytes read, the test ran).
6. Run at least five times per side, alternating before and after; report median and range. A gap inside the spread
   is "no measurable difference".
7. Measure the path a user waits on; show a micro result as a share of it.

Report the verdict first (faster, slower, no measurable difference, inconclusive), then the number with its unit,
the runs, the range and the limiter. The reviewer lens is "Explain every number"
([review-lenses.md](review-lenses.md)).

## Speed budget for gates

A gate nobody can run in a few minutes gets skipped or run less. Give it a budget and check against it.

- Set explicit numbers per project: the full gate in minutes, the quick loop (changed files only) in seconds, and
  the slowest single test. Write them next to the gate's definition so a reviewer can see them.
- Measure every run (command, wall time, per-step time) and keep the history, so a creep of five percent a week is
  visible. A threshold that fails loudly on a jump is cheaper than a quarterly review.
- When a budget is exceeded, the fix goes in before new work that makes it worse: profile with the tools above, take
  the largest item, and measure again.
- Per step, in the order that usually pays: cache and incremental runs; parallelism on the test runner; faster
  fixtures; removing duplicate work between hooks and CI; a faster tool for the same check. Cutting a check from the
  gate is the last resort and needs a replacement guard ([quality-gate.md](quality-gate.md)).

## Observed in

- Observed in a Python monorepo: a profiler run by hand on the contract recorder cut a test stage from 256 s to
  159 s (commit 8da82776); the cost was shared setup done per test, not the code under test. This was not
  independently re-verified for this page.

See also [continuous-improvement.md](continuous-improvement.md): baseline, budget, guard and record for every fix; wall time goes to history, not to a gate.

## Sources

- pstack (MIT, Lauren Tan): <https://github.com/cursor/plugins/tree/main/pstack> (`benchmark-checklist, principle-explain-the-number`)
- py-spy, README: https://github.com/benfred/py-spy
- cargo-flamegraph, README: https://github.com/flamegraph-rs/flamegraph
- samply, README: https://github.com/mstange/samply
- Vitest, `slowTestThreshold`: https://vitest.dev/config/slowtestthreshold
- PostgreSQL, "Using EXPLAIN": https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL, `pg_stat_statements`: https://www.postgresql.org/docs/current/pgstatstatements.html
- Chrome for Developers, "Analyze runtime performance" (Performance panel): https://developer.chrome.com/docs/devtools/performance
- Chrome for Developers, Lighthouse overview: https://developer.chrome.com/docs/lighthouse/overview
- Brendan Gregg, "Flame Graphs": https://www.brendangregg.com/flamegraphs.html
