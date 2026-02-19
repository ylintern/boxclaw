---
name: swap-arb
version: 1.1.0
role: Swap routing, token ratio management, arbitrage detection, MEV protection
requires: SKILL.md, skills/blockchain-rpc.md, skills/wallet-manager.md, skills/the-graph.md
upgrade-from: 1.0.0
---

# 🔄 Swap & Arb — Role Skill v1.1

> Handles token ratio management after fee collection and detects
> arbitrage opportunities between pools.

**v1.1 changes:**
- Added MEV protection (Flashbots Protect routing on mainnet)
- Added sandwich detection abort
- Added cross-chain routing via bridges
- Added 1inch, Uniswap V4 and Jupiter (Solana) routing
- All swaps now routed through blockchain-rpc for execution
- Slippage hard limits enforced (not advisory)

---

## Role Objective

Answer two questions:
1. **After collecting fees, how do I swap back into the right ratio for my position?**
2. **Is there a price discrepancy between pools worth capturing?**

---

## Token Ratio Management (post-fee collection)

After collecting fees, tokens may not be in the right ratio to add back as liquidity.

```
WORKFLOW
────────
1. Receive collected amounts: (token0_amount, token1_amount)
2. Know position range: (tickLower, tickUpper) and currentTick
3. Calculate target ratio (see formula below)
4. Determine swap: which token to sell, how much
5. Check MEV risk (see sandwich detection)
6. Get best route (1inch aggregator for mainnet, Uniswap V4 direct for L2)
7. Enforce slippage hard limits
8. Submit for signing via wallet-manager
9. Execute via blockchain-rpc (MEV-protected on L1)
10. Return new balances to lp-manager for add-liquidity
```

### Ratio Calculation (from v1.0 — unchanged)

```python
from decimal import Decimal

def calculate_needed_ratio(
    current_tick: int,
    tick_lower: int,
    tick_upper: int,
    amount_usd: Decimal
) -> tuple[Decimal, Decimal]:
    """
    Returns: (amount0_usd, amount1_usd) target split.
    """
    sqrt_lower  = tick_to_sqrt_price_decimal(tick_lower)
    sqrt_upper  = tick_to_sqrt_price_decimal(tick_upper)
    sqrt_current = tick_to_sqrt_price_decimal(current_tick)

    if current_tick < tick_lower:
        return (amount_usd, Decimal(0))       # 100% token0
    elif current_tick >= tick_upper:
        return (Decimal(0), amount_usd)       # 100% token1
    else:
        pct_token0 = (sqrt_upper - sqrt_current) / (sqrt_upper - sqrt_lower)
        pct_token1 = Decimal(1) - pct_token0
        return (amount_usd * pct_token0, amount_usd * pct_token1)
```

---

## Swap Routing (v1.1)

### Router Priority

| Chain | Router | Why |
|-------|--------|-----|
| Ethereum L1 | 1inch Fusion | Best price across all DEXes, MEV protection built-in |
| Arbitrum | Uniswap V4 / 1inch | Low gas — can afford comparison |
| Base | Uniswap V4 | Native, deepest liquidity on Base |
| Solana | Jupiter | Best aggregator for Solana ecosystem |

```python
def get_best_route(
    token_in: str,
    token_out: str,
    amount_in: int,
    chain_id: int,
    slippage_bps: int
) -> dict:
    """
    Get best swap route. Returns route with expected output and calldata.
    Compares 1inch vs direct Uniswap V4 and picks better price.
    """
    routes = []

    # 1inch (covers all major DEXes)
    if chain_id in {1, 42161, 8453, 10, 137}:
        oneinch = get_1inch_quote(
            chain_id, token_in, token_out, amount_in, slippage_bps
        )
        routes.append({"source": "1inch", **oneinch})

    # Direct Uniswap V4 quote (compare)
    univ4 = get_uniswap_v4_quote(chain_id, token_in, token_out, amount_in)
    routes.append({"source": "uniswap_v4", **univ4})

    # Pick best output (after gas cost)
    best = max(routes, key=lambda r: r["amount_out_after_gas"])

    log_routing_decision(routes, best)
    return best

def get_1inch_quote(chain_id, token_in, token_out, amount, slippage_bps):
    api_key = vault.get_scoped_token("ONEINCH_API_KEY")
    url = f"https://api.1inch.dev/swap/v6.0/{chain_id}/swap"
    params = {
        "src": token_in,
        "dst": token_out,
        "amount": amount,
        "from": wallet_manager.get_address("operations"),
        "slippage": slippage_bps / 100,
        "disableEstimate": False,
        "allowPartialFill": False,
    }
    response = requests.get(url, headers={"Authorization": f"Bearer {api_key}"}, params=params)
    return response.json()
```

---

## MEV Protection (NEW v1.1)

### Sandwich Detection

Before executing any swap, detect potential sandwiching:

```python
from decimal import Decimal

def detect_sandwich_risk(
    pool_address: str,
    amount_in_usd: Decimal,
    expected_price_impact: Decimal,
    chain_id: int
) -> dict:
    """
    Compares expected price impact vs actual pool depth.
    Returns: {risk_level, recommendation, abort}
    """
    # Get tick distribution from The Graph
    pool_state = the_graph.query("PoolState", pool_id=pool_address)
    tvl = Decimal(pool_state["pool"]["totalValueLockedUSD"])

    # Price impact ratio: our trade vs pool TVL
    impact_ratio = amount_in_usd / tvl

    # Check for anomalous price impact (sign of thin book or manipulation)
    if expected_price_impact > Decimal("0.02") and impact_ratio < Decimal("0.001"):
        # Our trade is tiny but impact is high — suspicious
        return {
            "risk_level": "HIGH",
            "recommendation": "ABORT — price impact anomalously high for trade size",
            "abort": True
        }

    if expected_price_impact > Decimal("0.03"):
        # Any trade with > 3% impact is high risk on mainnet
        return {
            "risk_level": "HIGH",
            "recommendation": "Reduce trade size or use limit order via 1inch Fusion",
            "abort": chain_id == 1  # abort on mainnet, warn on L2
        }

    return {"risk_level": "LOW", "recommendation": "PROCEED", "abort": False}

# Hard slippage limits — from zero-trust section of SKILL.md
MAX_SLIPPAGE = {
    "stable":   Decimal("0.005"),   # 0.5% — USDC/USDT, stablecoin pairs
    "volatile": Decimal("0.010"),   # 1.0% — ETH/USDC, BTC/ETH
    "exotic":   Decimal("0.020"),   # 2.0% — small cap, new pools
}

def enforce_slippage_limit(quoted_out: int, pair_type: str) -> int:
    """Returns amountOutMinimum. Raises if slippage limit violated."""
    max_slip = MAX_SLIPPAGE.get(pair_type, MAX_SLIPPAGE["volatile"])
    min_out = int(Decimal(quoted_out) * (1 - max_slip))
    assert min_out > 0, "amountOutMinimum cannot be 0"
    return min_out
```

---

## Arbitrage Detection (v1.1 — enhanced)

```
SCAN WORKFLOW
─────────────
1. Get spot price for target pair on Pool A (The Graph — live)
2. Get spot price for same pair on Pool B (The Graph — live)
3. Compute gross spread:
     spread = abs(priceA - priceB) / min(priceA, priceB)

4. Compute net profit after all costs:
     gross_profit = spread × trade_size
     gas_cost     = blockchain-rpc gas estimate (both legs)
     bridge_cost  = 0 (same chain) | bridge_fee (cross-chain)
     slippage_est = expected_impact_A + expected_impact_B
     net_profit   = gross_profit - gas_cost - bridge_cost - slippage_est

5. Profitable if:
     net_profit > 0
     AND net_profit > gas_cost × 2  (2× margin for safety)
     AND spread has persisted > 2 blocks (not already arbed)

6. Sandwich check on BOTH legs before proceeding

7. Report to UniClaw master with full breakdown
   NEVER execute arb without master confirmation
```

```python
def scan_arbitrage(
    token_a: str,
    token_b: str,
    pools: list[dict],      # [{"address": "0x...", "chain_id": 1, "fee_tier": 3000}]
    trade_size_usd: Decimal
) -> list[dict]:
    """
    Scan N pools for arbitrage opportunities.
    Returns: sorted list of opportunities by net_profit, best first.
    """
    opportunities = []
    prices = {}

    for pool in pools:
        state = the_graph.query("PoolState", pool_id=pool["address"].lower())
        prices[pool["address"]] = {
            "price": Decimal(state["pool"]["token0Price"]),
            "tvl": Decimal(state["pool"]["totalValueLockedUSD"]),
            "pool": pool
        }

    # Compare all pool pairs
    pool_list = list(prices.items())
    for i in range(len(pool_list)):
        for j in range(i + 1, len(pool_list)):
            addr_a, data_a = pool_list[i]
            addr_b, data_b = pool_list[j]

            spread = abs(data_a["price"] - data_b["price"]) / min(data_a["price"], data_b["price"])
            gross = spread * trade_size_usd

            # Gas (both legs)
            gas_a = blockchain_rpc.gas_cost_usd(data_a["pool"]["chain_id"], "swap")
            gas_b = blockchain_rpc.gas_cost_usd(data_b["pool"]["chain_id"], "swap")
            bridge = get_bridge_cost(data_a["pool"]["chain_id"], data_b["pool"]["chain_id"])

            net = gross - gas_a - gas_b - bridge

            if net > 0 and net > (gas_a + gas_b) * 2:
                opportunities.append({
                    "buy_pool":  addr_b if data_a["price"] > data_b["price"] else addr_a,
                    "sell_pool": addr_a if data_a["price"] > data_b["price"] else addr_b,
                    "spread":    spread,
                    "gross_usd": gross,
                    "gas_usd":   gas_a + gas_b + bridge,
                    "net_usd":   net,
                    "safe":      True,  # updated after sandwich check
                })

    return sorted(opportunities, key=lambda x: x["net_usd"], reverse=True)
```

---

## Cross-Chain Routing (NEW v1.1)

```python
BRIDGE_REGISTRY = {
    (1, 42161): {"name": "Arbitrum Bridge", "avg_time_min": 15, "fee_pct": 0.0006},
    (1, 8453):  {"name": "Base Bridge",     "avg_time_min": 7,  "fee_pct": 0.0005},
    (1, 10):    {"name": "Optimism Bridge", "avg_time_min": 7,  "fee_pct": 0.0005},
    (1, 42161): {"name": "Across Protocol", "avg_time_min": 2,  "fee_pct": 0.0010},  # faster
}

def evaluate_bridge(from_chain: int, to_chain: int, amount_usd: Decimal) -> dict:
    """
    Compare bridge options. Only recommend if net savings > $20.
    """
    key = (min(from_chain, to_chain), max(from_chain, to_chain))
    bridge = BRIDGE_REGISTRY.get(key)

    if not bridge:
        return {"viable": False, "reason": "No known bridge for this route"}

    bridge_fee = amount_usd * Decimal(str(bridge["fee_pct"]))
    l2_gas_saving = estimate_l2_gas_saving(amount_usd, from_chain, to_chain)
    net_saving = l2_gas_saving - bridge_fee

    return {
        "viable": net_saving > Decimal("20"),
        "bridge": bridge["name"],
        "fee_usd": bridge_fee,
        "saving_usd": net_saving,
        "time_min": bridge["avg_time_min"],
        "recommendation": "BRIDGE" if net_saving > Decimal("20") else "STAY ON CHAIN"
    }

# Rule: NEVER auto-bridge. Always present to Sensei with full breakdown.
```

---

## What to Return to UniClaw

```
SWAP REPORT v1.1
═════════════════
Action:         [Ratio rebalance | Arb execution | Bridge swap]
Chain:          [name] (chainId: [N])

ROUTING
  Router:       [1inch | Uniswap V4 | Jupiter]
  Route:        [tokenA] → [intermediate?] → [tokenB]
  Pool(s):      [address(es)]

TRADE
  Sell:         [N tokenX] ($[N])
  Buy (quoted): [N tokenY] ($[N])
  Price impact: [N]%
  Slippage max: [N]% ([stable|volatile|exotic])
  Min received: [N tokenY] ($[N])

RISK
  Sandwich:     [LOW | HIGH — details]
  MEV route:    [Flashbots Protect | Standard mempool]

COST
  Gas:          $[N]
  Bridge fee:   $[N] (if applicable)
  Net gain:     $[N]

Status:   AWAITING MASTER APPROVAL
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial | — | — |
| 1.1.0 | Added MEV protection (Flashbots, sandwich detection), cross-chain routing, 1inch + V4 routing, slippage hard limits, blockchain-rpc integration | Audit 2026-02-18 | 2026-02-18 |
