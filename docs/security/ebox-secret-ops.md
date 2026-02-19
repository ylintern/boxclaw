# ebox Secret Operations Protocol

Last updated: 2026-02-19

This protocol defines how to store, rotate, and use private keys/API keys for ebox integrations.

## Non-Negotiable Rules

- Never commit raw secrets to git (repo, docs, tests, issues, PRs).
- Never put raw secrets in `STATE*.md`, `DECISIONS.md`, or role outputs.
- Keep runtime secrets in local-only files (`.env.local`) or encrypted values (`enc2:`).
- Rotate any key that was previously pasted in plaintext.

## Supported Secret Names

Use these environment names locally:

- `COINGECKO_API_KEY`
- `THE_GRAPH_API_KEY`
- `DUNE_API_KEY`
- `ALCHEMY_API_KEY`
- `ALCHEMY_UNICHAIN_RPC_URL`
- `WALLET_AUTH_API_KEY`
- `WALLET_AUTH_ID`
- `GOLDSKY_RPC_URL`
- `GOLDSKY_SUBGRAPH_ID`
- `OPENROUTER_API_KEY`
- `OPENAI_API_KEY`
- `ANTHROPIC_ORG_ID`
- `ANTHROPIC_API_KEY`
- `GEMINI_API_KEY_PRIMARY`
- `GEMINI_API_KEY_SECONDARY`

## Local Setup (Non-Tracked)

1. Copy template:

```bash
cp .env.ebox.example .env.local
```

2. Fill `.env.local` with actual values.
3. Confirm `.env.local` stays untracked:

```bash
git status --short
```

## Encrypt for ZeroClaw Config

For keys used by the runtime config, store encrypted values only (`enc2:`):

1. Load `.env.local` into shell (local session only).
2. Use project secret-store workflow to produce encrypted `enc2:` values.
3. Put only encrypted outputs into config fields.

Reference: `src/security/secrets.rs` (`enc2:` ChaCha20-Poly1305 format).

## Service-Specific Usage Notes

- CoinGecko: use `COINGECKO_API_KEY` only in HTTP client headers/query construction at runtime; no hardcoded key literals.
- The Graph: use `THE_GRAPH_API_KEY` only through injected runtime env/config.
- Alchemy/Goldsky RPC: keep full URLs in `.env.local`; never in committed markdown/config.
- LLM providers (OpenAI/Anthropic/Gemini): runtime env injection only.

## Rotation Procedure

- Immediate rotate for any key previously posted in plaintext.
- Update `.env.local`.
- Re-encrypt values for config if needed.
- Revoke old provider keys.
- Record redacted rotation event in ops notes (no key material).

## Validation Checklist

```bash
# 1) No accidental raw secrets in tracked files
rg -n "(sk-|AIza|x_cg_demo_api_key=|api_key\s*[:=])" .

# 2) Ensure local secret file is ignored
git check-ignore .env.local

# 3) Verify clean staging area before commit
git diff --cached
```
