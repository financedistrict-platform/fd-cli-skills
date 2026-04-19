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

All three top-level fields are required. **Do not** unwrap the envelope and pass a single `accepts[]` entry — the wallet's parser rejects that as malformed (missing top-level `resource`). Pass the whole `config` object through verbatim; if the merchant response omits an `error` field, that's fine — it's optional on the envelope.

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

## Selection is handled by the wallet when `autoApprove=true`

You don't usually pre-filter `accepts[]` yourself. Pass the **whole envelope** to `authorizePayment` with `autoApprove: true` and the wallet picks the best entry based on the available balances it knows about, applying the following priorities:

1. **Balance-first** — entries the wallet can actually pay from a held chain/asset are preferred over anything requiring a swap.
2. **Fewest hops** — native stablecoin on a chain it already holds is preferred over swap-then-pay.
3. **Gasless confirmation** — x402 uses EIP-3009, which is gasless for the payer on supported tokens (USDC, FDUSD); no gas funding needed on the payment chain.
4. **Testnet discipline** — testnet chains (`:84532`, `:421614`, `:11155111`, `:97`) only settle with testnet funds.

If you want to verify before calling, you can read balances with the `getWalletOverview` MCP tool (`chainKey` parameter) — but don't narrow `accepts[]` yourself before passing it in.

If no entry can be satisfied, `authorizePayment` returns an error with the reason. Call `getMyInfo` to get the wallet address and relay to the user so they know which token + network to fund.

## Signing

Call the `authorizePayment` MCP tool:

- `paymentRequirementsResponseJson` — string-encoded JSON of the full envelope (the whole Prism `config` object from the merchant's checkout-session response)
- `autoApprove` — `true` (recommended) to let the wallet pick the best `accepts[]` entry itself

Returns an object containing:
- `paymentPayload` — signed EIP-3009 authorization (opaque — do not modify)
- `paymentRequirements` — the single `accepts[]` entry the wallet picked and signed against

## Including both payload + requirements in `complete`

Per the UCP spec and Prism handler, the `complete` request's `credential` must carry BOTH:

- `paymentPayload` — what the wallet returned
- `paymentRequirements` — the specific `accepts[]` entry echoed back by the wallet (NOT the whole envelope)

This is how the facilitator reconstructs and verifies the EIP-712 signature. Don't drop `paymentRequirements` and don't substitute the envelope.
