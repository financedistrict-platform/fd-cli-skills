# Generating Unique IDs (Request-Id, Idempotency-Key)

Agentic-commerce merchants require two unique-per-call header values:

- **`Request-Id`** — unique per HTTP call. Used for tracing and log correlation.
- **`Idempotency-Key`** — unique per logical operation (POST/PUT create/update/complete). Reusing the same key replays the same result — useful for retries after network failures, dangerous otherwise.

Both must be **distinct random strings**, ideally UUIDv4. Don't reuse across calls. Don't invent predictable values ("req-1", "req-2"). When retrying a genuinely idempotent operation, reuse the same Idempotency-Key but give the Request-Id a fresh value.

## Cross-platform one-liners

Pick whichever works in your environment. All produce a UUIDv4-like string.

### Python (works everywhere Python is installed — Windows, macOS, Linux, WSL)

```bash
python -c "import uuid; print(uuid.uuid4())"
```

Or if `python` isn't on PATH but `python3` is:

```bash
python3 -c "import uuid; print(uuid.uuid4())"
```

**This is the most portable default. Prefer it.**

### Node.js (anywhere Node is installed — Node 14.17+ has `crypto.randomUUID`)

```bash
node -e "console.log(require('crypto').randomUUID())"
```

### PowerShell (Windows native, no Python/Node needed)

```powershell
[guid]::NewGuid().ToString()
```

From a bash shell on Windows (Git Bash, WSL) calling out to PowerShell:

```bash
powershell -NoProfile -Command "[guid]::NewGuid().ToString()"
```

### macOS / Linux (native `uuidgen`)

```bash
uuidgen | tr 'A-Z' 'a-z'
```

Lowercase isn't strictly required but matches what most clients produce.

### Git Bash on Windows

`uuidgen` is not present. Use the Python or Node one-liner above, or PowerShell.

## Picking the right tool for this skill

Detection order (agent should probe in order, use the first that works):

1. `python --version` → use Python one-liner
2. `node --version` → use Node one-liner
3. On Windows (`$OS` = `Windows_NT` or Git Bash) → PowerShell one-liner
4. `uuidgen --help` → native `uuidgen`

Once you know which works in the current environment, stick with it for the whole session — don't re-probe on every call.

## Embedding in curl

Capture once per call with command substitution:

```bash
# Python approach (most portable)
curl -X POST "$URL" \
  -H "Request-Id: $(python -c 'import uuid; print(uuid.uuid4())')" \
  -H "Idempotency-Key: $(python -c 'import uuid; print(uuid.uuid4())')" \
  ...
```

Or generate once into variables for readability:

```bash
REQ_ID=$(python -c 'import uuid; print(uuid.uuid4())')
IDEM_KEY=$(python -c 'import uuid; print(uuid.uuid4())')
curl -X POST "$URL" -H "Request-Id: $REQ_ID" -H "Idempotency-Key: $IDEM_KEY" ...
```

## Fallbacks when nothing is available

If, genuinely, no UUID generator is reachable, a random-enough string works for testing (NOT for production):

```bash
# Epoch + random — good enough for dev, not truly unique at scale
echo "req-$(date +%s)-$RANDOM"
```

Don't use this pattern against real merchants — it can collide under concurrency.
