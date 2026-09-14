# Prism Merchant Operations Reference

This reference covers Prism workflow patterns and configuration guidance. For individual tool parameters, use `fdx prism <method> --help`.

## Merchant Setup Workflow

Setting up a new merchant follows this sequence:

1. **Authenticate**: Complete onboarding if not already authenticated (`fdx status`)
2. **Discover the provider catalog**: `fdx prism getProviderInfo --includeCountries true` — learn valid country codes, networks, and token symbols before creating the Project
3. **Create a Project**: `fdx prism createProjectFromWizard` — sets storefront, platform, country, environment, settlement networks, currencies, and wallet source together, and activates the Project
4. **Check for outstanding config**: `fdx prism getRequiredUpdates` — an empty list means nothing is left to fill in
5. **Fill any gaps**: `fdx prism updateSettlementCurrencies` / `fdx prism updateSettlementFx` as needed
6. **Create a Project Identify Token**: `fdx prism manageProjectIdentifyToken --action create --expiration <30d|180d|365d|none>` — the secret is returned **only once** on creation, so save it immediately
7. **Invite staff** (optional): `fdx prism manageStaff --action invite`
8. **Check status**: `fdx prism getHomeSummary` — the recommended entry point for reading a Project's status going forward

## Projects

A Project defines your merchant configuration — accepted assets, networks, and settlement wallets. Think of it as a payment profile.

- **Most tools default to your active Project** — only pass `--projectId` when managing multiple configurations
- **Configuration sections** can be read and updated independently: `getSettlementSettings` reads networks, currencies, and FX settings in one call; `updateSettlementNetworks`, `updateSettlementCurrencies`, and `updateSettlementFx` update them individually; `resetSettlementDefaults` resets networks, currencies, or wallet source back to platform defaults
- Use `--help` on `createProjectFromWizard`, `updateProjectAccount`, and the `updateSettlement*` tools for the exact JSON structures expected
- `getHomeSummary` is the recommended entry point for reading a Project's overall status — prefer it over calling earnings/settlement/payment tools separately

## Payment Monitoring Workflow

For a merchant checking on their business:

1. **Quick overview**: `getRecentPayments` for recent activity with a timeframe filter (7d, 30d, 90d, 180d, 1y)
2. **Revenue summary**: `getEarnings` for total earnings, optionally filtered by date range
3. **Payment details**: `getPaymentDetails` for a specific payment — returns blockchain tx hash, chain info, asset amounts, payer address, status, settlement breakdown, and fee lines
4. **Full history**: `listPayments` with pagination and sorting for complete transaction history

## Project Identify Token Management

- **Create**: Tokens are generated with a name and an `expiration` of `30d`, `180d`, `365d`, or `none` (never expires). The secret is shown only at creation — if lost, delete the token and create a new one
- **Disable vs Delete**: Disabling revokes access without removing the token (reversible). Deleting is permanent
- **List**: Secret values are never returned in list responses

## Staff Management

Staff members are users granted access to a specific Project. Each `(projectId, email)` pair is an independent record with its own `staffId`. The same email may be staff on multiple projects — each one is managed separately via two Prism tools: `listStaff` (read-only) and `manageStaff` (mutating).

### Workflow

1. **List current staff**: `fdx prism listStaff` — returns all staff with `staffId`, email, `projectId`, status (`Active`/`Revoked`), `invitedAt`, and `revokedAt`
2. **Invite**: Grant a user access to a project by email (reactivates if previously revoked on that project)
3. **Revoke**: Remove access from a specific project by `staffId`
4. **Multi-project**: To invite a user to several projects, call `invite` once per project

### Commands

```bash
# List all staff across tenant
fdx prism listStaff

# List staff for a specific project
fdx prism listStaff --projectId "project-guid"

# Invite staff to a specific project (reactivates if previously revoked)
fdx prism manageStaff --action invite --projectId "project-guid" --email staff@example.com

# Revoke staff from a specific project (staffId from listStaff output)
fdx prism manageStaff --action revoke --projectId "project-guid" --staffId "staff-guid"
```

### Parameters

| Param       | Required for | Format                    |
|-------------|--------------|---------------------------|
| `action`    | all          | `invite` \| `revoke`      |
| `projectId` | all          | GUID of the project       |
| `email`     | `invite`     | email string              |
| `staffId`   | `revoke`     | GUID from `listStaff`     |

**Notes:**
- Always run `listStaff` first to get `staffId` values before revoking
- Staff records are per-project — revoking from one project does not affect other project records
- To revoke all access for a user, revoke from each project individually
- Staff management is owner-only; staff cannot manage other staff via CLI

## Settlement Wallets

Settlement wallets define where a Project's payments are received.

- **Read**: `fdx prism getWalletSettings` — the authoritative settlement destination: wallet source, the merchant's own saved addresses, and the effective receiving address per network
- **Update**: `fdx prism updateWalletSettings --walletSource <fd-agent|own>` — `fd-agent` routes every network to the FD-managed agent wallet; `own` uses the merchant's own addresses, either one `--walletAddress` for every network or a per-network `--networkWallets` mapping (CAIP-2 network id → address)
- This moves real money — confirm the addresses with the merchant before calling `updateWalletSettings`, and verify the result with `getWalletSettings` afterward
- Use `--help` on `updateWalletSettings` for the exact parameters
