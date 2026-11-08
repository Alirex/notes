---
summary: "Secret scanning, dependency audit and cooldown, Actions hardening, repo settings"
group: security
tags: [security, deps, ci]
applies-to: [all]
reviewed: 2026-10-30
status: stable
check: "script: sec-check"
---

# Supply-chain and repository security

What keeps a secret out of git, a malicious release out of the lockfile, and a workflow from becoming a way in.
App-level security (passwords, sessions, 2FA) is in [auth-and-security.md](https://github.com/evocaslab/hq/blob/main/practices/auth-and-security.md), the OWASP checklist in [web-app-security.md](https://github.com/evocaslab/hq/blob/main/practices/web-app-security.md); agents in
[agent-security.md](agent-security.md); the one command that runs these scanners over every repo in [security-tooling.md](security-tooling.md); hosts and containers in [host-security.md](https://github.com/evocaslab/hq/blob/main/practices/host-security.md), a VM that serves the internet in [internet-facing-vm.md](https://github.com/evocaslab/hq/blob/main/practices/internet-facing-vm.md).
Status checked 2026-10; tool versions move, `pre-commit-update` / `just repos-security` show the current state.

## Rules

1. **Secret scanning runs in the commit hook, in every repo.** The hook is the only defence that does not depend
   on a GitHub plan (see "GitHub settings"). Never allowlist a real secret: rotate it, then rewrite the history.
2. **Every ecosystem has an audit step** that runs in CI or the weekly chore (not per commit: it needs the network).
3. **Lockfiles are committed and enforced**: `uv sync --locked/--frozen`, `pnpm install --frozen-lockfile`,
   `cargo ... --locked`. CI never resolves.
4. **New releases wait.** Every dependency update path has a minimum release age (a cooldown) of about 7 days.
5. **Workflows are least-privilege and pinned**: `permissions:` set per workflow, actions pinned by full commit SHA,
   zizmor in the gate.
6. **Turn on what GitHub offers for the repo's plan** and record what is not available.

Private shared libraries (git tag pins, deploy keys, how repos learn of a new tag) are in [shared-libraries.md](shared-libraries.md).

Cloud infrastructure code (OpenTofu state, SOPS secrets, Cloudflare token scopes, Trivy config scan) has its own page: [infrastructure-as-code.md](https://github.com/evocaslab/hq/blob/main/practices/infrastructure-as-code.md).

## Secret scanning

- **gitleaks** (`gitleaks/gitleaks`, hook id `gitleaks`, v8.30.1, still released in 2026) scans the staged changes.
  The hook builds the Go binary itself (prek and pre-commit handle `language: golang`), so no system install.
  **Betterleaks** (`betterleaks/betterleaks`, by gitleaks' original author, MIT, reads gitleaks configs, hook ids
  `betterleaks`/`betterleaks-system`) is the successor; its 2.0 line is still release candidates, so stay on gitleaks
  and revisit when 2.0 is stable.
- Add a manual-stage twin that scans the whole history (`entry: gitleaks git --redact --verbose`, `stages: [manual]`,
  `alias: gitleaks-history`) and run it once when adopting, then monthly. HQ: `just q-secrets`.
- False positives (a SHA-256 of a file, a test token that is not a credential) go in `.gitleaksignore` by
  fingerprint, one line with a comment, never in a broad path or rule allowlist. A real hit stops the work:
  report it, rotate the credential, only then clean the history.
- `--redact` always: findings end up in logs and agent transcripts.
- Why not only GitHub secret scanning: push protection is a push-time check, after the secret is on the machine, in
  reflogs and possibly in a PR; and on private repos of a free plan it is not offered at all.

## Dependency audit

| Ecosystem | Audit | Notes |
| --- | --- | --- |
| Python (uv) | `uv audit --locked` (OSV; flags `--ignore ID`, `--ignore-until-fixed ID`, `--output-format json\|sarif`) | still marked experimental; reports no severity. `uv audit --script x.py` for PEP 723 scripts. Alternative: `pip-audit`, `osv-scanner` |
| Python install-time | `UV_MALWARE_CHECK=1` | opt-in: `uv add`/`uv sync` stop on a known-malicious package (OSV MAL advisories) |
| Node (pnpm) | `pnpm audit --prod` | dev tools matter too (they run on the dev machine), but the signal is noisier |
| Rust | `cargo deny check` | advisories, licenses, bans, sources in one tool ([rust-service-tooling.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service-tooling.md#dependencies)) |
| Any lockfile | `osv-scanner scan -L <lockfile>` | one tool for mixed repos |

- A finding is fixed or ignored with a reason and an expiry (`--ignore-until-fixed` re-fails when a fix appears).
  No silent ignores ([lint-strictness.md](lint-strictness.md)).
- Audit databases only know what was reported; the cooldown below is the other half.

### Scheduled watch

New advisories appear after a lockfile was last checked, so audit on a clock too, not only in the gate.

- A periodic job (a systemd user timer, or a CI cron) runs the OSV audit (`uv audit --frozen`, `osv-scanner`),
  `cargo deny check advisories` and `npm audit` / `pnpm audit` per repo. Read-only permissions; in CI the actions are
  SHA-pinned like everywhere else.
- A hit files a backlog item (area and priority from the finding) instead of failing silently or paging at night.
  One item per subject, never a second one while one is open.
- No indicator-of-compromise scanner until there is a trustworthy feed to feed it.
- HQ's version of the whole watch, machine and repos: `just setup-outdated` and `hq-outdated.timer` (below).

## Updates by class

Detect everything, install only what is safe to install unattended. A report that nobody reads is the failure to
avoid, so it is weekly, short, and files an item only when something needs a decision.

| Class | Examples | Policy |
| --- | --- | --- |
| Installs itself | OS security channel (`unattended-upgrades`), snap refresh, tools with their own updater | Automatic. The weekly report only flags it when it is still pending after 7 days (the automation is stuck) |
| Detected, then a recipe | uv and uv tools, Node LTS, hook revisions | Reported weekly; a person runs `just setup-upgrade` / `setup-node` and commits the change |
| Dependencies of a repo | Python, JS and Rust locks, pinned GitHub Action SHAs | Reported weekly; fixed through a branch or a bot PR and the repo's gate, with the cooldown below. Whether a repo runs Renovate or Dependabot is that repo's decision (`area: review` item), never enabled from HQ |
| Manual window | Kernel, NVIDIA driver, Docker engine, container images, other apt updates, reboots, anything on the production host | Reported weekly, installed by a person at a chosen time with the services checked after. Never unattended |
| Never touched | The production app's data directory and database container | Not even to update; the owner repo moves them |

- **Security ranks above "a newer version exists."** Known-vulnerability data is OSV (`uv audit`, `pnpm audit`,
  `cargo deny`); a hit that is not auto-installed files an item at once, a plain newer version only after it has stayed
  30 days.
- The check is read-only: no sudo (so `apt` lists are as fresh as the system's own daily refresh), no installs, no image
  pulls. Docker tags come from the registry's tag list (one request per image per week); a major-only or local tag
  cannot be compared and is listed as not checkable.
- **Each check is isolated**: a missing tool or a network error marks that check "not checked" in the report and the rest
  go on. A report must say what it could not look at.
- Alerts reuse the session watch's channel: one desktop notification per security finding, once.
- HQ: `just setup-outdated` (`--json`, `--only os|snap|uv|node|docker|prek|hq|repos`), `just setup-outdated-report`,
  the line in `just setup-check`, and `hq-outdated.timer` (`infra/services.yaml`; `just setup-services diff` shows it,
  the maintainer installs it).

## Cooldown (minimum release age)

Most malicious releases are found within days. Do not install what is younger than about a week.

- **uv**: `[tool.uv] exclude-newer = "7 days"` (relative durations; calendar units are refused). The lockfile
  records the span and moves it only when you re-resolve (`uv lock --upgrade`). A single urgent fix:
  `exclude-newer-package = { pkg = "..." }`.
- **pnpm**: `minimumReleaseAge: 10080` (minutes) in `pnpm-workspace.yaml`, with `minimumReleaseAgeExclude` for an
  urgent fix. Set it explicitly even where a newer pnpm defaults it.
- **Cargo**: no native setting; Dependabot/Renovate cooldown on the `cargo` ecosystem, and `cargo update` only
  in the reviewed monthly commit ([rust-service-tooling.md](https://github.com/evocaslab/hq/blob/main/practices/rust-service-tooling.md#dependencies)).
- **Bots**: Dependabot `cooldown: {default-days: 7}` per ecosystem (covers `github-actions`, `pre-commit`, `uv`,
  `npm`, `cargo`, `docker`); security updates bypass it by design. Renovate: `minimumReleaseAge: "7 days"`.
  Keep the bot's value equal to the package manager's, or the bot opens PRs the lock step cannot resolve.
- Security fixes that matter now are applied by hand with the exclusion, after reading the advisory.
- Hook revisions follow the same rule: no auto-updater hook (such as `pre-commit-update` run as a hook) that moves
  revs inside the gate. Revs change only through Dependabot's `pre-commit` ecosystem or a reviewed commit, under the
  cooldown; an updater may run as a report, never as a fixer.

## GitHub Actions

- `permissions: {}` at workflow top, then the minimum per job (`contents: read` usually). Never rely on the
  repository default token scope.
- **Pin by full SHA** with the version in a comment (`uses: actions/checkout@<sha> # v7.0.0`). Tags move; a SHA does
  not. Bots (Dependabot `github-actions`) update SHA pins. Turn on the repo/org policy `sha_pinning_required`
  (Settings, Actions, General) once all workflows are pinned. `pinact` converts existing tags.
- `actions/checkout` with `persist-credentials: false`; no `secrets: inherit`.
- **No `pull_request_target`, `workflow_run` or `issue_comment` with a checkout of PR code**, and no
  `${{ github.event.* }}` interpolated into `run:` (pass it through `env:`). These are how fork PRs and issue
  titles become code execution with secrets.
- Do not run untrusted PRs on self-hosted runners; do not store long-lived cloud keys as secrets when OIDC works.
- **zizmor** (`zizmorcore/zizmor-pre-commit`, hook id `zizmor`, v1.31) catches all of the above:
  `unpinned-uses`, `excessive-permissions`, `dangerous-triggers`, `template-injection`, `artipacked`,
  `cache-poisoning`, `secrets-inherit`, `impostor-commit`, `known-vulnerable-actions`. It only checks workflow,
  `dependabot.yml`, `action.yml` and pre-commit config files, so the hook costs nothing in a repo without them.
  Add it to a repo's config together with its first workflow.

## GitHub settings

Read-only state of every registry repo: `just repos-security` (`scripts/repos/security.sh`, GET requests only).
Per repo, as far as the plan allows:

- Secret scanning and push protection on (free on public repos; private repos need GitHub Secret Protection, a
  paid add-on: the API answers `n/a` and the local hook is the defence).
- Dependabot alerts on (free everywhere; the `vulnerability-alerts` endpoint). Security updates are optional.
- Private vulnerability reporting on for public repos; `SECURITY.md` (supported versions, where to report) for
  every public repo or one that others use. Private personal repos need none.
- Default-branch protection or a ruleset: no force push, no deletion, required status checks when CI exists.
  Rulesets/branch protection on private repos need a paid plan. Where unavailable, the rule is behavioural:
  agents never force-push the default branch ([git-worktrees-and-autoland.md](git-worktrees-and-autoland.md)).
- Actions: default token read-only, `sha_pinning_required`, fork PR workflows need approval.
- A settings change is the maintainer's: report it as a `manual-action` ([maintainer-questions.md](maintainer-questions.md)).

## Why

- A leaked key in git history is valid until rotated, and history is copied (clones, forks, agent transcripts);
  catching it before the commit is the only cheap point.
- A cooldown turns a 0-day malicious release into an event that only hits you after the registry removed it. It
  costs a week of lag on non-security updates.
- Workflows run with secrets and write tokens on triggers an outsider controls; SHA pins and least-privilege make a
  compromised action or PR a small incident.

## Sources

- Unattended upgrades (security channel by default), snap refresh timing and Docker Hub / ghcr tag lists as used by
  the weekly check: <https://ubuntu.com/server/docs/how-to/software/automatic-updates/>,
  <https://snapcraft.io/docs/managing-updates>, <https://docs.docker.com/reference/api/hub/latest/>
- gitleaks pre-commit hook definition: <https://github.com/gitleaks/gitleaks/blob/master/.pre-commit-hooks.yaml>
- Betterleaks: <https://github.com/betterleaks/betterleaks>,
  <https://www.bleepingcomputer.com/news/security/betterleaks-a-new-open-source-secrets-scanner-to-replace-gitleaks/>
- uv audit and `UV_MALWARE_CHECK`: <https://astral.sh/blog/uv-audit>; relative `exclude-newer`:
  <https://docs.astral.sh/uv/reference/settings/#exclude-newer>; uv and Dependabot:
  <https://docs.astral.sh/uv/guides/integration/dependabot/>
- Dependency cooldowns across managers: <https://nesbitt.io/2026/03/04/package-managers-need-to-cool-down>
- Dependabot `cooldown`: <https://docs.github.com/en/code-security/dependabot/working-with-dependabot/dependabot-options-reference>
- zizmor audits and pre-commit integration: <https://docs.zizmor.sh/audits/>, <https://docs.zizmor.sh/integrations/>
- SHA pinning policy: <https://github.blog/changelog/2025-08-15-github-actions-policy-now-supports-blocking-and-sha-pinning-actions/>
- Observed 2026-10-25 with `gh api` on HQ's registry repos (private repos on a free plan: rulesets and branch
  protection answer 403 "Upgrade to GitHub Pro"; secret scanning fields absent).
- Observed in a Fable review of a managed app (2026-11-01); no external source beyond those linked above.
