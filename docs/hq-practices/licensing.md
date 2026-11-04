---
summary: "Licensing a public repo with mixed content: MIT for code, CC BY 4.0 for texts, all rights reserved for personal material and brand; LICENSE files, NOTICE, README and footer wording"
group: docs
tags: [license, open-source, static-site]
applies-to: [all]
reviewed: 2026-10-30
status: stable
---

# Licensing a public repo

A public repository without a license is all rights reserved: nobody may legally reuse it. Pick licenses by kind of
content and say what is excluded. Dependencies and supply chain are in
[supply-chain-security.md](supply-chain-security.md); the brand kit in [branding.md](https://github.com/evocaslab/hq/blob/main/practices/branding.md). This is a plain
statement of intent, not legal advice. Status checked 2026-10.

## Rules

1. **Split by kind, not one license for everything:**

   | Kind | License | Why |
   | --- | --- | --- |
   | Code | MIT (or ISC, the same in effect; Apache-2.0 when a patent grant matters) | short, permissive, the norm for small tools |
   | Written content (posts, docs, site texts, translations) | CC BY 4.0 (BY-SA 4.0 only to force share-alike) | made for content; credit required |
   | Data without personal content | CC0 | no friction |
   | CV, portraits, photos, personal name, brand name and logo | all rights reserved | personal identity and brand are not open content |

   Creative Commons advises against its licenses for software, so code and texts get different ones.
2. **Say it in the repo, in files a reader and a tool find:** `LICENSE` (code), `LICENSE-CONTENT.md` (content, with the
   legalcode link and a credit line), `NOTICE` (what is excluded and the third-party notes), a `## License` section
   in the README, `"license"` in `package.json` (or the manifest of the language), and one line in the site footer
   linking to `NOTICE`.
3. **Copyright is automatic, a trademark is not.** Texts, logo artwork and portraits are protected the moment they
   exist; a name is protected only as a used or registered trademark. State that the licenses grant no right to the
   name or logo and that forks must not suggest endorsement; registering is optional and costs money.
4. **Third-party material keeps its own license.** Icon sets (simple-icons is CC0 data, but the logos are trademarks of
   their owners; not every icon is CC0, check its disclaimer) are used only to name the technology. List the
   dependency licenses with `pnpm licenses list` or the language's equivalent when something ships to users.
5. **REUSE (SPDX headers or `REUSE.toml`, `reuse lint`) is optional.** For a tiny repo plain files are enough; adopt it
   when a CI check of every file's license is wanted.
6. **Decide once, then record the decision** in `NOTICE` and the repo's AGENTS.md, so agents label new material (code,
   text or personal) correctly.

## Wording to copy

README:

> ## License
> - Code: MIT (`LICENSE`).
> - Blog posts and site texts: CC BY 4.0 (`LICENSE-CONTENT.md`).
> - **Not licensed for reuse (all rights reserved):** the CV, portraits and photos, the name "<Person>" as a personal
>   identity, and the "<Brand>" name and logo. Forks must remove or replace them and must not suggest endorsement.
> - Third-party material keeps its own license: see `NOTICE`.

Footer (translate it): "Code MIT · texts CC BY 4.0 · CV, portrait, name and logo: all rights reserved · License details".

## Sources

- REUSE specification: https://reuse.software/spec-3.3/
- Choosing licenses for mixed deposits (university guide): https://www.rug.nl/digital-competence-centre/research-data/archive-and-publish/how-to-work-with-dataversenl/choosing-a-licence-for-your-dataset
- Unverified: Creative Commons' own text on software; simple-icons per-icon license details.
