---
name: Connect an agent to the Leadspace MCP server
description: >-
  Add Leadspace as a remote MCP connector in Claude, ChatGPT, Claude Code or any
  MCP-compatible client, authenticate with OAuth, and work the enrich → find →
  reveal → intent flow without burning credits by accident.
api: mcp/leadspace-mcp.yml
endpoint: https://skprod.leadspace.com/mcp/v1
operations:
  - enrich_account
method: generated
source: https://www.leadspace.com/solutions/leadspace-mcp/claude
generated: '2026-08-13'
---

# Connect an agent to the Leadspace MCP server

Leadspace runs a hosted, remote MCP server. There is no package to install and
no local process to run — you point an MCP client at one HTTPS endpoint and sign
in.

**Endpoint:** `https://skprod.leadspace.com/mcp/v1`

This is a different product from the Leadspace v4 gateway API
(`apigw.leadspace.com`). A Program ID and perpetual bearer token authenticate the
gateway and will **not** work here; an MCP token will not work there. Pick the
surface that matches the job: MCP for conversational, single-record, per-user
work; the v4 gateway for contracted, bulk, system-to-system enrichment.

## 1. Get an account

Create a free Leadspace account with a work email. If you already use Leadspace
Sidekick, that same login works — do not create a second account.

The free **Starter** tier is real: 25 credits per day, no credit card.

## 2. Add the connector

**Claude (web, desktop):** Settings → Connectors → *Add custom connector*. Paste
`https://skprod.leadspace.com/mcp/v1`, **leave the OAuth client ID and secret
fields blank**, click Connect, and sign in with your Leadspace account. The
blank fields are correct — the client registers itself dynamically (RFC 7591)
against Leadspace's Auth0 tenant.

**Claude Code:**

```
claude mcp add --transport http leadspace https://skprod.leadspace.com/mcp/v1
```

then `/mcp` → Authenticate.

**ChatGPT:** requires a plan that supports custom connectors and Developer Mode
(Pro or higher). Settings → Connectors → enable Developer Mode under Advanced
Settings → Create → Name `Leadspace`, URL
`https://skprod.leadspace.com/mcp/v1`, Authentication `OAuth`, leave the client
fields blank → Create → Connect → sign in. Then enable Leadspace from the
`+` / Apps menu inside a chat.

**Any other MCP client** (Codex included) uses the same endpoint and the same
sign-in.

## 3. Know what the token does and does not carry

The access token carries OIDC identity scopes only — `openid`, `profile`,
`email`, `offline_access`. There is no Leadspace-defined authorization scope, so
you **cannot** scope an agent down to "lookups only" or "no phone reveals" at
the token level. Every entitlement decision is made server-side against the
account.

Practical consequence: the credit allowance is the only spend control. Treat it
as the budget, not as a safety net.

## 4. Work the flow

Discover the live tool list from the client after authenticating — `tools/list`
is auth-gated, so the published surface is described by capability rather than
by tool name. The one tool name Leadspace publishes is `enrich_account`.

The intended sequence is **enrich → find → reveal → act**:

1. **Enrich an account.** Industry, size, revenue, location, full corporate
   hierarchy. Free — no credit cost.
2. **Find the right person** inside that account. Browsing company and contact
   intelligence is free.
3. **Reveal** only the contact you actually need. This is the metered step.
4. **Check buying intent** for the account if you need prioritization, at 1
   credit per company.

## 5. Credit rules that change how you should prompt

| Action | Cost |
|---|---|
| Company and contact lookup | Free |
| Account intelligence (industry, size, revenue, hierarchy) | Free |
| Buying intent, per company | 1 credit |
| Reveal business email | 1 credit |
| Reveal phone (direct + mobile) | 3 credits |
| Re-revealing something already unlocked | Free |

- **Never reveal in a loop.** A single "reveal phone numbers for these 50
  contacts" instruction costs 150 credits and exhausts a Starter account six
  times over. Filter first with free lookups, reveal last, one at a time.
- **Re-reveals are free**, so re-asking for a contact you already unlocked costs
  nothing. Do not cache aggressively to save credits; cache to save latency.
- Allowances: Starter 25/day, Solo 1,500/month, Pro 3,500/month, Enterprise
  custom with full rollover.

## 6. Handle the data responsibly

Responses carry personal data about identified individuals — business email,
direct dial, mobile. Handle under your Leadspace DPA. Leadspace publishes a Do
Not Sell process and a Texas data-broker notice; an agent that stores or
redistributes revealed contact data inherits those obligations.

## Failure modes

- **401 with `WWW-Authenticate: Bearer realm="leadspace-builders"`** — the
  connector is not authenticated. Re-run the OAuth sign-in; do not retry the
  call.
- **Connector added but no tools appear** — the OAuth flow was not completed.
  In ChatGPT, also confirm Leadspace is enabled from the `+` / Apps menu in the
  specific chat.
- **Credits exhausted** — reveals stop. Lookups still work. On the free tier the
  allowance resets daily.
