# UCP Protocol Flow — Full Detail

The Universal Commerce Protocol (UCP) spec lives at https://github.com/Universal-Commerce-Protocol/ucp. This doc captures the parts you actually need to drive a checkout against a UCP-compliant merchant using the Finance District Prism payment handler.

## Base URL resolution

The merchant publishes a discovery document at `GET {merchant}/.well-known/ucp`:

```json
{
  "ucp": {
    "version": "2026-01-11",
    "services": {
      "dev.ucp.shopping": [
        { "version": "2026-01-11", "transport": "rest", "endpoint": "https://merchant.example/ucp" }
      ]
    },
    "capabilities": {
      "dev.ucp.shopping.catalog.search":  [{ "version": "2026-01-11" }],
      "dev.ucp.shopping.catalog.lookup":  [{ "version": "2026-01-11" }],
      "dev.ucp.shopping.checkout":        [{ "version": "2026-01-11" }],
      "dev.ucp.shopping.cart":            [{ "version": "2026-01-11" }],
      "dev.ucp.shopping.order":           [{ "version": "2026-01-11" }]
    },
    "payment_handlers": {
      "xyz.fd.prism_payment": [ ... ]
    }
  }
}
```

- Use `services["dev.ucp.shopping"][0].endpoint` as the base UCP URL (call it `$UCP`).
- Some merchants return an empty `payment_handlers` object on discovery — that is OK; the handler list on a checkout session response is authoritative.

## Headers (every request)

- `UCP-Agent: <agent-id>/<version>` — identifies your agent. Merchants may rate-limit or audit by this.
- `Request-Id: <unique>` — a per-request unique id (use UUID v4).
- `Idempotency-Key: <unique>` — required on POST and PUT. Same key = same outcome.
- `Content-Type: application/json` — for POST/PUT bodies.

## Catalog search

`POST $UCP/catalog/search`

Request:
```json
{
  "query": "t-shirt",
  "filters": { "category": "Apparel", "min_price": 100, "max_price": 10000 },
  "pagination": { "limit": 20, "offset": 0 }
}
```

Response:
```json
{
  "ucp": { "version": "...", "status": "success" },
  "products": [
    {
      "id": "prod_...",
      "title": "FD Unisex T-Shirt",
      "description": "...",
      "categories": ["Apparel"],
      "price_range": { "min": {"amount": 2900, "currency": "eur"}, "max": {...} },
      "variants": [
        { "id": "variant_...", "title": "M",
          "price": {"amount": 2900, "currency": "eur"},
          "availability": {"available": true, "status": "in_stock"} }
      ],
      "media": [{ "url": "...", "type": "image" }]
    }
  ],
  "pagination": { "total": 1, "limit": 20, "offset": 0, "has_more": false }
}
```

- `price.amount` is in the currency's atomic units — EUR `2900` = €29.00.
- Put `variant.id` into `line_items[].item.id` on checkout.

## Catalog lookup

`POST $UCP/catalog/lookup` — same product shape, when you already know ids.

```json
{ "ids": ["prod_...", "variant_..."] }
```

## Checkout session lifecycle

### Statuses

Per spec:
- `incomplete` — fields missing
- `ready_for_complete` — all required fields present, ready to pay
- `complete_in_progress` — complete request accepted, settlement running
- `completed` — final state, order created
- `requires_escalation` — manual handling needed (fraud, KYC, etc.)
- `canceled` — user/merchant canceled
- `expired` — session TTL elapsed (`expires_at` in response)

Only call `complete` when `status == "ready_for_complete"`.

### Create — POST $UCP/checkout-sessions

```json
{
  "line_items": [ { "item": { "id": "variant_..." }, "quantity": 1 } ],
  "context": { "address_country": "DE", "currency": "eur" },
  "buyer": { "email": "...", "first_name": "...", "last_name": "..." },
  "shipping_address": { ... }
}
```

`buyer` and `shipping_address` are optional on create — if you omit them you get `recoverable` errors telling you what to PUT.

### Update — PUT $UCP/checkout-sessions/{id}

Same body shape as create, all fields optional. Fields you include overwrite the session's current value.

### Session response shape

```json
{
  "ucp": {
    "version": "2026-01-11",
    "status": "success",
    "capabilities": { ... },
    "payment_handlers": {
      "xyz.fd.prism_payment": [
        { "id": "x402", "version": "2026-01-15", "config": { "accepts": [ ... ], "resource": { ... }, "x402Version": 2 } }
      ]
    }
  },
  "id": "cart_...",
  "status": "incomplete",
  "currency": "EUR",
  "line_items": [
    { "id": "cali_...", "item": {"id": "variant_...", "title": "...", "price": 2900},
      "quantity": 1, "totals": [{ "type": "line_total", "amount": 2900 }] }
  ],
  "totals": [
    { "type": "subtotal",    "display_text": "Subtotal", "amount": 2900 },
    { "type": "fulfillment", "display_text": "Shipping", "amount": 500  },
    { "type": "total",       "display_text": "Total",    "amount": 3400 }
  ],
  "messages": [ ... ],
  "links":    [ ... ],
  "buyer":    { ... },
  "shipping_address": { ... },
  "expires_at": "2026-04-18T09:18:29.379Z"
}
```

### Complete — POST $UCP/checkout-sessions/{id}/complete

```json
{
  "payment": {
    "instruments": [
      {
        "id": "inst_1",
        "handler_id": "xyz.fd.prism_payment",
        "type": "default",
        "credential": {
          "type": "default",
          "paymentPayload":      { /* from fdx authorizePayment */ },
          "paymentRequirements": { /* the accepts[] entry you picked */ }
        }
      }
    ]
  }
}
```

A successful response carries `status: "completed"`, an `order` object (with the merchant's order id), and on-chain settlement info including a `tx_hash`. The `messages[]` array may include an `info` entry linking to a receipt; `links[]` may include a `receipt` or `order` URL.

## Cancel

`POST $UCP/checkout-sessions/{id}/cancel` — empty body. Moves status to `canceled`. Only available before `complete_in_progress`.

## Order retrieval

`GET $UCP/orders/{order_id}` — fetch final order details after completion. Use the order id returned in the complete response.

## Shipping address schema

Per UCP spec `postal_address.json`:
```json
{
  "first_name": "Jane",
  "last_name":  "Doe",
  "street_address":  "Marszalkowska 1",
  "extended_address": "Apt 4",
  "address_locality": "Warsaw",
  "address_region":   "Mazovian",
  "address_country":  "PL",
  "postal_code": "00-001",
  "phone_number": "+48..."
}
```

- `address_country` — prefer ISO 3166-1 alpha-2. Alpha-3 and full names accepted but merchants may reject.
- Merchant-supported countries vary. If you get `country_not_supported`, the error lists what IS supported — present those to the user.
