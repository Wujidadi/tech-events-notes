# Grafana 全家桶的最後一塊拼圖—自建 Sentry 帶你攻略 Error tracking

> 林宗翰 (Jeff Lin)／Cloud Engineer, SHOPLINE

> 2026-04-23 16:30 - 17:10 @ 張榮發基金會國際會議中心 11F

> [議程](https://o11yday.ithome.com.tw/2026/session-page/4406)

> [簡報](../../../static/events/observability_day_2026/grafana_sentry_error_tracking.pdf)

> 在 Prometheus 與 Loki 建立的監控堡壘下，為什麼我們還需要 Sentry？
>
> 本議程將補上 Observability 的最後一塊拼圖，探討 Grafana 全家桶在「代碼層級除錯」上的不足，將分析 Metrics、Logs、Traces 在應用程式錯誤定位上的限制，以及實務上難以快速還原錯誤情境的痛點。
>
> 內容聚焦於 自建 Sentry 的實戰經驗：涵蓋架構選型、部署限制、資料量控管與整合挑戰，從建置到維運落地過程中的「踩坑心得」。
>
> 帶你了解我們如何面對自建過程的種種挑戰，並展望未來與 Grafana 生態系結合，打造無死角的監控視野。

> **聽眾收穫：**
>
> - 釐清監控盲點：理解為何在已有 Prometheus 與 Loki 的架構下仍需 Sentry，以及 Log (日誌) 與 Error Tracking (錯誤追蹤) 在代碼除錯上的本質差異。
> - 自建實戰避坑：獲取自建 Sentry 的第一手經驗，包含 Kafka 與 ClickHouse 的資源調教、數據量控管及架構維護的「血淚心得」，大幅降低團隊導入時的試錯成本。
> - 全方位整合視野：學習如何將 Sentry 與 Grafana 生態系結合，打破 Dev 與 Ops 的數據孤島，實現從「基礎設施監控」無縫銜接至「程式碼級除錯」的完整觀測體系。

---

## 講者介紹

Jeff Lin 是 SHOPLINE 的 Cloud Engineer，擁有 7 年工程經驗，包含：

- 2 年網路維運經驗
- 5 年軟體開發與維運經驗
- Cloud Infrastructure，包含 IaC
- CI/CD
- Monitoring

目前主要負責 SHOPLINE 的服務部署與監控工具維運。本次分享的 Sentry 自建，也是其負責的工作之一。

---

## 1. 我們的可觀測性現狀

SHOPLINE 原本已經具備可觀測性的三大支柱：

- Prometheus：Metrics 指標監控
- Loki：Log 日誌查詢
- Tempo：Traces 鏈路追蹤

從架構上看，這似乎已經是一座完備的監控堡壘。

但真正的問題是：

> 當半夜 On-call 響起時，我們真的能秒解問題嗎？

### 事故發生時的真實場景

實際事故發生時，常見流程如下：

1. 監控告警  
   Ops 看到 Metrics 飆高。

2. 查詢日誌  
   到 Loki 撈 Log，尋找線索。

3. 大量搜尋  
   使用 `grep ERROR` 面對海量日誌。

4. 回報 Dev  
   最後只丟一行錯誤訊息給開發者。

這時 Dev 往往會崩潰：

> 只有這一行 Error 訊息，我怎麼知道前因後果？呼叫參數是什麼？上下文在哪裡？

### Log 不等於 Error Tracking

Logs 與 Error Tracking 的設計目的不同，混用會讓問題更難解。

#### Logs：以 Loki 為例

Logs 的特性包括：

- 線性時間軸，依序輸出
- 非結構化或半結構化文字
- 數量龐大，充滿雜訊
- 缺乏聚合與分組能力

Logs 適合用來追蹤事件時間線，但不適合直接管理錯誤生命週期。

#### Error Tracking：以 Sentry 為例

Error Tracking 的特性包括：

- 自動聚合相同錯誤，支援 Grouping
- 保留完整上下文 Context
- 提供 Stack Trace 與 Local Variables
- 支援 Issue 生命週期管理

Error Tracking 的目標不是取代 Logs，而是讓錯誤調查從「搜尋大量文字」變成「定位具體 Issue」。

### Sentry 帶來的核心價值

Sentry 的核心價值，是讓團隊從「大海撈針」進入「精準定位」。

#### 自動聚合

Sentry 會將相同根因的錯誤自動分類，讓團隊一眼看清：

- 影響範圍
- 發生頻率
- 是否為重複錯誤
- 是否已經修復或再次發生

#### 完整堆疊追蹤

Sentry 提供完整 Stack Trace，可直接指向問題程式碼行，並附帶錯誤當下的 Local Variables。

這使開發者不需要只靠一行 Error Message 猜測問題，而是能直接進入程式碼脈絡。

#### 縮短 MTTR

有了完整錯誤上下文後，Dev 不再需要靠猜測 debug，而能直接進入修復流程。

這能顯著縮短 MTTR，也就是 Mean Time To Repair / Recovery。

---

## 2. Sentry UI 實際操作

簡報接著展示 Sentry UI 的實際操作畫面。

### Project Overview

Project Overview 頁面提供專案層級的錯誤概況，讓團隊可以快速掌握：

- 近期錯誤數量
- 錯誤趨勢
- 受影響使用者
- 主要 Issue
- 錯誤是否持續發生

### Issue Detail

Issue Detail 是 Sentry Error Tracking 的核心頁面。

這裡可以看到單一錯誤 Issue 的詳細資訊，例如：

- 錯誤訊息
- Stack Trace
- 發生時間
- 發生環境
- 受影響版本
- 使用者資訊
- Request context
- Local variables
- Breadcrumbs
- Event tags

Issue Detail 的價值，在於把原本散落在 Logs、Metrics、Trace 中的線索集中到同一個錯誤脈絡中。

---

## 3. 為何選擇 Self-Hosted Sentry？

Sentry SaaS 版本很好用，但 SHOPLINE 最後選擇 Self-Hosted，主要有三個考量。

### 資料隱私合規

Error Tracking 會收集錯誤上下文，而錯誤上下文中可能包含：

- 使用者資料
- Request payload
- Cookies
- IP
- Header
- Local variables
- 業務敏感資訊

因此，團隊希望這些資料能自主掌控，降低資料外流與合規風險。

### 預算控制

SaaS 的事件量費用難以預測。

當事件量變大時，SaaS 成本可能快速攀升。Self-Hosted 雖然需要自行維運，但在大量事件下成本較可控。

### 初期流量評估

自建環境可以依照實際流量動態調整規格，避免在尚未確定使用量前，就承擔高額授權費用。

---

## 4. Sentry Self-Hosted 組件解析

Self-Hosted Sentry 的組件數量相當多，這也是部署比想像中複雜的原因。

### 入口層

入口層包含：

- Relay
- Web

Relay 負責接收 SDK 送出的事件，Web 則提供 Sentry 的 UI 與 API。

### 訊息佇列

訊息佇列包含：

- Kafka
- RabbitMQ
- Redis

這些元件負責事件緩衝、非同步處理與工作排程。

### 工作處理層

工作處理層包含：

- Celery Workers
- Ingest Consumer
- Snuba Consumer
- Subscription Consumer
- Symbolicator
- Post-Process Forwarder

這些服務負責事件解析、符號化、索引、訂閱、後處理與寫入儲存層。

### 儲存層

儲存層包含：

- PostgreSQL
- Redis
- ClickHouse

PostgreSQL 負責 Sentry 主要 metadata，Redis 負責快取與 queue，ClickHouse 則承擔大量事件查詢與分析。

### 資料流複雜度

組件多只是開始，真正複雜的是資料流。

從 SDK 送出事件到 UI 顯示 Issue，中間會經過：

- 6 個以上服務
- 2 套訊息佇列
- 3 個儲存系統

任何一個環節出問題，都可能導致事件丟失或延遲。

---

## 5. 踩雷 1：部署踩坑

一開始的目標看似很單純：

> 用 Helm Chart 把 Sentry 部署起來，讓它能接收事件、顯示 Issue。

表面上看起來只是：

```bash
helm install
```

但實際上，這只是噩夢的開始。

### 沒想到：Helm install 後的驚喜

#### Chart values 繁雜

官方 Helm Chart 有數百個可設定項。

預設值多半是針對 demo 環境調校，若直接套用到 production，容易出現資源不足或服務不穩定。

#### Pod 啟動順序問題

Sentry 各服務存在特定啟動相依性。

如果 readiness probe 設定不當，可能導致服務反覆重啟，進而影響整體部署流程。

#### 初始化腳本問題

各組件有各自的初始化腳本，而且初始化流程之間也有相依性。

某些初始化 Pod 在腳本跑完後就會消失，如果沒有理解其生命週期，容易誤判部署狀態。

### 還好：讓部署不再是賭博

#### 從 self-hosted repo 反推 Helm 設定

官方 self-hosted repo 是最真實的參考來源。

相較於只看 Helm chart repo，團隊選擇將 self-hosted repo 中的：

- 環境變數
- 資源限制
- 服務設定
- 啟動順序

對照翻譯成 Helm values。

這比直接依賴 chart 預設值更可靠。

#### 先在 staging 驗證，再上 production

團隊先用最小可用規格在 staging 跑通完整流程，確認事件能從 SDK 流到 UI。

驗證完成後，再複製設定到 production，並調高資源。

### 避坑建議

不要以為 `helm install` 完就結束。

部署 Sentry 前應預留足夠緩衝時間，尤其不要在 release 前一天才開始部署。

---

## 6. 踩雷 2：基礎設施選型

Self-Hosted Sentry 牽涉許多基礎設施元件，因此需要決定哪些自建、哪些使用 Managed Service。

### 初始策略

團隊一開始的策略是：

> 能自建就自建，不得不 Managed 的才 Managed。

這聽起來很合理，因為可以同時兼顧：

- 降低維運複雜度
- 確保高可用
- 控制成本

但很快就發現，有些組件的 HA 比想像中複雜得多。

### 沒想到：HA 的複雜度遠超預期

#### 自建 Redis HA 比想像中複雜

Redis Sentinel / Cluster 模式在 EKS 上的部署與維運相當繁瑣。

更麻煩的是，Sentry Client 端的連線設定不一定能完全配合 Redis HA 的連線方式調整。

#### Chart 內建 ClickHouse 沒有 HA

Sentry 官方 Helm Chart 內建的 ClickHouse 不支援 HA 配置，存在單點故障風險。

這對 production Error Tracking 系統來說，是很大的可靠性問題。

### 還好：HA 優先，自建謹慎評估

最後各組件選型如下。

#### PostgreSQL：Managed

PostgreSQL 是公司既有技術選型，已有 DBA 協助維運，因此直接沿用 Managed PostgreSQL。

#### Redis：遷移至 Managed

Redis 自建 HA 成本高，且連線模式有相容性問題，因此遷移至 Managed Redis，以避免單點故障。

#### ClickHouse：維持自建

雖然 Chart 內建 ClickHouse 不支援 HA，但團隊改用 Altinity ClickHouse Operator 來支援 HA。

因此 ClickHouse 維持自建。

#### Kafka：維持自建

團隊成員具備 Kafka 維運經驗，且 Managed Kafka 費用偏高，因此 Kafka 維持自建。

#### RabbitMQ：整合至 Redis

為了減少需要維護的組件，RabbitMQ 被整合至 Redis。

Celery Queue 可由 Redis 承擔，因此可以降低維運負擔。

### 避坑建議

在決定自建或 Managed 前，應先列出每個組件是否有 HA 需求，並評估自建維運成本。

事後遷移的代價，通常遠高於事前評估。

---

## 7. 踩雷 3：SDK 配置陷阱

Sentry 跑起來後，下一步是讓各服務接入 Sentry SDK。

一開始看起來很簡單：

- 在各語言服務接入 Sentry SDK
- 讓錯誤自動上報
- 啟用 PII 收集，取得使用者 IP
- 只收 Error 事件，不收 Transaction
- 避免不必要的儲存與處理負擔

看起來只是裝 SDK、改幾行設定，但實際上也有坑。

### 沒想到：配置範例咬了我們一口

#### `traces_sample_rate` 預設範例設為 1.0

Sentry 建立 Project 時提供的 SDK 配置範例，預設會將 `traces_sample_rate` 設為 `1.0`。

團隊照著範例使用後，在不知情的情況下收集了 Transaction 資料，導致資料量暴增。

但團隊原本的目標是：

> 只收 Error，不收 Transaction。

#### `send_default_pii=true` 多收了一些東西

為了收集使用者 IP，團隊啟用了：

```text
send_default_pii=true
```

結果除了 IP 之外，瀏覽器 Cookies 也被送進 Sentry。

出於隱私合規考量，這些 Cookies 不應該被收集。

這兩個坑的共同點是：

> 範例程式碼預設開啟，但副作用沒有明確警告。

### 證據：官方串接範例預設

簡報中展示了官方串接範例，說明 Sentry Project 建立時提供的 SDK 初始化設定，可能預設帶有容易造成資料量或隱私風險的參數。

因此不能盲目複製官方範例。

### 還好：不信任預設值，每個參數都自己指定

#### 明確設定 `traces_sample_rate=0`

團隊不再照抄範例，而是在每個語言的 SDK 初始化時都明確寫上：

```text
traces_sample_rate=0
```

這確保不收集 Transaction 資料。

#### 使用 Sensitive Fields 過濾敏感資料

團隊在 Sentry 設定中，將 Cookies 加入 Global Sensitive Fields。

如此一來，Sentry 在接收事件時會自動隱藏這些欄位。

這樣可以同時達成：

- 保留 IP 收集
- 避免 Cookies 外洩
- 降低隱私合規風險

### 避坑建議

應建立公司內部 SDK 設定範本，不要讓工程師自行猜測預設值是什麼。

每個 SDK 參數都應明確指定，尤其是：

- 是否收 Transaction
- 是否收 PII
- 是否收 Cookies
- Sample rate
- Environment
- Release version

---

## 8. 踩雷 4：版本地雷

團隊選用了當時的最新穩定版 Sentry 25.7.0。

當時看起來是安全選擇，因為：

- 是最新穩定版
- 文件完整
- changelog 沒有明顯 breaking change
- 看起來適合 production 使用

同時，團隊也設定了 Metrics Alert 告警規則，例如：

> Error 發生頻率超過每分鐘 100 次，就發送 Slack 通知。

部署完成後，SDK 串接成功，團隊信心滿滿地在 production 開始正式使用 Sentry。

但後來發現一件讓人冷汗直流的事。

### 沒想到：告警靜默，我們渾然不知

Metrics Alert 功能在該版本失效，導致告警不會觸發。

這代表雖然 Sentry 有收到事件，也能顯示 Issue，但原本期待的主動告警能力失效。

最危險的是：

> 告警失效本身不一定會主動告警。

### 證據：GitHub Issue 回報

團隊後續追蹤 GitHub Issue，確認這是官方已知問題。

整個過程可以整理為：

1. 安裝 25.7.0  
   滿懷期待完成部署。

2. 發現災難  
   Metrics Alert 功能失效，告警不會觸發。

3. 追蹤 Bug  
   從官方 GitHub Issue 確認問題。

4. 嘗試應對  
   在 staging 環境升版 26.2.1，並嘗試社群 workaround。

教訓是：

> 升級前後，務必驗證 Alert 端到端流程。

### 還好：建立輔助監控與替代方案

#### Rollback 至上一個穩定版本

確認 Bug 在最新 26.2.1 尚未解決後，團隊立即 rollback 回上一個穩定版本。

#### 改用 Issue Alert 取代 Metrics Alert

Issue Alert 的彈性不如 Metrics Alert，但在當前場景仍然足夠使用。

團隊改用 Issue Alert 維持基本告警能力。

### 避坑建議

不要盲目追最新版。

自建開源軟體的生存法則是：

- 等社群回報穩定後再升級
- 升級前先在 staging 驗證
- 升級後驗證核心功能
- 對告警系統做端到端測試
- 保留 rollback 機制

---

## 9. 踩雷 5：資源調校

Sentry 跑起來後，真正的挑戰才開始。

Production 環境需要承載真實事件量：

- staging 每天可能只有幾百筆事件
- production 可能變成每天幾萬筆事件

工程師也會期待錯誤發生後幾秒內就能在 Sentry UI 看到 Issue，而不是幾分鐘後才出現。

官方建議規格只能當起點，實際需求仍需依事件量壓測驗證。

### 沒想到：系統跑起來了，但事件不見了

#### Celery Workers 消化不及

高事件量時，Celery queue 持續堆積。

Worker 消費速度跟不上事件生產速度，導致事件處理延遲。

#### Kafka Consumer Lag 累積

Kafka consumer group lag 持續累積，事件處理延遲，影響 Sentry 即時性。

這代表事件並不是真的消失，而是卡在 queue 或 lag 中，還沒被處理到 UI 可見的狀態。

### 還好：監控先行，壓測驗證

#### 增加 Celery Workers

團隊先調高 Celery worker replica 數量驗證效果，後續再改用 autoscaling 機制。

#### 增加 Kafka Partition 和 Consumer

團隊增加 Kafka partition 與 consumer 數量，使事件可以平行處理，並充分利用新增 partition。

#### 監控 Queue Size 與 Consumer Lag

團隊針對以下指標建立監控告警：

- RabbitMQ queue length
- Redis queue 狀態
- Kafka consumer lag
- Worker processing rate
- Event ingestion delay

目標是在問題惡化前就先被發現。

### 避坑建議

上線前應模擬真實事件量做壓測，確認 lag 能有效消化後再進 production。

尤其要確認：

- queue 是否會持續成長
- consumer lag 是否會歸零
- UI 是否能在合理時間內看到事件
- autoscaling 是否有效
- 高峰後系統是否能恢復

---

## 10. 踩雷 6：User 管理自動化

團隊也希望透過 API 自動化管理 Sentry 使用者生命週期。

目標包括：

- 自動新增 User
- 自動刪除 User
- 整合 HR 系統
- 新成員加入時自動加入 Sentry 組織
- 成員離職或權限異動時自動移除
- 降低人工維運負擔

表面上看起來，只要串接 Sentry API 就能完成。

### 沒想到：Remove 不等於 Delete

#### Remove User API 其實是 Leave Org

Sentry 的 Remove Member API 並不是刪除帳號，而是讓該 user 離開組織。

帳號本身仍然存在於 Sentry 系統中。

#### 重建 User 報錯：User 已存在

當團隊嘗試重新建立同一個 user 時，Sentry 回傳錯誤，因為該帳號從未真正刪除，只是離開了組織。

自動化流程因此卡住，無法正常重建成員。

核心問題是：

> Remove 和 Delete 是兩件事，但 API 文件的命名容易讓人混淆。

### 證據：Member 頁面的操作是 Leave，不是 Delete

簡報中展示 Sentry Member 頁面操作，指出介面語意其實是 Leave，而非 Delete。

這說明 API 行為更接近「離開組織」，而不是真正刪除使用者帳號。

### 還好：Shell 清帳與事前驗證 API 行為

#### 透過 Sentry Shell 手動刪除殘留帳號

團隊進入 Sentry 容器執行 `sentry shell`，找出所有已 Leave Org 但未真正刪除的 user，逐一清除乾淨。

這讓自動化流程得以正常重建帳號。

#### 事前確認 API 實際行為

在串接任何 Sentry API 前，應先使用測試帳號實際驗證 API 行為，而不是只看文件名稱就假設語意。

### 避坑建議

自動化流程上線前，務必在 staging 環境跑完整循環：

```text
create → remove → recreate
```

並確認每個步驟的實際效果。

---

## 11. 現況與下一步

經過部署、選型、SDK、版本、資源與 API 管理等多個踩坑後，目前 Sentry 已穩定運行。

### 現況

#### Sentry 穩定運行中

6 個坑都踩過後，系統目前已能穩定承載 production 流量，Dev / Ops 都已在日常使用。

#### Error Tracking 補上最後一塊拼圖

原本可觀測性堆疊已包含：

- Metrics
- Logs
- Traces

現在再加上 Error Tracking，使可觀測性拼圖更加完整。

### Next Action：Grafana Sentry Data Source

下一步計畫是透過 Grafana Plugin 將 Sentry Issues 拉入 Dashboard。

目標是讓 Dev / Ops 不需要在 Grafana 與 Sentry 之間來回切換工具，而能在同一個觀測入口中看到：

- Metrics
- Logs
- Traces
- Sentry Issues

---

## 12. Sentry Cloud vs. Self-Hosted：怎麼選？

最後，簡報整理了 Sentry Cloud 與 Self-Hosted 的選擇建議。

### Sentry Cloud：SaaS

#### 優點

- 零維運負擔，開箱即用
- 自動升級，永遠使用最新版
- 官方 SLA 保障，穩定性有保證
- 適合快速驗證與小團隊

#### 缺點

- 資料託管，可能有隱私疑慮
- 事件量大時費用快速攀升
- 客製化空間有限

### Self-Hosted

#### 優點

- 資料完全自主
- 減少機敏資料外流風險
- 大量事件下成本可控
- 可客製化 Relay 規則、Retention 等設定

#### 缺點

- 需要專人維運 Kafka、ClickHouse、Redis 等元件
- 版本升級可能踩雷
- 需要建立降版與 rollback 機制
- 需要自行負責 HA、監控、備份與壓測

### 誰適合選 Sentry Cloud？

適合選 Sentry Cloud 的情境包括：

- 人力有限，沒有專職 SRE / DevOps
- 機敏資料要求不嚴格
- 事件量不大，例如每月小於數百萬筆
- 想快速上線，不想踩自建維運坑

### 誰適合選 Self-Hosted？

適合選 Self-Hosted 的情境包括：

- 機敏資料有合規需求
- 有足夠 SRE / DevOps 人力維運
- 事件量大，SaaS 費用難以接受
- 已有 Kubernetes / EKS 基礎設施
- 團隊有能力維運 Kafka、ClickHouse、Redis 等元件

### 不確定怎麼選？

如果還不確定，建議：

> 先從 Cloud 開始，等規模到了再評估遷移。

---

## 13. Q&A 與總結

本次分享的核心觀點可以總結為三個方向。

### Log 不等於 Error Tracking

即使已經有 Loki，仍然不代表已經具備 Error Tracking。

Logs 與 Error Tracking 的設計目的不同：

- Logs 適合記錄事件與時間線
- Error Tracking 適合聚合錯誤、保存上下文、管理 Issue 生命週期

### 踩坑避坑實錄

Self-Hosted Sentry 的挑戰不只在部署，而是涵蓋整個生命週期：

- 部署
- 基礎設施選型
- SDK 配置
- 版本升級
- 資源調校
- API 自動化
- User 管理
- 告警驗證

### 要不要自建？

Self-Hosted 與 SaaS 沒有絕對好壞，重點是要依照組織需求選擇。

評估時應考量：

- 資料隱私
- 合規要求
- 事件量
- 維運人力
- 現有基礎設施
- 成本模型
- 對客製化的需求

最後，Self-Hosted Sentry 能補上 Grafana Observability Stack 中 Error Tracking 這塊拼圖，但它不是免費午餐。

它帶來資料掌控權與成本彈性，同時也帶來 Kafka、ClickHouse、Redis、版本升級、資源調校與 API 行為差異等維運責任。

真正的重點不是「能不能部署起來」，而是：

> 能不能讓它在 production 長期穩定、可監控、可升級、可維運。
