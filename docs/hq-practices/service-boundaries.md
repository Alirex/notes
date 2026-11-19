---
summary: "Where one part of a system ends: a bounded context is a language boundary, modular monolith before any split, the distributed-monolith checks, a context map template and a boundary checklist"
group: deploy
tags: [architecture, ddd]
applies-to: [all]
reviewed: 2026-11-09
status: draft
check: checklist
---

# Service boundaries

Where one part of a system ends and the next begins, and when a part deserves its own process, repo or deploy. The
subject is strategic Domain-Driven Design read for one maintainer with a few repos and agent workers: the useful
part is a way of seeing boundaries, not a team process. The research with every source and evidence level is in
[docs/research/2026-11-09-ddd-and-related.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-09-ddd-and-related.md); the terms are in
the glossary scope `architecture` ([glossary.md](https://github.com/evocaslab/hq/blob/main/practices/glossary.md)).

## The rule

| Rule | Check |
| --- | --- |
| A boundary is a **language boundary**: inside it one word has one meaning and one model; where the same word means two things, there are two contexts. A database schema, a repo or a deployable is not a boundary by itself. | The glossary has one file per context (project glossary) or one scope per topic (HQ's); a term with two meanings has an `avoid` entry; code names follow the glossary. |
| **Modular monolith first.** A new part is a module (crate, package, script folder) with an explicit API inside the existing process. It becomes its own process, repo or deploy only when a reason in "When to split" holds. | A proposal to split names the reason from the table and the checks it passed. |
| A module hides **one design decision** that is hard or likely to change (Parnas). Modules are cut by what changes together, not by processing step or technical layer. | "What does this module hide?" has a one-line answer; the module's API does not expose it. |
| Every **shared store** (a table, a file format, a JSON shape) that two contexts read or write has **one owner**, a **published language** (a schema file or one parser) and a test that the other side still reads it. | No DDL or parser of a shared shape exists twice; the copying side imports or calls the owner's. |
| The repo set has a **context map**: one page naming each context, its language, its stores and its relations, with the smells found. It is reviewed against the code, dated, and updated in the commit that adds a context, a store or a relation. | `docs/architecture/context-map.md` carries "Reviewed against the code on `<date>`". |
| **Anticorruption layer** only at external systems (another vendor's API, a tool's transcript format, a CLI). Inside the repo set the contexts talk through published languages, not translation layers. | External formats are parsed in one adapter module; the domain types never carry the external field names. |

## Bounded context = language boundary

Evans defines a bounded context as the boundary within which a particular model is defined and applicable, typically
a subsystem or the work of a team, and the ubiquitous language as the shared vocabulary *within* that context [1].
Fowler's test is the simplest: when the same word means different things in two areas, you have two contexts, and
language and culture are the main cue, not code shape [2]. Brandolini found the same at the wall: context boundaries
appear where people disagree on what a term means [9].

For a solo maintainer this is the whole test, since the team half is moot. Applied here: "session" is a Claude Code
conversation in the fleet context and a browser sign-in in the web-app context; the glossary already names the
second one "sign-in". A context's language lives in one glossary file (a project's `docs/glossary/<context>.yaml`, key `context`); the code's type, table and file names follow
it. A term that crosses a boundary is translated at the edge, in a named adapter, not carried along.

## Distributed monolith: the checks

A distributed monolith is a system split into deployables without a split of models: the parts must still move
together. The symptoms, from the secondary literature (Newman's own wording was not found) and the Azure boundary
guidance [10], [11]:

| Smell | Check in the code | Why it hurts |
| --- | --- | --- |
| **Lockstep release**: a change in one part needs a release of another to work | the last 20 landings: how many touched two deployables for one behaviour | the split bought cost and no independence |
| **Shared database tables**: two parts read or write one table | every `CREATE TABLE`, migration and parser of a shared shape: one owner, one file | a schema change breaks the other side silently; the table is the hidden API |
| **Copied rules**: the same business rule (a ranking, a validity check) implemented twice | grep the rule's constants in every language | the copies drift; Khononov's functional coupling at full strength [12] |
| **Chatty calls**: one user action needs several synchronous round trips between parts | trace one request; count the hops and the parts that must be up | latency and a failure mode per hop; the parts are one process in denial |
| **Shared business library**: a library with domain logic that every part must upgrade together | a change in the library forces releases of its consumers | the library is a shared kernel without the agreement [1] |
| **Intrusive coupling**: a part imports another's internals (a module loaded by file path, a private table) | by-path imports and direct store access across folders or crates | the strongest integration strength [12]: any internal change is a breaking change |

Inside one process the same smells exist and cost less; they still mark a boundary in the wrong place. The fix is
the same at any distance: one owner, a published language, a test on the consumer's side.

## When not to split: modular monolith first

Fowler: most successful microservice systems started as monoliths and were split once the boundaries were known;
building a new system as services often goes badly, because early on feedback speed matters more than service
independence [3]. The "premium" (deploy automation, monitoring, eventual consistency, failure handling) pays only when
complexity, team size, independent scaling or independent evolution demand it [4], [5], and only with the operational
prerequisites in place: fast provisioning, monitoring of technical and business metrics, automated deploy with
rollback [6].

A modular monolith is one deployment unit whose code is split into modules with encapsulation, an explicit API and
few, weak dependencies, cut by business feature [8]. It is the default here: one process per app, one database per
app, modules as crates or script folders, the context map as the record of their borders.

Reasons that justify a split, each with the check that it is real:

| Reason | Real when |
| --- | --- |
| Independent release cadence | the part's changes wait on another part's gate or release more than once a month |
| A different runtime or host | the part must run where the rest cannot (another machine, the GPU, a browser) |
| Failure isolation | a crash or an overload of the part must not take the rest down, and it has |
| Independent scaling | the part's load profile differs by an order of magnitude and is measured |
| A different owner | a second person or team owns the part end to end (Conway: the design follows who talks to whom [7]) |

Without one of these, keep the module in the process and fix the boundary instead.

## Context map template

Evans: identify each model in play, name its context, make the names part of the language, describe the points of
contact with explicit translation, sharing, isolation and influence; map the existing terrain first and take up
transformations later [1]. One page, hand-written, dated against the code:

```markdown
# <system> context map

Reviewed against the code on <date>.

## Contexts
| Context | Language (glossary file, the words it owns) | Code | Stores it owns |

## Relations
| From | To | Kind | Through | Smell |
(Kind: shared kernel, conformist, open-host service, published language,
 anticorruption layer, separate ways; upstream is the side whose change breaks the other.)

## External systems
| System | Adapter (the anticorruption layer) | What is translated |

## Core, supporting, generic
(where the effort goes; generic parts are candidates for a library)

## Boundary smells (file:line) and the items that fix them
```

The relationship names are Evans' [1], used for the shape of the code-level coupling, not for team negotiation:
*shared kernel* is code or a schema two contexts own together (keep it small, change it with a test on both sides);
*conformist* is a downstream that uses the upstream's shape as is; *open-host service* is one protocol for all
consumers (an MCP server, an HTTP API, a CLI with `--json`); *published language* is the documented shape they
exchange; *anticorruption layer* is the adapter that keeps a foreign model out; *separate ways* is no connection.

Distillation decides where the effort goes [1]: the **core** is what the system exists for and gets the best design;
**supporting** parts serve it; **generic** parts (auth, mail, diagrams) are candidates for an off-the-shelf library
and get the least custom code.

## Checklist

Before a new crate, package, service or repo, and in a review of an existing one:

1. **One language.** Does every word in the module's API mean one thing, the glossary's thing? Does any word mean
   something else in a neighbour?
2. **One decision hidden.** What design decision does this module hide from the others? Could it change without a
   change on the other side?
3. **No shared tables.** Does any other part read or write this module's store directly? Then who owns the schema,
   and where is the one parser?
4. **No copied rule.** Is any rule here also implemented elsewhere, in another language? Then which copy is the
   owner, and what test keeps the others equal?
5. **Change together, live together.** Of the last ten changes to this module, how many also changed one specific
   neighbour? A high count says the boundary runs through a concept.
6. **Chatty?** For one user action, how many hops cross this boundary, and must every hop be up?
7. **Coupling versus distance.** Strong coupling (shared internals, shared data) belongs at small distance (same
   crate); large distance (another process, another repo) needs weak coupling (a contract) [12]. Which pair is this?
8. **Split reason.** If the proposal is a new process or repo: which row of "When to split" holds, with its evidence?
9. **Map updated.** Is the context map changed in the same commit, with its reviewed date?

## Why

**Language, not schema, because a model is what a boundary protects.** Evans' definitions tie both the bounded
context and the ubiquitous language to a model [1]; a shared schema with two meanings for one column is exactly the
blur the context map exists to show [1], [2].

**Modular monolith first, because boundary mistakes are cheaper inside one process.** The split is a bet on the
boundary being right; Fowler's MonolithFirst and premium notes say the bet is usually lost when placed early [3], [4],
and the operational prerequisites alone cost more than a solo maintainer's time allows [6].

**Parnas, because the flowchart is the default mistake.** Decomposing by processing step is the most common approach
and "almost always incorrect" past small systems; each module should hide one difficult or likely-to-change decision
[13]. Script folders cut by phase (collect, compute, report) are that flowchart.

**One owner per shared shape, because a shared table is the hidden API.** Fowler names the shared integration
database as the major coupling source and asks that each service's data be reachable only through its API [5]; Evans'
shared kernel allows sharing but only explicit, small and with an integration process [1].

**Coupling against distance, because both cannot be high.** Khononov's rule: high integration strength plus high
distance means cascading change; keep strong coupling close or make it weak before moving it far [12].

**Conway, because one maintainer is still an organisation.** The design copies the communication structure [7];
with one person and many agent sessions, the structure is the briefs and the shared files, and the map makes it
visible.

## Checks

`check: checklist`: a reviewer walks the checklist above for a new crate, package or repo, and the context map's
reviewed date is updated in the same commit. A hook for the "no DDL twice" and "no by-path import across folders"
rules is open work (the `B-261109-*` boundary items).

## Sources

1. Evans, "Domain-Driven Design Reference" (2015), definitions and part IV-V (context mapping, distillation), read
   2026-11-09: <https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf>
2. Fowler, "BoundedContext" (2014), read 2026-11-09: <https://martinfowler.com/bliki/BoundedContext.html>
3. Fowler, "MonolithFirst" (2015), read 2026-11-09: <https://martinfowler.com/bliki/MonolithFirst.html>
4. Fowler, "MicroservicePremium" (2015), read 2026-11-09: <https://martinfowler.com/bliki/MicroservicePremium.html>
5. Fowler, "Microservice Trade-Offs" (2015), read 2026-11-09: <https://martinfowler.com/articles/microservice-trade-offs.html>
6. Fowler, "MicroservicePrerequisites" (2014), read 2026-11-09: <https://martinfowler.com/bliki/MicroservicePrerequisites.html>
7. Conway, "How Do Committees Invent?" (1968), read 2026-11-09: <http://www.melconway.com/Home/Committees_Paper.html>
8. Grzybek, "Modular Monolith: A Primer" (blog, weak), read 2026-11-09:
   <https://www.kamilgrzybek.com/blog/posts/modular-monolith-primer>
9. Brandolini, "Introducing Event Storming" (2013), read 2026-11-09:
   <https://ziobrando.blogspot.com/2013/11/introducing-event-storming.html>
10. Nolle, "The distributed monolith: What it is and how to escape it" (TechTarget, weak), read 2026-11-09:
    <https://www.techtarget.com/searchapparchitecture/tip/The-distributed-monolith-What-it-is-and-how-to-escape-it>
11. Microsoft, "Identify microservice boundaries", read 2026-11-09:
    <https://learn.microsoft.com/en-us/azure/architecture/microservices/model/microservice-boundaries>
12. InfoQ podcast with Vlad Khononov, "Balancing Coupling in Software Design" (transcript, weak), read 2026-11-09:
    <https://www.infoq.com/podcasts/balancing-coupling-software-design>
13. Parnas, "On the Criteria To Be Used in Decomposing Systems into Modules" (CACM 1972), read 2026-11-09:
    <https://wstomv.win.tue.nl/edu/2ip30/references/criteria_for_modularization.pdf>
14. Research note with the not-taken register: [docs/research/2026-11-09-ddd-and-related.md](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-09-ddd-and-related.md)
