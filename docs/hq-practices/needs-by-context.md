---
summary: "needs.yaml: what a repo needs to run, develop, test, build in CI and release (tools, display, GPU, services, network), checked and rendered by one vendored script"
group: quality
tags: [setup, ci, gate, docs]
applies-to: [all]
reviewed: 2026-11-08
status: stable
check: "script: needs-file"
---

# Needs by context: `needs.yaml`

A repo states what it needs, **per context**, in one small file at its root: `needs.yaml`. Without it the needs are found
by failure: a GUI test that dies without a display, a gate that wants Chromium, a CI job missing one apt line. The
survey of 26 repos (2026-11-08) found 14 that need something no README, setup recipe or workflow names (display and GL
in a desktop app, Chromium in two Vue repos, a Docker socket, Postgres, a GPU, a private git dependency's token).
Package-level dependencies (dev-dependencies, dependency groups, `devDependencies`) stay in their native files and never
go here ([docs-match-code.md](docs-match-code.md): one place per fact).

## Binding rule

- Every registry repo has `needs.yaml` (a docs-only repo uses the five-line template below).
- A new tool, package, service, display, GPU, network, env or data need is added to it **in the same commit** that
  introduces it.
- The checker is vendored as `scripts/needs.py` (one self-contained [uv](tooling-and-deps.md) script: Python 3.11+,
  standard library and `pyyaml`, no HQ import) so it runs on a GitHub-hosted runner and on any machine.
  `just practices-sync <repo>` copies HQ's `scripts/repos/needs.py` there; a copy that differs from HQ's fails
  `just repos-conformance`.
- The setup-check tool list and the README "Needs" table are **generated** from the file between markers
  (`scripts/needs.py render`); `render --check` is the freshness gate. Never edit between the markers.

## Contexts

| Context | Meaning |
| --- | --- |
| `run` | a user runs the released program |
| `develop` | edit, build and run the gate on a developer machine |
| `test` | what the tests themselves need (a display, a database, a browser) |
| `ci` | the hosted runner (usually `develop` + `test` minus the desktop) |
| `release` | build the artifacts for every target |

A need lists every context it belongs to. An optional `part` narrows it inside the contexts (`gui`, `screenshots`,
`db-tests`, `windows-exe`), so "the CLI and core tests need no display" is expressible without a sixth context.

## Schema (version 1)

```yaml
# needs.yaml: what this repo needs, by context. Convention: HQ practices/needs-by-context.md (v1).
# contexts: run | develop | test | ci | release.  level: required | optional.  Never a secret value.
version: 1
needs:
  - kind: tool            # tool | package | hardware | service | network | env | data
    name: cargo           # tool/hardware/service/data: the command or item; optional for package/network/env
    pinned_by: rust-toolchain.toml   # tool only: the native file that holds the version (the checker reads it)
    contexts: [develop, test, ci, release]
    part: gui             # optional sub-scope
    level: required       # required | optional (optional needs a `fallback`)
    why: builds every crate           # one line, at most 110 characters, mandatory
    check: command        # what the checker runs, see below
    fix: install rustup   # optional hint the generated setup-check shows when the tool is missing
    os: [linux]           # optional: only on these OSes (linux, macos, windows)
```

Extra keys per kind (any other key is a lint error):

| Kind | Extra keys | Checks |
| --- | --- | --- |
| `tool` | `pinned_by`, `min` | `command` (on PATH; `min` compared with `--version`), `none` |
| `package` | `apt`, `brew`, `dnf` (lists) | `package` (`dpkg -s`, `brew list`, `rpm -q`), `none` |
| `hardware` (`display`, `gpu`, `memory`, `disk`) | `protocols`, `virtual_ok`; `api`, `software_ok` | `display` (Wayland or X11, or `xvfb-run` when `virtual_ok`), `gpu` (a render node, `nvidia-smi` for `cuda`; `software_ok` turns a miss into "software"), `none` |
| `service` | `provided_by` (`throwaway-container`, `compose`, `env:VAR`), `version` | `docker` (`docker info`), `service` (the env var is set, or a container can start), `none` |
| `network` | `hosts`, `login` (`gh`, `token:VAR`) | `url` (HEAD, only with `--network`), `none` |
| `env` | `names` (upper-case names), `secret`, `example` | `env` (set; the value is never printed) |
| `data` | `path`, `url`, `fetched_by` | `file` (exists), `none` |

Words are borrowed on purpose: `required`/`optional` from Debian and CycloneDX, `x11`/`wayland`/`opengl` from
Flatpak and Snap, `gpu` and `optional` from Dev Containers. No existing cross-ecosystem convention covers display, GPU,
services and test needs (see Sources).

### Point at the pin, never copy a version

A tool whose version a native file pins carries `pinned_by: <file>[#key.path]` and no `min`. The checker parses
`rust-toolchain.toml` (`toolchain.channel`), `.python-version`, `.node-version`, `package.json#engines.node`,
`pyproject.toml#tool.uv.required-version`; lint fails when the pin file is missing or yields nothing, or when `pinned_by`
and `min` are both set. `min` is for tools with no native pin (just, prek, cargo-deny, docker, gh, jq).

### Rules that keep the file honest

- `why` is mandatory and one line: it is what stops the list growing. Padding is rejected by lint, not by taste.
- `optional` needs a `fallback` (what happens without it: "software GL", "the test is skipped with a reason").
- `check: none` is allowed and listed ("not checked") so a network or run-time service can be stated without
  pretending to verify it.
- `env` holds names only, never values. Needs every HQ repo shares (git, `gh`) are written out in each file: no include
  mechanism, the file reads on its own.

## Example: Rust desktop GUI with a Docker-only reference model

```yaml
version: 1
needs:
  - {kind: tool, name: cargo, pinned_by: rust/rust-toolchain.toml, contexts: [develop, test, ci, release],
     level: required, why: builds every crate, check: command, fix: install rustup}
  - {kind: tool, name: just, min: "1.40", contexts: [develop, test, ci, release], level: required,
     why: every command is a recipe, check: command}
  - {kind: tool, name: docker, contexts: [develop, test], part: r-model, level: optional,
     fallback: the R parity test is skipped with a reason, why: the R reference model runs only in its pinned image,
     check: command}
  - {kind: package, apt: [libxkbcommon-dev, libwayland-dev, libxcb1-dev], contexts: [develop, ci, release],
     part: gui, os: [linux], level: required, why: eframe links winit's X11 and Wayland backends, check: package}
  - {kind: hardware, name: display, protocols: [wayland, x11], contexts: [run, develop], part: gui, os: [linux],
     level: required, why: the GUI opens a window; the CLI and core need none, check: display}
  - {kind: hardware, name: display, virtual_ok: true, contexts: [test, ci], part: screenshots, level: required,
     why: --screenshot renders real frames; run it under xvfb-run on a server, check: display}
  - {kind: hardware, name: gpu, api: opengl, software_ok: true, contexts: [run, develop, test], part: gui,
     level: optional, fallback: Mesa llvmpipe software GL, why: the glow renderer needs a GL context, check: gpu}
  - {kind: network, hosts: [crates.io, static.crates.io], contexts: [develop, ci, release], level: required,
     why: the first build fetches crates; later builds work offline with --locked, check: none}
```

## Example: Python and uv repo with a browser and a database in its tests

```yaml
version: 1
needs:
  - {kind: tool, name: python, pinned_by: .python-version, contexts: [develop, test, ci], level: required,
     why: the interpreter uv installs, check: command}
  - {kind: tool, name: uv, contexts: [develop, test, ci], level: required, why: runs every script, check: command}
  - {kind: service, name: docker-daemon, contexts: [develop, test], part: db-tests, level: optional,
     provided_by: throwaway-container, fallback: set TEST_DATABASE_URL to a Postgres, check: docker,
     why: the tests start a throwaway Postgres}
  - {kind: env, names: [TEST_DATABASE_URL], contexts: [test, ci], level: optional,
     fallback: a throwaway container is started, why: reuse a running Postgres, check: env}
  - {kind: data, name: chromium, fetched_by: "uv run playwright install chromium", path: ~/.cache/ms-playwright,
     contexts: [test, ci], part: e2e, level: required, why: the browser tests, check: file}
  - {kind: package, apt: [libnss3, libnspr4, libasound2t64], contexts: [test, ci], part: e2e, os: [linux],
     level: required, why: Chromium's shared libraries, check: package}
```

A docs-only repo's file (five needs; `scripts/repos/needs.template.yaml` in HQ): git, uv, prek, `gh` (optional), Node
for markdownlint (optional).

## How to check

```bash
scripts/needs.py check --context test          # this machine against the file, one line per need
scripts/needs.py check --context ci --json     # machine-readable; exit 1 on a MISS in that context
scripts/needs.py lint                          # schema, vocabularies, pins, `why`; exit 1 on a problem
scripts/needs.py render --check                # generated README table and setup-check block are current
scripts/needs.py render                        # rewrite them (appends a "Needs" README section with markers)
scripts/needs.py ci                            # workflows against the declared ci needs
```

States of `check`: `ok`, `virt` or `soft` (a virtual display or software GL stands in), `opt` (an optional need is
missing: the fallback applies), `MISS` (a required need is missing), `n.a.` (other OS), `skip` (`check: none`).
`--part` limits the run to one sub-scope, `--network` also probes URLs.

## How CI is checked

The same script runs on the runner (`scripts/needs.py check --context ci`) and `scripts/needs.py ci` compares the
active `.github/workflows/*.yml` with the `ci` needs: the apt packages installed in `run` steps equal the declared `apt`
lists (declare or remove), `xvfb-run` appears iff a `ci` display need has `virtual_ok`, and `playwright install` appears
iff a `ci` need names it in `fetched_by`. A repo whose CI is only a draft (`workflow-drafts/`, `_workflows/`) has no
active workflow: the check is `n.a.`, which is itself a finding. Not compared yet: `services:` blocks.

`just repos-conformance` runs five checks over every clone: `needs-file` (exists), `needs-lint`, `needs-vendored` (the
copy of `scripts/needs.py` equals HQ's), `needs-ci`, `needs-generated` (README block, and the setup-check block when it
has markers).

## Generated blocks

- README: a table between `<!-- needs:begin ... -->` and `<!-- needs:end -->` (need, contexts, level, why).
- Setup-check: the `tool need|want <cmd> "<fix>"` lines between `# [needs]-[BEGIN]` and `# [needs]-[END]`, from the
  `tool` needs with the `develop` context (`required` is `need`, `optional` is `want`).

## Checklist for adopting it

1. Copy the template (or run `just practices-sync <repo>`, which adds `scripts/needs.py` and a template `needs.yaml`
   when none exists) and edit it for the repo.
2. `scripts/needs.py lint`; add a prek hook running `lint` and `render --check`.
3. `scripts/needs.py render`; commit the generated blocks.
4. Make the workflows match (`scripts/needs.py ci`) and add one line to the repo's AGENTS.md linking this page.

## Sources

- Research report of 2026-11-08 (Fable, 26 clones surveyed): the survey table, the mechanism comparison and the schema
  proposal this page follows.
- PEP 735 dependency groups: <https://peps.python.org/pep-0735/>; PEP 725 external dependencies (draft):
  <https://peps.python.org/pep-0725/>
- Cargo manifest and `[package.metadata]`: <https://doc.rust-lang.org/cargo/reference/manifest.html>
- pixi system requirements: <https://pixi.prefix.dev/dev/workspace/system_requirements/>
- Dev Containers `hostRequirements`: <https://containers.dev/implementors/json_reference/>
- Debian Policy ch. 7 (Depends, Recommends, Suggests): <https://www.debian.org/doc/debian-policy/ch-relationships.html>
- CycloneDX 1.6 `scope`: <https://cyclonedx.org/docs/1.6/json/>
- Flatpak sandbox permissions: <https://docs.flatpak.org/en/latest/sandbox-permissions.html>; Snap desktop interfaces:
  <https://snapcraft.io/docs/desktop-interfaces>
- Electron headless CI (`xvfb-run`): <https://electronjs.org/docs/latest/tutorial/testing-on-headless-ci>
- Playwright CI (`install --with-deps`): <https://playwright.bootcss.com/docs/ci>
- egui CI and kittest: <https://raw.githubusercontent.com/emilk/egui/main/.github/workflows/rust.yml>,
  <https://docs.rs/egui_kittest>; Vello CI (llvmpipe and lavapipe apt list):
  <https://skia.googlesource.com/external/github.com/linebender/vello/+/4c6f75f499a225c4a97df71ade47b0ccc3a5096b/.github/workflows/ci.yml>
- Related: [desktop-app-folders-config-logs.md](https://github.com/evocaslab/hq/blob/main/practices/desktop-app-folders-config-logs.md),
  [rust-egui-gui-for-agents.md](https://github.com/evocaslab/hq/blob/main/practices/rust-egui-gui-for-agents.md), [quality-gate.md](quality-gate.md).
