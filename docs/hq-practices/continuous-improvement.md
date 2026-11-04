---
summary: "The improvement loop once (baseline, budget, guard, record), regression guards on noisy machines, traceability from backlog item to error, test-suite and agent-flow numbers"
group: quality
tags: [performance, traceability, tests, agents, process]
applies-to: [all]
reviewed: 2026-11-01
status: draft
---

# Continuous improvement: measure, guard, trace

The loop already exists in pieces: [profiling.md](profiling.md) (tools, judging a run, the gate speed budget),
[machine-capacity.md](https://github.com/evocaslab/hq/blob/main/practices/machine-capacity.md) (profile before tuning), the `/optimize` skill, the weekly Retro and
[agent-workflow.md](agent-workflow.md#periodic-self-analysis). They cover gate speed, machine load and plan spend.
This page names the loop once, extends it to the product, tests, deploys and the agent flow, and prefers **guards that
fail loudly** to reviews that depend on someone remembering.

## The rule

| Rule | Why |
| --- | --- |
| Every improvement has four artefacts: a **baseline** (command, data, machine, spread), a **budget**, a **guard** that fails, and a **record** (commit, changelog or item with before and after) | A fix without a baseline cannot be shown to work; a budget without a guard is a wish |
| No budget without a guard, no guard without history | A fixed threshold on a noisy number cries wolf; history tells what normal is |
| A gate fails only on a noise-free number: instruction counts, bytes, row or buffer counts, test counts. Wall time goes to a history file and is judged against it | Shared and virtual machines make wall time unstable; the Criterion FAQ says not to rely on it in cloud CI |
| Flag a result that is an outlier against **its own history** (IQR fence, or median of a window before and after), and re-run to confirm before acting | Fixed percentages hide small drift and flag noise; Mozilla and rustc-perf do the same at scale |
| Each running app has 3 to 5 numbers with limits, written next to the code | A budget in someone's head is not checked |
| Profile on demand by default; always-on profiling only when the error tracker already offers it as a switch | One host and one maintainer do not feed a profiling server |
| One id chain from backlog item to error (below) | "Which change caused this?" must take one command |
| Three flow numbers per service, weekly, as a trend, never as a target | Targets get gamed (DORA's own warning) |
| One change for the worst number each week | Several changes at once cannot be told apart |

## The loop

1. **Measure** from what is already recorded (`just machine-metrics runs --steps`, `just docs-flow`, `just cc-cost`,
   the repo's `q-timings`); a new run only at low load and at least twice ([profiling.md](profiling.md#judging-a-run)).
2. **Pick** the largest avoidable cost; check the backlog for an item that already covers it.
3. **Fix** in the owning repo by its rules; large or unclear work is a backlog item with the numbers.
4. **Guard**: turn the budget into a check (a test, a gate step, a size limit, a threshold on history).
5. **Record**: before and after in the commit or changelog; a repo-neutral lesson goes to `practices/`.

| What | Budget and guard | History | Cadence |
| --- | --- | --- | --- |
| Gate and test stages | Gate speed budget with a loud threshold ([profiling.md](profiling.md#speed-budget-for-gates)) | Stage times per run | Monthly Speed review |
| Product | p95 of the hot endpoints, slowest `pg_stat_statements` entry, bundle size, Lighthouse budgets | Weekly snapshot | With each release |
| Tests | Slowest N printed every run; quarantine cap | Slowest N per run | Weekly |
| Deploys | Lead time, deploy frequency, change fail rate | Deploy record | Weekly Retro |
| Agent flow | Plan cost, retries and time to land per landed item | `cc-cost --by branch`, `docs-flow` | Weekly Retro |

## Assess the effect, not the activity

Counting what was done (items landed, sessions run, calls made) says the work happened, not that it helped. For a change
meant to alter behaviour or a number, write the **effect assessment** next to the loop's baseline: one row, filled in
when the item is filed and re-read at the Retro.

| Field | Meaning | Example: MCP adoption |
| --- | --- | --- |
| Effect | What should be different in practice | Sessions in other repos find practices and tasks through the `hq` tools |
| Criterion | How we would know | A call in the calls log from a session outside HQ |
| Measure of effectiveness | Are we doing the right things (not: did we do the things) | Repos with at least one `practices_*` or `backlog_*` call per week |
| Source | Where the number comes from | `~/.local/state/hq/mcp/calls.jsonl` |
| Baseline and success | Today's value; the value that counts as done | 0 repos; calls from 2 or more repos |
| Assessment and confidence | Current reading; how good the sources are (quality, count) | "Not met"; one source, but a direct count |
| Recommendation | Keep, change, stop, or measure better | Fix registration, add the rule to each repo |

- The backlog item's **Done when** line names the effect and its source, not the activity ("calls from two repos", not
  "rule added to AGENTS.md").
- The weekly Retro re-reads open rows; a row with low confidence gets a better source before it gets a verdict.
- `/optimize` and `/deep-review` reports end with this row for each finding they act on.
- Performance (did we do it) and effectiveness (did it help) are separate columns; a green activity count with a flat
  effect number is a finding.

## Budgets and regression guards

- **Instruction counts may fail a gate.** Valgrind-based tools (`gungraun`/iai-callgrind for Rust) count instructions
  even on virtual CI machines; use `EXPLAIN` row and buffer counts for SQL and bytes for bundles and payloads
  ([web-loading.md](https://github.com/evocaslab/hq/blob/main/practices/web-loading.md), [egress-and-payload.md](https://github.com/evocaslab/hq/blob/main/practices/egress-and-payload.md)).
- **Wall-clock benchmarks never fail a gate.** `criterion`, `hyperfine` (`--warmup`, `--prepare` for a cold cache, JSON
  export) and `pytest-benchmark` append to a history file; a change is flagged against that history.
- **Change detection**: an outlier against the benchmark's own history, confirmed by re-runs (Mozilla's sheriffing
  retriggers noisy alerts three times before anyone files). For HQ the `machine-metrics` history plus an IQR fence is
  enough; no hosted benchmark service.
- **Product budgets** use the platform's format where there is one: Lighthouse reads a `budget.json` (`timings`,
  `resourceSizes`, `resourceCounts`) through `--budget-path`.
- Keep a weekly `pg_stat_statements` top-10 per database ([database-performance.md](https://github.com/evocaslab/hq/blob/main/practices/database-performance.md)).

## Traceability: from item to error

One chain, every link an id that the next link can read:

| Link | Carries |
| --- | --- |
| Backlog item `B-YYMMDD-slug` | The request, the numbers |
| Landing commit trailers | `Item: B-…`, and where the flow has them `Session:` and `Gate:` (read with `git log --format=%(trailers)`) |
| Image labels | `org.opencontainers.image.revision` and `.source`, `.version` where versions exist |
| Status route | The same revision, build date and dirty flag ([observability-and-incidents.md](https://github.com/evocaslab/hq/blob/main/practices/observability-and-incidents.md)) |
| Error tracker release | That revision ([releases-and-versions.md](releases-and-versions.md)) |
| Request id in logs, `traceparent` across 3 or more own services | The call ([observability-and-incidents.md](https://github.com/evocaslab/hq/blob/main/practices/observability-and-incidents.md#tracing-across-services)) |
| Deploy record | What, when, rolled back or not ([deploy-and-operations.md](https://github.com/evocaslab/hq/blob/main/practices/deploy-and-operations.md)) |

- **Provenance and SBOM at the small-team level**: keep buildx's default `mode=min` provenance, add `--sbom=true` on
  release builds, and use GitHub's artifact attestations for published images or CLIs (SLSA build level 2); verify in the
  release recipe. Never `mode=max` with secrets in build args; do not build toward level 3
  ([supply-chain-security.md](supply-chain-security.md)).
- **Flow numbers**: change lead time (`requested` to deployed), deploy frequency and change fail rate, per service.
  `just docs-flow` already gives cycle time; change fail rate needs the deploy record above.

## Tests as a measured system

- Print the **slowest N** on every full run and keep them: `pytest --durations=10 --durations-min=1.0`, nextest
  `slow-timeout` with `terminate-after`, Vitest `slowTestThreshold`.
- The quarantine has a **cap** (a number of tests or a number of days); the gate fails when it is exceeded. A retried
  pass is reported as flaky and may be set to fail (`nextest --flaky-result fail`). See
  [testing.md](testing.md#time-concurrency-and-flaky-tests) and the FLAKY_TESTS doc.
- On a failure, keep what the runner records (Playwright `trace: 'on-first-retry'`, captured logs with the request id,
  nextest output) and print the request id of the failing call. OpenTelemetry spans per test are not worth it yet: the
  test semantic conventions are still in Development.
- Shard Playwright (`--shard`, blob reports, `merge-reports`) only when one machine cannot run it within the budget;
  `--only-changed` selects changed test files, which is not test impact analysis.

## The agent flow

Per landed item record plan cost (`just cc-cost --by branch`), retries and API errors, gate re-runs and time to land;
the weekly Retro compares medians and changes one tier or effort rule. Claude Code's OpenTelemetry export
(`claude_code.cost.usage`, `api_request`, `api_error`, `tool_result`, all with `session.id`) is optional and worth it
only where a collector runs anyway; the transcripts carry the same facts. The `Session:` trailer is what joins cost to
a landing.

## Leave out

| Skip | Why |
| --- | --- |
| A full collector with Tempo, Loki, Prometheus and Grafana | "Count before you graph"; traces only at 3 or more services |
| Pyroscope or Parca servers | An always-on service, storage and a UI nobody opens for one host |
| Hosted benchmark dashboards (Bencher, CodSpeed) | A history file and an IQR fence give the same decision |
| DORA dashboards and targets | Three numbers in the Retro table; targets invite gaming |
| SLSA level 3, keyless signing for everything | Level 2 comes with GitHub attestations; level 3 buys nothing for this threat model |
| Story points and velocity for agents | Rejected in [agile-lessons.md](agile-lessons.md) |

## Open

- Whether the `Item:`, `Session:`, `Gate:` trailers become a landing check in every repo or stay a recommendation
  (HQ first): backlog item `B-261101-commit-trailers-traceability`.
- Whether product benchmarks get a wall-clock history recipe like `machine-metrics`.
- Change fail rate needs a deploy record per service; the updater may not write one yet.
- Evidence is thin for continuous profiling and test spans at this scale; both are judgment, not findings.

## Observed in

HQ: `machine-metrics` (stage times per run, `just/machine.just`), `docs-flow` (cycle time), `cc-cost`, the Speed review
and Retro chores, `q-integrity` (`Gate-change:`). report-bot: `scripts/quality/timings.sh` with `q-timings` and
`q-slow`, nextest slow-timeout, `/version` with commit and dirty flag, image revision labels, FLAKY_TESTS.md, chores for
flaky tests, mutation, gate speed and coverage trend. teleporto: bundle size budget (`web/scripts/size-budget.mjs`),
`/api/health` with build commit. Gaps: no request id binding or tracing in report-bot, optiomni or teleporto; no
timings in ui, site, vigor, auth, lms or repo-template; no SBOM or provenance anywhere.

## Sources

- Maintainer's effect-assessment table (2026-11-01; operational effect, criterion, measure of effectiveness, source,
  indicator, baseline, success criteria, assessment, confidence, recommendation): the shape of the row above
- DORA, the four keys and the warning about targets: <https://dora.dev/guides/dora-metrics-four-keys/>
- Performance budgets: <https://web.dev/articles/performance-budgets-101>,
  <https://web.dev/articles/use-lighthouse-for-performance-budgets>
- Criterion FAQ on CI noise: <https://bheisler.github.io/criterion.rs/book/faq.html>
- iai-callgrind / gungraun: <https://github.com/iai-callgrind/iai-callgrind>
- hyperfine: <https://github.com/sharkdp/hyperfine>
- rustc-perf comparison analysis: <https://github.com/rust-lang/rustc-perf/blob/master/docs/comparison-analysis.md>
- Mozilla performance sheriffing: <https://wiki.mozilla.org/TestEngineering/Performance/Sheriffing>
- Change-point detection in CI: <https://arxiv.org/abs/2003.00584>
- Bencher thresholds: <https://bencher.dev/docs/explanation/thresholds/>
- Continuous profiling overhead: <https://grafana.com/docs/pyroscope/latest/introduction/continuous-profiling/>,
  <https://docs.sentry.io/product/explore/profiling/performance-overhead/>
- git trailers: <https://git-scm.com/docs/git-interpret-trailers>
- OCI annotations: <https://github.com/opencontainers/image-spec/blob/main/annotations.md>
- SLSA levels: <https://slsa.dev/spec/v1.0/levels>
- Docker attestations: <https://docs.docker.com/build/metadata/attestations/>
- GitHub artifact attestations: <https://docs.github.com/en/actions/concepts/security/artifact-attestations>
- Playwright trace viewer and sharding: <https://playwright.dev/docs/trace-viewer>, <https://playwright.dev/docs/test-sharding>
- OpenTelemetry test and CI/CD conventions: <https://opentelemetry.io/docs/specs/semconv/registry/attributes/test/>
- pytest durations: <https://docs.pytest.org/en/stable/how-to/usage.html>
- nextest slow tests and retries: <https://nexte.st/docs/features/slow-tests/>, <https://nexte.st/docs/features/retries/>
- Fowler on non-determinism and quarantine: <https://martinfowler.com/articles/nonDeterminism.html>
- Claude Code monitoring: <https://code.claude.com/docs/en/monitoring-usage>
