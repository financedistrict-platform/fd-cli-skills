---
name: smart-accounts
description: '[DEPRECATED] Smart account deployment and ownership management tools (deploySmartAccount, manageSmartAccountOwnership) have been removed from the MCP server. Smart Accounts can still be used as the fromAccountAddress in transferTokens, swapTokens, depositForYield, and withdrawFromYield. See wallet-overview skill to view existing Smart Accounts.'
user-invocable: false
disable-model-invocation: true
allowed-tools: ['Bash(fdx call getWalletOverview*)']
---

# Smart Account Management (Deprecated)

> **Note:** The `deploySmartAccount` and `manageSmartAccountOwnership` tools have been removed from the MCP server. This skill is deprecated.

Smart Accounts that were previously deployed still work and can be used as `--fromAccountAddress` in other tools like `transferTokens`, `swapTokens`, `depositForYield`, and `withdrawFromYield`.

## Viewing Existing Smart Accounts

To see your Smart Accounts and their balances:

```bash
fdx call getWalletOverview
```

To view a specific Smart Account:

```bash
fdx call getWalletOverview --accountAddress 0xSmartAccount... --chainKey ethereum
```
