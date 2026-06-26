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
    └── mcp-connection.md         # endpoint, auth tiers, tool schemas, raw examples
```

- **[SKILL.md](SKILL.md)** is what agents load. It documents the two usage pathways, the tool set and
  call order, intake guidance, worked examples, and the MCP connection details.
- **[references/mcp-connection.md](references/mcp-connection.md)** is the technical reference for the
  MCP endpoint: URL, authentication, context headers, per-tool schemas, and copy-paste JSON-RPC
  examples.

## Connecting to the MCP

- **Endpoint:** `https://mcp.quotor.ai` (Streamable HTTP,
  JSON-RPC 2.0).
- **Auth, two tiers:** read-only tools (eligibility, status, option details) work with no key; the
  tools that run the paid rating pipeline or touch personal data require a per-partner API key
  (`Authorization: Bearer <key>`). **End customers never need a key.**
- **Get a partner key (instant, self-serve):** `POST <endpoint>/keys` with `{ "partner_label": "...", "email": "..." }` → returns an `api_key` shown once.

See [references/mcp-connection.md](references/mcp-connection.md) for the full handshake.

## Security & privacy posture

- **Quote-only scope.** The MCP cannot reach carrier portal credentials, cannot service or cancel
  existing policies, and cannot finalize a bind without a licensed human. A bind request is a routed
  handoff, not an automated purchase.
- **Per-partner keys**, rate-limited and spend-capped, so no single integrator can run up cost.
- **Every call logged**; anomalous usage (auth-failure spikes, off-hours surges) is monitored.
- **Customer data is protected:** quotes create prospect records and never overwrite existing
  customers; a returning customer's current premium is only ever returned to an authenticated
  partner, never on an anonymous call.

## Status

Live. The MCP is deployed and enforced in production — per-partner API keys, rate and spend caps, and
full audit logging are in force. Coverage is Texas today (home, auto, bundles; other lines via lead
capture), expanding nationally; always call `check_eligibility` first.

## License

Apache-2.0.
