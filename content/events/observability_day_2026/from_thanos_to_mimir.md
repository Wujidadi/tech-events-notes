# 從 Thanos 到 Mimir 3.0：長期指標基礎設施的光榮進化

> 許宏翔 (Mike Hsu)／DevOps, OpenNet

> 2026-04-23 15:35 - 16:15 @ 張榮發基金會國際會議中心 10F

> [議程](https://o11yday.ithome.com.tw/2026/session-page/4405)

> [簡報](../../../static/events/observability_day_2026/from_thanos_to_mimir.pdf)

> 本議程將直接分享我們如何將核心監控架構從 Thanos 遷移至 Grafana Mimir 3.0 的歷程心得。
>
> 面對長期指標存儲的效能瓶頸，我們放棄了熟悉的 Thanos ，轉而擁抱 Mimir 的「光榮進化」 過程。
>
> 內容將聚焦於兩大技術核心：
>
> 1. 深入比較 Thanos 與 Mimir 在架構設計上的本質差異與取捨，包含寫入路徑、對象存儲優化及查詢分片機制的對決。
> 2. 剖析 Mimir 3.0 的革新技術，探討其如何透過 Kafka 來進一步提升擴展性與穩定性的上限。
>
> 講者更將公開遷移過程中的實戰「眉角」，從資源架優化、歷史數據的無縫接軌，到生產環境中遇到的非預期坑洞，提供一條可參考的指標基礎設施升級路徑。

> **聽眾收穫：**
>
> 參與本場議程，聽眾將獲得針對大規模指標系統的具體實戰知識：
>
> 1. 架構選型評估：從維運成本、查詢效能與架構複雜度三個維度，掌握 Thanos 與 Mimir 的適用場景。
> 2. Mimir 3.0 核心原理詳解：深入理解 Mimir 如何利用先進的索引機制大幅降低儲存成本並提升速度。更加了解 Mimir 架構中關於核心元件的運作細節。
> 3. 分佈式寫入鏈路的心法：掌握 Prometheus，Mimir，Kafka 交互運作的鏈路精髓。引入 Kafka 作為 Remote Write Buffer 雖提升了穩定性，但也增加了複雜度。聽眾將學會如何在各節點中進行精準的故障定位，以確保穩定可控的概念。

---

## 講者介紹

Mike Hsu，曾撰寫多個與 Kubernetes、Grafana、可觀測性及 LLM 系統相關的系列文章，包括：

- 「從異世界歸來發現只剩自己不會 Kubernetes」
- 「你以為你在學 Grafana 其實你建立了 Kubernetes 可觀測性宇宙」
- 「後 Grafana 時代的自我修養」
- 「初探 LLM 可觀測性：打造可持續擴展的 AI 系統」

## AI Agent 時代下的 Metrics 後端挑戰

我們 Metrics 後端的主要使用者，已經不再只是人類。

過去的觀測模式通常是人類工程師偶爾查詢 Grafana、Prometheus 或其他監控系統；但在 AI Agent 時代，觀測者逐漸從人變成 Agent，查詢模式也從「偶爾查」變成「24/7 連續查」。

這代表基礎設施正在面對新的壓力：

- 查詢頻率更高
- 查詢範圍更廣
- 回溯歷史資料的需求增加
- 單一操作可能觸發大量 PromQL
- Metrics 後端必須支撐自動化、長時間、連續性的使用模式

## 目前的 Metrics 規模

目前系統規模大致如下：

- 40 個 EKS clusters
- 120M peak active series
- 8M samples / sec
- 365 天 retention period

Active Series 之所以重要，是因為常駐記憶體與 active series 數量高度相關，包含：

- TSDB head chunk
- In-memory postings index
- Label set 本身

估算方式：

```text
120M × 8 KB = 960 GB RAM
```

2023 年時，團隊選擇 Thanos 作為長期指標儲存方案。當時 Thanos 是最成熟的開源選項之一，而且它可以透過 Sidecar 模式無侵入地接在 Prometheus 旁邊。

---

## 01. 長期指標後端架構

長期指標後端架構主要面對的是兩種架構與兩種設計哲學。

### Prometheus 孤掌難鳴

Prometheus 本身有一些先天限制：

- 本地資料保留約 14 天就可能滿
- 單機儲存，單點壞掉資料就可能遺失
- 跨叢集無法自然統一查詢
- 記憶體使用量會隨 active series 線性成長，形成垂直擴展瓶頸

但實際需求卻不斷增加，例如：

- 查看上個月 baseline
- 比較熱門節日與平日
- 計算 SLO 年度達成率
- 讓 AI Agent 連續回溯歷史資料
- 單一操作視窗可能產生數百個 PromQL

因此需要的是一個：

- 高吞吐
- 低延遲
- 成本便宜
- 能擺脫單機天花板

的長期儲存後端。

### 兩種整合模式：Sidecar vs. Remote-Write

長期指標後端與 Prometheus 的整合，大致可分為兩種模式。

#### Sidecar Mode

Sidecar Mode 是 Thanos 的代表性模式。

架構概念：

```text
Prometheus
  ↓
Thanos Sidecar
  ↓ Upload
Object Store
```

Sidecar 會寄生在 Prometheus Pod 旁邊，直接將 Prometheus 產生的 TSDB block 上傳到 object store。

這種模式的特色是：

- 對 Prometheus 侵入性低
- 不需要改變 Prometheus 寫入流程
- 直接重用 Prometheus 本地產生的 TSDB block

#### Remote-Write Mode

Remote-Write Mode 是 Thanos、Mimir、Cortex 等系統主流支援的模式。

架構概念：

```text
Prometheus
  ↓ remote-write
Long-term Storage Backend
  ↓ Upload
Object Store
```

Prometheus 將 samples push 給後端，後端再負責壓縮、建立 index，並寫入 object store。

這種模式的特色是：

- 後端承擔更多處理工作
- 架構更適合集中化處理
- 有助於降低 Prometheus 端的長期儲存負擔

### Sidecar 的巧妙之處

Sidecar 的優點在於它「不重工，效率至上」。

Prometheus 本地 TSDB 每隔一段時間產生 TSDB block，Sidecar 只需要將這些 block 原封不動上傳到 S3。

關鍵洞察是：

- S3 上的 block 和本地 disk 上的 block 完全一樣
- Sidecar 不做額外運算
- 不需要重新解碼、壓縮、建立 index

相較之下，Remote-Write 模式下，後端需要：

- 解 protobuf
- 壓縮資料
- 重建 index

也就是說，某些運算等於做了兩次。

### 三年後面臨的痛點

雖然 Thanos Sidecar 架構在早期非常有效，但三年後逐漸出現兩個主要痛點：

1. 短期查詢瓶頸
2. 長期查詢瓶頸

### 痛點一：短期查詢的垂直擴展瓶頸

在短期查詢上，Sidecar 反而會放大 Prometheus 的垂直擴展瓶頸。

查詢路徑大致如下：

```text
Grafana / AI Agent
  ↓
Thanos Querier
  ↓
Thanos Sidecar
  ↓ remote-read
Prometheus
```

實際情況中，單一 Prometheus Pod 可能需要：

- 512 GiB RAM
- Prometheus 本身使用 400+ GiB
- Sidecar 使用 50+ GiB

Sidecar 需要替短期查詢承擔 remote-read buffer，使記憶體壓力被放大。

如果繼續走這條路，只剩兩種選擇：

- 買更大的機器
- 換架構

### 痛點二：長期查詢的細節差異

這裡不是說 Thanos 不好，而是實務上踩到了許多細節問題。

#### 1. 靜態 Sharding

Thanos 的 sharding 寫死在 manifest 中，調整一次就需要重新分配所有 block。

問題包括：

- 重新分配成本高
- 有可能產生 partial response
- 擴展與調整不夠動態

相較之下，Mimir 使用 Hash Ring 自動 rebalance，能降低過渡空窗問題。

#### 2. 社群開發節奏差異

Thanos 的開發量能相對有限，某些優化需求較難快速消化。

例如 Mimir 2.7 在 2023 年 1 月就預設支援 batched streaming StoreGateway，每個 gRPC message 可處理 5000 series。

Thanos 則直到 2026 年 1 月才 merge PR #8623 補齊相關能力；但團隊在 2025 年 9 月做決策時，這個功能尚未可用。

這也顯示 Grafana 官方積極管理社群，對專案演進速度有實際影響。

#### 3. 多租戶不友善

在 Thanos 中，一個 heavy long-range query 可能佔滿所有 worker，導致其他租戶全部排隊。

Mimir 則支援 per-tenant fair queuing，可以避免吵鬧鄰居影響其他租戶。

換句話說，在 Thanos 架構下，單點 heavy query 有機會被放大成全域問題。

### 三個決策維度

在開始遷移前，需要先釐清三個決策維度。

#### 1. Sidecar vs. Remote-Write

這是短期瓶頸的決策。

Remote-Write 可以把壓縮與建立 index 的工作丟給後端，緩解 Prometheus 的垂直瓶頸。

目標是：緩解短期垂直瓶頸。

#### 2. Thanos vs. Mimir

這是長期瓶頸的決策。

Mimir 具備：

- bucket-index
- 動態 sharding
- 為多租戶規模設計的架構

目標是：解決長期查詢瓶頸。

#### 3. Prom Server vs. Prom Agent

這是採集端的決策。

如果採用 Prom Agent，所有查詢都會依賴統一儲存後端。

這是最激進的方案，同時也是風險最高的方案。

### 五個排列組合與刪去法

可選組合包括：

1. Sidecar + Thanos + Prom Server  
   維持現況。

2. Remote-Write + Thanos + Prom Server  
   可處理短期瓶頸，但長期查詢瓶頸仍在。

3. Remote-Write + Thanos + Prom Agent  
   偏向長期設計，但風險較高。

4. Remote-Write + Mimir + Prom Server  
   同時緩解短期瓶頸與長期瓶頸，並保留 Prom Server 作為最後防線。

5. Remote-Write + Mimir + Prom Agent  
   HPA、KEDA、Alert 全部依賴 Mimir，是穩定後的終極優化方向。

最後透過刪去法，選擇第 4 種方案：

```text
Remote-Write + Mimir + Prom Server
```

也就是：

- 後端換成 Mimir
- 採用 Remote-Write
- Prom Server 暫時保留，作為最後防線

---

## 02. Mimir 3.0 架構的新地基

這一段說明為什麼偏偏現在適合換到 Mimir 3.0。

### Mimir 3.0 剛好到位

Mimir 3.0 的架構核心包含：

- query-frontend
- querier
- store-gateway
- distributor
- Kafka
- ingesters
- object storage
- compactor

它在可靠性、效能與成本上都帶來明顯改善。

#### Reliability：寫讀徹底解耦

Mimir 3.0 透過 Kafka 作為中繼層實現持久性。

特性包括：

- 讀取路徑異常時，寫入仍可照常進行
- quorum 從 2/3 降到 1
- 寫入與讀取路徑解耦

#### Performance：MQE Streaming

Mimir Query Engine 帶來查詢效能改善：

- Peak CPU 下降約 80%
- Peak Memory 下降約 3 倍

#### Cost：成本下降

Mimir 3.0 讓 Querier / Ingester 使用量減少。

由於不再單純依賴 RF=3 堆出可用性，整體 TCO 約可下降 25%。

### Ingest Storage：從寫三次到寫一次

Mimir 3.0 的 Ingest Storage 將寫入流程從傳統的多副本寫入，改為透過 Kafka 作為 linearized log。

傳統 Classic v2 模式中，資料需要透過 RF=3 寫入多份副本。

在 Ingest Storage v3 中：

```text
Distributor
  ↓ write once
Kafka
  ↓ consume
Ingester / Block Builder
  ↓
S3
```

為什麼 quorum = 1 就足夠？

因為：

- Kafka partition 是 linearized log
- 每個 consumer 都 replay 同一份 log
- 沒有分歧
- 不需要 overlap

也就是說，Kafka 承擔了寫入順序與持久性的保證。

### 讀節點異常，寫節點照常

透過 Kafka 持久性，Mimir 3.0 將寫入與讀取路徑解耦。

即使讀取路徑異常，寫入路徑仍可以維持健康。

#### 讀取可用性

v2 模式：

- 需要過半 zone 健康才算可用

v3 模式：

- 每個 partition 有一個 consumer 就算可用

#### 可用性解耦

即使熱查詢非常兇，寫入只要成功進到 Kafka 就結束。

因此：

```text
Write path healthy
Read path 可獨立處理異常
```

### 寫入持久性：從 Ingester Quorum 到 Kafka

Mimir 3.0 將可用性模型從「靠 RF 堆出來」，轉為「Kafka 保證 + partition 調整」。

| 維度          | Classic RF=3 + 3 Zones |       Ingest 3 Zones |       Ingest 2 Zones |
| ------------- | ---------------------: | -------------------: | -------------------: |
| 副本決定方式  |          RF=3，寫 3 次 | zone 數決定，寫 1 次 | zone 數決定，寫 1 次 |
| 實際副本數    |                     3x |                   3x |                   2x |
| Write 容錯    |     1 zone，2/3 quorum |           Kafka 負責 |           Kafka 負責 |
| Read quorum   |                    2/3 |                  1/3 |                  1/2 |
| Read 容錯     |                 1 zone |              2 zones |               1 zone |
| Ingester 成本 |                     3x |                   3x |                   2x |
| 額外成本      |                     無 |                Kafka |                Kafka |

核心轉變是：

```text
把可用性從「RF 堆出來」換成「Kafka 保證 + partition 調整」
```

### 無痛升級：Mimir Query Engine

Mimir Query Engine 在查詢效能與資源使用上帶來明顯改善。

測試情境：

```text
sum(disk_used_bytes)
1h query range with 15s step
1,000 series
```

結果顯示：

- 相較 Prometheus engine，記憶體使用減少 92%
- 執行速度提升 38%
- Querier peak memory 下降 3 倍
- Querier peak CPU 下降 80%

這代表 Mimir Query Engine 不只是架構升級，也直接改善了查詢成本與資源壓力。

### 遷移後：寫讀兩端資源同時下降

遷移後，Ingester 與 Querier 的 CPU、Memory 使用量皆明顯下降。

觀察項目包含：

- Ingester CPU
- Ingester Memory
- Querier CPU
- Querier Memory

這證明新架構不只是理論上更有效率，在實際遷移後也反映於生產環境的資源曲線上。

---

## 03. Kafka 選型

Mimir 3.0 架構中多了一個 Kafka 元件，因此需要回答一個問題：

> 多了一個元件，我們是不是在自找麻煩？

### 多加一個 Kafka 是否讓事情更複雜？

在回答這個問題前，可以先看 Little’s Law：

$$L = \lambda \cdot W$$

其中：

- $$L$$：Queue 中的平均任務數
- $$\lambda$$：系統吞吐率，例如 samples/s
- $$W$$：每件事花的處理時間

也就是：

```text
L 堆積 = 流入量（停不住） × 處理時間（下游拖慢）
```

這正是背壓的來源。

### 整條鏈路，牽一髮動全身

加入 Kafka 後，整條寫入鏈路大致如下：

```text
Prometheus
  ↓ remote-write
Distributor
  ↓ produce
Kafka
  ↓ consume
Ingester
```

這條鏈路中，只要某一段出現延遲或卡住，就可能形成連鎖反應。

真實情境可能是：

```text
Ingester 卡住
  ↓
Kafka 堆積
  ↓
Distributor produce timeout
  ↓
Prometheus queue full
  ↓
Alert 派對
```

這種跨元件的背壓傳遞，是 Kafka 架構的固有複雜度。

團隊學到的是：

- Kafka 不永遠低延遲
- Rebalance 可能造成延遲
- leader 切換可能造成延遲
- consumer lag 可能造成延遲
- 任一事件都可能讓 5ms 變成 5 秒
- 一個波動背壓，就足以造成全鏈路雪崩

換句話說，Kafka 讓系統更有彈性，但也要求團隊更理解每個元件的狀態與交互作用。

### 為什麼還是選了 Kafka？

團隊選 Kafka，不是只賭今天的 Kafka，而是賭明天的 Kafka。

如果只是要解耦，未必需要跳進 Kafka 這個複雜度裡。但過去兩年，Kafka 社群正在往新的方向演進，尤其是 Diskless Kafka。

### 下一個十年：Diskless Kafka

Kafka 社群與商業產品在 2023–2026 年間出現明顯趨勢：

- 2023 年 8 月：WarpStream 發表 Kafka API on S3，首發商用化
- 2024 年 5 月：Confluent Freight 發表
- 2024 年 7 月：AutoMQ 1.0 發布，支援 S3 Direct 寫入
- 2024 年 9 月：Confluent 收購 WarpStream，金額約 220M 美元
- 2025 年 4 月：Aiven 提出 KIP-1150 Diskless Topics
- 2025 年 11 月：Redpanda 發表 Cloud Topics
- 2026 年 3 月：Apache Kafka KIP-1150 正式通過，社群正式擁抱 diskless

這代表 Kafka 社群已經開始接受一個新方向：將 Kafka 從傳統強依賴本地磁碟與 broker 狀態的架構，逐步推向 shared storage 與 diskless topic。

### AutoMQ：最小改動的 Diskless Kafka 分支

AutoMQ 是一個以最小改動實現 Diskless Kafka 的分支，透過 S3Stream 介面將資料持久化到 shared storage。

#### 傳統 Kafka 的三大痛點

傳統 Kafka 的主要痛點包括：

1. Broker 有狀態  
   每次重啟或遷移都牽涉資料搬移。

2. Rebalance storm  
   加減 broker 可能造成大規模 partition 遷移。

3. 跨 AZ 流量成本高  
   跨 AZ replication、producer/consumer 與 leader 不同 AZ，都會產生大量網路費用。

#### AutoMQ + S3Stream 的方向

AutoMQ 透過 S3Stream 將 Kafka broker 推向 stateless：

- Broker stateless
- 適合 spot instance
- 秒級 rebalance
- S3 自帶多副本
- broker 端可做到 zero replication
- 維持 100% Kafka API 相容
- Producer / Consumer 不需要改動

### 跨 AZ 流量：傳統 Kafka 的黑洞

傳統 Kafka 在多 AZ 環境中，可能出現多種跨 AZ 流量：

1. Producer → Broker  
   寫入可能打到其他 AZ 的 leader。

2. Broker ↔ Broker Replication  
   replication 幾乎必定跨 AZ。

3. Consumer ← Broker  
   Consumer 不一定跟 leader 在同一個 AZ。

AutoMQ 官方指出，大叢集跨 AZ 流量可能占 Kafka 總成本的 60–70%。這些成本不是花在業務流量，而是花在跨區網路傳輸。

### AutoMQ 的解法：Zero-Zone Router

AutoMQ 透過 Zero-Zone Router 設計流量路由，降低或消除跨 AZ 成本。

設計重點包括：

- Producer 寫入本地 AZ 的 broker
- Rack-aware Router 透過 S3 路由給 leader partition
- 其他 AZ 從 S3 同步拿 readonly 副本
- Consumer 從本地 AZ 的 readonly replica 讀取

在 AWS 同一個 region 內，S3 流量可降低跨 AZ 傳輸成本，進而把原本 60–70% 的 Kafka 網路帳單大幅下降。

### 容量與彈性：從「預留」到「按用量」

傳統 Apache Kafka 採固定容量模式：

- Local disk 需要依 peak 預配
- scaling 通常以「小時」計
- 容易 over-provision
- 超過 50% 容量可能長期閒置

AutoMQ 則轉向 pay-as-you-go：

- S3 容量近乎無限
- partition 搬移以「秒」計
- Broker 可跑在 spot instance
- 更接近 true auto-scaling

這代表容量規劃從傳統「預留」轉向「按用量」。

### AutoMQ 的缺點：延遲

AutoMQ 與 S3Stream 帶來成本與彈性優勢，但代價是延遲。

比較如下：

| 架構                    | Produce ACK P99 |
| ----------------------- | --------------: |
| Traditional Kafka + EBS |          5–50ms |
| AutoMQ + S3Stream       |        500ms–1s |

兩者可能有 10 倍以上差距。在極端情況下，資料可能需要約 30 秒才能在 Mimir 中被讀取。

但這是可接受的取捨，因為：

- Alert、HPA、KEDA 繼續走 Prom Server，仍維持毫秒級
- Mimir 定位為秒級長期後端
- 業務即時性不受影響

---

## 04. 數字說話

此段以三週生產並行與 AWS 真實帳單驗證遷移效果。

### 實測：Mimir 勝

測試設計如下：

- 同一個 production 環境
- 相同 tenant
- 8 種 query
- 6 個時間範圍
- 共 48 組測試
- 進行 cache busting
- 時間範圍從 1h 到 30d 全覆蓋

結果：

```text
45 / 48 tests 勝出
```

具體改善包括：

- 平均查詢加速 3.4 倍
- Cross-metric join 30d 加速 16.7 倍
- High-cardinality 1h 加速 8.4 倍
- 長期 30d 查詢加速 6.3 倍

### AWS 帳單實測

三週 AWS 帳單實測結果：

```text
省 49%
```

整體效果可概括為：

- 3.4x 更快
- 多數場景勝出
- 約 6x 性價比

不同環境量級不同，絕對數字僅供參考；但在該環境下，annual saving 落在幾十萬美元量級。

---

## 05. 下一站：可觀測性 2.0？

最後一段聚焦 PromCon 2025 所觀察到的社群前沿趨勢。

### 單一事實來源：Wide Events

可觀測性 2.0 的一個方向，是將 logs、traces、metrics 全部倒進 Data Warehouse 或 Data Lake，並用統一查詢引擎做交叉分析。

這種方向可稱為：

```text
Single Source of Truth + Wide Events
```

#### 主要提倡者

幾個推動此趨勢的重要力量包括：

- Honeycomb  
  Wide events 先驅，將 Scuba 商用化。

- ClickHouse  
  自家 observability 規模撐到 100 PB+。

- Iceberg / Delta Lake  
  Open lake format 成為資料匯流點。

### 為什麼 Prometheus 還沒被取代？

雖然可觀測性 2.0 正在形成，但 Prometheus 還沒有被完全取代，原因包括：

- PromQL、Alerts、HPA、KEDA 生態深度綁定
- OTel semantic conventions 尚未完全成熟
- 新興技術仍在演進，多數團隊仍處於觀望階段

Prometheus 的價值不只在儲存 metrics，而在於整個雲原生生態系已經圍繞 PromQL 與 Prometheus API 建立大量依賴。

### Parquet Gateway：三大專案核心成員同台

PromCon 2025 中，Cortex、Thanos、Mimir 的 maintainer 共同推動 Parquet Gateway / parquet-common 方向。

相關成員包括：

- Jesús Vázquez，Senior Software Engineer @ Grafana
- Michael Hoffmann，Senior Software Engineer @ Cloudflare
- Alan Protasio，Senior Software Engineer @ AWS

共同 artifact：`prometheus-community / parquet-common`

能力包括：

- 通過 100% PromQL acceptance tests
- Built-in Queryable implementation
- TSDB block → Parquet schema converter

測試結果顯示：

- 83.6% faster queries
- 89.3% less bucket get-range
- 72.4% less memory
- 41.6% fewer allocations

### I/O 經濟學：資料結構特性才是關鍵

Parquet Gateway 背後的關鍵問題是：

> 為什麼 TSDB 不該直接住在 S3？

#### I/O Performance 差異

典型延遲差異：

- SSD random read：約 100µs
- S3 random read：約 10–100ms

差異可達 100–1000 倍。

#### TSDB on S3 的問題

TSDB 的資料結構原本是為本地 SSD random read 設計。

當 TSDB block 被搬到 S3 之後，讀取模式會變成大量小型 random GET：

- 可能需要 100+ random GETs
- 每個 GET 都是一次 HTTP round-trip
- 延遲與成本都會被放大

#### Parquet on S3 的優勢

Parquet 更適合 S3 這類 object storage：

- 3–4 次 sequential reads
- row-group index
- columnar skip
- 更符合 object storage 的 I/O 特性

因此問題不只是儲存在哪裡，而是資料結構是否適合該儲存介質。

### Store Gateway 的連鎖代價

因為 TSDB 結構不適合直接在 S3 上高效查詢，Store Gateway 被迫承擔許多代價：

- 需要昂貴的本地 disk
- 啟動時間長
- 被迫 stateful
- 運維彈性下降

這再次呼應前面的核心觀點：

```text
資料結構特性才是關鍵。
```

---

## 結語

整場分享的主線可以總結為：

```text
Thanos → Mimir 3.0 → AutoMQ → Parquet Gateway
```

### 1. 選型

理解瓶頸在哪裡，比追新技術更重要。

技術選型不應只看流行程度，而是要回到自身系統的真實瓶頸：

- 是短期查詢瓶頸？
- 是長期查詢瓶頸？
- 是寫入可用性問題？
- 是多租戶隔離問題？
- 是成本問題？
- 是資料結構與儲存介質不匹配？

### 2. 架構

Stateless 是運維自由的基礎。

從 Thanos Sidecar 到 Mimir 3.0，再到 AutoMQ 與 Diskless Kafka，整體方向都是讓系統逐步降低 stateful 元件的運維負擔。

架構演進方向包括：

- 寫讀解耦
- 以 Kafka 承擔持久性
- 以 object storage 承擔資料保存
- 減少 broker 與 gateway 的本地狀態
- 讓擴展、遷移、重啟、rebalance 更容易

### 3. 心態

技術選型永遠是 time-sensitive。

今天不成熟的技術，可能是明天的主流；今天看似穩定的架構，也可能在規模成長後變成瓶頸。

因此，面對基礎設施演進，需要保持：

```text
保持好奇 · 保持懷疑 · 保持實驗
```
