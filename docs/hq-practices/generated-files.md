---
summary: "Generated files must not conflict in merges: merge driver, regenerate after the merge, a hook that checks"
group: quality
tags: [git, merge, generated, ship]
applies-to: [all]
reviewed: 2026-11-11
status: draft
---

# Generated files

A generated file is edited by every branch, so branches that each regenerate it conflict on every merge, land one at
a time, and each rebase conflicts again. Observed in HQ and report-bot (retro F18): `practices/practices.json` changed
in 247 commits, i18n files in 104 (HQ) and 168 (report-bot), openapi/revision/schema files in 76-110; 54 report-bot
merges carry `Conflicts:`; the 149 branch pairs that shared generated files made the ship queue (gate seats) the
bottleneck.

## Rules

- Mark each generated file in `.gitattributes` with a repo-local merge driver that keeps ours (`merge=<name>`); a
  merge never conflicts on it. A file that mixes hand-written text and a generated block (a README with a table) gets
  a driver that ignores the generated block and merges the rest normally.
- The driver is repo-local git config, so a script installs it (`just setup-hooks`, part of `setup-all`) and the ship
  step sets it too; `.gitattributes` alone does nothing.
- The ship step regenerates the files after the merge and commits the result, so the landed tree is always current.
- A pre-commit hook regenerates the files and fails when the result differs from the commit (HQ: `practices-current`,
  `openapi-stale`); the driver only removes conflicts, the hook proves correctness.
- Never hand-edit a generated file to resolve a conflict.
- When regenerating is expensive (a compiler build), the ship step regenerates only when both sides changed the file
  since the merge base: only then did the driver keep ours, so only then can the result be stale.
- A contract revision that clients compare by "at least" stays a counter, not a hash: a hash has no order. The counter
  is derived from the spec hash (it moves only when the hash moves), and after a merge that kept ours the regenerated
  spec bumps ours by one, so two branches that both took N+1 land as N+1 and N+2, without a conflict.

## HQ

`practices/practices.json`, `practices/map/index.html` and the `practices/README.md` table:
`scripts/practices/merge-generated.sh`, `.gitattributes`, regenerate step in `scripts/repos/ops.sh ship`. The explainer
`i18n.data.js` files, `explainers/index.html` and `explainers/_shared/learn.data.js` use the same driver. The embedded
diagram wasm module and its fingerprint too: ship runs `scripts/docs/wasm.sh build` when both sides changed the
fingerprint, and the `wasm-current` hook proves it. The API files (`crates/hq-api/openapi.json`, `api_revision.json`, the page's
`web/hq/src/api/schema.ts` and `revision.ts`) too: the `openapi-stale` hook regenerates them (`cargo-hook.sh
openapi-fix`, the same as `just web-openapi`) and fails when they moved; ship runs `openapi-fix` when both sides changed
`openapi.json`, in the gate's build dir.

## Sources

- gitattributes, "Defining a custom merge driver": <https://git-scm.com/docs/gitattributes#_defining_a_custom_merge_driver>
- Measured in HQ and report-bot git history (retro R-2611080029-2e46, F18); the driver design is HQ's own.
