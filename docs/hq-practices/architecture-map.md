---
summary: "Hub for the architecture subjects: which page to read for language, boundaries (DDD), use cases, specs, tests (TDD) and diagrams (Atlas), the order of work, and the global versus project glossary with a template"
group: docs
tags: [architecture, ddd, testing, diagrams, docs]
applies-to: [all]
reviewed: 2026-11-11
status: draft
check: "none: a map of other pages, each of which has its own check"
---

# Architecture map

Where to read about each architecture subject. The only rule of its own is the glossary split; every other rule
lives in the linked page.

## Where to read

| Subject | Page | Read when |
| --- | --- | --- |
| Words: one meaning per word | [glossary-and-vocabulary.md](glossary-and-vocabulary.md), terms in [glossary.md](https://github.com/evocaslab/hq/blob/main/practices/glossary.md) | naming a type, status, role or screen; two words compete |
| Boundaries (strategic DDD): bounded context, context map, modular monolith | [service-boundaries.md](service-boundaries.md) | adding a module, crate, repo or shared store; a split is proposed |
| Use cases; tactical DDD: entity, value object, aggregate, domain event, port and adapter | [use-cases-and-domain-model.md](use-cases-and-domain-model.md) | writing a behaviour item; a rule spans several objects |
| The item as the spec: Done when, Not in scope | [spec-driven-work.md](spec-driven-work.md) | writing or reviewing a backlog item |
| Spec of an input format or feature | [specifications.md](specifications.md) | a parser, an API or a file format is about to be built |
| Fields of an entity: ids, timestamps, archive, audit | [entity-fields.md](https://github.com/evocaslab/hq/blob/main/practices/entity-fields.md) | adding a table or a model |
| Types that cannot hold an invalid value | [parse-dont-validate.md](https://github.com/evocaslab/hq/blob/main/practices/parse-dont-validate.md) | input crosses a boundary |
| TDD: the test list, small steps, property and mutation tests | [tdd.md](tdd.md) | starting a behaviour change |
| Test craft: fixtures, e2e, flaky tests | [testing.md](testing.md) | writing or fixing a test |
| Which check catches what, type check to monitoring | [verification-layers.md](verification-layers.md) | deciding where a new check runs |
| Decision records: why a choice was made, superseded not edited | [docs-backlog-changelog.md](docs-backlog-changelog.md#decisions) | a choice is hard to reverse or a reader will ask "why" |
| Architecture tests (fitness functions): import-linter contracts, a crate-graph test | [architecture-diagrams.md](architecture-diagrams.md#invariants-as-tests) | a boundary is written in `ARCHITECTURE.md` and nothing checks it |
| Architecture docs: which views, generated views, pins to code | [architecture-diagrams.md](architecture-diagrams.md) | creating or refreshing a repo's architecture pages |
| Diagram tools, validation, text alternatives | [diagrams-as-code.md](diagrams-as-code.md) | choosing a tool for one diagram |

## Order of work

For a new part or a new behaviour, in this order; each step feeds the next:

1. **Language.** Write the nouns and statuses into the glossary before the code names them
   ([glossary-and-vocabulary.md](glossary-and-vocabulary.md)).
2. **Boundaries.** Decide which context owns the words and the data; update the context map
   ([service-boundaries.md](service-boundaries.md#context-map-template)).
3. **Use cases.** One backlog item per actor goal: main scenario, extensions as Done-when lines
   ([use-cases-and-domain-model.md](use-cases-and-domain-model.md#a-use-case-as-a-backlog-item),
   [spec-driven-work.md](spec-driven-work.md)).
4. **Tests.** The Done-when lines are the test list; one failing test at a time ([tdd.md](tdd.md),
   [testing.md](testing.md)).
5. **Diagrams.** Draw what the code now is, generated where possible, checked by a hook
   ([architecture-diagrams.md](architecture-diagrams.md)).

A small change starts at step 3 or 4; steps 1 and 2 run again only when a new word or a new store appears.

## Glossaries: global and project

Two levels, one yaml shape (`scope` or `context`, `title`, `intro`, `terms` with `id`, `term`, `definition`, optional
`icon`, `aliases`, `avoid`, `see`, `external`, `status`, `note`). The first key differs on purpose: HQ's global files
name a **scope** (a topic group such as `fleet` or `architecture`); a project file names a **context** (a bounded
context such as `billing`). "Scope" is kept off the project side because it also means "Not in scope" in a spec.

| | Global glossary | Project glossary |
| --- | --- | --- |
| Where | HQ, `practices/glossary/<scope>.yaml`, generated into [glossary.md](https://github.com/evocaslab/hq/blob/main/practices/glossary.md) | the repo, `docs/glossary/<context>.yaml`, one file per bounded context |
| Holds | words that mean the same in every repo: the workflow (session, item, landing, gate) and the architecture vocabulary (scope `architecture`: bounded context, aggregate, use case) | the ubiquitous language of that context: its domain nouns, statuses, roles, document names |
| Changed by | an HQ commit | the repo, in the commit that adds the type, status or screen |

Which term lives where:

- A word about *how the work is done* or about architecture in general is global. A repo never redefines it.
- A word about *what the product does* is the project's, even when two repos use it: each context keeps its own
  meaning ([service-boundaries.md](service-boundaries.md#bounded-context--language-boundary)).
- When a project word collides with a global one, the project term gets an `avoid` entry that qualifies it
  (`{not: "session (a sign-in)", say: "sign-in"}`); the global meaning stays.
- A project word moves to the global glossary only when it stopped being domain: a practice page uses it.

A repo points to the global glossary in two places: one line in its AGENTS.md ("Vocabulary: `docs/glossary/`;
workflow and architecture words: the HQ glossary, with its URL") and the `intro` of each context file. A project term
that mirrors a global one names it in `external`.

Template, `docs/glossary/<context>.yaml`:

```yaml
context: billing          # the bounded context's name; one file per context (not "scope")
title: Billing
intro: The words of the billing context. Workflow and architecture words are in the HQ glossary (URL).
terms:
  - id: billing-invoice   # a lower-case slug, unique in the repo; prefix it with the context
    term: invoice
    definition: One sentence - what it is, where it is stored, which command or screen shows it.
    avoid:
      - {not: "bill", say: "invoice"}
    see: [docs/architecture/context-map.md]
```

**Tooling today.** HQ's generator (`scripts/practices/practices.py`, `just practices-index`) cannot be pointed at a
repo yet: the folder is fixed to `practices/glossary/`, the outputs to `practices/glossary.md` and `.json`, and a run
also needs HQ's practices index; it reads `scope`, not `context`. A repo validates the shape by hand until the
generator has a glossary-only mode (backlog: `B-261111-glossary-tool-for-repos` in HQ).

## Atlas: the diagram tool

HQ's diagram tooling is two Rust crates; the rules for what to draw stay in
[architecture-diagrams.md](architecture-diagrams.md) and [diagrams-as-code.md](diagrams-as-code.md).

- **`atlas`** keeps an architecture model as files (format `atlas/0`: one yaml per element, relation and view) and
  checks it: `validate` (ids, references, views), `check-tags` (each element's source tag resolves to a real crate,
  folder, file, unit or compose service; code a view covers has an element), `query` (depends-on, used-by, impact,
  drift), `summary`, `diff` between two git revisions, and exports to Mermaid, LikeC4 and `hq-diagram`.
- **`hq-diagram`** renders a yaml model (`*.diagram.yaml`) deterministically to SVG, a one-file interactive page, PDF
  or PlantUML, and diffs two models.

Views Atlas draws, by the view's `style`: **C4-style boxes** (context, containers, components, chosen by a query over
the model), **flow** (UML activity: decisions, forks, swimlanes), **state** (a state machine) and **context** (the DDD
context map: contexts with their words, relations with their pattern; `atlas context-map` writes the tables into a
page and fails when they are stale). `hq-diagram` alone also draws information-flow models and an entity model
generated from the database schema. Sequence and class diagrams stay in Mermaid.

A repo adopts it by adding a workspace folder `docs/atlas/` (`atlas.yaml`, `elements/`, `relations/`, `views/`) and
running the checks from an HQ checkout with that folder as the argument: `just docs-atlas-validate <folder>`,
`just docs-atlas-check-tags <folder>`, `just docs-atlas-open <view> <folder>`. The format is version 0 and may change,
and the commit hook runs on HQ's own models only; the format is described in HQ's `docs/design/atlas-format.md`, and
HQ's own pictures are in `docs/architecture/`.

## Sources

The reasoning and the outside sources are in each linked page. Read for this page on 2026-11-11, in HQ:
`scripts/practices/practices.py` (the glossary loader and renderer), `practices/glossary/*.yaml`,
`crates/atlas/src/main.rs`, `docs/design/atlas-format.md`, `crates/hq-diagram/README.md` and `just/docs.just`.
