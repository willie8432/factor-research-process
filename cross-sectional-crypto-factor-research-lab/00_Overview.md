# Cross-Sectional-Crypto-Factor-Research-Lab

## 1. Project Overview

針對 10 種大市值加密貨幣現貨的三因子策略研究，涵蓋因子假設、單因子測試、因子合成與樣本外績效對比。

## 2. Pipeline

[cross-sectional-crypto-factor-research-lab]

- **Data Acquisition & Alignment**: 4H bars 轉換為 point-in-time 面板資料。
- **Single-Factor Testing**: Rank IC, IC IR, Tercile Spread, Rolling 6-month IC, 多變量覆蓋回歸 (multivariate spanning regression)
- **Factor Combination**
- **Backtesting**

## 3. Setup

| | |
|---|---|
| Universe | BTC, ETH, SOL, BNB, XRP, ADA, DOGE, TRX, LINK, AVAX (Binance USDT spot) |
| Data | Binance 4h OHLCV only |
| Period | 2020-01 → 2026-08, 79 realized monthly returns |
| Rebalance | Monthly, 每個月第一個 4 小時 K 線的收盤時間 (04:00 UTC) |
| In-sample | 2020-01 → 2023-04 (n = 40) |
| Out-of-sample | 2023-06 → 2026-08 (39 months) |
| Costs | 26 bps round-trip = 2 x ( 3 bps fee + 10 bps slippage) |


```
💡 Cost Model

假設使用 BNB 支付 + referral (推薦人代碼) + VIP tier。成交 1 單位扣 13 bps。

```

## 4. Factors

| ID | Name | Horizon | Description |
|---|---|---|---|
| f001 | **MOM** | 1M forward | 上個月表現好的代幣，傾向下個月繼續保持領先 |
| f002 | **IVOL** (intraday vol regime) | 1M forward |  趨勢結構反映機構資金流持續性 |
| f003 | **AMIHUD** | 1M forward |  非流動性溢價 |




## (WIP) Results

- Every factor's CI includes zero.
- 僅使用兩個因子（MOM + AMIHUD）執行兩種策略: Rule-based, LightGBM
- OOS Backtesting (202306 -> 202608, 39-month)

| | BTC buy-and-hold | Rule-based | LightGBM |
|---|---|---|---|
| Sharpe | **+0.52** | +0.24 | +0.23 |
| Max drawdown | **−48.8%** | −64.1% | −52.3% |
| CAGR | **+27.6%** | +19.0% | +15.8% |
| Cumulative cost | — | 816.5 bps | 442.6 bps |

**Overfit test.** Rule-based Sharpe fell from +0.93 (IS) to +0.24 (OOS).
The difference, +0.69, has CI [−0.74, +2.23], so the decline is large but not statistically
distinguishable from noise at this sample size.

**BTC dominates on all six primary metrics.** Rule-based vs LightGBM Sharpe difference is 0.008, statistically indistinguishable — this confirms a pre-registered prediction about the small universe × top-5 selection structurally constraining model differentiation.

**Rule-based full sample (79 months):** Sharpe +0.6483, total net return +5,355% (54.55× multiple) — carries a strong universe-survivorship caveat.

**IS vs OOS (Rule-based, Metric 8):** IS Sharpe +0.9314, OOS +0.2422, difference +0.6892 with bootstrap CI [−0.7444, +2.2267]. CI includes zero → no statistically significant overfit signal under the strict test, though attribution between overfitting and regime shift (2020–2022 bull vs 2023–2026 mixed) cannot be resolved without a BTC IS/OOS regime control.