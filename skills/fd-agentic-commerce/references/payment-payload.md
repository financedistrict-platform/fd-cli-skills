# x402 Payment Requirements — Selecting & Signing

The merchant's checkout session returns a Prism config at `ucp.payment_handlers["xyz.fd.prism_payment"][i].config`. That object **is** the x402 `PaymentRequirementsResponse` envelope — the exact shape the wallet's `authorizePayment` tool expects as its `paymentRequirementsResponseJson` argument.

## Envelope shape (what you pass to authorizePayment)

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://merchant.example/ucp/checkout-sessions/cart_01…",
    "description": "Purchase from <merchant>"
  },
  "accepts": [ /* one or more PaymentRequirements entries — see below */ ]
}
```

All three top-level fields are required (`x402Version`, `resource`, `accepts`). `error` is optional.

**Two common ways to get this wrong, both rejected by the wallet:**

1. **Unwrapping** — passing just one `accepts[]` entry as if it were the whole envelope. The wallet needs the envelope.
2. **Rebuilding** — narrowing `accepts[]` to the asset you want and constructing a fresh envelope around it without copying `resource` over. The wallet needs `resource` at the top level.

**Always clone the merchant's `config` object, then replace only `accepts[]`.** Don't rebuild from scratch. If the merchant omits `error`, that's fine — it's optional.

## Anatomy of one `accepts[]` entry

```json
{
  "scheme":  "exact",
  "network": "eip155:84532",
  "asset":   "0x036cbd53842c5426634e7929541ec2318f3dcf7e",
  "amount":  "290000",
  "payTo":   "0x40a01003f7543a3a3ee64ffb05504173bdb1c4fd",
  "maxTimeoutSeconds": 300,
  "extra":   { "name": "USDC", "version": "2" }
}
```

- `scheme: "exact"` — pay exactly `amount` atomic units. Currently the only scheme.
- `network` — CAIP-2 chain id. See table below.
- `asset` — ERC-20 token contract on `network`.
- `amount` — atomic units. Decimals depend on the token (see table).
- `payTo` — merchant settlement address.
- `maxTimeoutSeconds` — signature validity window. Sign and submit well within it.
- `extra.name` / `extra.version` — EIP-712 domain name + version for the signature.

## Common CAIP-2 network ids (testnets)

| CAIP-2 id | Chain | Notes |
| --- | --- | --- |
| `eip155:84532` | Base Sepolia | USDC 6-dp |
| `eip155:421614` | Arbitrum Sepolia | USDC / FDUSD |
| `eip155:11155111` | Ethereum Sepolia | USDC / FDUSD |
| `eip155:97` | BNB testnet | FDUSD 18-dp |
| `eip155:8453` | Base mainnet | Production |
| `eip155:42161` | Arbitrum One | Production |
| `eip155:1` | Ethereum mainnet | Production |

## Token decimals seen in Prism

| Asset (by name) | Typical decimals | Atomic for $1 |
| --- | --- | --- |
| USDC | 6 | `1000000` |
| USD Coin | 6 | `1000000` |
| First Digital USD (FDUSD) on BSC/ETH/Arbitrum | 18 | `1000000000000000000` |

Divide `amount` by `10^decimals` to get the human number. If you're unsure of decimals, read `extra.name` and infer from the table, or ask the user to confirm.

## Selection — chosen by the skill, narrowed into the envelope before signing

The skill (not the wallet) picks the preferred asset in §4.3 of SKILL.md, based on:

1. **Currency match** — EUR cart prefers EURC; USD cart prefers USDC / FDUSD. Same-currency pairs carry the least FX slippage (and zero, for pegged pairs).
2. **Balance-first** — only entries the wallet can actually cover with a held balance are considered.
3. **Fallback chain** — walk a preference list (e.g. EUR → EURC → USDC → FDUSD → USDT) until an asset with sufficient balance is found.
4. **Testnet discipline** — testnet chains (`:84532`, `:421614`, `:11155111`, `:97`) only settle with testnet funds; mainnet is disjoint. Match the merchant's testnet/mainnet stance.

Once chosen, **narrow `accepts[]` in the envelope** to just the matching entry (or entries if multiple chains would work) before calling `authorizePayment`. Keep every other top-level field (`x402Version`, `resource`, `error`, ...) exactly as the merchant sent it. This gives the wallet a valid envelope with an unambiguous signing target.

If the narrowed list is empty (shouldn't happen if §4.3 did its job), pass the full envelope instead and let the wallet's `autoApprove` fall back to whatever it can pay.

If no entry can be satisfied, `authorizePayment` returns an error with the reason. Call `getMyInfo` to get the wallet address and relay to the user so they know which token + network to fund.

## Signing

Call the `authorizePayment` MCP tool:

- `paymentRequirementsResponseJson` — string-encoded JSON of the **narrowed envelope** (full envelope shape; `accepts[]` filtered to the asset chosen in §4.3)
- `autoApprove` — `true` (recommended). With a narrowed `accepts[]`, there's usually just one valid entry; `autoApprove` signs it without further prompting.

Returns an object containing:
- `paymentPayload` — signed EIP-3009 authorization (opaque — do not modify)
- `paymentRequirements` — the single `accepts[]` entry the wallet signed against (should match what you narrowed to)

## Including both payload + requirements in `complete`

Per the UCP spec and Prism handler, the `complete` request's `credential` must carry BOTH:

- `paymentPayload` — what the wallet returned
- `paymentRequirements` — the specific `accepts[]` entry echoed back by the wallet (NOT the whole envelope)

This is how the facilitator reconstructs and verifies the EIP-712 signature. Don't drop `paymentRequirements` and don't substitute the envelope.
