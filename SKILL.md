---
name: finance-district
description: Manage agent crypto wallets and merchant payments via the Finance District platform (fdx CLI). Use when the user mentions wallets, tokens, crypto, DeFi, or merchant payments. Wallet operations — create/setup wallet, send/receive/transfer tokens, swap/trade/exchange crypto, check balance/portfolio, token prices, earn yield, stake, deposit/withdraw from DeFi vaults, x402 payments, pay for API services, fund wallet, bridge tokens, supported chains. Prism merchant operations — Points of Service (POS), create/list/manage API keys, settlement wallets, payment history, list payments, earnings, revenue. Trigger phrases include "set up wallet", "send ETH", "swap tokens", "check my balance", "show portfolio", "how much do I have", "earn yield", "buy/sell crypto", "use fdx", "create API key", "point of service", "payment history", "Prism". Covers EVM chains (Ethereum, Base, Polygon, Arbitrum, Optimism), Solana, and Bitcoin. Do NOT use for general payment processing unrelated to Finance District or fdx.
user-invocable: true
disable-model-invocation: false
allowed-tools:
  - "Bash(fdx *)"
  - "Bash(fdx)"
  - "Bash(npm install -g @financedistrict/fdx*)"
  - "Bash(npx @financedistrict/fdx*)"
  - "Bash(which fdx)"
metadata:
  openclaw:
    emoji: "🏦"
    requires:
      bins:
        - fdx
    install:
      - type: node
        package: "@financedistrict/fdx"
---

# Finance District

The `fdx` CLI is the Finance District platform CLI with two service areas:

- **Wallet** (`fdx wallet`) — crypto wallet management, token transfers, DEX swaps, DeFi yield, and x402 payments across EVM chains, Solana, and Bitcoin
- **Prism** (`fdx prism`) — payment gateway purpose-built for agentic commerce. Exposes tools to manage merchant accounts, Points of Service, API keys, and settlement wallets

## 1. Prerequisites

The `fdx` CLI must be available. Check with:

```bash
which fdx
```

If `fdx` is not installed, ask the human to install it with `npm install -g @financedistrict/fdx`, or offer to run the installation with their approval.

## 2. Tool Discovery with --help

The `fdx` CLI has comprehensive built-in help. **Use it as your primary source for tool parameters, input formats, and output structures.** Do not guess parameters — always check `--help` first.

```bash
fdx wallet                        # list all wallet tools
fdx prism                         # list all Prism tools
fdx wallet <method> --help        # full details for a specific wallet tool
fdx prism <method> --help         # full details for a specific Prism tool
```

All commands return structured JSON.

## 3. Authentication

Authentication uses email OTP — no browser required.

### Autonomous Path (with email inbox access)

If you have access to an email inbox (e.g. via AgentMail.to or another email API tool):

1. Create or use an existing email inbox
2. Register: `fdx register --email <inbox-email>`
3. Read the 8-digit OTP from the inbox
4. Verify: `fdx verify --code <OTP>`
5. Confirm: `fdx status`

### Human-Assisted Path

If you do not have email inbox access:

1. Ask the human for their email address
2. New users: `fdx register --email <email>` / Returning users: `fdx login --email <email>`
3. Tell the human: "Check your email for an 8-digit code from Finance District and share it with me."
4. Verify: `fdx verify --code <OTP>`
5. Confirm: `fdx status`

For details on autonomous email setup, register vs login, and credential storage, see [references/authentication.md](references/authentication.md).

## 4. Pre-Operation Checklist

Before any wallet operation, always:

1. **Check authentication**: `fdx status` — if not authenticated, complete Section 3 first
2. **Check balances**: `fdx wallet getWalletOverview --chainKey <chain>` — verify sufficient token balance AND native gas token balance
3. **Confirm with the human** before executing irreversible operations (transfers, swaps, deposits) — state the amount, asset, chain, and recipient clearly
4. **Do not assume** — if any detail is ambiguous (chain, token, amount, recipient), ask for clarification

If balance is insufficient: suggest funding the wallet, or swapping into the needed token/gas token if the user holds other tokens on that chain.

## 5. Wallet Workflows

Wallet operations work across EVM chains, Solana, and Bitcoin. Basic operations (balance, send, receive) work on all chains. Swap, DeFi yield, and x402 payments are available on EVM and Solana but not Bitcoin. Always complete the pre-operation checklist (Section 4) before executing any workflow below.

### Checking wallet state

Use `getWalletOverview` to see balances across all chains or for a specific chain. Use `getMyInfo` for the user's profile and wallet addresses. Use `getTokenPrice` to look up current token prices.

### Sending tokens

Confirm recipient, amount, chain, and asset with the human → execute transfer → verify with account activity.

Supports ENS (.eth), SNS (.sol), and Unstoppable Domains — resolve names with `resolveNameService` or pass them directly to the transfer tool. Always double-check that the chain matches the recipient's expected chain — sending on the wrong chain may result in lost funds.

### Swapping tokens

Get a quote first (default mode is QuoteOnly) → present the quote to the human → execute with `--mode Execute`.

For large swaps, set slippage explicitly. Common patterns: buy gas tokens by swapping stablecoins into the chain's native token, rebalance portfolio across tokens, or prepare tokens for a DeFi deposit.

### Earning yield (DeFi)

Discover strategies (filter by chain, token, protocol) → present risk level and APY to the human → get human's explicit approval → deposit → track position via wallet overview.

To withdraw, you need the vault token address from the original deposit or from the wallet overview. Always let the human make the final decision on DeFi deposits — these carry smart contract risk.

### Paying for services (x402)

x402 is an open payment protocol where resource servers respond with HTTP `402 Payment Required` containing payment requirements. The agent's role is to handle the HTTP communication and use the CLI only for payment authorization signing.

**Primary workflow** (when you have HTTP access via curl, wget, or equivalent):

1. Call the resource server → receive 402 with `PaymentRequired` JSON (contains `accepts[]` with payment options)
2. Pass the payment requirements to `fdx wallet authorizePayment` → receive signed `PaymentPayload`
3. Retry the request with the payment payload attached

**Fallback** (when you do not have HTTP tools available): use `fdx wallet getX402Content` which bundles the entire flow into a single command.

For the full x402 protocol flow and implementation details, see [references/x402-payment-flow.md](references/x402-payment-flow.md).

### Funding the wallet

The wallet is funded by direct token transfer to the wallet address (from another wallet or exchange) or through the Finance District web dashboard. Get the wallet address from `getWalletOverview`.

For detailed chain capabilities, safety notes, and advanced patterns, see [references/operations.md](references/operations.md).

## 6. Prism Workflows

Prism is the Finance District payment gateway purpose-built for agentic commerce. Use `fdx prism` to list all available tools. Always complete the pre-operation checklist (Section 4) before executing any workflow below.

### Setting up as a merchant

Set account type (Personal/Business) → create a Point of Service → configure accepted assets and networks → set up settlement wallets → create API keys.

### Managing payments and earnings

Use the payment and earnings tools to view transaction history, individual payment details (including blockchain tx hash and settlement breakdown), and earnings summaries over time.

### Points of Service

A Point of Service (PoS) defines your merchant configuration — accepted assets, networks, and settlement wallets. Most Prism tools default to your active PoS. Only pass `--posId` when managing multiple configurations.

For detailed Prism workflow patterns, see [references/prism-operations.md](references/prism-operations.md).

## 7. Principles

- **Use --help liberally**: When unsure about a tool's parameters or behavior, check `--help` before guessing
- **Smart Accounts**: Use existing Smart Account addresses as `--fromAccountAddress` in transfers, swaps, and yield operations
- **Confirm before executing**: Any operation that moves funds requires human confirmation first
- **Discover available tools**: Run `fdx wallet` or `fdx prism` with no arguments to see what's available

## 8. Troubleshooting

| Error                                     | Action                                                                  |
| ----------------------------------------- | ----------------------------------------------------------------------- |
| "not authenticated"                       | Run `fdx login --email <email>` then `fdx verify --code <OTP>`          |
| "token expired" with refresh token        | Auto-refreshes on next call — no action needed                          |
| "SESSION_EXPIRED" / "AUTH_REFRESH_FAILED" | Refresh token expired — run `fdx login` again                           |
| "Insufficient balance"                    | Check balance with `getWalletOverview`; fund or swap                    |
| "No liquidity"                            | Try smaller amount or different token pair                              |
| "tool not found"                          | Run `fdx wallet` or `fdx prism` to list available tools; check spelling |
| "provider not found"                      | Complete Prism onboarding — set account type with `updateAccountType`   |

For diagnostic commands and issue reporting, see [references/troubleshooting.md](references/troubleshooting.md).

## 9. Reference Files

Consult these for deeper guidance on specific topics. **Do not load them all — only read the one relevant to your current task.**

- [references/authentication.md](references/authentication.md) — autonomous email setup, register vs login, credential storage, logout
- [references/operations.md](references/operations.md) — chain capability matrix, safety patterns, advanced wallet workflows
- [references/prism-operations.md](references/prism-operations.md) — merchant setup patterns, PoS configuration, API key and settlement wallet management
- [references/x402-payment-flow.md](references/x402-payment-flow.md) — x402 protocol flow, authorizePayment usage, and getX402Content fallback
- [references/troubleshooting.md](references/troubleshooting.md) — detailed error reference, diagnostic commands, issue reporting
