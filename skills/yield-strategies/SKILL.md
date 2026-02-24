---
name: yield-strategies
description: Discover, deposit into, and withdraw from DeFi yield generation strategies across multiple chains and protocols. Use when you or the user want to earn yield, find APY, stake tokens, explore DeFi, farm, deposit into a protocol, withdraw earnings, check yield opportunities, or grow their holdings. Covers "how can I earn on my USDC?", "find me yield", "stake my ETH", "where can I get the best APY?", "withdraw from Aave".
user-invocable: true
disable-model-invocation: false
allowed-tools:
  [
    'Bash(fdx status*)',
    'Bash(fdx call discoverYieldStrategies*)',
    'Bash(fdx call depositForYield*)',
    'Bash(fdx call withdrawFromYield*)',
    'Bash(fdx call getWalletOverview*)',
  ]
---

# DeFi Yield Strategies

Discover yield opportunities across DeFi protocols (Aave, Compound, and others), deposit tokens to earn yield, and withdraw when ready. This is a full lifecycle skill covering discovery through to exit.

## Confirm wallet is authenticated

```bash
fdx status
```

If the wallet is not authenticated, refer to the `authenticate` skill.

## Step 1: Discover Yield Strategies

Search for available yield opportunities:

```bash
# Browse all available strategies
fdx call discoverYieldStrategies

# Filter by chain
fdx call discoverYieldStrategies --chainKey ethereum

# Filter by token
fdx call discoverYieldStrategies --chainKey ethereum --token USDC

# Filter by protocol
fdx call discoverYieldStrategies --chainKey bsc --protocolSlug venus

# Sort results by APY descending
fdx call discoverYieldStrategies --chainKey ethereum --sortBy apy --sortDirection desc --limit 10
```

### discoverYieldStrategies Parameters

| Parameter         | Required | Description                                                           |
| ----------------- | -------- | --------------------------------------------------------------------- |
| `--chainKey`      | No       | Filter by blockchain (e.g. `ethereum`, `polygon`, `arbitrum`, `base`) |
| `--token`         | No       | Filter by token symbol or address (e.g. `USDC`, `WETH`, or `0x...`)   |
| `--protocolSlug`  | No       | Filter by protocol (e.g. `aave-v3`, `venus`, `compound-v3`)           |
| `--sortBy`        | No       | Sort field: `apy` or `risk` (default: `apy`)                          |
| `--sortDirection` | No       | Sort direction: `desc` or `asc` (default: `desc`)                     |
| `--limit`         | No       | Max results to return, 1-100 (default: 30)                            |

## Step 2: Deposit for Yield

Once a strategy is selected, deposit tokens into it:

```bash
fdx call depositForYield \
  --strategyId <strategyId> \
  --fromAccountAddress <address> \
  --token <token> \
  --amount <amount> \
  --chainKey <chain>
```

### depositForYield Parameters

| Parameter              | Required | Description                                                |
| ---------------------- | -------- | ---------------------------------------------------------- |
| `--strategyId`         | Yes      | Strategy identifier from `discoverYieldStrategies` results |
| `--fromAccountAddress` | Yes      | Account address to deposit from (EOA or Smart Account)     |
| `--token`              | Yes      | Token to deposit (e.g. `USDC`, `WETH`, or `0x...`)         |
| `--amount`             | Yes      | Amount to deposit (decimal)                                |
| `--chainKey`           | Yes      | Blockchain where the strategy runs                         |

## Step 3: Withdraw from Yield

Exit a position and retrieve tokens:

```bash
fdx call withdrawFromYield \
  --vaultTokenAddress <vaultToken> \
  --underlyingToken <token> \
  --withdrawAmount <amount> \
  --fromAccountAddress <address> \
  --chainKey <chain>
```

### withdrawFromYield Parameters

| Parameter              | Required | Description                                                         |
| ---------------------- | -------- | ------------------------------------------------------------------- |
| `--vaultTokenAddress`  | Yes      | Vault token address (0x...) — use `discoverYieldStrategies` to find |
| `--underlyingToken`    | Yes      | Underlying token to receive (e.g. `USDC`, `WETH`)                   |
| `--withdrawAmount`     | Yes      | Amount of vault tokens to withdraw (decimal)                        |
| `--fromAccountAddress` | Yes      | Account holding the vault tokens (EOA or Smart Account)             |
| `--chainKey`           | Yes      | Blockchain identifier                                               |

## Example Session

```bash
# Check auth and balance
fdx status
fdx call getWalletOverview --chainKey ethereum

# Discover USDC yield strategies on Ethereum, sorted by APY
fdx call discoverYieldStrategies \
  --chainKey ethereum \
  --token USDC \
  --sortBy apy \
  --sortDirection desc

# Deposit 1000 USDC into a chosen strategy
fdx call depositForYield \
  --strategyId <strategyId-from-discovery> \
  --fromAccountAddress <your-account-address> \
  --token USDC \
  --amount 1000 \
  --chainKey ethereum

# Later: withdraw from the vault
fdx call withdrawFromYield \
  --vaultTokenAddress <vault-token-from-strategy> \
  --underlyingToken USDC \
  --withdrawAmount 1000 \
  --fromAccountAddress <your-account-address> \
  --chainKey ethereum
```

## Flow

1. Check authentication with `fdx status`
2. Check available balance with `fdx call getWalletOverview --chainKey <chain>`
3. Discover strategies with `fdx call discoverYieldStrategies` — present options to the human
4. Human selects a strategy — confirm the choice, risks, and deposit amount
5. Execute deposit with `fdx call depositForYield`
6. When the human wants to exit, withdraw with `fdx call withdrawFromYield`

**Important:** DeFi protocols carry smart contract risk. Always present the risk level to your human and let them make the final decision on which strategy to use and how much to deposit.

## Prerequisites

- Must be authenticated (`fdx status` to check, see `authenticate` skill)
- Wallet must hold sufficient balance of the deposit token on the target chain
- If insufficient funds, suggest using the `fund-wallet` skill or `swap-tokens` skill to acquire the needed token

## Error Handling

- "Not authenticated" — Run `fdx login` first, or see `authenticate` skill
- "Insufficient balance" — Check balance; see `fund-wallet` skill
- "Invalid strategyId" — Re-run `discoverYieldStrategies` to get current strategy IDs
- "Invalid vaultTokenAddress" — Use `discoverYieldStrategies` to find the correct vault token address
