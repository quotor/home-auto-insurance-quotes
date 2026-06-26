# MCP connection reference

Operating details for connecting to the quoting-network MCP server and driving the quote-to-bind
flow programmatically (Pathway B). Pathway A (conversational) callers can use this too, but most of
what you need is in [../SKILL.md](../SKILL.md).

## Endpoint

| | |
|---|---|
| **URL** | `https://mcp.quotor.ai` |
| **Transport** | Streamable HTTP, JSON-RPC 2.0 |
| **MCP protocol** | `2025-06-18` |
| **Health** | `GET {URL}/health` → `{ "ok": true, "service": "...", "tool_count": N }` |
| **Branded domain** | `mcp.quotor.ai` (Quotor) is the live canonical endpoint. |

Standard MCP handshake applies: `initialize` → `tools/list` → `tools/call`.

## Authentication

Two tiers. Send the key (when required) as a bearer token:

```
Authorization: Bearer <YOUR_PARTNER_API_KEY>
```

| Tier | Tools | Key required |
|------|-------|--------------|
| **Open** (read-only, no cost) | `check_eligibility`, `check_quote_status`, `check_late_arrivals`, `get_option_details` | No |
| **Keyed** (paid rating / personal data) | `start_quote`, `update_quote`, `get_quote_options`, `submit_lead`, `get_bind_link`, `request_bind_inline`, `resume_quote` | Yes |

- Keys are **per-partner**, rate-limited, and spend-capped. Don't share or embed them in client-side
  code or public repos.
- **Get a key (instant, self-serve):** `POST {URL}/keys` with `{ "partner_label": "...", "email": "..." }`
  → `{ "api_key": "mcpk_…", "key_prefix": "mcpk_…" }`. The `api_key` is shown **once** — store it.
- The **end customer never needs a key.**

## Context headers (not tool arguments)

The customer's location is provided to the server out-of-band so you don't have to ask for it:

| Header | Value | Notes |
|--------|-------|-------|
| `X-MCP-State` | 2-letter US state (e.g. `TX`) | Drives eligibility + routing. Don't pass state as a tool arg. |
| `X-MCP-Zip` | 5-digit ZIP | Optional, improves accuracy. |
| `X-MCP-Platform` | `claude` \| `chatgpt` \| `libertas-web` \| `other` | Identifies the calling surface (telemetry). |

## Call sequence

```
check_eligibility            # open — confirm we can quote/bind this state+product
  └─ start_quote             # keyed — returns { quote_id }
       └─ update_quote       # keyed — add drivers/vehicles/property facts
            └─ get_quote_options   # keyed — runs rating, returns options now
                 ├─ check_quote_status / check_late_arrivals   # open — poll for stragglers
                 ├─ get_option_details                          # open — drill into one option
                 └─ request_bind_inline / get_bind_link         # keyed — start the binding
       └─ submit_lead        # keyed — if the customer drops off, capture for follow-up
       └─ resume_quote       # keyed — continue an earlier quote_id
```

## Tool argument summary

Authoritative schemas are returned by `tools/list` (each tool's `inputSchema`); every tool sets
`additionalProperties: false`, so unknown top-level keys are rejected. Summary (required fields in
**bold**):

- **`check_eligibility`** — `{ `**`product`**`: "auto"|"home"|"bundle", intent?: "quote"|"bind" }`
- **`start_quote`** — `{ `**`product`**`, referral_source? }` → `{ quote_id }`. *Personal/property/
  vehicle details are NOT passed here — send them via `update_quote.patch`.*
- **`update_quote`** — `{ `**`quote_id`**`, `**`patch`**`: { ...intake fields: applicants, drivers,
  vehicles, property facts... } }`. The `patch` object carries all the intake data.
- **`get_quote_options`** — `{ `**`quote_id`**`, max_options? }` → `{ status:"running", started_at, expected_seconds }` (NON-blocking — poll check_quote_status for the option cards)
- **`check_quote_status`** — `{ `**`quote_id`**`, max_options? }` → `{ status, options }`
- **`check_late_arrivals`** — `{ `**`quote_id`**` }` → `{ new_options }`
- **`get_option_details`** — `{ `**`quote_id`**`, `**`option_id`**` }` → `{ premium_annual, premium_monthly, coverages, deductibles, ... }`
- **`get_bind_link`** — `{ `**`quote_id`**`, `**`option_id`**` }` → `{ bind_url }` (single-use, tokenized)
- **`request_bind_inline`** — `{ `**`quote_id`**`, option_id? (or option_ids?), contact_pref?,
  best_time?, pay_plan?, customer_notes? }` → files a routed bind request + returns a bind link.
  *Contact identity comes from the quote session, not this call.*
- **`submit_lead`** — `{ `**`interest`**`, first_name?, last_name?, email?, phone?, quote_id?,
  preferred_contact_method?, preferred_contact_time?, notes? }`
- **`resume_quote`** — `{ `**`quote_id`**`, verification? }`

> Validate your arguments before sending. Top-level schemas reject unknown keys; the network is
> additionally tightening string-pattern validation (5-digit ZIP, 2-letter state, UUID `quote_id`,
> VIN charset) and value-checking the intake `patch`. Send clean, well-formed data.

## Raw JSON-RPC examples

**List tools (open):**
```bash
curl -s -X POST https://mcp.quotor.ai \
  -H "content-type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

**Check eligibility (open):**
```bash
curl -s -X POST https://mcp.quotor.ai \
  -H "content-type: application/json" -H "X-MCP-State: TX" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call",
       "params":{"name":"check_eligibility","arguments":{"product":"auto"}}}'
```

**Start a quote (keyed):**
```bash
curl -s -X POST https://mcp.quotor.ai \
  -H "content-type: application/json" -H "X-MCP-State: TX" \
  -H "Authorization: Bearer $PARTNER_API_KEY" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call",
       "params":{"name":"start_quote","arguments":{"product":"auto"}}}'
```

## Errors & limits

- **Auth:** missing/invalid key on a keyed tool → JSON-RPC error (auth required). Open tools never
  require a key.
- **Rate / spend limits:** per-key. Exceeding them returns a throttling error — back off and retry
  later; don't hammer.
- **Empty results:** rating may return zero options (out of appetite, unsupported state). Don't
  fabricate a quote — offer `submit_lead` so an agent can follow up.

## Scope boundary (for the security-minded)

This MCP is **quote-only**. It cannot reach carrier portal credentials, cannot service or cancel an
existing policy, and cannot finalize a bind without a licensed human. A bind request is a *routed
handoff*, not an automated purchase.
