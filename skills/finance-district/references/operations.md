# Wallet Operations Guide

This reference covers the core wallet operations in detail. For every operation, always follow the pre-operation checklist in the main skill: check auth, check balance, confirm with human.

## Supported Chains

The wallet supports all popular EVM chains, Solana, and Bitcoin. Common chain keys include `ethereum`, `polygon`, `arbitrum`, `base`, `optimism`, `avalanche`, `bsc`, `solana`, and `bitcoin`. Run `fdx wallet getWalletOverview` to see all available chains.

**Chain capability matrix:**

| Capability | EVM chains | Solana | Bitcoin |
| ---------- | ---------- | ------ | ------- |
| Wallet (balance, send, receive) | Yes | Yes | Yes |
| Swap / Trade | Yes | Yes | No |
| DeFi Yield | Yes | Yes | No |
| x402 Payments | Yes | Yes | No |

## Sending Tokens

### Basic transfer

```bash
fdx wallet transferTokens \
  --toAddress <address> \
  --amount <amount> \
  --asset <symbol-or-contract> \
  --chainKey <chain>
```

### Key details

- `--toAddress` accepts 0x addresses (EVM), Base58 (Solana), ENS names (.eth), SNS names (.sol), and Unstoppable Domains
- `--asset` accepts token symbols (ETH, USDC, SOL) or contract addresses
- `--fromAccountAddress` is optional — auto-selected if omitted. Use it to send from a specific Smart Account
- `--autoApprove` (bool) — auto-approve transfer up to configured limit without confirmation prompt
- Resolve names first with `fdx wallet resolveNameService --nameOrAddress "name.eth"` or pass names directly to `--toAddress`

### Safety

- Always confirm recipient address, amount, chain, and asset with the human before executing
- Blockchain transactions are irreversible
- Double-check the chain matches the recipient's expected chain — sending on the wrong chain may result in lost funds

## Swapping Tokens

### Basic swap

```bash
fdx wallet swapTokens \
  --chainKey <chain> \
  --tokenIn <symbol> \
  --tokenOut <symbol> \
  --amount <amount> \
  --mode Execute
```

### Key details

- Default `--mode` is `QuoteOnly` (returns quote without executing). Pass `--mode Execute` to actually swap
- `--objective` controls routing strategy (default: `BestExecution`)
- `--maxSlippageBps` controls slippage tolerance in basis points (100 = 1%). Set explicitly for large swaps (default: 50 = 0.5%)
- `--deadlineSeconds` sets a transaction deadline (default: 120)
- `--tokenIn` and `--tokenOut` accept symbols or contract addresses

### Common patterns

- **Buy gas tokens**: swap stablecoins into the chain's native token (ETH, MATIC, SOL, etc.)
- **Rebalance portfolio**: swap between tokens on the same chain
- **Prepare for DeFi**: swap into the token required by a yield strategy

## Earning Yield (DeFi)

### Discover strategies

```bash
fdx wallet discoverYieldStrategies \
  --chainKey <chain> \
  --token <symbol> \
  --sortBy apy \
  --sortDirection desc \
  --limit 10
```

Filter by `--protocolSlug` (e.g. `aave-v3`, `compound-v3`, `venus`) for specific protocols.

### Deposit

```bash
fdx wallet depositForYield \
  --strategyId <id-from-discovery> \
  --fromAccountAddress <address> \
  --token <symbol> \
  --amount <amount> \
  --chainKey <chain>
```

### Withdraw

```bash
fdx wallet withdrawFromYield \
  --vaultTokenAddress <vault-token> \
  --underlyingToken <symbol> \
  --withdrawAmount <amount> \
  --fromAccountAddress <address> \
  --chainKey <chain>
```

### Key details

- Always present risk level and APY to the human before depositing
- DeFi protocols carry smart contract risk — let the human make the final decision
- Use `discoverYieldStrategies` to find the `strategyId` (for deposits) and `vaultTokenAddress` (for withdrawals)
- Both EOA and Smart Account addresses work as `--fromAccountAddress`

## Paying for Services (x402)

### Fetch paid content

```bash
fdx wallet getX402Content --url <x402-endpoint>
```

### With preferences

```bash
fdx wallet getX402Content \
  --url <endpoint> \
  --preferredNetwork base \
  --preferredAsset USDC
```

### Authorize from a 402 response

If you already have payment requirements JSON from an HTTP 402 response:

```bash
fdx wallet authorizePayment \
  --paymentRequirementsResponseJson '<json>' \
  --autoApprove true
```

`--autoApprove` skips the confirmation prompt and authorizes payment immediately.

### Key details

- Finance District supports multi-chain and multi-asset x402 payments (not limited to Base USDC)
- Always inform the human about the payment amount before executing, especially for unfamiliar endpoints
- Check wallet balance on the preferred payment chain before calling

## Funding the Wallet

There is no CLI command for direct onramp. The wallet is funded by:

1. **Direct transfer**: Send tokens from any external wallet or exchange to the wallet address (get it from `fdx wallet getWalletOverview`)
2. **Web dashboard**: The Finance District platform provides a web-based onramp with credit card support

### Confirmation times

- Ethereum: ~12 seconds per block, may take a few minutes for finality
- Polygon/Base/Arbitrum: typically a few seconds
- Solana: near-instant
- Exchange withdrawals: additional processing time

After funding, verify with `fdx wallet getWalletOverview --chainKey <chain>`.
