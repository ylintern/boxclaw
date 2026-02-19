# ebox Integration Audit — 2026-02-19

Scope: repository-level static audit for integration wiring, dependency references, and required credentials.

## Method

Commands used:

```bash
rg -n "api[_-]?key|token|secret|rpc|coingecko|goldsky|alchemy|the graph|dune|wallet" src docs roles skills workshops .env.example .env.ebox.example
rg -n "OPENROUTER_API_KEY|OPENAI_API_KEY|ANTHROPIC|GEMINI_API_KEY|GLM_API_KEY|ZAI_API_KEY|BRAVE_API_KEY|COMPOSIO" src docs .env.example
```

## Key Findings

### 1) Cross-integration gaps (missing files / unresolved role dependencies)

- `roles/strategist.md` requires `skills/backtester.md`, but that file does not exist.
- `roles/swap-arb-v1.1.md` requires `skills/wallet-manager.md`, but that file does not exist.
- `IDENTITY boxclaw.md` references operational core files (`SOUL.md`, `SKILL.md`, `STATE.md`, `DECISIONS.md`, `BACKLOG.md`) that are not present at repo root.

Impact:
- Role orchestration is partially declarative but not executable end-to-end from local repo artifacts.

### 2) Service wiring status vs requested stack

#### Detected in runtime code (or config schema/docs-backed runtime)

- LLM providers: OpenAI / Anthropic / Gemini / OpenRouter / GLM / ZAI and others.
- Gateway/channel integrations: Telegram, Discord, Slack, Matrix, Mattermost, WhatsApp, Signal, etc.
- Tool integrations: Composio, Brave web search.

#### Present only in role/skill markdown (not first-class Rust runtime integrations)

- Dune Analytics (skill docs).
- The Graph (skill docs).
- CoinGecko (workshop content patterns).
- Goldsky RPC/Subgraph references (external workflow context).
- Wallet Auth service references (external workflow context).

Impact:
- For ebox quant workflows, these services are documented but not yet implemented as native Rust tools/providers in `src/`.

### 3) Secrets posture and immediate risk

- Credentials were shared in plaintext in conversation context.
- Required action: rotate all shared keys/tokens immediately and treat old values as compromised.
- Repo scan found no committed copies of the provided raw key strings.

## Credentials Matrix — What you still need to provide

Legend:
- **Provided (rotate required)**: value was supplied out-of-band but must be replaced.
- **Missing**: not provided yet and required for target workflow.
- **Optional**: only needed if enabling that integration.

| Service | Env/Config Key | Status | Notes |
|---|---|---|---|
| CoinGecko | `COINGECKO_API_KEY` | Provided (rotate required) | Needed for price API workflows. |
| The Graph | `THE_GRAPH_API_KEY` | Provided (rotate required) | Needed for Graph-hosted queries. |
| Alchemy | `ALCHEMY_API_KEY`, `ALCHEMY_UNICHAIN_RPC_URL` | Provided (rotate required) | Keep URL/key local-only. |
| Wallet Auth | `WALLET_AUTH_API_KEY`, `WALLET_AUTH_ID` | Provided (rotate required) | External auth workflow dependency. |
| Goldsky | `GOLDSKY_RPC_URL`, `GOLDSKY_SUBGRAPH_ID` | Provided (rotate required) | External RPC/subgraph dependency. |
| OpenAI | `OPENAI_API_KEY` | Provided (rotate required) | Runtime provider key. |
| Anthropic/Claude | `ANTHROPIC_API_KEY`, `ANTHROPIC_ORG_ID` | Provided (rotate required) | Org ID alone is insufficient without active API key. |
| Gemini | `GEMINI_API_KEY_PRIMARY` / `GEMINI_API_KEY_SECONDARY` | Provided (rotate required) | Prefer one active key + one standby. |
| Dune Analytics | `DUNE_API_KEY` | **Missing** | Required by `skills/dune-analytics.md` workflows. |
| OpenRouter (default ZeroClaw path) | `OPENROUTER_API_KEY` | **Missing** (if using default provider) | Required unless `default_provider` switched to another provider. |
| Brave Search | `BRAVE_API_KEY` | Optional | Needed only when `web_search.provider = "brave"`. |
| Composio | `COMPOSIO_API_KEY` (`config.composio.api_key`) | Optional | Needed only when Composio tool is enabled. |
| Telegram channel | `TELEGRAM_BOT_TOKEN` | Optional | Needed only if Telegram channel enabled. |
| Discord channel | `DISCORD_BOT_TOKEN` | Optional | Needed only if Discord channel enabled. |
| Slack channel | `SLACK_BOT_TOKEN` + app token | Optional | Needed only if Slack channel enabled. |
| WhatsApp channel | access/verify/app secret | Optional | Needed only if WhatsApp channel enabled. |

## Next Steps (execution order)

1. **Security first**
   - Rotate every key previously exposed in plaintext.
   - Populate new values in local `.env.local` only.

2. **Complete missing credential set**
   - Provide `DUNE_API_KEY`.
   - Provide `OPENROUTER_API_KEY` (or explicitly set/use another default provider key).

3. **Close integration/documentation gaps**
   - Add missing role dependency files:
     - `skills/backtester.md` (or update strategist dependency to existing role path).
     - `skills/wallet-manager.md` (or update swap-arb dependencies).
   - Either add root files expected by identity (`SOUL.md`, `SKILL.md`, `STATE.md`, `DECISIONS.md`, `BACKLOG.md`) or update identity references to existing equivalents.

4. **Implement native tools for quant stack (recommended)**
   - Add first-class Rust tools under `src/tools/` for:
     - The Graph query execution
     - Dune query execution
     - CoinGecko price fetch
   - This removes dependence on markdown-only operational assumptions.

5. **Validation after key setup**

```bash
# verify remotes
 git remote -v

# verify secrets file is ignored
 git check-ignore .env.local

# verify no raw secrets in tracked changes
 git diff --cached

# runtime sanity
 cargo test
 zeroclaw status
 zeroclaw doctor
```
