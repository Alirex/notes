---
summary: "One wrapper over existing security scanners for every repo: pinned tools, not-checked is never ok, expiring allowlist, findings as backlog items"
group: security
tags: [security, deps, ci, tooling]
applies-to: [all]
reviewed: 2026-11-04
status: draft
---

# Security tooling: one wrapper over existing scanners

How a set of repos gets the same security checks without writing a scanner: use maintained tools, pin them, run them
from one command, and turn what they find into work items. The checks that belong in a commit hook, the Actions rules
and the GitHub settings are in [supply-chain-security.md](supply-chain-security.md); agents in
[agent-security.md](agent-security.md); hosts and containers in [host-security.md](https://github.com/evocaslab/hq/blob/main/practices/host-security.md); the web and
login checklists in [web-app-security.md](https://github.com/evocaslab/hq/blob/main/practices/web-app-security.md) and [auth-and-security.md](https://github.com/evocaslab/hq/blob/main/practices/auth-and-security.md).
Status 2026-11-04: slice 1 (dependencies, workflows, secrets) is built in HQ (`just sec-check`); containers, systemd,
agent settings and web headers are planned. The research behind the choices is in
[docs/research/security-tooling-2026-11.md](https://github.com/evocaslab/hq/blob/main/docs/research/security-tooling-2026-11.md).

## Rules

1. **Wrap maintained tools; do not write a scanner.** The wrapper only finds the repos, runs the tools, keeps the
   output and files the result. A wrapper that parses code or advisories itself is a new thing to patch.
2. **A tool that did not run is `not checked`, never `ok`.** A missing binary, a timeout, an unparsable report or an
   absent clone prints `not checked` and makes the exit code non-zero. Green means "ran and found nothing".
3. **One line per tool and repo; the full output goes to a file.** `ok`, `FINDINGS`, `n/a` (nothing to scan) or
   `not checked`, plus a count. The report lives under `~/.local/state/<tool>/<repo>/<date>/` with mode 0600, never in
   chat or a ticket.
4. **A secret hit prints the repo, the file and the rule, never the value, and is never allowlisted.** Rotate the
   credential first, then clean the history. The parser reads only those two fields, so the value cannot leak by
   accident; run the scanner with its redact option as well.
5. **Pin every tool: exact version, release URL, SHA256 of the download, in one data file.** The installer refuses a
   hash mismatch. Never `latest`, never an action or install script by a moving tag, never a package manager's "newest".
   Take a release a few days after it appears (the same cooldown as for dependencies).
6. **One allowlist with an owner, a reason and an expiry of at most 90 days.** An expired entry fails the run. It is
   translated into each tool's own ignore format at run time so the tool enforces the expiry too. No private data in
   it, and no entry for secrets.
7. **One backlog item per subject, never two.** A repo with findings gets one item (secret hits raise its priority);
   the next run files nothing while it is open. Deleting the item is how a fix is recorded.
8. **Network checks run on a timer, not in the commit hook.** The hook stays offline and fast (staged-change secret
   scan, workflow lint); the weekly timer runs the history and dependency scans. A new timer is installed disabled and
   enabled by the maintainer.

## The tools

| Area | Tool | Notes |
| --- | --- | --- |
| Dependencies | osv-scanner | Apache-2.0. Reads `uv.lock`, `pnpm-lock.yaml`, `Cargo.lock` and others; `[[IgnoredVulns]]` with `ignoreUntil` and `reason`. Sends package names and versions to osv.dev, never code. Exit 1 = findings, 128 = no packages (so `n/a`). |
| Workflows | zizmor | Static audit of GitHub Actions; run `--no-online-audits` so nothing leaves the machine. |
| Secrets | gitleaks | Whole history with `--redact`. A repo's own `.gitleaks.toml` is honoured. |
| Rust | cargo-deny | Already in the Rust hooks; not duplicated here. |
| Containers, compose, IaC | trivy `config` | Planned: a pinned binary only, see "Why". |
| systemd | `systemd-analyze security` | Planned for HQ's own units; not yet run on `--user` units. |
| Public repos | OpenSSF Scorecard | Planned, public repos only. |

Skipped on purpose: trufflehog (gitleaks covers the need), cargo-audit (cargo-deny reads the same database), young
agent-config scanners (a few own rules from [agent-security.md](agent-security.md) are cheaper), ZAP as a standing check.

## How to apply it

- **Registry-driven.** The repo list comes from the registry; what to scan comes from detection (tracked lockfiles,
  `.github/workflows`). A new repo is covered by registering it.
- **Hits are judged by the repo's session.** The wrapper reports; the repo's own rules decide the fix. A vulnerability
  with no fix yet becomes an allowlist entry with a reason and a date, not a silenced scanner.
- **Read the history scan with the repo's fixtures in mind.** Recorded API responses and generated test data trigger
  generic key rules by the hundred. Check that the values are fake before acting, and add the repo's own allowlist
  for the file path (the tool's path allowlist, not a per-secret one) once confirmed. Never skip the scan.
- **Keep the pins current on purpose.** `sec-check tools` shows the pinned and installed versions; bump after the
  cooldown, compare the hash with the release's checksum file, and keep the old pin if the new release is under a
  week old.
- **Tests for the logic, a real run for the claim.** The allowlist rules, the line format and the not-checked path are
  unit-tested; "works" is claimed only after a run on real repos, named in the report.

## Why

- Scanners disagree and drift; a thin wrapper keeps the policy (what counts as ok, who may ignore what, for how long)
  in one place while the tools stay replaceable.
- "Not checked" printed as "ok" is the usual failure of home-made security scripts: the tool is missing on a new
  machine and the dashboard stays green. Treating it as a failure costs one line and removes the blind spot.
- Security tools are a supply-chain target themselves. In March 2026 the Trivy GitHub Action tags and some releases
  were compromised (CVE-2026-33634, advisory GHSA-69fq-xp46-6x23), so a tool pulled by tag or `latest` would have run
  attacker code with the repo's secrets. A hash-pinned binary installed from a file in the repo cannot change under us.
- A permanent ignore is a finding nobody reads again. An expiry of at most 90 days turns "accepted risk" into a
  question that comes back.
- One ticket per repo keeps the backlog readable; a ticket per finding buries the one that matters (a leaked secret).

## Sources

- osv-scanner supported lockfiles and configuration (`[[IgnoredVulns]]`, `ignoreUntil`): <https://google.github.io/osv-scanner/supported-languages-and-lockfiles/>,
  <https://google.github.io/osv-scanner/configuration/> (read 2026-11-04)
- Trivy compromise: GitHub advisory <https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23>
  and NVD <https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2026-33634> (CVE-2026-33634; read 2026-11-04)
- zizmor: <https://docs.zizmor.sh/> (audits, `--no-online-audits`); gitleaks: <https://github.com/gitleaks/gitleaks>
  (history scan, redaction, `.gitleaks.toml`); release assets and checksums read through the GitHub releases API of
  each project on 2026-11-04
- Observed 2026-11-04 on HQ's registry repos with the tools above; counts and what stayed unverified are in
  [docs/research/security-tooling-2026-11.md](https://github.com/evocaslab/hq/blob/main/docs/research/security-tooling-2026-11.md).
