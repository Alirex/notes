---
summary: "Write the spec of an input format or feature before the code: status banner, principles, one sample, what the parser gets, a field table, open questions; the sample runs as a test"
group: docs
tags: [docs, process, testing, api]
applies-to: [all]
reviewed: 2026-11-04
status: draft
---

# Specifications for formats and features

A spec is the page people agree on before the parser, form and manual change together. It is cheap to change, so
disagreements surface there and not in code. Example: JAS Bot's strike format
(`docs/manuals/corps-daily/strikes-next.md` in `auto-joint-lab/report-bot`).

## The shape of a spec page

| Part | What it holds |
| --- | --- |
| Status banner | `draft for agreement` / `implemented` / `replaced`, the date, what the code reads today, the backlog item. A spec nobody can tell from the code is worse than none |
| Principles | The few rules that explain every field (one label, multi-value lists, separators, what a missing field means), each with the decision and its date |
| One sample | A complete, realistic input with invented data; every field appears once |
| What we get | The same sample as the parsed result (YAML or JSON), with the planned field names, warnings and checks that follow from it |
| Field table | One row per field: label, type, required in the message, required in storage, rule, example values, aliases. Types come from a short legend written under the table (number, text, enum from a catalog, list); an empty value equals a missing field |
| Open questions | Each with a default, an owner and a date; closed ones move into Principles |

Rules of wording:

- MUST / SHOULD / MAY only where the code enforces or warns; they carry the meaning of RFC 2119, not decoration.
- Strict first ([strict-schemas.md](https://github.com/evocaslab/hq/blob/main/practices/strict-schemas.md)): a field starts required or typed, and loosens when real input
  proves it must.
- Warn, do not reject, when the data is still useful (a sum that does not add up, an unknown catalog value).
- Invented data only in examples; never copy a real record into a spec.

## Decisions that repeat in every input format

- **Required in the message vs required in storage.** Two columns, not one: a field can be absent from the text and
  still required in the record, because the system fills it from the context (chat binding, file name, report date).
  The check "missing" then fires only when neither the text nor the context gives it.
- **Error or warning, per mode.** A checks table with one row per check and two columns: strict (new data from chats
  and channels: the record is not saved and the sender is told what to fix) and lenient (old data and an opt-in import
  mode: the same finding is saved as a warning). Write the sender-facing reply for each common error next to it.
- **One word per concept.** Pick the unit of data (here: «запис») and keep an overloaded everyday word (strike, hit,
  event) for the real-world thing; a glossary line and a test or lint that the page does not drift back.
- **Canonical label plus aliases.** One canonical label per field, a short alias list (the stored name, one synonym);
  aliases must not collide, and writing the canonical label and an alias together is its own error. Test every alias
  once and the collisions.
- **Old input as one more input kind.** Do not make people rewrite: a line-by-line table "old line → field", each row
  marked mechanical or heuristic, a worked example, and the rule "unrecognised text goes to notes with a warning, never
  guessed". It reuses the extension point of the format (an extra reader that produces canonical fields, then the same
  checks) instead of a second pipeline.
- **A value list that may be wrong is closed and tested** (e.g. the allowed results): error in strict mode, warning in
  lenient mode, the word kept as written.

## Make people able to read it

The spec has two audiences, the author of the parser and the sender. Give the sender a front door: a "Briefly" part
with a flow diagram of what happens to a record (saved / saved with a warning / rejected), an empty copy-paste template,
the sample explained group by group, and the bot's exact replies for the common errors. Mark outcomes with fixed
symbols (⛔ error, ⚠️ warning, ✅ ok) and show sums as a small text bar chart. Keep these parts before the reference
tables, and keep the test anchors (headings) stable when you add them.

## Templates people fill in

Senders are often not engineers. A template with hints in it gets filled wrong in predictable ways; say so on the page.

- **Say what to replace, in plain words.** "Replace everything in square brackets with your own text, brackets
  included." Add a two-column wrong/right table built from real fields (`Name: [command post]` is wrong, `Name: command
  post` is right). People type their value inside the hint's brackets unless told not to.
- **Tell hint from value by a marker, not by brackets alone.** A bracketed value that starts with the fixed hint words
  (here «обов’язково», «за потреби») is an unfilled hint: the field is absent. Any other wholly bracketed value is the
  sender's own text: strip the brackets, keep the text, warn once. Never drop data because of decoration; brackets inside
  a sentence stay untouched.
- **Examples come from the real reference list.** Take type names, codes and units from the actual list (or its export),
  not from memory: an invented example ("Command post", "Warehouse") teaches senders a value the checker rejects. Generate
  the list's table in the spec from the same file.
- **Rename a field label by keeping the old one as an alias.** A better canonical name ("code" instead of "number" when
  it holds both) costs senders nothing when the old label stays an accepted alias; the stored key does not change. The
  spec test that feeds the sample to the current parser maps the new label back until the parser knows it.
- **Write the maintainer's answers into the spec the same day**, as the rule and as a row in the open-questions table
  when the answer opens a new question ("depends on the source" is a question, not a rule).

Keep the exact anchor text (a bold line or heading) that the spec test looks up; add instructions below it, never change
the anchor to add punctuation.

## Make the spec run

A spec that is not executed drifts ([docs-match-code.md](docs-match-code.md)). Treat the sample as the first test case
(specification by example: concrete examples are the requirements and become automated tests):

1. The sample and "what we get" block are extracted by the test, parsed, and compared (a golden example). While the
   parser does not exist, the test pins the gap (it fails when the parser starts to read the format, as a reminder to
   replace it by the full comparison) with the backlog item id.
2. Every row of the field table has at least one accepted and one rejected or warned example; a table-driven test
   reads the examples column.
3. Changing the parser and the spec in one commit; the gate fails when the sample no longer parses to the block.
4. At implementation: flip the banner to `implemented`, delete the "planned" markers, keep the page (it is the manual's
   reference).

## Where it applies

Input formats people type (reports, forms, commands), config files, API payloads (OpenAPI `examples` follow the same
sample-plus-result idea), import/export files. For JAS Bot: every report type that senders write by hand gets a
spec page before its parser changes; the sender manual links to it, and the operator view can show the warnings it
defines ([measures-of-performance-and-effectiveness.md](measures-of-performance-and-effectiveness.md)).

## Open

- Whether a shared helper that extracts fenced sample/result pairs from Markdown belongs in HQ or in each repo
  (one repo uses it so far, as a test with a few helpers; stays in that repo until a second one needs it).
- INCOSE/ISO 29148 wording for requirements was not checked; this page is for formats, not system requirements.

## Sources

- RFC 2119, key words MUST/SHOULD/MAY: <https://www.rfc-editor.org/rfc/rfc2119> (not re-read for this page; well known).
- Specification by Example (examples as requirements, automated tests, living documentation), read 2026-11-04:
  <https://concepts.dsebastien.net/concept/specification-by-example/>
- Observed in `auto-joint-lab/report-bot`, 2026-11-03 to 2026-11-04: the strike format page (status banner, principles,
  sample, parsed result, field table with types, multi-value rules); its parser did not exist yet. Later the same days:
  the two required columns, strict/lenient checks table, aliases, the unit-of-data rename, a legacy-report reader table
  and the sender-facing "Briefly" part; the page is run by `rust/crates/jasbot-core/tests/strike_spec.rs`.
- Plain-language guidance for instructions and error messages (not re-read; general knowledge): plainlanguage.gov
  guidelines and the Nielsen Norman Group error-message guidelines.
- Observed in `auto-joint-lab/report-bot`, 2026-11-04 (second round): senders could write values inside the template's
  brackets, so the page got a replace-the-hints instruction, a wrong/right table and the strip-and-warn rule; type
  examples invented from memory were replaced from the exported reference list; the first field was renamed from
  "number" to "code" with the old label kept as an alias; the spec test needed the old label until the parser changed.
