---
name: sentiment-analyst
version: 1.1.0
role: On-chain signals, market context and regime detection
requires: SKILL.md, skills/dune-analytics.md, skills/the-graph.md
upgrade-from: 1.0.0
---

# Sentiment Analyst — Role Skill v1.1

> Reads market context before strategic decisions.
> Provides regime detection and on-chain signals to inform range sizing and timing.

**v1.1 changes:**
- Added explicit on-chain data layer — Dune + The Graph queries now specified
- Regime report now REQUIRES data from at least one on-chain source
- Added signal confidence scoring
- Added smart money detection using Dune whale movements
- Unverified regime reports now blocked

---

## Role Objective

Answer: **"What is the market doing right now, and how does it affect our LP strategy?"**

Feeds into Strategist and UniClaw master to adjust:
- Range width (wider in high-vol regimes)
- Rebalance thresholds (faster in trending markets)
- Position sizing (reduce in extreme uncertainty)

**RULE (v1.1):** A regime report without on-chain data is an OPINION, not a signal.
Always query Dune and The Graph before issuing a REGIME report.

---

## Market Regime Detection (unchanged from v1.0)

```
REGIMES
───────
LOW_VOL:   30d realized vol < 25%
           → Narrow ranges, compound aggressively, low rebalance cost

NORMAL:    25% ≤ vol < 50%
           → Standard ranges, normal thresholds

HIGH_VOL:  50% ≤ vol < 80%
           → Wide ranges, higher rebalance threshold, reduce position size

EXTREME:   vol ≥ 80%
           → Alert Sensei. Consider exiting or going very wide.
           → IL risk dominates fee capture

TRENDING:  Price moved > 2σ in 24h
           → High boundary exit risk. Report immediately.
```

---

## On-Chain Data Collection Workflow (NEW v1.1)

```
STEP 1 — Fetch live pool state (The Graph)
  Query: PoolState for the target pool
  Extract:
    - current tick
    - 7-day poolDayData (volume, fees, txCount)
    - TVL
    - Spot token0Price

STEP 2 — Fetch historical vol data (Dune)
  Query: Q1 Pool Daily Volume (last 30 days)
  Extract:
    - 30d realized vol (compute from close price series)
    - 7d vs 30d volume ratio (spike detection)

STEP 3 — Fetch tick distribution (The Graph)
  Query: TickDistribution around current tick ± 500 ticks
  Extract:
    - Is liquidity thin around current price? (sandwich risk)
    - Are there big liquidity walls above/below? (price magnets)

STEP 4 — Fetch whale movements (Dune)
  Query: Q4 Whale LP Movements (last 7 days, min $50k)
  Extract:
    - Are whales adding or removing? (direction signal)
    - Are smart wallets setting wider or narrower ranges? (vol expectation proxy)

STEP 5 — Fetch gas trend (Dune)
  Query: Q6 Gas Cost History (last 14 days)
  Extract:
    - Current gas vs 14d average
    - Is high gas making rebalancing unprofitable?

STEP 6 — Compute regime (from collected data)
  Use realized_volatility() from SKILL.md
  Apply TRENDING detection (2σ rule)
  Score each signal → signal confidence (see below)

STEP 7 — Issue regime report
```

---

## Signal Collection and Confidence Scoring (NEW v1.1)

```python
@dataclass
class Signal:
    name: str
    value: float
    direction: str      # BULLISH | BEARISH | NEUTRAL
    strength: float     # 0.0–1.0
    source: str         # the-graph | dune | computed

def collect_signals(pool: str, lookback_days: int = 30) -> list[Signal]:
    signals = []

    # 1. Volume momentum
    vol_data = dune.run_query("uniclaw_pool_volume", {"pool_address": pool, "days": lookback_days})
    vol_7d = vol_data.tail(7)["volume_usd"].mean()
    vol_30d = vol_data["volume_usd"].mean()
    vol_ratio = vol_7d / vol_30d if vol_30d > 0 else 1.0
    signals.append(Signal(
        name="volume_momentum",
        value=vol_ratio,
        direction="BULLISH" if vol_ratio > 1.3 else "BEARISH" if vol_ratio < 0.7 else "NEUTRAL",
        strength=min(abs(vol_ratio - 1.0) / 0.5, 1.0),
        source="dune"
    ))

    # 2. Whale activity
    whales = dune.run_query("uniclaw_whale_lp_moves", {
        "pool_address": pool, "min_usd": 50000, "days": 7
    })
    net_whale = (
        whales[whales["event_type"] == "mint"]["amount_usd"].sum() -
        whales[whales["event_type"] == "burn"]["amount_usd"].sum()
    )
    signals.append(Signal(
        name="whale_flow",
        value=net_whale,
        direction="BULLISH" if net_whale > 100000 else "BEARISH" if net_whale < -100000 else "NEUTRAL",
        strength=min(abs(net_whale) / 500000, 1.0),
        source="dune"
    ))

    # 3. Price trend (from The Graph OHLCV)
    pool_state = the_graph.query("PoolState", pool_id=pool)
    closes = [float(d["close"]) for d in pool_state["pool"]["poolDayData"]]
    if len(closes) >= 7:
        price_change_7d = (closes[0] - closes[-1]) / closes[-1]
        sigma = compute_realized_vol(closes)
        z_score = price_change_7d / sigma if sigma > 0 else 0
        signals.append(Signal(
            name="price_trend",
            value=z_score,
            direction="BULLISH" if z_score > 1 else "BEARISH" if z_score < -1 else "NEUTRAL",
            strength=min(abs(z_score) / 2.0, 1.0),
            source="the-graph"
        ))

    return signals

def overall_confidence(signals: list[Signal]) -> float:
    """Weighted average confidence across all signals."""
    weights = {"the-graph": 1.0, "dune": 0.9, "computed": 0.8}
    total = sum(s.strength * weights.get(s.source, 0.7) for s in signals)
    return min(total / len(signals), 1.0) if signals else 0.0
```

---

## Signals to Monitor (updated v1.1 — with data sources)

```
ON-CHAIN (via The Graph — live)
  ✅ Pool volume 24h vs 7d average     → Query: PoolState.poolDayData
  ✅ Tick distribution                  → Query: TickDistribution
  ✅ TVL change 7d                      → Query: PoolState.totalValueLockedUSD
  ✅ Spot price and tick                → Query: PoolState.tick

HISTORICAL (via Dune — verified)
  ✅ 30d realized volatility            → Computed from Q5 price series
  ✅ 7d vs 30d volume ratio             → Computed from Q1 pool volume
  ✅ Whale LP add/remove (>$50k)        → Query: Q4 whale movements
  ✅ Gas price trend                    → Query: Q6 gas history

COMPUTED (from SKILL.md formulas)
  ✅ EWMA volatility (lambda=0.94)      → from realized_volatility()
  ✅ Price Z-score vs 30d mean          → from price series
  ✅ IL risk at current range           → from SKILL.md calculate_impermanent_loss()

EXTERNAL (manual / API — lower confidence)
  ⚠️  ETH technical levels              → Manual from Sensei or TradingView
  ⚠️  Funding rates on perps            → Exchange API (Binance / dYdX)
  ⚠️  Protocol upgrade events           → Sensei / governance forums
```

---

## Regime Report Format (updated v1.1)

```
MARKET REGIME REPORT v1.1
══════════════════════════
Timestamp:    [ISO]
Pool:         [pair + fee tier]

DATA SOURCES USED
  The Graph:  ✅ [pool state + tick distribution — N ms ago]
  Dune:       ✅ [volume + whale + gas data — Nmin ago]
  Quality:    [HIGH | MEDIUM | LOW — flag if any source unavailable]

REGIME:       [LOW_VOL | NORMAL | HIGH_VOL | EXTREME | TRENDING]
Confidence:   [N]% ([N] signals, [N] sources)

VOLATILITY
  30d realized:  [N]%
  7d EWMA:       [N]%
  Z-score:       [N]σ from 30d mean

SIGNALS
  Volume:        [N]× vs 30d avg  [↑ BULLISH | → NEUTRAL | ↓ BEARISH]
  Whale flow:    [NET ADDING $N | NET REMOVING $N | NEUTRAL]
  Tick depth:    [DEEP | NORMAL | THIN — sandwich risk]
  Gas:           $[N] avg ([N]× vs 14d avg — affects rebalance cost)
  Trend:         [NEUTRAL | BULLISH +N% | BEARISH -N%]

LP STRATEGY IMPACT
  Range width:     [NARROW | STANDARD | WIDE | VERY WIDE]
  Rebalance freq:  [PASSIVE | ACTIVE | VERY ACTIVE]
  Fee collection:  [HOLD | COLLECT NOW — gas is [cheap/expensive]]
  Position risk:   [LOW | MODERATE | HIGH | EXTREME]

RECOMMENDATION
  [One clear sentence about what this means for current positions]

ACTION NEEDED: [YES — [specific action] | NO — monitor]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial role definition | — | — |
| 1.1.0 | Added on-chain data layer (Dune + The Graph), signal confidence scoring, whale detection, blocked unverified regime reports | Audit 2026-02-18 — sentiment-analyst had no data source definitions | 2026-02-18 |
