# Finance District CLI Skill

Agent Skill for the [Finance District CLI](https://www.npmjs.com/package/@financedistrict/fdx) (`fdx`). This skill enables AI agents to access Finance District services through a single CLI — **Agent Wallet** for on-chain operations (hold, send, swap, earn DeFi yield across EVM chains, Solana, and Bitcoin) and **Prism** payment gateway for merchant payments, settlements, Points of Service, and API key management.

## Skills

| Skill | Description |
| ----- | ----------- |
| [fd-cli](./SKILL.md) | Finance District CLI — Agent Wallet (transfers, swaps, DeFi yield, x402 payments) and Prism payment gateway (merchant accounts, payments, settlements, Points of Service) |
| [fd-agentic-commerce](./skills/fd-agentic-commerce/SKILL.md) | Complete shopping checkouts at any agentic-commerce merchant that accepts the Finance District Prism payment handler — catalog search, checkout session lifecycle, x402 payment authorization, and order confirmation. v1 covers UCP; ACP support planned |

## Installation

Install with [Vercel's Skills CLI](https://skills.sh):

```bash
npx skills add financedistrict-platform/fd-cli-skills
```

## Usage

The skill is automatically available once installed. The agent will use it when relevant tasks are detected.

**Examples:**

```text
Sign in to my Finance District account
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

```text
Show my recent payments
```

```text
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
