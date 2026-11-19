---
summary: "Version scheme by consumer, version+sha at runtime, local release script, update notice, rollback"
group: deploy
tags: [release]
applies-to: [all]
reviewed: 2026-11-16
status: stable
---

# Releases, versions and rollback

How code gets from `main` to the running bot or a published package, and how to get back. Landing on `main` is in
[git-worktrees-and-autoland.md](git-worktrees-and-autoland.md); migrations at release in
[db-migrations.md](https://github.com/evocaslab/hq/blob/main/practices/db-migrations.md); the image side (build stamp, tags equal to the commit) in
[docker-deploy-config.md](https://github.com/evocaslab/hq/blob/main/practices/docker-deploy-config.md); per-branch changelog files in
[docs-backlog-changelog.md](docs-backlog-changelog.md); libraries other repos depend on in
[shared-libraries.md](shared-libraries.md).

## Two kinds of repo

- **Deployed service** (a web service): a release is "this commit runs in production". No version bump per deploy; the
  identity is the commit, shown by the status route next to the manifest's SemVer ([a service gets a version too](#the-release-step)). Tag each deployed commit `deploy-YYYYMMDD-HHMM`: cheap, gives
  rollback targets and "what changed since".
- **Published package or CLI** (a CLI, templates): a release is a SemVer tag `vX.Y.Z` plus a published
  artifact. The version lives in one place (`pyproject.toml` / `Cargo.toml`) and the tag must equal it; a gate
  step fails when they differ.

## The scheme follows the consumer

Who reads the version decides what it is. Private does not remove the need: it moves it. The version is for people and
for bug reports and results; the commit is the identity. Every kind below keeps **one version source of truth**
(workspace `Cargo.toml`, `pyproject.toml`, `package.json`; never a second copy), **a tag per release**, **a changelog**,
and **the version plus git sha (and a dirty flag) visible in the running thing**.

| Kind | Version | Identity shown | Release |
| --- | --- | --- | --- |
| Library other repos use | SemVer; breaking = major (minor in 0.x) | version + tag | tag + changelog entry |
| CLI / desktop tool | SemVer, one workspace version | `--version` prints `X.Y.Z (sha[-dirty])`; About shows the same | tag + `SHA256SUMS`; a local script when there is no CI |
| Deployed service | none to bump; `deploy-*` tag | commit on the status route | as above |
| Research model / paper code | SemVer; a minor bump when model numbers change, listed in the changelog with the golden checksums | version + sha + dirty flag in every output folder; `CITATION.cff` at the root | tag per paper submission |

- **Stamp the build, do not type it.** Compile `git describe --always --dirty` into the binary (a small `build.rs`
  printing `cargo:rustc-env=…`, or the `vergen` / `git-version` crates; Python: `importlib.metadata` plus a build-time
  sha). Fall back to `unknown` when there is no `.git` (archive builds). A hand-written stamp goes stale.
- **Results carry their origin.** A run writes `version`, `git describe`, dirty flag, seed and config hash into its
  output folder (a manifest file), so a number in a paper traces to a commit. A dirty tree is recorded, not refused.
- **`CITATION.cff`** (Citation File Format) at the root of research code; GitHub shows "Cite this repository". Where no
  DOI exists, cite repo URL plus the commit or tag.
- **Private repo without CI: a local release script** (`scripts/release.sh`, `just dist-release`). It refuses a dirty
  tree, a version without a `CHANGELOG.md` entry and an existing tag; builds the archives, writes `SHA256SUMS`, makes an
  annotated tag `vX.Y.Z`; `--push` pushes the tag and, if `gh` is logged in, creates the Release. Nothing leaves the
  machine without `--push`. No new CI workflow for a private repo; a workflow stays manual-only. Reference:
  `auto-joint-lab/combat_models` (`scripts/release.sh`, `CHANGELOG.md`).
- **Tooling choice.** `release-plz` is CI-first; `cargo-release` is the CLI tool when per-crate bumps and `Cargo.lock`
  handling start to hurt; `cargo-dist` ignores `publish = false` crates unless `dist = true`. For a private workspace
  of a few crates the local script is the least machinery.
- **Not applicable:** docs-only or notes repos and config repos with no artifact keep no version; say so in the repo's
  AGENTS.md instead of adding one.

## The release step

A release is one planned step after a green landing (stage 6a of [agent-workflow.md](agent-workflow.md#stages)), not a
by-product of landing:

1. **Pick the level** by SemVer (see Rules) from the changelog entries since the last tag.
2. **Bump the one source** and every mirror in the same commit: workspace `Cargo.toml` (members inherit with
   `version.workspace = true`), `package.json`, `CITATION.cff`; refresh the lock file. Never leave two literal copies
   that differ.
3. **Changelog**: a dated entry for the version (`CHANGELOG.md`, or the per-branch files in `docs/changelog/` rolled up).
4. **Check the stamp**: build, run `--version` and open About; they print `X.Y.Z (sha)` and the clean tree has no
   `-dirty`. The sha is the release commit.
5. **Tag** the release commit `vX.Y.Z` (annotated; a service gets `deploy-YYYYMMDD-HHMM` as well), push the tag. Then
   publish only if the repo's rules allow (see "Publishing" below).

Where the repo has a recipe (`just release-check` / `just release <version>`, or `scripts/release.sh`), it does 2 to 5
and refuses a dirty tree, a missing changelog entry and an existing tag; without one, the steps above are the checklist.

**Checked by machine.** `just repos-conformance` runs `version-one-source` over every clone: the literal versions in
`Cargo.toml` (workspace or package), `pyproject.toml`, `package.json` and `CITATION.cff` are equal (`0.0.0` placeholders
and inherited versions do not count), a `CHANGELOG.md` or `docs/changelog/*.md` exists, and a repo that builds a binary
or a web app stamps the commit (`build.rs`, `vite.config`, `Dockerfile`, `justfile` mention `git describe`, `rev-parse`,
`vergen` or a `*_COMMIT` variable). It cannot run the binary: step 4 stays a human or test step. A repo with no
artifact says `Versioning: none` in its AGENTS.md and the check is `n.a.`. A monorepo that deliberately versions parts
separately (apps, packages) lists that in its AGENTS.md and is reported by the check until it chooses one source: pick
a workspace version for everything that ships together, or split the repo.

**A service gets a version too.** The "Deployed service" row's identity is still the commit, but the manifest carries a
SemVer for people and bug reports, and the status route and About show `version + sha`; only the *bump per deploy* is
dropped.

## Web page versions and the update notice

A web page and its server ship separately (a bundle on disk, a binary or image), and a tab stays open for days. So
the page carries its own version, notices a new deploy while open, and checks that the server speaks its API.
Reference: HQ (`web/hq`, `crates/hq-api/src/app.rs`).

| Piece | Rule | HQ |
| --- | --- | --- |
| Build stamp | SemVer from `package.json`, the git commit (`HQ_WEB_BUILD_COMMIT` in a Docker build, else `git`, else `unknown`), build time and a dirty flag, compiled in with Vite `define` | `vite.config.ts` (`__BUILD_*__`) |
| `build.json` | The build writes the same stamp next to `index.html` | `vite.config.ts` (`hq-build-info`) |
| Caching | `index.html` and `build.json`: `Cache-Control: no-cache`; hashed files (`static/web/`): `immutable` for a year; a missing build file is a 404, never the SPA shell | `page_fallback` |
| Deploy check | Poll `build.json` uncached (`cache: 'no-store'`) on focus and every five minutes; stale = both commits known and different; a failed or odd answer gives no verdict | `api/version.ts`, `version/stale.ts` |
| Notice | "Update available: version X (sha) is deployed" with a Reload button and tooltip; it stays until the reload, and a dot on the account avatar plus a Reload item at the top of its menu and an alert in the About dialog keep it reachable after the snackbar is gone | `StaleTabSnackbar.vue`, `AccountMenu.vue`, `VersionCard.vue` |
| API revision | The page is generated against an API revision; `/version` reports the server's. Older server: a warning ("update the server"), newer is fine (the API only grows inside `/v1`) | `version/compat.ts`, `ServerOlderSnackbar.vue` |
| Where the version lives | Not on every screen: a short "Page X · sha" under About in the account menu, and the About dialog with page and server side by side (version, commit, build time, dirty, API revision). Support needs it a click away; everyday work does not need it in view | `AccountMenu.vue`, `VersionCard.vue` |

Why each part:

- **The commit, not the SemVer, decides staleness.** Agents land many commits between version bumps; the version is
  for people, the commit is the identity ("Deployed service" above).
- **`no-cache` on the shell is what makes Reload work.** Without a `Cache-Control` header browsers apply heuristic
  freshness to `index.html` (a share of its age since `Last-Modified`), so a reload can load the old shell, the old
  bundle, and show the notice again. Hashed names make the long cache on the rest safe.
- **The check is off in plain dev** (the bundle is the working tree, there is no build file) but on against the mock
  API, which can imitate a deploy (HQ: `window.hqMock.deploy()` in `pnpm dev:mock`), so the notice can be seen
  without building.
- **Reload is the user's choice**, never automatic: an open form would lose its edits.

Tests that pin it: unit tests for the stale verdict, the `build.json` parser and the fetch (404, HTML shell,
offline → no verdict); component tests that a simulated deploy shows the notice in the snackbar and the account menu; a
server test for the three `Cache-Control` values and the 404s.

Desktop apps and CLIs have the same need in another shape: show the version (`--version`, an About box) and check
for a newer release on start, never on every call.

## A `prod` branch for auto-deployed services

When agents auto-land on `main` and a land redeploys the live service (for example a systemd path unit watching
`refs/heads/main`), every land is a production deploy that nobody chose. Split the two:

- **`main` stays the trunk**: agents land there as today ([git-worktrees-and-autoland.md](git-worktrees-and-autoland.md)),
  the dev or smoke stack follows it.
- **`prod` is what runs in production**, and the deploy watcher follows `refs/heads/prod` instead of `main`. It only
  ever moves by **fast-forward to a commit already on `main`**, through a recipe (`just release`): green gate on that
  commit, smoke checks on the dev stack, then `git push origin <sha>:prod` and the `deploy-*` tag. `prod` is always an
  ancestor of `main`, so it never needs a merge and never drifts.
- **Promoting is a maintainer decision** (or a standing rule written in the repo's AGENTS.md, e.g. "promote after the
  nightly smoke run"); agents never push `prod` on their own. A rollback is moving `prod` back to the previous
  `deploy-*` tag (`git push --force-with-lease origin deploy-…:prod`, plus the migration rule below).
- **Hotfix when `main` holds work not ready for prod**: land the fix on `main` as usual, cherry-pick it onto a
  `hotfix/*` branch cut from `prod`, fast-forward `prod` to that branch, then merge `prod` back into `main` so `prod`
  is an ancestor again. Rare by design: prefer feature flags, or promote all of `main` once it is green.
- **Not git-flow**: no `develop`, `release/*` or long-lived branches with their own commits; that adds merges and
  drift for no gain with one deployment. A repo with no live deployment (HQ, templates) or one that ships packages
  (a CLI: SemVer tags from `main`) needs no `prod` branch at all.
- Protect `prod` on GitHub (no direct pushes except the release recipe's actor, no force pushes except rollback):
  a settings change, so a `manual-action` for the maintainer.

## Rules

- **Cut releases from a green `main`**, by a recipe (`just release-check`, then `just release <version>`), not
  hand-typed commands. The recipe refuses a dirty tree, a branch other than `main`, a failing gate, or a tag that
  already exists.
- **Generated output carries its version too**: the tool `VERSION` and a hash of the inputs go in the header of every
  generated file ([architecture-diagrams.md](architecture-diagrams.md#generated-views)); bump the tool's `VERSION` when
  its behaviour changes.
- **SemVer for packages**: breaking change = major (minor while `0.x`), feature = minor, fix = patch. A CLI flag,
  an exit code, an output format scripts parse and a config key are public surface.
- **Release notes come from the changelog files** landed since the last tag, not from `git log`. A branch without
  a changelog entry is a gate finding, not a gap to fill at release time.
- **Publishing** ("Publishing" in the release step): the maintainer's step until the repo has trusted publishing: the release skill stops at "tag
  and notes ready" and files a `manual-action` issue ([maintainer-questions.md](maintainer-questions.md)).
- **Deploy order for a service**: build images from the tag, take the pre-migration backup, run the init
  (migrate) step, start, wait for readiness, run the smoke checks, tag. Anything red before "start" aborts with the
  old release still running.

## Rollback

- Keep the previous release's images (tag = commit) and the pre-migration backup. A rollback is
  `TAG=<previous> docker compose up -d`; if a migration ran, **restore the backup first** (migrations are
  forward-only). Both commands go into the runbook ([observability-and-incidents.md](https://github.com/evocaslab/hq/blob/main/practices/observability-and-incidents.md)).
- Test a rollback once per service on the smoke stack, like a restore ([backup-and-restore.md](https://github.com/evocaslab/hq/blob/main/practices/backup-and-restore.md)).
- When users are affected, revert the change on `main` fast; fix forward in calm.

Toolchain pins (language, uv, node, base images) are released like code: own branch, green gate
([tooling-and-deps.md](tooling-and-deps.md)).

## Sources

- Semantic Versioning 2.0.0: <https://semver.org/>
- Keep a Changelog (entries grouped by type): <https://keepachangelog.com/>
- RFC 9111 HTTP Caching (heuristic freshness, `no-cache`): <https://www.rfc-editor.org/rfc/rfc9111#section-4.2.2>
- MDN, Cache-Control (`no-cache`, `immutable`): <https://developer.mozilla.org/docs/Web/HTTP/Headers/Cache-Control>
- SemVer or CalVer by consumer: <https://dev.to/christian98/choosing-the-right-versioning-scheme-its-about-who-consumes-your-software-1i3>, <https://frontside.com/blog/2022-02-09-semver-or-calver-by-project-type/>
- Citing versions and commits, `CITATION.cff`: <https://www.software.ac.uk/blog/standard-format-citation-files>, <https://citation-file-format.github.io/>
- Rust release tools: <https://release-plz.pages.dev/docs/why>, <https://github.com/crate-ci/cargo-release>, <https://github.com/axodotdev/cargo-dist>
- Build stamps: <https://docs.rs/vergen/latest>, <https://crates.io/crates/git-version>
- Vite, `define` (compile-time constants): <https://vite.dev/config/shared-options#define>
