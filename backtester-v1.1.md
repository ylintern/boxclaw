---
name: backtester
version: 1.1.0
role: Historical simulation and strategy validation
requires: SKILL.md, skills/dune-analytics.md, skills/the-graph.md
upgrade-from: 1.0.0
---

# Backtester — Role Skill v1.1

> Validates strategies on historical data before any live capital is used.
> The gatekeeper. If it doesn't pass backtest, it doesn't get deployed.

**v1.1 changes:**
- Added explicit data source layer (Dune + The Graph)
- Added multi-source cross-validation requirement
- Added data quality gates before simulation starts
- Added context-budget awareness for long backtests

---

## Role Objective

Given a strategy spec, simulate it on historical price data and return
a clear verdict: **PASS** or **FAIL** with evidence.

---

## Backtest Workflow

```
1. RECEIVE strategy spec from Strategist or UniClaw master

2. LOAD DATA (NEW v1.1 — explicit source priority)
     Primary:   The Graph poolHourDatas (block-level, accurate fee growth)
     Secondary: Dune Q5 price series (cross-validation)
     Fallback:  CoinGecko/DeFiLlama hourly OHLCV (last resort — less granular)

     CROSS-VALIDATE: price series from both sources must agree within 0.5%
     If divergence > 0.5%: flag data quality issue, do not proceed silently

3. DATA QUALITY GATE (NEW v1.1)
     □ Date range completeness: < 2% missing hours
     □ Price sanity: no zero prices, no 50%+ single-candle moves (unless verified event)
     □ Volume present: at least 10% of hours have non-zero volume
     □ Fee data present: feeGrowthGlobal values exist for fee calculation

4. SIMULATE position over time:
     For each hourly candle:
       - Is position in range? → accrue fees (use actual feeGrowth from The Graph)
       - Did price cross a boundary? → trigger rebalance
       - Apply gas cost on each rebalance (use Dune gas history for period)
       - Track: fees, IL, rebalances, capital efficiency

5. COMPUTE metrics (same as v1.0)

6. COMPARE vs baseline (HODL or previous strategy)

7. RETURN verdict with full data
```

---

## Data Loading Module (v1.1)

```python
from dataclasses import dataclass
from decimal import Decimal
import pandas as pd

@dataclass
class BacktestData:
    price_series: pd.DataFrame    # columns: timestamp, open, high, low, close, volume_usd, fees_usd
    gas_series: pd.DataFrame      # columns: day, avg_gwei, median_gwei
    tick_series: pd.DataFrame     # columns: timestamp, tick (for range analysis)
    source_primary: str
    source_secondary: str
    quality_score: float          # 0.0–1.0

def load_backtest_data(
    pool_address: str,
    date_start: str,
    date_end: str,
    fee_tier: int
) -> BacktestData:
    """
    Load and cross-validate price data from The Graph + Dune.
    Raises DataQualityError if data does not meet quality gate.
    """

    # Primary: The Graph hourly OHLCV
    graph_data = the_graph.query(
        "PoolHourlyData",
        pool_id=pool_address.lower(),
        start_time=to_unix(date_start),
        end_time=to_unix(date_end)
    )
    df_graph = pd.DataFrame(graph_data)

    # Secondary: Dune price series (cross-validation)
    df_dune = dune.run_query("uniclaw_price_series", {
        "pool_address": pool_address,
        "date_start": date_start,
        "date_end": date_end
    })

    # Cross-validate: resample to same hourly frequency and compare
    merged = df_graph.merge(df_dune, on="hour", suffixes=("_graph", "_dune"))
    price_divergence = abs(
        merged["close_graph"] - merged["close_dune"]
    ) / merged["close_graph"]
    max_divergence = price_divergence.max()

    if max_divergence > 0.005:  # 0.5% tolerance
        log_warning(f"Price divergence {max_divergence:.2%} — using Graph as primary")
        if max_divergence > 0.02:
            raise DataQualityError(
                f"Critical price divergence {max_divergence:.2%} "
                f"between Graph and Dune — investigate before backtesting"
            )

    # Gas data from Dune (always — no equivalent in The Graph)
    df_gas = dune.run_query("uniclaw_gas_history", {"days": 365})

    return BacktestData(
        price_series=df_graph,
        gas_series=df_gas,
        tick_series=extract_ticks(df_graph),
        source_primary="the-graph",
        source_secondary="dune",
        quality_score=1.0 - float(price_divergence.mean())
    )
```

---

## Simulation Engine (unchanged from v1.0 — reproduced for completeness)

```python
class BacktestResult:
    strategy_name: str
    period: str                  # "2024-01-01 → 2024-12-31"
    initial_capital: float

    # Performance
    final_value: float
    total_fees: float
    total_il: float
    total_gas: float
    net_profit: float            # fees - IL - gas
    roi_pct: float

    # Risk
    sharpe_ratio: float
    max_drawdown: float
    sortino_ratio: float

    # Efficiency
    time_in_range_pct: float     # capital efficiency
    n_rebalances: int
    avg_fee_per_day: float
    win_rate: float              # % of periods profitable

    # Data quality (NEW v1.1)
    data_source: str             # "graph+dune" | "graph_only" | "dune_only"
    data_quality_score: float    # 0.0–1.0 (flag if < 0.85)

    # Verdict
    verdict: str                 # PASS | FAIL | INCONCLUSIVE
    verdict_reason: str
```

---

## Pass Thresholds (unchanged from v1.0)

```python
PASS_CRITERIA = {
    "net_profit":      lambda x: x > 0,
    "roi_vs_hodl":     lambda x: x > 0,
    "sharpe_ratio":    lambda x: x > 1.0,
    "max_drawdown":    lambda x: x < 0.20,
    "time_in_range":   lambda x: x > 0.60,
}
# ALL criteria must pass for PASS verdict
# NEW v1.1: if data_quality_score < 0.85 → INCONCLUSIVE, not PASS
```

---

## Context Budget Awareness (NEW v1.1)

Long backtests (> 90 days, > 3 pools) can exhaust context.

```
BEFORE STARTING A LONG BACKTEST:
  1. Check context fill via context-manager skill
  2. If context > 50% → spawn dedicated Backtester Agent
     Agent gets: SKILL.md + backtester.md + strategy spec only
     Agent does NOT need: IDENTITY.md, other role skills
  3. Agent returns BacktestResult object only — not full data
  4. UniClaw master stores result in STATE.md
```

---

## Backtest Report Format

```
BACKTEST REPORT
════════════════
Strategy:    [name]
Pool:        [pair + fee tier]
Period:      [start → end] ([N] days)
Capital:     $[initial]

DATA SOURCES
  Primary:      [the-graph | dune]
  Secondary:    [dune | the-graph | none]
  Quality:      [N]% (✅ good | ⚠️ marginal | ❌ poor)
  Divergence:   [N]% between sources

RETURNS
  Net Profit:    $[N] ([N]% ROI)
  vs HODL:       +[N]% (or -[N]%)
  Fees earned:   $[N]
  IL incurred:   -$[N]
  Gas spent:     -$[N]

RISK
  Sharpe Ratio:  [N]
  Max Drawdown:  [N]%
  Sortino Ratio: [N]

EFFICIENCY
  Time in range: [N]%
  Rebalances:    [N] (avg cost: $[N])
  Avg fees/day:  $[N]
  Win rate:      [N]%

VERDICT: [✅ PASS | ❌ FAIL | ⚠️ INCONCLUSIVE]
Reason:  [One sentence — what made or broke the result]
```

---

## Improvement Log

| Version | Change | Evidence | Date |
|---------|--------|----------|------|
| 1.0.0 | Initial role definition | — | — |
| 1.1.0 | Added data source layer (Dune + Graph), cross-validation, data quality gate, context budget awareness | Audit 2026-02-18 — backtester had no defined data source | 2026-02-18 |
