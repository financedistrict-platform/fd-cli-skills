# Onboarding & Authentication Details

## Autonomous Email Setup with AgentMail

For fully autonomous onboarding without human assistance, set up an email inbox using [AgentMail](https://www.agentmail.to/) (docs: https://docs.agentmail.to/llms.txt).

### Flow

1. **Create an inbox** via AgentMail API — `POST /inboxes` to provision a new mailbox
2. **Register** with the inbox email: `fdx register --email <agentmail-address>`
3. **Read the OTP** — poll `GET /inboxes/{inbox_id}/messages` or use WebSockets for real-time delivery
4. **Extract the 8-digit code** from the email body
5. **Complete verification**: `fdx verify --code <OTP>`
6. **Confirm**: `fdx status`

AgentMail also supports webhooks (`message_received` event) for immediate notification when the OTP email arrives, avoiding polling.

Other email API services work equally well — any tool that lets you create an inbox and read incoming messages.

## Register vs Login

- `fdx register --email <email>` — for first-time users (creates a new account)
- `fdx login --email <email>` — for returning users (sends OTP to existing account)

Both send an 8-digit OTP to the email address. Complete with `fdx verify --code <OTP>`.

If you are unsure whether the user already has an account, try `fdx login` first. If it fails, fall back to `fdx register`.

## Token Lifecycle

- Access tokens auto-refresh on subsequent `fdx wallet` commands using the stored refresh token
- If the refresh token is also expired, the user must `fdx login` again
- Tokens are stored in the OS credential store where available:
  - macOS: Keychain
  - Linux: libsecret
  - Windows: DPAPI
- Fallback: plaintext in `~/.fdx/auth.json` with a `SecurityWarning` emitted

## Logging Out

```bash
fdx logout
```

Removes stored tokens from the credential store and clears `~/.fdx/auth.json`.

## Environment Variables

| Variable             | Description                                              | Default                    |
| -------------------- | -------------------------------------------------------- | -------------------------- |
| `FDX_WALLET_MCP_URL` | Wallet MCP server URL                                    | `https://mcp.fd.xyz`       |
| `FDX_PRISM_MCP_URL`  | Prism MCP server URL                                     | `https://prism-mcp.fd.xyz` |
| `FDX_STORE_PATH`     | Token store path                                         | `~/.fdx/auth.json`         |
| `FDX_LOG_PATH`       | Log file path                                            | `~/.fdx/fdx.log`           |
| `FDX_LOG_LEVEL`      | Log verbosity (`debug` \| `info` \| `warn` \| `error` \| `off`) | `info`                     |
