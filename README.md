# Finance District Agent Skills

Agent Skills for the Finance District platform. Installing this package gives your agent multiple skills that cover wallet management, merchant operations, and agentic shopping.

## Skills

| Skill | Description |
| ----- | ----------- |
| [finance-district](./skills/finance-district/SKILL.md) | Finance District CLI — Agent Wallet (transfers, swaps, DeFi yield, x402 payments) and Prism payment gateway (merchant accounts, payments, settlements, Points of Service, staff management) |
| [fd-agentic-commerce](./skills/fd-agentic-commerce/SKILL.md) | Complete shopping checkouts at any agentic-commerce merchant that accepts the Finance District Prism payment handler. Auto-detects and dispatches to UCP or ACP — catalog/feed browsing, checkout session lifecycle, x402 payment authorization, and order confirmation |

## Installation

Install with [Vercel's Skills CLI](https://skills.sh):

```bash
npx skills add financedistrict-platform/fd-cli-skills
```

## Usage

Both skills are automatically available once installed. The agent picks the right one based on what you ask.

**finance-district** (wallet, DeFi, merchant ops):

```text
Sign in to my Finance District account
Show my balance on Ethereum
Send 10 USDC to 0x1234...abcd on Base
Find yield strategies for my USDC
Show my recent payments
Create an API key for my store
```

**fd-agentic-commerce** (agent-driven shopping):

```text
Buy me a gift from https://medusa.test.1stdigital.tech for my brother
Order a t-shirt from the FD merch store and ship it to my Berlin address
Checkout the items in my cart at <merchant>
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
