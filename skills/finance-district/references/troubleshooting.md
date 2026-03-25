# Troubleshooting & Support

## Authentication Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| "not authenticated" | No valid session | `fdx login --email <email>` then `fdx verify --code <OTP>` |
| "token expired" | Access token expired but refresh token available | Auto-refreshes on next `fdx wallet` — no action needed |
| "SESSION_EXPIRED" | Both access and refresh tokens expired | Run `fdx login` again |
| "AUTH_REFRESH_FAILED" | Token refresh failed | Run `fdx login` to re-authenticate |

## Operation Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| "Insufficient balance" | Not enough tokens for the operation | Check balance with `getWalletOverview`; fund wallet or swap tokens |
| "Invalid recipient" | Address format doesn't match the target chain | Verify address format (0x for EVM, Base58 for Solana) |
| "Cannot swap a token to itself" | `tokenIn` equals `tokenOut` | Use different tokens |
| "No liquidity" | DEX has insufficient liquidity for the pair/amount | Try smaller amount or different token pair |
| "Swap failed" | Slippage exceeded or other DEX error | Retry with higher `--maxSlippageBps` |
| "Invalid strategyId" | Strategy no longer available | Re-run `discoverYieldStrategies` for current IDs |
| "No x402 payment requirements found" | URL is not an x402-enabled endpoint | Verify the URL supports x402 |

## Diagnostic Information

Collect this information before reporting issues:

```bash
# CLI version
fdx wallet getAppVersion

# Authentication state
fdx status

# Wallet state on relevant chain
fdx wallet getWalletOverview --chainKey <chain>
```

## Reporting Issues

When you encounter a problem you cannot resolve:

```bash
fdx wallet reportIssue \
  --title "<short summary>" \
  --description "<detailed description including steps to reproduce, error messages, and diagnostic info>" \
  --labels "bug"
```

Include in the description:
- What you were trying to do
- The exact error message
- The CLI version (`getAppVersion`)
- The chain and operation involved

Available labels: `bug`, `mcp-tool`, `testing`, or comma-separated combinations.
