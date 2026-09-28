## Feature Extraction and Engineering

## Goal

將非結構化的 crypto news 抽取成結構化 factor-ready features。7 個 KEEP 因子當中總共有 4 個因子的公式會用到 News features。(**f004 regulatory**, **f005 novelty**, **f007 systemic entity**, **f014 attribution**)

## Objectives
1. [News Corpus Fetch](#news-corpus-fetch)
2. [Schema design](#schema-design)
3. [News Extraction](#news-extraction)
4. [Embedding-based novelty](#embedding-based-novelty)

```
💡 LLM-based Feature Extraction

新聞特徵擷取是 LLM 發揮作用的地方，用來執行以往 ML model 做不到的特徵擷取。
```

## News Corpus Fetch

News 數據蒐集與整理，有很多的真實世界限制，以下列出各個 API source 以及其限制，都是在執行過程中被發現、記錄下來的:

- [CoinDesk Data & Indices API](https://data.coindesk.com/blogs/changes-to-coindesk-data-indices-api-free-tier-access): 在 May 21, 2026 retired the free tier API, unlucky。後續不開放用個人版訂。
- [CoinGecko News endpoint](https://docs.coingecko.com/reference/news): 必須要到 Analyst Plan 才有 news 資料。
- [CryptoPanic](https://cryptopanic.com/developers/api/plans): Growth 版本，但是 API 不支援 data range 篩選
- [CryptoNews API](https://cryptonews-api.com/documentation): Premium 方案，使用 date parameters 取得 historical news

本專案使用 [CryptoNews API](https://cryptonews-api.com/) Premium plan。CryptoNews API 作為專業的 aggregation api 自 2019 年營運至今共 7 年，提供技術管道把 50 個權威 data source 資料整理成結構化的文章 (例如: CoinTelegraph, CoinDesk, NewsBTC, Decrypt 等 trusted sources)。產出格式支援 CSV export、date-range 篩選，Historical news data 最遠可回溯至 2020 年 12 月 (符合專案需求):
- Time range: 2026-03-12 to 2026-09-07 (Corpus 時間戳記經過整理，對齊 UTC trading days)
- Corpus: 27,272 articles (180 Days)

```
💡 Vendor Label

本專案使用的所有 feature (sentiment, topic, entity, regulatory etc.) 都是由 LLM extracted 出來的。

CryptoNews API 雖然有提供 sentiment / topic 等 labels，但是不作為任何 feature input 用途。這些標籤只作為 baseline ，後續來跟 LLM 結果作比對與參考。
```


## Schema design

4 個 factor 需要新聞特徵，總共 16 field 的抽取。定義好寫入 prompt，強制要求 LLM 提供結構化的抽取結果。

| Group | Fields | Serving |
|---|---|---|
| **BTC Relevance(2)** | `mentions_btc`, `primary_asset` | 判斷文章是否與 BTC 相關。 f005 gate |
| **Regulatory(4)** | `is_regulatory`, `jurisdiction`, `concreteness`, `direction_sign` | f004 formula (J × C × D) |
| **Sentiment & Content(5)** | `sentiment`, `topic_label`, `content_type`, `is_forward_looking`, `contains_causal_claim` | f005 novelty × sentiment, f007 sentiment, f014 attribution |
| **Entities(3)** | `named_entities`, `primary_entity`, `entity_sentiment_direction` | f007 tier weighting, f014 crypto-entity marker |
| **Meta / QC(2)** | `extraction_confidence`, `is_promotional` | f014 hard exclusions, QC diagnostics |

**f004 regulatory_jcd_score**
- Article gate: `is_regulatory == true`
- Formula: 
  - `jurisdiction` (J 分量)
  - `concreteness` (C 分量)
  - `direction_sign`(D 分量)

**f005 novelty_weighted_sentiment**
- Article gate: `primary_asset != "none"`
- Formula:
  - `sentiment` ([-2, +2])
  - Novelty embedding cosine similarity 計算，不是 schema field

**f007 systemic_entity_negative_index**
- Article gate: `primary_entity != "none"`
- Formula:
  - `sentiment` (只取 negative intensity)
  - `named_entities` (取 highest tier weight)

**f014 unexplained_move_attribution_reversal**
- Formula: (取所有可能造成歸因的新聞)
  - `is_regulatory == true`
  - `primary_asset != "none"` 且 ≠ "macro"
  - `topic_label` in {regulatory, institutional_move, hack_security, macro_economic}
  - `contains_causal_claim` == true
  - `named_entities`

```
🔧 Technical Note
primary_entity 一定要在 named_entities 裡面，避免模型幻覺，隨意放資訊進去。事後可再透過 quality check 驗證品質。

mentions_btc 跟公式化的 regex 以及 vendor label  做比較，一樣可再透過 quality check 驗證品質
```


## News Extraction

使用 AWS Bedrock Haiku 4.5，Prompt Engineering 只有精簡的三個直覺設計:
1. 要求模型準確獲取 16 個 fields
2. 觸發 Bedrock prompt caching: 有 cache-hit 的呼叫成本，比每一筆直接訪問低了 90%
3. 極小化 output token: 透過 JSON schema 範例，限制冗長的推理說明

```
🔧 Input token 反直覺現實

一開始，為了降低成本，基於「較短的 prompt = 較少的 input tokens = 更低的成本」假設，將初版 prompt 壓縮到大約 1,800 個 tokens。

但是經過研究後發現，Haiku 4.5's Bedrock prompt caching [1]，快取中斷點有 4,096-token 的最低門檻。低於這個門檻快取機制不會觸發，導致每次 API call 都變成全額的 input rate 費率。(uncached 金額估計 ~$249)

後續加入更多的 JSON 範例，將 prompt 擴展到 ~4,663 tokens，full corpus 掃完，99.99% cache hit, 共 $39.99。
```

[1] https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-anthropic-claude-haiku-4-5.html


```
🔧 Concurrent Calls and Multi-worker

原先採用 Single-worker 序列執行，雖然一開始大約前十篇文章有 burst rate 每篇文章約 1.5s，但是之後陸續出現 429 ThrottlingException，同時也開始 throttle wait time punishment。因此使用 Boto3 client-side retry mode [2] 的方式拉長訪問時間。

因此 single worker node 經過實驗最佳設定是 5.9 s/article。而 multi-worker concurrent calls 併發呼叫，只會爭奪同一個 throughput pool，造成大家都遭受懲罰、都降速。因此計算: Full corpus (27,272 articles): ~44 hours sequential.

若想增加 throughput，除了購買 Provisioned Throughput，也可以使用 cross-region inference 來多 worker 訪問。Bedrock rate limits 是 per region，經過實驗測試不同的 inference profile 獨立計算 pool [3]。例如: us-east-1/us 與 us-west-2/global 屬於各自獨立的 Pool。因此四個 worker、四個 inference profile 就有四個 Pools，就等於有四倍的 Throughput。

```
[2] https://aws.amazon.com/tw/blogs/machine-learning/optimize-your-applications-for-scale-and-reliability-on-amazon-bedrock/

[3] increase throughput and performance by enabling cross-region inference on AWS, https://repost.aws/articles/ARjL1-GiHRTJKQLP4V7-0tow/implementing-cross-region-inference-with-amazon-bedrock-while-maintaining-your-landing-zone-structure

### Pilot 驗證執行

進行全文本抽取之前 (27K calls)，使用 prompt and schema 分層抽樣 50 篇文章試跑並驗證 structural data 以及品質，同時由另外一個 LLM 模型進行評分。


### Full-corpus 執行結果

最後 Full corpus 使用 4 workers，資料採用交錯分片，每個 worker 按照時間索引選取一篇。若是發生 fail，文章採樣覆蓋率也會足夠，不會是一整塊連續的時間區段的資料 Missing。Feature Extraction 結果:

- Full corpus: 27,272 articles x 16 fields
- 4,847 total 429 responses (client backoff retry 均解決，沒有 error)
- Cache hit rate 99.99%
- Wall clock: 9.07 hours 
- Cost: $39.99

### Quality Check

在真正進入到 aggregation function 開發前，檢驗一下抽離出來的語料。同時再檢查一次樣本可行性。

| Check | Compliance | Interpretation |
|---|---|---|
| `mentions_btc` LLM vs regex | 98.69% | 高一致性 |
| Regulatory unit consistency | 99.36% | `is_regulatory` = null 時，其餘 Regulatory field 也應為 null |
| `primary_entity` ∈ `named_entities` | 99.62% | Anti-hallucination constraint holds |
| Vendor sentiment agreement (baseline) | 66.12% | - |


## Embedding-based novelty

為了符合 f005 使用到的文章新穎程度計算:
- Novelty: 1 − max cos_sim(article, prior_7_day_pool)
- Formula: Σ(sentiment_score × novelty) / Σ(novelty)

，使用 sentence-transformers/all-MiniLM-L6-v2，對全部 27,272 篇文章產生 384D embeddings，計算與前 7 天所有文章 embedding pool 的最大 cosine similarity。越接近舊文章 → cos_sim 越高 → novelty 越低。最終 factor 用 novelty 加權 sentiment:Σ(sentiment × novelty) / Σ(novelty)。

```
🔧 MiniLM-L6-v2 的原因:

1. 384 維已足夠 vs BERT-base 768 維。 
2. 全部 27,272 篇 embeddings 在 local CPU 上 ~40 分鐘跑完
3. 因子尋找的是「這兩篇是否講類似 event」，不是精細語意分析，不需要更複雜的模型
```
