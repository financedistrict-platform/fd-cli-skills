# Prism Platform Operations Guide

This reference covers the Prism platform tools in detail. Prism tools are dynamically discovered — run `fdx prism call` to list all available tools or `fdx prism call <method> --help` for parameter details.

Most tools accept an optional `--posId` parameter. If omitted, your default active Point of Service is used. Only pass `--posId` when managing multiple PoS configurations.

## Points of Service

A Point of Service (PoS) defines your merchant configuration — accepted assets, networks, and settlement wallets.

### List and inspect

```bash
fdx prism call listPointsOfService
fdx prism call getPointOfServiceDetails                 # default PoS
fdx prism call getPointOfServiceDetails --posId <id>    # specific PoS
```

### Create a PoS

```bash
fdx prism call createPointOfService \
  --name "My Store" \
  --configurationJson '{"assets":{"acceptedAssets":[]},"networks":{"acceptedNetworks":[]}}'
```

### Update, activate, or deactivate

```bash
fdx prism call updatePointOfService --action update \
  --name "Updated Store Name" \
  --configurationJson '{"assets":{"acceptedAssets":[]},"networks":{"acceptedNetworks":[]}}'

fdx prism call updatePointOfService --action activate
fdx prism call updatePointOfService --action deactivate
```

### Configure assets, networks, and wallets

The `configurePointOfService` tool reads or updates individual configuration sections:

```bash
# Read current config
fdx prism call configurePointOfService --action getAssets
fdx prism call configurePointOfService --action getNetworks

# Update accepted assets
fdx prism call configurePointOfService --action updateAssets \
  --assetsJson '[{"symbol":"USDC","minAmount":1.0,"isPreferred":true,"priority":1}]'

# Update accepted networks
fdx prism call configurePointOfService --action updateNetworks \
  --networksJson '{"acceptedNetworks":[{"networkId":"base","walletAddress":"0x...","isPreferred":true,"priority":1}]}'

# Update wallet mappings
fdx prism call configurePointOfService --action updateWallets \
  --walletsJson '{"base":"0xABC...","ethereum":"0xDEF..."}'
```

## Payments

### List payments

```bash
fdx prism call listPayments
fdx prism call listPayments --page 1 --pageSize 10 --sort "createdAt desc"
```

### Get payment details

```bash
fdx prism call getPaymentDetails --paymentId <id>
```

Returns blockchain tx hash, chain info, asset amounts, payer address, status, settlement breakdown, and fee lines.

## Dashboard

### Earnings summary

```bash
fdx prism call getEarnings
fdx prism call getEarnings \
  --startDateTime "2025-01-01T00:00:00Z" \
  --endDateTime "2025-12-31T23:59:59Z"
```

### Recent payments

```bash
fdx prism call getRecentPayments
fdx prism call getRecentPayments --timeframe 30d --limit 10
```

Accepted timeframes: `7d`, `30d`, `90d`, `180d`, `1y`.

## API Keys

### List keys

```bash
fdx prism call listApiKeys
```

Secret values are never returned in list responses.

### Create a key

```bash
fdx prism call manageApiKey --action create --name "Production Key"
```

The secret is returned **only once** on creation. Save it immediately.

### Disable or delete

```bash
fdx prism call manageApiKey --action disable --apiKeyId <id>
fdx prism call manageApiKey --action delete --apiKeyId <id>
```

Disabling revokes access without deleting. Deleting is permanent.

## Settlement Wallets

### List wallets

```bash
fdx prism call listWallets
```

### Create a wallet

```bash
fdx prism call manageWallet --action create \
  --chainId 8453 \
  --asset USDC \
  --address 0x... \
  --label "Base USDC" \
  --isDefault true
```

`--chainId` is the numeric chain ID (e.g. 8453 for Base, 1 for Ethereum).

### Update or delete

```bash
fdx prism call manageWallet --action update \
  --walletId <id> \
  --address 0x... \
  --label "Updated Label"

fdx prism call manageWallet --action delete --walletId <id>
```

## Provider Profile

### Get provider info

```bash
fdx prism call getProviderInfo
fdx prism call getProviderInfo --includeChains true --includeTokens true
```

Returns provider name, account type, status, and the full list of supported networks and assets.

### Set account type

```bash
fdx prism call updateAccountType --accountType Business
```

Accepted values: `Personal` or `Business` (case-insensitive). Used during onboarding.
