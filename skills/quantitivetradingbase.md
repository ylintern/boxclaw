---
name: quant-trading-knowledge
description: >
  Deep quantitative trading knowledge base covering factor models, strategy
  design, backtesting, ML/NLP signal generation, risk management, and portfolio
  optimization. Use this skill whenever the user asks about ANY of the following:
  multi-factor investing, momentum/value/quality/volatility factors, stock
  screening, alpha generation, building or evaluating trading strategies,
  backtesting methodology, Sharpe/Sortino/drawdown metrics, position sizing,
  VaR/CVaR risk limits, portfolio optimization (risk parity, mean-variance),
  sentiment analysis for trading, feature engineering for financial ML,
  real-time market data pipelines, technical indicators, NLP on news/earnings,
  LLM-powered market analysis, or anything related to quantitative finance.
  This is a research and reasoning knowledge base — not an API guide.
---

# Quantitative Trading Knowledge Base

This skill contains condensed, practitioner-level knowledge across five domains
of quantitative trading. Use it to reason about, design, critique, or explain
any part of a modern quant trading system.

---

## DOMAIN 1 — Data Management & Feature Engineering

### 1.1 Market Data Concepts

**OHLCV** (Open, High, Low, Close, Volume) is the atomic unit of price data.
Every higher-level feature ultimately derives from it.

**Tick data → bar data** pipeline:
raw ticks → aggregate into fixed-time bars (1m, 5m, 1d) or volume/dollar bars
(uniform information content) → clean → feature-engineer → store.

**Dollar bars vs. time bars**: Dollar bars (fixed notional traded per bar) have
better statistical properties for ML — more uniform variance, fewer intraday
seasonality artefacts. Time bars over-sample quiet periods and under-sample
volatile ones.

**Data quality issues to always handle:**
- Forward-fill gaps only during market hours; never across overnight/weekend
- Survivorship bias: always include delisted stocks in backtests
- Look-ahead bias: features must only use data available *at* the bar timestamp
- Corporate actions: split-adjust prices; dividend-adjust for total-return calcs
- Outlier detection: winsorize extreme returns at 1st/99th percentile before
  factor cross-sections; flag volume spikes

### 1.2 Technical Indicators (Feature Engineering)

**Trend / Momentum**
```
SMA(n)   = mean of last n closes
EMA(n)   = close_t * (2/(n+1)) + EMA_{t-1} * (1 - 2/(n+1))
MACD     = EMA(12) - EMA(26);  Signal = EMA(9) of MACD
RSI(14)  = 100 - 100/(1 + AvgGain/AvgLoss)  over 14 bars
ROC(n)   = (close_t - close_{t-n}) / close_{t-n}
```

**Volatility**
```
ATR(14)  = EMA of max(high-low, |high-prev_close|, |low-prev_close|)
BBands   = SMA ± k * rolling_std   (k=2 typical)
HV       = annualised std of log returns over rolling window
```

**Volume / Microstructure**
```
OBV      = running total: +volume if close>prev, -volume if close<prev
VWAP     = sum(price * volume) / sum(volume)  over session
```

**Statistical features used in ML pipelines:**
- Z-score of each feature cross-sectionally (rank normalise across universe)
- Rolling correlation of asset to benchmark (time-varying beta)
- Autocorrelation of returns (mean-reversion signal)
- Realised skewness and kurtosis over 20-day windows

### 1.3 Data Pipeline Architecture

```
Raw Feed ──► Validation ──► Cleaning ──► Feature Calc ──► Storage
              (gaps, bad    (outlier,     (indicators,     (tiered:
               ticks)        adj)          factors)         hot/warm/cold)
```

**Storage tiering pattern:**
- Hot (Redis / in-memory): last N bars for live signal calculation
- Warm (SQLite / PostgreSQL): months of OHLCV + pre-computed features
- Cold (Parquet / S3): years of raw and processed history

---

## DOMAIN 2 — Factor Models & Quantitative Analysis

### 2.1 Why Factors Work

Factors capture persistent, pervasive return premia with economic or
behavioural explanations. The empirical bar for a factor to be investable
(per Berkin & Swedroe) is: persistent across decades, pervasive across
markets, robust to definition variants, investable after costs.

Only five classic factors clear this bar: **Market Beta, Size, Value,
Momentum, Quality**.

### 2.2 The Five Core Factors

**1. Market (Beta)**
- Source: CAPM risk premium
- Measure: regression beta vs market index
- Expected premium: equity risk premium (~5-6% historically)

**2. Size (SMB — Small Minus Big)**
- Source: liquidity premium, lower analyst coverage
- Measure: market capitalisation (natural log for regression)
- Watch-out: much of size premium concentrated in micro-caps with high friction

**3. Value (HML — High Minus Low)**
- Source: distress risk, mean-reversion of fundamentals
- Common metrics: P/E, P/B (book-to-price), EV/EBITDA, EBIT/EV
- Composite value score is more robust than any single metric
- Value has negative correlation to momentum (useful in combo portfolios)

**4. Momentum (WML — Winners Minus Losers)**
- Source: investor under-reaction to new information; trend-following herding
- Measure: 12-1 month return (skip last month to avoid short-term reversal)
- Variants: cross-sectional (relative rank) vs time-series (absolute trend)
- Factor momentum: buy factors themselves when they show positive 12-1 momentum
- Known crashes: sharp momentum reversals during market rebounds (2009, 2020)

**5. Quality (QMJ — Quality Minus Junk)**
- Source: structural mispricing; investors overpay for "exciting" junk stocks
- Measure composite of:
  - Profitability: ROE, ROA, gross margin, FCF yield
  - Safety: leverage (D/E), earnings stability, low beta
  - Growth: 5-year earnings CAGR
- Quality premium (1964–2023): ~4.7% p.a., Sharpe 0.47
- Negative correlation to Market Beta (−0.59) → excellent diversifier
- IC (Information Coefficient) in cross-sectional regression typically highest
  among factor categories

**Low Volatility (honourable mention)**
- Anomalous: low-risk stocks outperform on risk-adjusted basis
- Measure: 12-month realised volatility, beta, idiosyncratic vol
- Explanation: leverage constraints force managers to reach for high-beta stocks

### 2.3 Multi-Factor Composite Scoring

**Cross-sectional Z-score method (standard practice):**
```
1. For each factor f and date t, compute raw score for each stock i
2. Winsorise at 1st/99th percentile to remove outliers
3. Z-score: z_i = (raw_i - mean) / std  across universe on date t
4. Composite = weighted sum of z-scores across factors
5. Rank all stocks; long top quintile, short bottom quintile
```

**Information Coefficient (IC):**
IC = rank correlation between factor score at t and forward return at t+1
Good factor IC: > 0.05 | Excellent: > 0.10 | Typical for quality: ~0.125

**Factor combination approaches:**
- Equal weight: simple, robust, avoids overfitting
- IC-weighted: give higher weight to factors with better recent ICs
- Risk-parity weighting: equalise factor return volatility contribution
- Regression-based: OLS on training period (prone to overfitting)

### 2.4 Portfolio Optimisation

**Mean-variance optimisation (Markowitz):**
```
max  w' μ - λ w' Σ w
s.t. sum(w) = 1,  w >= 0  (long-only)
```
Problem: highly sensitive to μ estimates, which are noisy. Regularise with
shrinkage (Ledoit-Wolf covariance) or use Black-Litterman to blend views.

**Risk Parity:**
Each asset (or factor) contributes equally to portfolio variance.
```
RC_i = w_i * (Σw)_i / (w' Σ w)  →  set RC_i = 1/N for all i
```
No return estimates needed — depends only on covariance. More robust OOS.

**Minimum variance portfolio:** minimise w'Σw. Historically competitive
Sharpe because it loads on low-vol stocks which carry quality premium.

---

## DOMAIN 3 — Strategy Framework

### 3.1 Strategy Taxonomy

| Category | Core Logic | Typical Holding Period |
|----------|-----------|----------------------|
| Momentum | Buy winners, sell losers | 1–12 months |
| Mean Reversion | Buy oversold, sell overbought | Hours – 2 weeks |
| Trend Following | Enter direction of moving average | Days – months |
| Statistical Arbitrage | Exploit cointegrated spread | Hours – days |
| Factor Long/Short | Long top quintile, short bottom | Monthly rebalance |
| Volatility Breakout | Trade ATR/Bollinger breakouts | Intraday – days |
| ML Signal | Learned features → direction | Flexible |
| Sentiment | News/social score → allocation | Hours – days |

### 3.2 Momentum Strategy — Design Details

**Signal construction:**
- Lookback window: 12 months, skip last 1 month (avoid reversal)
- Cross-sectional: rank all stocks by 12-1 return, long top decile
- Time-series: go long asset only if 12-1 return > 0 (absolute momentum)

**Rebalancing:** Monthly works well for cross-sectional; weekly for crypto.

**Decay pattern:** Momentum IC peaks at 1-month forward horizon and decays
to zero by ~12 months — design portfolio rebalance frequency accordingly.

**Known failure modes:** crash risk during sharp market rebounds; earnings
surprise reversals; crowding when too many funds run same signal.

**HQM (High Quality Momentum) scoring — practitioner composite:**
```python
hqm_score = mean([
    percentile_rank(1d_return),
    percentile_rank(5d_return)  * 1.5,   # weight recent
    percentile_rank(1m_return)  * 1.25,
    percentile_rank(3m_return)  * 0.25,  # downweight slow
])
```

### 3.3 Mean Reversion Strategy — Design Details

**Z-score entry/exit:**
```
z = (price - rolling_mean(n)) / rolling_std(n)
Enter long:   z < -2.0
Exit long:    z > -0.5
Enter short:  z > +2.0
Exit short:   z < +0.5
```
Window n typically 20 days for equities; 48–96 hours for crypto.

**Pairs trading (cointegration-based):**
1. Test pair for cointegration (Engle-Granger or Johansen test)
2. Estimate hedge ratio β via rolling OLS
3. Spread = asset_A - β * asset_B
4. Trade spread z-score using above entry/exit rules
5. Stop-loss: if spread deviates > 4σ, cointegration may have broken

### 3.4 Trend Following — Design Details

**Moving average crossover:**
```
Signal = +1  if  EMA(fast) > EMA(slow)
Signal = -1  if  EMA(fast) < EMA(slow)
```
Common periods: fast=50, slow=200 (golden/death cross for daily charts).

**Donchian channel breakout:**
Go long when price breaks above N-day high; short below N-day low. Exit
at opposite channel or ATR-based stop.

**ATR-based position sizing:**
```
position_size = (account * risk_pct) / (entry_price * ATR_multiple * ATR)
```
Typical values: risk_pct = 0.01 (1% of account), ATR_multiple = 2.

### 3.5 Strategy Evaluation Criteria

Before trusting any backtest result, verify:
1. Out-of-sample performance (walk-forward, not just in-sample)
2. Transaction costs included (at minimum 0.1% per side for equities)
3. No survivorship bias in universe
4. No look-ahead bias in signal construction
5. Reasonable number of trades (not just 3 great lucky trades)
6. Performance across different market regimes (bull, bear, sideways)

---

## DOMAIN 4 — Backtesting & Performance Metrics

### 4.1 Backtest Engine Design

**Event-driven vs vectorised:**
- Vectorised: fast, good for research; can't model order-book realism
- Event-driven: slower but models market impact, partial fills, latency

**Standard parameters:**
```
initial_capital = 100_000  (USD nominal)
commission      = 0.001    (0.1% per trade, each side)
slippage        = 0.0005   (0.05% market impact estimate)
rebalance_freq  = "monthly"
```

**Walk-forward testing (anti-overfitting):**
```
Train window: 3 years → Test window: 1 year → Roll forward 1 year
Repeat across full history. Report OOS metrics only.
```

### 4.2 Performance Metrics Reference

| Metric | Formula | Target / Benchmark |
|--------|---------|-------------------|
| **Total Return** | (final_equity / initial - 1) | > benchmark |
| **Annualised Return** | (1 + total_return)^(1/years) - 1 | > 8% |
| **Volatility** | std(daily_returns) * √252 | < 20% |
| **Sharpe Ratio** | (return - rf) / volatility | > 1.0 good, > 2.0 excellent |
| **Sortino Ratio** | (return - rf) / downside_vol | > 1.5 good |
| **Max Drawdown** | max(peak - trough) / peak | < 20% |
| **Calmar Ratio** | annualised_return / max_drawdown | > 1.0 |
| **Win Rate** | wins / total_trades | context-dependent |
| **Profit Factor** | gross_profit / gross_loss | > 1.5 |
| **Hit Rate** | % of months positive | > 60% |

**Interpretation nuances:**
- Sharpe < 0.5: not worth trading. 0.5–1.0: acceptable with low correlation
  to existing portfolio. >1.0: genuinely good. >2.0: may indicate overfitting.
- Max drawdown is the metric most correlated with investor abandonment.
  Even a theoretically good strategy becomes unmanageable above 30–40% DD.
- Sortino is better than Sharpe when return distribution has positive skew.

### 4.3 Common Backtest Pitfalls

- **Overfitting:** Too many parameters tuned on same data → walk-forward test
- **Data snooping bias:** Testing hundreds of strategies → expect 5% to pass
  at p<0.05 by chance. Use Bonferroni correction or out-of-sample holdout.
- **Transaction cost underestimation:** Especially for high-turnover strategies
- **Regime sensitivity:** Many strategies fail during 2008, 2020 — test separately
- **Beta disguised as alpha:** Long-only strategy in a bull market

---

## DOMAIN 5 — Risk Management

### 5.1 Value at Risk (VaR)

VaR answers: *"What is the maximum loss at X% confidence over N days?"*

**Historical VaR (preferred for fat-tailed assets):**
```python
var_95 = np.percentile(returns, 5)   # 5th percentile loss
# e.g., if var_95 = -0.0372, there is 95% confidence of not losing >3.72% in 1 day
```

**Monte Carlo VaR:**
```python
mu, sigma = returns.mean(), returns.std()
simulations = np.random.normal(mu, sigma, 10_000)
var_mc = np.percentile(simulations, 5)
```

**Parametric VaR (Gaussian assumption):**
```
VaR_95 = μ - 1.645σ
VaR_99 = μ - 2.326σ
```
Underestimates tail risk for fat-tailed assets (crypto, small-caps).

### 5.2 Conditional Value at Risk (CVaR / Expected Shortfall)

CVaR = average of losses *beyond* the VaR threshold. Captures tail severity.
```python
var = np.percentile(returns, 5)
cvar = -returns[returns <= var].mean()  # average of worst 5% outcomes
```
CVaR is always >= VaR. A strategy with low VaR but high CVaR has a thick tail.
CVaR is more suitable for portfolio optimisation than VaR (coherent risk measure).

### 5.3 Risk Limit Framework

**Standard hard limits (QuantMuse defaults):**
```
max_var_95          = 0.02   (2% 1-day at 95% confidence)
max_cvar_95         = 0.03   (3% expected shortfall)
max_drawdown        = 0.20   (20% portfolio hard stop)
max_leverage        = 2.0    (2× notional)
max_position_pct    = 0.10   (single position ≤ 10% NAV)
commission          = 0.001  (0.1% per trade)
```

**Three lines of defence:**
1. Pre-trade check: does this order violate any limit if executed?
2. Real-time portfolio monitoring: trigger alert/auto-reduce above threshold
3. End-of-day reconciliation: full P&L attribution and limit verification

### 5.4 Position Sizing Methods

**Fixed fractional (% of account per trade):**
```
size = account_value * risk_fraction / (entry - stop_loss)
```
Typical risk_fraction = 0.01–0.02 (1–2% of account per trade)

**Kelly Criterion (theoretically optimal, practically modified):**
```
full_kelly = (p * b - (1-p)) / b
# p = win probability, b = reward/risk ratio
# In practice: use half-kelly or quarter-kelly to reduce variance
```

**Volatility-targeting:**
```
target_vol  = 0.15  (15% annualised portfolio volatility)
asset_vol   = realised_vol_20d * sqrt(252)
size_scalar = target_vol / (asset_vol * correlation_to_portfolio)
```
This is the industry standard for risk-parity and trend-following funds.
Reduces size automatically when volatility spikes → defensive in crises.

### 5.5 Drawdown Management

**Maximum Adverse Excursion (MAE):** worst intra-trade loss. Use to set stops.

**Drawdown anatomy:**
- Underwater period: time from peak to recovery (equity curve below watermark)
- Recovery ratio: time to recover / time to reach drawdown bottom

**Portfolio-level stop:** pause trading / halve risk when portfolio drawdown
exceeds a predefined level (e.g., 10% intraday, 15% from monthly high).

---

## DOMAIN 6 — AI & ML for Trading

### 6.1 ML Signal Generation Pipeline

```
Raw Price Data ──► Feature Engineering ──► Label Generation
       │                                          │
       ▼                                          ▼
Technical Indicators                    Forward Return Quintile
Fundamental Ratios                      Direction (+1/-1/0)
Sentiment Scores                        Volatility Regime
Cross-sectional Z-scores                         │
       │                                          │
       └──────────────► Model Training ◄──────────┘
                              │
                         Prediction
                              │
                    Position Sizing → Risk Check → Order
```

**Label construction choices:**
- Binary: sign of 20-day forward return
- Quintile: rank 1–5 of forward return in cross-section (preferred)
- Regression: magnitude of return (noisier target)

### 6.2 Model Selection Guide

| Model | Best For | Watch-out |
|-------|---------|-----------|
| XGBoost | Tabular features, non-linear relations | Needs careful regularisation; can overfit regimes |
| Random Forest | Robust baseline; handles correlated features | Slower; lower ceiling than XGBoost |
| LSTM/Transformer | Sequence patterns in time series | Data-hungry; expensive; often beaten by simpler models |
| Linear (Ridge/Lasso) | Interpretability; robustness | Misses non-linearities |
| Ensemble | Combining diverse models | Complexity cost |

**General ML in finance principle:** simpler models generalise better OOS
because the signal-to-noise ratio is extremely low. Complexity ≠ performance.

**Walk-forward CV for time series:**
Never use random k-fold — it causes look-ahead bias. Use purged walk-forward
with an embargo (gap between train end and test start = 5× signal half-life).

### 6.3 Feature Engineering for ML

**Price-based features (normalised for stationarity):**
```
log_return_1d, log_return_5d, log_return_20d, log_return_60d
vol_20d (annualised realised vol)
momentum_12m_skip1  (12-1 month return)
rsi_14
macd_histogram
atr_14_normalised  (ATR / price)
```

**Cross-sectional features (rank normalise on each date):**
```
zscore(pe_ratio)       # value
zscore(pb_ratio)
zscore(roe)            # quality
zscore(gross_margin)
zscore(momentum_12m)   # momentum
zscore(market_cap)     # size
```

**Target leakage checklist:**
- All features computed from data strictly before bar close
- Fundamental data dated by actual *release date*, not period-end date
- Earnings surprises use *actual* announcement date

### 6.4 NLP & Sentiment Analysis

**Signal sources:**
- Earnings call transcripts (tone, uncertainty words, guidance language)
- News headlines (VADER for rule-based; FinBERT for transformer-based)
- SEC filings (10-K/10-Q language changes year-over-year)
- Social media (Reddit/StockTwits — noisy but fast)

**Standard sentiment pipeline:**
```
1. Fetch text with timestamp (must be precise — minutes matter)
2. Preprocess: clean, tokenise, remove boilerplate
3. Score with FinBERT (or VADER for speed): [-1, +1]
4. Aggregate: e.g., volume-weighted average score over 24h window
5. Normalise cross-sectionally across universe on each date
6. Combine with price factors as an additional feature
```

**Key findings from research:**
- Earnings call tone predicts 1-3 day post-earnings drift (IC ~0.06)
- Management language uncertainty (high modal verb usage) → negative signal
- Analyst report sentiment has IC ~0.04 for 30-day forward returns
- Social media sentiment: high IC for 1-day return, near-zero at 5+ days

### 6.5 LLM Integration Patterns for Trading Research

LLMs add value in four modes:

1. **Narrative synthesis:** Summarise earnings calls, macro news, 10-K language
   changes into a structured signal. LLM extracts: sentiment score, key risks,
   forward guidance tone, management confidence score.

2. **Hypothesis generation:** Given factor data and market context, generate
   candidate strategy ideas to test (not to trade blind).

3. **Anomaly explanation:** When a model fires an unusual signal, use LLM to
   cross-reference news for a fundamental reason — filter false positives.

4. **Report generation:** Automated portfolio commentary, factor attribution
   summaries, risk report narratives for human review.

**LLM output structure for market analysis:**
```json
{
  "sentiment": "positive|neutral|negative",
  "confidence": 0.0 to 1.0,
  "key_themes": ["rate_sensitivity", "guidance_raise", ...],
  "risk_flags": ["litigation", "margin_compression", ...],
  "recommendation": "BUY|HOLD|SELL",
  "reasoning": "..."
}
```

**Limitations of LLMs in trading:**
- Cannot reliably predict short-term price movements
- Hallucinate specific financial data — always validate numbers against source
- Best used as qualitative filter / confirmation, not primary alpha source

---

## DOMAIN 7 — End-to-End System Design

### 7.1 Full Pipeline Flow

```
[Data Ingestion]
  Multi-source → Validate → Clean → Normalise → Store (tiered)

[Signal Generation]
  Load features → Factor scoring → ML inference → Sentiment score
  → Combine signals → Rank universe

[Portfolio Construction]
  Screened universe → Optimise weights (risk parity / MVO)
  → Apply constraints (max position, sector limits, turnover budget)
  → Target portfolio

[Pre-Trade Risk Check]
  Compare target vs current → Generate orders → Validate:
  VaR, CVaR, drawdown, leverage, position concentration limits
  → Approved orders

[Execution]
  Route orders → TWAP/VWAP for larger orders → Fill tracking
  → Reconcile vs target

[Monitoring & Attribution]
  Real-time P&L → Factor attribution → Risk metrics dashboard
  → Alert on threshold breach → Daily end-of-day report
```

### 7.2 Key Design Principles

1. **Separation of research and production:** Research code can be exploratory;
   production code must be deterministic, logged, and auditable.

2. **Feature store pattern:** Pre-compute and version all features. Guarantees
   consistency between backtest and live trading (avoids "live drift").

3. **Kill switch / circuit breakers:** Automatic trading halt at portfolio-level
   drawdown limits; also at system-level (unusual order flow, data feed failure).

4. **Modular strategy registry:** Register strategies by name; swap in/out
   without changing execution infrastructure. Enables A/B testing live.

5. **Vectorised backtesting for research, event-driven for production:**
   Use vectorised for fast iteration on ideas; rebuild winning strategies as
   event-driven before going live to model execution realistically.

---

## QUICK REFERENCE — Common Formulas

```
Sharpe         = (R_p - R_f) / σ_p
Sortino        = (R_p - R_f) / σ_downside
Calmar         = R_p_annual / max_drawdown
Information Ratio = active_return / tracking_error
Beta           = Cov(R_p, R_m) / Var(R_m)
Alpha          = R_p - (R_f + β*(R_m - R_f))
Kelly fraction = (p*b - q) / b  where q=1-p, b=reward/risk
VaR_95 hist    = np.percentile(returns, 5)
CVaR_95        = mean of returns below VaR_95
Risk Parity    = solve for w such that w_i*(Σw)_i = constant ∀i
```

---

*Source: Synthesised from QuantMuse system architecture and documentation
(0xemmkty/QuantMuse, MIT License), academic factor literature (Fama-French,
AQR), and quantitative finance practitioner standards. For research and
educational purposes only.*
