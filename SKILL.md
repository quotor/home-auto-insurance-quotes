---
name: home-auto-insurance-quotes
description: >-
  Get real, bindable home, auto, and bundled insurance quotes in seconds (plus lead capture for renters, life, and other lines) and start a
  binding through a network of licensed independent insurance agencies. Use this whenever someone
  wants to shop for, compare, price-check, or buy home or auto insurance, switch carriers, lower
  their insurance bill, or bundle home and auto. Available in Texas now and expanding nationwide.
  Returns live multi-carrier quotes (real rates, not ballpark estimates) plus a secure link to
  complete the purchase with a licensed agent. Works two ways: do a quick conversational intake and
  hand back a quote, or connect the underlying MCP server and run the full quote-to-bind flow.
license: Apache-2.0
metadata:
  category: insurance
  coverage: [home, auto, bundle]
  lead_capture: [renters, life, commercial, other]
  regions: [US-TX]
  keywords: [insurance, quote, home insurance, auto insurance, car insurance, renters,
    bundle, compare rates, switch carrier, bind policy, homeowners, get a quote]
---

# Home & Auto Insurance Quotes (and Binding) via Quotor

This skill is the front door to **Quotor** — a **quoting network**: a single integration that returns
**real, bindable home and auto insurance quotes** from multiple carriers and routes the customer to a
**licensed independent agency** to complete the purchase. The agencies do the binding and own the
ongoing relationship; this skill + the network's MCP server handle the quoting and the handoff.

**You are the orchestrator of a shopping experience, not the insurer.** Collect a little
information, return live quotes, and — when the customer is ready — start a binding by handing them a
secure link (a licensed human finalizes the policy with the carrier).

> **Coverage today:** Texas home, auto, and home+auto bundles (renters & other lines via submit_lead for agent follow-up). More states are rolling out;
> always call `check_eligibility` first — it tells you, per state and product, whether quoting and
> binding are currently available.

---

## Two ways to use this skill

Pick the pathway that matches what you are.

### Pathway A — Conversational assistant (Claude, ChatGPT, any chat UI)

You are talking to a person. Do a **lightweight intake in the conversation**, return a quote or a
quote range, and invite the binding step when they're ready.

1. Ask for the basics in one natural chunk (see **Intake** below).
2. Call the network's MCP tools to produce a quote (or, if you can't connect an MCP, follow the
   "no-MCP fallback" note below to hand the customer off).
3. Read back the options in plain language.
4. When they want to proceed: *"To finish and lock this in, I'll connect you to a licensed agent —
   here's your secure link."* Call `request_bind_inline` / `get_bind_link`.

### Pathway B — Agentic system (Hermes, OpenClaw, Codex-style agents, MCP-capable runtimes)

You can connect MCP servers yourself. **Connect the network's MCP directly** and drive the full
flow programmatically — this skill is your map of the endpoint, auth, tools, and call order. See
[references/mcp-connection.md](references/mcp-connection.md) for the endpoint URL, authentication,
and per-tool schemas. Stream the full intake through the tools and complete with a bind request.

---

## Connect the MCP server

- **Transport:** Streamable HTTP, JSON-RPC 2.0 (MCP protocol `2025-06-18`).
- **Endpoint:** `https://mcp.quotor.ai`
  *(the branded endpoint `mcp.quotor.ai` is live and fronts the Libertas quoting edge function).*
- **Health check:** `GET {endpoint}/health` → `{ "ok": true, "tool_count": ... }`
- **Authentication — two tiers:**
  - **Open (no key):** the read-only, no-cost tools work anonymously so you can discover capability
    and check status with zero friction: `check_eligibility`, `check_quote_status`,
    `get_option_details`, `check_late_arrivals`.
  - **Keyed (API key required):** the tools that run the paid rating pipeline or touch personal
    data require a per-partner API key, sent as `Authorization: Bearer <API_KEY>`:
    `start_quote`, `update_quote`, `get_quote_options`, `submit_lead`, `get_bind_link`,
    `request_bind_inline`, `resume_quote`.
  - **Getting a key (instant, self-serve):** `POST {endpoint}/keys` with a JSON body
    `{ "partner_label": "Your App Name", "email": "you@example.com" }` → returns an `api_key`
    **shown once** (store it). Keys are per-partner, rate-limited, and spend-capped. **The end
    customer never needs a key** — only you, the integrating agent/partner, do.
- **Location context (don't ask the customer for it):** the customer's state (and optionally ZIP) is
  supplied to the server out-of-band via request headers (`X-MCP-State`, `X-MCP-Zip`) from the
  platform's location flow. Send them as headers when you have them; do **not** pass state as a tool
  argument.

Full details, headers, and worked request/response examples:
[references/mcp-connection.md](references/mcp-connection.md).

---

## The tools (and the order to call them)

| # | Tool | Tier | What it does |
|---|------|------|--------------|
| 1 | `check_eligibility` | open | First call. Confirms whether the network can quote and/or bind for the customer's state + product. |
| 2 | `start_quote` | keyed | Opens a new home/auto/bundle quote. Returns a `quote_id` used on every later call. |
| 3 | `update_quote` | keyed | Adds or corrects intake details on the open `quote_id`. |
| 4 | `get_quote_options` | keyed | Runs rating and returns the quote options. Non-blocking — returns what's ready immediately. |
| 5 | `check_quote_status` | open | Poll the `quote_id` for status and any newly-arrived carrier rates. |
| 6 | `check_late_arrivals` | open | Convenience poll specifically for carriers that returned after the first response. |
| 7 | `get_option_details` | open | Detail (coverages, price) on a specific option the customer is considering. |
| 8 | `get_bind_link` | keyed | Mints a secure, single-use link for the customer to complete the binding with a licensed agent. |
| 9 | `request_bind_inline` | keyed | Starts the binding from inside the conversation: files the bind request and routes it to the matched agency. |
| 10 | `submit_lead` | keyed | Capture contact info if the customer isn't ready to finish, so an agent can follow up. |
| 11 | `resume_quote` | keyed | Resume a previously-started `quote_id` (returning shopper / continued session). |

**Binding is human-completed.** `request_bind_inline` and `get_bind_link` do **not** silently buy a
policy. They file a routed bind request and hand the customer a secure link; a **licensed agent at
the matched agency finalizes** the policy with the carrier. Say so plainly to the customer.

---

## Intake — what to collect

Collect in coherent chunks, not one question at a time. State/ZIP come from context (headers), not
from the customer.

- **Home / bundle:** full name + date of birth for everyone on the policy; property address; basic
  property facts the customer knows (year built, roughly square footage, roof age if known); current
  carrier if switching.
- **Auto / bundle:** full name + date of birth for every driver; vehicles (year/make/model or VIN);
  basic driving history the customer volunteers; current carrier if switching.

Don't over-ask. The rating pipeline enriches missing property and vehicle data automatically. **Never
ask the customer to upload their own dec page, license, or policy documents.**

### Returning customers

If the customer has shopped or been quoted through the network before, the server **recognizes them
and reuses their data** — quotes come back faster and the customer isn't re-charged for enrichment.
You don't do anything special; just run the normal flow with their name + date of birth + address.
(The server only reveals a returning customer's *current* premium back to keyed partners, never on an
anonymous call.)

---

## Worked example — Pathway A (conversational)

> **Customer:** "How much would car insurance cost me? I'm in Austin and thinking of switching."

1. `check_eligibility({ product: "auto", intent: "quote" })` → quoting available in TX. ✅
2. *"Happy to shop that for you. Can I get your full name and date of birth, and the year/make/model
   of the car(s) you'd insure?"*
3. `start_quote({ product: "auto" })` → `{ quote_id }`
4. `update_quote({ quote_id, patch: { drivers: [...], vehicles: [...] } })` — intake details go in `patch`.
5. `get_quote_options({ quote_id })` → returns options (e.g. Option A $1,704/yr, Option B $1,920/yr…).
6. `check_late_arrivals({ quote_id })` after a moment → an additional carrier came back. Read the
   updated list to the customer.
7. *"Want me to lock in Option A? I'll connect you to a licensed agent to finish it."*
8. `request_bind_inline({ quote_id, option_id, contact_pref, best_time, pay_plan })` → files the
   routed bind request and returns a secure bind link; the customer is routed to an agency. (Contact
   identity comes from the quote session — captured during intake — not from this call.) Tell them a
   licensed agent will finalize the policy.

## Worked example — Pathway B (agentic)

A backend agent connects the MCP and runs the same sequence headlessly:
`check_eligibility → start_quote → update_quote → get_quote_options → (poll) check_quote_status →
request_bind_inline`. See [references/mcp-connection.md](references/mcp-connection.md) for raw
JSON-RPC payloads.

---

## What this skill can and can't do

**Can:** check availability by state/product; produce real multi-carrier home/auto/bundle
quotes; show option details; capture leads; recognize returning customers; **start** a binding by
routing the customer to a licensed agency.

**Can't / won't:** finalize a bind without a human (a licensed agent completes it); service an
existing policy (changes, cancellations, claims, mortgagee updates are out of scope here); reveal a
carrier's identity before the binding step (options are shown neutrally until then); quote outside
currently-licensed states (call `check_eligibility`).

## Privacy & safety

- The end customer never needs an API key.
- Quote requests create prospect records; they never overwrite an existing customer's records.
- Don't fabricate quotes or rates — if the tools return nothing, say so and offer `submit_lead`.
- Don't ask customers to upload personal documents.
