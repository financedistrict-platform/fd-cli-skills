---
name: fd-agentic-commerce
description: Complete a shopping checkout at any agentic-commerce merchant that accepts the Finance District Prism payment handler (xyz.fd.prism_payment). Works with both UCP (Universal Commerce Protocol) and ACP (Agentic Checkout Protocol) merchants — the skill auto-detects which protocol the store speaks. Use when the user asks to "buy", "order", "purchase", "shop", "checkout", or "get me something" from a storefront. Covers the full flow — merchant discovery, catalog/product browsing, checkout session create/update/complete, shipping-address collection, x402 payment authorization via the FD Agent Wallet MCP, and order confirmation. Single-merchant purchases only (no cross-merchant carts). Do NOT use for general crypto wallet operations or non-checkout wallet flows.
user-invocable: true
disable-model-invocation: false
allowed-tools:
  - "Bash(curl *)"
  - "Read"
  - "Write"
  - "Edit"
  - "Bash(git config user.email)"
  - "Bash(git config user.name)"
metadata:
  openclaw:
    emoji: "🛍️"
    requires:
      tools:
        - curl
---

# FD Agentic Commerce

Shop on behalf of the user at any agentic-commerce merchant that accepts
Finance District payment. Auto-detects the store's protocol (UCP or ACP) and
handles everything behind the scenes — the user never needs to hear about
protocols. You drive HTTP with `curl`; the FD Agent Wallet signs the payment.

**First rule: see §0 on how to talk to the user.**

## 0. Talk Like a Shopper, Not a Protocol Nerd

**The user is shopping. They do not care about UCP, ACP, x402, EIP-3009, handlers, or any other protocol/technical name.** Keep that vocabulary out of user-visible messages entirely.

| Instead of saying… | Say this |
| --- | --- |
| "Detecting which protocol the store speaks" | "Checking the store" |
| "Both UCP and ACP available, using UCP" | (say nothing — just proceed) |
| "Prism payment handler confirmed" | (say nothing — verified internally) |
| "Authorizing x402 payment via EIP-3009" | "Paying now" |
| "On-chain tx hash: 0x…" | "Here's your payment confirmation: 0x…" (or just "Transaction: 0x…") |
| "Checkout session ready_for_complete" | (say nothing — move to the confirmation step) |
| "Wallet signing via MCP authorizePayment" | "Paying with your wallet" |

Internally, still do all the protocol-level work (that's what this skill is for). But to the user, narrate only: looking up products, presenting recommendations, asking for shipping details, confirming price, paying, showing proof of purchase.

Rule of thumb: **if a word sounds like it belongs in an RFC, don't say it to the user.**

## 1. Prerequisites

- **FD Agent Wallet MCP attached** — you must have the FD Agent Wallet MCP tools available in your tool list. Look for `authorizePayment`, `getMyInfo`, and `getWalletOverview`. If they are not attached, stop immediately and tell the user to add the MCP server at `https://mcp.fd.xyz` to their MCP client (e.g. Claude Desktop config → `mcpServers`, Claude Code → `claude mcp add fd-wallet https://mcp.fd.xyz`) and sign in through the OAuth flow the server triggers. Do not attempt to proceed without it.
- **Wallet is signed in** — call the `getMyInfo` MCP tool once. If it returns an auth error, tell the user to complete the OAuth flow in their MCP client, then retry. If it returns the wallet identity, proceed.
- **Merchant URL** — the user must give you the merchant's base URL (e.g. `https://medusa.test.1stdigital.tech`). Do not invent one.
- **Curl or equivalent HTTP tool** — for every merchant request.
- **ACP only: merchant API key** — ACP merchants require a Bearer token issued out-of-band. If the merchant speaks ACP, ask the user for their API key before proceeding. UCP does not require one.
- **A way to generate unique IDs** — every HTTP call needs a unique `Request-Id` and (for POST/PUT) an `Idempotency-Key`. `uuidgen` does not exist on Windows. Detect once at session start, use for all calls. See [references/unique-ids.md](references/unique-ids.md) for the detection order and one-liners (Python is the most portable default).

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

### 4.3 Collect shipping + buyer info — memory-first

**Don't just ask.** A good personal shopper remembers the customer. Before asking the user to type anything, check available memory sources for their details and propose them for confirmation.

**Prefer the agent's native memory. Fall back to a skill-local file only if none is available.**

Most modern agent runtimes (Claude.ai, Claude Code, Cursor, OpenClaw, etc.) provide a memory mechanism — a native `Memory` tool, hierarchical `CLAUDE.md`, a user-profile KB, or similar. That memory is shared across skills and sessions, is the user's one source of truth, and is how they already manage their own info. The skill should use it, not duplicate it.

**Check, in order:**

1. **Conversation context** — has the user mentioned their name, email, or address earlier in this session? Use that.
2. **Agent's native memory system** — whatever the harness provides:
   - Claude Code: hierarchical `CLAUDE.md` (`~/.claude/CLAUDE.md`, project `CLAUDE.md`); native `Memory` tool if exposed.
   - Claude.ai: platform memory / user profile (read via the memory tool).
   - Other harnesses: their documented memory/profile mechanism.
   - If you can query this memory, do — ask for "shipping address", "email", "phone", "name" directly.
3. **Skill-local profile file** — `~/.claude/skills/fd-agentic-commerce/profile.md`. **Fallback only.** Use this when the harness has no memory system, or when the user explicitly asked to keep shopping details separate from their general agent memory. See [references/user-profile.md](references/user-profile.md) for the format.
4. **OS / git identity** — `git config user.email`, `git config user.name`, `$USER`. Useful for first-time users; weak signal, always confirm.
5. **The user is shopping for someone else** — if the request says "for my brother" / "for a friend", the recipient name may differ from the user's. The user's email stays as the order-confirmation email; only the shipping name/address is the recipient's.

**Present, don't ask** — once you have candidates, propose them:

> I have your details on file:
> - **Name**: Janno Jarv (from your profile)
> - **Email**: j.jaerv@1stdigital.com
> - **Ship to**: Järveoja tee 8, Veibi, 52220, Estonia
> - **Phone**: +372 5803 3175
>
> Use these, or give me new ones?

If you have **nothing** on file, then ask — but ask compactly in one message, not a bulleted list of six questions. Give context: "I'll need shipping details. Share a name + address block (one line is fine) and I'll parse it."

**After checkout succeeds**, offer to save or update if info was new:

> Want me to remember this address as "home" for next time? (yes/no)

If yes, **write to the same place you read from**:
- If the harness has a native memory tool, use it. (One source of truth for the user.)
- Only fall back to the skill-local `profile.md` when no harness memory is available.

Never save silently. Never save without explicit user consent. See [references/user-profile.md](references/user-profile.md) for the fallback file format.

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

Call the `authorizePayment` MCP tool. It expects a **full x402 `PaymentRequirementsResponse`** envelope as a JSON-serialized string — not a single `accepts[]` entry. Given the envelope, the wallet picks the best entry for you based on balances when `autoApprove: true`.

The merchant already gives you the envelope: it's `ucp.payment_handlers["xyz.fd.prism_payment"][i].config` on the checkout-session response (with `x402Version`, `resource`, and `accepts[]` at the top level). Pass it through verbatim.

Arguments:

- `paymentRequirementsResponseJson` — string-encoded JSON of the full response envelope (top-level `x402Version`, `resource: { url, description }`, `accepts: [...]`)
- `autoApprove` — `true` to let the wallet pick the best `accepts[]` entry from what you can actually pay

The tool returns an object containing a signed `paymentPayload` and the specific `paymentRequirements` entry the wallet chose. Use BOTH in the UCP complete call, or (for ACP) pass the whole returned object as the base64-encoded `credential.authorization`.

If `authorizePayment` errors with an insufficient-balance or no-matching-option reason, call `getMyInfo` to get the wallet address, call `getWalletOverview` to confirm what's held, and relay to the user: "to pay, you need <asset> on <network>; your wallet address is 0x…".

For envelope shape, network/asset tables, and atomic-unit conversion, see [references/payment-payload.md](references/payment-payload.md).

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
- [references/unique-ids.md](references/unique-ids.md) — cross-platform Request-Id / Idempotency-Key generation (Python, Node, PowerShell, `uuidgen`)
- [references/user-profile.md](references/user-profile.md) — persistent shopper profile strategy: prefer the agent runtime's native memory (CLAUDE.md, Memory tool, etc.); skill-local fallback file at `~/.claude/skills/fd-agentic-commerce/profile.md` is documented there. Read before asking for shipping; write only with user consent.
- [examples/gift-for-brother.md](examples/gift-for-brother.md) — full walkthrough of the acceptance-test scenario (UCP against the live demo store)
