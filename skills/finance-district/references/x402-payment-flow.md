# x402 Payment Flow — MCP & HTTP Paths

## Path Selection

| Condition                                                        | Path        |
|------------------------------------------------------------------|-------------|
| HTTP endpoint returns `402 Payment Required`                     | **HTTP path** via `fdx wallet getX402Content` |
| MCP tool returns `isError: true` with `x402Version` + `accepts`  | **MCP path** — 3-step flow below |

Use whichever path matches the resource type. If the MCP client does not support `_meta`, fall back to HTTP path.

---

## HTTP Path

```bash
fdx wallet getX402Content --url <x402-protected-endpoint>
```

CLI handles negotiation, signing, and retry automatically. Use `--help` for parameter details.

---

## MCP Path — 3-Step Flow

```
Agent --1--> Paid Tool Server (call tool normally, no payment)
       <---- PaymentRequired error (isError: true, x402Version, accepts[])

Agent --2--> Agent Wallet MCP Server (authorizePayment tool)
       <---- Signed authorization (EIP-3009)

Agent --3--> Paid Tool Server (retry tool + _meta["x402/payment"])
       <---- Tool result + _meta["x402/payment-response"] (settlement receipt)
```

### Step 1: Call Paid Tool

Call the tool normally. On PaymentRequired, the response contains payment requirements in:
- `content[0].text` — raw JSON string (use this for Step 2)
- `structuredContent` — parsed object (for inspection)

### Step 2: Authorize Payment

Call `authorizePayment` on the Agent Wallet MCP Server. The tool's own description has full parameter details.

**CRITICAL — what the tool description won't tell you:**
- `paymentRequirementsResponseJson` must be the **raw JSON string** from `content[0].text` — do NOT parse it into an object first

### Step 3: Retry with Payment

From the `authorizePayment` response, extract **`authorization.paymentPayload`** only.

**DO NOT** use the full `authorization` object — only `paymentPayload`.

Pass it as `_meta["x402/payment"]` when retrying the tool. `_meta` goes in MCP request **params**, not inside tool `arguments`:

```json
{
  "method": "tools/call",
  "params": {
    "name": "<tool_name>",
    "arguments": { "...same args as Step 1..." },
    "_meta": {
      "x402/payment": { "...authorization.paymentPayload object..." }
    }
  }
}
```

On success, `_meta["x402/payment-response"]` contains `transaction` hash verifiable on block explorer.
