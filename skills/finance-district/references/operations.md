# Wallet Operations Reference

This reference covers wallet workflow patterns, chain capabilities, and safety considerations. For individual tool parameters, use `fdx wallet <method> --help`.

## Chain Capabilities

| Capability | EVM chains | Solana | Bitcoin |
| ---------- | ---------- | ------ | ------- |
| Balance, send, receive | Yes | Yes | Yes |
| Swap / Trade | Yes | Yes | No |
| DeFi Yield | Yes | Yes | No |
| x402 Payments | Yes | Yes | No |

Common chain keys: `ethereum`, `polygon`, `arbitrum`, `base`, `optimism`, `avalanche`, `bsc`, `solana`, `bitcoin`. Run `fdx wallet getWalletOverview` to see all available chains.

## Sending Tokens — Safety Patterns

- **Always confirm** recipient address, amount, chain, and asset with the human before executing — blockchain transactions are irreversible
- **Double-check the chain** matches the recipient's expected chain — sending on the wrong chain may result in lost funds
- **Name resolution**: ENS (.eth), SNS (.sol), and Unstoppable Domains are supported. You can resolve names with `resolveNameService` or pass them directly to the transfer tool
- **Smart Accounts**: You can send from a specific Smart Account by passing its address as `--fromAccountAddress`

## Swapping Tokens — Common Patterns

- **Buy gas tokens**: Swap stablecoins into the chain's native token (ETH, MATIC, SOL, etc.) when the user needs gas
- **Rebalance portfolio**: Swap between tokens on the same chain
- **Prepare for DeFi**: Swap into the token required by a yield strategy before depositing
- **Quote first**: Default mode is `QuoteOnly` — always get a quote and present it before executing with `--mode Execute`
- **Slippage**: For large swaps, set slippage explicitly to avoid unexpected losses. Use `--help` for the slippage parameter details

## Earning Yield — Workflow Patterns

1. **Discover**: Use `discoverYieldStrategies` to find available strategies. Filter by chain, token, or specific protocol (e.g. `aave-v3`, `compound-v3`)
2. **Evaluate**: Present the risk level, APY, and protocol name to the human. DeFi protocols carry smart contract risk — let the human make the final decision
3. **Deposit**: Use the `strategyId` from discovery. Both EOA and Smart Account addresses work
4. **Monitor**: Check positions via `getWalletOverview`
5. **Withdraw**: You need the `vaultTokenAddress` — find it from the original deposit response or wallet overview

## Funding the Wallet

There is no CLI onramp command. The wallet is funded by:

1. **Direct transfer**: Send tokens from any external wallet or exchange to the wallet address (get it from `getWalletOverview` or `getMyInfo`)
2. **Web dashboard**: The Finance District platform provides a web-based onramp

### Confirmation times

- Ethereum: ~12 seconds per block, may take a few minutes for finality
- Polygon/Base/Arbitrum: typically a few seconds
- Solana: near-instant
- Exchange withdrawals: additional processing time on the exchange side

After funding, verify with `fdx wallet getWalletOverview --chainKey <chain>`.
