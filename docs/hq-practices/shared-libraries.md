---
summary: "Private shared libraries: git tags as delivery, SemVer, how consumers learn of a new version, built-in practices"
group: deploy
tags: [release, versions, deps, libraries]
applies-to: [all]
reviewed: 2026-11-05
status: draft
---

# Shared internal libraries: delivery, versions, updates

When two or more managed repos carry the same code (an auth crate, a Vue component package, an outbox), it moves into
one library repo. This page is how the library is **delivered**, **versioned** and **updated** without a paid registry.
Release mechanics of one repo are in [releases-and-versions.md](releases-and-versions.md); dependency cooldown, audit
and lockfiles in [supply-chain-security.md](supply-chain-security.md); the survey of what is duplicated today is
backlog item `B-261101-shared-libraries`. Status: in use; `evocaslab/auth` (v0.3.0, 2026-11-02: second-factor policy, WebAuthn) and `evocaslab/ui` are released, teleporto and optiomni run on `auth`. Which crates exist and why: [shared-rust-libraries.md](https://github.com/evocaslab/hq/blob/main/docs/design/shared-rust-libraries.md).

## Rules

1. **A library is its own repo** (`evocaslab/<name>`, private), one library per repo. A monorepo of libraries needs
   per-package tags and tooling; with two libraries it is more machinery than it saves.
2. **Delivery is a git dependency pinned to a tag `vX.Y.Z`**, never a branch and never a bare commit by hand. No registry.
3. **SemVer** ([semver.org](https://semver.org/)); breaking change = major (minor while `0.x`), and the CHANGELOG
   entry carries a *Migration* note. The public surface is the exported API, the config keys and the DB or wire format.
4. **One release recipe** in the library (`just release <version>`, [releases-and-versions.md](releases-and-versions.md)):
   green gate, tag equals the version in `Cargo.toml` / `package.json`, CHANGELOG section for the version, annotated tag
   pushed. Tags are never moved or deleted; a bad release is superseded by the next patch.
5. **Every consumer learns of a new tag without anyone remembering** (see "Learning of a new version").
6. **An upgrade is a branch in the consumer**, gated by the consumer's own gate, landed by its own rules. The cooldown
   of [supply-chain-security.md](supply-chain-security.md) applies (a tag younger than 7 days waits), except for a
   security fix the maintainer asks for.
7. **Each consumer has a compatibility test** that exercises the library through the consumer's real wiring (login
   through the real router; a component inside the real app shell). The library's own tests prove the library; this one
   proves the pairing.
8. **Practices are built in as tests inside the library**, not as a template to copy: the password hash parameters, the
   constant-time compare, the throttle, the focus return of a dialog are asserted by the library's tests. A consumer
   that upgrades gets the practice; a consumer that diverges has to override it visibly.

## Choosing what becomes a library

- Two or more repos carry it **and behave the same where it matters**; byte-identical copies are rare, so compare
  behaviour (inputs, outputs, errors), not diffs.
- It has a security or correctness surface worth testing once.
- It takes its inputs as arguments (`now`, keys, a lookup, probes): no hidden clock, environment, database or framework.
- Wiring stays in the consumer: config loading, error-code enums, storage. A library gives types and functions, the
  consumer maps them.
- A bug in one consumer that the library fixes (a server that ignores SIGTERM) is the best adoption argument: put it in the
  adoption item.
- Check the consumers before filing adoption items: another session may have adopted already.

## Delivery without a paid registry

| Option | Rust crate | Vue package | Verdict |
| --- | --- | --- | --- |
| Git dependency on a tag | `git = "…", tag = "v1.2.0"`; `Cargo.lock` records the commit and changes only on `cargo update` | `pnpm add github:evocaslab/ui#v1.2.0` (or `#semver:^1.2.0`); a `path:` fragment installs a subdirectory | **Default** for both |
| GitHub Packages | Not offered: the registries are npm, RubyGems, Maven, Gradle, NuGet, Docker | npm registry exists, but private packages need a classic PAT with `read:packages` (no fine-grained token), and the Free plan has 500 MB storage and 1 GB transfer per month for private repos, shared with Actions artifacts | Only for npm, only if a git dep proves too slow; not now |
| Monorepo of libraries | One workspace | One pnpm workspace with `workspace:` | Not now (rule 1) |
| Copy-in (`practices-sync`) | | | For files that are *templates* (justfile, hooks), never for code with a security surface |
| crates.io / npmjs | Public and permanent | Public | Only for libraries the maintainer decides to publish |

Why git tags win: no new service, no quota, the same access control as the code, `git log` between two tags is the diff.
The cost is that the consumer's CI needs read access to the library repo and a build compiles the library from source.

Facts that shape the details:

- Cargo takes `branch`, `tag` or `rev` for a git dependency; `Cargo.lock` pins the resolved commit and re-resolves only on
  `cargo update`. A `#fragment` in the URL is ignored. `version` next to `git` only checks, it does not select.
- pnpm accepts `github:owner/repo#<tag>`, `#semver:<range>` and `#path:/sub/dir`; for GitHub it records a git or archive
  resolution over HTTPS and never an SSH URL. Use the exact tag form, not a range, so the lockfile diff is the upgrade.
- A crate that is a dependency of another crate in the same library repo uses a `path` dependency inside it, and
  consumers depend on the top crate only.

### Access to a private library

- **Dev machine**: Cargo over SSH needs `ssh-agent` with the key and a full `ssh://git@github.com/…` URL (the built-in
  client ignores `~/.ssh/config`); simplest is `net.git-fetch-with-cli = true` in the consumer's `.cargo/config.toml`, so
  the normal `git` credentials work. pnpm uses the machine's git credentials the same way.
- **CI**: a read-only **deploy key on the library repo** (one per consumer repo, stored as a secret of that consumer) or a
  fine-grained token limited to the library repo with `Contents: read`. Never a classic PAT with wide scope, never a key in
  the repo. Load it only in the job step that fetches ([supply-chain-security.md](supply-chain-security.md#github-actions)).
- **Docker builds**: pass the key as a BuildKit secret mount, never `ARG`/`ENV`, and never leave it in a layer
  ([docker-deploy-config.md](https://github.com/evocaslab/hq/blob/main/practices/docker-deploy-config.md)).
- A library that becomes public later drops all of this; consumers change nothing but the URL.

## Learning of a new version

The consumer's lockfile pins an old tag on purpose, so something must say "v1.3.0 exists". Three mechanisms, one default:

| Mechanism | What it does | Limits |
| --- | --- | --- |
| **HQ check (default)**: a "shared library pins" subject in `just setup-outdated` | Reads each registry repo's `Cargo.toml` / `package.json` git pins, lists the library's tags with `git ls-remote --tags`, reports "pinned v1.2.0, latest v1.4.1, 2 minor behind"; the weekly timer files one backlog item per consumer and library (`area: review`, `repo:`) | Detection only: a human or a session opens the upgrade branch |
| Renovate | Cargo manager lists the `git-tags` / `git-refs` datasources; opens upgrade PRs | The docs do not say how a `tag =` pin maps to the datasource: trial on one consumer before relying on it; needs a token for private repos (`hostRules`) and a runner (the app or self-hosted) |
| Dependabot | Cargo and npm are supported ecosystems with `cooldown` and a schedule | The docs do not state support for **git-sourced** dependencies; assume none until a trial shows a PR |

**Default and why**: the HQ check. It works for every ecosystem and for private repos with the checkout HQ already has, it
does not depend on a bot's undocumented behavior, it adds no GitHub App with write access to private code, and it fits
the existing weekly report and notification path. A repo may add Renovate on top (the repo's decision, `area: review`
item, never enabled from HQ); if a trial shows it opens correct tag-bump PRs, that PR is the upgrade branch and the HQ
check stays as the safety net.

## Upgrading

1. The branch changes the tag in `Cargo.toml` / `package.json`, runs `cargo update -p <lib>` / `pnpm install`, and reads
   the library's CHANGELOG between the two tags (migration notes first).
2. The consumer's gate and compatibility test run. Red: stay on the old tag and file an item; do not patch around the
   library inside the consumer.
3. Land by the consumer's rules. Several consumers on different tags is normal; the HQ report shows the spread, and a
   major version has a deadline in its migration note after which the old line gets no fixes.
4. Fix needed in the library: branch in the library repo, release a patch, then bump the consumer. Never a `[patch]` or a
   local `path` override committed in a consumer.

## Library repo layout

`evocaslab/repo-template` is the base for tooling (justfile and `just/*.just`, `scripts/{claude,docs,git,quality,release,
setup}`, hooks, AGENTS.md, CI skeleton). A library repo is the template plus:

| Variant | On top of the template |
| --- | --- |
| Rust crate | `Cargo.toml` with `rust-version`, `edition`, features documented; `rustfmt.toml`, `rust-toolchain.toml`, `deny.toml`; hooks cargo fmt/clippy/doc/test and cargo-deny; `cargo doc` with `#![deny(missing_docs)]`; `examples/`; `tests/` with the built-in practice tests; `just release` checking tag = `Cargo.toml` version; a `semver-checks` step (`cargo-semver-checks`) to catch an accidental break |
| Vue package | `package.json` with `exports` and `files`, `pnpm-lock.yaml`, `vue` and `vuetify` as `peerDependencies` (so the consumer's single copy is used), TypeScript declarations, Vitest component tests with the a11y checks, a built `dist/` only if consumers cannot take TS sources; a Storybook-style demo page is optional; `just release` checking tag = `package.json` version |

Both variants: a `CHANGELOG.md` (Keep a Changelog) with a *Migration* line for each major, a README with the install line
(tag form) and an "Upgrading" section, the branch protection on `main`, and tags protected from deletion.

## Lessons from the first two libraries

From building `evocaslab/auth` (Rust) and `evocaslab/ui` (Vue) on the template variants, 2026-11-01.

**Demo, examples and tests (UI libraries)**

- Every runtime export has a demo page (live examples of each state, keyboard and a11y notes, light/dark switch), a
  documented snippet and a test. A test reads the export list and fails when a page is missing, so a new export cannot
  ship bare. The demo is dev-only (not in `files`), aliased to the library sources.
- Snippets come from the example files the demo renders (`import.meta.glob` with `?raw`), and the committed
  `docs/examples.md` is generated from them; a test compares the two, so docs cannot drift.
- Look at the demo in a real browser: it found two bugs that jsdom tests had passed (focus not returned to the opener
  after a model-driven `v-dialog` closes; `autofocus` ignored on an element added after page load). Pin each with a
  test that fails on the old code. jsdom stubs transitions, so tests emit Vuetify's `afterEnter` by hand.

**Template and variants**

- A variant adds stages through an optional `scripts/quality/gate-extra.sh`, not a forked `gate.sh`; the prek hooks of
  the same checks are skipped inside the gate so nothing runs twice.
- `q-integrity` flags the imported gate scripts and the template's own `|| true` lines on a new repo's first landing: expect
  one empty commit with a `Gate-change:` line, and say so in the template's README.
- `apply-variant` leaves template leftovers (`docs/changelog/*` entries, the example README, the example Python package
  and with it the `typer` dev dependency that the repo scripts import); clear them or list them as manual cleanup.
- `just setup-all` installs user-level things (shortcut, autostart); run it on purpose, not as part of a library task.
- Both checkouts have `main`: land by a fast-forward in the primary checkout, and run `just release <version> push` there.

**Releasing**

- `just release <version> [push]` makes a local annotated tag; the push is the outward step and takes an explicit word.
  Order: clean `main`, tag free and equal to the manifest version, recorded gate pass, CHANGELOG section, tag.
- `cargo-semver-checks` needs a baseline (`--baseline-rev <previous tag>`); skip it explicitly for the first release and keep it in
  `just release`, not in the gate.
- **Cooldown:** the 7-day wait is for a consumer taking a *young third-party* tag. A library's first tag, written by the
  maintainer's own session, can be adopted at once; a later patch from the same line waits unless it is a security fix.

**Rust crates**

- An MSRV-aware resolver can hold a vulnerable dependency below the MSRV: when `cargo update -p` says "0 packages", use
  `--precise` and raise `rust-version`, then record why.
- Secrets and policy types need a redacted `Debug` and a test for it. Recovery codes use rejection sampling (never
  `byte % N`), with a statistical test for the bias. A hand-rolled HMAC over the digest is checked against the RFC 4231 and
  RFC 2202 vectors.
- cargo-deny: list the allowed licenses and set `unused-allowed-license = "allow"`.

**Vue packages**

- `vue` and `vuetify` are `peerDependencies` and `devDependencies` (tests mount against the dev copy); ship `.vue`/TS sources
  through `exports`, no `dist/`, while every consumer uses Vite.
- Add `@types/node` when tests read repo files. Vitest gives an empty string for CSS `?raw` and `import.meta.url` is not a
  `file:` URL: read the stylesheet with `readFileSync(resolve(process.cwd(), …))`.
- axe on a mounted fragment: turn the page-level `region` rule off and keep every other rule.
- Testing Vuetify components in Vitest: `server.deps.inline: ["vuetify"]` (its CSS imports), stub `ResizeObserver` and
  `window.visualViewport`, mount with `createVuetify({ components, directives })`, unmount in `afterEach` (a component's
  `window` listeners outlive the test otherwise), and run axe on `[role=dialog]` excluding the closed `.v-tooltip` shells.
- Extracting app components: user-visible texts and icons come in through one injection key with English defaults
  (`uiDialogConfigKey` in `evocaslab/ui`), router/i18n/store reads become props; `vue-router` is a peer dependency when
  a composable calls `useRoute`.
- vue-tsc lags TypeScript majors (vue-tsc 3.3 fails on TypeScript 7): pin `typescript@^5`. `pnpm-lock.yaml` has several
  YAML documents: exclude it from `check-yaml`. `minimumReleaseAge` in `pnpm-workspace.yaml` is the cooldown.

**Parallel workers and releases (the `ui` app shell, 2026-11-01)**

- Parallel workers collide in append-only files (export index, changelog, demo registry). Each works in its own
  worktree; one integrator merges them one at a time and regenerates the generated files.
- Workers write their changelog note under `[Unreleased]`; the lander assigns the version and runs `just release`.
- A release recipe that needs a recorded gate pass means re-running the gate after each merge, not reusing a branch's pass.
- After resolving conflicts in a registry file run the type check before the gate; a bad merge fails there in seconds.
- Workers that only run jsdom did not look at the page: "looked at in a real browser" is its own backlog item.
- A navigation matcher matches by subtree, ignores the query string and lets the most specific entry win.

## Lessons from the next five library releases (2026-11-02)

From `auth` 0.2, `keel` 0.1 to 0.3 and `anchor` 0.1, built by parallel Sonnet builders from Haiku surveys.

- **Survey in rounds, build what two surveys agree on.** A cheap read-only survey (facts with `file:line`, ranked
  candidates, a "leave out" list) before each round; the supervisor judges, the builder gets the sources to read. A round
  also finds bugs in passing (a derived `Debug` that prints a key): file them as their own item at once.
- **Builder brief:** own worktree, no push, tag or consumer change, "report back, the supervisor lands", the gate
  through the prepared commands, and a list of design doubts in the report. Verify the diff stat, the gate result and
  any `Gate-change:` line before merging; a builder that must relax `deny.toml` (a license for a data bundle such as
  root certificates) says so in the commit.
- **A library takes lookups, not the environment:** loaders accept a `get` closure, clocks and keys come in as
  arguments, timeouts have no hidden default (`PoolConfig` and `ClientConfig` have no `Default`), so tests need no
  process state and a consumer cannot inherit a silent `unwrap_or_default()`.
- **Replace count checks by version checks** when a library verifies a migration history: "N applied" misses a gap in
  the middle; compare the sets of versions, in the order dirty, behind, gap.
- **Database tests are in the gate, not skipped:** a wrapper script starts a throwaway container for the test stage, and
  the tests fail when `CI` is set and no database is configured. Locally they print a visible skip.
- **One more outward step:** creating a repository is an outward action like a push or a tag; ask for it by name and
  expect the permission layer to ask again after an unclear "ok".
- **Adopting:** check an adoption item's premise against the code first (one item named a `_FILE` lookup that did not
  exist: drop it). Keep a piece local when taking it would change the consumer's public contract (an error code name,
  a health body that carries more fields) and file that as its own small item. A private git dependency needs
  `net.git-fetch-with-cli = true` and an `allow-git` entry in `deny.toml`, each with a `Gate-change:` line.
- **Never trust "gate passed" from a builder:** the keel builder's pass hid a clippy failure that the supervisor's rerun
  on the merged tree found. The release recipe's recorded pass for the committed tree is the check that counts.

## Using a library in the consumers

From the `ui` v0.28.0 round (2026-11-05: collapsible trigger robust against a host `button:not(.v-btn)` reset, adopted in four apps).

- **A workaround for a library control is a library bug.** Fix it in the library, release, bump every pin and delete the
  app's override in the same round; a leftover override hides the next regression.
- **Find consumers by grepping every `package.json`** (and `Cargo.toml`), nested ones included (`web/hq`, `web/jasbot`,
  `teleporto/web`), not only `*/web/package.json`.
- **Before keeping an app-local copy of a component or helper, diff it against the library** and adopt unless the
  divergence is intended; write the intent next to the copy.
- **CSS that must survive a host reset** is a library rule with tests: [vue-frontend.md](https://github.com/evocaslab/hq/blob/main/practices/vue-frontend.md).

## Sources

- Semantic Versioning 2.0.0: <https://semver.org/>
- Cargo, specifying dependencies (git, `tag`, `rev`, lockfile pinning): <https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html>
- Cargo, git authentication (ssh-agent, credential helpers, `net.git-fetch-with-cli`): <https://doc.rust-lang.org/cargo/appendix/git-authentication.html>
- pnpm, supported package sources (`github:`, `#semver:`, `#path:`): <https://pnpm.io/package-sources>
- GitHub Packages, billing (Free: 500 MB storage, 1 GB transfer for private repos): <https://docs.github.com/en/billing/concepts/product-billing/github-packages>
- GitHub Packages, supported registries and token types: <https://docs.github.com/en/packages/learn-github-packages/introduction-to-github-packages>
- Renovate, Cargo manager (datasources `git-refs`, `git-tags`, `hostRules` for private git): <https://docs.renovatebot.com/modules/manager/cargo/>
- Renovate, git-tags datasource: <https://docs.renovatebot.com/modules/datasource/git-tags/>
- Dependabot, supported ecosystems and `cooldown`: <https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories>
- RFC 4231 (HMAC-SHA-2 test vectors): <https://www.rfc-editor.org/rfc/rfc4231>; RFC 2202 (HMAC-SHA-1 and MD5): <https://www.rfc-editor.org/rfc/rfc2202>
- Keep a Changelog: <https://keepachangelog.com/>
- cargo-semver-checks: <https://github.com/obi1kenobi/cargo-semver-checks>
- MDN, CSS specificity (why a host `button:not(.x)` beats a one-class library selector): <https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity>
