---
summary: "Diagrams as text: which tool per diagram class, a gate that parses and renders every diagram, text alternatives, size limits"
group: docs
tags: [docs, diagrams]
applies-to: [all]
reviewed: 2026-11-07
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
| Data (ER), interactive | `hq-diagram` model generated from the migrated schema (`schema.diagram.yaml`: a box per table, `members:` rows, `from_card` / `to_card` at the line ends) | | same hover, click panel and fold as the information model; Mermaid stays the default on GitHub pages |
| Class diagrams | Mermaid `classDiagram` | the same `members:` boxes in `hq-diagram` when the model is generated from code | no operations compartment, inheritance or visibility yet |
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

## A data model drawn by a small renderer

The page, wasm and accessibility side of such a renderer: [wasm-self-contained-pages.md](wasm-self-contained-pages.md).

Lessons from `hq-diagram` (Rust, YAML model to SVG, HTML and A3 PDF) on the report-bot information model
(2026-11-06). Use it when the picture is data (who sends what, where it lands) and a general tool's auto-layout would
reshuffle it on every change; run it with `just docs-diagram`.

- **The model is the source, the picture a build product.** Divisions, nodes and edges live in YAML read through a
  strict parser (unknown fields refused); a node must name its `sources:` or say `unknown: true`, and the validator
  rejects anything else. The gap shows on the picture as a dotted unknown source instead of a guess.
- **Pin the layout, do not search for one.** Fixed columns (sources, collectors, outputs), one bus per source (stub,
  spine, branches) and one lane per flow keep the output the same for the same input; a golden SVG test shows every layout change in the diff.
- **Measure text, do not estimate it.** A flat "7 px per character" overflowed boxes with capitals and Cyrillic. Per-glyph
  advances (integer maths, bold wider) and wrapping by pixel width, plus a test that no line is wider than its box, fixed it.
- **Colour is never the only cue.** Give each division its own marker shape (circle, square, diamond, triangles, plus,
  cross, hexagon), the same one on the bus and in the key, and mark the unknown source dotted. Test it: distinct shapes,
  and colours that collapse under simulated deuteranopia (Machado matrix) or greyscale may do so only if the shapes differ.
  Use a colour-blind-safe palette (Okabe-Ito) as the base.
- **Look at it at both sizes.** Render, then screenshot the phone width (375 px) and the print size. The phone view
  needed a zoom toolbar and a fit-to-width button; the A3 PDF needed the whole picture on one page.
- **Group, label and tag in the model, not on the canvas.** Containers (`groups:` and `group:` on a node), icons and
  per-document parts with a format (Word, Excel, PDF, text, text and pictures) are model fields; the renderer keeps
  the members of a group together and draws the frame, so reordering the file never leaves a frame around strangers.
- **Hover to trace, click to read.** In the interactive page, dim everything not connected to the hovered or focused
  box, show a short tooltip, and put the long text, examples and pictures in the click panel (keyboard and deep link
  too). Every box and container needs a keyboard path and a native `<title>` for viewers without script.
- **Fold, filter, pin.** A big model needs controls in the interactive page: fold a container, filter by source and
  name, pin a box to light its whole chain; keep motion optional (`prefers-reduced-motion`). Attachments are model
  fields (`file` embedded, `url` linked, a size limit) under the same export flag.
- **Let the page edit, keep the source.** An edit mode in the page cannot write files, so it downloads a sidecar
  (`<model>.edits.json`) that the renderer applies on top of the YAML: comments and review diffs of the model stay
  clean, and an edit naming a removed id fails loudly. Tag parts (`tags:`) so one flag (`--hide bot`) leaves a whole
  audience out of the picture.
- **A local server closes the loop.** For editing in the page, a loopback-only server that re-renders on every load and
  writes the YAML and the sidecar beats download-and-copy: put a random token on every request, check the `Host`
  header (DNS rebinding), write only the two files, atomically, and refuse YAML that does not load and validate, so the
  file on disk always renders. Show the error plus the text to fix when the files were broken from outside. For visual
  structure edits (add, delete, connect, move) turn each click into a small line patch of the YAML text, not a re-dump of
  the parsed model: comments and order survive, the diff is one line, and the same check refuses a patch that breaks it.
- **Make "what leaves the building" a flag.** Examples and attachments in an exported page can be sensitive: give each
  document and part an `export:` flag and the CLI a `--documents all|marked|none` switch, and test that withheld text
  and pictures are absent from the output, not just hidden.
- **A gate hook can eat fixtures.** A formatter or SVG minifier that rewrites `tests/fixtures/` breaks a byte-exact
  golden test: exclude fixture folders from such hooks.
- **Cut crossings by counting them.** Pinned columns still cross lines. Write a function that counts crossings on the
  finished layout, then order boxes and lanes with a few barycentre sweeps (containers move as one block). Lay out each
  candidate in full and keep the file order unless a candidate crosses strictly less, so a model without crossings keeps
  its order and ties never reshuffle it. Integers only, no randomness: the result is the same in the wasm build and in
  the golden test. Test a ceiling ("at most N on this fixture") with the old count written in the test. Search moves that
  gave no gain and cost time (insertion moves, every corridor mode per move) were not worth keeping.
- **A page that translates itself changes text widths.** If the page swaps labels to another language in the browser,
  a chip or pill sized on the first language overflows. Size drawn shapes for the wider of the two texts, and check the
  other language with a screenshot.
- **One relayout engine for the page and the CLI.** Embed the same layout code as wasm so folding and dragging redraw
  without a server; rebuild the wasm asset once after a batch of layout changes (a stale asset silently draws the old
  layout), and make a test that the page starts in the orientation and warnings it was rendered with.
- **Keyboard and contrast are features of the canvas.** Make boxes and lines focusable, give every mouse edit a key
  (edit, nudge, delete with confirm), keep a live region for announcements, and offer a high-contrast theme.
- **Parallel agents on one crate:** give each its own worktree and file area, have them commit without pushing, and
  cherry-pick them one by one; conflicts concentrate in the shared page file and the string table, so put each feature
  in a marked block (`/* a11y begin */ ... end */`) and regenerate the golden file and the wasm only once at the end.

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
- Observed in `crates/hq-diagram` and `auto-joint-lab/report-bot`'s information model, 2026-11-06 and 2026-11-07
  (crossings 170 to 75 on the fixture, parallel agent round).
- Ordering: Sugiyama, Tagawa and Toda, [Methods for visual understanding of hierarchical system structures](https://doi.org/10.1109/TSMC.1981.4308636) (1981), the barycentre method for crossing reduction.
- Colour vision: Machado, Oliveira and Fernandes, [A physiologically-based model for simulation of color vision deficiency](https://www.inf.ufrgs.br/~oliveira/pubs_files/CVD_Simulation/CVD_Simulation.html) (2009);
  Okabe and Ito, [Color Universal Design](https://jfly.uni-koeln.de/color/); WCAG 2.2
  [1.4.1 Use of Color](https://www.w3.org/WAI/WCAG22/Understanding/use-of-color.html).
