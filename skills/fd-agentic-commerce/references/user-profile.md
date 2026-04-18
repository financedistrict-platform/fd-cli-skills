# User Profile — Persistent Memory for Faster Checkouts

**This file is a fallback.** If the agent's runtime provides a native memory system (Claude Code's `CLAUDE.md` hierarchy, Claude.ai memory, a `Memory` tool, etc.), prefer that — it's the user's single source of truth across all their skills and agents. Only use the skill-local file below when no harness memory is available, or when the user explicitly asks to keep shopping details separate.

When the harness has native memory, map the same concepts (buyer, addresses, recipients, preferences) into whatever structure that memory uses. Don't maintain two copies.

---

## Fallback: skill-local profile file

The skill can maintain a single markdown file with the user's shopping preferences and shipping details, so repeat checkouts don't require re-typing addresses. Use this only when no better memory option exists.

## File location

```
~/.claude/skills/fd-agentic-commerce/profile.md
```

(`~` = user home on macOS/Linux, `%USERPROFILE%` on Windows.)

## Format

Plain markdown with YAML frontmatter. Human-readable and editable. Example:

```markdown
---
buyer:
  name: Janno Jarv
  email: j.jaerv@1stdigital.com
  phone: "+372 5803 3175"

addresses:
  - label: home
    default: true
    recipient: Janno Jarv
    line_one: Järveoja tee 8
    line_two: null
    city: Veibi
    state: Harjumaa
    postal_code: "52220"
    country: EE

  - label: office
    recipient: Janno Jarv / FD Technologies
    line_one: Tornimäe 5
    city: Tallinn
    postal_code: "10145"
    country: EE

recipients:
  - label: brother
    name: Max Jarv
    email: max@example.com
    phone: null
    default_address: home    # or inline address block

preferences:
  payment_network: eip155:8453    # prefer Base mainnet for payments
  payment_asset: USDC
  currency_display: USD           # show totals in USD even when merchant is EUR

last_used_at: "2026-04-18T12:30:00Z"
---

# Shopping profile

Freeform notes the user has added (optional). Agents may read but should not
overwrite this section unless the user asks.
```

All fields are optional. Start empty; build up as the user confirms new details.

## Reading

Before asking for shipping info (§4.3 of SKILL.md), read the file. If it doesn't exist, skip silently — the user hasn't shopped via this skill before.

Present what's there as a confirmation, not a form:

> Use your home address (Järveoja tee 8, Veibi, Estonia) and the Janno Jarv / j.jaerv@1stdigital.com details?

## Writing / updating

**Never write without explicit user consent.** After a successful checkout, if *any* info was new or different from what was on file:

> Want me to save these for next time?
> - Update email to <new>?
> - Save shipping address as a new entry labeled "<suggest a label>"?
> - Remember <recipient name> as a gift recipient?

Write only the fields the user confirms. Preserve anything else in the file untouched.

If the file doesn't exist yet, create it on first confirmed save with the frontmatter block only (no freeform notes).

## Security & privacy

- This file can contain PII (name, address, phone, email). It is local-only and never transmitted except as part of a checkout request the user confirmed.
- Do not sync this file to a public repo. The skill assumes it lives in the user's home directory under `.claude/`.
- If the user asks "what do you know about me?" read the file and show them. If they ask to wipe it, delete the file entirely.
- Do NOT store anything payment-related beyond the `payment_network` / `payment_asset` preferences (those are just routing hints). Never store wallet private keys, session tokens, or API keys — those live elsewhere (`fdx` CLI manages its own credentials).

## Multiple recipients

The `recipients[]` array is useful when the user regularly ships gifts — it associates a name with a preferred address without requiring them to dictate it each time:

> "Buy a gift for my brother" → look up `recipients[label=brother]` → "Shipping to Max Jarv at your home address — ok, or different?"

If a recipient has a `default_address` that's a label, resolve it from `addresses[]`.

## Agent behavior summary

1. Read profile at start of checkout flow (before §4.3).
2. Propose the best match; never hard-fail if the file is missing.
3. After checkout, ask once whether to save new info. One yes/no question, not a form.
4. Respect "don't save this one" — the user may not want every address remembered.
