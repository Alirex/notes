---
summary: "How to build a README: first screen, when to split, what each platform renders (GitHub, PyPI, crates.io, npm, GitLab), generated blocks, badges, images, diagrams, and the cheapest check for each cause of drift"
group: docs
tags: [readme, diagrams, tooling]
applies-to: [all]
reviewed: 2026-11-13
status: draft
---

# README design

[docs-match-code.md](docs-match-code.md#the-readme-is-the-front-door) says the README is the front door and must be
current in the same commit. This page says how to build one that can stay current: what goes on the first screen, what
moves out, what each platform does to the file, which parts a generator owns, and which check catches which kind of
drift. Each rule carries an evidence label from [research-flow.md](research-flow.md): `strong`, `moderate`, `weak`,
`none`. Much README advice is convention with no measured effect; the page says so where that is the case.

Our own READMEs measured against these rules: [README survey 2026-11-13](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-13-readme-survey.md).

## The rules

| # | Rule | Evidence |
| --- | --- | --- |
| 1 | **First screen answers four questions in this order:** what it is (one sentence under the title), how to get it or run it (one copy-paste block), one example of use with its output, where the rest is (links). | `moderate` for the content (GitHub's list [1], Google's minimum [2], the "what" and "how" being the commonest README sections in 393 sampled repos [3]); `none` for the exact order (convention [4]) |
| 2 | **One H1, then H2 sections.** Sections past the first screen: install or run variants, configuration (a pointer to the config file and `--help`, not a copy), checks, layout, needs, license. | `moderate` (GitHub builds its outline from the headings [1]; markdownlint MD025 [5]) |
| 3 | **Split by reader, not by line count.** The README keeps what a first-time user needs. A section moves out when its reader is someone else: contributors (`CONTRIBUTING.md` or `docs/dev.md`), people changing the structure (`ARCHITECTURE.md`), people looking up every option (`docs/` reference or generated `--help` pages), people reading history (`CHANGELOG.md`). The README keeps one line and a link for each. | `moderate` (GitHub: a README holds what is needed to start, longer docs go elsewhere [1]; Google: the README is a short summary that links to the docs [2]) |
| 4 | **A size signal, not a limit:** past about 150 lines or when one H2 section is longer than the rest together, check rule 3. A table of contents is only needed where the host shows no outline (PyPI, crates.io, npm). | `weak` (Standard Readme asks for a contents list from 100 lines [4]; the numbers are our own survey's, not measured on readers) |
| 5 | **Per-package READMEs only where a package is published or entered on its own** (a crate, an npm package, a top-level folder a person opens first). Each says what the package is, its status, one usage example and a link up. No README per internal folder. | `moderate` (Google asks for one per top-level directory of a code package [2]; npm and Cargo read the README from the package root [6] [7]) |
| 6 | **Community files once per owner, not per repo:** `CONTRIBUTING`, `SECURITY`, `SUPPORT`, `CODE_OF_CONDUCT` can live in the owner's public `.github` repository and apply to every repo without its own copy. `LICENSE` must be in each repo. | `moderate` (GitHub docs [8]) |
| 7 | **A README that is published to a registry has absolute links and images in the published copy.** The repository copy keeps relative links (they follow the branch and work in a clone [1]). The published copy is generated at build time and never edited by hand. See [Platforms](#what-each-platform-renders). | `strong` for PyPI (reproduced, below), `moderate` for crates.io and npm |
| 8 | **Generate every list that has another home** (recipes, needs, CLI options, layout, crate list) between marker comments, with a check that fails when the block is stale. Prose stays hand-written. | `moderate` (tools that do exactly this [9] [10] [11]; our own `needs` block) |
| 9 | **Examples in the README run in a test.** | `strong` for Rust (rustdoc documents the pattern [12]); `moderate` elsewhere |
| 10 | **Badges: at most about five, each backed by an automated check or a registry fact** (CI status, version, docs, coverage, license). No badge that only states an intention. | `moderate` (one large study, npm only [13]) |
| 11 | **Every image has alt text that says what the image shows; a diagram or screenshot with content also has that content as text near it.** No information only in an image. | `strong` (W3C [14]) |
| 12 | **No animation that runs longer than five seconds without a way to stop it.** A GIF demo is a link to a video or a still picture with a "watch" link; or it stops within five seconds. | `strong` for the rule (WCAG 2.2.2, level A [15]); a README cannot add a pause control, so the practical form is ours |
| 13 | **Emoji never carry the meaning alone and never sit inside a sentence; at most one per heading, at its end, or none.** | `weak` (one accessibility vendor's guide [16]; no standard covers emoji) |
| 14 | **One diagram at most on the README: the context view.** Mermaid when the README is only read on GitHub or GitLab; a committed SVG with an absolute URL when it is published to a registry. The source of the picture is the architecture model, not the README. | `moderate` for where Mermaid renders [17] [18]; `none` (memory, not tested) for "PyPI, crates.io and npm do not render Mermaid" |
| 15 | **Links are checked offline in the gate and online on a schedule.** | `moderate` (lychee's documented modes [19]) |

## The first screen

A reader decides in the first screen whether to stay. What the sources agree on:

- GitHub lists what a README "typically" holds: what the project does, why it is useful, how to get started, where to
  get help, who maintains it [1].
- Google's style guide sets the minimum lower: a link to the documentation, and for a package what it is, who to
  contact, its status, how to use it [2].
- A manual classification of 4,226 sections from 393 random GitHub repositories found the "what" and the "how" in
  nearly all READMEs, while purpose ("why") and status were often missing [3]. So the usual gap is not the install
  command: it is one sentence on why the project exists and whether it is maintained.
- Standard Readme fixes an order (title, badges, short description under 120 characters, contents, install, usage,
  contributing, license last) [4]. It is a convention by one author; no study shows this order works better.

What the evidence does **not** show: that a better README brings users or contributors. A 2025 study of 4,226 READMEs
of Debian-packaged projects found that projects write minimal, usage-centred READMEs early and add `CONTRIBUTING`
after contributions arrive, not before [20]. "Good READMEs make popular projects" is folklore; the honest reason to
keep a README good is that our own agents and the maintainer read it first.

Our form, for a repo that runs (an app, a service, a CLI):

```markdown
# name

One sentence: what it is and for whom. One more: status (in use, experimental, archived).

## Quick start

| Goal | Command | Needs |
| --- | --- | --- |

(one fenced block that works from a clean clone)

## Docs

- [Development](docs/dev.md), [Architecture](ARCHITECTURE.md), [Changelog](CHANGELOG.md)
```

For a library the Quick start is the install line and one example with its output (both crates in the survey do this).

## Splitting

The survey found the same three shapes in every long README of ours:

| Shape | Example in the survey | Where it goes |
| --- | --- | --- |
| The manual inside the README (every behaviour, every option) | a CLI's "Notes", 200 of 355 lines | `docs/` page per subject; the README keeps the three commonest cases |
| Tool installation for developers (install just, uv, ruff, hooks) | a service's "Development", 220 of 324 lines | `docs/dev.md` or `CONTRIBUTING.md`; better, one `just setup-all` and the generated Needs table |
| A step-by-step guide for a non-programmer | two apps' Quick starts of 190 and 370 lines | `docs/guide/` page (it can have pictures); the README keeps the five commands |

Rules for the split:

- **A fact has one home** ([dry-and-design-principles.md](dry-and-design-principles.md)). When a section moves, the
  README keeps a link, never a summary that repeats the numbers or the option names.
- **`ARCHITECTURE.md` stays short and names things instead of linking to them**; it is revisited a few times a year,
  not synced on every change [21] ([architecture-diagrams.md](architecture-diagrams.md)).
- **Do not split for the sake of it.** A 100-line README with one reader is fine. Three files that each need the
  others open are worse than one.
- **The split is a move, in one commit, with the link check run after it.** Anchors that other repos link to
  (`#quick-start`) keep their heading text.

## What each platform renders

The same file is shown by different programs. What was read at the source:

| Platform | Reads the README from | Relative links and images | Updated when | Evidence |
| --- | --- | --- | --- | --- |
| GitHub | `.github/`, then the root, then `docs/` [1] | work; recommended, relative to the file, leading `/` is the repo root [1] | on every push | `moderate` |
| GitLab | the repository | work in repository files (not between a wiki and the repo) [18] | on every push | `moderate` |
| PyPI | the `readme` named in `pyproject.toml`, copied into the package metadata at build [22] | **not resolved**: the request to resolve them is open since 2020 with no decision [23] | only with a new release | `strong` |
| crates.io | the `readme` of `Cargo.toml`, sent on publish [7] | rewritten to the `repository` URL; fixed for READMEs in a sub-folder in 2021 [24]; an open bug since 2024 still gives wrong image URLs in some layouts, and the advice from a contributor is absolute image URLs [25] | only with a new release | `moderate` |
| npm | `README.md` in the package root [6] | rewritten for GitHub repositories, by a 2015 blog post [26]; not re-read for this page, so treat as unknown for other hosts and for files outside the package | only when a new version is published [6] | `weak` |

Consequences:

- **The registry page is a snapshot of a release; the link targets are not.** A link to `blob/main/docs/x.md` on the
  page of version 1.2 breaks when the file moves in 1.3. Pin the published copy's links to the release tag
  (`blob/v1.2.0/...`), not to `main`.
- **PyPI: check before upload.** `twine check dist/*` reports a README that will not render [22]. It does not find
  relative links; a second check greps the built `PKG-INFO` for `](` targets that do not start with `http`, `#` or
  `mailto:`.
- **Features that are GitHub-only:** alerts (`> [!NOTE]`), the `<picture>` light/dark switch [27], Mermaid [17], the
  heading outline [1]. GitLab has Mermaid, alerts since 17.10 and its own `[[_TOC_]]` and include directive [18]. On a
  registry page write as if none of them exist. `none` (memory): how each registry shows an alert block; look at a
  pre-release page once.

### How `Alirex/uv_upgrade` does it

Read in the clone (`repos/Alirex/uv_upgrade`, 2026-11-13) and confirmed on PyPI:

- `README.md` is the source, with relative links. A prek hook runs
  `mdremotifier --input README.md --output README.remotified.md --url-prefix https://github.com/Alirex/uv_upgrade/blob/main/ --img-url-prefix https://raw.githubusercontent.com/Alirex/uv_upgrade/main/`
  (`.pre-commit-config.yaml:152-162`), and `pyproject.toml:6` has `readme = "README.remotified.md"`.
- The published description of `uv-upx` 0.5.0 has six absolute `docs_extra` targets and no relative one (PyPI JSON
  API [28]). `strong`: reproduced.
- Gaps: the prefix is `main`, not the release tag; mdremotifier needs a Linux-like environment (it uses `pexpect`)
  and its last release is 1.0.0 from 2025-01-12 [29]; nothing fails when a relative link survives (the generator is
  trusted, not checked).

Three ways to produce the published copy, cheapest first:

| Way | How | When |
| --- | --- | --- |
| Write absolute links by hand in the few places that need them | no tool | a README with one or two links (a library: `LICENSE`, `CHANGELOG`) |
| A build-time rewrite | `hatch-fancy-pypi-readme`: fragments of files plus regex substitutions, `$HFPR_VERSION` for the tag, a CLI that prints the result [9] | Python packages built with hatchling |
| A committed generated copy | mdremotifier [29], or a 30-line script that prefixes relative targets; an is-current hook | any build backend; other registries |

The second and third need the "no relative target left" check, because a regex that misses a link form fails silently.

## Generated parts

A README drifts where it repeats a list that the code already holds. The fix is the one from
[generated-files.md](generated-files.md) and
[architecture-diagrams.md](architecture-diagrams.md#generated-blocks-inside-hand-written-files): the list is written
between two marker comments by a generator that has `write` and `check`.

| Block | Single source | Generator | Evidence |
| --- | --- | --- | --- |
| Needs | `needs.yaml` | `scripts/needs.py render [--check]` (vendored from HQ; in 23 of the 26 surveyed repos) | own, in use |
| Commands | the `justfile` (recipe, group, comment line) | to build: `just --dump --dump-format json` gives the data | proposed |
| Layout | a `layout:` list in a small YAML, or the top-level folders with one comment each | to build | proposed |
| CLI options | the program's `--help` | exported help pages (`uv_upgrade` `docs_extra/cli_help_*.md`), linked, not pasted | own, in use |
| Crate or package list | `cargo metadata`, the uv workspace | the diagrams generator already reads both | proposed |
| Rust crate intro | the crate's `//!` doc comment | `cargo rdme`: writes it between `<!-- cargo-rdme start -->` and `end`, `cargo rdme --check` exits 3 when stale [10] | `moderate` |
| Rust examples | the README itself | `#[doc = include_str!("../README.md")] #[cfg(doctest)] pub struct ReadmeDoctests;` runs the README's code blocks as doc tests [12] | `strong` |
| Any text computed by Python | code inside the file | Cog: code between `[[[cog` and `]]]`, output up to `[[[end]]]` [11] | `moderate` (its check flag was not read) |
| PyPI copy | `README.md` | above | `strong` |

Rules:

- **Markers name the generator:** `<!-- needs:begin (generated by scripts/needs.py render; do not edit) -->`. A reader
  who wants to change the table then knows where.
- **Wrap a generated table in `<!-- prettier-ignore-start -->` when a formatter runs on the README**, or the formatter
  and the generator rewrite each other (seen in the library repos).
- **A check, not only a writer.** A generator without `--check` in the gate is a manual step that gets skipped.
- **Do not generate the prose.** A whole README from a template reads like one, and a three-line block that has no
  second home needs no generator ([dry-and-design-principles.md](dry-and-design-principles.md), rule 3).
- **Doc comment or README as the source for a Rust crate: pick one direction.** `cargo rdme` (doc comment → README)
  keeps intra-doc links working on docs.rs but needs a pinned nightly for them [10]; `include_str!` (README → docs)
  needs nothing but shows README-only parts (badges, dev notes) on docs.rs unless it is used only under
  `cfg(doctest)`. For a small library: README as the source, tested with `cfg(doctest)`.
- `cargo-readme` was not opened for this page (`not opened`); `cargo-rdme` covers the same job with a check mode.

## Badges, emoji, pictures

**Badges.** The one large study (294,941 npm packages, a survey and regression models, ICSE 2018) found badges in 46 %
of packages and that badges for build status, coverage and up-to-date dependencies are "mostly reliable signals": they
correlate with more tests, better pull requests and fresher dependencies. The effect is strongest for *assessment*
badges, which are produced by a check, and absent for badges that only state an intention (a "PRs welcome" or
"code style" label) [13]. Limits: npm only, correlation, 2018. For us:

- Keep: CI status, registry version, docs, license (from the registry), coverage when it is measured.
- Drop: static labels, "made with", visitor counters, anything a person must update.
- A badge is an image that loads from another server on every view: each one is a request, a tracking point and a
  possible broken picture. Standard Readme suggests hosting static ones locally for that reason [4].
- A badge is a link image: its alt text says where it goes ("CI status"), by the W3C rule for functional images [14].
- crates.io removed its own badge display; badges belong in the README [7].
- A private repo needs none: nobody is choosing between it and another.

**Emoji.** A screen reader speaks each emoji's Unicode name, not the meaning intended; a row of them is a row of
announcements, and one in the middle of a sentence breaks it [16]. This is one vendor guide (`weak`); no WCAG text is
about emoji. Our diagrams page already uses one emoji per role as a repeat of shape and colour
([architecture-diagrams.md](architecture-diagrams.md#readable-diagrams)); the same limit holds here: a repeat, never the
only carrier. Emoji as bullets or in every heading is decoration with a cost and no measured gain.

**Screenshots and GIFs.**

- One still screenshot of the main screen or of the terminal output is worth its upkeep for a GUI or an interactive
  CLI: it answers "what is it" faster than text. `none`: no study found for this; it is our judgment.
- A screenshot drifts with every UI change and no check sees it. Cheapest guard: generate it (a Playwright or
  `freeze`-style script with a recipe) and put its date in the caption; when it cannot be generated, keep one.
- Alt text says what the picture shows ("Interactive mode: three proposed upgrades, each with Accept and Reject"), not
  the file name. A picture of text (terminal output) also has that text in a code block [14].
- Light and dark: GitHub documents `<picture>` with `prefers-color-scheme` sources [27]. It is GitHub-only markup; on a
  registry the `<img>` fallback is shown. Prefer one image that reads on both backgrounds (opaque background, the
  palette of [architecture-diagrams.md](architecture-diagrams.md#colour)).
- A looping GIF longer than five seconds fails WCAG 2.2.2 unless it can be paused; the documented technique is to stop
  the animation within five seconds [15]. It is also large in git forever. Link a video, or use a short non-looping one.
- Images in the repo, under `docs/images/`, compressed ([image-compression.md](https://github.com/evocaslab/hq/blob/main/practices/image-compression.md)); never a link
  to an issue attachment.

## Diagrams on the README

| | Mermaid block | Committed SVG or PNG |
| --- | --- | --- |
| Renders on | GitHub (issues, PRs, wikis, Markdown files) [17], GitLab [18] | everywhere an image does |
| On PyPI, crates.io, npm | shown as a code block (`none`: from memory, test before relying on it) | works with an absolute URL |
| Source of truth | the text itself | the model it was rendered from; needs an is-current check |
| Diff in review | readable | binary or noisy |
| Light and dark | the host switches the theme | must read on both backgrounds |
| Text alternative | `accTitle` / `accDescr` in the block | alt text plus one sentence under it |

So: a private or GitHub-only repo puts the context view as Mermaid, copied by the generator from the architecture page
into a marked block, so the README and `ARCHITECTURE.md` cannot differ. A published package uses an SVG.

**Atlas.** HQ's Atlas crate already renders one view as a static picture (`atlas render <view> --format svg`,
`crates/atlas/src/main.rs`) and exports Mermaid and LikeC4 (`just docs-atlas-export`). What a README needs on top,
filed as `B-261113-readme-atlas-export`:

1. `atlas readme <view> --page README.md [--write]`: writes a marked block (the image line, the alt text from the
   view's title and description, the one-sentence text alternative) and checks it, like `context-map` and `failures`
   do for their pages.
2. `--url-base <prefix>` (or `repository:` in `atlas.yaml`): the image line uses an absolute raw URL, pinned to a tag
   when given, so the same block works on a registry.
3. A README-safe SVG profile: opaque background, the palette's fixed text colour, no scripts or external fonts, a
   `<title>` and `<desc>` inside the file, width that fits about 800 px, and a size cap.
4. `--format mermaid` for the same block when the repo is GitHub-only.
5. The is-current check in the gate (`atlas readme --check`), so the picture cannot be older than the model.

## Checks

| Check | Tool | Where it runs |
| --- | --- | --- |
| Relative links and anchors resolve | `lychee --offline --include-fragments` over `README.md` and `docs/` (Markdown, local files, fragments; Apache-2.0 or MIT; has a pre-commit hook and a GitHub Action) [19] | gate (no network) |
| External links answer | `lychee` with its cache, exclusions in `.lycheeignore` [19] | weekly job, files a backlog item; never the gate (a third party's outage must not block a commit) |
| Structure | markdownlint: MD025 one H1, MD045 images have alt text, MD051 link fragments are valid, MD059 link text is descriptive [5] | gate |
| Generated blocks current | each generator's `--check` | gate |
| PyPI copy renders | `twine check dist/*` [22] | release check |
| Published copy has no relative target | grep over `PKG-INFO` / the file named by `readme` | release check |
| Rust README examples compile | the `cfg(doctest)` include [12] | `cargo test --doc` |
| Quick start works | run its commands in a clean clone (a container) | before landing a README change; scheduled for public repos |
| The look | open the rendered page once per change of structure: GitHub light and dark, the registry page of a pre-release | by hand |

HQ's own markdownlint config turns MD025 off for its user guides; a repo README should have it on.

## Why READMEs drift, and the cheapest check for each cause

| Cause | Sign | Cheapest check |
| --- | --- | --- |
| A list is copied from code (recipes, options, env vars, folders) | the list is right on the day it was written | generate the block, `--check` in the gate |
| A command was typed, never run | the Quick start fails on a clean machine | run the Quick start in a container (rule in docs-match-code) |
| A file moved or a heading was renamed | a 404 or a link that lands at the top of a page | offline link and fragment check |
| The registry copy is a different render | links work on GitHub, break on PyPI | generated published copy plus the "no relative target" grep |
| The registry page is frozen, the repo is not | old versions' pages point at moved files | tag-pinned links in the published copy |
| The picture shows an old UI | nobody notices | generated screenshot, or a dated caption and one picture only |
| Two docs describe one thing | README and `docs/dev.md` disagree | one home per fact; the README links |
| The README is the manual | every feature change touches it, so some do not | split by reader |
| Nobody owns it in a change | code lands, README does not | the AGENTS.md rule and the "docs" review lens ([review-lenses.md](review-lenses.md)) |
| A third party changed (badge service, external page) | broken image, dead link | weekly online link check |

The pattern behind most rows: the README holds a second copy of something. Remove the copy (link or generate) before
adding a check for it; a check on a copy is the more expensive fix.

## Public repos: extra steps

A repo that is public or published gets, on top of the above:

1. The repository description and the README's first sentence say the same thing (Standard Readme asks for this [4]).
2. License named in the first screen, `LICENSE` in the repo [8].
3. Install from the registry first (`uv tool install`, `cargo add`), from source second.
4. The published copy: absolute, tag-pinned links; `twine check` or the registry's dry run in the release check.
5. Status and support in one sentence: maintained or not, where to ask.
6. Three to five assessment badges.
7. The registry page looked at once after the first release and after any change of README structure.
8. `public: true` and `description` on the registry entry in HQ, so the checks know which rules apply.

## Not verified

- npm's present-day link rewriting (only a 2015 post was read [26]).
- Whether PyPI, crates.io and npm show a Mermaid block as code, and how they show GitHub alerts (memory).
- The exact URL form crates.io writes (`blob/HEAD`, `raw/HEAD`): the issues name it, the code was not read [24] [25].
- `cargo-readme`, Cog's check flag, "Art of README" (404 on two URLs), the older `#gh-dark-mode-only` image fragment.
- No second reader re-opened the sources: confidence `medium`. Re-check in 6 months (registry behaviour and tool
  versions change in months).

## Sources

All read 2026-11-13 unless a line says otherwise.

1. GitHub Docs, "About the repository README file" (locations, 500 KiB limit, outline, relative links, what it holds):
   <https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-readmes>
2. Google documentation style guide, "READMEs":
   <https://google.github.io/styleguide/docguide/READMEs.html>
3. Prana, Treude, Thung, Atapattu, Lo, "Categorizing the Content of GitHub README Files" (2018), abstract read:
   <https://arxiv.org/abs/1802.06997>
4. Richard Litt, Standard Readme specification (a convention, not a study):
   <https://github.com/RichardLitt/standard-readme/blob/main/spec.md>
5. markdownlint rules (MD025, MD045, MD051, MD059):
   <https://github.com/DavidAnson/markdownlint/blob/main/doc/Rules.md>
6. npm Docs, "About package README files":
   <https://docs.npmjs.com/about-package-readme-files>
7. The Cargo Book, "The Manifest Format" (`readme`, `repository`, `[badges]`):
   <https://doc.rust-lang.org/cargo/reference/manifest.html>
8. GitHub Docs, "Creating a default community health file":
   <https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file>
9. hatch-fancy-pypi-readme (fragments, substitutions, CLI):
   <https://github.com/hynek/hatch-fancy-pypi-readme>
10. cargo-rdme (markers, `--check` exit codes, intralinks):
    <https://github.com/orium/cargo-rdme>
11. Cog (markers; the check option was not on the page read):
    <https://cog.readthedocs.io/en/latest/>
12. The rustdoc book, "Documentation tests" (README doctests under `cfg(doctest)`):
    <https://doc.rust-lang.org/rustdoc/write-documentation/documentation-tests.html>
13. Trockman, Zhou, Kästner, Vasilescu, "Adding Sparkle to Social Coding: An Empirical Study of Repository Badges in
    the npm Ecosystem", ICSE 2018, pages 1-2 read:
    <https://cmustrudel.github.io/papers/icse18badges.pdf>
14. W3C WAI, "An alt Decision Tree":
    <https://www.w3.org/WAI/tutorials/images/decision-tree/>
15. W3C, Understanding WCAG 2.2 SC 2.2.2 "Pause, Stop, Hide" (technique G152 for animated GIFs):
    <https://www.w3.org/WAI/WCAG22/Understanding/pause-stop-hide.html>
16. Pope Tech, "Making emojis and icons screen reader accessible" (updated 2026-04-21; a vendor guide):
    <https://blog.pope.tech/2026/04/01/making-emojis-and-icons-screen-reader-accessible/>
17. GitHub Docs, "Creating diagrams":
    <https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams>
18. GitLab Docs, "GitLab Flavored Markdown" (Mermaid, relative links, alerts, includes, TOC):
    <https://docs.gitlab.com/user/markdown/>
19. lychee (offline mode, fragments, cache, hooks, license):
    <https://github.com/lycheeverse/lychee>
20. Gaughan, Champion, Hwang, Shaw, "The Introduction of README and CONTRIBUTING Files in Open Source Software
    Development", CHASE 2025, abstract read: <https://arxiv.org/abs/2502.18440>
21. Aleksey Kladov, "ARCHITECTURE.md":
    <https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html>
22. Python Packaging User Guide, "Making a PyPI-friendly README" (formats, `twine check`):
    <https://packaging.python.org/en/latest/guides/making-a-pypi-friendly-readme/>
23. pypa/readme_renderer issue 163, "Resolve relative links in long_description" (open since 2020-03-27):
    <https://github.com/pypa/readme_renderer/issues/163>
24. rust-lang/crates.io issue 3484 (closed 2022-01-03) and pull request 3861 (merged 2021-09-12):
    <https://github.com/rust-lang/crates.io/issues/3484>, <https://github.com/rust-lang/crates.io/pull/3861>;
    background: kornel, "PSA: Please use absolute URLs in crate READMEs" (2020-06-29),
    <https://users.rust-lang.org/t/psa-please-use-absolute-urls-in-crate-readmes/45136>, and rust-lang/cargo issue
    8434 (closed 2025-02-03 in favour of the next source), <https://github.com/rust-lang/cargo/issues/8434>
25. rust-lang/crates.io issue 9939, "Wrong link to image in markdown" (open, 2024-11-14):
    <https://github.com/rust-lang/crates.io/issues/9939>
26. npm blog, "Nicely presented markup" (2015-01-29; old):
    <https://blog.npmjs.org/post/109508231330/nicely-presented-markup.html>
27. GitHub Docs, "Quickstart for writing on GitHub" (`<picture>` with `prefers-color-scheme`), and the GitHub Blog
    post of 2025-04-18:
    <https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/quickstart-for-writing-on-github>,
    <https://github.blog/developer-skills/github/how-to-make-your-images-in-markdown-on-github-adjust-for-dark-mode-and-light-mode/>
28. PyPI JSON API, `uv-upx` 0.5.0 description (own check of the published links):
    <https://pypi.org/pypi/uv-upx/json>
29. mdremotifier 1.0.0 on PyPI (read through the JSON API; the HTML page did not load):
    <https://pypi.org/project/mdremotifier/>
