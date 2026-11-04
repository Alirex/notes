---
summary: "Researching health and supplement claims with subagents: evidence levels, sourcing rules, blocked sources, no personal advice"
group: agents
tags: [research, health, supplements, docs, subagents]
applies-to: [all]
reviewed: 2026-11-01
status: draft
---

# Supplement and health evidence research

For a notes repo that collects health or sport claims. Notes are for reading, not for
advice.

## Page contract

- One page per topic; a short summary in the reader's language on top, the full text in one language below.
- A top-level disclaimer in the README and in the index: research notes, not medical advice.
- Every claim carries a source URL and an evidence level: `strong` (several consistent systematic reviews or
  meta-analyses, or a regulator or position stand built on them), `moderate` (one good review or several RCTs with
  limits), `weak` (single small trials, narrative reviews, animal or mechanistic data, search summaries), `none`
  (manufacturer or retailer statement, or nothing found). A maker's statement about its own product is `none` as
  evidence, even when quoted exactly.
- A dose is quoted only as "what trials used", with the study, population and limits. No "take X".
- A "Not verified" section on every page lists what was not opened or not found; nothing is dropped silently.
- Prices: shop, URL and date, since they go stale in days; a price seen only in a search snippet is marked so.

## Sources, best first

Systematic reviews and meta-analyses (Cochrane, journals), position stands (ISSN, ACSM), regulators (EFSA opinions,
NIH ODS fact sheets), manufacturers' certificates of analysis for product claims. Treat trade press, aggregators and
retailer pages as `weak` at best. Note author ties to the industry when a position stand or review discloses them.

## Method with subagents

1. Give each cheap (Sonnet or Haiku) subagent one topic, the page contract above and a target file in a scratch folder
   outside the repo; ask for the file path and a list of unverified items only.
2. Require "cite only what you opened; never invent a citation" and an explicit list of sources that failed.
3. The supervisor reviews: spot-check tags against the draft, fix contradictions between the comparison table and the
   pages, and check any figure that looks too clean (own arithmetic must be labelled as such).
4. Expect blocked sources: NIH ODS, EFSA, PubMed and publishers often answer 403 or a captcha to automated fetchers;
   the Europe PMC REST API returns abstracts. Drafts built on abstracts and search summaries are drafts; file a backlog
   item to re-verify in a real browser and say so in the index.
5. Check the brief for facts you stated from memory: the first run found a wrong journal for a cited meta-analysis.

## Pitfalls seen

- Quality marks cover different things (banned-substance testing, label match, raw-material origin); none proves
  efficacy, and none stops a counterfeit sold on a marketplace.
- Position stands and narrative reviews are often written by people with industry ties; say so.
- Own molar-mass arithmetic or a per-dose calculation is useful but is not a source; label it.
- A wait for slow subagents is a completion notice, never a sleep loop.
- Cite one landing page per paper (`https://europepmc.org/abstract/MED/<PMID>`, else the DOI), never a Europe PMC
  REST query URL: a query is not a source and its results change. Say "abstract only" until a full text was read.
- Every safety caveat, interaction and "limits unverified" note goes into every language's summary, not just the
  body or one language; sweep for it as a separate pass (a review found the RU and EN summaries diverging).
- Do not put the maintainer's own health, age or training profile in the repo: write the audience as one generic
  line (the repo's visibility may change).
- Split a page sweep across agents by disjoint file sets, one branch each, and land them one at a time; the
  index and comparison pages are edited by one agent only, or after the sweep.

## Citing

- Cite the human-readable landing page (the DOI, or `europepmc.org/abstract/MED/<PMID>`), never an API query URL.
- Every safety caveat in the English summary appears in every translated summary too.

## Sources

- ISSN position stands, via Europe PMC (titles, authors and years read 2026-11-01; abstracts not read):
  creatine 2017, Kreider et al., doi:10.1186/s12970-017-0173-z; protein 2017, Jäger et al., doi:10.1186/s12970-017-0177-8.
- ISSN review (not a position stand): Antonio et al. 2021, "Common questions and misconceptions about creatine
  supplementation", doi:10.1186/s12970-021-00412-w.
- Caffeine 2021 ISSN position stand: DOI not confirmed in this review; look it up before citing.
- EFSA opinions and the NIH ODS fact sheets are preferred sources by this page's method, but they were not opened in
  this review (EFSA redirects to a paywalled host, ODS returned 403 to an automated fetch): open them in a browser.
- GRADE, the four certainty levels this page simplifies: <https://www.gradeworkinggroup.org/>
- EFSA and NIH Office of Dietary Supplements as the preferred regulator and fact-sheet sources.
- Evidence-level wording adapted from the GRADE idea of certainty, simplified to four labels.
- First run: a private health-notes repo, 2026-11-01.
- Observed in a Fable review of a managed app (2026-11-01); no external source beyond those linked above.
