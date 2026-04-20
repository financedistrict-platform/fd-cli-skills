# Finance District CLI Skills

Claude skills for the Finance District command-line tooling (`fdx`). Covers wallet management, DeFi operations, and Prism merchant administration — all driven through the `fdx` CLI.

Looking for the **shopping assistant** (pays at FD-enabled stores on your behalf)? That skill moved to [**fd-assistant-skills**](https://github.com/financedistrict-platform/fd-assistant-skills) — it's MCP-only and no longer needs the CLI.

See [SKILL.md](./SKILL.md) for the full skill definition — covers Agent Wallet (transfers, swaps, DeFi yield, x402 payments) and Prism payment gateway (merchant accounts, payments, settlements, Points of Service, staff management).

## Installation

Install with [Vercel's Skills CLI](https://skills.sh):

```bash
npx skills add financedistrict-platform/fd-cli-skills
```

## Usage

**finance-district** (wallet, DeFi, merchant ops):

```text
Sign in to my Finance District account
Show my balance on Ethereum
Send 10 USDC to 0x1234...abcd on Base
Find yield strategies for my USDC
Show my recent payments
Create an API key for my store
```

## Prerequisites

Install the CLI globally:

```bash
npm install -g @financedistrict/fdx
```

Once installed, the `fdx` command is available directly — no `npx` needed.

## Key Differentiators

- **Two service domains**: Agent Wallet for on-chain operations and Prism for payment gateway — one CLI for both
- **Multi-chain**: Supports all EVM chains, Solana, and Bitcoin (not locked to a single chain)
- **DEX swaps**: Token swaps execute through decentralized exchanges, not centralized order books
- **DeFi yield**: Discover and invest in yield strategies across protocols like Aave, Compound, and more
- **Smart accounts**: Deploy and manage multi-sig wallets on-chain
- **Multi-asset x402**: Pay for services with any supported asset on any chain
- **Prism payment gateway**: Manage merchant accounts, payments, API keys, settlement wallets, and Points of Service
- **Dynamic tool discovery**: Prism tools are auto-discovered from the service at runtime

## Contributing

See the [Agent Skills specification](https://agentskills.io/specification) for the skill format.

## License

MIT
