---
summary: "Strictest linters and type checkers; fix findings, never silence them"
group: quality
tags: [lint, types]
applies-to: [all]
reviewed: 2026-10-27
status: stable
---

# Strict checks: fix, don't silence

Linters, type checkers and tests run in their strictest mode. A finding is fixed in the code, not silenced. A rule may
be turned off, but only on purpose, in one place, with a reason. The same goes for everything else that can be made
quiet: exceptions, tests, runtime warnings, hooks, CI steps.

## Fix first, in this order

When a check fails, take the first option that works:

1. **Fix the code**: the finding is usually right.
2. **Make the types stricter.** Most type findings come from data with no type (`dict[str, Any]`, parsed JSON/TOML,
   untyped libraries). Validate it once at the boundary into a typed model, or narrow it, then the rest of the code is
   typed (see [Stricter types](#stricter-types)).
3. **Restructure**: too many arguments → an options object; too complex → smaller functions; a broad `except` → the
   specific exceptions.
4. **Suppress one line**, scoped to the rule, with a reason on the same line or just above:
   `# noqa: S603  # fixed argv, no shell`. Never a bare `# noqa`, `# type: ignore` or `@ts-ignore`.
5. **Turn the rule off** in the config, only when it is wrong for the whole project (it conflicts with the formatter,
   or the style it asks for is not wanted). One comment line per rule with the reason.

A `cast`, `as X`, `!` (non-null assertion) or `unwrap()` is a suppression too: it tells the checker to trust you.
Prefer validation or narrowing.

## Stricter types

| Instead of | Use |
| --- | --- |
| `Any` / `any` for unknown data | `object` / `unknown`, then narrow (`isinstance`, `TypeIs`, a type guard) |
| `dict[str, Any]` from JSON/TOML/env | a pydantic model, `TypedDict` + `TypeAdapter`, a `serde` struct, a zod schema, validated once |
| `str` / `int` for IDs, names, paths | `NewType`, a frozen pydantic `RootModel`, a Rust newtype, a TS branded type |
| a bag of `bool` / `str` options | an enum or `Literal`, a frozen dataclass of options |
| an untyped library in every caller | stubs (`types-*`), or one small typed wrapper module: the only place that sees its `Any` |
| `as Config` on an object literal (TS) | `satisfies Config` |

## Suppressions must stay needed

An ignore that outlived its cause hides the next real finding on that line. Make the tools report unused ones:

| Tool | Setting |
| --- | --- |
| ruff | `RUF100` (unused `noqa`), included in `ALL` |
| basedpyright | `reportUnnecessaryTypeIgnoreComment = "error"`, `reportIgnoreCommentWithoutRule = "error"` (both on in `all`) |
| mypy | `warn_unused_ignores` (in `strict`), `enable_error_code = ["ignore-without-code"]` |
| TypeScript / ESLint | `@ts-expect-error` with a description, never `@ts-ignore` (`ban-ts-comment`); `linterOptions.reportUnusedDisableDirectives: "error"` |
| Rust | `cargo clippy --workspace --all-targets --all-features -- -D warnings` with the `pedantic`, `nursery` and `cargo` groups plus single restriction lints (`unwrap_used`, `expect_used`, `panic`, `unreachable`, `allow_attributes`, ...) warned in `[workspace.lints]`, `unsafe_code = "forbid"`, `cargo doc` with `-D warnings`; tests exempt through `clippy.toml` (`allow-unwrap-in-tests`), not attributes ([rust-service.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service.md#lints)) |

## Strictest mode per tool

| Tool | Setting |
| --- | --- |
| ruff | `extend-select = ["ALL"]`, commented `extend-ignore` ([python-style.md](https://github.com/evocaslab/hq/blob/main/practices/python-style.md)) |
| basedpyright | `typeCheckingMode = "all"`, `failOnWarnings = true`; an existing repo moves there by the ratchet ([quality-gate.md](quality-gate.md#strict-typing)) |
| mypy (if used besides basedpyright) | `strict = true`, plus the plugins of the libraries in use (`pydantic.mypy`) |
| pytest | `--strict-markers --strict-config`, `xfail_strict = true`, `filterwarnings = ["error"]` |
| TypeScript | `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride` |
| ESLint | `typescript-eslint` `strictTypeChecked` + `stylisticTypeChecked`, `--max-warnings 0` |
| Rust | `cargo clippy --workspace --all-targets --all-features -- -D warnings` with `clippy::pedantic` and the `unwrap_used`/`expect_used`/`allow_attributes` restriction lints warned in `[workspace.lints]`, `unsafe_code = "forbid"`; tests exempt through `clippy.toml` (`allow-unwrap-in-tests`), not attributes ([rust-service.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service.md#lints)) |
| shellcheck | `enable=all` in `.shellcheckrc`, disabling single optional checks with a reason |
| markdownlint, prettier, YAML | default rules, line length set once for the repo |

Warnings fail the gate (`failOnWarnings`, `--max-warnings 0`, `-D warnings`, `filterwarnings = error`): a warning
nobody has to fix just piles up.

Every repo with shell scripts keeps its own `.shellcheckrc`, even an empty one: shellcheck walks up the directories
and uses the first rc it finds, so a repo cloned inside another (HQ's `repos/`) would otherwise run under the outer
repo's rules and its gate would differ by where it is checked out.

Tests are code: the same rules apply. Relax only what a test needs, by a per-file ignore for test files with a reason
(`S101` assert, `PLR2004` magic values, `D` docstrings).

## Not only linters

The same rule for every other way to make a failure quiet:

- **Exceptions**: no bare or broad `except` that drops the error. Catch the specific exception; a top-level catch-all
  logs it with the traceback and re-raises or exits non-zero. No `except: pass`, no `.catch(() => {})`, no `let _ =`
  on a `Result` that matters.
- **Tests**: never `skip`, `xfail` or delete a test to get green. A flaky test is fixed, or quarantined with a backlog
  item and a date; a skip names the condition (`skipif(sys.platform == "win32", reason=...)`).
- **Runtime warnings**: deprecation and resource warnings are errors in tests (`filterwarnings = ["error"]`, Node
  `--throw-deprecation`); ignore a third-party one by its exact message and module, with a reason.
- **Hooks and CI**: no `--no-verify`, no `SKIP=` for a failing hook, no `continue-on-error` or `|| true` on a check.
  A hook that is not a check (a version bumper) may be skipped in CI by name, with a comment.
- **Coverage and thresholds**: never lower a threshold to pass; exclude a line with a reason
  (`# pragma: no cover  # unreachable: ...`).

## Keeping it that way

- **A guard counts suppressions.** The gate fails on a suppression with no reason (example:
  `report-bot/scripts/quality/check_type_hatches.py`), so new ones are visible in review.
- **A ratchet for an existing repo.** Turn the strict mode on, record the current count of findings or suppressions
  per rule in a baseline file, and fail the gate when a count grows. Lower the baseline in every change that fixes
  some; never raise it.
- **Every exception is debt.** Per-file ignores and lowered rules go into `TECHNICAL_DEBT.md`; remove one per month
  ([tooling-and-deps.md](tooling-and-deps.md)).
- **A tool upgrade that brings new rules**: fix the findings in the same change as the upgrade. Turn a rule off only
  if it fails step 5 above, and say so in the commit message.
- **Agents fix, too.** An agent that meets a failing check fixes the cause; it never adds an ignore, a skip or a lower
  threshold just to get the gate green. If the fix is large, it files a backlog item and leaves the check failing on
  its branch.

## Why

A silenced warning hides the bug it pointed at, and the next one like it. Typed data at the boundary removes whole
classes of findings at once, where per-line ignores have to be repeated in every caller. Unused-suppression checks
keep the remaining ignores honest. A strict baseline costs most on day one; adding strictness to an old codebase later
costs more.
