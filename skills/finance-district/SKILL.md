---
name: finance-district
description: Manage crypto wallets and merchant payments via the Finance District platform (fdx CLI). Use when the user mentions wallets, tokens, crypto, DeFi, or merchant payments. Wallet operations — create/setup wallet, send/receive/transfer tokens, swap/trade/exchange crypto, check balance/portfolio, token prices, earn yield, stake, deposit/withdraw from DeFi vaults, x402 payments, pay for API services, fund wallet, bridge tokens, supported chains. Prism merchant operations — Points of Service (POS), create/list/manage API keys, settlement wallets, payment history, list payments, earnings, revenue. Trigger phrases include "set up wallet", "send ETH", "swap tokens", "check my balance", "show portfolio", "how much do I have", "earn yield", "buy/sell crypto", "use fdx", "create API key", "point of service", "payment history", "Prism". Covers EVM chains (Ethereum, Base, Polygon, Arbitrum, Optimism), Solana, and Bitcoin. Do NOT use for general payment processing unrelated to Finance District or fdx.
version: 1.0.0
homepage: https://fd.xyz
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
- **Prism** (`fdx prism`) — merchant platform for payment processing, API key management, and Points of Service configuration

All commands return structured JSON.

## 1. Installation & Setup

Check if `fdx` is installed:

```bash
which fdx
```

If not found, install it:

```bash
npm install -g @financedistrict/fdx
```

## 2. Onboarding

Authentication uses email OTP — no browser required. There are two paths depending on whether you have email inbox access.

### Autonomous Path (with email inbox access)

If you have access to an email inbox (e.g. via AgentMail.to or another email API tool), you can complete onboarding without human assistance:

1. Create or use an existing email inbox
2. Register: `fdx register --email <inbox-email>`
3. Read the 8-digit OTP from the inbox
4. Verify: `fdx verify --code <OTP>`
5. Confirm: `fdx status`

For guided setup, use `fdx wallet onboardingAssistant --question "How do I get started?"` — it walks through each step interactively.

For setting up autonomous email access, see `references/onboarding.md`.

### Human-Assisted Path

If you do not have email inbox access:

1. Ask the human for their email address
2. Register or login:
   - New users: `fdx register --email <email>`
   - Returning users: `fdx login --email <email>`
3. Tell the human: "Check your email for an 8-digit code from Finance District and share it with me."
4. Verify: `fdx verify --code <OTP>`
5. Confirm: `fdx status`

To log out and clear stored credentials: `fdx logout`

## 3. Wallet Pre-Operation Checklist

Before any wallet operation, always:

1. **Check authentication**: `fdx status` — if not authenticated, complete onboarding first
2. **Check balances**: `fdx wallet getWalletOverview --chainKey <chain>` — verify sufficient token balance AND gas token balance for the target chain
3. **Confirm with the human** before executing irreversible operations (transfers, swaps, deposits) — state the amount, asset, chain, and recipient clearly
4. **Do not assume** — if any detail is ambiguous (chain, token, amount, recipient), ask for clarification

## 4. Wallet Operations

Wallet operations work across all popular EVM chains, Solana, and Bitcoin. Basic operations (balance, send, receive) work on all chains. Swap, DeFi yield, and x402 payments are available on EVM chains and Solana but not Bitcoin. Use `fdx wallet` without arguments to list all available methods. Use `fdx wallet <method> --help` for parameter details.

### Check Wallet State

```bash
fdx wallet getWalletOverview                          # all chains
fdx wallet getWalletOverview --chainKey ethereum      # specific chain
fdx wallet getMyInfo                                  # user profile and wallet addresses
fdx wallet getTokenPrice --token ETH                  # token price lookup
fdx wallet getAccountActivity --accountAddress <addr> --chainKey <chain>  # tx history
```

### Send Tokens

```bash
fdx wallet transferTokens \
  --toAddress <address-or-ENS-name> \
  --amount <amount> \
  --asset <symbol> \
  --chainKey <chain>
```

Supports ENS (.eth), SNS (.sol), and Unstoppable Domains. Resolve names with `fdx wallet resolveNameService --nameOrAddress "name.eth"`.

### Swap Tokens

```bash
fdx wallet swapTokens \
  --chainKey <chain> \
  --tokenIn <symbol> \
  --tokenOut <symbol> \
  --amount <amount> \
  --mode Execute
```

Default mode is `QuoteOnly` (returns quote without executing). Pass `--mode Execute` to actually swap. For large swaps, set `--maxSlippageBps` explicitly (100 = 1%).

### Earn Yield (DeFi)

```bash
# Discover strategies
fdx wallet discoverYieldStrategies --chainKey <chain> --token <symbol>

# Deposit
fdx wallet depositForYield --strategyId <id> --fromAccountAddress <addr> --token <symbol> --amount <amount> --chainKey <chain>

# Withdraw
fdx wallet withdrawFromYield --vaultTokenAddress <vault> --underlyingToken <symbol> --withdrawAmount <amount> --fromAccountAddress <addr> --chainKey <chain>
```

Always present risk level and APY to the human before depositing.

### Pay for Services (x402)

Two paths depending on resource type:

**HTTP path** — when the resource is an HTTP endpoint returning `402 Payment Required`:

```bash
fdx wallet getX402Content --url <x402-endpoint>
```

**MCP path** — when an MCP tool returns `isError: true` with `x402Version` and `accepts` array:
1. Call the paid tool normally → receive PaymentRequired error
2. Pass `content[0].text` (raw JSON string) to `authorizePayment` tool on Agent Wallet MCP Server
3. Retry the tool with `_meta["x402/payment"]` set to `authorization.paymentPayload` from step 2

Supports multi-chain and multi-asset x402 payments. Inform the human about payment amounts before executing. For detailed protocol flow, see `references/x402-payment-flow.md`.

### Fund the Wallet

The wallet is funded by direct token transfer to the wallet address (from another wallet or exchange) or through the Finance District web dashboard. Get the wallet address from `fdx wallet getWalletOverview`.

For detailed operation guides, see `references/operations.md`.

## 5. Prism Platform Operations

Prism is the Finance District merchant platform — manage Points of Service, payments, API keys, and settlement wallets. Use `fdx prism` without arguments to list all available tools. Use `fdx prism <method> --help` for parameter details.

### Points of Service

A Point of Service (PoS) is your merchant configuration — it defines which assets and networks you accept. Most Prism tools use `--posId` to target a specific PoS. If omitted, your default active PoS is used.

```bash
fdx prism listPointsOfService                     # list all PoS
fdx prism getPointOfServiceDetails                 # default PoS details
fdx prism createPointOfService --name "My Store" \
  --configurationJson '{"assets":{"acceptedAssets":[]},"networks":{"acceptedNetworks":[]}}'
```

### View Payments & Earnings

```bash
fdx prism listPayments                             # all payments
fdx prism getPaymentDetails --paymentId <id>       # single payment details
fdx prism getEarnings                              # total earnings summary
fdx prism getRecentPayments --timeframe 30d        # recent activity
```

### Manage API Keys

```bash
fdx prism listApiKeys
fdx prism manageApiKey --action create --name "Production Key"
fdx prism manageApiKey --action disable --apiKeyId <id>
fdx prism manageApiKey --action delete --apiKeyId <id>
```

The secret is returned only once on creation — save it immediately.

### Manage Settlement Wallets

```bash
fdx prism listWallets
fdx prism manageWallet --action create \
  --chainId 8453 --asset USDC --address 0x... --label "Base USDC"
```

### Provider Profile

```bash
fdx prism getProviderInfo                          # profile + supported chains/tokens
fdx prism updateAccountType --accountType Business
```

For detailed Prism operations guide, see `references/prism-operations.md`.

## 6. Decision-Making Principles

- **Balance check first**: Before sends, swaps, or deposits — check both the token balance and the native gas token balance on that chain
- **Insufficient gas?** Suggest the human fund gas tokens, or swap into the gas token if they hold other tokens on that chain
- **Insufficient token balance?** Suggest funding the wallet or swapping from another token
- **Discover available tools**: Run `fdx wallet` with no arguments to see all methods
- **Parameter help**: Run `fdx wallet <method> --help` for any method's parameters
- **Smart Accounts**: Use existing Smart Account addresses as `--fromAccountAddress` in transfers, swaps, and yield operations
- **Prism tool discovery**: Run `fdx prism` with no arguments to see all merchant tools
- **Point of Service context**: Most Prism tools default to your active PoS — only pass `--posId` when managing multiple configurations

## 7. Troubleshooting & Support

| Error | Action |
|-------|--------|
| "not authenticated" | Run `fdx login --email <email>` then `fdx verify --code <OTP>` |
| "token expired" with refresh token | Auto-refreshes on next call — no action needed. Tokens stored in OS credential store (macOS Keychain, Linux libsecret, Windows DPAPI); fallback: `~/.fdx/auth.json` |
| "SESSION_EXPIRED" / "AUTH_REFRESH_FAILED" | Refresh token expired — run `fdx login` again |
| "Insufficient balance" | Check balance with `getWalletOverview`; fund or swap |
| "No liquidity" | Try smaller amount or different token pair |
| "tool not found" (Prism) | Run `fdx prism` to list available tools; check spelling |
| "provider not found" | Complete Prism onboarding — set account type with `updateAccountType` |

For conceptual questions about wallet safety, Prism configuration, key management, or fees:

```bash
fdx wallet helpNarrative --question "How does key delegation work?"
```

When something goes wrong that you cannot resolve, report it:

```bash
fdx wallet reportIssue \
  --title "<short summary>" \
  --description "<what happened, steps to reproduce>"
```

Check CLI version for diagnostics: `fdx wallet getAppVersion`

For detailed troubleshooting, see `references/troubleshooting.md`.
