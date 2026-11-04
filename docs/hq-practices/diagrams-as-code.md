---
summary: "Diagrams as text: which tool per diagram class, a gate that parses and renders every diagram, text alternatives, size limits"
group: docs
tags: [docs, diagrams]
applies-to: [all]
reviewed: 2026-11-04
status: stable
---

# Diagrams as code: tools, validation, process

[architecture-diagrams.md](architecture-diagrams.md) says *which* views a repo draws. This page says *with what* and *how
they are kept honest*: one default tool, a recommendation per diagram class, and a gate that renders every diagram.
Mermaid stays the default (GitHub draws it, agents write it, diffs are text); the work is in rules and checks around it,
and in a few classes where another tool is better.

## Rules

| Rule | Why |
| --- | --- |
| Diagram source is **text next to the code it describes**, in the Markdown page that needs it; no pictures without a source | a diagram nobody can regenerate or diff rots first |
| **Every diagram is parsed and rendered by the real renderer in the gate** (HQ: `mermaid-parses` hook, `just docs-mermaid`) | Mermaid errors show only when rendered: GitHub draws a red box, the commit looked fine |
| Every diagram has a **text alternative**: `accTitle` and `accDescr` (Mermaid), or a title and long description beside an image; the gate fails without them | WCAG: a complex image needs a short name plus the essential content as text; it also helps agents that read the source |
| **One fact per label, quoted**: `["text"]`, edge labels `-->\|"text"\|`; no semicolon or `%%` in sequence messages (write the entity `#59;` if needed); no `end` in lower case as an id; ids `[A-Za-z0-9_]` | the common causes of "syntax error" (HQ's sign-in flow had a `;` in a message) |
| No raw HTML in labels except `<br/>`; no `<small>`, `<b>`, `<span>` | the HTML label mode produces SVG that is not valid XML: exported or checked-in SVG shows boxes without text, PDF and other renderers print the tags |
| **Size limit** about 15 nodes and 25 edges per diagram (the gate warns at 30 and 45); split by boundary or move to a generated page | past that the auto-layout produces long crossing edges and nobody reads it |
| **Render a committed SVG only** where the surface cannot draw the source (the web app, D2, PlantUML, Structurizr; a PDF export draws Mermaid itself: HQ's `just docs-pdf`); keep the source beside it and an is-current check in the gate | GitHub draws Mermaid itself; a committed render of it goes stale and diffs as noise |
| A diagram change is reviewed as **text diff plus a rendered look** (`just docs-mermaid render --out DIR`, open the SVG) | the diff shows what moved, only the picture shows whether it is readable |
| Generated views are written by a script and checked for being current ([architecture-diagrams.md](architecture-diagrams.md#generated-views)) | a generated diagram cannot drift |
| Pin the renderer: exact version, lock file with integrity hashes, installed with scripts off | Mermaid 12 changed the default layout (ELK), theme and look; an unpinned tool re-lays-out every diagram |

## Which tool for which diagram

| Class | Default | Use instead when | Notes |
| --- | --- | --- | --- |
| Context, containers | Mermaid `flowchart` with `subgraph` boundaries | one model must feed many views (more than about 4 views): Structurizr DSL or LikeC4 with exports; layout too tight: D2 with ELK, SVG committed | Mermaid `C4Context` has no auto-layout, `architecture-beta` is beta |
| Flows | Mermaid `sequenceDiagram` | long or nested flows: PlantUML | the strongest Mermaid type |
| State machines | Mermaid `stateDiagram-v2`, pinned by a test | | |
| Data (ER) | Mermaid `erDiagram`, generated from the migrated schema | schema with more than about 25 tables: tbls or Graphviz, SVG committed | |
| Deployment, infrastructure | Mermaid `flowchart`, or generated from the manifest | several environments from one model: Structurizr deployment views | |
| Generated graphs (crates, packages, compose) | generate Mermaid, or Graphviz DOT when bigger than the limit | | `cargo depgraph`, `dot -Tsvg` |
| Illustrations, posters, one-offs | hand-written SVG, or `.drawio.svg` / Excalidraw SVG with the embedded source | print quality: Typst with CeTZ | an image with alt text and long description; agents write these badly |

## Tools compared

Facts as of 2026-10-31 (release dates from the registries and GitHub APIs); "agent" is a judgement, no benchmark exists.

| Tool | Best for | Layout | Output | GitHub draws it | Headless | Licence | State | Agent writes it |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Mermaid 12.2 | everything in Markdown | dagre, ELK (bundled and the default in 12) | SVG, PNG via browser | yes, an older version | `mermaid.parse` needs a DOM; `mmdc` needs Chromium | MIT | very active; 12.0 was breaking | very well |
| D2 0.9 | nicer boxes and arrows, themes, sketch look | dagre, ELK, TALA (paid) | SVG, PNG, PDF | no | one Go binary | MPL-2.0 | active, still 0.x | fairly well, small syntax |
| Structurizr DSL | one C4 model, many views | its own renderer, exports | PlantUML, Mermaid, SVG, a site | no | JVM; the `cli` and `lite` repos are archived, development moved to `structurizr/structurizr` | Apache-2.0 | active (v2026.10.26) | fairly well |
| PlantUML 1.2026.9 | sequence, state, deployment; C4-PlantUML | Graphviz (sequence is built in) | SVG, PNG | no | Java, Graphviz | LGPL-3.0 | active | very well (large corpus), plain layout |
| Graphviz DOT 16 | generated graphs, dependency graphs | dot, neato, fdp | SVG, PNG, PDF | no | apt package | EPL | active | well, weak on nesting and polish |
| LikeC4 1.60 | C4-like model with dynamic views, a browsable site | Graphviz WASM | PNG, site, exports to Mermaid, D2, DOT | no | npm; `likec4 validate` | MIT | very active, young | little data |
| Kroki 0.34 | one HTTP front for about 30 languages | per engine | per engine | no | self-hosted container | MIT | active | not a language; never send diagrams to the public server |
| draw.io, Excalidraw | hand-drawn illustrations | manual | `.drawio.svg`, `.excalidraw.svg` with embedded source | SVG as an image | desktop app only | Apache-2.0, MIT | active | badly, noisy XML or JSON |
| Typst + CeTZ, plain SVG | print-quality figures | manual | SVG, PDF | SVG as an image | one binary | Apache-2.0 | active | coordinates by hand |

**Verdict**: no tool replaces Mermaid for the common classes; D2 and Structurizr/LikeC4 pay off only when container
views outgrow Mermaid's layout or one model feeds many views, and then the SVG is committed and checked. HQ has no such
diagram today (its largest container view renders readably), so none was converted; the first repo that needs it adds
the format and its render check to the gate in the same way as `mermaid_check.py`. D2's TALA layout is closed and paid:
use dagre or ELK only.

## Mermaid, the pain points and what we do

| Pain | Do |
| --- | --- |
| Layout is automatic: long labels, crossing edges | short labels (detail goes in the text under the diagram), `subgraph` per boundary, `TB` for tall stacks, split at the size limit; look at the render once |
| Syntax breaks on special characters | quote every label, escape `"` as `#quot;`, see the rules above; the gate catches the rest |
| Version differences: 12.0 made ELK the default layout and a new theme and look | the gate pins one version; set `config: layout:` in the front matter only once GitHub's version is known (a manual action is open for it: GitHub's docs say a block containing `info` prints it); GitLab was still on 11.x, so keep to features both draw |
| Dark mode and theme | no theme or `style` in the source; roles are `classDef` colours that repeat the shape ([architecture-diagrams.md](architecture-diagrams.md#colour)) |
| Diffs of diagrams | one statement per line, stable ids, no reordering; render both sides when the change is a layout change |
| Not rendered in the HQ web app or `docs-pdf` (the fence shows as code) | accept for docs read on GitHub; for a PDF render the SVG with `just docs-mermaid render` and link the image, or add a Mermaid render to the viewer when a page needs it |
| Large graphs | Mermaid's limits are `maxTextSize` 50 000 characters and `maxEdges` 500, far above readable; the practical limit is the size rule |
| Accessibility | `accTitle` and `accDescr` on every block, plus the relationships as a sentence or table beside it when the diagram is the only place they are written |

## The validation hook

HQ: `scripts/docs/diagrams/mermaid_check.py`, `just docs-mermaid` (`check`, `render --out DIR [--theme dark]`), hook
`mermaid-parses`.

- Extracts every ` ```mermaid ` block of the tracked Markdown files, parses and renders them with `mermaid.parse` and
  `mermaid.render` in headless Chrome (the browser `docs-pdf` already needs), behind a throwaway localhost server
  because Chrome's `--dump-dom` does not wait for module scripts.
- Mermaid is `mermaid/package.json` (exact version) plus `package-lock.json` (integrity hashes), installed with
  `npm ci --ignore-scripts` into `~/.cache/hq/mermaid-check/<version>-<lock hash>/`; nothing in the repo gets a
  `node_modules`.
- Results are cached per block text, version, theme and page revision: a warm run is about 0.1 s, a cold run of 15
  diagrams about 5 s. The hook runs when a Markdown file or the checker changes.
- Fails on a parse or render error (with file and line), on a missing `accTitle` or `accDescr`; warns above 30 nodes or 45
  edges. `render` turns HTML labels off so the SVG is standalone.
- Tests: `test_mermaid_check.py` (hook `mermaid-check-tests`): block extraction, cache keys, exact pin and lock integrity,
  the report, the alt-text rule.
- Without Chrome or npm: `npx @mermaid-lint/cli` parses with jsdom and no browser, but it pins Mermaid 11.16.1, is a young
  0.x project and does not render; `@mermaid-js/parser` does not cover flowchart, sequence, state, class or ER. A repo
  that lacks Chrome may use it as a fallback and says so in its AGENTS.md.
- Other formats follow the same shape: a pinned binary or container, a `check` over the tracked sources, a cache, and an
  is-current check for the committed SVG (`d2 --check`-style formatting is not enough: render and compare).

## Diagrams in a sender-facing spec

Lessons from a spec with a flow diagram, a record-anatomy diagram and a reader flowchart (2026-11-04):

- **One palette for every diagram**, as `classDef` names with a fixed meaning: input blue, check violet, ok green,
  warn amber, bad red, note grey dashed. A reader learns the colours once.
- **Notes are part of the picture.** A grey dashed note node, joined by a dotted edge to the step it explains, holds the
  rule that would otherwise clutter the labels (the duplicate rule, strict vs lenient mode).
- **Add an anatomy diagram for a record format**: fields grouped by meaning, required fields starred, one note per trap.
  Check every note against the field table (a note once said the bot computes a value the sender writes).
- **Render and look, not only parse.** `mermaid-check` proves the syntax; open the rendered image once to catch
  overlapping edges and clipped labels. For a PDF, export from the checkout that holds the latest text and look at
  the pages; a PDF made from another checkout shows the old version.

## Sources

- Mermaid: [releases](https://github.com/mermaid-js/mermaid/releases) (12.0.0 breaking: ELK bundled and default, new
  theme and look, Node 22.12+), [layouts](https://mermaid.js.org/config/layouts.html),
  [usage and `parse`](https://mermaid.js.org/config/usage.html),
  [accessibility](https://mermaid.js.org/config/accessibility.html),
  [flowchart syntax](https://mermaid.js.org/syntax/flowchart.html),
  [config schema](https://mermaid.js.org/config/schema-docs/config.html) (`maxTextSize`, `maxEdges`),
  [C4](https://mermaid.js.org/syntax/c4.html), [mermaid-cli](https://github.com/mermaid-js/mermaid-cli),
  [@mermaid-js/parser grammars](https://github.com/mermaid-js/mermaid/tree/develop/packages/parser/src/language).
- GitHub, [Creating diagrams](https://docs.github.com/en/get-started/writing-on-github/working-with-advanced-formatting/creating-diagrams)
  (the `info` block); community requests for newer Mermaid and ELK:
  [138426](https://github.com/orgs/community/discussions/138426),
  [198799](https://github.com/orgs/community/discussions/198799).
- Validators: [@mermaid-lint/cli](https://github.com/jasonworden/mermaid-lint),
  [maid](https://github.com/probelabs/maid).
- Tools: [D2 layouts](https://d2lang.com/tour/layouts/), [D2 releases](https://github.com/terrastruct/d2/releases),
  [TALA](https://github.com/terrastruct/TALA); [Structurizr export](https://docs.structurizr.com/export),
  [structurizr/structurizr](https://github.com/structurizr/structurizr); [PlantUML](https://plantuml.com/faq-install),
  [C4-PlantUML](https://github.com/plantuml-stdlib/C4-PlantUML); [Graphviz](https://graphviz.org/);
  [LikeC4 CLI](https://likec4.dev/tooling/cli/); [Kroki](https://kroki.io/); [draw.io](https://github.com/jgraph/drawio);
  [Excalidraw export](https://docs.excalidraw.com/docs/@excalidraw/excalidraw/api/utils/export).
- C4 model, [notation](https://c4model.com/diagrams/notation) and [checklist](https://c4model.com/diagrams/checklist):
  title and key on every diagram, labelled one-way relationships, a diagram that stands alone.
- W3C WAI, [Complex images](https://www.w3.org/WAI/tutorials/images/complex/): a short alternative plus a long
  description in the page; WCAG 2.2 [1.1.1 Non-text Content](https://www.w3.org/WAI/WCAG22/Understanding/non-text-content.html).
- Write the Docs, [Docs as code](https://www.writethedocs.org/guide/docs-as-code/).
- Not primary: committing SVG next to its source is common docs-as-code advice without a standard; the rule above is
  this page's judgement.
- Observed in `auto-joint-lab/report-bot`, 2026-11-04: three Mermaid diagrams with notes in the strike spec, rendered with
  `mermaid_check.py render`, checked as images and exported with `just docs-pdf`.
