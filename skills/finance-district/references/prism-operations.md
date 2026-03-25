# Prism Platform Operations Guide

This reference covers the Prism platform tools in detail. Prism tools are dynamically discovered — run `fdx prism` to list all available tools or `fdx prism <method> --help` for parameter details.

Most tools accept an optional `--posId` parameter. If omitted, your default active Point of Service is used. Only pass `--posId` when managing multiple PoS configurations.

## Points of Service

A Point of Service (PoS) defines your merchant configuration — accepted assets, networks, and settlement wallets.

### List and inspect

```bash
fdx prism listPointsOfService
fdx prism getPointOfServiceDetails                 # default PoS
fdx prism getPointOfServiceDetails --posId <id>    # specific PoS
```

### Create a PoS

```bash
fdx prism createPointOfService \
  --name "My Store" \
  --configurationJson '{"assets":{"acceptedAssets":[]},"networks":{"acceptedNetworks":[]}}'
```

### Update, activate, or deactivate

```bash
fdx prism updatePointOfService --action update \
  --name "Updated Store Name" \
  --configurationJson '{"assets":{"acceptedAssets":[]},"networks":{"acceptedNetworks":[]}}'

fdx prism updatePointOfService --action activate
fdx prism updatePointOfService --action deactivate
```

### Configure assets, networks, and wallets

The `configurePointOfService` tool reads or updates individual configuration sections:

```bash
# Read current config
fdx prism configurePointOfService --action getAssets
fdx prism configurePointOfService --action getNetworks

# Update accepted assets
fdx prism configurePointOfService --action updateAssets \
  --assetsJson '[{"symbol":"USDC","minAmount":1.0,"isPreferred":true,"priority":1}]'

# Update accepted networks
fdx prism configurePointOfService --action updateNetworks \
  --networksJson '{"acceptedNetworks":[{"networkId":"base","walletAddress":"0x...","isPreferred":true,"priority":1}]}'

# Update wallet mappings
fdx prism configurePointOfService --action updateWallets \
  --walletsJson '{"base":"0xABC...","ethereum":"0xDEF..."}'
```

## Payments

### List payments

```bash
fdx prism listPayments
fdx prism listPayments --page 1 --pageSize 10 --sort "createdAt desc"
```

### Get payment details

```bash
fdx prism getPaymentDetails --paymentId <id>
```

Returns blockchain tx hash, chain info, asset amounts, payer address, status, settlement breakdown, and fee lines.

## Dashboard

### Earnings summary

```bash
fdx prism getEarnings
fdx prism getEarnings \
  --startDateTime "2025-01-01T00:00:00Z" \
  --endDateTime "2025-12-31T23:59:59Z"
```

### Recent payments

```bash
fdx prism getRecentPayments
fdx prism getRecentPayments --timeframe 30d --limit 10
```

Accepted timeframes: `7d`, `30d`, `90d`, `180d`, `1y`.

## API Keys

### List keys

```bash
fdx prism listApiKeys
```

Secret values are never returned in list responses.

### Create a key

```bash
fdx prism manageApiKey --action create --name "Production Key"
```

The secret is returned **only once** on creation. Save it immediately.

### Disable or delete

```bash
fdx prism manageApiKey --action disable --apiKeyId <id>
fdx prism manageApiKey --action delete --apiKeyId <id>
```

Disabling revokes access without deleting. Deleting is permanent.

## Settlement Wallets

### List wallets

```bash
fdx prism listWallets
```

### Create a wallet

```bash
fdx prism manageWallet --action create \
  --chainId 8453 \
  --asset USDC \
  --address 0x... \
  --label "Base USDC" \
  --isDefault true
```

`--chainId` is the numeric chain ID (e.g. 8453 for Base, 1 for Ethereum).

### Update or delete

```bash
fdx prism manageWallet --action update \
  --walletId <id> \
  --address 0x... \
  --label "Updated Label"

fdx prism manageWallet --action delete --walletId <id>
```

## Provider Profile

### Get provider info

```bash
fdx prism getProviderInfo
fdx prism getProviderInfo --includeChains true --includeTokens true
```

Returns provider name, account type, status, and the full list of supported networks and assets.

### Set account type

```bash
fdx prism updateAccountType --accountType Business
```

Accepted values: `Personal` or `Business` (case-insensitive). Used during onboarding.
