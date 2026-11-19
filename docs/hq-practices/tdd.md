---
summary: "TDD for a maintainer with agent workers: the test list is the item's Done when, one test at a time in small steps, mutation on changed lines, property tests for parsers, contracts as types and asserts, branch age as a trunk measure, a walking skeleton first"
group: quality
tags: [testing, workflow, agents]
applies-to: [all]
reviewed: 2026-11-09
status: draft
---

# Test-driven development and its neighbours

What to take from TDD, BDD, property-based and mutation testing, Design by Contract, trunk-based development and
the walking skeleton for one maintainer whose code is mostly written by agent sessions. The test set-up itself
(fixtures, containers, flaky tests, e2e) is in [testing.md](testing.md); how big a spec an item needs is in
[spec-driven-work.md](spec-driven-work.md); the tests among the other verification layers:
[verification-layers.md](verification-layers.md). Sources and evidence levels:
[docs/research/2026-11-09-ddd-and-related.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-09-ddd-and-related.md).

## The rule

| Rule | Check |
| --- | --- |
| **The item's Done-when lines are the test list.** Before code, the list of scenarios exists (Beck's first step); for a behaviour change each line becomes one test. | the hand-back maps each Done-when line to a test name or to "not testable: why" |
| **One test at a time, small steps.** Turn one line into a failing test, make it and the earlier tests pass, then refactor while green. The step is small enough that a failure has one cause. | a test that never failed is not evidence; a commit that adds many tests and the code at once is a smell at review |
| **Behaviour, not structure.** A test drives the code the way its user does and checks what that user sees. Tests of private structure are deleted, not kept for coverage. | the two-edits check of [change-discipline.md](change-discipline.md): break the behaviour, it fails; restructure the internals, it still passes |
| **Mutation testing on the changed lines**, after a change to a parser, a rule or a check, not whole modules monthly. | the hand-back of such a change names the surviving mutants or "none" |
| **A property test for every parser, serializer and round trip.** The example tests pin known cases; the property test searches the input space and shrinks the failure. | each `parse`/`render`, `encode`/`decode` pair has a round-trip property; each parser a "never panics on arbitrary input" property |
| **Contracts as types and asserts, not defensive checks.** Parse at the boundary into a type that cannot hold an invalid value ([parse-dont-validate.md](https://github.com/evocaslab/hq/blob/main/practices/parse-dont-validate.md)); inside, state the invariant once (`debug_assert!`, `assert`, an `expect("...")` naming it); never re-check what the type already guarantees. | no duplicate validation of a value the type has parsed; a stated invariant per non-obvious assumption |
| **Branch age is the trunk measure.** A work branch merges `main` daily and lands within two days of its first commit, review wait included; older branches are reported by the round, not left. | `cc-fleet-round` lists branches over two days old |
| **A walking skeleton first** for a new app or service: the thinnest end-to-end slice (one route, one row, one screen) through the gate and the deploy before any feature. | the first landed item of a new app is the skeleton, with its deploy and health check |

## The loop, for an agent

Beck's canon [1]: write a list of the test scenarios; turn exactly one into a concrete, runnable test; change the code
so that test and all earlier tests pass; optionally refactor; repeat until the list is empty. The misreadings he names
are the ones agent sessions make: launching into code without the list, turning the whole list into tests before any
passes, assertion-free tests for coverage, copying the computed value into the expected value, mixing refactoring
into the passing step, abstracting on the first duplication.

Here the list already exists: the item's Done-when lines ([spec-driven-work.md](spec-driven-work.md)), each "when X,
Y" line being a scenario in North's givens, event, outcomes form [8]. The worker's loop is one line, one failing test
seen to fail for the expected reason, the smallest change, green, refactor, next line. The hand-back maps lines to
tests. A bug fix starts with the test that reproduces it ([testing.md](testing.md#test-first-for-behavior-changes)).

The evidence supports the *shape* of the loop more than its *order*: in the controlled study the test-first order had
no important effect, the granularity and uniformity of the steps did [3]; the industrial case studies that reported
40-90 % fewer defects also reported 15-35 % more initial time, and are case studies, not experiments [4] (weak). So
the rule is small, steady, checked steps with the list first; "always test-first" is not a rule here.

## The pyramid as a cost rule

Many fast unit tests, fewer through the API, very few end to end; a check goes as low in the pyramid as it can and
is not duplicated at a higher level [5], [6]. End-to-end tests are brittle, slow and non-deterministic; a bug they find
is reproduced with a unit test first [5]. TDD and self-testing code are different things; heavy mocking to make TDD
work is design damage, and the trade-offs differ by context [7]. The explainer and quiz are on
[testing.md](testing.md).

## Mutation testing on changed lines

A mutant is a code change no test notices. Google runs mutation testing diff-based (only the changed lines), samples
the mutants and suppresses "arid" ones, and surfaces the survivors in code review [9]; a whole-codebase run is
neither needed nor affordable. The same shape here: after a change to a parser, a ranking rule or a check, run the
tool on the changed functions (cargo-mutants [10]; mutmut re-tests only mutants in functions whose source changed
[11]), review the survivors, kill the ones that change observable behaviour with a test, ignore equivalent ones. The
run stays outside the gate, behind the heavy lock ([testing.md](testing.md#mutation-testing)).

## Property-based tests

QuickCheck states properties as functions over generated input, with custom generators where the default is wrong,
and gets good coverage from fine-grained properties over pure functions [12]. Hypothesis and proptest bring the same
to Python and Rust and shrink a failing input to a minimal case [13], [14]. Property tests complement example tests:
the examples pin known edge cases and past bugs, the properties search for the inputs nobody thought of [14].

Where they pay: parsers (front matter, YAML models, CLI output), serializers and their round trips, ordering and
ranking rules (the sort is stable and total), merges and deltas (commutative where they claim to be), anything with
an invariant that must hold for every input. A parser gets two properties at least: it never panics, and
`render(parse(x)) == x` or `parse(render(v)) == v`. Keep the generators narrow enough that a failure means a bug.

## Contracts as types and asserts

Meyer's Design by Contract: defensive "just in case" checks add code, hence new failure points, hence more checks; a
routine is an implementation of a specification with a precondition (what every call must satisfy), a postcondition
(what the call ensures) and class invariants (what holds whenever a client can see the object) [15]. The Eiffel
guidance keeps precondition checks on and switches postcondition and invariant checks off once a class is trusted
[16].

Without a framework: the precondition is the parameter type (a parsed newtype, an enum, a non-empty collection), so
an invalid call does not compile or does not parse; the postcondition is the test; the invariant is one
`debug_assert!` or `assert` with a message that states it, or the type's constructor. One check at the boundary, none
inside; a re-check of a parsed value is deleted at review.

## Branch age as the trunk measure

Trunk-based development keeps one trunk, merges to it at least daily and keeps feature branches under about two days,
review wait included [17]; DORA associates three or fewer active branches, daily merges and no code freezes with
higher delivery performance [18]. The flow here (one branch per item in a worktree, a full gate, auto-land, pull
main; [git-worktrees-and-autoland.md](git-worktrees-and-autoland.md)) is trunk-style; the missing part is the number.
So: merge `main` into the work branch daily, land within two days, and let the round report the branches that are
older, with their item. A branch that cannot land in two days is an item that was too big: split it, land the first
slice. Feature flags are not taken: nothing here ships dark over several landings.

## Walking skeleton

For a new app or service, build the thinnest slice that touches every architectural part (one route, one table row,
one screen, one log line) and take it through the gate, the container and the deploy with a health check before any
feature. The sources for the term are secondary (attributed to Cockburn; his own text was not found, so weak), but
the practice is the first item of every new repo here: it proves the pipeline, gives every later item a place to
land, and is kept and grown, unlike a prototype.

## Studied and not taken

- Gherkin, Cucumber, pytest-bdd: the value of BDD is the conversation; the executable examples are a side effect
  [19]. One maintainer and agents have the conversation in the item; the Given/When/Then *form* is taken for
  behaviour lines, the tooling is not.
- A TDD mandate or a test-first metric: the evidence favours step size over order [3].
- DbC frameworks: types and asserts do it without a dependency.
- Mutation testing in the gate: slow, all cores; it stays a heavy-lock job after the changes that need it.
- Coverage thresholds as a universal: already in the practices register (`docs/design/practices-system.md`).
- Feature flags: no parallel feature work that must ship dark.

## Sources

1. Beck, "Canon TDD" (2023-12-11), read 2026-11-09: <https://newsletter.kentbeck.com/p/canon-tdd>
2. Fowler, "TestDrivenDevelopment" (2023-12-11), read 2026-11-09: <https://martinfowler.com/bliki/TestDrivenDevelopment.html>
3. Fucci et al., "A Dissection of the Test-Driven Development Process: Does It Really Matter to Test-First or to
   Test-Last?" (2016), abstract read 2026-11-09: <https://arxiv.org/abs/1611.05994>
4. Nagappan, Maximilien, Bhat, Williams, "Realizing quality improvement through test driven development", Empirical
   Software Engineering 13(3), 2008 (not opened; secondary summary, weak).
5. Fowler, "TestPyramid" (2012-05-01), read 2026-11-09: <https://martinfowler.com/bliki/TestPyramid.html>
6. Vocke, "The Practical Test Pyramid" (2018-02-26), read 2026-11-09:
   <https://martinfowler.com/articles/practical-test-pyramid.html>
7. Fowler, Beck, Heinemeier Hansson, "Is TDD Dead?" (2014), read 2026-11-09: <https://martinfowler.com/articles/is-tdd-dead/>
8. North, "Introducing BDD" (2006), read 2026-11-09: <https://dannorth.net/blog/introducing-bdd/>
9. Petrović, Ivanković, "State of Mutation Testing at Google" (ICSE SEIP 2018), abstract read 2026-11-09:
   <https://research.google/pubs/state-of-mutation-testing-at-google/>
10. cargo-mutants, read 2026-11-09: <https://mutants.rs/>
11. mutmut documentation, read 2026-11-09: <https://mutmut.readthedocs.io/>
12. Claessen, Hughes, "QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs" (ICFP 2000), abstract
    and introduction read 2026-11-09: <https://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quick.pdf>
13. Hypothesis documentation, read 2026-11-09: <https://hypothesis.readthedocs.io/en/latest/>
14. proptest book, introduction, read 2026-11-09: <https://proptest-rs.github.io/proptest/intro.html>
15. Meyer, "Applying 'Design by Contract'", IEEE Computer 25(10), 1992, key sections read 2026-11-09:
    <https://se.inf.ethz.ch/~meyer/publications/computer/contract.pdf>
16. Eiffel, "Design by Contract and Assertions", read 2026-11-09:
    <https://www.eiffel.org/doc/solutions/Design_by_Contract_and_Assertions>
17. "Trunk Based Development" (home, short-lived feature branches, feature flags; community site, weak), read
    2026-11-09: <https://trunkbaseddevelopment.com/>
18. DORA, "Trunk-based development", read 2026-11-09: <https://dora.dev/capabilities/trunk-based-development/>
19. Cucumber, "Behaviour-Driven Development", read 2026-11-09: <https://cucumber.io/docs/bdd/>
20. Research note with the full register: [docs/research/2026-11-09-ddd-and-related.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-09-ddd-and-related.md)
