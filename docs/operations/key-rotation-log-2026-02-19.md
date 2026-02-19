# Key Rotation Log — 2026-02-19 (Redacted)

Scope: response to plaintext credential exposure in operator chat.

## Actions Executed

- Initialized local secret workspace file: `.env.local` (git-ignored).
- Confirmed `.env.local` ignore status to prevent accidental commit.
- Updated secret protocol and audit docs to require chat-safe handling and rotation-first recovery.

## Rotation / Revocation Status (Redacted)

| Service Group | Rotation | Revocation | Notes |
|---|---|---|---|
| LLM provider keys | Required | Pending operator at provider dashboard | Values were exposed in chat; replace with new keys. |
| RPC / data provider keys | Required | Pending operator at provider dashboard | Includes chain/data API credentials. |
| Channel bot tokens | Required | Pending operator at provider dashboard | Discord + Telegram marked ready but rotate before production. |
| Infra / edge credentials | Required | Pending operator at provider dashboard | Rotate API tokens and cert-related secrets. |

## Required Manual Steps (Operator)

1. Revoke previously exposed keys in each provider console.
2. Generate replacement keys/tokens.
3. Update local `.env.local` with new values.
4. Re-encrypt config-bound values into `enc2:` form before writing to tracked config artifacts.
5. Validate with:

```bash
git check-ignore .env.local
git diff --cached
zeroclaw status
zeroclaw doctor
```

## Compliance Note

This log intentionally contains no key material, tokens, IDs, URLs-with-secrets, or personal data.
