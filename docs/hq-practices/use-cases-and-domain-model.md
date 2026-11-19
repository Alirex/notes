---
summary: "A use case (actor, goal, main scenario, extensions) written as a backlog item and a test list, and the tactical DDD blocks worth knowing (entity, value object, aggregate, domain event, port and adapter) with where each lives in code"
group: docs
tags: [architecture, ddd, testing, agents, backlog]
applies-to: [all]
reviewed: 2026-11-11
status: draft
check: checklist
---

# Use cases and the domain model

Two things the other architecture pages leave out: how a piece of behaviour is written down before it is built (a
use case), and the names for the parts of a domain model (tactical Domain-Driven Design). Both are taken only as far
as they help one maintainer who hands work to agent workers. The hub for the neighbouring subjects is
[architecture-map.md](architecture-map.md); boundaries between contexts are in
[service-boundaries.md](service-boundaries.md).

## The rule

| Rule | Check |
| --- | --- |
| A behaviour item names its **primary actor and goal** in the title, at the user-goal level. | The title reads actor, verb, object ("Operator exports the month's report"); the actor can stop after it and be done. |
| The item's body has the **main success scenario** in 3 to 9 steps; every **extension** (a condition the system can detect and must handle) is one Done-when line, "when X, Y". | Each Done-when line maps to a test name in the hand-back ([tdd.md](tdd.md)). |
| One use case is **one function at the application boundary**, named in the context's language; it gets its outside world (store, clock, other systems) as parameters. | The use case runs in a test with no database, network or framework. |
| An **invariant** that spans several objects lives in one aggregate root, and one use case changes one aggregate. | Nothing outside the aggregate's module writes its parts; a second aggregate is changed by a later step or an event. |
| The tactical blocks are **vocabulary, not a mandate**. Name a block when the domain has it; do not add layers to a script or a CRUD screen. | A review asks "which invariant or which second adapter needs this?" before a new trait or layer lands. |

## A use case as a backlog item

Cockburn defines a use case as a contract between the stakeholders of a system about its behaviour: the primary actor
starts an interaction to reach a goal, and the use case collects the scenarios that can unfold [3] (moderate). He
writes it in four precision levels, breadth first: actor and goal; the main success scenario; the extension
conditions; the extension handling [3]. Fowler adds that the value is in the text, not in the UML diagram, and that
use cases work when they stay short [5] (moderate).

The levels line up with item sizes ([spec-driven-work.md](spec-driven-work.md)); the mapping is ours:

| Cockburn's part | In the item | Size |
| --- | --- | --- |
| Primary actor and goal | the title | `s` |
| Main success scenario, 3-9 steps, each an actor's intention, not a screen detail | a numbered list under the request | `m` |
| Extension conditions, numbered against the step they leave (`3a`) | `## Done when`, one "when X, Y" line each | `m` |
| Preconditions and guarantees | the first Done-when lines ("given ...", "after it, ... holds") | `l` |
| Out of scope, open points | `## Not in scope`, `## Open` | `l` |

The right level is the user goal: one person, one sitting, and the actor "can go away happy" after it [3]. A goal
above that level (a whole process) is several items; one below it (a sub-step such as "validate the code") is a step
or an extension, not an item.

```markdown
# Operator exports the month's report

Actor: operator. Trigger: the month is closed.

1. The operator asks for the report of a month.
2. The system collects the month's entries and totals them.
3. The system writes the file and shows where it is.

## Done when

- when the month has no entries, the system says so and writes no file
- when the month is not closed, the export is refused with the reason
- after an export, the file's totals equal the sum of its rows (property test)
```

**The test list.** The main success scenario is one test through the use-case function; each extension is one more,
named after its condition. That is the test list of [tdd.md](tdd.md), written before code. Cockburn's hexagonal
article says the same from the other side: the use cases belong at the application boundary, and an automated test is
one more thing that drives the application through that boundary [4] (moderate).

## The tactical blocks

Definitions are Evans's [1] (strong: pages 9-18 read); the "in code" column is this page's reading of them.

| Block | What it is | Take it when | In code |
| --- | --- | --- | --- |
| **Entity** | An object defined by identity that continues through a lifecycle while its attributes change; "the model must define what it means to be the same thing" [1]. | A thing is tracked over time and two with equal fields are still two. | A type with an id type of its own; a table with a primary key; fields by [entity-fields.md](https://github.com/evocaslab/hq/blob/main/practices/entity-fields.md). |
| **Value object** | An object where only the attributes matter: no identity, immutable, operations without side effects [1]; equal when its values are equal [6]. | A measure, a range, a code, a name, money, a parsed input. | A newtype, frozen dataclass or readonly type built by one parsing constructor ([parse-dont-validate.md](https://github.com/evocaslab/hq/blob/main/practices/parse-dont-validate.md)); columns, never its own table. |
| **Aggregate** | A cluster of entities and values with one root; outside code holds references to the root only, and the root enforces the invariants of the whole [1]. | A rule spans several objects that are saved together. | One module; the root's methods are the only way to change the parts; loaded and saved as a whole. |
| **Domain event** | A record of something that happened and that the domain cares about; immutable, with the time and the ids of the entities involved [1]. | A second part must react, or the cause of a change must be kept. | A past-tense struct or enum variant; an append-only table or log. |
| **Repository** | Access to aggregate roots that looks like an in-memory collection and hides storage; only for roots that need direct access [1]. | The use case must run without the database in tests. | A driven port: a small trait or protocol named by what the domain asks ("open invoices of a customer"). |
| **Domain service** | A process or transformation that is not a natural responsibility of an entity or value [1]. | A calculation over several aggregates. | A plain function in the domain module. |
| **Port and adapter** | A port is a purposeful conversation of the application with the outside; an adapter converts it to one technology [4]. | Always for I/O; a trait only when there is a second adapter (a test fake counts). | The port is owned by the application; adapters sit at the edge. |

### Aggregates: four rules that keep them small

Vernon's rules of thumb [2] (strong: the rules and their sections read in parts I and II; part III not opened):

1. **Model true invariants in consistency boundaries.** An aggregate holds exactly what must be consistent at the end
   of one transaction; a well-designed context changes one aggregate instance per transaction.
2. **Design small aggregates.** A large cluster loads more than the change needs and makes unrelated edits collide.
3. **Reference other aggregates by identity**: hold the other root's id, not the object.
4. **Use eventual consistency outside the boundary.** A rule across aggregates is met a little later, by an event or a
   job. The tie-breaker, which Vernon credits to Evans: if making the data consistent is the job of the user running
   the use case, do it in the transaction; if it is another user's or the system's job, let it follow.

Fowler states the same pair in two sentences: references from outside go to the root only, and transactions do not
cross aggregate boundaries [7] (moderate). An aggregate is a domain concept (an order with its lines), not a
collection class.

### Domain events are not event sourcing

Recording that something happened is cheap. Rebuilding state from the events is a different, far larger decision:
Fowler calls the event-centred style significant and awkward at first [8] (weak: a draft page). Take the event; leave
event sourcing and CQRS until a measured need.

### Ports and adapters

Cockburn's intent: an application that can be driven equally by users, programs, automated tests and batch scripts,
and developed and tested apart from its devices and databases [4]. A *driving* (primary) actor starts the
conversation: a route, a CLI command, a timer, a test. A *driven* (secondary) actor is one the application calls: a
database, a mail service, a clock; in tests a fake stands in for it. Evans's layered architecture has the same goal,
isolation of the domain, and names hexagonal architecture as an equal or better way to get it [1].

## Where each lives in code

The sources fix the direction of dependencies (the domain imports nothing from the edge), not folder names. One
arrangement that keeps the direction visible, for one bounded context:

```text
<context>/
  domain/     entities, value objects, aggregates, events, pure functions; no I/O, no framework imports
  app/        one function per use case; takes ports as parameters; opens the transaction
  ports       the traits or protocols app needs: repositories, clock, notifier
  adapters/   driving: http routes, cli, jobs; driven: sql, api clients, files
```

A small tool keeps this as three files or one; the test is the import direction, not the tree.

## Why this helps with agent workers

Reasoned from the sources and from briefs written here, not measured:

- **A brief can be exact.** "Use case X, aggregate Y, behind port Z" bounds the files a worker may touch and gives the
  reviewer the diff's expected shape.
- **The test list exists before the code**, so the hand-back is checked line by line instead of by impression.
- **An invariant has one home.** A worker that adds a second way to write a part skips the rule; "only the root
  writes" is a check a reviewer or a grep can make.
- **Use-case tests need no services**, so a worker runs them in seconds and the gate stays cheap
  ([verification-layers.md](verification-layers.md)).
- **Names come from the glossary**, so two workers do not invent two words for one thing
  ([glossary-and-vocabulary.md](glossary-and-vocabulary.md)).

## Studied and not taken

- Fully dressed use cases (stakeholder tables, a template per use case): Cockburn's own casual form is enough for a
  close-knit group [3]; reopen when a customer must sign the behaviour off.
- UML use-case diagrams: the text carries the value [5]; a flow is drawn only when it has branches worth a picture
  ([architecture-diagrams.md](architecture-diagrams.md)).
- Tactical DDD as a mandate, factories as their own objects, event sourcing, CQRS: overhead without a domain that asks
  for them (register: `docs/design/practices-system.md` in HQ); reopen when invariants span several entities in more
  than one place, or a read model is measured as the bottleneck.

## Checklist

- [ ] The item's title is an actor and a goal at the user-goal level.
- [ ] The main success scenario has 3 to 9 steps; every extension is a Done-when line and a test.
- [ ] The use-case function runs in a test without a database or network.
- [ ] Each invariant across objects has one aggregate root that enforces it; other aggregates are held by id.
- [ ] One use case changes one aggregate; anything else follows by an event or a job.
- [ ] Domain code imports nothing from adapters; names follow the context's glossary.

## Sources

Confidence: medium. [1]-[3] were read as PDF text by the writer; [4]-[8] were read once through a fetch summary and
not re-opened by a second reader.

1. Evans, "Domain-Driven Design Reference" (2015), part II "Building Blocks of a Model-Driven Design", pp. 9-18 read
   in full 2026-11-11: <https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf>
2. Vernon, "Effective Aggregate Design", part I "Modeling a Single Aggregate" and part II "Making Aggregates Work
   Together" (2011), the four rules and "Ask Whose Job It Is" read 2026-11-11:
   <https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_1.pdf>,
   <https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_2.pdf>. Part III not opened.
3. Cockburn, *Writing Effective Use Cases*, pre-publication draft 3 (2000; published by Addison-Wesley, 2001), the
   reminders, the writing process, chapter 1 and the user-goal section read 2026-11-11 from a course mirror:
   <https://kurzy.kpi.fei.tuke.sk/zsi/resources/CockburnBookDraft.pdf>. A draft on a third-party host, hence
   moderate; the printed book was not opened, and its "use cases to test cases" section is not in the draft's text.
4. Cockburn, "Hexagonal architecture" (2005), read 2026-11-11: <https://alistair.cockburn.us/hexagonal-architecture>
5. Fowler, "UseCase" (2004-08-03), read 2026-11-11: <https://martinfowler.com/bliki/UseCase.html>
6. Fowler, "ValueObject" (2016-11-14), read 2026-11-11: <https://martinfowler.com/bliki/ValueObject.html>
7. Fowler, "DDD_Aggregate" (2013-04-23), read 2026-11-11: <https://martinfowler.com/bliki/DDD_Aggregate.html>
8. Fowler, "Domain Event" (a draft of unfinished writing), read 2026-11-11:
   <https://martinfowler.com/eaaDev/DomainEvent.html>
9. Evans, *Domain-Driven Design* (2003) and Vernon, *Implementing Domain-Driven Design* (2013): books not opened;
   known through [1] and [2].
