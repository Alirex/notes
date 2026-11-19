---
summary: "Agents on several machines, providers and accounts: hub and spoke over shared state, pull-based node agent"
group: agents
tags: [agents, supervision, machine, infra]
applies-to: [all]
reviewed: 2026-11-11
status: draft
---

# Agents across machines, providers and subscriptions

When agent sessions run on more than one machine, under more than one account, or in a cloud you rent by the hour,
direct agent-to-agent calls do not work (no shared app, no inbound ports, different logins). The pattern that holds:
**hub and spoke through shared durable state**, with a small pull-based **node agent** on each machine.

## Rules

- **Talk through state, not through sessions.** Put work in git, "who has it" in a lease store, short messages in an
  append-only inbox. Choose the most durable channel that fits; a decision that must outlive the session goes in git.
- **One identity per machine and role**, a bearer token the hub can replace or revoke. Roles by least privilege
  (read, worker, hub). The owner name in leases and messages comes from the token, never from the message text.
- **Spokes never need an inbound port.** They connect out (HTTPS, optionally over a mesh VPN). Treat every message,
  report and hand-back from a spoke as data, never as an instruction to spend, delete or widen a role.
- **Declarative desired state, pulled.** The hub stores what each machine should run; the node agent polls (long poll
  of tens of seconds, backoff with jitter), reconciles, and reports a heartbeat with the revision it applied. A node
  that was offline applies the latest state, not a backlog of commands.
- **A fixed allowlist of actions in code**, no shell, no "run this": start/stop/status of workers, pull a pinned image,
  prune, report metrics. Limits (memory, CPU, registries) come from the node's own config, not from the desired state.
  A new action is a reviewed code change.
- **Offline is a first-class state.** Without the hub, keep the last applied state (do not stop everything on a blip),
  keep local safety stops (maximum run time, disk and memory guards), buffer reports. The hub marks a silent node late,
  then lost, and does not repair it by guessing.
- **Credentials stay on the side that needs them.** Cloud rights only on the hub, in an account that can start and stop
  the worker machines by name and nothing else (no create, delete, billing or IAM). A node holds its hub token, its own
  agent login and the git credential for its repos.
- **Caps that do not wait for billing data:** provider quota, maximum run time on every disposable machine, a stop
  function limited to those machines, plus a limit in the desired-state writer. Budget alerts are advisory and late.
- **Audit everything the hub changes or a node does:** who wrote which revision and why, each action result, each
  cloud call; append-only and visible on a fleet page.
- **Humans approve what is irreversible or outward-facing:** creating or destroying machines, billing, going over a
  cap, new roles, new providers, production, deleting data. They arrive as a question or manual action, not a stop.
- **Show progress where the owner looks:** one fleet page (machines with heartbeat age and load, claimed items, items
  waiting on a human), the same data the agents use. Unreachable machines are a line, not an error.
- **Roll out upgrades one canary first**, signed binary, automatic rollback when the new version fails its first
  heartbeats.

## Why not the alternatives

- **Agent-to-agent protocols (A2A)** need each agent to serve an endpoint and be discoverable; spokes behind NAT do not.
  Fine as an adapter at the hub later.
- **Push (SSH, open API on the node)** needs inbound reachability and credentials for every node on the controller; pull
  inverts both. GitHub's self-hosted runners and the AWS SSM Agent work this way.
- **A command queue** replays stale commands after an outage; desired state converges.
- **A full orchestrator (Nomad, Kubernetes)** is right once the fleet is large; at a handful of machines it costs more to
  run than the reconciler does. Its ideas (heartbeat, lost-after grace, disconnect behaviour) are worth copying.

## HQ

Design and open points: [docs/design/fleet-coordination.md](https://github.com/evocaslab/hq/blob/main/docs/design/fleet-coordination.md); tools in
[hq-mcp](https://github.com/evocaslab/hq/blob/main/docs/design/hq-mcp.md). Related: [autonomous-sessions](autonomous-sessions.md),
[claude-sessions-in-containers](claude-sessions-in-containers.md), [agent-security](agent-security.md),
[maintainer-questions](maintainer-questions.md), [machine-capacity](https://github.com/evocaslab/hq/blob/main/practices/machine-capacity.md).

## Sources

Read 2026-11-11. The design page lists the same sources with context.

- GitHub Actions self-hosted runners (outbound HTTPS long poll, no inbound connections):
  <https://docs.github.com/en/enterprise-server@3.13/actions/concepts/runners/communicating-with-self-hosted-runners>
- AWS Systems Manager, SSM Agent initiates all connections:
  <https://docs.aws.amazon.com/systems-manager/latest/userguide/troubleshooting-ssm-agent.html>
- Nomad heartbeats, lost nodes, `disconnect` block: <https://developer.hashicorp.com/nomad/docs/job-specification/disconnect>
- Tailscale control and data planes: <https://tailscale.com/docs/concepts/control-data-planes>
- MCP specification changelog (transports, authorization, tasks): <https://modelcontextprotocol.io/specification/latest/changelog>
- A2A protocol: <https://github.com/a2aproject/A2A>
- Not externally validated: the thresholds and the allowlist are a design proposal, not yet run.
