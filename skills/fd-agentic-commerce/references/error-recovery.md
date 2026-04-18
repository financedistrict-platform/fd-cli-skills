# Error Recovery Playbook

Both UCP and ACP return structured errors. Shapes differ slightly; severity semantics are the same.

## UCP error shape

UCP errors arrive in the response body under `messages[]`. Each entry:

```json
{
  "type": "error" | "warning" | "info",
  "code": "missing_email",
  "content": "Buyer email is required. Provide it via PUT ...",
  "severity": "recoverable" | "requires_buyer_input" | "requires_buyer_review" | "requires_escalation" | "unrecoverable",
  "path":   "$.buyer.email"
}
```

You also see HTTP-level errors (4xx/5xx), which are separate — those come as `{ "type": "invalid_data", "message": "..." }` without a `messages[]` array.

## Severity → action

| Severity | Do this |
| --- | --- |
| `recoverable` | You can fix it yourself by PUT-ing the right field. Use `path` (JSONPath) to locate what to send. |
| `requires_buyer_input` | Ask the user for the info and then PUT. |
| `requires_buyer_review` | Re-show the summary to the user and ask them to confirm. Usually triggered when totals change mid-flow. |
| `requires_escalation` | Stop. Do NOT retry. Show the user the full message and suggest contacting the merchant. |
| `unrecoverable` | Session is dead. Offer to start a new checkout. |

## Retry budget

- Up to **3 recoverable retries** per checkout. If you're still stuck after 3, ask the user for guidance.
- Use a **fresh `Idempotency-Key` on each retry** — the key is meant to dedupe *identical* requests, not to mean "retry."

## Common codes seen against Medusa + Prism merchants

| Code | Severity | Fix |
| --- | --- | --- |
| `missing_email` | recoverable | PUT `buyer.email` |
| `missing_shipping_address` | recoverable | PUT `shipping_address` |
| `country_not_supported` | recoverable | Ask user for a country from the list in `content` |
| `invalid_variant` / `variant_not_found` | recoverable | Re-run `catalog/lookup` with the id — it may have been sold out or deleted |
| `out_of_stock` | requires_buyer_input | Tell the user; offer alternative variant |
| `price_changed` | requires_buyer_review | Re-show summary with new total, re-confirm |
| `ready_for_complete` (type: info) | — | Proceed to user confirmation + complete |
| `payment_rejected` | requires_escalation | Stop, surface message. Don't retry blindly — wallet signature or facilitator rejected. |

## HTTP-level errors

Direct validation errors (wrong JSON shape, missing required field at the schema level) come back as:

```json
{ "type": "invalid_data", "message": "Invalid request: Unrecognized fields: 'limit'" }
```

These mean the request shape is wrong — not a business-logic error. Read the message carefully; usually a schema/path issue. Examples:

- `Unrecognized fields: 'limit'` — field is in the wrong place (e.g. `limit` belongs under `pagination.limit`, not top-level)
- `Expected string, received number` — type mismatch

Fix the request and retry with a new `Idempotency-Key`.

## What to say to the user

- **Recoverable, you can self-heal** → don't bother the user; just retry. Mention only if it repeats.
- **Needs their input** → one clean question, not a dump of the raw error.
- **Escalation / unrecoverable** → show the full `content` field verbatim, plus a suggested next step (contact merchant, try another variant, etc.).

## ACP error shape

ACP errors on 4xx/5xx responses come as a top-level object (not nested in `messages[]`):

```json
{
  "type":    "invalid_request" | "rate_limit" | "authentication_error" | "processing_error" | "api_error",
  "code":    "unauthorized" | "missing_api_version" | "missing_payment_data" | ...,
  "message": "Human-readable description",
  "param":   "optional.path.to.field"
}
```

- **4xx** = client error; fix and retry.
- **5xx** = server error; wait + retry with same `Idempotency-Key`.

For **success responses** with recoverable issues (e.g. missing shipping), ACP returns `messages[]` inside the 200 response body — same severity semantics as UCP:

```json
{ "status": "incomplete",
  "messages": [
    { "type": "error", "code": "missing_shipping", "severity": "recoverable",
      "path": "$.fulfillment_details.address", "content": "..." }
  ]
}
```

## Common ACP error codes

| Code | HTTP | Action |
| --- | --- | --- |
| `unauthorized` | 401 | Missing or invalid Bearer token — ask user for the merchant's API key |
| `missing_api_version` | 400 | Add `Api-Version: 2026-01-30` header |
| `unsupported_api_version` | 400 | Use a version from the `supported_versions` array on `.well-known/acp.json` |
| `missing_payment_data` | 400 | Payload missing `payment_data.instrument.credential.authorization` |
| `not_found` | 404 | Session id invalid or expired |

## Path→action shortcuts

When a recoverable error carries a `path` or `param`:

| Path | PUT/POST this field |
| --- | --- |
| `$.buyer.email` | `buyer.email` (both protocols) |
| `$.shipping_address` | UCP: `shipping_address`. ACP: `fulfillment_details.address`. |
| `$.shipping_address.address_country` | The country isn't supported — ask user for a country from the list in the error's `content` |
| `$.line_items[0].id` or `$.line_items[0].item.id` | Re-run catalog lookup; item may be sold out or removed |
