# LLM Factor Discovery

## Goal

LLM 模型具備金融文獻、量化研究、市場結構分析等專業，是比單一人類研究員更適合的「因子提案人」。因此本專案使用 LLM 系統挖掘、發想候選因子，產出結構化的因子資訊，然後透過人工以明確地判斷標準做嚴格篩選，判斷可行性以及哪些值得進一步實作。

為了引導 LLM 模型產出高品質的因子提案，本章節也會重點說明階段式 prompt 設計以及引導 LLM 進行因子發掘的一些小技巧以及該避免的注意事項。

## Objective

1. [Two-step Prompt Design](#two-step-prompt-design)
   - **Step A. Factor dimensions 框架設計**: 構想出大方向的因子維度
   - **Step B. Factor candidate 細節具體化**: 產出各個因子結構化資訊 
2. [Add Behavioral Factor](#add-behavioral-factor)
   - 額外執行一次 **Step B**，實驗者補上想加入的行為經濟學 dimension 以及因子
3. [Candidate Factor List](#candidate-factor-list)
4. [Curation Criteria](#curation-criteria)
5. [❗ Sample Data Discussion](#sample-data-discussion): 由 factor IC 與 95% CI 討論所需資料量以及專案目的對焦  



## Two-step Prompt Design

使用兩階段 prompt，引導 Amazon Bedrock - Sonnet 4.6 產生 12 個 candidate factors。把「結構框架設計」跟「細節具體化」拆開為兩個任務。**Step A.** 先請模型做一次整體的、top-down、限制較少的框架思考設計，產出 dimension；**Step B.** 再 by dimension 去填各別因子內容，確保每個因子的發想深度。

**使用單一個大塊 prompt 直接產出 12 個因子的風險與壞處**
- 一次產出所有因子，前面的因子思考周密品質較高；而後半段因子品質顯著下降。
- 同上，後半段的幾個因子會明顯類似前面因子的變種。降低獨特性以及思考深度。


**使用兩階段 prompt 產出的好處**
- 兩階段 prompt 符合 extended thinking 作法，就如同 Claude 推理方式，解決問題前先列出思考步驟。
- 因子挖掘需要推理深度，而不是速度。強迫模型在挖掘前列出思考步驟以及原因。有機會做更深入的推理、辯論以及自我反駁。例如: 餵給另一個 session 做批判性思考與優化。
- 若單一 dimension 有問題，可以單純重新探勘那個 dimension。例如本專案後續加入行為經濟學類別。



### Step A. Factor dimensions 框架設計

prompt 填入市場情境，例如 regime description、notable event 等等。讓 model 知道他在對什麼市場做推理發想。

```
🔧 Technical Note - Prompt 移除定量數字，只保留客觀的定性描述

Step A. prompt 在放入 market 敘述的時候，記得移掉所有定量的數字。由這些本來就是統計得出的數字，所生成的因子，之後 in-sample IC 自然會很高。同時，直接餵給 LLM 會掉入 look-ahead 陷阱，會暗示模型往這個方向生成因子。

例如: 本 BTC 樣本期間，週末報酬平均為平日的 0.59 倍。會誤導模型往 weekly level factor 去探索。
```

產出四個 dimension，分別是:
1. Technical (技術面/價量結構因子): 純粹從 OHLCV 資料衍生的訊號。
   - Raw data source: OHLCV
   - 探勘因子數: 3
   - 優點: 實作可行性高
   - 缺點: 已經被市場測試過很多次了，紅海中的紅海
2. News (新聞因子): 從新聞、論壇文本中透過 LLM 抽取的訊號。不只是正負面情緒分數，包含更進階的事件分類（監管法規、交易所負面資訊、ETF 等等）。
   - Raw data source: crypto 新聞文章 
   - 預計探勘因子數: 4
   - 缺點: 大量的 LLM 計算需求

3. Non-traditional（非典型角度因子): 訊號來自 BTC 自身價量資料之外的地方。例如日曆效應、時區效應、Cross-Asset Lead-Lag、Google Trends 等等。
   - Data source: (依照因子需求不同)
   - 預計探勘因子數: 4

4. On-chain (鏈上資料因子): 從區塊鏈本身抽取的訊號。交易所淨流入/流出、活躍地址數、流動性變化等鏈上指標。
   - Data source: Blockchain data
   - 預計探勘因子數: 1
   - 缺點: 數據難取得且粒度不一難以對齊

```
💡 分散 dimension 的意義:

不同的 Dimension 有不同的挑戰。例如 Technical factors 會因為 Regime shift 而失效；News factors 會因為 Vendor bias 而失效；On-chain 則會因為 Data availability 問題而失效。

分散 dimension 帶來兩個獨立的好處:
1. 統計上: 相關性較低。使用來自不同 dimension 的 factors，比起來自同一 Dimension 的兩個 Factors，相關性較低。
2. 結構上:失效原因不相關。同一 dimension 的因子較有可能共享失效模式。
```


```
🔧 Technical Note - AWS Bedrock 使用注意

1. 跨 provider 的 Model ID 格式不一致。Anthropic 使用 claude-sonnet-4-6；而 Bedrock 使用 us.anthropic.claude-sonnet-4-6 [https://docs.aws.amazon.com/bedrock/latest/userguide/inference-profiles-view.html]

2. 你可能注意到，本專案使用 AWS Bedrock Region: us-west-2. Bedrock inference profiles use us. prefix。因為要執行的是 batch job thinking 沒有延遲問題。同時 us region 是 Amazon 模型較多、資源較充足、新模型上版較快的 region。
```

### Step B. Factor candidate 細節具體化

在同一個 dimension 尋找高品質的 factor，要求每個 factor 遵照 10 個 factor schema 思考，產出結構性的 JSON 結果，確保不會出現品質下降的情況。共有四個 dimension 因此 **Step B prompt** 會執行四次。

**10 個 factor schema** 
| # | Schema | Type | Note |
|---|---|---|---|
| 1 | **`name`** | string | Factor 唯一識別碼 | 
| 2 | **`dimension`** | enum: technical / news_sentiment / non_traditional / on_chain / behavioral | Factor 屬於哪個因子維度。決定後續階段所需 data pipeline。 |
| 3 | **`expected_direction`** | enum: positive / negative | 預測 Factor 值高時 → 未來 return 高(positive)或低(negative) | 
| 4 | **`time_horizon`** | enum: intraday / days / weeks | Factor signal 的預期作用時間尺度，決定後續 IC 使用哪個 forward return 來計算 |
| 5 | **`computation_formula`** | pseudocode string | Factor value 的計算公式 | 
| 6 | **`data_requirements`** | list of strings | 需要什麼資料、data source。後續評估有缺項的話 factor 不繼續實作 | 
| 7 | **`economic_rationale`** | prose (≥30 words) | 金融原理、直覺。為什麼這個 factor 應該會有 signal | 
| 8 | **`why_might_still_have_alpha`** | prose (≥30 words) | 為什麼還沒有被套利完? | 
| 9 | **`orthogonality_hypothesis`** | prose (other factors) | 這個 factor 與哪些**其他 factors 相關** | 
| 10 | **`falsifiability_test`** | named test + numeric threshold | 什麼實質證據、具體數字出現時會 kill 這個 factor? |

```
💡 falsifiability test

要求 model 思考 falsifiability test，失效要準確地可驗證，從因子探勘階段就開始思考因子的可證偽性。
```

## Add Behavioral Factor

本專案再補上一個 dimension 讓模型做發想。使用 Claude Chat Fable 5.1 補上行為經濟學領域的 3 個因子。因此 **Step B prompt** 再執行一次。

## Candidate Factor List

共產出 15 個 candidate factors:
- 前 4 個 dimension 共 12 個因子。Claude Sonnet 4.6 via Bedrock API。
- 加上 1 個 dimension 3 個因子。Claude Fable 5.1 via chat interface。

本階段採 Human in the loop 閱讀因子，review 每一個因子的判斷訊號、金融原理、因子公式、falsifiability test 等合理性。因子所需資料問題、高度相關性問題，會在下一階段以 criteria 收斂掉。

```
💡 

但若是這批 batch 產物有系統性上的弱化，例如品質下降、欄位空值等等，則需要回頭修 prompt 重跑。
```


### Technical Dimension

**f001: wick_asymmetry_accumulation (蠟燭影線不對稱累積)**

- 判斷訊號: 最近 10 天，下影線是否持續比上影線長?
- 金融原理: 蠟燭下影線代表當天價格跌到某個低點試探，但是被買方吸收回來了。因此連續多天下影線大於上影線代表: 有人在低位持續默默吸收供給，成為了主動買盤。
- Direction / Horizon: Positive / Days
- Data requirement: Daily OHLC, BTC/USDT
- 因子想抓: 多天連續下影線大於上影線，機構在默默累積的階段，預測未來大波段起漲

**f002 · vw_close_position_signal(成交量加權收盤位置訊號)**

- 判斷訊號: 最近 10 天大量成交那幾天，收盤 close 是靠近 high 還是 low
- 金融原理: 看每天 BTC 收盤價是靠近當天最高點還是最低點。再觀察成交量加權平均，比較 10 天 vs 40 天是不是更靠近高點，近期需求是否在增強。
- Direction / Horizon: Positive / Days
- Data requirement: Daily OHLC, Volume BTC/USDT
- 因子想抓: 高交易量日的 close 靠 high，若 10 天 VWCP > 40 天 VWCP，買方信念在增強，預測價格續漲

**f003 · vol_compression_breakout(波動壓縮突破)**

- 判斷訊號: 波動壓縮至少 5 天後，首次強力擴張的方向 = 未來趨勢方向
- 金融原理: BTC 有大量槓桿合約 (perpetual futures)。波動長時間壓縮 = 多空雙方僵持 + 槓桿部位持續累積。壓縮越久，兩側停損單堆積越多。一旦突破，先觸發那側停損 → 引發槓桿清算連鎖 → 放大單邊行情。
- Direction / Horizon: Positive / Weeks
- Data requirement: Daily OHLC BTC/USDT
- 因子想抓: 長時間累積槓桿突然釋放後的動量延續。

> 這在 crypto 特別強，因為 crypto 槓桿倍數遠高於股票。

### News Dimension

**f004 · regulatory_jcd_score(監管事件三維評分)**

- 判斷訊號: 監管新聞 = 地區權重 (Jurisdiction) × 具體程度 (Concreteness) × 方向 (Direction) ，判斷新聞是誰說的、有多具體、好消息還壞消息。
- 金融原理: BTC 對 regulatory action 高度敏感。且這些監管新聞不是同質類別。例如: 美國 SEC 最終裁定 vs 某小國謠傳限制對市場的影響差異巨大。
- Direction / Horizon: Positive / Days
- Data requirement: News corpus、LLM extraction、taxonomy tables 
- 因子想抓: 三個維度乘機:
  - 地區權重(J): US SEC = 3.0、EU MiCA = 2.0、中國 = 2.0、日本/新加坡 = 1.5、其他 = 0.5
  - 具體程度(C): 最終裁決 = 4、執法行動 = 3、提案規則 = 2、調查/聽證 = 1、謠言 = 0.5
  - 方向(D): 限制性 = -1、模糊 = 0、允許性 = +1
  - 總分 = J × C × D，當日的總分大於或是小於正常分數，預測 1-3 天 BTC 方向


> 一般 ML 情緒感知模型只能判斷 positive or negative。LLM 額外可以區分 地區 & 具體程度。

**f005 · novelty_weighted_sentiment(新穎度加權情緒)**

- 判斷訊號: 七天之內新穎度高的新聞，比重複報導的 (雜訊)新聞有影響
- 金融原理: 重複報導同一件事，散戶對重複資訊反應下降，只會產生短期波動。相反的，真正推動基本面的新資訊，應該有更持久的價格 impact。
- Direction / Horizon: Positive / Days
- Data requirement: News corpus、Pretrained sentence embedding model、LLM extraction 
- 因子想抓: 放大 novel 文章的加權。
  - 對每篇新聞產生 embedding (語意向量)
  - 對照過去 7 天所有新聞，算最大相似度
  - 新穎度 = 1 - 最大相似度
  - 每天加權情緒 = Σ(情緒 × 新穎度) / Σ(新穎度)


> 唯一需要 sentence embedding model 的因子，用 cosine similarity 來比較 novelty

**f006 · echo_chamber_reversal(回音室反轉)**

- 判斷訊號: 七天之內只有低新穎度的文章，大量重複的正面新聞，代表後續會反轉
- 金融原理: 新聞新穎度低 (全是重複報導)，代表沒有新資訊支撐的過度反應，應該均值回歸。
- Direction / Horizon: Negative / Days
- Data requirement: News corpus、Pretrained sentence embedding model、LLM extraction
- 因子想抓: 短期因低 novelty 的 echo 文章產生的過度反應，接下來會fade。

> 缺點: 均值回歸在 BTC 太弱，BTC 受 regime 影響太大
>
> 風險: 跟 f005 共享 embedding 基礎。若 embedding 品質不好，兩個因子將會同時失敗且無法互相診斷。也展現了後續特徵工程的重要性。

**f007 · systemic_entity_negative_index(系統性實體負面新聞指數)**

- 判斷訊號: 涉及重要交易所、系統性的負面新聞，未來 5-10 天負報酬
- 金融原理: BTC 生態高度集中交易所。這些實體出事 = 系統性風險。
- Direction / Horizon: Negative / Weeks
- Data requirement: News corpus、LLM entity extraction、LLM article sentiment score
- 因子想抓: 尋找系統性重要 entity 的 negative 新聞。entity 分層:
  - Tier 1(權重 3.0): Binance、Coinbase、Tether、BlackRock IBIT、CME、OKX、Bybit
  - Tier 2(權重 2.0): Kraken、Gemini、Fidelity、Grayscale、Bakkt、BitGo、Fireblocks
  - Tier 3(權重 1.0): 其他有名交易所

### Non-traditional Dimension

**f008 · cme_expiry_phase_signal(CME 到期日曆訊號)**

- 判斷訊號: CME 期貨每月最後周五到期，產生日曆決定性的機械性 flow。
- 金融原理: CME BTC futures 每月最後週五結算。機構 CME 多方部位要展期或平倉再對沖 spot。這種轉換降低現貨淨買力(前 5 天壓抑，因為機構主要精力在處理到期部位，暫停或減少新的現貨增倉)。到期後新合約開倉需要現貨對沖，現貨買壓(後 3 天上衝)。
- Direction / Horizon: Positive / Days
- Data requirement: Daily OHLCV BTC/USDT、CME BTC 期貨月度到期日期
- 因子想抓: CME 機構清算期貨日，對現貨的機械性影響。預測未來 BTC 走勢。純日曆，不需要公式。

> 問題: CME 每月最後一個週五清算，現有 dataset 來說只有 6 次

**f009 · corr_regime_decoupling_spy(BTC 與 SPY 相關性脫鉤)**

- 判斷訊號: BTC 跟 SPY 突然低相關，買方是 crypto-native，未來多週正報酬
- 金融原理: BTC 有兩種買家，可由 BTC 對 SPY 的動態 correlation state 去觀察現在市場上是由哪種買家主導:
  - Macro / 機構風險管理: 把 BTC 當 risk-on 代理 → 跟 SPY 高相關
  - Crypto-native (長期持有者、信仰者): 有自己的邏輯 → 跟 SPY 低相關
- Direction / Horizon: Positive (值高 = decoupling) / Weeks
- Data requirement: Daily OHLCV BTC/USDT、Daily SPY
- 因子想抓: 當 20 天 BTC-SPY 相關性顯著低於 60 天基準，correlation state 轉變為 decouple，邊際主導買方是 crypto-native，預測未來週級別正報酬。

> 需要抓 SPY 資料 (Yahoo Finance / Stooq)。

**f010 · weekend_return_reversal(週末報酬反轉)**

- 判斷訊號: 週末強烈報酬，週一到周三會反向修正。
- 金融原理: 周末參與者組成結構性改變，機構參與度低，散戶佔比高，單邊行情容易被放大。周一機構回歸，把周末的過激反應修正回來。aka 與周末反向。
- Direction / Horizon: Positive / Days
- Data Requirements: Daily BTC/USDT OHLCV bars、Day-of-week labels
- 因子想抓: -weekend_return (Monday only, 可 carry forward to Tue+Wed)


> 好消息: Model 自行從 crypto 市場特性與參與者結構性，推理出這個因子，根本原因與 Equity Weekend Effect 不同。BTC 版本的 alpha 來自 Monday institutional 修正 flow，不是週末假期效應在周一一次反應
> 
> 問題: 現有資料每個週一實際發生次數: 約 25 次

**f011 · btc_gold_narrative_transition(BTC 黃金敘事轉換)**

- 判斷訊號: BTC 跟 GLD 相關性上升、跟 SPY 相關性下降 →「數位黃金」買家佔上風 → 更穩定的多週上漲
- 金融原理: BTC 有兩個敘事:
  - 數位黃金: Store of value、抗通膨 → 買家是長期 macro 基金，低換手。儲值資產。
  - 高科技價值: 跟 Nasdaq/SPY 一起漲跌 → 買家是投機性、對股市波動敏感。成長型資產。
- Direction / Horizon: Positive / Weeks
- Data Requirements: Daily BTC/USDT OHLCV bars、Daily GLD、Daily SPY
- 因子想抓: 當最近 20 天 BTC-GOLD correlation 上升；且 BTC-SPY correlation 下降。從 「高科技價值敘事」 轉向 「數位黃金敘事」 的 marginal shift，預測 15 天 forward return。

> 好消息: Model 自行從 crypto 敘事角度出發發想因子
> 
> 問題: 跟 f009 因子想抓的公式相似

### On-chain Dimension

**f012 · addr_price_divergence_14d(活躍地址-價格背離)**

- 判斷訊號: 比較 BTC 活躍地址增長率 - 價格增長率 > 0，代表網路真實用戶正在快速湧入 BTC 網路上交易，而使用領先價格 → 未來多週上漲
- 金融原理: Fundamental 需求先出現，價格 catch-up 較慢。
- Direction / Horizon: Positive / Weeks
- Data Requirements: Daily BTC/USDT OHLCV、active addresses for Bitcoin
- 因子想抓: BTC 網路真實使用者度量與價格動量之間的落差。預測未來 1-3 週的漲勢。

> 風險: 鏈上資料隔一天才公布前一天的數據，因此只能用來預測下兩天的 return。

### Behavioral Dimension

**f013 · round_number_anchor_regime (圓整價格錨定機制)**

- 判斷訊號: 到整數價格 ($70k、$80k) 有兩種狀態，下方接近時是壓力位，站穩上方之後是支撐位。
- 金融原理: BTC 沒有其他的基本面定錨數字，所以整數美元價格形成唯一全球唯一通用的 "參考點"。媒體標題:「BTC hits $100k」永遠比「BTC hits $97,342」多報導。
- Direction / Horizon: Positive / Days
- Data Requirements: Daily BTC/USDT OHLC bars
- 因子想抓: 圓整數字的兩種心理錨定狀態。

> 這是 BTC 特有，因為其他標的會看很多其他指標 (例如: 股票有 P/E、EPS 等其他 anchor)，而散戶 App 裡面只會看到價格數字，強化價格的整數定錨參考點。

**f014 · unexplained_move_attribution_reversal(沒有歸因敘事的大波動反轉)**

- 判斷訊號: 大波動 + 有新聞覆蓋 = 趨勢繼續；大波動 + 無新聞覆蓋 = reversal。單獨看「大波動」或是「有新聞」都是一般事件。兩者交互作用才能捕捉到人們的歸因心理。
- 金融原理: 散戶行動方式取決於這個波動有沒有故事解釋
  - 有故事: 散戶看到會將波動歸因於該故事 -> 走勢延續。
  - 無故事: 歸因於鯨魚或是 liquidation 等強制行為 -> 隨機恐慌、走勢回調
- Direction / Horizon: Positive / Days
- Data Requirements: Daily BTC/USDT OHLC bars、news corpus
- 因子想抓: 大波動是否有敘事可歸因 (內在心理過程)


**f015 · breakeven_supply_overhang(Breakeven 供給壓力)**

- 判斷訊號: 現價上方若有高成交量 -> 未來到那裏會有拋壓，因為之前被套牢了；現價下方會有高成交量 -> 持有者會反射性買回檔，繼續賺。形成影響未來、無形的 wall、order book。
- 金融原理: 散戶的 reference point 是進場價，不是 fair value
  - 上方 overhang: 高價大量成交的價格帶，都是套牢持有者的存量。他們是「潛在賣家」，在價格回到成本時出貨。
  - 下方 underhang: 現價下方的高成交帶，是在賺的一群人。他們被「買進得賺」獎勵過，在價格回到成本時會反射性 buy the dip。

- 因子想抓: 預測 latent 買賣壓的失衡。 進場價(個人 reference)
- Direction / Horizon: Positive / Weeks
- Data Requirements: Daily BTC/USDT OHLCV bars
- 因子想抓: 當前價格上方 vs 下方的 volume 密集區差異，預測未來 10 天走勢

> 問題: 有 60-day rolling volume window warmup


### Candidate Factors Summary

**全部 15 個因子的  Direction X Horizon 分析**

|  | Days | Weeks |
|---|---|---|
| Positive | f001, f002, f004, f005, f008, f010, f013, f014 | f003, f009, f011, f012, f015 |
| Negative | f006 | f007 |

代表多數因子策略在預測短期上漲 (short-term long-biased)，若市場短期反轉，大量 positive / days因子會同時 fail。合成之後訊號互相 reinforce 可能過度集中。就像是去賭場全部都只會押「大」。

```
🔧 Technical Note - LLM 的語言偏好

87% 是 positive-direction，推測可能是因為 prompt 敘述有要求「 positive return」，LLM 模型自然朝向 「 signal direction 需要為positive」、「long-biased operation」框架思考。

這帶來三個具體影響:
1. 無法做 balanced long/short 策略
2. 多頭偏向導致同時失敗風險
3. 需要 negative 因子來平衡多樣性。例如: f007 systemic entity 是最寶貴的一個

未來若要平衡，需要在 prompt 中明確要求 model 考慮 short-side framings。因為對稱式的描述並沒有辦法讓模型自動保持中立，並不會自動探索雙向機制。
```

## Curation Criteria


本階段屬於 pre-backtest 篩選，透過明確的共 5 個的 criteria 檢查清單經過人工篩選，對每一個因子進行評分，將 factor 收斂下來，變成可以實作的因子集。

Criteria 共分成兩種:

- Hard filter:
  - **A. Data Availability (資料可及性)**: 資料可以取得嗎? 授權允許嗎?
  - **B. Computational Determinism (計算確定性)**: factors 公式明確嗎? 可以完全重現? 
  - **C. Falsifiability (可證偽性)**: 有明確的統計失敗條件?
- Soft filter:
  - **D. Orthogonality**: 跟其他因子的獨立性
  - **E: Signal Strength**: 金融原理是否是正確假設? novelty? 

> (Opt.) 以及一個隱藏因子，個人主觀評分項目 G，只有在分數一樣時做決定。



若一個 factor 只在少數幾天偵測，樣本數可能過少，Information Coefficient (IC) 結果無法分辨真訊號還是隨機 noise，會標註為 `KEEP-CAVEAT` (機制合理但無法在本專案 sample 上測)
- f003 (vol_compression_breakout) 只在波動壓縮 ≥5 天後突破時偵測，180 天預期 5-15 events → `KEEP-CAVEAT factor`
- f010 (weekend_return_reversal) 只在週一偵測，180 天 ≈ 25 Mondays → `KEEP-CAVEAT factor`
- f008(cme_expiry_phase_signal) 每月一次，180 天 6 events → `KEEP-CAVEAT factor`


```
💡 Sample size (n) must be larger than 30

根據 Central Limit Theorem(中央極限定理)，樣本數 n ≥ 30 時近似 normal distribution，而對於其餘統計工具 (t-test、OLS regression) 都假設樣本是 normal distribution。換句話說，樣本數 n < 30 檢定結果基本上不可信。
```

**Curation 結果表**
| Factor ID | Name | Dimension | Criteria Note | Result |
|---|---|---|---|---|
| **f001** | wick_asymmetry_accumulation | technical | fires 每天, 180 天樣本足以做 IC 分析 | ✅ `KEEP` |
| **f002** | vw_close_position_signal | technical | BTC-specific close mechanics, 10-40 天 | ✅ `KEEP` |
| **f003** | vol_compression_breakout | technical | Mechanism 最有趣 (BTC leveraged liquidation cascade) 但 fires 極稀 (180 天預期 5-15 events。sample-size feasibility 問題, 非 quality 問題 | ⚠️ `KEEP-CAVEAT` |
| **f004** | regulatory_jcd_score | news_sentiment | J × C × D 三維乘法比單一 News 資訊 richer, 但 regulatory event 在 180 天內是否夠多需要等下一階段實際測試  | ✅ `KEEP` |
| **f005** | novelty_weighted_sentiment | news_sentiment | Embedding-based novelty weighting，抓每天真新聞 sentiment，mechanism 獨立於價格，且每天 fires 足夠 | ✅ `KEEP` |
| **f006** | echo_chamber_reversal | news_sentiment | 與 f005 相同 embedding + sentiment pipeline，兩者在 embedding 品質退化時會**同時失效**; 為避免同源冗餘， curation 選 f005 保留 | ❌ `KILL` |
| **f007** | systemic_entity_negative_index | news_sentiment | 該 batch 最強概念，BTC infrastructure concentration 真實論述，entity-agnostic sentiment tools | ✅ `KEEP` |
| **f008** | cme_expiry_phase_signal | non_traditional | 完全依靠  calendar signal (無需 forecasting)，mechanism 清晰但 effect size 小 + 180 天只有 6 個 expiry events | ⚠️ `KEEP-CAVEAT` |
| **f009** | corr_regime_decoupling_spy | non_traditional | 該 batch 最經濟合理的 factor (mechanism specifies who is driving price),data requirement 極小 (僅 SPY)，testable on 180 天 | ✅ `KEEP` |
| **f010** | weekend_return_reversal | non_traditional | Simplest + operationally cleanest(零外部依賴)，mechanism 與 equity weekend effect 機制上不同; 但 180 天內只 ~25 個 Monday，sample size 不足以測 | ⚠️ `KEEP-CAVEAT` |
| **f011** | btc_gold_narrative_transition | non_traditional | 與 f009 partial duplication (共享 SPY correlation component) | ❌ `KILL` |
| **f012** | addr_price_divergence_14d | on_chain | paid-tier 鏈上資料 | ❌ `KILL` |
| **f013** | round_number_anchor_regime | behavioral | 該 batch 「最純 behavioral mechanism」(round-number 心理錨定)，但 180 天內只 fire 4-8 次 level approaches | ⚠️ `KEEP-CAVEAT` |
| **f014** | unexplained_move_attribution_reversal | behavioral | 該 batch 最強 factor，180 天內 conditioning events 夠多；新聞歸因 vs cascade 反轉的 mechanism 明確 | ✅ `KEEP` |
| **f015** | breakeven_supply_overhang | behavioral | 60 天 lookback 消耗 180 天樣本的 1/3; overlap with f002 | ❌ `KILL` |

**Result**: 15 factors scored, provisional 7 KEEP + 4 KEEP-CAVEAT + 4 KILL

### 收斂後 ✅ KEEP (7 個因子)



| ID | Dimension | Direction | Horizon |
|---|---|---|---|
| f001 wick_asymmetry_accumulation | technical | positive | days |
| f002 vw_close_position_signal | technical | positive | days |
| f004 regulatory_jcd_score | news_sentiment | positive | days |
| f005 novelty_weighted_sentiment | news_sentiment | positive | days |
| f007 systemic_entity_negative_index | news_sentiment | negative | weeks |
| f009 corr_regime_decoupling_spy | non_traditional | positive | weeks |
| f014 unexplained_move_attribution_reversal | behavioral | positive | days |
 
**Factor Distribution:**
- 6 / 7 是 positive direction
- 5 / 7 是 days horizon
- 4 / 7 是 LLM-based 因子 (f004, f005, f007, f014)


## Sample Data Discussion

我們繼續預先計算一下，下一階段算出來的 IC 的 95% 真實信賴區間 (confidence interval, CI) 是多少。

同上，在常態分佈中，「平均 ± 1.96 個標準差才涵蓋了中央 95% 的機率」，也就是重複做這實驗 100 次 有 95 的機率找到真實的 CI。

也因此，95%  CI ≈ IC ± 1.96 × SE。即使 factor 每天都觸發，共 180 個 event sample:
* SE (Standard Error, 標準誤) = 1/√(180-1) ≈ 0.075。

通常 factor IC  介於 0.03-0.08 算不錯，0.05 可以進到模型訓練用因子，而 > 0.15 通常是過擬合造成的。我們假設我們的 factor IC 算出來為 0.05，則實際的 95 % 信賴區間: 
* 95% CI ≈ 0.05 ± 1.96 × 0.075 ≈ [-0.1, +0.2]

真實 factor IC 可能是區間內的任何值，同時包含 0。代表真實 IC 可能是 0，180 天 sample 沒辦法證明 factor 有預測能力，也沒辦法證明 factor 沒有預測能力。少樣本下，95% CI 寬到 factor 的真實效應被 noise 掩蓋。


```
💡 

Sample size (n) 大 
  → SE 小 (SE = 1/(√n-1))
    → CI 窄 (CI = ± 1.96 × SE)
      → CI 不包含 0 的機率高
        → 更容易拒絕 null
          → 更容易確認 factor 有效

```

同樣我們再來計算一段，若一樣是 factor IC 算出來為 0.05。需要多少樣本才能可靠檢測呢? 也就是讓 95% CI ≈ IC ± 1.96 × SE 不包含 0。
- 1.96 × SE < 0.05
- SE = 1/√(n-1) < 0.0255
- √(n-1) > 39.2
- n > 1537 sample (天) ≈ 6 年

因此，若要可靠的檢測 IC = 0.05 需要至少 6 年的日數據。1537 sample 則是需要的樣本量。

而我們以 180 天 sample data 為主的專案是針對 LLM-based factor extraction 做的設計與取捨，做的 IC 分析必然不顯著。但這不代表 factor 無效，只代表以現有 sample 數量測不出來。





