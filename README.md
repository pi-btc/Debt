# Two Supplementary Observations on LlamaRisk's Debt Ceiling Methodology

> **May 2026 · Feedback and corrections welcome**

LlamaRisk's debt ceiling methodology [[1]](https://llamarisk.com/research/debt-ceiling-methdology) is one of the most systematic pieces of risk management work in the Aave ecosystem. While studying the report, we attempted some supplementary analysis on top of their framework and found two things that might be worth discussing. We're sharing them here for community feedback — criticism of the methodology is very much welcome.

---

## What We Tried to Do

LlamaRisk's approach estimates a correlation matrix from the full historical return sample and uses it to generate synthetic shock scenarios. Our question was simple: does it make a difference if you estimate the correlation matrix separately for normal and stress periods, rather than pooling them together?

We used a Hidden Markov Model (HMM) to split roughly 1,500 trading days of returns (April 2022 to May 2026) into two regimes, then applied Marchenko–Pastur (MP) noise cleaning from Random Matrix Theory (RMT) to each regime's covariance matrix separately. We regenerated shock scenarios using stress-regime parameters and compared the results against the original method.

| | |
|---|---|
| Stress regime days identified | **234 days (15.6%)** |
| Normal regime days | **1,266 days (84.4%)** |
| Volatility ratio (stress / normal) | **2.25×** |

---

## Main Finding: Rolling Backtest Results

The chart we found most informative is the rolling backtest — comparing each method's VaR99 estimate against the realised equal-weight portfolio return across 1,247 trading days from January 2023 to May 2026.

![Rolling backtest](rmt_full_analysis.png)
<img width="804" height="209" alt="backtesting" src="https://github.com/user-attachments/assets/cef6e1b2-05fc-4820-8206-c81f6506cfe1" />


**Reading the chart (middle row):**
- 🔵 **Blue line** — VaR99 estimated by LlamaRisk's current method
- 🔴 **Red line** — VaR99 under RMT regime separation (generally more conservative)
- 🟢 **Green dots** — breach events missed by the current method but covered by RMT
- ✖ **Black crosses** — extreme black-swan events missed by both methods

The red line sits below the blue line for most of the sample period, providing additional coverage at several of the green-dot events. The most striking example is the FTX collapse in November 2022:

### FTX Collapse Event Backtest (Nov–Dec 2022)

Both models trained on pre-FTX data only, then tested on the event period:

| | |
|---|---|
| Realised maximum loss | **−20.1%** |
| Current method VaR99 | −12.9% *(underestimates by ~56%; does not cover the realised loss)* |
| RMT-improved VaR99 | −22.0% *(fully covers the −20.1% realised loss)* |

### Full Backtest Breach Statistics (1,247 days)

| Method | Breaches | Overall rate | Stress-day rate |
|---|---|---|---|
| Current method (mixed regime) | 15 | 1.20% ⚠️ | 2.19% ⚠️ |
| RMT regime separation | 7 | **0.56%** ✅ | **1.64%** |
| Theoretical bound | — | 1.00% | 1.00% |

It's worth being clear about the black crosses on the chart: the April 2024 Iran–Israel geopolitical shock, the August 2024 yen carry-trade unwind, and the October 2025 maximum drawdown of −18.79% were not covered by either method. These are genuine tail-of-tail events that VaR99 is not designed to capture — this is a fundamental limitation of the framework, not something a regime-separation improvement can fix.

> **Robustness check:** we repeated the analysis using different stress threshold definitions (top 5%–30% of cross-sectional volatility) rather than HMM output. The CVaR99 gap ranged from 8.7pp to 21.5pp across all definitions, always in the same direction. The finding does not appear to be sensitive to the specific regime identification method.

| Stress threshold | Stress days | CVaR current | CVaR improved | Gap |
|---|---|---|---|---|
| Top 5% | 75 | −20.8% | −42.2% | −21.5pp |
| Top 10% | 150 | −20.8% | −39.5% | −18.8pp |
| Top 15% (HMM baseline) | 225 | −20.8% | −36.1% | −15.3pp |
| Top 20% | 300 | −20.8% | −33.4% | −12.6pp |
| Top 25% | 375 | −20.8% | −31.4% | −10.7pp |
| Top 30% | 450 | −20.8% | −29.4% | −8.7pp |

---

## A Second Observation: PT Market Liquidity Competition

LlamaRisk uses a 60-day TVL moving average to set liquidity-based debt ceilings for Pendle PT collateral. We built a cross-market liquidity covariance matrix across six active PT stablecoin pools and found a correlation of **−0.45** between USDG and reUSD — their TVLs moved in opposite directions over the study period, consistent with capital rotating from one pool into the other.

![PT liquidity analysis](pt_liquidity_analysis.png)
<img width="2274" height="1589" alt="pt_liquidity_analysis" src="https://github.com/user-attachments/assets/35840c53-44db-434d-a6d5-e65a45f75dd5" />


| Market | TVL start | TVL current | Change | 60-day mean bias |
|---|---|---|---|---|
| USDG | $17.7M | $54.9M | +210% | +10.9% overestimate |
| apxUSD | $0.5M | $28.2M | +5100% | −70.5% underestimate |
| apyUSD | $6.7M | $12.9M | +92% | −27.4% underestimate |
| sNUSD | $44.0M | $24.0M | −45% | **+46.1% overestimate** |
| reUSD | $9.9M | $5.2M | −47% | **+61.2% overestimate** |
| reUSDe | $5.1M | $5.4M | +6% | +0.7% overestimate |

For markets in sustained TVL decline (sNUSD down 45%, reUSD down 47%), the 60-day mean overstates current available liquidity by 46–61%. That said, our data window is only 61 days, so the statistical power here is limited — please treat this as directional evidence rather than a firm finding.

---

## If These Findings Hold, What Would the Adjustment Look Like?

We derived a simple illustrative adjustment framework, normalised to LlamaRisk's static baseline of $100M:

| Adjustment step | Factor | Implied ceiling |
|---|---|---|
| LlamaRisk static baseline | — | $100M |
| Regime separation (stress/normal VaR99 ratio) | ×0.85 | $85M |
| PT liquidity competition (declining markets only) | ×0.85 | $72M |

This is intended as a directional illustration only. The specific haircut values would need much more careful calibration before being used in practice.

---

## Limitations We Want to Be Upfront About

- **Incomplete asset universe:** we use ETH, BTC, LINK, AAVE, MKR, UNI, and CRV as proxies. LST assets (wstETH, weETH) could not be included due to data availability issues, despite being significant Aave collateral with potentially different stress dynamics.

- **Short PT data window:** only 61 days of data were available across all six markets. The liquidity competition finding has limited statistical power and may look different with more data.

- **Free Probability approach abandoned:** we initially attempted to use the R-transform from Free Probability to decompose the observed covariance via additive convolution. We dropped this after finding that the free independence assumption required by the approach does not hold for empirically mixed-regime covariance matrices. The MP noise cleaning used here is the more appropriate RMT tool for this setting.

- **Tail-of-tail events remain uncovered:** several of the most extreme historical loss events are not captured by either method. This is a known limitation of the VaR99 framework and is not something regime separation can address.

---

## Data & Code

| | |
|---|---|
| Price data | CryptoCompare daily OHLCV, April 2022 – May 2026 |
| PT liquidity data | Pendle Finance v3 API, March – May 2026 |
| Assets analyzed | ETH, BTC, LINK, AAVE, MKR, UNI, CRV |
| PT markets | USDG, apxUSD, apyUSD, sNUSD, reUSD, reUSDe |

Replication code available in this repository. This is exploratory analysis — we would very much welcome feedback from the LlamaRisk team and the broader community.

---

## References

[1] LlamaRisk (2025). *Debt Ceiling Methodology*. https://llamarisk.com/research/debt-ceiling-methdology
