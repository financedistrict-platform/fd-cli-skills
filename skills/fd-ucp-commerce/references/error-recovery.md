# Error Recovery Playbook

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
