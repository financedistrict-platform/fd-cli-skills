---
name: fd-agentic-commerce
description: Complete a shopping checkout at any agentic-commerce merchant that accepts the Finance District Prism payment handler (xyz.fd.prism_payment). Works with both UCP (Universal Commerce Protocol) and ACP (Agentic Checkout Protocol) merchants — the skill auto-detects which protocol the store speaks. Use when the user asks to "buy", "order", "purchase", "shop", "checkout", or "get me something" from a storefront. Covers the full flow — merchant discovery, catalog/product browsing, checkout session create/update/complete, shipping-address collection, x402 payment authorization via the FD Agent Wallet, and order confirmation. Single-merchant purchases only (no cross-merchant carts). Do NOT use for general crypto wallet operations (that's the finance-district skill).
user-invocable: true
disable-model-invocation: false
allowed-tools:
  - "Bash(curl *)"
  - "Bash(fdx wallet authorizePayment*)"
  - "Bash(fdx wallet getWalletOverview*)"
  - "Bash(fdx status*)"
  - "Bash(fdx wallet getMyInfo*)"
metadata:
  openclaw:
    emoji: "🛍️"
    requires:
      bins:
        - fdx
      tools:
        - curl
    install:
      - type: node
        package: "@financedistrict/fdx"
---

# FD Agentic Commerce

Complete purchases at any agentic-commerce merchant that accepts the Finance
District **Prism payment handler** (`xyz.fd.prism_payment`). The skill
auto-detects whether the store speaks **UCP** (Universal Commerce Protocol) or
**ACP** (Agentic Checkout Protocol) and dispatches to the right wire. Users
don't need to know — they just shop.

You drive the HTTP flow with `curl`; the FD Agent Wallet signs the x402
payment.

## 1. Prerequisites

- **FD Agent Wallet available** — either `fdx` CLI (`fdx status` returns authenticated) or the FD MCP server at `mcp.fd.xyz` exposing `authorizePayment`. If neither is present, stop and tell the user to install `@financedistrict/fdx` and sign in via the `finance-district` skill.
- **Merchant URL** — the user must give you the merchant's base URL (e.g. `https://medusa.test.1stdigital.tech`). Do not invent one.
- **Curl or equivalent HTTP tool** — for every request.
- **ACP only: merchant API key** — ACP merchants require a Bearer token issued out-of-band. If the merchant speaks ACP, ask the user for their API key before proceeding. UCP does not require one.

## 2. Safety Rules — Non-Negotiable

These protect the user's money. Follow them every time, regardless of protocol.

1. **Verify the Prism handler before paying.** The merchant's discovery doc (or the checkout session response, depending on protocol) must advertise `xyz.fd.prism_payment`. If it doesn't, refuse to pay.
2. **Never pay silently.** Before calling `complete`, show the user: line items, shipping, grand total with currency, shipping destination, and the payment network + asset + human-readable amount. Require explicit "yes, charge $X" confirmation.
3. **Spending cap: $500 USD-equivalent** (configurable — see §7). If the total exceeds the cap, require the user to type back the exact amount.
4. **Never overwrite an existing shipping address without asking.** Confirm before updating.
5. **After payment, always display** the order id, the on-chain tx hash, and any receipt link. The user needs these to verify settlement.

## 3. Protocol Dispatch

First action of any flow: detect which protocol the merchant speaks.

```bash
# Try ACP first (handlers populate at discovery, cheap signal)
curl -s -w "\n%{http_code}" "$MERCHANT/.well-known/acp.json"

# If that 404s or doesn't return a valid body, try UCP
curl -s -w "\n%{http_code}" "$MERCHANT/.well-known/ucp"
```

**ACP signature** — response shape:
```json
{ "protocol": { "name": "acp", "version": "2026-01-30" },
  "api_base_url": "<base>/acp",
  "capabilities": { "payment": { "handlers": [ { "id": "xyz.fd.prism_payment", ... } ] } } }
```

**UCP signature** — response shape:
```json
{ "ucp": { "version": "2026-01-11",
           "services": { "dev.ucp.shopping": [{ "endpoint": "<base>/ucp" }] },
           "payment_handlers": { ... } } }
```

If the merchant publishes **both** (our demo store `medusa.test.1stdigital.tech` does), prefer **UCP** — it requires no API key and has richer browse endpoints (JSON search + lookup vs ACP's RSS product feed).

Set `$PROTOCOL` to `ucp` or `acp` and branch. For the wire-level detail of each protocol, read the matching reference file **only**:

- UCP → [references/ucp-wire.md](references/ucp-wire.md)
- ACP → [references/acp-wire.md](references/acp-wire.md)

## 4. Universal Flow

The steps below are the same regardless of protocol; only the request/response shapes change (see the wire docs).

### 4.1 Discover + verify handler

Fetch the discovery doc. Confirm `xyz.fd.prism_payment` is advertised. If not, refuse.

### 4.2 Browse the catalog

- **UCP**: `POST $UCP/catalog/search` and `POST $UCP/catalog/lookup` (both JSON). See [ucp-wire.md](references/ucp-wire.md).
- **ACP**: `GET $ACP/../product-feed` returns a Google Shopping–style RSS/XML feed. Parse it to get product ids, titles, prices, and links. There is no JSON search endpoint. See [acp-wire.md](references/acp-wire.md).

Present 2–3 thoughtful picks (title, price, why-this-one). Ask which to buy.

### 4.3 Collect shipping + buyer info

Ask the user for: full name, email, street, city, postal code, country (ISO 3166-1 alpha-2 preferred). Don't guess.

### 4.4 Create the checkout session

- **UCP**: `POST $UCP/checkout-sessions` with `line_items[].item.id` + `quantity`, optional `buyer` and `shipping_address`.
- **ACP**: `POST $ACP/checkout_sessions` with `line_items[].id` + `quantity`, optional `buyer` and `fulfillment_details.address`. Requires `Authorization: Bearer <key>` and `Api-Version: 2026-01-30`.

Capture the session id.

### 4.5 Update with missing fields (if needed)

- **UCP**: `PUT $UCP/checkout-sessions/{id}`
- **ACP**: `POST $ACP/checkout_sessions/{id}` (yes, POST — ACP uses POST for updates)

Wait for `status: "ready_for_complete"`.

### 4.6 Confirm with the user (MANDATORY)

Plain-English summary:
```
Order from <merchant> (via <UCP|ACP>):
  • <qty>× <title> — <price>
  Shipping to <city, country>: <shipping>
  Total: <currency> <grand_total>

Paying via Prism → x402 on <network>, <asset_name>, <human_amount>.

Confirm? (yes/no)
```

If total > $500 USD-equivalent, require the user to type back the exact amount.

### 4.7 Authorize payment via FD Agent Wallet

Pick an x402 `accepts[]` entry (or the ACP equivalent — see wire docs) where the wallet has balance on that network/asset.

```bash
fdx wallet authorizePayment --requirements '<JSON of the chosen accepts[] entry>'
```

Or via MCP: call the `authorizePayment` tool with the same object. Wallet returns a signed `PaymentPayload` (for UCP) or EIP-3009 authorization string (ACP extracts it from the same payload).

For payment selection heuristics, network/asset tables, and atomic-unit conversion, see [references/payment-payload.md](references/payment-payload.md).

### 4.8 Complete the checkout

- **UCP**: `POST $UCP/checkout-sessions/{id}/complete` with `payment.instruments[]` carrying both `paymentPayload` and `paymentRequirements`.
- **ACP**: `POST $ACP/checkout_sessions/{id}/complete` with `payment_data.instrument.credential.authorization` (the EIP-3009 auth string) and `x402_version`.

Exact bodies in the wire docs.

### 4.9 Confirm to the user

Show: `order.id`, `tx_hash`, block-explorer link if derivable from CAIP-2 network id, and any receipt URL. Tell the user delivery is on its way.

## 5. Error Recovery

Both protocols return structured errors with severity levels:

| Severity | Action |
| --- | --- |
| `recoverable` | Fix and retry (usually PUT/POST a missing field) |
| `requires_buyer_input` | Ask user, then retry |
| `requires_buyer_review` | Re-show summary, re-confirm |
| `requires_escalation` | Stop; surface the full message |
| `unrecoverable` | Session is dead; offer to start over |

Retry budget: 3 recoverable retries per session, then ask the user.

For protocol-specific error shapes and common codes, see [references/error-recovery.md](references/error-recovery.md).

## 6. Scope

**In scope:** single merchant, single checkout, full browse-to-pay on either UCP or ACP.

**Out of scope (v1):**
- Cross-merchant carts (no spec defines atomicity across merchants)
- Merchant discovery via a registry (there's no registry yet)
- Subscriptions / recurring payments
- Refunds and cancellations — separate skill

## 7. Configuration

- **Spending cap**: default **$500 USD-equivalent**. Raise only if the user explicitly says so in the current conversation.
- **Retry limit**: 3 recoverable errors per session.
- **Protocol preference** when both are available: **UCP** (no API key required).
- **User-Agent**: default `claude-code/1.0` — set to whatever identifies your harness.

## 8. Reference Files

Load only the one you need for the current step.

- [references/ucp-wire.md](references/ucp-wire.md) — UCP request/response shapes, status lifecycle, headers
- [references/acp-wire.md](references/acp-wire.md) — ACP request/response shapes, API key + version headers, product-feed parsing
- [references/payment-payload.md](references/payment-payload.md) — x402 `accepts[]` anatomy, CAIP-2 networks, token decimals, selection heuristics
- [references/error-recovery.md](references/error-recovery.md) — severity taxonomy, common codes for both protocols
- [examples/gift-for-brother.md](examples/gift-for-brother.md) — full walkthrough of the acceptance-test scenario (UCP against the live demo store)
