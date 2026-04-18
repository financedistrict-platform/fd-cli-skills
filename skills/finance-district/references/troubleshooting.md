# Troubleshooting & Support

## Error Reference

### Authentication Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| "not authenticated" | No valid session | `fdx login --email <email>` then `fdx verify --code <OTP>` |
| "token expired" | Access token expired, refresh token available | Auto-refreshes on next call — no action needed |
| "SESSION_EXPIRED" | Both access and refresh tokens expired | Run `fdx login` again |
| "AUTH_REFRESH_FAILED" | Token refresh failed | Run `fdx login` to re-authenticate |

### Operation Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| "Insufficient balance" | Not enough tokens | Check balance with `getWalletOverview`; fund wallet or swap tokens |
| "Invalid recipient" | Address format doesn't match target chain | Verify format: 0x for EVM, Base58 for Solana |
| "Cannot swap a token to itself" | tokenIn equals tokenOut | Use different tokens |
| "No liquidity" | DEX has insufficient liquidity | Try smaller amount or different token pair |
| "Swap failed" | Slippage exceeded or DEX error | Retry with higher slippage tolerance |
| "Invalid strategyId" | Strategy no longer available | Re-run `discoverYieldStrategies` for current IDs |
| "No x402 payment requirements found" | URL is not x402-enabled | Verify the URL supports x402 |
| "tool not found" | Misspelled or unavailable tool | Run `fdx wallet` or `fdx prism` to list available tools |
| "provider not found" | Prism onboarding incomplete | Set account type with `updateAccountType` |

## Diagnostic Commands

When investigating issues, collect this information:

```bash
fdx --version                         # CLI version (works without auth)
fdx status                            # authentication state
fdx wallet getWalletOverview --chainKey <chain>  # wallet state on relevant chain
```

## Reporting Issues

When you encounter a problem you cannot resolve, report it with `fdx wallet reportIssue`. Include:

- What you were trying to do
- The exact error message
- The CLI version (from `getAppVersion`)
- The chain and operation involved

Use `fdx wallet reportIssue --help` for the exact parameters.
