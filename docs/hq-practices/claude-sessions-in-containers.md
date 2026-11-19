---
summary: "Which Claude sessions and helpers belong in containers (unattended workers and gates, not desktop sessions), how they reach each other, GUI, the manager pattern and Spot VMs"
group: agents
tags: [agents, docker, machine]
applies-to: [all]
reviewed: 2026-11-09
status: draft
---

# Claude sessions in containers

A container helps with **reboots, orphans and memory**, not with **sleep**. Rules for deciding what to containerise;
reasoning, numbers and options: [design note](https://github.com/evocaslab/hq/blob/main/docs/design/claude-with-containers.md). Container basics:
[containers-by-default.md](https://github.com/evocaslab/hq/blob/main/practices/containers-by-default.md); recovery: [heal-after-interruption.md](heal-after-interruption.md).

## Rule

| Piece | Where | Why |
| --- | --- | --- |
| Desktop-app session, supervisor (`/fleet`), the HQ web app, queue, sampler | **host** | the app owns the process; they read `/proc`, cgroups, tmux and worktrees |
| tmux autopilot / branch-only code **worker**, unattended | **container** when adopted (one per session) | `--memory` and `--pids-limit` per session; `docker stop` ends the whole tree; `--restart unless-stopped` after a reboot |
| Gate, test stack, headless browser | **container**, started by a prepared command | a limit per run; nothing left behind |
| A session that must start containers | **host** (or a narrow manager command) | a mounted `docker.sock` is root on the host |

The supervisor may move off the laptop to the always-on hub VM, as a plain tmux session with Remote Control and no
work of its own; it manages workers and Spot VMs through a narrow command and scoped tokens (open design:
[design note](https://github.com/evocaslab/hq/blob/main/docs/design/claude-with-containers.md#supervisor-on-the-stable-vm-maintainer-note-r-2611091242-356d)).

The laptop prototype ran ([result](https://github.com/evocaslab/hq/blob/main/docs/research/2026-11-09-container-worker-prototype-result.md): conditional go for
unattended `claude -p` workers, after the firewall and hook fixes below); the VM run and the drill remain
(`B-261108-container-worker-prototype`). `scripts/claude/wip-push.sh` is the timed WIP push.

## Rules for a containerised session

- **Sleep is not solved.** A suspend freezes host kernel and containers alike; the API link drops either way. Heal as
  before.
- **A worker's state is the branch and the lease, not the host.** Sleep, a move to another network and a power cut all
  end a laptop session without warning: push a `wip/` branch on a timer, keep a heartbeat on the lease, and let the
  supervisor release an expired lease and restart from the branch tip. Address machines by tailnet name. On battery
  follow [power-saving-mode](https://github.com/evocaslab/hq/blob/main/practices/power-saving-mode.md). Table and shutdown order:
  [design](https://github.com/evocaslab/hq/blob/main/docs/design/claude-with-containers.md#power-cuts-sleep-and-moving-the-laptop-maintainer-note-2026-11-09).
- **Limits on every run:** `--memory`, `--pids-limit`, `--cpus`, `--restart unless-stopped`, log rotation
  (`--log-opt max-size`, `max-file`), `--read-only`, `--cap-drop ALL`, `no-new-privileges`, non-root user (the
  bypass flag is refused as root).
- **Mounts:** the worktree read-write at the same absolute path as on the host (hook paths are absolute), HQ checkout
  and `~/.local/state/hq` only where the role needs them, credentials as a scoped token or a per-worker config volume
  (`CLAUDE_CONFIG_DIR`), never `~/.ssh` or the host `~/.claude`. A bypass-mode session can exfiltrate anything inside
  the container, so mount only what a stolen token may lose.
- **Firewall at start, not after:** the reference container applies its firewall in `postStartCommand`; the rules die
  with the container's network namespace, so after `docker restart` or a restart-policy start the session ran with
  none (measured). Run it from the entrypoint and refuse to start `claude` without it.
- **Hooks need their runtime:** `uv run --script` hooks fail in a bare image (no `uv`, PyPI blocked); bake `uv` and the
  warm script cache into the image. An `--env-file` value must have no quotes (a quoted token gives a 401).
- **Reaching it:** a container session and a host session cannot `SendMessage` each other; the supervisor reaches a
  worker through Remote Control (refused with a token; needs an interactive `claude auth login`), the backlog/ledger files and hand-back files, and
  locally `docker exec -it <c> tmux attach`.
- **No raw Docker socket.** Give a session that must start containers one of: a narrow prepared command on the host
  (preferred: start/stop/logs/status, fixed limits, an audit log), a socket proxy that allows only the needed API calls
  (HTTP verbs and path groups, no body filtering), or its own rootless daemon with the nested-container cost.
- **GUI:** headless first (Chrome/Playwright need no display; Xvfb for the rest); a human view through Xvfb + VNC +
  noVNC bound to loopback or the tailnet with a password; the host Wayland/X socket only for a trusted app, because a
  shared X socket allows keylogging and screen capture.
- **Caches:** one `CARGO_HOME` and uv/npm cache volume shared (brief package-cache lock waits only); one target
  directory **per worktree**, never shared: cargo serialises builds on a shared target dir.
- **Spot VM:** a session there resumes from git (pushed branch plus a copied ledger), never from memory. Pattern: a
  prepared custom image, a separate persistent data disk (a cache), a timed WIP push plus a push on the notice. GCP's
  notice is 0 s by default (120 s optional, shutdown script best effort 30 s); AWS gives 2 minutes. A fresh launch
  creates compute plus a new boot disk from the image; STOP/stop keeps disks, DELETE/terminate does not.
- **Version and settings are files in git.** A container session does not inherit `~/.claude/settings.json`: the context
  window and `autoCompactWindow` live in `deploy/claude/settings.json` (copied to the session's config directory), and
  the Claude Code version is pinned in `deploy/claude/claude-code.version`, which `just setup-outdated --only claude`
  compares with the latest npm release. The image that installs the pin is `B-261114-claude-session-image`.
- **Checkpoint/restore** (`docker checkpoint`, CRIU) is experimental and refuses containers with a terminal: not used.

## Sources

Read 2026-11-09 unless noted; re-check in 3 months (flags and docs move).

- Claude Code dev containers (credentials, volumes, bypass flag, exfiltration warning, Remote Control and non-essential
  traffic): <https://code.claude.com/docs/en/devcontainer>
- Docker restart policies: <https://docs.docker.com/engine/containers/start-containers-automatically/>
- Docker rootless (linger, subuid): <https://docs.docker.com/engine/security/rootless/>
- Docker socket proxy: <https://github.com/Tecnativa/docker-socket-proxy>
- Docker checkpoint (experimental, CRIU): <https://docs.docker.com/reference/cli/docker/checkpoint/>
- x11docker (host X socket risks, nested servers): <https://github.com/mviereck/x11docker>
- noVNC: <https://github.com/novnc/noVNC>
- Compute Engine Spot VMs: <https://docs.cloud.google.com/compute/docs/instances/spot>
- GCP Spot create/use (preempted metadata, 120 s notice, shutdown script): <https://docs.cloud.google.com/compute/docs/instances/create-use-spot>
- GCP snapshots: <https://docs.cloud.google.com/compute/docs/disks/snapshots>
- AWS Spot interruptions, notices, behavior: <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-instance-termination-notices.html>,
  <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/interruption-behavior.html>; pricing page (no figures):
  <https://aws.amazon.com/ec2/spot/pricing/>
- Cargo build cache locking: not opened (the page is a redirect stub); the lock behaviour is from memory (weak).
