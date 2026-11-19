---
summary: "Who lands a branch (worker or supervisor), the ship command, and how to read gate results before landing"
group: agents
tags: [agents, workflow]
applies-to: [all]
reviewed: 2026-11-12
status: stable
---

# Agent landing: who lands and gate results

Part of [agent-workflow.md](agent-workflow.md#stages); the picture is the Atlas `workflow` view in `docs/architecture/workflow/`.

## Who lands

A worker that waits for the supervisor to land becomes a stale idle session (2026-11-07/08: a dozen). So landing
follows the brief, not the role:

- **`land: self`** (the default for size `s` and `m` items with a green gate, when the machine is not in the
  branch-only mode of [autonomous-sessions.md](autonomous-sessions.md#use-the-plan-limits-fully)): the worker runs
  `just repos-ship <repo> <branch>` (merge main, gate on the merged result, land, finish, pull), puts the `repos-ship`
  result line in its hand-back and ends. The supervisor checks afterwards with `just repos-dod` / `repos-landed`, a report
  is still a claim.
- **What a ship waits for.** The heavy lock protects the timing of Playwright and vitest, so a *full* gate waits for
  it. A *light* gate (docs, practices, scripts closure: `gate_scope.py --kind`, no `crates/` or `web/hq/` file) cannot
  disturb them and waits only for room (no memory strain, load under 1.2 per core): HQ docs ships had waited 742 min in
  27 h behind one long gate (`HQ_SHIP_LIGHT_BYPASS=0` restores the single wait). A repo whose justfile has a `q-passed`
  recipe that exits 0 skips the gate (`gate: skipped (repo pass check)`; `HQ_SHIP_PASS_CHECK=0` turns it off). A repo whose justfile has a `q-precheck` recipe (its cheap checks: lint, fast
  types, `cargo check`) has it run after the ship fixers and before the heavy wait; a failure is `NOT shipped at precheck:
  <last output line>` and the ship never waits for heavy (`HQ_SHIP_PRECHECK=0` turns it off). The same stage runs the
  backlog check (when the branch touched `docs/backlog/`) and the conformance checks that pass on the base, so a
  malformed item fails in seconds (`NOT shipped at precheck: backlog check: <first line>`; `HQ_SHIP_CHEAP=0` off).
  A merged tree that already has a gate PASS record (the proof names the tree hash, never a branch; any change of the
  tree is another hash) waits for no room and no slot and runs no gate (`gate_kind: carried`; `HQ_SHIP_PROOF=0` off). `land`
  refuses a branch that lacks the base (`base moved, ship again`), so a moved base is never landed ungated, with one
  exception: when the commits that arrived on the base since the branch's merge base touch no file the branch changed,
  none of them a shared build or gate file (justfile, `just/`, hook config, `scripts/quality/`, `.github/`, manifests,
  lockfiles, generated API files, `needs.yaml`, `AGENTS.md`), and the base names no item file the branch deletes, `land`
  merges and pushes without another gate and prints `landed on moved base: <n> commits, paths <...>, no re-gate` (a 6 min
  gate repeated four times in 30 min because other sessions landed meanwhile). Disjoint paths are a path rule, not proof
  of independence: the gate that ran on the branch already covered the shared files. A shared-file arrival, an overlap,
  a conflict or a doubt keeps the refusal; `HQ_SHIP_REGATE_ALWAYS=1` turns the exception off. No repo
  is owned by a machine: a set `owner_machine` is a warning (`HQ_SHIP_OWNER_STRICT=1` refuses). `ships.jsonl` rows carry
  `gate_kind` (light, full, carried), `heavy_holder` and `bypass`; `just repos-ship-waits` splits the wait by kind. A failed row (`shipped:false`) also
  carries `failed_stage` (the failed hook names, else the `NOT shipped at <stage>` word, or `killed`) and a one-line
  `reason` (at most 200 chars); `just repos-ship-stats --view failures` groups failed attempts by repo, stage and reason
  with the minutes lost. When the landing cannot fast-forward the main checkout (a dirty tracked file), `repos-finish`
  and `land` print `MAIN CHECKOUT BEHIND: <reason>`; never stash or reset it, fix it by hand.
- **`land: supervisor`** (size `l`, a gate change, a security or auth change, an outward-facing step, or branch-only
  mode): the worker pushes the branch, queues a landing request
  (`scripts/claude/fleet/fleet-starts.py add <repo> <item> --kind landing`), hands back and ends; it does not wait. The
  supervisor reviews, gates and lands. The Stop hook (`scripts/claude/hooks/stop-next.py`) refuses to end a `worker`
  turn whose branch is ahead of main with no such request.
- **One hand-back per worker.** A worker sends its hand-back block once, when it ends. A second one for the same sha
  (a re-send, a late "completed" note) is a no-op: the supervisor checks the sha with `repos-landed`, does not re-review
  and does not answer it. A changed sha is a new hand-back and is checked again.
- A worker never ends with an unpushed branch; after the hand-back it has nothing left to wait for, so the supervisor
  archives it at the next round (the four checks below) without a nudge.

**Definition of Done** (a worker's hand-back block shows each line; a missing line is "not done"; the reasons are in
[agile-lessons.md](agile-lessons.md)):

- [ ] Gate green on the merged result. After a failed run: the failing hooks, the load average at the time and the
      second result; a second failure is a finding, not "flaky".
- [ ] Item file deleted in the landing commit.
- [ ] Pushed, landed, and the main checkout pulled, **shown as proof**: the output line of `just repos-landed <repo>
      <sha>` (pushed: the `ls-remote` line). A note that says "landed" or "pushed" with no sha or branch is `check`
      in `just cc-fleet-round` (2026-11-06: a fast-forward "pushed" that origin/main did not contain).
- [ ] `Request: R-… -> <commit | item id | answered>` for every maintainer request the work served, so the ledger
      closes from the hand-back (`scripts/docs/requests.py accept FILE` marks what the line proves: a sha on origin's
      default branch, an item file, or `answered`; `just docs-requests check` lists the rest). `just repos-dod <repo> <ref>
      [--item --request --handback FILE]` prints one `ok`/`FAIL` line per checkable item above.
- [ ] A task a supervisor sends gets a fleet-ledger row `dispatched YYYY-MM-DD <handle> <repo> expects sha|branch <ref>`;
      `just cc-fleet-round` lists it as `waiting` until that artefact is on the default branch.
- [ ] Lessons saved, as the line `Lessons saved: <practices page or item id | none-needed>` (a `practices/` commit or an
      `area: practices` item; `just cc-fleet-round` shows a landing without this line as `landed, no lessons`).
- [ ] The closing line `Retire: ready branch=<b> sha=<s> lessons=<page|none-needed>`
      ([the Retire line](agent-workflow.md#the-retire-line)): the sha and branch the retire checks verify with git.
- [ ] Docs match the code (README, AGENTS.md, changelog where the repo keeps one).
- [ ] **Checked in:** `real app` / `mock app` / `tests only`, with the data it used (the real log, a fixture, a seed).
      `real app` is named by its access level ([agent-access-to-apps.md](agent-access-to-apps.md)): `real API (token
      <label>, scope)`, `real browser, human session, read-only` or `real browser, agent identity <role>`; say what
      that level did not see.
      A UI or endpoint change that says `mock app` or `tests only` is flagged by the supervisor before it reports
      done: it is looked at in the real app, or listed as "unchecked in the real app" in the round table. (The MCP stats
      card landed after a check on a mock only; the real log showed it differently.)
- [ ] **Outcome measured:** `yes` (number, source) / `not yet` (check date) / `n/a`: gate green is MOP, the outcome
      is MOE ([measures-of-performance-and-effectiveness.md](measures-of-performance-and-effectiveness.md)).
- [ ] **Done when:** each line of the item's Done when → evidence (command output, screen, test name) or `not met (why)`
      (size `m` and up).
- [ ] `Flow: vX.Y.Z` and `Role: <role>`: the flow version the session follows (the first line of its brief or of the
      digest it was shown at start) and its role (worker, autopilot, reviewer, ...). The supervisor's round shows both
      per session and flags `stale` when a major or minor flow change came after the version; a stale session gets the
      changelog slice or a restart, not a full re-brief ([flow-versioning.md](flow-versioning.md)).
- [ ] Session retired ([Retire a session](agent-retire.md#retire-a-session)).

Small work skips 3 and the separate reviewer but not 5's checks. Questions never stop the flow: a
`maintainer-question` issue, safest reversible choice meanwhile ([autonomous-sessions.md](autonomous-sessions.md)).

## Landing lane

Landing is one lane per repo, and a gate takes minutes, so ships that run one after another make a queue (observed
2026-11-12: 43 branches behind one lane, a hand-back waited hours). Merge queues solve the same problem by testing
the queued changes together as one speculative result, not one by one. The rules:

- **Run the dry run every round**, not on demand (HQ: `just repos-land-batch <repo>`); it sorts branches into
  patch-equal, clean and conflicting, and lists the conflicting ones in rebase order (a `git merge-tree` trial
  against the default branch names the conflict files; fewest files and least overlap with the others first).
- **Batch the clean branches** into one candidate branch and one gate; do not ship them singly while a batch could
  take them. If the batch gate fails, split it and keep the passing half. HQ: a repo entry with `landing: batch`
  (registry) opts into `just repos-batch-round [repo] [--apply]`, which builds and ships the batch and plans the singles.
- **Rebase the conflicting ones by worker**, one per group of shared files (agent docs, hook config, glossary,
  generated files, the keeper script). Generated files are regenerated, never merged by hand: a merge that conflicts
  only in files of the generated-files manifest (`scripts/repos/generated-files.yaml` in HQ) takes the base's version and
  runs each file's generator (`resolve_generated.py`, called by `repos-ship` and the batch trial merge); one other
  conflicting file still stops it.
- **Land earlier**: a worker of size `s`/`m` with a green gate lands its own branch (`land: self`, above), so
  nothing waits for the supervisor.
- **Close stale branches**: older than a day with no new commits and not a live worktree: save a backlog item, close
  the branch, so it stops counting as work in progress.
- **Cap the queue**: at most a handful of ships waiting (HQ: 5); more hand-backs wait as branches for the batch.
- Keep hand-backs small: many small branches land faster than one large one.

### Batch order, deferral and rerere

- Build the batch smallest diff first, and keep two branches that touch the same file out of one batch: the later one
  waits for the next batch even when it merges cleanly, so each batch gate tests independent changes and a failure
  points at one branch.
- A conflicting branch is dropped, never stops the batch; the line names the files and the batched branch they
  overlap, and the rebase worker's brief is printed with it (HQ: `just repos-land-batch <repo>`).
- Turn on `git rerere` (`git config rerere.enabled true`): git records how a conflict was resolved and replays it. The
  data sits in the repository's common dir, so every linked worktree shares it; a rebase that hits the same conflict
  twice (a retry, a re-queued branch) is not resolved by hand again. HQ sets it from `repos-land-batch`.
- A conflict only in backlog item files (progress text every worker edits) needs no worker: take the base side for
  those files and keep the branch's other changes; a branch left with only a backlog edit still goes to a rebase
  worker, so its edit is not dropped. A conflicting branch whose item is `blocked_by` something, or whose tip is over
  24 h old with no worktree, is parked: listed once as "decision needed" with its supersede command, not counted as
  conflicting.

### Superseded branches

Observed 2026-11-12: 18 of 20 conflicting branches held work that the default branch already had in another form (a
second session built the same feature, or ported the useful part and left the branch). Four classifier rounds and
about 300k subagent tokens found it; the branches had waited days. A conflicting branch is not always unmerged work.

- **Before a brief goes out**, search the default branch for the feature, not only for the item id: `git log
  origin/main --grep=<id>` and `git grep` for the key identifier or recipe name. A worker on a feature that main has
  wastes its whole run.
- **Land or close in the round the hand-back arrives.** A branch with no session that is older than a day is "land or
  close", never a standing branch.
- **Read the dry run's evidence**: a conflicting branch whose item id or commit subjects are on main, or whose every
  touched file main rewrote, is probably superseded. Confirm cheaply (a read-only classifier, then a spot check on
  main), do not rebase first.
- **Close with an archive tag, not a delete**: push `archive/superseded/<branch>` at the tip, save any uncommitted
  files of its worktree outside the repo, then remove the worktree and the branches. The closing is reversible, and
  the maintainer's OK is needed only for branches the classifier cannot prove superseded. The command checks the proof
  itself, not the free-text evidence: tip merged, every patch equal (`git cherry`), or a backlog item id of the branch
  or evidence cited on the default branch; and it refuses a tip younger than 2 h (a session may still push). Without
  proof or past the age check it needs `--maintainer-ok` with the maintainer's words, recorded in the tag message.
- **Treat a worker's "merges cleanly" as a claim**: re-run `git merge-tree --write-tree --name-only` yourself; one
  worker called a conflicting merge clean and the conclusion still held only by luck.

## Gate results and landing

- `EXIT=143` is a killed gate (a TERM or memory pressure), not failing tests: rerun it once room is back
  (`machine-wait --for heavy`); `gate-bg.sh result` prints `KILLED`, not `FAIL`.
- On a `FAIL`, `gate-bg.sh result` adds a line `triage: <log>.triage.md` when the local model has written a short note
  (failing hook, test, first error, likely cause); read it before the whole log, and check it against the log.
- Run the full gate on the merged tree (main merged in) and land right after PASS; a gate on a stale base proves nothing.
- A worker's gate on a dirty tree (uncommitted edits, a hook that rewrote a file) records no pass for the commit:
  commit first, then gate the committed tree; the supervisor re-gates it if the worker's line says "not recorded".
- Run the repo's fixer step (`q-fix`) before the last commit: a gate that ends `PASS (dirty)` costs a second full gate.
  `repos-ship` also runs the fixers on the merged tree before the heavy slot and commits what they change.
- Interim notices are not reports: the supervisor acts on the hand-back block only, and a repeated "nothing pending"
  notice gets one line back.
- Check a backlog item's status against the vocabulary and audit a "missing" route list against the code before filing.
- Parallel agents that commit without pushing (`isolation: worktree`, disjoint file sets, one writer per file set):
  the supervisor cherry-picks, runs the repo's format, lint and test checks, pulls with rebase, pushes and checks
  `just repos-landed <repo> <sha>`. After a cherry-pick the shas differ, so `repos-finish` fails: remove the worktree
  and branch by hand (`git worktree remove --force`, `git branch -D`) once `repos-landed` proves the content. A
  running dev server keeps its old binary: restart it after a landing.
- Never `tail -f` a gate log in a shell: wait for the completion notice or read `q-gate-result`.
- Scoped gate (R-2611151822-8ecd): a branch whose files are all docs, backlog, practices and `scripts/<module>/` runs
  only the hooks those files wake (the module's tests, ruff, basedpyright, shellcheck, just-recipes lint), so the slow
  `repos-ship-tests` and `preflight-tests` run only when `ops.sh`, `preflight.sh` or their tests changed. The gate line
  stays `gate: fast path, scoped (<modules>), N file(s)`. Shared files (`justfile`, `just/*.just`,
  `.pre-commit-config.yaml`, pyproject and lock files, `scripts/quality/`, `scripts/web/`, `scripts/machine/profile/`) and
  any unmapped path force the full gate. A script a Rust or web file names, or a closure over 40 files, also runs full.

See also [team-of-teams.md](team-of-teams.md): context before authority, endorse in public and correct in private, for
agents and for people.

## Sources

- Split out of [agent-workflow.md](agent-workflow.md), which keeps the sources of the whole flow; the Atlas `workflow` view is in `docs/architecture/workflow/`.
- Claude Code hooks: <https://code.claude.com/docs/en/hooks>
- Merge queue (queued changes tested together on a temporary branch): <https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue>
- Zuul gating (speculative future states, shared test of queued changes): <https://zuul-ci.org/docs/zuul/latest/gating.html>
- Trunk-based development (small, short-lived branches): <https://trunkbaseddevelopment.com/short-lived-feature-branches/>
- The numbers (43 branches, 14 clean, 29 conflicting; later 18 of 20 conflicting superseded) are HQ's own, 2026-11-12.
- Git rerere (reuse recorded resolution): <https://git-scm.com/docs/git-rerere>; Pro Git, Rerere: <https://git-scm.com/book/en/v2/Git-Tools-Rerere>
- Git tags for archiving a ref without keeping a branch: <https://git-scm.com/docs/git-tag>; `git merge-tree --write-tree`
  (trial merge without touching the worktree): <https://git-scm.com/docs/git-merge-tree>
