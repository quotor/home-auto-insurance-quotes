# home-auto-insurance-quotes

**Quotor — an agent skill that gets real home & auto insurance quotes, and starts a binding, through
a network of licensed independent agencies.**

This repository packages a portable [Agent Skill](https://agentskills.io) that any AI agent or
assistant (Claude, ChatGPT, Hermes, OpenClaw, Codex-style agents, and other MCP-capable runtimes)
can pull in to quote and help bind home, auto, and bundled insurance. The skill is the
**front door**; a hardened MCP server is the infrastructure behind it.

> **Coverage:** Texas today (home, auto, home+auto bundles; renters & other lines via lead capture), expanding nationally. The skill
> always checks eligibility per state and product before quoting.

---

## What it does

- **Real quotes, not estimates.** Returns live multi-carrier rates for the customer's state and
  product.
- **Two integration styles:**
  - *Conversational* — an assistant does a quick intake in chat and reads back quotes.
  - *Agentic* — a system connects the MCP server directly and runs the full quote-to-bind flow.
- **Quote → bind handoff.** When the customer is ready, the skill starts a binding by routing them to
  a **licensed agency** with a secure link. A licensed human finalizes the policy — the network
  orchestrates the shopping; the agency owns the relationship.

## How it's structured

```
home-auto-insurance-quotes/
├── SKILL.md                      # the skill itself (agentskills.io spec)
├── README.md                     # this file (for humans browsing the repo)
└── references/
    └── mcp-connection.md         # endpoint, auth (none required), tool schemas, raw examples
```

- **[SKILL.md](SKILL.md)** is what agents load. It documents the two usage pathways, the tool set and
  call order, intake guidance, worked examples, and the MCP connection details.
- **[references/mcp-connection.md](references/mcp-connection.md)** is the technical reference for the
  MCP endpoint: URL, authentication, context headers, per-tool schemas, and copy-paste JSON-RPC
  examples.

## Connecting to the MCP

- **Endpoint:** `https://mcp.quotor.ai` (Streamable HTTP,
  JSON-RPC 2.0).
- **Auth: none required.** All tools work anonymously — connect and quote, no sign-up, no key
  (anonymous traffic is rate-limited and spend-capped server-side). **End customers never need a
  key.**
- **Optional partner key (instant, self-serve):** integrating at volume? `POST <endpoint>/keys`
  with `{ "partner_label": "...", "email": "..." }` → returns an `api_key` shown once. Send it as
  `Authorization: Bearer <api_key>` to give your integration its own rate/spend caps and
  attribution instead of sharing the anonymous pool.

**Add it to your MCP client** (claude.ai custom connectors, Claude Desktop, Cursor, VS Code, or any
Streamable-HTTP MCP client) — no headers needed:

```json
{
  "mcpServers": {
    "quotor": {
      "type": "http",
      "url": "https://mcp.quotor.ai"
    }
  }
}
```

The machine-readable manifest is [`server.json`](server.json) (official MCP registry schema) — that's
what client tooling and directory crawlers read.

See [references/mcp-connection.md](references/mcp-connection.md) for the full handshake.

## Security & privacy posture

- **Quote-only scope.** The MCP cannot reach carrier portal credentials, cannot service or cancel
  existing policies, and cannot finalize a bind without a licensed human. A bind request is a routed
  handoff, not an automated purchase.
- **Rate- and spend-capped:** anonymous traffic draws per-IP and per-quote budgets under a hard
  daily spend ceiling; optional partner keys carry their own caps — no single caller can run up cost.
- **Every call logged**; anomalous usage (auth-failure spikes, off-hours surges) is monitored.
- **Customer data is protected:** quotes create prospect records and never overwrite existing
  customers; a returning customer's current premium is only ever returned to an authenticated
  partner, never on an anonymous call.

## Status

Live. The MCP is deployed in production — rate and spend caps and full audit logging are in force;
API keys are optional (partner attribution + dedicated caps). Coverage is Texas today (home, auto,
bundles; other lines via lead capture), expanding nationally; always call `check_eligibility` first.

## License

Apache-2.0.
