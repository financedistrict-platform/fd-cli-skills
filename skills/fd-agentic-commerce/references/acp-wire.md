# ACP Wire — Full Request/Response Detail

The Agentic Checkout Protocol (ACP) is originally from OpenAI + Stripe + Shopify. Finance District's Prism plugin implements it with the `xyz.fd.prism_payment` handler so agents can pay via x402.

## Key differences from UCP (at a glance)

| Aspect | ACP | UCP |
| --- | --- | --- |
| Discovery | `/.well-known/acp.json` | `/.well-known/ucp` |
| Base path | `/acp` | `/ucp` |
| API key | **Required** (`Authorization: Bearer <key>`) | Optional |
| Version header | `Api-Version: 2026-01-30` | Version in body, not header |
| Browse | RSS/XML product feed only | JSON search + lookup |
| Update method | `POST /checkout_sessions/{id}` | `PUT /checkout-sessions/{id}` |
| Address field | `fulfillment_details.address` (fields: `name`, `line_one`, `city`, `state`, `country`, `postal_code`) | `shipping_address` (fields: `first_name`, `last_name`, `street_address`, `address_locality`, `address_country`, `postal_code`) |
| Item id field | `line_items[].id` | `line_items[].item.id` |
| Complete credential | Flat: `credential.authorization` (EIP-3009 string) + `x402_version` | Nested: `credential.paymentPayload` + `credential.paymentRequirements` |

## Required headers (every request)

| Header | Value |
| --- | --- |
| `Authorization` | `Bearer <merchant-issued API key>` |
| `Api-Version` | `2026-01-30` |
| `Content-Type` | `application/json` (POST bodies) |
| `Idempotency-Key` | fresh unique per POST |
| `Request-Id` | fresh unique per call (echoed back) |
| `User-Agent` | agent identifier, e.g. `claude-code/1.0` |
| `Signature` | optional HMAC — only if merchant requires it |

The API key comes from the merchant (usually via their admin dashboard). If the user doesn't have one, you cannot use ACP against this merchant — either ask the merchant for a key, or fall back to UCP if they speak both.

## Discovery — GET /.well-known/acp.json

```json
{
  "protocol": { "name": "acp", "version": "2026-01-30", "supported_versions": ["2026-01-30"] },
  "api_base_url": "https://merchant.example/acp",
  "transports": ["rest"],
  "capabilities": {
    "services": ["checkout", "orders"],
    "payment": {
      "handlers": [
        { "id": "xyz.fd.prism_payment", "name": "Prism Payment", "version": "2026-01-15",
          "psp": "prism", "requires_delegate_payment": false,
          "instrument_schemas": [
            { "type": "x402_authorization",
              "credential_schema": {
                "type": "object", "required": ["authorization"],
                "properties": {
                  "authorization": { "type": "string" },
                  "x402_version":  { "type": "integer", "enum": [1, 2], "default": 2 }
                }
              }
            }
          ]
        }
      ]
    },
    "supported_currencies": ["eur"],
    "supported_locales": ["en"]
  }
}
```

Verify `capabilities.payment.handlers[*].id` contains `xyz.fd.prism_payment`. Use `api_base_url` as `$ACP`.

## Product feed — GET {merchant}/acp/product-feed

Returns **Google Shopping RSS 2.0** (XML, not JSON). Unlike UCP there's no search endpoint — you fetch the whole feed and filter client-side.

```xml
<rss version="2.0" xmlns:g="http://base.google.com/ns/1.0">
  <channel>
    <title>Store Product Feed</title>
    <item>
      <title><![CDATA[FD Unisex T-Shirt]]></title>
      <link>https://merchant.example/products/fd-unisex-tshirt</link>
      <g:id>prod_...</g:id>
      <g:price>29.00 EUR</g:price>
      <g:availability>in stock</g:availability>
      <g:image_link>...</g:image_link>
    </item>
    <!-- more items -->
  </channel>
</rss>
```

Parse with any XML tool. Match products by keyword/category on the client side. Note: `g:id` is a product id — you'll need to expand to variant ids (often the product itself acts as a sellable unit; confirm by checking the complete checkout flow). Against our Medusa plugin the catalog ids usable in `line_items[].id` are **variant ids** (`variant_...`), which are not in the feed by default. Practical advice: since ACP browsing is limited, **prefer UCP** if the merchant speaks both. If ACP-only, the user may need to supply a variant id directly.

## Create checkout session — POST $ACP/checkout_sessions

```json
{
  "line_items":  [{ "id": "variant_...", "quantity": 1 }],
  "currency":    "eur",
  "capabilities": { "payment": { "handlers": [] } },
  "buyer":       { "email": "...", "first_name": "...", "last_name": "..." },
  "fulfillment_details": {
    "name":  "Max Doe",
    "email": "max@example.com",
    "phone_number": "+49...",
    "address": {
      "name": "Max Doe",
      "line_one": "Unter den Linden 1",
      "line_two": "Apt 4",
      "city":    "Berlin",
      "state":   "BE",
      "country": "DE",
      "postal_code": "10115"
    }
  }
}
```

Required: `line_items`, `currency`, `capabilities`. Buyer + fulfillment are optional on create (missing = recoverable error back).

### Response

```json
{
  "id":       "cart_...",
  "status":   "incomplete" | "ready_for_complete" | "complete_in_progress" | "completed" | "canceled" | "expired",
  "currency": "eur",
  "line_items": [ ... ],
  "totals":     [ { "type": "subtotal", "amount": 2900 }, ... ],
  "buyer":      { ... },
  "fulfillment_details": { ... },
  "messages":   [ ... ],
  "payment_provider": {
    "provider": "stripe" | "prism",
    "payment_intent_client_secret": "..."   // or handler-specific payload
  },
  "links": [ ... ],
  "expires_at": "..."
}
```

The Prism handler's payment requirements live in `payment_provider` (or `payment_handlers` depending on plugin version). Inspect the response to find the x402 `accepts[]` entries — same shape as UCP.

## Update — POST $ACP/checkout_sessions/{id}

**POST, not PUT.** Body shape matches create; all fields optional.

## Complete — POST $ACP/checkout_sessions/{id}/complete

```json
{
  "payment_data": {
    "handler_id": "xyz.fd.prism_payment",
    "instrument": {
      "type": "x402_authorization",
      "credential": {
        "type": "x402_authorization",
        "authorization": "<base64-encoded x402 PaymentAuthorizationResult — see §below>",
        "x402_version":  2
      }
    },
    "billing_address": { /* optional, ACP address shape */ }
  }
}
```

Key difference vs UCP: the credential is **flat** — just an `authorization` string and version number, not a nested `paymentPayload` + `paymentRequirements` pair. `authorizePayment` returns `{ paymentPayload, paymentRequirements }`; for ACP, base64-encode the whole returned object (serialized as the x402 `PaymentAuthorizationResult`) and put that string in `credential.authorization`. See [payment-payload.md](./payment-payload.md) for the envelope shape you pass as input.

### Response

```json
{ "status": "completed",
  "order":  { "id": "order_..." },
  "payment": { "tx_hash": "0x...", "network": "eip155:84532", "amount": "290000", ... },
  "links":  [ { "type": "receipt", "url": "..." } ]
}
```

## Address schema

ACP address (spec: `schema.agentic_checkout.json`):
```json
{
  "name":    "Max Doe",
  "line_one": "Unter den Linden 1",
  "line_two": "Apt 4",
  "city":    "Berlin",
  "state":   "BE",
  "country": "DE",
  "postal_code": "10115"
}
```

Required: `name`, `line_one`, `city`, `state`, `country` (alpha-2), `postal_code`. Note `state` is required in ACP (unlike UCP's optional `address_region`).

## Cancel — POST $ACP/checkout_sessions/{id}/cancel

Empty body. Only available pre-completion.

## Order — GET $ACP/orders/{order_id}

Returns the final order document after `complete`.
