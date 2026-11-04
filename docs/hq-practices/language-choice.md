---
summary: "Which language for a new tool or service: Rust by default, Python or shell only for throwaway work, the growth trigger that forces a port"
group: quality
tags: [rust, python, language, scripts]
applies-to: [all]
reviewed: 2026-11-01
status: draft
---

# Language choice

A tool is written in the language it will still be maintained in. Small scripts do not stay small: a service that
began as a Python script and grew is rewritten later at the price of the whole history, while a tool started in Rust
pays its cost once. The maintainer's decision (2026-11-01): **Rust is the default**; this page says where it is not and
when a script must move.

## Rule

| Work | Language |
| --- | --- |
| Service, daemon, bot, API, long-running process | Rust ([rust-service.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service.md)) |
| Tool that keeps state, is called by other programs or sessions, runs in a gate or a timer, or handles untrusted input | Rust |
| Logic two programs need (parsing a file format, a registry, a protocol) | a Rust crate, shared by both |
| One-off pivot, migration run once, probe, data munging for a single question | Python or shell, deleted afterwards or promoted to Rust the moment it is kept |
| Glue of a few lines (`just` recipe, hook, a `git` or `gh` call chain) | shell or `just` |
| Browser frontend | Vue and TypeScript ([vue-frontend.md](https://github.com/evocaslab/hq/blob/main/practices/vue-frontend.md)); Rust only where the app is Rust end to end |

Python follows [python-style.md](https://github.com/evocaslab/hq/blob/main/practices/python-style.md) wherever it stays.

## Growth trigger

A Python or shell file moves to Rust in the next change that touches it when any one holds:

- it is over about 300 lines, or its test file is longer than the script;
- a second program, hook or session calls it, or it has a `--json` output others parse;
- it keeps state (a database, a lock, a queue, a cache) or runs unattended;
- it needs types to be correct (a state machine, a protocol, money, time zones, paths from users).

An old script already past the trigger is filed as a backlog item, ranked by how often it runs, how many callers it
has and what it costs in the gate; it is ported one at a time with the parity tests of
[rust-service.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service.md#porting-an-existing-tool), and the old file is deleted when the port lands.
A repo that measured its scripts (2026-11-01: 60 Python files, 16 900 lines, nine over 700 lines) has its worst
cases in the tools that run on timers and in the gate, not in the one-off helpers.

## Why

- **Guarantees are built in.** In Python, types, lints and tests are added on top (a strict type checker, `ruff`, models,
  tests), each costs gate time and memory, and each still leaves gaps (runtime `None`, shape of parsed data). In Rust
  the compiler checks them once, at build time, for the whole program.
- **Rewriting is the expensive path.** The cost of a port grows with the size and the number of behaviours to keep;
  starting where the tool will live avoids it.
- **One toolchain for shared logic.** A parser in a crate is used by the web API, the CLI and the MCP server; the same
  logic in Python is copied or called through a process.

## Costs, stated honestly

- Compile time and RAM: [rust-build-times.md](https://github.com/evocaslab/hq/blob/main/practices/rust-build-times.md); the cargo hooks hold the machine-wide heavy lock.
- A slower first draft, and more code for shell-heavy work (spawning tools, parsing their output): use a small helper
  crate for it instead of repeating it.
- A second toolchain to keep current ([rust-service-tooling.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service-tooling.md)).

These do not outweigh a rewrite, but they do outweigh Rust for a ten-line probe.

## Not a reason to rewrite

A working script under the trigger, a one-off, or a frontend. Do not port for taste: port when a trigger fires.

## Sources

- Maintainer decision, 2026-11-01 (Rust default, growth trigger, frontend exception).
- [rust-service.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service.md) (layout, errors, porting with parity tests), [python-style.md](https://github.com/evocaslab/hq/blob/main/practices/python-style.md).
- The Rust Programming Language, "Fearless concurrency" and ownership chapters: <https://doc.rust-lang.org/book/>.
