---
summary: "Keep one glossary as data (yaml: term, icon, definition, avoid/say, see), generated into Markdown and JSON and shown in the app; one meaning per word, icons are decoration next to the word"
group: docs
tags: [docs, workflow, frontend, agents]
applies-to: [all]
reviewed: 2026-11-09
status: draft
---

# Glossary and vocabulary

HQ's sessions, skills, practices and screens must mean the same thing by "session", "item", "landing" or "gate". The
words live in one glossary that is data, not prose, so a tool can print it, lint against it and show it in the app.

## The rule

- **One glossary, as data.** Terms live in `practices/glossary/<scope>.yaml` (one file per area of vocabulary).
  `just practices-index` generates `practices/glossary.md` (read by people and agents) and `practices/glossary.json`
  (read by the HQ web app's Glossary page, `GET /hq/api/v1/glossary`, and the MCP). Never edit the generated files.
- **One meaning per word, one word per meaning.** A term has an `id`, the `term`, a one-sentence `definition` (what it
  is, where it is stored or run, the command that shows it), and optional `aliases`, `avoid` (`{not, say}` pairs for
  words that clash), `see` (term ids or repo paths), `external` (the same word in Claude Code or another tool), `note`.
  When a word means two things, qualify the second one in `avoid` instead of letting both live.
- **A new word enters with the artefact that defines it.** A role, status, mode, recipe family or registry field added
  to a script, skill or practice gets its term in the same commit; a term that no artefact uses is deleted.
- **Icons: one emoji per term, optional `icon:`** (also on the scope). One icon per meaning: two terms of one scope
  share an icon only when one lists the other in `see`; `just practices-check` fails otherwise. An icon is a short
  emoji string, never letters or digits.
- **An icon is decoration with the word beside it.** It is never the only cue: the term's text is always shown next to
  it, the icon is `aria-hidden` in the app, and nothing is explained by the icon alone.
- **Prefer the glossary's word** in briefs, commit messages, item files and UI copy. A repo with its own domain keeps a
  project glossary in the same shape, one file per bounded context in `docs/glossary/<context>.yaml`, with the key
  `context` where HQ's files have `scope` (a topic group), and links here: which term lives where, and the template, are in
  [architecture-map.md](architecture-map.md#glossaries-global-and-project)
  ([backlog](https://github.com/evocaslab/hq/blob/main/docs/backlog/B-261108-repo-domain-glossaries.md)).

## Why

- **Ubiquitous language.** Evans's term (Domain-Driven Design) is a common, rigorous language that developers and
  domain experts build together, grounded in the domain model; Fowler stresses that software handles ambiguity badly
  and that the language evolves as understanding grows ([1]). HQ's "experts" are the maintainer and the agents: both
  read the same sentence, so the same word must resolve to the same thing. Agents also copy whatever word they saw
  last, so a drifting word spreads.
- **Controlled vocabulary.** A glossary with preferred and avoided terms is the lightest form of one: the `avoid`
  table is the "use for / do not use" mapping of a thesaurus, and it can be linted later without changing the format.
- **Data, not prose.** A page of prose cannot be searched by id, shown as a tooltip, returned by a tool or checked for
  duplicates. The yaml schema gives all four from one source; the generated page keeps a plain reading path.
- **Why an icon at all.** A glyph beside a word gives the eye a second handle in a long list and makes groups
  recognizable (status columns, roles). Icons are often ambiguous and need a visible text label ([4]), so the word
  stays; a purely decorative icon should be ignorable by assistive technology and carry no alt text ([2]); and icon or
  colour must never be the only means of conveying information ([3]). That is why the schema allows one emoji, the page
  prints it before the word, and the check fails on a shared icon between unrelated terms (a repeated icon would
  suggest a relation that is not there).
- **Why emoji, not SVG.** An emoji is plain text: it renders in Markdown, terminals, JSON and the web without an asset
  pipeline. Rendering differs by platform, which is acceptable for decoration. A brand or diagram symbol that must look
  identical is a diagram's job ([architecture-diagrams.md](architecture-diagrams.md)), not a term's.

## Checklist

- [ ] A new role, status, mode or command family has a term in `practices/glossary/*.yaml`.
- [ ] Each term has a definition of one sentence and an `icon`; no two unrelated terms in a scope share one.
- [ ] `just practices-index` and `just practices-check` pass; the generated files are committed.
- [ ] UI text shows the term next to its icon; the icon is `aria-hidden`.

## Sources

1. Martin Fowler, "Ubiquitous Language" (bliki): <https://martinfowler.com/bliki/UbiquitousLanguage.html>, read 2026-11-09.
2. W3C, WCAG 2.2 Understanding SC 1.1.1 Non-text Content (decorative content ignorable by assistive technology):
   <https://www.w3.org/WAI/WCAG22/Understanding/non-text-content.html>, read 2026-11-09.
3. W3C, WCAG 2.2 Understanding SC 1.4.1 Use of Color (a non-colour cue such as shape or text is required):
   <https://www.w3.org/WAI/WCAG22/Understanding/use-of-color.html>, read 2026-11-09.
4. Nielsen Norman Group, "Icon Usability" (icons need an always-visible text label; most are ambiguous):
   <https://www.nngroup.com/articles/icon-usability/>, read 2026-11-09.
5. Eric Evans, *Domain-Driven Design* (2003), ubiquitous language: known from Fowler's summary [1], book not opened.
