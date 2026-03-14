---
name: agent-wallet
description: This skill should be used when the user asks about crypto wallets, sending or receiving tokens, swapping or trading crypto, checking balances or portfolio, earning DeFi yield, staking, or paying for API services. Trigger on phrases like "set up wallet", "create a wallet", "send ETH", "swap tokens", "check my balance", "show my portfolio", "how much do I have", "earn yield", "deposit into DeFi", "withdraw from vault", "pay for service", "pay for API", "fund my wallet", "what chains are supported", "bridge tokens", or "buy/sell crypto". Covers the Finance District Agent Wallet CLI (fdx) for wallet management across EVM chains, Solana, and Bitcoin, including onboarding, authentication, transfers, swaps, DeFi strategies, x402 payments, and troubleshooting.
user-invocable: true
disable-model-invocation: false
allowed-tools:
  - "Bash(fdx *)"
  - "Bash(fdx)"
  - "Bash(npm install -g @financedistrict/fdx*)"
  - "Bash(npx @financedistrict/fdx*)"
  - "Bash(which fdx)"
---

# Finance District Agent Wallet

The `fdx` CLI gives agents crypto wallet capabilities — hold, send, swap, and earn yield on assets across EVM chains, Solana, and Bitcoin — without managing private keys. All commands return structured JSON.

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

## 3. Pre-Operation Checklist

Before any wallet operation, always:

1. **Check authentication**: `fdx status` — if not authenticated, complete onboarding first
2. **Check balances**: `fdx call getWalletOverview --chainKey <chain>` — verify sufficient token balance AND gas token balance for the target chain
3. **Confirm with the human** before executing irreversible operations (transfers, swaps, deposits) — state the amount, asset, chain, and recipient clearly
4. **Do not assume** — if any detail is ambiguous (chain, token, amount, recipient), ask for clarification

## 4. Core Operations

The wallet supports operations across all popular EVM chains, Solana, and Bitcoin. Basic wallet operations (balance, send, receive) work on all chains. Swap, DeFi yield, and x402 payments are available on EVM chains and Solana but not Bitcoin. Use `fdx call` without arguments to list all available methods. Use `fdx call <method> --help` for parameter details.

### Check Wallet State

```bash
fdx call getWalletOverview                          # all chains
fdx call getWalletOverview --chainKey ethereum      # specific chain
fdx call getMyInfo                                  # user profile and wallet addresses
fdx call getTokenPrice --token ETH                  # token price lookup
fdx call getAccountActivity --accountAddress <addr> --chainKey <chain>  # tx history
```

### Send Tokens

```bash
fdx call transferTokens \
  --toAddress <address-or-ENS-name> \
  --amount <amount> \
  --asset <symbol> \
  --chainKey <chain>
```

Supports ENS (.eth), SNS (.sol), and Unstoppable Domains. Resolve names with `fdx call resolveNameService --nameOrAddress "name.eth"`.

### Swap Tokens

```bash
fdx call swapTokens \
  --chainKey <chain> \
  --tokenIn <symbol> \
  --tokenOut <symbol> \
  --amount <amount>
```

For large swaps, set `--maxSlippageBps` explicitly (100 = 1%).

### Earn Yield (DeFi)

```bash
# Discover strategies
fdx call discoverYieldStrategies --chainKey <chain> --token <symbol>

# Deposit
fdx call depositForYield --strategyId <id> --fromAccountAddress <addr> --token <symbol> --amount <amount> --chainKey <chain>

# Withdraw
fdx call withdrawFromYield --vaultTokenAddress <vault> --underlyingToken <symbol> --withdrawAmount <amount> --fromAccountAddress <addr> --chainKey <chain>
```

Always present risk level and APY to the human before depositing.

### Pay for Services (x402)

```bash
fdx call getX402Content --url <x402-endpoint>
```

Supports multi-chain and multi-asset x402 payments. Inform the human about payment amounts before executing.

### Fund the Wallet

The wallet is funded by direct token transfer to the wallet address (from another wallet or exchange) or through the Finance District web dashboard. Get the wallet address from `fdx call getWalletOverview`.

For detailed operation guides, see `references/operations.md`.

## 5. Decision-Making Principles

- **Balance check first**: Before sends, swaps, or deposits — check both the token balance and the native gas token balance on that chain
- **Insufficient gas?** Suggest the human fund gas tokens, or swap into the gas token if they hold other tokens on that chain
- **Insufficient token balance?** Suggest funding the wallet or swapping from another token
- **Discover available tools**: Run `fdx call` with no arguments to see all methods
- **Parameter help**: Run `fdx call <method> --help` for any method's parameters
- **Smart Accounts**: Previously deployed Smart Accounts can still be used as `--fromAccountAddress` in transfers, swaps, and yield operations

## 6. Troubleshooting & Support

Common errors and resolutions:

| Error | Action |
|-------|--------|
| "not authenticated" | Run `fdx login --email <email>` then `fdx verify --code <OTP>` |
| "token expired" with refresh token | Auto-refreshes on next call — no action needed |
| "SESSION_EXPIRED" / "AUTH_REFRESH_FAILED" | Refresh token expired — run `fdx login` again |
| "Insufficient balance" | Check balance with `getWalletOverview`; fund or swap |
| "No liquidity" | Try smaller amount or different token pair |

When something goes wrong that you cannot resolve, report it:

```bash
fdx call reportIssue \
  --title "<short summary>" \
  --description "<what happened, steps to reproduce>"
```

Check CLI version for diagnostics: `fdx call getAppVersion`

For detailed troubleshooting, see `references/troubleshooting.md`.
