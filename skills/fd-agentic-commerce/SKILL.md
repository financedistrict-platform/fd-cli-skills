---
name: fd-agentic-commerce
description: Complete a shopping checkout at any agentic-commerce merchant that accepts the Finance District Prism payment handler (xyz.fd.prism_payment). Supports UCP today; ACP support planned. Use when the user asks to "buy", "order", "purchase", "shop", "checkout", or "get me something" from a specific store URL or an agentic-commerce storefront. Covers the full flow — merchant discovery, catalog search and product lookup, checkout session create/update/complete, shipping-address collection, x402 payment authorization via the FD Agent Wallet, and order confirmation. Single-merchant purchases only (no cross-merchant carts). Do NOT use for general crypto wallet operations (that's the finance-district skill) or for non-agentic merchant sites.
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
District **Prism payment handler** (`xyz.fd.prism_payment`). v1 covers the
Universal Commerce Protocol (UCP); ACP support is planned. You drive the HTTP
flow with `curl`; the FD Agent Wallet signs the x402 payment.

## 1. Prerequisites

- **FD Agent Wallet available** — either the `fdx` CLI (`fdx status` returns authenticated) or the FD MCP server at `mcp.fd.xyz` exposing `authorizePayment`. If neither is present, stop and tell the user to install `@financedistrict/fdx` and sign in via the `finance-district` skill.
- **Merchant URL** — the user must give you the merchant's base URL (e.g. `https://medusa.test.1stdigital.tech`). Do not invent one.
- **Curl or equivalent HTTP tool** — used for every UCP request.

## 2. Safety Rules — Non-Negotiable

These protect the user's money. Follow them every time.

1. **Verify the Prism handler before paying.** Inspect the checkout session response (see §4.3) — its `ucp.payment_handlers` must contain `xyz.fd.prism_payment`. If it does not, refuse to pay and tell the user the merchant is not Prism-compatible.
2. **Never pay silently.** Before calling `complete`, show the user: line items, shipping total, grand total (with currency), shipping destination, and the payment network + asset + atomic amount. Require explicit "yes, charge $X" confirmation.
3. **Spending cap: $500 USD-equivalent** (configurable — see §6). If the total in user-facing currency exceeds the cap, require the user to type the exact amount back.
4. **Never overwrite an existing shipping address without asking.** If the checkout session already has `shipping_address` set and the user asks to change it, confirm before PUT.
5. **After payment, always display** the `order.id`, the blockchain `tx hash` from the completion response, and any `links[]` entry of type `receipt` or `order`. The user needs these to verify settlement.

## 3. Required HTTP Headers

Send these on every UCP request (the merchant rejects calls without them):

| Header | Value | Notes |
| ------ | ----- | ----- |
| `Content-Type` | `application/json` | POST/PUT bodies |
| `UCP-Agent` | `claude-code/1.0` (or whatever identifies your agent) | Required on every call |
| `Request-Id` | a fresh unique string per request | Use a UUID or `req-<epoch>-<rand>` |
| `Idempotency-Key` | a fresh unique string per POST/PUT | Same key replays the same result |

## 4. Core Flow

### 4.1 Discover the merchant

```bash
curl -s "$MERCHANT/.well-known/ucp"
```

Expected response shape:
```json
{ "ucp": { "version": "2026-01-11",
           "services": { "dev.ucp.shopping": [{ "endpoint": "<base>/ucp", ... }] },
           "capabilities": { "dev.ucp.shopping.catalog.search": [...], ... },
           "payment_handlers": { ... } } }
```

- Use the advertised `services["dev.ucp.shopping"][0].endpoint` as the UCP base URL.
- `payment_handlers` on `.well-known` may be **empty on some merchants**; the authoritative handler list comes from the checkout session response (§4.3). That is still enough — you will verify the Prism handler there before paying.

### 4.2 Browse the catalog

Search:
```bash
curl -s -X POST "$UCP/catalog/search" \
  -H "Content-Type: application/json" \
  -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: $(uuidgen)" \
  -d '{"query":"<terms>","pagination":{"limit":10}}'
```

Lookup (when you already have product or variant ids):
```bash
curl -s -X POST "$UCP/catalog/lookup" \
  -H "Content-Type: application/json" \
  -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: $(uuidgen)" \
  -d '{"ids":["prod_...","variant_..."]}'
```

Each product exposes `variants[]` with an `id` — that variant id is what goes into `line_items[].item.id` on checkout. Prices are in the merchant's currency atomic units (e.g. EUR cents: `2900` = €29.00).

Present 2–3 thoughtful picks to the user with title, price, and why you chose them. Ask which to buy.

### 4.3 Create a checkout session

```bash
curl -s -X POST "$UCP/checkout-sessions" \
  -H "Content-Type: application/json" \
  -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: $(uuidgen)" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "line_items":[{"item":{"id":"variant_..."},"quantity":1}],
    "context":{"address_country":"DE","currency":"eur"}
  }'
```

The response carries:
- `id` — the checkout session id (often prefixed `cart_...`)
- `status` — will be `incomplete` at this stage (buyer + shipping still missing)
- `ucp.payment_handlers["xyz.fd.prism_payment"]` — **verify this exists**, per §2 rule 1. Each entry has a `config.accepts[]` of x402 payment options (asset, network, amount, payTo, scheme, maxTimeoutSeconds) — this is the `paymentRequirements` you'll sign in §4.6.
- `messages[]` — `recoverable` errors telling you what's missing (e.g. `missing_email`, `missing_shipping_address`) with a `path` pointing to the field to PUT.
- `totals[]` — running totals (subtotal, fulfillment, total) in atomic units.

### 4.4 Collect buyer + shipping, then PUT

Ask the user for: full name, email, street, city, postal code, country. Do not guess. Country must be ISO 3166-1 alpha-2 (e.g. `DE`, `PL`, `IT`). The spec also accepts alpha-3 and full names but alpha-2 is safest.

```bash
curl -s -X PUT "$UCP/checkout-sessions/$ID" \
  -H "Content-Type: application/json" \
  -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: $(uuidgen)" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "buyer":{"email":"...","first_name":"...","last_name":"..."},
    "shipping_address":{"first_name":"...","last_name":"...",
      "street_address":"...","address_locality":"<city>",
      "address_country":"DE","postal_code":"..."}
  }'
```

After the PUT, expect `status: "ready_for_complete"` and `totals[]` including a `fulfillment` line (shipping). If `status` is still `incomplete`, read `messages[]` — see §5 for recovery patterns.

### 4.5 Confirm with the user (MANDATORY)

Display a plain-English summary:
```
Order from <merchant>:
  • <qty>× <title> — <price>
  Shipping to <city, country>: <shipping>
  Total: <currency> <grand_total>

Paying via Prism → x402 on <network>, <asset_name>, <human_amount> (atomic: <raw>).

Confirm? (yes/no)
```

Convert atomic units to human-readable using token decimals (USDC/FDUSD = 6 decimals on `eip155:*` chains with 6-dp stablecoins; check the x402 `accepts[]` entry — if `amount` is a very large integer string it's likely 18-decimal). For UCP `totals[]`, the merchant's currency is typically in minor units (cents) — divide by 100.

If the total (in USD equivalent) exceeds **$500**, require the user to type back the exact amount.

### 4.6 Authorize payment with the FD Agent Wallet

Pick the **first acceptable** `accepts[]` entry — ideally one you have balance on. The entry itself is the `paymentRequirements`.

**Via CLI** (preferred when `fdx` is installed):
```bash
fdx wallet authorizePayment --requirements '<JSON of one accepts[] entry>'
```

**Via MCP** (when running in an MCP host such as Claude Desktop connected to `mcp.fd.xyz`): call the `authorizePayment` tool with the same `paymentRequirements` object.

The wallet returns a `PaymentPayload` object — capture it verbatim; it includes the EIP-3009 signature.

If the wallet reports insufficient balance on the chosen network/asset, pick a different `accepts[]` entry. If none work, stop and tell the user which assets they need to fund (show their FD Agent Wallet address).

### 4.7 Complete the checkout session

The UCP `complete` body is a `payment.instruments[]` array. Each instrument's `credential` carries both the signed `paymentPayload` and the original `paymentRequirements`:

```bash
curl -s -X POST "$UCP/checkout-sessions/$ID/complete" \
  -H "Content-Type: application/json" \
  -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: $(uuidgen)" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "payment": {
      "instruments": [{
        "id": "inst_1",
        "handler_id": "xyz.fd.prism_payment",
        "type": "default",
        "credential": {
          "type": "default",
          "paymentPayload": <from fdx authorizePayment>,
          "paymentRequirements": <the accepts[] entry you chose>
        }
      }]
    }
  }'
```

Successful completion returns a response with:
- `status: "completed"`
- `order` — with `id` and merchant order metadata
- Transaction details in the payment result — look for `tx_hash` / `transaction` fields, and any `links[]` entry of type `receipt`.

### 4.8 Confirm to the user

Show: `order.id`, `tx_hash`, a block-explorer link if you can derive one from the CAIP-2 network id, and any `receipt` link from `links[]`. Tell the user the purchase is complete and delivery is to the address they gave.

## 5. Error Recovery

UCP responses carry a `messages[]` array. Each message has `type` (`error` | `warning` | `info`), `severity` (for errors), and often a JSONPath `path`.

| severity | Meaning | What to do |
| --- | --- | --- |
| `recoverable` | Fixable by sending another request. | Read `path` → PUT the missing/fixed field → retry. |
| `requires_buyer_input` | You need more info from the user. | Ask the user; then PUT the field. |
| `requires_buyer_review` | User must confirm something (e.g. total changed). | Re-show the summary and ask to confirm. |
| `requires_escalation` | Out-of-band handling (fraud check, KYC, manual review). | Stop. Report the full message to the user — do not retry. |
| `unrecoverable` | Session is dead. | Tell the user, offer to start over. |

Common recoverable codes seen in the wild:
- `missing_email` — PUT `buyer.email`
- `missing_shipping_address` — PUT `shipping_address`
- `country_not_supported` — ask the user for a country from the list the error names (e.g. "Supported countries: de, dk, es, fr, gb, it, se")
- `invalid_variant` — re-check the variant id via `catalog/lookup`

For deeper error handling, see [references/error-recovery.md](references/error-recovery.md).

## 6. Configuration

The skill honors these soft knobs (state them back to the user if you deviate):

- **Spending cap**: default **$500 USD-equivalent**. Raise only if the user explicitly says so in the current conversation.
- **Retry limit on recoverable errors**: 3 attempts, then stop and ask the user.
- **User agent**: default `claude-code/1.0` — set to whatever identifies your harness.

## 7. Scope

**In scope:** single merchant, single checkout session, full browse-to-pay flow, shipping collection, x402 authorization, order confirmation.

**Out of scope (v1):**
- Cross-merchant carts (UCP doesn't define atomicity across merchants)
- Merchant discovery across a registry
- Subscriptions / recurring payments
- Refunds and cancellations — a dedicated skill will handle those

## 8. Reference Files

Load only the one relevant to the step you're on.

- [references/ucp-flow.md](references/ucp-flow.md) — full protocol detail, field-by-field request/response shapes, status transitions
- [references/error-recovery.md](references/error-recovery.md) — full error taxonomy, recovery playbook, path→field map
- [references/payment-payload.md](references/payment-payload.md) — x402 `accepts[]` entry anatomy, network/asset selection heuristics, CAIP-2 id table, atomic-units conversion
- [examples/gift-for-brother.md](examples/gift-for-brother.md) — a full walkthrough of the acceptance-test scenario against the live demo store
