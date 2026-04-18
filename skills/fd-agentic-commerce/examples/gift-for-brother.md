# Example: "Gift for my brother from Finance District merch store"

Full walkthrough of the acceptance-test scenario against the live demo store:
`https://medusa.test.1stdigital.tech`

## The user prompt

> "Purchase a gift for my brother from Finance District merch store. Pick a few
> recommendations, confirm with me, then place the order."

## Step 1 — Protocol dispatch

Try ACP first, then UCP:

```bash
curl -s -w "\n%{http_code}" https://medusa.test.1stdigital.tech/.well-known/acp.json
# → 200, includes protocol.name == "acp", handlers include xyz.fd.prism_payment
#   (would work, but ACP requires a merchant-issued API key we don't have)

curl -s https://medusa.test.1stdigital.tech/.well-known/ucp
# → also supports UCP. Prefer UCP (no API key needed, JSON search endpoints).
```

This merchant supports both. Per §7 of SKILL.md we prefer **UCP** when both are
available. Set `$PROTOCOL=ucp`, `$UCP=https://medusa.test.1stdigital.tech/ucp`.

Response (abbreviated) from UCP discovery:
```json
{ "ucp": {
    "version": "2026-01-11",
    "services": { "dev.ucp.shopping": [{ "endpoint": "https://medusa.test.1stdigital.tech/ucp" }] },
    "capabilities": { "dev.ucp.shopping.catalog.search": [...], ... },
    "payment_handlers": {}
} }
```

`payment_handlers` is empty on discovery — fine, we'll verify on the checkout session response.

Base URL: `https://medusa.test.1stdigital.tech/ucp`.

## Step 2 — Browse and pick 2-3 options

```bash
curl -s -X POST "$UCP/catalog/search" \
  -H "Content-Type: application/json" -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: req-$(date +%s)-a" \
  -d '{"pagination":{"limit":10}}'
```

Pick 2-3 thoughtful items. Example agent message:

> I found these at the FD store:
>
> 1. **FD Unisex T-Shirt** (size M) — €29.00. Cotton, black, FD logo. Safe go-to.
> 2. **FD Hoodie** — €65.00. Heavier, good if brother is in a cold climate.
> 3. **FD Cap** — €19.00. Low-commitment stocking-filler.
>
> Which should I pick? Also — what's the ship-to country and address, and your
> email for the receipt?

User picks: T-shirt size M. Shipping to 10115 Berlin, Germany. Email: brother@example.com.

## Step 3 — Create the checkout session

```bash
curl -s -X POST "$UCP/checkout-sessions" \
  -H "Content-Type: application/json" -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: req-$(date +%s)-b" -H "Idempotency-Key: idem-$(date +%s)-b" \
  -d '{"line_items":[{"item":{"id":"variant_01KNS3N3CY19J6J3AWGC0KJDQK"},"quantity":1}],
       "context":{"address_country":"DE","currency":"eur"}}'
```

Response (abbreviated):
```json
{ "id": "cart_01KPF...",
  "status": "incomplete",
  "ucp": { "payment_handlers": {
      "xyz.fd.prism_payment": [{ "id": "x402", "config": { "accepts": [ ... ], "x402Version": 2 } }]
  } },
  "totals": [ {"type":"subtotal","amount":2900}, {"type":"total","amount":2900} ],
  "messages": [
    {"code":"missing_email","severity":"recoverable","path":"$.buyer.email"},
    {"code":"missing_shipping_address","severity":"recoverable","path":"$.shipping_address"}
  ]
}
```

**Verify** `ucp.payment_handlers["xyz.fd.prism_payment"]` is present — it is. Safe to proceed.

## Step 4 — PUT buyer + shipping

```bash
curl -s -X PUT "$UCP/checkout-sessions/cart_01KPF..." \
  -H "Content-Type: application/json" -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: req-$(date +%s)-c" -H "Idempotency-Key: idem-$(date +%s)-c" \
  -d '{
    "buyer": { "email": "brother@example.com", "first_name": "Max", "last_name": "Doe" },
    "shipping_address": {
      "first_name": "Max", "last_name": "Doe",
      "street_address": "Unter den Linden 1",
      "address_locality": "Berlin",
      "address_country": "DE",
      "postal_code": "10115"
    }
  }'
```

Response:
```json
{ "status": "ready_for_complete",
  "totals": [
    {"type":"subtotal","amount":2900},
    {"type":"fulfillment","amount":500},
    {"type":"total","amount":3400}
  ],
  "messages": [{"type":"info","code":"ready_for_complete","content":"Checkout is ready. POST ..."}]
}
```

## Step 5 — Confirm with user

> Here's the order:
>
> - 1× FD Unisex T-Shirt (M) — €29.00
> - Shipping to Berlin, DE — €5.00
> - **Total: €34.00**
>
> Paying via Prism → x402 on Base Sepolia, USDC, $0.34 (340000 atomic).
>
> Confirm? (yes/no)

User says "yes".

## Step 6 — Authorize

Pick the USDC-on-Base-Sepolia entry (wallet has balance):

```bash
fdx wallet authorizePayment --requirements '{
  "scheme":"exact","network":"eip155:84532",
  "asset":"0x036cbd53842c5426634e7929541ec2318f3dcf7e",
  "amount":"340000",
  "payTo":"0x40a01003f7543a3a3ee64ffb05504173bdb1c4fd",
  "maxTimeoutSeconds":300,
  "extra":{"name":"USDC","version":"2"}
}'
```

Returns a signed `PaymentPayload` — save it.

## Step 7 — Complete

```bash
curl -s -X POST "$UCP/checkout-sessions/cart_01KPF.../complete" \
  -H "Content-Type: application/json" -H "UCP-Agent: claude-code/1.0" \
  -H "Request-Id: req-$(date +%s)-d" -H "Idempotency-Key: idem-$(date +%s)-d" \
  -d '{
    "payment": {
      "instruments": [{
        "id": "inst_1",
        "handler_id": "xyz.fd.prism_payment",
        "type": "default",
        "credential": {
          "type": "default",
          "paymentPayload": { /* from authorizePayment */ },
          "paymentRequirements": { /* the same accepts[] entry, verbatim */ }
        }
      }]
    }
  }'
```

## Step 8 — Confirm to user

> Done! Order `order_01KPF...` placed.
>
> On-chain tx: `0xabc123...` (Base Sepolia)
> Receipt: https://medusa.test.1stdigital.tech/orders/...
>
> Shipping to Max Doe, Unter den Linden 1, 10115 Berlin, DE. Brother's gonna love it.

---

## ACP variant — same flow, different wire

If the merchant only spoke ACP (or we had a valid API key and preferred ACP):

```bash
# Discovery
curl -s https://merchant/.well-known/acp.json

# Browse — parse the RSS feed (no JSON search)
curl -s https://merchant/acp/product-feed

# Create (note: line_items[].id, not item.id; requires Authorization + Api-Version)
curl -s -X POST https://merchant/acp/checkout_sessions \
  -H "Authorization: Bearer $ACP_KEY" \
  -H "Api-Version: 2026-01-30" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"line_items":[{"id":"variant_...","quantity":1}],"currency":"eur","capabilities":{}}'

# Update uses POST (not PUT), fulfillment_details.address
curl -s -X POST https://merchant/acp/checkout_sessions/$ID \
  -H "Authorization: Bearer $ACP_KEY" -H "Api-Version: 2026-01-30" \
  -H "Content-Type: application/json" -H "Idempotency-Key: $(uuidgen)" \
  -d '{"buyer":{"email":"..."},"fulfillment_details":{"address":{"name":"...","line_one":"...","city":"...","state":"...","country":"DE","postal_code":"..."}}}'

# Complete — flat credential shape
curl -s -X POST https://merchant/acp/checkout_sessions/$ID/complete \
  -H "Authorization: Bearer $ACP_KEY" -H "Api-Version: 2026-01-30" \
  -H "Content-Type: application/json" -H "Idempotency-Key: $(uuidgen)" \
  -d '{"payment_data":{"handler_id":"xyz.fd.prism_payment","instrument":{"type":"x402_authorization","credential":{"type":"x402_authorization","authorization":"<eip3009 auth>","x402_version":2}}}}'
```

The user-facing steps (pick, confirm, pay, show tx) are identical.
