---
summary: "Catalogue of rules a machine checks: the guard test, hook or lint per rule, where it exists, copy-ready snippets"
group: quality
tags: [tests, lint]
applies-to: [all]
reviewed: 2026-11-07
status: stable
---

# Guards: rules a machine checks

A rule in a page is advice until a test or lint fails when it is broken. This page lists, per rule that can be
checked mechanically, the guard to copy. A repo adds the guard with the rule's short binding line in its AGENTS.md.
See also [testing.md](testing.md#guard-tests-over-prose) (how to write a guard) and [quality-gate.md](quality-gate.md)
(where it runs). Open gaps per repo: `just docs-gaps`. Status checked 2026-10.

## Catalogue

| Rule (page) | Guard | Where it exists |
| --- | --- | --- |
| Security headers on every response ([web-app-security.md](https://github.com/evocaslab/hq/blob/main/practices/web-app-security.md)) | Test: request an API route, a 404 and the SPA index; assert the header set | HQ `security_headers_on_every_response`, `crates/hq-api/src/app.rs` |
| Every route is authenticated or on the public list ([web-app-security.md](https://github.com/evocaslab/hq/blob/main/practices/web-app-security.md)) | Test: walk the route table | a Python service's route-table test |
| Every FK column is indexed ([database-performance.md](https://github.com/evocaslab/hq/blob/main/practices/database-performance.md)) | Test: catalogue query (below) | HQ `every_foreign_key_column_is_indexed`, `crates/hq-db/src/lib.rs` |
| New SQLite tables are `STRICT` ([strict-schemas.md](https://github.com/evocaslab/hq/blob/main/practices/strict-schemas.md)) | Hook on migration files | HQ `scripts/quality/check-migrations-strict.sh` |
| Landed migrations never change ([db-migrations.md](https://github.com/evocaslab/hq/blob/main/practices/db-migrations.md)) | Hook: diff against the merge base | HQ `scripts/quality/check-migrations-immutable.sh` |
| Outbound calls have a timeout ([jobs-and-integrations.md](https://github.com/evocaslab/hq/blob/main/practices/jobs-and-integrations.md)) | Lint: ruff `S113` (requests without timeout), `ASYNC` rules; Rust: clippy `disallowed_methods` for `reqwest::Client::new` | none yet |
| Lists are bounded ([database-performance.md](https://github.com/evocaslab/hq/blob/main/practices/database-performance.md)) | Test: call each list endpoint with 250 rows, assert at most the cap | none yet |
| Graceful shutdown ([deploy-and-operations.md](https://github.com/evocaslab/hq/blob/main/practices/deploy-and-operations.md)) | Test: start the server, send `SIGTERM`, assert exit within the bound | HQ `crates/hq/tests/shutdown.rs`, a Rust web service's `backend/tests/shutdown.rs` |
| Config fails fast ([deploy-and-operations.md](https://github.com/evocaslab/hq/blob/main/practices/deploy-and-operations.md)) | Test: each required setting missing gives an error naming it | a Rust web service's `config.rs` tests |
| No raw HTML sinks ([web-app-security.md](https://github.com/evocaslab/hq/blob/main/practices/web-app-security.md)) | ESLint `vue/no-v-html` as error | none yet |
| a11y ([accessibility.md](https://github.com/evocaslab/hq/blob/main/practices/accessibility.md)) | ESLint `vuejs-accessibility/*` as errors; axe in Playwright | none yet |
| i18n key parity ([i18n.md](https://github.com/evocaslab/hq/blob/main/practices/i18n.md)) | Test: same key set in every locale; `@intlify/vue-i18n/no-raw-text` | HQ `messages.spec.ts`, a Vue app's `usedKeys.spec.ts` |
| OpenAPI file is current ([testing.md](testing.md)) | Hook: regenerate and diff | HQ `openapi.json is current` |
| Never weaken a gate to get green ([review-lenses.md](review-lenses.md#gate-integrity)) | Script on the branch diff: new skip/xfail/`only`, fewer assertions or tests, raised timeouts, `--no-verify`, `continue-on-error`, bare suppressions; passes only with a `Gate-change:` commit line | HQ `just q-integrity`, `scripts/quality/check_gate_integrity.py` |
| No destructive shell command by accident ([agent-security.md](agent-security.md#command-guard)) | `PreToolUse` Bash hook: deny `rm -r` on `/` and `~`, force-push to main; ask on `reset --hard`, `--no-verify`, `DROP TABLE`; deny sleep loops and `sleep` of 30 s or more | HQ `scripts/claude/guard/guard.sh` |
| Dependencies audited ([supply-chain-security.md](supply-chain-security.md)) | `just q-audit`, `q-deny`, secret scan in the hook | HQ gate |

## Snippets

### FK index (SQLite, Rust or any language)

For every table run `PRAGMA foreign_key_list(t)`, collect the first column of each `PRAGMA index_list(t)` index
(`PRAGMA index_info`), plus the single-column primary key, and fail on a foreign key column not in that set. The HQ
test is the reference. PostgreSQL: join `pg_constraint` (`contype = 'f'`) with `pg_index` on the constraint's
first column; SQLAlchemy: `inspect(engine).get_foreign_keys(t)` against `get_indexes(t)`.

### Timeouts (Python, Ruff)

```toml
[tool.ruff.lint]
extend-select = ["S113", "ASYNC", "PLW1514"]
```

`S113` flags `requests`/`httpx` calls without `timeout`. Wrap clients once and ban direct construction.

### Frontend (ESLint flat config)

```js
import vueA11y from 'eslint-plugin-vuejs-accessibility'
import vueI18n from '@intlify/eslint-plugin-vue-i18n'

export default [
  ...vueA11y.configs['flat/recommended'],
  ...vueI18n.configs['flat/recommended'],
  { rules: { 'vue/no-v-html': 'error', '@intlify/vue-i18n/no-raw-text': 'error' } },
]
```

### axe in Playwright

```ts
import AxeBuilder from '@axe-core/playwright'
test('home has no serious a11y issues', async ({ page }) => {
  await page.goto('/')
  const { violations } = await new AxeBuilder({ page }).analyze()
  expect(violations.filter((v) => ['serious', 'critical'].includes(v.impact ?? ''))).toEqual([])
})
```

The snippets are references from the plugins' documentation and were not run in every repo; run the repo's gate
after adding one and fix what it finds instead of silencing the rule ([lint-strictness.md](lint-strictness.md)).

## Pick the level

A mistake becomes a class after its second occurrence (commits, reverts, review findings, repeated corrections). Fix
the class at the highest level that works, in this order:

1. Architecture: one owner, one supported way; delete the old way an agent would copy.
2. Types: the bad state cannot be written.
3. A lint or check whose error names the file or function to use instead; ratcheted when the pattern is already common.
4. A behaviour test.
5. A doc line, only for judgment calls: nothing fails when an agent skips it.

Prove a new guard on the real past mistake: it must fail on the code that had the mistake. A suppression carries a
reason, an expiry date and the maintainer's approval, and the gate fails on an expired one.

## Review the prompts

An agent that stops at a permission prompt costs the maintainer a decision and the run a wait. Treat every prompt as
a defect and fix it by class, in this order:

1. A prepared command exists: the guard message names it (`just repos-finish`, `just machine-kill-own`), and the
   briefing of the sessions says so.
2. The same chain repeats: make it a prepared command ([prepared-commands.md](prepared-commands.md)).
3. The form is safe but the guard cannot tell: add a proof (merged or pushed branch, a path under the worktree) and
   judge every part of a chained command, not only the first match.
4. A real risk: keep the prompt and make its message say what to do instead.

Log every ask and deny with the masked command and the reason, and rank the log (`just cc-guard-asks`), so the
prompts that repeat are fixed, not the ones remembered. Each fix gets a test of the allowed form, of the unsafe
neighbour, and an eval task. The `/guard-care` skill is this loop. A weekly timer without an LLM runs the ranking for
you: it re-checks each prompt shape against the guard as it is now, writes the still-asking ones with a class guess and
a next step to one report, and files a single backlog item when a shape asked three times, so nobody has to paste
screenshots (HQ: `just cc-guard-weekly`, `~/.local/state/hq/guard/weekly.md`).

## Hook semantics (Claude Code)

A hook is a deterministic guard on the agent itself; the model cannot talk its way past it (read 2026-11-07; see Sources).

- A `PreToolUse` hook that exits 2 blocks the call and feeds its stderr back to the model, so the message must say what
  to do instead. Exit 0 does not approve: the normal permission flow still applies. With several hooks on one call the
  most restrictive answer wins.
- Hook types: `command`, `prompt`, `agent` (experimental), HTTP. Prefer `command` for anything that must be reliable.
- Evidence over claims: a hook or brief asks for the command that was run and its output, not "tests pass".
- A `Stop` hook that runs the check and exits 2 on failure keeps the agent working until the check passes. Claude Code
  lets the agent stop after 8 consecutive blocks with no tool call in between, so the hook is a nudge with a cap, not a
  hard gate; the real gate stays the commit hook and the landing run ([quality-gate.md](quality-gate.md)).
- HQ today: a `PreToolUse` Bash guard (`scripts/claude/guard/guard.sh`) and a preflight gate on session and agent
  starts (`.claude/settings.json`); no `Stop` hook. A `Stop` hook running a fast type check is a candidate, not a rule:
  measure first that it saves worker round trips and that it does not run a heavy check on every turn.

## Rules

1. A rule that can be checked gets a guard in the same change that adds the rule or the code.
2. A guard fails with the offending names, not a count.
3. A guard is never skipped to land a change; the rule or the code changes.
4. New pages with a checkable rule add a row to the catalogue.

## Sources

- pstack (MIT, Lauren Tan): <https://github.com/cursor/plugins/tree/main/pstack> (`correct`)
- ESLint flat config: <https://eslint.org/docs/latest/use/configure/configuration-files>
- Ruff rules (`S113`, `ASYNC`): <https://docs.astral.sh/ruff/rules/>
- Clippy `disallowed_methods`: <https://rust-lang.github.io/rust-clippy/master/index.html#disallowed_methods>
- axe-core Playwright: <https://playwright.dev/docs/accessibility-testing>
- Hooks: exit code 2 blocks and returns stderr, exit 0 does not approve, most restrictive wins, hook types, a Stop hook
  with `|| exit 2`, the override after 8 consecutive blocks (read 2026-11-07): <https://code.claude.com/docs/en/hooks-guide>
- Ask for the command run and its output as evidence (read 2026-11-07): <https://code.claude.com/docs/en/best-practices>
