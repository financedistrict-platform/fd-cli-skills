# x402 Payment Requirements — Selecting & Signing

The checkout session's `ucp.payment_handlers["xyz.fd.prism_payment"][i].config.accepts[]` array holds the x402 payment options the merchant will accept. Each entry **is** the `paymentRequirements` object you pass to `fdx wallet authorizePayment`.

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

## Selecting which entry to sign

Priority:

1. **Balance-first** — pick an entry where the wallet has `amount` available on `network`. Check with `fdx wallet getWalletOverview --chainKey <chain>` before signing.
2. **Fewest hops** — prefer native stablecoin on a chain you already hold balance on over swapping first.
3. **Gasless confirmation** — x402 uses EIP-3009, which is gasless for the payer on supported tokens (USDC, FDUSD). No gas funding needed on the payment chain.
4. **Testnet discipline** — testnet chains (`:84532`, `:421614`, `:11155111`, `:97`) only settle with testnet funds; do not attempt on mainnet and vice versa.

If no entry matches the wallet's current balance, tell the user which token + network they need to fund, showing their FD Agent Wallet address from `fdx wallet getMyInfo`.

## Signing

```bash
fdx wallet authorizePayment --requirements '<JSON of one accepts[] entry>'
```

Returns a `PaymentPayload` — an object containing the signed EIP-3009 authorization. Do not modify it. Submit it verbatim inside the `complete` request's `credential.paymentPayload`.

## MCP alternative

If your agent runs inside an MCP host with `mcp.fd.xyz` attached, call the `authorizePayment` tool with the same `paymentRequirements` argument. The returned object slots into the same place.

## Including both payload + requirements

Per the UCP spec and Prism handler, the `complete` request's `credential` must carry BOTH:

- `paymentPayload` — what the wallet returned
- `paymentRequirements` — the exact `accepts[]` entry the wallet signed against

This is how the facilitator reconstructs and verifies the EIP-712 signature. Don't drop `paymentRequirements`.
