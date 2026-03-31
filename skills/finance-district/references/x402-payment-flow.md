# x402 Payment Protocol Flow

x402 is an open payment protocol ([spec](https://github.com/coinbase/x402/blob/main/specs/x402-specification-v2.md)) that enables paid API access using crypto. Resource servers signal payment requirements via HTTP 402, and clients pay by signing an on-chain authorization.

The `fdx` CLI's role is specifically **signing the payment authorization**. The agent handles HTTP communication directly.

## Protocol Flow

Three parties are involved: your agent (client), the resource server (API), and a facilitator (verifies and settles payments on-chain).

### Step 1: Request the resource

Call the resource server normally (e.g. via curl). If payment is required, the server responds with HTTP `402 Payment Required` and a JSON body containing:

- `x402Version` — protocol version (currently `2`)
- `accepts[]` — array of acceptable payment methods, each with: `scheme`, `network` (CAIP-2 format, e.g. `eip155:8453` for Base), `amount` (atomic units), `asset` (token contract address), `payTo` (recipient address), `maxTimeoutSeconds`
- `resource` — describes the protected resource (url, description, mimeType)

### Step 2: Sign the payment authorization

Pass the payment requirements to `fdx wallet authorizePayment`. The CLI:

1. Selects the best payment option from the `accepts[]` array (matching available balance and supported networks)
2. Signs an EIP-3009 `transferWithAuthorization` using the wallet's key
3. Returns a `PaymentPayload` containing the signed authorization

Use `fdx wallet authorizePayment --help` for parameter details.

### Step 3: Retry with payment

Retry the original request with the `PaymentPayload` attached. The resource server forwards it to the facilitator, which verifies the signature, simulates the transaction, and settles on-chain. On success, the resource server returns the requested content along with settlement details (including a transaction hash verifiable on a block explorer).

How the payload is attached depends on the resource server's expectations — typically via a request header or body field. Check the resource server's documentation.

## Fallback: getX402Content

If you do not have access to HTTP tools (curl, wget, or equivalent), use `fdx wallet getX402Content` as a fallback. It bundles the entire flow — initial request, authorization signing, and retry — into a single command.

Use `fdx wallet getX402Content --help` for parameter details.

## Key Concepts

- **Facilitator**: A third-party service that verifies payment authorizations and settles them on-chain. The agent never interacts with the facilitator directly — the resource server handles that
- **EIP-3009**: A token standard for gasless transfers via signed authorization. The payer signs a message, and the facilitator executes the transfer on-chain — no gas needed from the payer
- **Atomic units**: Payment amounts in `accepts[]` are in atomic units (e.g. 1000000 = 1 USDC with 6 decimals)
- **CAIP-2 network IDs**: Chain identifiers like `eip155:8453` (Base), `eip155:1` (Ethereum), `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` (Solana mainnet)

## Before Paying

- Check wallet balance on the payment chain before authorizing
- Inform the human about the payment amount, especially for unfamiliar endpoints
- The `accepts[]` array may offer multiple payment options across different chains and assets — the CLI selects the best match automatically
