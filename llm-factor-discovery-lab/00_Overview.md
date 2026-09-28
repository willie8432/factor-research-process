# LLM-Factor-Discovery-Lab

## 1. Project Overview

使用 LLM 做加密貨幣因子研究:包含因子假設、特徵工程、到統計檢定。專案目標在建立並驗證一套 LLM-based 的因子研究機制，主張:

- LLM 能產出傳統 classifier 做不到的結構化特徵抽取。例如: 法規維度拆解、實體影響加權、新穎度加權等。
- 機制產出的因子具備可證偽性 (falsifiability) ，每個因子在探勘階段就寫上數值門檻的失敗條件


## 2. Pipeline

[factor-research-process-llm-factor-discovery-lab]

- **Data Acquisition & Alignment**: Binance OHLCV + CryptoNews corpus
- **Factor Discovery**: LLM (Sonnet 4.6 + Fable 5.1)
- **Feature Engineering**: LLM (Haiku 4.5) + MiniLM embeddings
- **Factor Computation**
- **Single-Factor Testing**



## 3. Data & Setup

| | |
|---|---|
| 標的 | BTC/USDT Spot, 單一標的 |
| 頻率 | 日線 |
| 期間 | 2026-03-12 → 2026-09-07, 180 天 |
| 價格 | OHLCV |
| 新聞 | CryptoNews API Premium,**27,272 篇**,經 LLM 抽取 16 欄特徵 |
| LLM | AWS Bedrock:Sonnet 4.6(探勘)、Haiku 4.5(抽取) |
| 成本 | 約 $75 |

```
💡 單一標的 × 180 天的統計後果

通常因子研究算的是 cross-sectional IC: 每天對 N 檔股票排序，得到 T 個 IC，
breadth ≈ N × T。

加密貨幣由 BTC 單一資產主導。但是單一資產下橫截面無定義，本專案只能算 time-series IC，每一個 horizon 只產出一個數字，breadth = T。
```


## 4. Candidate Factors

進行 Factor Computation、Single-Factor Testing 的七個因子，有 4 個依賴 LLM 抽取的新聞特徵 —— 這是專案的核心差異點。

| ID | Name | Dimension | Direction | Horizon | LLM |
|---|---|---|:---:|:---:|:---:|
| f001 | wick_asymmetry_accumulation | technical | + | days | |
| f002 | vw_close_position_signal | technical | + | days | |
| f004 | regulatory_jcd_score | news | + | days | V |
| f005 | novelty_weighted_sentiment | news | + | days | V |
| f007 | systemic_entity_negative_index | news | **−** | weeks | V |
| f009 | corr_regime_decoupling_spy | non-traditional | + | weeks | |
| f014 | unexplained_move_attribution_reversal | behavioral | + | days | V |

## 5. Result

IC 95% CI 區間全部包含 0，判斷 fail 不進入 Factor Combination。 


