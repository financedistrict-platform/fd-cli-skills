---
name: finance-district
description: This skill should be used when the user asks about Finance District tools, the fdx CLI, crypto wallets, sending or receiving tokens, swapping or trading crypto, checking balances or portfolio, earning DeFi yield, staking, paying for API services, x402 payments, merchant payments, Points of Service, API keys, settlement wallets, or the Prism platform. Trigger on phrases like "set up wallet", "create a wallet", "send ETH", "swap tokens", "check my balance", "show my portfolio", "how much do I have", "earn yield", "deposit into DeFi", "withdraw from vault", "pay for service", "pay for API", "fund my wallet", "what chains are supported", "bridge tokens", "buy/sell crypto", "use fdx", "Finance District", "list payments", "payment history", "create API key", "manage API keys", "point of service", "POS", "merchant", "earnings", "revenue", "prism", or "Prism Platform". Covers the Finance District platform CLI (fdx) for wallet management, token operations, DeFi, payments across EVM chains, Solana, and Bitcoin, and the Prism merchant platform for payment processing, API key management, and Points of Service configuration. Do NOT use for general payment processing unrelated to Finance District.
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

- **Wallet** (`fdx wallet call`) — crypto wallet management, token transfers, DEX swaps, DeFi yield, and x402 payments across EVM chains, Solana, and Bitcoin
- **Prism** (`fdx prism call`) — merchant platform for payment processing, API key management, and Points of Service configuration

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

For guided setup, use `fdx wallet call onboardingAssistant --question "How do I get started?"` — it walks through each step interactively.

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
2. **Check balances**: `fdx wallet call getWalletOverview --chainKey <chain>` — verify sufficient token balance AND gas token balance for the target chain
3. **Confirm with the human** before executing irreversible operations (transfers, swaps, deposits) — state the amount, asset, chain, and recipient clearly
4. **Do not assume** — if any detail is ambiguous (chain, token, amount, recipient), ask for clarification

## 4. Wallet Operations

Wallet operations work across all popular EVM chains, Solana, and Bitcoin. Basic operations (balance, send, receive) work on all chains. Swap, DeFi yield, and x402 payments are available on EVM chains and Solana but not Bitcoin. Use `fdx wallet call` without arguments to list all available methods. Use `fdx wallet call <method> --help` for parameter details.

### Check Wallet State

```bash
fdx wallet call getWalletOverview                          # all chains
fdx wallet call getWalletOverview --chainKey ethereum      # specific chain
fdx wallet call getMyInfo                                  # user profile and wallet addresses
fdx wallet call getTokenPrice --token ETH                  # token price lookup
fdx wallet call getAccountActivity --accountAddress <addr> --chainKey <chain>  # tx history
```

### Send Tokens

```bash
fdx wallet call transferTokens \
  --toAddress <address-or-ENS-name> \
  --amount <amount> \
  --asset <symbol> \
  --chainKey <chain>
```

Supports ENS (.eth), SNS (.sol), and Unstoppable Domains. Resolve names with `fdx wallet call resolveNameService --nameOrAddress "name.eth"`.

### Swap Tokens

```bash
fdx wallet call swapTokens \
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
fdx wallet call discoverYieldStrategies --chainKey <chain> --token <symbol>

# Deposit
fdx wallet call depositForYield --strategyId <id> --fromAccountAddress <addr> --token <symbol> --amount <amount> --chainKey <chain>

# Withdraw
fdx wallet call withdrawFromYield --vaultTokenAddress <vault> --underlyingToken <symbol> --withdrawAmount <amount> --fromAccountAddress <addr> --chainKey <chain>
```

Always present risk level and APY to the human before depositing.

### Pay for Services (x402)

```bash
fdx wallet call getX402Content --url <x402-endpoint>
```

Supports multi-chain and multi-asset x402 payments. Inform the human about payment amounts before executing.

### Fund the Wallet

The wallet is funded by direct token transfer to the wallet address (from another wallet or exchange) or through the Finance District web dashboard. Get the wallet address from `fdx wallet call getWalletOverview`.

For detailed operation guides, see `references/operations.md`.

## 5. Prism Platform Operations

Prism is the Finance District merchant platform — manage Points of Service, payments, API keys, and settlement wallets. Use `fdx prism call` without arguments to list all available tools. Use `fdx prism call <method> --help` for parameter details.

### Points of Service

A Point of Service (PoS) is your merchant configuration — it defines which assets and networks you accept. Most Prism tools use `--posId` to target a specific PoS. If omitted, your default active PoS is used.

```bash
fdx prism call listPointsOfService                     # list all PoS
fdx prism call getPointOfServiceDetails                 # default PoS details
fdx prism call createPointOfService --name "My Store" \
  --configurationJson '{"assets":{"acceptedAssets":[]},"networks":{"acceptedNetworks":[]}}'
```

### View Payments & Earnings

```bash
fdx prism call listPayments                             # all payments
fdx prism call getPaymentDetails --paymentId <id>       # single payment details
fdx prism call getEarnings                              # total earnings summary
fdx prism call getRecentPayments --timeframe 30d        # recent activity
```

### Manage API Keys

```bash
fdx prism call listApiKeys
fdx prism call manageApiKey --action create --name "Production Key"
fdx prism call manageApiKey --action disable --apiKeyId <id>
fdx prism call manageApiKey --action delete --apiKeyId <id>
```

The secret is returned only once on creation — save it immediately.

### Manage Settlement Wallets

```bash
fdx prism call listWallets
fdx prism call manageWallet --action create \
  --chainId 8453 --asset USDC --address 0x... --label "Base USDC"
```

### Provider Profile

```bash
fdx prism call getProviderInfo                          # profile + supported chains/tokens
fdx prism call updateAccountType --accountType Business
```

For detailed Prism operations guide, see `references/prism-operations.md`.

## 6. Decision-Making Principles

- **Balance check first**: Before sends, swaps, or deposits — check both the token balance and the native gas token balance on that chain
- **Insufficient gas?** Suggest the human fund gas tokens, or swap into the gas token if they hold other tokens on that chain
- **Insufficient token balance?** Suggest funding the wallet or swapping from another token
- **Discover available tools**: Run `fdx wallet call` with no arguments to see all methods
- **Parameter help**: Run `fdx wallet call <method> --help` for any method's parameters
- **Smart Accounts**: Deploy with `deploySmartAccount`, add co-owners with `manageSmartAccountOwnership`. Use Smart Account addresses as `--fromAccountAddress` in transfers, swaps, and yield operations
- **Prism tool discovery**: Run `fdx prism call` with no arguments to see all merchant tools
- **Point of Service context**: Most Prism tools default to your active PoS — only pass `--posId` when managing multiple configurations

## 7. Troubleshooting & Support

| Error | Action |
|-------|--------|
| "not authenticated" | Run `fdx login --email <email>` then `fdx verify --code <OTP>` |
| "token expired" with refresh token | Auto-refreshes on next call — no action needed |
| "SESSION_EXPIRED" / "AUTH_REFRESH_FAILED" | Refresh token expired — run `fdx login` again |
| "Insufficient balance" | Check balance with `getWalletOverview`; fund or swap |
| "No liquidity" | Try smaller amount or different token pair |
| "tool not found" (Prism) | Run `fdx prism call` to list available tools; check spelling |
| "provider not found" | Complete Prism onboarding — set account type with `updateAccountType` |

For conceptual questions about wallet safety, Prism configuration, key management, or fees:

```bash
fdx wallet call helpNarrative --question "How does key delegation work?"
```

When something goes wrong that you cannot resolve, report it:

```bash
fdx wallet call reportIssue \
  --title "<short summary>" \
  --description "<what happened, steps to reproduce>"
```

Check CLI version for diagnostics: `fdx wallet call getAppVersion`

For detailed troubleshooting, see `references/troubleshooting.md`.
