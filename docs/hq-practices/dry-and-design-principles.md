---
summary: "DRY means one authoritative home per piece of knowledge, not no repeated text; with its neighbours (rule of three, YAGNI, orthogonality, generate-don't-copy, strict input) and where each one backfires"
group: quality
tags: [architecture, review]
applies-to: [all]
reviewed: 2026-11-12
status: draft
---

# DRY and the design principles next to it

Hunt and Thomas define DRY as "every piece of knowledge must have a single, unambiguous, authoritative representation
within a system" [1]. Their test is a change, not a diff: when one fact changes, do you edit several places, maybe in
several formats (code and docs, a schema and the struct that holds it)? Then the system is not DRY. Identical code
that stands for two different facts is a coincidence, not duplication [1]. Read every rule below that way: it is about
knowledge (a rule, a list, a constant, a format), never about lines that happen to look alike.

Neighbours: refactoring steps and smells in [refactoring-and-smells.md](refactoring-and-smells.md) (its rule 3 is the
rule of three for agents), types in [parse-dont-validate.md](https://github.com/evocaslab/hq/blob/main/practices/parse-dont-validate.md) and
[strict-schemas.md](https://github.com/evocaslab/hq/blob/main/practices/strict-schemas.md), generated output in [generated-files.md](generated-files.md), shared code
across repos in [shared-libraries.md](shared-libraries.md), and the "Design rules that save later rewrites" in
[change-discipline.md](change-discipline.md#design-rules-that-save-later-rewrites). This page does not repeat them.

## The rules

Each rule has one line on where it pays and one on where it backfires.

1. **One authoritative home per fact.** A rule, a status list, a threshold, a price or a file format lives in one
   place; every other place reads it, generates from it, or links to it [1].
   *Applies:* facts that change together: enums shared by two languages, a registry format, a model price table.
   *Backfires:* code that is only alike. Two validators with the same body for two different fields stay two [1].
2. **Name the home when a copy is unavoidable, and guard it.** A second language, a self-contained script or an
   external API forces a copy (Hunt and Thomas: "representational duplication" is inevitable [1]). Write
   "mirrors X" at the copy and add a check that fails when they differ (a parity test, a hook that compares the two).
   *Applies:* a Rust crate that mirrors a Python checker; a vocabulary file checked against constants in code.
   *Backfires:* a "mirrors" comment with no check is a promise, not a guard; it drifts like an unmarked copy.
3. **Generate, don't copy.** When a list or table must appear in several formats (README table, JSON catalogue,
   diagram, help text), keep one input and generate the rest, with an is-current hook [2]
   ([generated-files.md](generated-files.md), [architecture-diagrams.md](architecture-diagrams.md)).
   *Applies:* anything a person would otherwise update in three files by hand.
   *Backfires:* a generator for one copy of three lines costs more than it saves; a hand-written page around a
   generated block keeps the prose hand-written.
4. **Extract when the concept is the same; wait when only the text is.** The rule of three [3] and AHA, "avoid
   hasty abstractions" [4], say wait until the shape is known. An agent copies cheaply, so a copy of the same
   concept is extracted at the second use ([refactoring-and-smells.md](refactoring-and-smells.md#the-rule)).
   *Applies:* the second copy of a business rule or a file format.
   *Backfires:* extracting at the second copy of a coincidence gives the wrong abstraction (rule 5).
5. **Prefer duplication to the wrong abstraction, and undo one fast.** Metz: "duplication is far cheaper than the
   wrong abstraction" [5]. The sign is a shared helper that grows a parameter and a branch per caller. The fix is
   to inline it back into each caller, delete what each one does not use, and extract again from what is left [5].
   *Applies:* a helper with flags that select whole code paths; a base type every user overrides.
   *Backfires:* as an excuse to never extract; copies of one fact still drift (rule 1).
6. **Tests read on their own (DAMP over DRY).** A test is read when it fails; setup hidden in helpers costs more
   than a few repeated lines [6]. Share fixtures and builders, not the assertions or the scenario.
   *Applies:* test bodies, golden cases, table tests (one row per case reads fine).
   *Backfires:* copying a large fixture into every test; that is one fact (the fixture) and gets a builder.
7. **Orthogonality: unrelated things do not change together.** "Things that are not related conceptually should
   not be related in the system" [2]. A change to one part (storage, a screen, a CLI flag) should not need edits in an
   unrelated part. *Applies:* module boundaries, config keys, recipe groups.
   *Backfires:* splitting one concept across modules for symmetry; that is change amplification, the first symptom
   of complexity in Ousterhout's list [7].
8. **YAGNI for features, not for health.** Do not build for a presumed need: it costs build, delay, carry and
   repair [8]. YAGNI "does not apply to effort to make the software easier to modify" [8], so a refactor, a test or
   a guard is not YAGNI. *Applies:* options, plugin points, config switches with no user today.
   *Backfires:* a known, dated need (a second machine next month) is not presumptive; neither is a choice that adds
   no complexity [8].
9. **Deep modules over small ones (KISS by interface).** Simple means a small interface over real work, not few
   lines: a thin wrapper that only forwards adds interface without hiding anything [7]. Count what a reader must
   hold in mind, not lines [7].
   *Applies:* helpers, crates, recipes: one entry point that hides the steps.
   *Backfires:* hiding a decision callers need (a timeout, a destructive default) behind a "simple" call; that
   creates unknown unknowns, the worst symptom [7].
10. **Make illegal states unrepresentable.** Give each state its own type with only its own fields, so a wrong
    combination cannot be written [9] ([parse-dont-validate.md](https://github.com/evocaslab/hq/blob/main/practices/parse-dont-validate.md)). It is DRY for invariants:
    the type states the rule once instead of every caller checking it.
    *Applies:* statuses with data per status, request and config models.
    *Backfires:* readers of other people's data stay lenient ([parse-dont-validate.md](https://github.com/evocaslab/hq/blob/main/practices/parse-dont-validate.md#exceptions)).
11. **Strict on input you own; Postel only at a boundary you cannot fix.** "Be liberal in what you accept" lets
    quirks become de facto standards and hides bugs; RFC 9413 asks for fixing the source and failing loudly instead
    [10]. *Applies:* config files, front matter, internal APIs, generated files: reject and name the line.
    *Backfires:* external data you cannot get fixed (a third-party feed): accept, log it as a problem, do not drop it.
12. **Separation of concerns and convention over configuration.** Reason about one concern at a time [11]; where a
    choice is made once for every repo (layout, recipe names, file names), make it a convention so nobody decides it
    again [12]. *Applies:* the folder layout, `<prefix>-<action>` recipe names, front matter keys.
    *Backfires:* a convention nobody wrote down is a hidden rule; write it on one page and check it. Deviate when a
    repo measured a reason, and say so in that repo [12].
13. **Least surprise.** A name, flag or default does what a reader expects from its name and from its neighbours;
    the same word means the same thing everywhere ([glossary-and-vocabulary.md](glossary-and-vocabulary.md)).
    *Applies:* CLI flags, recipe verbs, status names.
    *Backfires:* keeping a surprising legacy name forever; rename it once, with an alias, rather than explain it in
    every doc.

## Rules in docs and agent instructions

Instructions are knowledge too, and agents copy them faster than code. A rule lives on one practices page; a repo's
AGENTS.md keeps a short binding line that links there ([agent-instructions.md](agent-instructions.md)); a skill links
to the rule and repeats only the command it runs. The binding line is a deliberate, named copy (rule 2): the check is
that it links to its home. A list of things that exist (skills, recipes, modules) is generated from the things or
checked against them, never only typed.

## Where DRY is the wrong call

- **Self-contained scripts.** A single-file uv script ([just-task-runner.md](just-task-runner.md)) repeats a small
  helper such as a `git()` wrapper rather than import a shared module; a five-line wrapper carries little knowledge.
  A file format or a registry layout is knowledge: that one gets one reader (a shared module vendored with a check,
  or a CLI the scripts call).
- **Across service or repo boundaries.** Two services with a copied type stay deployable alone; share the contract
  (a schema, OpenAPI), not the internals ([service-boundaries.md](service-boundaries.md),
  [shared-libraries.md](shared-libraries.md)).
- **YAML anchors and merge keys.** Anchors and aliases are YAML 1.2 core syntax; the `<<` merge key is a YAML 1.1
  type that never became part of the 1.2 spec [13]. Prefer a named `extends:` or `defaults:` in the tool's own
  schema, which can check cycles and override order, over `<<`.

## Checklist

- [ ] Each fact the change touches has one named home; copies are generated or carry a check.
- [ ] A new helper has at least two users of the same concept, and no flag that picks a whole code path.
- [ ] A list of existing things (skills, recipes, statuses) in a doc is generated or checked.
- [ ] New config or options have a user today.
- [ ] Input the repo owns fails loudly with file and line.

## Sources

1. Thomas and Hunt, *The Pragmatic Programmer*, 20th anniversary edition, "The Evils of Duplication" (sample
   chapter): <https://media.pragprog.com/titles/tpp20/dry.pdf> (read 2026-11-12; DRY definition, "DRY is about the
   duplication of knowledge, of intent", "Not all code duplication is knowledge duplication", representational and
   interdeveloper duplication)
2. Venners, "Orthogonality and the DRY Principle", interview with Hunt and Thomas:
   <https://www.artima.com/articles/orthogonality-and-the-dry-principle> (read 2026-11-12; code generators keep "single,
   authoritative representations", orthogonality)
3. Rule of three (Roberts, via Fowler's *Refactoring*):
   <https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming)> (secondary)
4. Dodds, "AHA Programming": <https://kentcdodds.com/blog/aha-programming> (read 2026-11-12; WET as rigid as DRY)
5. Metz, "The Wrong Abstraction" (2016): <https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction> (read 2026-11-12)
6. Google Testing Blog, "Tests Too DRY? Make Them DAMP!" (2019):
   <https://testing.googleblog.com/2019/12/testing-on-toilet-tests-too-dry-make.html> (read 2026-11-12; body thin in
   the fetch, title and recommendation)
7. Ousterhout, *A Philosophy of Software Design*, 2nd edition (2021): change amplification, cognitive load, unknown
   unknowns, deep modules, pass-through methods. Reading notes used for the chapter list:
   <https://danlebrero.com/2021/02/24/philosophy-of-software-design-summary/> (secondary)
8. Fowler, "Yagni": <https://martinfowler.com/bliki/Yagni.html> (read 2026-11-12)
9. Minsky, "Effective ML Revisited", "Make illegal states unrepresentable":
   <https://blog.janestreet.com/effective-ml-revisited/> (read 2026-11-12)
10. Thomson and Schinazi, RFC 9413, "Maintaining Robust Protocols" (IAB, 2023):
    <https://www.rfc-editor.org/rfc/rfc9413.html> (read 2026-11-12)
11. Dijkstra, EWD447, "On the role of scientific thought" (1974):
    <https://www.cs.utexas.edu/~EWD/transcriptions/EWD04xx/EWD447.html> (read 2026-11-12)
12. Rails Doctrine, "Convention over Configuration": <https://rubyonrails.org/doctrine> (read 2026-11-12)
13. YAML merge key type, YAML 1.1 working draft (2005): <https://yaml.org/type/merge.html> (read 2026-11-12); YAML
    1.2.2 spec: <https://yaml.org/spec/1.2.2/>
