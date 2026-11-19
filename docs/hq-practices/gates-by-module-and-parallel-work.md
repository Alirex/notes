---
summary: "Split a gate by module with an affected set and tiers, keep the full gate authoritative; parallel agents own disjoint paths cut along bounded contexts, short branches, seams land first"
group: quality
tags: [gate, architecture, subagents, agents, ddd, rust]
applies-to: [all]
reviewed: 2026-11-10
status: draft
check: checklist
---

# Gates by module and parallel work

## The rule

1. **Tier the gate by cost; only the top tier decides.** Edit loop (seconds: fixers, lint, type check, tests of the
   changed module) < pre-commit (changed files) < pre-push or branch gate (the affected set: changed modules plus
   everything that depends on them) < **full gate** (everything; the only tier that may say "done" and the one run
   before a landing) < nightly or weekly (slow checks: mutation, audits, full e2e). A lower tier may be wrong in the
   quiet direction (miss a failure); the full gate is never skipped to land.
2. **Affected set = changed files, mapped to modules, plus reverse dependencies.** Treat these as global (they select
   everything): lock files, toolchain and workspace config, CI and hook config, shared fixtures, build scripts,
   generated sources, anything the map does not know. An unmapped file means "run the full gate", not "nothing
   affected".
3. **One module, one owner at a time.** Parallel agents get disjoint paths, cut along bounded contexts
   ([service-boundaries.md](service-boundaries.md)). Two writers on one path is a scheduling error to fix by order,
   not a merge to resolve later.
4. **A seam is its own branch and lands first.** A change to a contract between modules (schema, trait, API type, file
   format) goes in one short branch with contract tests; the dependent branches start after it lands.
5. **Short branches, one lander.** A branch lives days, not weeks; each is "a merge not yet done". Land one at a
   time, merge main into the next, re-run its affected gate (then the full gate before landing).

## Why

The affected-set idea is old and every implementation shares its blind spot: Nx selects by declared task inputs
and warns that undeclared inputs are missed and lock-file changes are broad; Bazel gives `rdeps` over declared
targets. nextest does the same for Cargo with `rdeps`. A selection is only as good as the declared graph, so it
speeds the loop and never replaces the full run. Conway's law explains the second half: modules follow who works on
them, so cutting work by context gives the same cut for gates, branches and reviews. For one maintainer with agents,
"team" is "writer handle". Trunk-based guidance fits agents directly: each agent's change small, integrated one at a
time, branches of a couple of days.

## Splitting a Cargo workspace

- Map crate to path (`cargo metadata` gives manifests and the dependency graph; no hand-written list).
- Edit loop: `cargo check -p <crate>` and `cargo nextest run -p <crate>`.
- Branch gate: `cargo nextest run -E 'rdeps(<crate>)'` for each changed crate (union), with clippy and fmt on those
  crates. Reverse dependencies are included, so a change in a base crate runs its dependents.
- Global files (above) and changes to `Cargo.lock`, `rust-toolchain`, `build.rs`, `include!`/`include_str!` inputs run
  everything.
- `cargo-hakari` (workspace-hack, one feature set per dependency) only if `cargo build --timings` shows shared
  dependencies rebuilt for `-p` runs; not at ten crates by default.
- Mixed-language repos (an R or Python oracle next to a Rust core): the language boundary is a context boundary. Each
  side has its own tier-2/3 commands; the seam (golden files, a parity test, a schema) is a shared input to both, so a
  seam change selects both sides.
- Python: testmon-style selection is for the edit loop only (blind to SQL, JSON fixtures, env); see
  [quality-gate.md](quality-gate.md).

## Parallel work

- **Path map.** A small file listing context, paths and the current writer (CODEOWNERS pattern syntax is a fine
  format: gitignore-like, last match wins). Sessions read it before taking a branch; the map and the context map
  ([service-boundaries.md](service-boundaries.md#context-map-template)) are one cut.
- **Assign by disjoint paths first.** Pick the next item whose paths do not overlap an open branch. HQ already
  warns after the fact (`machine-capacity`: "two writers ... change the same files"); the map lets the supervisor
  choose before.
- **Contract tests at the seams** let two contexts move independently: each side tests against the shared contract
  ([testing.md](testing.md) parity tests), not against the other side's code.
- **Shared kernel stays small and single-owner.** Common types, config and fixtures are the usual cause of "every
  change touches everything"; if a PR needs them, it is the seam branch of rule 4.
- **Order beats resolution.** Two branches that must touch one file land in sequence; the second merges main first.
- A hosted merge queue (test a PR on top of the queue ahead of it) is what "land one, merge main, re-gate" does by
  hand; use the hosted one only with several human committers.

## Measure before and after

A tier that is not measured drifts. Record wall time per tier (HQ: `just machine-metrics runs --steps`) and two
misses: failures the full gate found that the affected tier passed (a miss means a missing edge in the map; fix the
map), and conflicts on landing. Targets are the repo's to set; a miss rate above zero over a month is a map bug.

## Studied and not taken

Bazel/Nx/Turborepo adoption for small workspaces, `cargo-hakari` by default, test-impact selection as the authority,
merge queue and CODEOWNERS enforcement for one maintainer, splitting into repos to gain parallelism. Reasons and
reopen conditions: [research note](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-10-gate-splitting-and-parallel-work.md).

## Checklist

- [ ] The gate has named tiers with commands and typical wall times.
- [ ] The affected set is computed from the real dependency graph, not a hand list.
- [ ] Global files are listed and select the full gate; an unmapped file does too.
- [ ] The full gate runs before every landing and on a schedule.
- [ ] A path map names context, paths and writer; assignments do not overlap.
- [ ] Seam changes are their own branch with contract tests and land first.
- [ ] Gate time per tier and affected-tier misses are recorded.

## Checks

Manual (checklist). A conformance check can later verify that the map covers every tracked path.

## Sources

Read 2026-11-10 (evidence: moderate = official docs):

- nextest filtersets (`package`, `deps`, `rdeps`): https://nexte.st/docs/filtersets/
- Nx affected tasks and their stated limits: https://nx.dev/ci/features/affected
- Bazel `rdeps` queries: https://bazel.build/docs/query-how-to
- cargo-hakari: https://docs.rs/cargo-hakari/latest/cargo_hakari/
- pytest-testmon (weak: README only): https://github.com/tarpas/pytest-testmon
- Trunk-based development, short-lived branches and agents: https://trunkbaseddevelopment.com/short-lived-feature-branches/
- GitHub merge queue: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue
- CODEOWNERS syntax: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners
- Conway's law: https://martinfowler.com/bliki/ConwaysLaw.html
- Team Topologies key concepts: https://teamtopologies.com/key-concepts
- HQ research note with the details and not-taken table: [2026-11-10-gate-splitting-and-parallel-work.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-10-gate-splitting-and-parallel-work.md)
