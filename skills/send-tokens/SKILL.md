---
name: send-tokens
description: Send or transfer tokens to any address on any supported chain (EVM or Solana). Use when you or the user want to send money, pay someone, transfer tokens, tip, donate, or move funds to another wallet address. Covers phrases like "send 10 USDC to", "pay 0x...", "transfer ETH to", "move tokens to my other wallet".
user-invocable: true
disable-model-invocation: false
allowed-tools: ["Bash(fdx status*)", "Bash(fdx call transferTokens*)", "Bash(fdx call getWalletOverview*)", "Bash(fdx call resolveNameService*)"]
---

# Sending Tokens

Use the `fdx call transferTokens` command to transfer tokens from the wallet to any address on any supported EVM chain or Solana. Supports asset symbols (e.g. `USDC`, `ETH`, `SOL`) and ENS/SNS name resolution.

## Confirm wallet is authenticated

```bash
fdx status
```

If the wallet is not authenticated, refer to the `authenticate` skill.

## Check Balance Before Sending

Always verify the wallet has sufficient balance before initiating a transfer:

```bash
fdx call getWalletOverview --chainKey <chain>
```

## Resolve a Name (Optional)

If the recipient provides an ENS (.eth), SNS (.sol), or Unstoppable Domain name, you can resolve it first:

```bash
fdx call resolveNameService --nameOrAddress "vitalik.eth"
```

Or pass the name directly to `--toAddress` — the server supports ENS/SNS resolution.

## Sending Tokens

```bash
fdx call transferTokens \
  --toAddress <address-or-name> \
  --amount <amount> \
  --asset <symbol-or-address> \
  --chainKey <chain>
```

### Parameters

| Parameter              | Required | Description                                                               |
| ---------------------- | -------- | ------------------------------------------------------------------------- |
| `--toAddress`          | Yes      | Recipient address (0x... for EVM, Base58 for Solana) or ENS/SNS name      |
| `--amount`             | Yes      | Amount to send (decimal, e.g. `10`, `0.5`)                                |
| `--asset`              | Yes      | Asset symbol (e.g. `ETH`, `USDC`, `SOL`) or token contract address        |
| `--chainKey`           | Yes      | Target blockchain (e.g. `ethereum`, `base`, `bsc`, `solana`)              |
| `--fromAccountAddress` | No       | Source wallet address (auto-selected if omitted)                          |
| `--autoApprove`        | No       | Auto-approve transfer up to configured limit (default: false)             |

## Examples

### Send native tokens

```bash
# Send 0.1 ETH on Ethereum
fdx call transferTokens \
  --toAddress 0x1234...abcd \
  --amount 0.1 \
  --asset ETH \
  --chainKey ethereum

# Send 1 SOL on Solana
fdx call transferTokens \
  --toAddress AbCd...1234 \
  --amount 1 \
  --asset SOL \
  --chainKey solana
```

### Send ERC-20 tokens

```bash
# Send 100 USDC on Ethereum
fdx call transferTokens \
  --toAddress 0x1234...abcd \
  --amount 100 \
  --asset USDC \
  --chainKey ethereum

# Send 50 USDT on BSC
fdx call transferTokens \
  --toAddress 0x1234...abcd \
  --amount 50 \
  --asset USDT \
  --chainKey bsc
```

### Send to an ENS name

```bash
fdx call transferTokens \
  --toAddress vitalik.eth \
  --amount 10 \
  --asset USDC \
  --chainKey ethereum
```

### Send from a specific account

```bash
fdx call transferTokens \
  --toAddress 0x1234...abcd \
  --amount 10 \
  --asset USDC \
  --chainKey base \
  --fromAccountAddress 0xMySmartAccount...
```

## Flow

1. Check authentication with `fdx status`
2. Check balance with `fdx call getWalletOverview --chainKey <chain>`
3. Optionally resolve ENS/SNS names: `fdx call resolveNameService --nameOrAddress "name.eth"`
4. Confirm the transfer details with the human (amount, recipient, chain, asset)
5. Execute with `fdx call transferTokens`
6. Report the transaction result to the human

**Important:** Always confirm the recipient address and amount with your human before executing, especially for large amounts. Blockchain transactions are irreversible.

## Prerequisites

- Must be authenticated (`fdx status` to check, see `authenticate` skill)
- Wallet must have sufficient balance on the target chain
- If sending insufficient funds, suggest using the `fund-wallet` skill

## Error Handling

- "Not authenticated" — Run `fdx setup` first, or see `authenticate` skill
- "Insufficient balance" — Check balance with `getWalletOverview`; see `fund-wallet` skill
- "Invalid recipient" — Verify the address format matches the target chain
