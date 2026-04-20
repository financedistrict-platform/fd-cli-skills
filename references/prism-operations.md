# Prism Merchant Operations Reference

This reference covers Prism workflow patterns and configuration guidance. For individual tool parameters, use `fdx prism <method> --help`.

## Merchant Setup Workflow

Setting up a new merchant follows this sequence:

1. **Authenticate**: Complete onboarding if not already authenticated (`fdx status`)
2. **Set account type**: `fdx prism updateAccountType --accountType Business` (or Personal)
3. **Create a Point of Service**: Define your merchant configuration with accepted assets and networks
4. **Configure settlement wallets**: Add wallet addresses where you want to receive payments, by chain
5. **Create API keys**: Generate keys for integrating payments into your application — the secret is returned **only once** on creation, so save it immediately

## Points of Service (PoS)

A PoS defines your merchant configuration — accepted assets, networks, and settlement wallets. Think of it as a payment profile.

- **Most tools default to your active PoS** — only pass `--posId` when managing multiple configurations
- **Configuration sections** can be read and updated independently: assets, networks, and wallet mappings each have their own get/update actions via `configurePointOfService`
- Use `--help` on `createPointOfService`, `updatePointOfService`, and `configurePointOfService` for the exact JSON structures expected

## Payment Monitoring Workflow

For a merchant checking on their business:

1. **Quick overview**: `getRecentPayments` for recent activity with a timeframe filter (7d, 30d, 90d, 180d, 1y)
2. **Revenue summary**: `getEarnings` for total earnings, optionally filtered by date range
3. **Payment details**: `getPaymentDetails` for a specific payment — returns blockchain tx hash, chain info, asset amounts, payer address, status, settlement breakdown, and fee lines
4. **Full history**: `listPayments` with pagination and sorting for complete transaction history

## API Key Management

- **Create**: Keys are generated with a name. The secret is shown only at creation — if lost, delete the key and create a new one
- **Disable vs Delete**: Disabling revokes access without removing the key (reversible). Deleting is permanent
- **List**: Secret values are never returned in list responses

## Staff Management

Staff members are users granted access to a specific Point of Service. Each `(posId, email)` pair is an independent record with its own `staffId`. The same email may be staff on multiple POS — each one is managed separately via two Prism tools: `listStaff` (read-only) and `manageStaff` (mutating).

### Workflow

1. **List current staff**: `fdx prism listStaff` — returns all staff with `staffId`, email, `pointOfServiceId`, status (`Active`/`Revoked`), `invitedAt`, and `revokedAt`
2. **Invite**: Grant a user access to a POS by email (reactivates if previously revoked on that POS)
3. **Revoke**: Remove access from a specific POS by `staffId`
4. **Multi-POS**: To invite a user to several POS, call `invite` once per POS

### Commands

```bash
# List all staff across tenant
fdx prism listStaff

# List staff for a specific POS
fdx prism listStaff --posId "pos-guid"

# Invite staff to a specific POS (reactivates if previously revoked)
fdx prism manageStaff --action invite --posId "pos-guid" --email staff@example.com

# Revoke staff from a specific POS (staffId from listStaff output)
fdx prism manageStaff --action revoke --posId "pos-guid" --staffId "staff-guid"
```

### Parameters

| Param     | Required for | Format                  |
|-----------|--------------|-------------------------|
| `action`  | all          | `invite` \| `revoke`    |
| `posId`   | all          | GUID of the POS         |
| `email`   | `invite`     | email string            |
| `staffId` | `revoke`     | GUID from `listStaff`   |

**Notes:**
- Always run `listStaff` first to get `staffId` values before revoking
- Staff records are per-POS — revoking from one POS does not affect other POS records
- To revoke all access for a user, revoke from each POS individually
- Staff management is owner-only; staff cannot manage other staff via CLI

## Settlement Wallets

Settlement wallets define where merchant payments are received, per chain.

- Each wallet is associated with a specific chain (by numeric chain ID — e.g. 8453 for Base, 1 for Ethereum) and asset
- You can set a default wallet and label wallets for organization
- Use `--help` on `manageWallet` for the exact parameters for create, update, and delete actions
