# Finance District Agent Wallet Skills

[Agent Skills](https://skills.sh) for the Finance District crypto wallet. These skills enable AI agents to authenticate, manage wallets, send tokens, swap via DEXs, earn DeFi yield, and more using the [`fdx`](https://www.npmjs.com/package/@financedistrict/fdx) CLI.

## Available Skills

| Skill | Description |
| ----- | ----------- |
| [authenticate](./skills/authenticate/SKILL.md) | Sign in to the wallet via OAuth 2.1 (browser or device flow) |
| [wallet-overview](./skills/wallet-overview/SKILL.md) | Check balances, holdings, and transaction history across all chains |
| [send-tokens](./skills/send-tokens/SKILL.md) | Transfer tokens to any address on any EVM chain or Solana |
| [swap-tokens](./skills/swap-tokens/SKILL.md) | Swap tokens via decentralized exchanges on any supported chain |
| [fund-wallet](./skills/fund-wallet/SKILL.md) | Add funds via web onramp or direct transfer to wallet address |
| [yield-strategies](./skills/yield-strategies/SKILL.md) | Discover, deposit into, and withdraw from DeFi yield strategies |
| [smart-accounts](./skills/smart-accounts/SKILL.md) | Deploy and manage multi-signature smart accounts |
| [pay-for-service](./skills/pay-for-service/SKILL.md) | Access paid API endpoints via the x402 payment protocol |
| [help-and-support](./skills/help-and-support/SKILL.md) | Get help, onboarding guidance, and report issues |

## Installation

Install with [Vercel's Skills CLI](https://skills.sh):

```bash
npx skills add 1stdigital/fd-agent-wallet-skills
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

**Examples:**

```text
Sign in to my wallet
```

```text
Show my balance on Ethereum
```

```text
Send 10 USDC to 0x1234...abcd on Base
```

```text
Find yield strategies for my USDC
```

## Prerequisites

Install the CLI globally:

```bash
npm install -g @financedistrict/fdx
```

Once installed, the `fdx` command is available directly — no `npx` needed.

## Key Differentiators

- **Multi-chain**: Supports all EVM chains and Solana (not locked to a single chain)
- **DEX swaps**: Token swaps execute through decentralized exchanges, not centralized order books
- **DeFi yield**: Discover and invest in yield strategies across protocols like Aave, Compound, and more
- **Smart accounts**: Deploy and manage multi-sig wallets on-chain
- **Multi-asset x402**: Pay for services with any supported asset on any chain

## Contributing

To add a new skill:

1. Create a folder in `./skills/` with a lowercase, hyphenated name
2. Add a `SKILL.md` file with YAML frontmatter and instructions
3. Follow the [Agent Skills specification](https://agentskills.io/specification) for the complete format

## License

MIT
