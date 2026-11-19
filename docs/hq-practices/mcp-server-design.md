---
summary: "Designing and running an MCP server: tools vs resources vs prompts, small typed tools, output caps, authentication, logging, tests, injection"
group: security
tags: [agents, security]
applies-to: [all]
reviewed: 2026-11-10
status: draft
---

# MCP server design and operation

An MCP server gives an agent typed access to data and actions. The agent is an unreliable, injectable caller, so the
server is a trust boundary: small tools, bounded output, narrow access, a log. This page is for building or changing one
(HQ's `hq-mcp` is the worked example). How an agent uses a server safely is in [agent-security.md](agent-security.md);
when to pick MCP over the browser is in [agent-access-to-apps.md](agent-access-to-apps.md#mcp-or-browser).

Spec facts below are for revision 2026-07-28 ("latest" on 2026-11-10); revisions change fast, so re-read the linked page
before relying on a detail.

## Tools, resources, prompts

- The three differ by **who decides to use them**: tools are model-controlled, resources application-driven, prompts
  user-controlled [1], [10], [11]. Put an action or a query with arguments in a tool; read-only context a client can
  attach or browse in a resource; a reusable, user-started workflow in a prompt.
- A tool is the default, because the model can call it without the user acting. In Claude Code resources are attached
  with `@` mentions and prompts appear as slash commands [6], so what a server offers only as resources or prompts
  reaches the model through the user's action. HQ exposes tools only (`server.rs:316-320`).
- A **context provider** (rules, a digest of state) is read-only data with a size cap. Offer it as a tool so the model
  can pull it, or as server `instructions`, which Claude Code cuts at 4,096 characters [6]: keep that text to what the
  model needs before it picks a tool, and put the rest behind a call.

## Small typed tools

- `inputSchema` is required; give every argument a type, a description and bounds. `outputSchema` and
  `structuredContent` are optional; if used, also return the text form [1].
- Few tools that match whole tasks beat many thin API wrappers. Prefix names by area (`backlog_list`, `backlog_get`),
  and make list order deterministic. Names (spec SHOULD): 1-128 characters of `A-Za-z0-9_-.`, unique per server [1].
- Return what the model needs next: names over opaque ids, a concise mode by default and an optional detailed one
  (`response_format`) [4]. Write descriptions as for a new colleague; small wording changes move tool accuracy [4].
- Tool annotations (read-only, destructive, open-world) help clients ask for confirmation, but clients must treat them as
  untrusted unless the server is trusted [1]. Set them honestly, and never rely on them for security.
- Put write tools behind a separate switch or scope, off by default (HQ: `HQ_MCP_TOOLS`, `Role` in `http.rs`).

## Errors

- Two kinds: a **protocol error** (JSON-RPC, e.g. unknown tool) and a **tool execution error** (`isError: true` in a
  normal result) [1]. Use the second for anything the model can fix: say what was wrong, what a valid call looks like,
  and what to try next, not a stack trace [1], [4].
- Validate every input on the server; the schema is a hint to the model, not a guarantee [1].

## Output size

- **Cap every response.** Truncate with a visible hint that says how to narrow the call (a filter, `limit`, an id) [4].
  Claude Code warns at 10,000 tokens and caps at 25,000 by default (`MAX_MCP_OUTPUT_TOKENS`) [6]; a tool that must
  return more can raise its own limit with `_meta["anthropic/maxResultSizeChars"]` (ceiling 500,000) [6].
  A get tool for a long document returns the heading outline and the first section when the text is over the cap, and
  takes a `section` argument for the rest (HQ: `practices_get`), so a read costs one section, not the whole page.
- **Paginate lists.** The server chooses the page size and returns an opaque cursor; a bad cursor is error -32602 [7].
  A small `limit` argument with a hard clamp is the cheap form (HQ: `limit` 1-50, `MAX_CHARS` 20,000).
- Keep tool descriptions short: Claude Code cuts them at 4,096 characters (16,384 for deferred tools) and loads tool
  definitions through tool search by default [6], so a name and first sentence must say what the tool is for.

## Transport and authentication

- **stdio** for a local single-user server: credentials come from the environment, not the protocol [2].
- **Streamable HTTP** for a shared server. The spec requires validating the `Origin` header (reject with 403) and
  recommends binding to localhost and authenticating [3]. Bind to loopback or a private (tailnet) address, never
  `0.0.0.0`, and refuse to start otherwise.
- Authorization is optional in the spec. Over HTTP the server is an OAuth 2.1 resource server: it publishes
  protected-resource metadata (RFC 9728), validates the token **audience** (RFC 8707), answers 401 for a missing
  or invalid token and (SHOULD) 403 `insufficient_scope` with a `WWW-Authenticate` challenge for a missing scope, and **never passes the
  client's token on** to another API [2], [5]. Tokens go in the `Authorization` header, never the query string.
- For a private server a pragmatic bearer token is acceptable (see [agent-access-to-apps.md](agent-access-to-apps.md)):
  store only a hash, compare in constant time, show the token once, one token per client and machine, a role per
  token, an expiry, and a way to revoke one without touching the rest. (These token habits are practice, not spec.)
- Ask for the minimal scope first and step up when a call needs more [5]. Bind any server-side handle (a lease, a
  session id) to the authenticated user, so a guessed handle is useless; possessing a handle is not authentication [1], [5].
- Rate-limit per client; the spec lists rate limits next to input validation and output sanitizing as server MUSTs [1].

### Agent identity

When many agents share one machine or one human account, keep three things apart (practice, not spec):

- **Authority** is the token: its role and its expiry. It decides what a call may do and is the only thing the
  server trusts.
- **Address** is a stable `agent_id` (a UUIDv7 minted once per agent and bound to its token). The server takes it
  from the authenticated token, never from a request argument, and uses it for leases, inboxes, rate limits and
  log lines. A machine name is not an address: two agents of one machine would share it.
- **Label** is the human handle (`repo/adjective-noun`, a card `name`). It is for people, may repeat or change,
  and grants nothing.

One token per agent makes revoking one agent leave the others working. Keep tokens without an agent id valid
until they expire so the change rolls out without a flag day.

## Logging and call metrics

- Log one line per call: time, tool, client, arguments, ok or error, duration, output size, machine. This is the data
  for finding dead tools, slow tools and tools that always truncate [4]. Keep it append-only and local.
- **Redact before writing**, both the arguments and the result: logs must not hold secrets or personal data [8]. Cap
  or rotate the file.
- The MCP `logging` utility is deprecated in 2026-07-28; use stderr for stdio servers and OpenTelemetry otherwise [8].
- Evaluate tools against realistic multi-step tasks and read the transcripts and metrics, not only the pass rate [4].

## Redaction and untrusted data

- A server returns data that may come from a hostile source (an issue, a web page, a file). Mask known secret shapes in
  every result and error, and mark returned text as data (HQ appends a banner). A banner lowers the risk and does not
  remove it: the client decides what the model may then do ([agent-security.md](agent-security.md)).
- The spec lists output sanitizing among the server's duties [1]. Do not return secrets, tokens or environment dumps
  through a tool, even to an authenticated caller.

## Tests

- Unit-test the pure parts: argument parsing and clamps, redaction patterns, truncation, role checks.
- Test the **protocol path** in process: start the server over an in-process or child-process transport, call
  `tools/list` and each tool, and assert on the JSON a client sees (HQ: `crates/hq-mcp/tests/stdio.rs`, `http.rs`;
  rmcp ships the transports for it [9]).
- Test the **security rules** as cases: no token gives 401, a read token cannot call a write tool, a non-private bind
  is refused, a secret is masked, a bad `Origin` is refused, a call over the rate limit is refused.
- Try the server by hand with the MCP Inspector before wiring a client [12].

## Deploy and monitor

- Run an HTTP server as a service (systemd user unit) with its settings in the environment; give it a `/health`
  endpoint that checks something real (the data root is readable), not only that the process is up.
- Watch the call log (error rate, slow calls, truncations), rejected-login counts and the service state. A client that
  loses the connection retries on its own (Claude Code reconnects up to five times [6]), so an outage shows up as
  latency first.
- Version the tool set. Renaming or removing a tool breaks prompts and skills that name it: add, deprecate, remove.

## HQ's hq-mcp against this page

| Rule | State | Where |
| --- | --- | --- |
| Typed tools, annotations, actionable `isError` | follows | `server.rs:111-131`, `tools.rs` |
| Output cap with hint, `limit` clamp | follows | `support.rs:13-51`, `tools.rs:358,1422` |
| Pagination with cursor | partly (limit only) | `server.rs:322-331` |
| Loopback or tailnet bind only | follows | `http.rs:203-236` |
| Hashed, constant-time bearer tokens, roles | follows | `http.rs:141-160` |
| `Origin` validated | missing (`allowed_origins` unset) | `http.rs:288-290` |
| Token expiry, revoke, agent id per token | follows (revoke applies on restart) | `http.rs`, `identity.rs` |
| 401/403 challenge with scope detail | partly (401 only) | `http.rs:253` |
| Rate limit per client | follows | `audit.rs`, `server.rs:240-247` |
| Call log with machine and sizes | follows | `audit.rs:59-64` |
| Call-log arguments redacted, file rotated | missing | `audit.rs:59,82` |
| Result and error redaction, data banner | follows | `redact.rs`, `server.rs:256,309` |
| Protocol and security tests | follows | `crates/hq-mcp/tests/` |
| Real `/health`, unit installed somewhere | partly | `http.rs:298`, `infra/services.yaml:99` |
| Remote snapshots fit the cap | tracked | `B-261109-mcp-http-snapshot-cap-and-second-machine` |
| Tool evaluation from the call log | missing | none |

## Sources

All opened 2026-11-10; primary sources. The Claude Code numbers are product behaviour and can change.

1. MCP specification, tools (isError, schemas, annotations, naming, security duties):
   <https://modelcontextprotocol.io/specification/latest/server/tools>
2. MCP specification, authorization (resource server, RFC 9728, RFC 8707, challenges, stdio credentials):
   <https://modelcontextprotocol.io/specification/latest/basic/authorization>
3. MCP specification, Streamable HTTP transport (Origin MUST and 403, localhost, authentication):
   <https://modelcontextprotocol.io/specification/latest/basic/transports/streamable-http>
4. Anthropic, Writing effective tools for agents: <https://www.anthropic.com/engineering/writing-tools-for-agents>
5. MCP security best practices (token passthrough, scope minimization, state handles):
   <https://modelcontextprotocol.io/specification/latest/basic/security_best_practices>
6. Claude Code, MCP (limits, resources, prompts, instructions, reconnect): <https://code.claude.com/docs/en/mcp>
7. MCP specification, pagination: <https://modelcontextprotocol.io/specification/latest/server/utilities/pagination>
8. MCP specification, logging (deprecated, no secrets):
   <https://modelcontextprotocol.io/specification/latest/server/utilities/logging>
9. rmcp 3.7.0, the Rust SDK (tool macros, Streamable HTTP service, allowed hosts and origins, transports):
   <https://docs.rs/rmcp/3.7.0/rmcp/>
10. MCP specification, resources: <https://modelcontextprotocol.io/specification/latest/server/resources>
11. MCP specification, prompts: <https://modelcontextprotocol.io/specification/latest/server/prompts>
12. MCP Inspector: <https://modelcontextprotocol.io/docs/tools/inspector>
