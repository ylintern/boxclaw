---
name: the-graph
version: 1.0.0
role: Real-time on-chain data via GraphQL subgraph queries — pool state, swaps, positions
requires: SKILL.md, skills/api-key-vault.md
classification: DATA — real-time complement to dune-analytics
---

# 🕸️ The Graph — Role Skill

> The Graph is UniClaw's live pulse on the chain.
> Dune tells you what happened. The Graph tells you what is happening NOW.
> Block-level granularity. Sub-second latency. GraphQL interface.

---

## Role Objective

Answer the question: **"What is the current state of this pool, position, or protocol?"**

Used by:
- `lp-manager.md` — real-time position state and fee growth
- `sentiment-analyst.md` — live volume, tick distribution, liquidity depth
- `backtester.md` — high-resolution swap-level price series
- `swap-arb.md` — live pool prices for arbitrage detection
- UniClaw master — any real-time on-chain lookup

---

## Authentication

Key class: `MEDIUM` (read-only)
Key name: `THE_GRAPH_API_KEY`
Injected via: `api-key-vault.md` — never hardcoded

```python
GRAPH_BASE = "https://gateway.thegraph.com/api/{api_key}/subgraphs/id/{subgraph_id}"

def get_endpoint(subgraph_id: str) -> str:
    api_key = vault.get_scoped_token("THE_GRAPH_API_KEY")
    return GRAPH_BASE.format(api_key=api_key, subgraph_id=subgraph_id)
```

---

## Subgraph Registry

| Protocol | Network | Subgraph ID | Best For |
|----------|---------|------------|---------|
| Uniswap V3 | Ethereum | `FbCGRftH4a3yGgW6h3JCpk8GNP5gF3Uvm8UEbq8iJqJ` | Mainnet pools, positions, swaps |
| Uniswap V3 | Arbitrum | `FQ6JYszEKApsBpAmiHesRsd9Igo1wodCuDMFfhSmRsdB` | Arbitrum pools |
| Uniswap V3 | Base | `43Hwfi3dJSoGpyas9VwNoDAv55yjgGrPpNSmbQZArzFG` | Base pools |
| Uniswap V3 | Optimism | `Cghf4LfVqPiFw6fp6Y5X5Ubc8UpmUhSfJL82zwiBFLaj` | Optimism pools |
| Uniswap V4 | Ethereum | *(use contract events directly — V4 subgraph in progress)* | — |

Always verify subgraph IDs on https://thegraph.com/explorer before deploying to production.

---

## Execution Pattern

```python
import requests
from typing import Any

def query_subgraph(
    subgraph_id: str,
    query: str,
    variables: dict = None
) -> dict:
    """
    Execute a GraphQL query against a subgraph.
    Returns the 'data' field directly.
    """
    endpoint = get_endpoint(subgraph_id)
    payload = {"query": query}
    if variables:
        payload["variables"] = variables

    response = requests.post(
        endpoint,
        json=payload,
        headers={"Content-Type": "application/json"},
        timeout=15
    )
    response.raise_for_status()
    result = response.json()

    # Surface GraphQL errors explicitly — never silently ignore
    if "errors" in result:
        raise GraphQLError(f"Subgraph error: {result['errors']}")

    return result["data"]
```

---

## Core Query Library

### Q1 — Current Pool State

```graphql
# Use for: live tick, price, liquidity, volume snapshot
query PoolState($poolId: ID!) {
  pool(id: $poolId) {
    id
    token0 {
      symbol
      decimals
      derivedETH
    }
    token1 {
      symbol
      decimals
      derivedETH
    }
    feeTier
    sqrtPrice
    tick
    liquidity
    token0Price
    token1Price
    volumeUSD
    feesUSD
    txCount
    totalValueLockedUSD
    poolDayData(first: 7, orderBy: date, orderDirection: desc) {
      date
      volumeUSD
      feesUSD
      txCount
      open
      close
      high
      low
    }
  }
}
```

**Variables:** `{ "poolId": "0x..." }` (lowercase address)

---

### Q2 — Position State (fee growth, liquidity)

```graphql
# Use for: exact fee calculation inputs, position health check
query PositionState($positionId: ID!) {
  position(id: $positionId) {
    id
    owner
    pool {
      id
      tick
      sqrtPrice
      feeGrowthGlobal0X128
      feeGrowthGlobal1X128
    }
    tickLower {
      tickIdx
      feeGrowthOutside0X128
      feeGrowthOutside1X128
      liquidityNet
      liquidityGross
    }
    tickUpper {
      tickIdx
      feeGrowthOutside0X128
      feeGrowthOutside1X128
      liquidityNet
      liquidityGross
    }
    liquidity
    feeGrowthInside0LastX128
    feeGrowthInside1LastX128
    collectedFeesToken0
    collectedFeesToken1
    depositedToken0
    depositedToken1
    withdrawnToken0
    withdrawnToken1
  }
}
```

**Variables:** `{ "positionId": "TOKEN_ID" }` (string, not int)

---

### Q3 — Recent Swaps (price series for vol calc)

```graphql
# Use for: realized volatility, EWMA vol, price impact analysis
query RecentSwaps($poolId: String!, $limit: Int!) {
  swaps(
    first: $limit
    where: { pool: $poolId }
    orderBy: timestamp
    orderDirection: desc
  ) {
    timestamp
    sqrtPriceX96
    tick
    amount0
    amount1
    amountUSD
    sender
  }
}
```

**Variables:** `{ "poolId": "0x...", "limit": 1000 }`
**Notes:** Max 1000 per query. For longer history, paginate with `timestamp_lt`.

---

### Q4 — Tick Distribution (liquidity map)

```graphql
# Use for: range selection, thin-book detection, sandwich risk assessment
query TickDistribution($poolId: String!, $tickLower: BigInt!, $tickUpper: BigInt!) {
  ticks(
    where: {
      pool: $poolId
      tickIdx_gte: $tickLower
      tickIdx_lte: $tickUpper
    }
    orderBy: tickIdx
    orderDirection: asc
    first: 1000
  ) {
    tickIdx
    liquidityNet
    liquidityGross
    feeGrowthOutside0X128
    feeGrowthOutside1X128
  }
}
```

**Variables:** `{ "poolId": "...", "tickLower": -887272, "tickUpper": 887272 }`

---

### Q5 — All Positions for a Wallet

```graphql
# Use for: portfolio overview, position health batch check
query WalletPositions($owner: String!) {
  positions(
    where: {
      owner: $owner
      liquidity_gt: "0"
    }
    first: 100
    orderBy: collectedFeesToken0
    orderDirection: desc
  ) {
    id
    pool {
      id
      token0 { symbol }
      token1 { symbol }
      feeTier
      tick
      token0Price
    }
    tickLower { tickIdx }
    tickUpper { tickIdx }
    liquidity
    collectedFeesToken0
    collectedFeesToken1
    depositedToken0
    depositedToken1
  }
}
```

**Variables:** `{ "owner": "0x..." }` (lowercase)

---

### Q6 — Pool Hourly OHLCV (for backtesting)

```graphql
# Use for: backtesting price data (complement to Dune)
query PoolHourlyData($poolId: ID!, $startTime: Int!, $endTime: Int!) {
  poolHourDatas(
    where: {
      pool: $poolId
      periodStartUnix_gte: $startTime
      periodStartUnix_lte: $endTime
    }
    orderBy: periodStartUnix
    orderDirection: asc
    first: 1000
  ) {
    periodStartUnix
    open
    close
    high
    low
    volumeUSD
    feesUSD
    txCount
    tick
    sqrtPrice
    liquidity
  }
}
```

**Variables:** `{ "poolId": "...", "startTime": 1700000000, "endTime": 1710000000 }`

---

## Pagination Helper

The Graph returns max 1000 records per query. Use skip for deeper history:

```python
def paginate_query(subgraph_id: str, query: str, variables: dict,
                   result_key: str, max_records: int = 10000) -> list:
    """Paginate through The Graph results using skip."""
    all_records = []
    skip = 0
    batch_size = 1000

    while len(all_records) < max_records:
        variables["first"] = batch_size
        variables["skip"] = skip
        data = query_subgraph(subgraph_id, query, variables)
        batch = data.get(result_key, [])
        if not batch:
            break
        all_records.extend(batch)
        if len(batch) < batch_size:
            break  # last page
        skip += batch_size

    return all_records
```

---

## Price Conversion Helpers

```python
from decimal import Decimal
import math

def sqrt_price_x96_to_price(sqrt_price_x96: int, decimals0: int, decimals1: int) -> Decimal:
    """Convert sqrtPriceX96 to human-readable price of token1 in token0."""
    sqrt_price = Decimal(sqrt_price_x96) / Decimal(2 ** 96)
    raw_price = sqrt_price ** 2
    adjusted = raw_price * Decimal(10 ** decimals0) / Decimal(10 ** decimals1)
    return adjusted

def tick_to_price(tick: int, decimals0: int, decimals1: int) -> Decimal:
    """Convert tick index to price."""
    raw_price = Decimal("1.0001") ** tick
    return raw_price * Decimal(10 ** decimals0) / Decimal(10 ** decimals1)

def price_to_tick(price: Decimal) -> int:
    """Convert price to nearest valid tick."""
    return int(math.log(float(price)) / math.log(1.0001))
```

---

## Data Freshness & Reliability

```
Subgraph indexing lag:
  Ethereum mainnet:  ~15-30 seconds behind head
  Arbitrum / Base:   ~5-15 seconds behind head

Reliability:
  Hosted service (The Graph Network):  99.5% uptime
  Self-hosted:  depends on infrastructure

Fallback strategy:
  If subgraph query fails → retry once after 10s
  If second attempt fails → fall back to direct RPC eth_call
  If RPC also fails → halt, alert Sensei
```

---

## What to Return to UniClaw

```
GRAPH DATA REPORT
══════════════════
Subgraph:    [name + network]
Query:       [template name]
Latency:     [ms]
Block lag:   [~N seconds behind]
Records:     [N]

KEY DATA
  [Most relevant fields for the requesting agent]
  Example for Pool State:
    Current tick:      -201423
    Token0 price:      $3,247.84 (WETH)
    TVL:               $18.4M
    24h volume:        $2.1M
    24h fees:          $1,050
    7d avg daily vol:  $1.8M

Status:  [FRESH | STALE (>60s lag) | FAILED]
```

---

## Error Handling

```python
class GraphErrors:
    SUBGRAPH_UNAVAILABLE  = "Subgraph not responding — use Dune fallback"
    INDEXING_ERROR        = "Subgraph indexing behind > 5 min — data stale"
    RATE_LIMITED          = "Rate limited — wait 30s, retry"
    AUTH_ERROR            = "API key invalid — check api-key-vault"
    EMPTY_RESULT          = "No data returned — verify pool ID and network"
    GRAPHQL_ERROR         = "Query syntax error — log and fix query"

# Escalation: if The Graph fails for > 5 minutes on a critical lookup,
# switch to blockchain-rpc direct call and alert Sensei
```

---

## The Graph vs Dune — When to Use Each

| Need | Use |
|------|-----|
| Current pool tick / sqrtPrice | The Graph ✅ |
| Live position fee growth | The Graph ✅ |
| Historical OHLCV for backtest | Both (cross-reference) |
| 30d volume trends | Dune ✅ (better SQL aggregations) |
| Whale LP movements | Dune ✅ |
| Gas cost history | Dune ✅ |
| Tick distribution (live) | The Graph ✅ |
| Wallet position list (live) | The Graph ✅ |
| Multi-protocol comparison | Dune ✅ |

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial — real-time data layer | Audit gap identified | 2026-02-18 |
