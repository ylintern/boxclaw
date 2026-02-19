---
name: dune-analytics
version: 1.0.0
role: On-chain SQL analytics via Dune API — historical pool data, LP flows, wallet activity
requires: SKILL.md, skills/api-key-vault.md
classification: DATA — feed for backtester and sentiment-analyst
---

# 📊 Dune Analytics — Role Skill

> Dune is UniClaw's historical truth layer.
> When you need to know what happened — volume, fees, LP flows, wallet behavior —
> Dune is the source. The Graph is for NOW. Dune is for HISTORY.

---

## Role Objective

Answer the question: **"What has this pool, wallet, or protocol done over time?"**

Used by:
- `backtester.md` — fetches historical price and volume data for simulations
- `sentiment-analyst.md` — fetches on-chain signals and regime context
- UniClaw master — ad-hoc pool research and position sizing decisions

---

## Authentication

Key class: `MEDIUM` (read-only)
Key name: `DUNE_API_KEY`
Injected via: `api-key-vault.md` — never hardcoded

```python
headers = {
    "X-Dune-API-Key": vault.get_scoped_token("DUNE_API_KEY"),
    "Content-Type": "application/json"
}
base_url = "https://api.dune.com/api/v1"
```

---

## Execution Pattern

Dune uses an async execute → poll → fetch pattern. Never assume instant results.

```python
def run_dune_query(query_id: int, params: dict) -> pd.DataFrame:
    """
    Full lifecycle: execute → poll → fetch results.
    """

    # Step 1: Execute
    execute_url = f"{base_url}/query/{query_id}/execute"
    payload = {"query_parameters": params, "performance": "medium"}
    response = requests.post(execute_url, headers=headers, json=payload)
    execution_id = response.json()["execution_id"]

    # Step 2: Poll until complete (max 120s, check every 5s)
    status_url = f"{base_url}/execution/{execution_id}/status"
    for attempt in range(24):
        time.sleep(5)
        status = requests.get(status_url, headers=headers).json()
        if status["state"] == "QUERY_STATE_COMPLETED":
            break
        if status["state"] == "QUERY_STATE_FAILED":
            raise DuneQueryError(f"Query {query_id} failed: {status.get('error')}")

    # Step 3: Fetch results
    results_url = f"{base_url}/execution/{execution_id}/results"
    data = requests.get(results_url, headers=headers).json()
    return pd.DataFrame(data["result"]["rows"])
```

---

## Core Query Library

### Q1 — Pool Daily Volume (last N days)

```sql
-- Query ID: uniclaw_pool_volume
SELECT
    date_trunc('day', block_time)  AS day,
    COUNT(*)                        AS swap_count,
    SUM(amount_usd)                 AS volume_usd,
    SUM(amount_usd * fee_rate)      AS fees_usd,
    AVG(price_after / price_before - 1) AS avg_price_impact
FROM uniswap_v3_ethereum.trades
WHERE pool = '{{pool_address}}'
  AND block_time >= NOW() - INTERVAL '{{days}}' DAY
GROUP BY 1
ORDER BY 1 ASC
```

**Params:** `pool_address` (checksummed), `days` (int, max 365)
**Used by:** backtester (volume data), sentiment-analyst (vol spike detection)

---

### Q2 — LP Position History for Wallet

```sql
-- Query ID: uniclaw_wallet_lp_history
SELECT
    p.id                           AS token_id,
    p.pool                         AS pool_address,
    p.tick_lower,
    p.tick_upper,
    p.liquidity,
    p.deposited_token0,
    p.deposited_token1,
    p.withdrawn_token0,
    p.withdrawn_token1,
    p.collected_fees_token0,
    p.collected_fees_token1,
    (p.collected_fees_token0 * t0.derived_eth * eth.price_usd +
     p.collected_fees_token1 * t1.derived_eth * eth.price_usd) AS fees_usd_approx,
    p.transaction                  AS open_tx
FROM uniswap_v3_ethereum.positions p
JOIN uniswap_v3_ethereum.tokens t0 ON t0.id = p.token0
JOIN uniswap_v3_ethereum.tokens t1 ON t1.id = p.token1
CROSS JOIN (SELECT price_usd FROM prices.usd WHERE symbol='WETH' ORDER BY minute DESC LIMIT 1) eth
WHERE p.owner = LOWER('{{wallet_address}}')
  AND p.liquidity > 0
ORDER BY fees_usd_approx DESC
```

**Params:** `wallet_address`
**Used by:** UniClaw master (portfolio review), lp-manager (fee tracking)

---

### Q3 — Pool Tick Activity Heatmap (liquidity concentration)

```sql
-- Query ID: uniclaw_tick_heatmap
SELECT
    tick_lower,
    tick_upper,
    SUM(amount) AS total_liquidity
FROM uniswap_v3_ethereum.mint_events
WHERE pool = '{{pool_address}}'
  AND block_time >= NOW() - INTERVAL '{{days}}' DAY
GROUP BY 1, 2
ORDER BY total_liquidity DESC
LIMIT 50
```

**Params:** `pool_address`, `days`
**Used by:** strategist (where is liquidity concentrated?), sentiment-analyst

---

### Q4 — Whale LP Movements (smart money signals)

```sql
-- Query ID: uniclaw_whale_lp_moves
SELECT
    block_time,
    event_type,      -- 'mint' or 'burn'
    owner,
    tick_lower,
    tick_upper,
    amount_usd
FROM (
    SELECT block_time, 'mint' AS event_type, owner,
           tick_lower, tick_upper, amount_usd
    FROM uniswap_v3_ethereum.mint_events
    WHERE pool = '{{pool_address}}' AND amount_usd > {{min_usd}}
    UNION ALL
    SELECT block_time, 'burn' AS event_type, owner,
           tick_lower, tick_upper, amount_usd
    FROM uniswap_v3_ethereum.burn_events
    WHERE pool = '{{pool_address}}' AND amount_usd > {{min_usd}}
) combined
WHERE block_time >= NOW() - INTERVAL '{{days}}' DAY
ORDER BY block_time DESC
```

**Params:** `pool_address`, `min_usd` (whale threshold, default 50000), `days`
**Used by:** sentiment-analyst (smart money signal)

---

### Q5 — Historical Price Series (for backtesting)

```sql
-- Query ID: uniclaw_price_series
SELECT
    date_trunc('hour', block_time) AS hour,
    AVG(price_after)               AS price_close,
    MIN(price_after)               AS price_low,
    MAX(price_after)               AS price_high,
    SUM(amount_usd)                AS volume_usd
FROM uniswap_v3_ethereum.trades
WHERE pool = '{{pool_address}}'
  AND block_time BETWEEN '{{date_start}}' AND '{{date_end}}'
GROUP BY 1
ORDER BY 1 ASC
```

**Params:** `pool_address`, `date_start`, `date_end` (ISO 8601)
**Used by:** backtester (primary price source)

---

### Q6 — Gas Cost History

```sql
-- Query ID: uniclaw_gas_history
SELECT
    date_trunc('day', block_time)  AS day,
    AVG(gas_price / 1e9)           AS avg_gwei,
    PERCENTILE_CONT(0.5)
        WITHIN GROUP (ORDER BY gas_price / 1e9) AS median_gwei,
    PERCENTILE_CONT(0.9)
        WITHIN GROUP (ORDER BY gas_price / 1e9) AS p90_gwei
FROM ethereum.transactions
WHERE block_time >= NOW() - INTERVAL '{{days}}' DAY
GROUP BY 1
ORDER BY 1 ASC
```

**Params:** `days`
**Used by:** sentiment-analyst (rebalance cost forecasting)

---

## Data Quality Rules

```python
VALIDATION_RULES = {
    "volume_usd": lambda x: x >= 0,
    "price_close": lambda x: x > 0,
    "fees_usd": lambda x: x >= 0,
    "liquidity": lambda x: x >= 0,
    "completeness": lambda df: df.isnull().sum().sum() / df.size < 0.05  # < 5% null
}

def validate_dune_result(df: pd.DataFrame, query_name: str) -> pd.DataFrame:
    for col, rule in VALIDATION_RULES.items():
        if col in df.columns:
            bad_rows = df[~df[col].apply(rule)]
            if len(bad_rows) > 0:
                log_warning(f"Dune {query_name}: {len(bad_rows)} invalid rows in {col}")
                df = df[df[col].apply(rule)]  # drop invalid rows

    if not VALIDATION_RULES["completeness"](df):
        raise DataQualityError(f"Dune {query_name}: too many nulls — data unreliable")

    return df
```

**Always cross-reference critical data (price series) with The Graph before backtest.**

---

## Local SQLite Cache

Don't re-fetch data that was already pulled this session.

```python
def get_or_fetch(query_id: int, params: dict, cache_ttl_hours: int = 6) -> pd.DataFrame:
    cache_key = f"dune_{query_id}_{hash(str(params))}"
    cached = zeroclaw_memory.get(cache_key)

    if cached and (now() - cached.timestamp).hours < cache_ttl_hours:
        return cached.data  # use cache

    result = run_dune_query(query_id, params)
    zeroclaw_memory.set(cache_key, result)
    return result
```

Cache TTL guidelines:
- Price series (historical, won't change): 24h
- Volume / fee data: 6h
- Whale movements: 1h
- Live gas prices: no cache

---

## What to Return to UniClaw

```
DUNE DATA REPORT
═════════════════
Query:       [name — e.g. "Pool Daily Volume"]
Pool:        [address — last 8 chars shown]
Period:      [start → end]
Rows:        [N]
Nulls:       [N] ([N]%)
Status:      [COMPLETE | FAILED | CACHED]
Latency:     [Ns]

KEY FINDINGS
  [Most relevant numbers for the requesting agent]
  Example:
    30d avg volume:   $2.4M/day
    30d total fees:   $47,200
    Volume spike:     Feb 14 — 4.2× average (likely ETH move)
    Data quality:     ✅ Clean (0.2% nulls)

Data available for: [backtester | sentiment-analyst | master]
```

---

## Error Handling

```python
class DuneErrors:
    QUERY_FAILED    = "Query execution failed — check SQL syntax or params"
    TIMEOUT         = "Query timed out after 120s — try smaller date range"
    RATE_LIMITED    = "Rate limited — wait 60s, then retry"
    AUTH_ERROR      = "API key invalid — check api-key-vault"
    DATA_QUALITY    = "Too many nulls — data unreliable for this period"

# On any error: log, report to master, suggest alternative data source
# Never silently return empty DataFrame — always surface the error
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial — historical data layer | Audit gap identified | 2026-02-18 |
