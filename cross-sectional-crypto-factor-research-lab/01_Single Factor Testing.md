# LLM Factor Discovery

## Goal (WIP)



## Objective (WIP)


## Factor Discovery

**f001: cross_sectional_short_momentum (MOM)**
- 判斷訊號: 過去 21 天報酬排名前 50% 的 crypto，預期下個月會繼續有較好的表現。
- 金融原理: 三個 crypto 特有機制放大動能:
  - retail 主導市場，追逐近期漲勢，動能延續強
  - 上漲的 coin 吸引媒體以及敘事，動能易延續
  - 分析師、機構進場慢，市場對基本面資訊反應較慢，價格需要一段時間才能完全反應，從而創造出可被動量策略捕捉的趨勢。
- Direction / Horizon: Positive / Months
- Data requirement: 4h OHLCV close price
- 因子想抓: 尋找 relative winners 延續漲勢

**f002: intraday_volatility_regime (IVOL)**
- 判斷訊號: 比較 4hr 波動和 1d 波動，log(sigma_4h / sigma_1d)。微結構判斷趨勢延續。
- 金融原理: 
  - 低 IVOL 代表趨勢: 4hr 走勢比 daily 隱含的更順、走勢同向，intraday 波動小於依時間等比例縮放的日線波動，說明市場由單向 flow 主導且雜訊少，說明 institutional 執行交易、持續進場。趨勢延續
- Direction / Horizon: Positive / Months
- Data requirement: 4h OHLCV bars
- 因子想抓: 尋找內部結構最像趨勢的 coin，預期趨勢延續

```
💡 策略定調為低 IVOL 趨勢型

高 IVOL 代表震盪: 4hr 線出現頻繁上下的柱，但日線走勢相當有限，代表買賣雙方力道拉鋸缺乏明確方向。均值回歸。

雖然高 IVOL 也可解釋，但是避免看到資料後 post-hoc sign-flipping、事後更改方向，這裡定調為 positive。

```

**f003: amihud_illiquidity (AMIHUD)**
- 判斷訊號: 量化單位美元交易量造成的價格衝擊。高 AMIHUD = 高 illiquidity (流動性低) → 高 forward return。
- 金融原理: 
  - 當天價格變動幅度 / 當天成交金額 = 每一美元的交易量造成多少百分比的價格變動
  - 流動性差的資產 → 預期未來報酬較高 (因為投資者要求風險補償)

- Direction / Horizon: Positive / Months
- Data requirement: Daily close, volume from OHLCV bars
- 因子想抓: 哪個 coin 目前流動性最差，未來可能有 illiquidity premium

```
💡 Illiquidity premium (非流動性溢價、流動性風險溢價)

投資者需要額外報酬補償，才願意承擔持有流動性差的資產帶來的風險。換句話說 — 流動性差的資產報酬，需要高於流動性好的資產，才吸引資金進場。

但是在 Crypto 中:
流動性差的 crypto 除了 premium 之外，同時可能也代表危險訊號 — 這個 coin 快下架、失去交易所支援、開發團隊跑路。這種情況反而是 negative 訊號 (流動性差 → 未來歸零)。專案一樣不做 flip-signal 只做 positive 的。

```

| Factor | Dimension | Signal Source | Signal decay |
|---|---|---|---|
| MOM | Time-series 動能 | 過去報酬 | 快 (1-3M) |
| IVOL | Microstructure regime | 波動結構 | 快 (Weeks) |
| AMIHUD | Cross-sectional 流動性 | 交易活動 | 慢 (數月至數季) |

三個因子刻意選金融原理上不同維度 — 動能 (behavioral)、微結構 (regime)、流動性 (structural)，目的是降低 factor spanning。


```
💡 Fama-French 三因子模型

crypto 沒有資產負債表，且目前專案沒有流通量資料 (市值 = 價格 × 流通量)。因此挑選這三個因子做實驗目的是後續解釋 「這個因子的預測力 IC 是否能被其他因子的預測力 IC 解釋」

```

## Single Factor Testing


共同 78 期對齊表 (2020-01-01 → 2026-07-01,排除 2020-03-01)

[table 78 date metrics.png]

三個因子都沒辦法顯示出可用的預測力，信賴區間全部都包含 0、t 值全部都小於 1.6、IR 沒有超出「完全沒有預測力」時該有的程度。所以這 78 期的證據而言這三個因子都無法宣稱有效。且失效型態不同。

### MOM 失效 
IC_mean = -0.002、t = -0.05。IR = -0.0051 同樣幾乎不波動，因此不是方向錯了，是完全沒有訊號。

rolling 6-month IC 中位數 +0.0323、60.6% 的窗口為正。但是也都在 ±1 SE = ±0.136 的雜訊範圍內，擺盪全部來自 10 個幣的橫截面雜訊。

### IVOL 反方向
IC_mean = −0.0397、median −0.0424、rolling median = −0.0404，三個都是負的，且只有 38% 的窗口為正。不同統計量表達出一個較弱的反向因子。但是不做事後 flip-signal。

IR 略偏下方，且與 MOM 呈反向、高低點交錯。若提升量級、樣本，可再測試一次因子，分類出弱反向或是無訊號。

### AMIHUD 分歧
IC_mean = -0.0626，三者中最負，且 t 值 = -1.57 是三者中最大。只有 28.2 % 的窗口為正。

但是 tercile spread 最正 = + 0.0183，說明每個月 +1.83%。是三者中最大。說明: 大部分月份，流動性差的幣表現不如預期，但是少數月份的大漲，足以把平均值拉成正的。因子多數小負，少數大正。

IR 安靜且振幅最小，近幾年持續偏移負向。少數月份的表現雖然把 tercile spread 的平均拉成正值，但是在 rolling IC 上卻只是幾個回到零軸附近的短暫區段，不足以改變整體位置。訊號存在方向性，但正負出現在不同的月份。用單一平均值描述這個因子會失真。

### 小結
排除 IVOL 因為之前定義方向承諾，不隨著結果更改經濟故事，不在樣本內進行反轉。判定證偽，不進入下一階段。

MOM 是乾淨的無訊號零；AMIHUD 是方向分歧但未被推翻。兩者因此保留。



```
💡 78 期樣本對齊: 缺值與結果無關，是與資料中斷有關

Binance 在 2020-02-19 缺一根 4h bar (08:00 → 16:00)，造成當時已經上線的 8 個幣的資料不連續性，影響到 IVOL 以及 AMIHUD 視窗，該期時間戳比對失敗，沒有產出因子值、回傳 NaN。而 MOM 是讀頭尾兩個 21 天 close 端點，不檢查視窗內部，不受連續性影響，照常算得出因子值。

但是巧合的是，該 forward window 正好是 2020 年 3 月 COVID 崩盤，該期的 MOM 有最極端的橫截面，MOM 的 Full sample (79) IC: −0.0113 裡有九成來自這一期的崩盤資料，移除時 MOM (78) IC: -0.002，該調整同時也把唯一一次動能劇烈反轉的證據拿掉了。之後可以調整 window 以及因子值再觀測一次這段時間。
```

[ICIR fig1_rolling_ic_three_panel]


```
💡 IC IR 雜訊振幅

振幅幾乎完全由橫截面雜訊決定。

值得注意的是位置,不是振幅。 在這個樣本規模下,振幅幾乎完全由橫截面雜訊決定,三個因子的振幅差異不具解釋力。真正帶有資訊的是曲線待在零軸哪一側,以及待多久——AMIHUD 的 28.2% 與 MOM 的 60.6% 之間的差距,比任何一個尖峰都更值得討論。

```




