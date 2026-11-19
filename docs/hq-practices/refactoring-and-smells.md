---
summary: "When to refactor, behaviour-preserving diff rules for agents, smell to lint-rule table (ruff, clippy, ESLint), the lens checklist, safe-change techniques, and where classic patterns do not apply"
group: agents
tags: [lint, review]
applies-to: [all]
reviewed: 2026-11-09
status: draft
---

# Refactoring and code smells

Refactoring is a series of small changes to internal structure that leave observable behaviour the same [1]. A code
smell is a quick-to-spot sign that usually points at a deeper problem; it is a reason to look, not a defect, and some
long functions are fine [2]. This page says when to refactor, how an agent keeps a refactor behaviour-preserving,
which linter rule counts which smell, what a reviewer still has to judge, and which classic techniques and patterns
apply in Rust, Python and Vue. The surrounding workflow is in [change-discipline.md](change-discipline.md#refactoring);
the lens is "Structure" in [review-lenses.md](review-lenses.md). Research:
[docs/research/2026-11-09-refactoring.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-09-refactoring.md).

## The rule

1. **Refactor when the change in front of you needs it**: before a feature (preparatory: make the change easy, then
   make the easy change [3]), around a bug fix, or as a review finding; the camp-site rule, only with a passing suite
   [4]. No drive-by cleanup in an unrelated diff ([change-discipline.md](change-discipline.md#keep-the-change-small)).
2. **Small steps, tests green after each.** Pin behaviour first ([testing.md](testing.md#test-first-for-behavior-changes));
   one structural move per commit; the same tests pass unchanged. A failing test means revert the step, not edit the
   test. A system that stays broken for hours is a rewrite, not a refactor [1].
3. **Extract at the second copy when it is the same concept, at the third when only the text is alike.** The rule of
   three [5] assumes humans who copy slowly; an agent copies cheaply, so duplicates spread fast.
4. **The linter counts, the lens judges.** A countable smell (length, arguments, branches) is a lint rule, not a
   review comment. Reviewer attention goes to the smells that need meaning (checklist below).
5. **Keep the tools' default thresholds** unless a repo measured a reason; the numbers are conventions, not evidence
   (below). Raising one or adding a `noqa` follows [gate integrity](review-lenses.md#gate-integrity) and
   [lint-strictness.md](lint-strictness.md#suppressions-must-stay-needed).
6. **Use Fowler's current names** (Extract Function, not Extract Method) when linking [6].

## Behaviour-preserving diff rules for agents

What goes wrong when an agent refactors: behaviour drift (an "equivalent" rewrite that is not), a fix tangled with a
cleanup, over-abstraction (a trait or base class with one user), and a huge diff nobody can review. Raw LLM
extract-method suggestions were hallucinated in up to 76 % of cases until a static check filtered them [7]; agents
solved 22 % of multi-file refactoring tasks against 87 % for a developer [8]; a refactor tangled into an agent's fix
correlates with patches that do not compile [9]. So a refactor diff follows these rules, and the reviewer checks them
before anything else:

- **Separate commit, labelled.** The message says "refactor:" and names the technique (Extract Function, Move,
  Rename, Inline). A behaviour change in the same commit is a defect of the commit, not of the code.
- **The tests are the oracle and are not touched.** Expectations, fixtures and golden files stay byte-identical. A test
  that needs changing means the step changed behaviour: revert, or split off a behaviour commit with its own test.
  Untested code gets a characterization test first ([testing.md](testing.md#characterization-tests-before-refactoring)).
- **One technique per step, the step small enough to revert.** Rename, then move, then extract; never "rewrite this
  module". A step that fails twice is reverted and its prerequisite is written down (the Mikado idea, below).
- **No new abstraction without two real users today.** A trait, base class, wrapper, option or config flag with one
  user is speculative generality; delete it. Search before building: the duplicate concept usually exists.
- **Keep the public surface.** Signatures, CLI flags, routes, file formats and log lines that something else reads
  stay; a change to them is a parallel change (expand, migrate, contract), not a refactor.
- **Say what was checked.** The commit or hand-back names the test command, the golden diff (empty) and any `git diff
  --stat` for a move, so the supervisor can re-run it ([review-lenses.md](review-lenses.md#evidence-levels)).

## Smell to rule

| Smell | Python (ruff) | Rust (clippy) | Vue / TypeScript (ESLint) |
| --- | --- | --- | --- |
| Long function | `PLR0915` (`PLR0914` preview) | `too_many_lines` | `max-lines-per-function` |
| Long parameter list | `PLR0913`, `PLR0917` | `too_many_arguments` | `max-params` |
| Branching, nesting | `PLR0912`, `PLR0911`, `C901` (`PLR1702` preview) | `cognitive_complexity` (nursery, threshold 25) | `complexity`, `max-depth`, `max-nested-callbacks` |
| Large class or component | `PLR0904` (preview) | `large_enum_variant` (size, not design) | `vue/max-props`, `vue/max-template-depth`, `max-lines` |
| Magic number | `PLR2004` | none | `no-magic-numbers` |
| Boolean flag argument | `FBT001`, `FBT003` | `fn_params_excessive_bools` | none: use a union type |
| Dead or commented-out code | `ERA001`, `ARG001`-`ARG005`, `B007`; vulture | rustc `dead_code`, `unused_*` | `no-unused-vars`, `vue/no-unused-properties` |
| Nested conditionals | `RET505`-`RET508`, `SIM102` | `collapsible_if` | `no-else-return`, `no-lonely-if` |
| Global state | `PLW0603` | none | none |
| Reaching into internals | `SLF001` | the compiler (privacy) | `vue/no-mutating-props` |

Ruff ids were checked with `uvx ruff rule` and the ESLint ones against their rule lists on 2026-11-09 (research note);
the clippy groups of the pedantic lints were not re-confirmed. Ruff `ALL` and clippy `pedantic` already enable the
Python and Rust rows ([lint-strictness.md](lint-strictness.md)); ruff's `preview` rules do not run under `ALL` without
`preview = true`. The `vue/max-*` rules are in no preset and ESLint's `complexity` and `max-*` rules need a config line
[10], [11]; the Vue repos have none today (`B-261109-vue-complexity-lint-rules`). Duplicate code: the near-duplicate
signal in [change-discipline.md](change-discipline.md#code-health-signals); a cheap proxy is an AST pass listing
top-level function names defined in three or more files.

A rule chain (`if ... return` per case) or a big `match` trips `C901`/`PLR0911` and may still be the clearest form
([change-discipline.md](change-discipline.md#code-health-signals)); a `noqa` with the reason "one rule per case" is
fine, a route or rule *table* is better when the cases share a shape.

## Lens checklist: smells a linter cannot count

Apply with the lens "Structure"; report `file:line`; file as a backlog item, do not refactor unasked.

- **Shotgun surgery**: one rule needed edits in N places. The rule has no single owner.
- **Divergent change**: one file changed for unrelated reasons (`git log --stat`). Two reasons, two modules.
- **Feature envy, message chains**: a function reads mostly another module's data, or chains `a.b.c.d`. Move it, or
  pass the needed value.
- **Data clumps, primitive obsession**: the same three or more values travel together, or a `str` carries a closed
  set. Make a struct, newtype or enum.
- **Switch on a type code** repeated in several places: one enum with the behaviour beside it, or a table.
- **Speculative generality, lazy class, middle man**: a trait, base class, wrapper or option with one user or none.
  Delete it. Agents over-abstract, so check every new abstraction first.
- **Duplicate concept** under different text: two helpers doing one thing; use the existing one ("Search before
  building").
- **Temporary field, or a flag that must stay in sync with another**: the model is missing
  ([Signs a change needs a shape](change-discipline.md#signs-a-change-needs-a-shape)).

## Techniques for a change too big for one step

| Situation | Technique | Rule |
| --- | --- | --- |
| A signature, route, schema or file format must change and callers exist | Parallel change [12] | Expand (add the new beside the old), migrate every caller, contract (delete the old). The contract step is a backlog item if it does not land in the same round; never skip it. |
| A large implementation swap must ship on main in small commits | Branch by abstraction [13] | One abstraction, old and new suppliers behind it, callers moved one by one, old supplier deleted, abstraction inlined if it has no second use. |
| An old system is replaced piece by piece | Strangler fig [14] | New beside old at a seam, traffic moved in stages, old parts deleted as they fall silent; HQ's version: [rust-web-api.md](https://github.com/evocaslab/hq/blob/main/practices/rust-web-api.md#porting-an-existing-api-strangler-fig), [retiring-the-old-stack.md](retiring-the-old-stack.md). |
| Code to change has no tests | Characterization tests [15] | Record what it does today (failing placeholder, paste the output), then refactor; a surprise is a note, not a fix, until a behaviour commit says so. |
| A step keeps failing | Mikado [16] | Revert, write the prerequisite down (a Done-when line in the item), do the prerequisite first, retry. The item's Done-when list is the graph. |
| Naming the debt | Technical-debt quadrant [17] | A backlog item says whether the shortcut was deliberate and prudent ("ship now, repay in `B-…`"), or inadvertent ("we now know a better design"); reckless shortcuts are not filed, they are fixed. |

## Patterns: when not to apply

The 22 GoF patterns are object-oriented solutions; take the idea, not the class diagram. refactoring.guru's own
criticism page says patterns compensate for weak languages and get applied rigidly [18].

| Pattern | In HQ's languages | Take |
| --- | --- | --- |
| State, Strategy, Command, Visitor | Rust enum and `match`; Python function or `Protocol`; closures | Enum or function; a trait object only for an open set |
| Factory Method, Abstract Factory, Builder | `new`/`From`; keyword arguments, dataclass | A constructor function; a builder only for many optional fields |
| Singleton | Module-level value, `OnceLock`, state passed in | Pass it in; no global mutable state |
| Observer | Vue reactivity and emits; Rust channels | The framework's own mechanism |
| Decorator, Adapter, Facade | Python decorators, Rust newtype or `impl Trait`, a thin module | At the edge, around a foreign API |
| Template Method, Iterator | Higher-order function; `Iterator`, generators | The language feature |
| Repository, Service layer, DTO mappers | A module with functions over the store; `serde`/pydantic types | Only at a real boundary (process, network, storage); not between two modules of one crate |

An abstraction needs two real users today. SOLID names are principles, not checks: a finding cites the concrete
symptom from the checklist above, not the acronym. A solo maintainer with agent workers does not need a dashboard,
a pattern catalogue or a debt register beyond the backlog; the linters, the lens and the backlog are the system.

## What the evidence says

Smell research is correlational, mostly Java and open source, and uneven across smells. Large studies find smelly
classes more change- and fault-prone [19]; a controlled experiment found file size and churn explain the maintenance
effort and no smell adds to it [20]; where a smell's effect on faults exists it is small, and Switch Statements had
none [21]. Most smells are born with the file and rarely refactored away [22]. So countable smells go to linters at
default thresholds, the lens checklist stays short, and no numeric threshold lives in this page. No study of
agent-written code beyond [7], [8] and [9] was found; the diff rules above are the response to those three.

Not taken (research note): the pattern catalogue as a checklist, SOLID as review rules, a hard rule of three, fixed
thresholds, ruff `preview` for the extra complexity rules, SonarQube-style dashboards, the Mikado method as a required
procedure.

## Sources

1. Fowler, "DefinitionOfRefactoring" and "RefactoringMalapropism": <https://martinfowler.com/bliki/DefinitionOfRefactoring.html>,
   <https://martinfowler.com/bliki/RefactoringMalapropism.html> (read 2026-11-09)
2. Fowler, "CodeSmell": <https://martinfowler.com/bliki/CodeSmell.html> (read 2026-11-09)
3. Fowler, "An example of preparatory refactoring": <https://martinfowler.com/articles/preparatory-refactoring-example.html>
   (read 2026-11-09)
4. Fowler, "OpportunisticRefactoring": <https://martinfowler.com/bliki/OpportunisticRefactoring.html> (read 2026-11-09)
5. Rule of three (Roberts, via Fowler): <https://en.wikipedia.org/wiki/Rule_of_three_(computer_programming)> (read
   2026-11-09; secondary)
6. Fowler, refactoring catalog, 2nd edition: <https://refactoring.com/catalog/> (read 2026-11-09)
7. Pomian et al., EM-Assist, arXiv 2401.15298: <https://arxiv.org/abs/2401.15298> (abstract read 2026-11-09)
8. Gautam et al., RefactorBench, arXiv 2503.07832: <https://arxiv.org/abs/2503.07832> (abstract read 2026-11-09)
9. Tian et al., "Refactoring Runaway", arXiv 2605.22526: <https://arxiv.org/abs/2605.22526> (abstract read 2026-11-09)
10. eslint-plugin-vue rules: <https://eslint.vuejs.org/rules/> (read 2026-11-09)
11. ESLint rules: <https://eslint.org/docs/latest/rules/> (read 2026-11-09); ruff: <https://docs.astral.sh/ruff/rules/>
    and `uvx ruff rule <id>` (2026-11-09)
12. Sato, "ParallelChange": <https://martinfowler.com/bliki/ParallelChange.html> (read 2026-11-09)
13. Fowler, "BranchByAbstraction": <https://martinfowler.com/bliki/BranchByAbstraction.html> (read 2026-11-09)
14. Fowler, "StranglerFigApplication": <https://martinfowler.com/bliki/StranglerFigApplication.html> (read 2026-11-09)
15. Feathers, "Characterization Testing": <https://michaelfeathers.silvrback.com/characterization-testing> (read
    2026-11-09)
16. Mikado method: <https://mikadomethod.info/> (read 2026-11-09; thin page, weak)
17. Fowler, "TechnicalDebtQuadrant": <https://martinfowler.com/bliki/TechnicalDebtQuadrant.html> (read 2026-11-09)
18. refactoring.guru, pattern catalogue and criticism: <https://refactoring.guru/design-patterns/catalog>,
    <https://refactoring.guru/design-patterns/criticism> (read 2026-11-09; vendor page)
19. Palomba et al., EMSE 2018, DOI 10.1007/s10664-017-9535-z; Khomh et al., EMSE 2012, DOI 10.1007/s10664-011-9171-y
    (summaries only, 2026-11-09; weak)
20. Sjøberg et al., "Quantifying the Effect of Code Smells on Maintenance Effort", TSE 2013, DOI 10.1109/TSE.2012.89
    (summary only, 2026-11-09; weak)
21. Hall et al., "Some Code Smells Have a Significant but Small Effect on Faults", TOSEM 2014, DOI 10.1145/2629648
    (summary only, 2026-11-09; weak)
22. Tufano et al., "When and Why Your Code Starts to Smell Bad", TSE 2017: <https://www.cs.wm.edu/semeru/data/code-smells>
    (summary only, 2026-11-09; weak)
