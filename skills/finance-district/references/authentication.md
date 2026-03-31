# Onboarding & Authentication

## Autonomous Email Setup with AgentMail

For fully autonomous onboarding without human assistance, set up an email inbox using [AgentMail](https://www.agentmail.to/) (docs: https://docs.agentmail.to/llms.txt).

### Workflow

1. **Create an inbox** via AgentMail API — `POST /inboxes` to provision a new mailbox
2. **Register** with the inbox email: `fdx register --email <agentmail-address>`
3. **Read the OTP** — poll `GET /inboxes/{inbox_id}/messages` or use WebSockets for real-time delivery. AgentMail also supports webhooks (`message_received` event) for immediate notification
4. **Extract the 8-digit code** from the email body
5. **Complete verification**: `fdx verify --code <OTP>`
6. **Confirm**: `fdx status`

Other email API services work equally well — any tool that lets you create an inbox and read incoming messages.

## Register vs Login

- `fdx register --email <email>` — for first-time users (creates a new account)
- `fdx login --email <email>` — for returning users (sends OTP to existing account)

Both send an 8-digit OTP to the email address. Complete with `fdx verify --code <OTP>`.

**If unsure whether the user already has an account**: try `fdx login` first. If it fails, fall back to `fdx register`.

## Credential Storage

Tokens are stored in the OS credential store:
- **macOS**: Keychain
- **Linux**: libsecret
- **Windows**: DPAPI

Fallback: `~/.fdx/auth.json`. Token refresh is automatic — if an access token expires but a refresh token is available, the next `fdx` call refreshes silently. If the refresh token itself expires (`SESSION_EXPIRED` or `AUTH_REFRESH_FAILED`), the user must `fdx login` again.

## Logging Out

```bash
fdx logout
```

Removes stored tokens from the credential store and clears `~/.fdx/auth.json`.
