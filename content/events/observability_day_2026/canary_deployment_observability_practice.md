# 金絲雀部署的可觀測性實踐

> 謝政廷 (Duran)／Section Manager, TSMC / GitHub Star

> 2026-04-23 14:40 - 15:20 @ 張榮發基金會國際會議中心 11F

> [議程](https://o11yday.ithome.com.tw/2026/session-page/4400)

> [簡報](https://drive.google.com/file/d/1Z4CSDLuDnh8D_vsoWqXaKLCdd2BasKVA/view)

> 金絲雀部署是降低上線風險的關鍵策略，但如何在流量切換過程中即時掌握系統健康狀態？
>
> 本場次將分享在 Kubernetes 環境中，以 APISIX 作為 API Gateway 實現動態流量分流，搭配 Prometheus 指標監控、Grafana 視覺化儀表板、OpenTelemetry Collector 收集 Traces，以及 Tempo 進行分散式追蹤的完整可觀測性架構。
>
> 透過實際案例，展示如何在逐步升級流量比例（10% → 50% → 100%）的過程中，即時觀察 Stable 與 Canary 版本的 QPS、延遲、錯誤率差異，並快速定位問題根因。

> **聽眾收穫：**
>
> - 學會如何將 APISIX metrics（延遲、狀態碼）與應用程式 Traces 透過 OTel Collector 統一收集
> - 取得 4 種 Dashboard JSON 範本
> - 了解 Admin API 動態調整權重搭配 ServiceMonitor 自動發現的雙軌管理策略
> - 掌握 Tempo Trace 面板與 Prometheus 指標查詢技巧

---

## 前言：部署不是問題，看不見才是問題

這場分享聚焦在金絲雀部署與可觀測性的結合，使用的主要技術包括：

- APISIX
- OPA
- OpenTelemetry
- Grafana
- ArgoCD

講者一開始提出幾個常見問題：

- 週四上午部署新版本，全公司使用者同時踩到 bug。
    - 問題：沒有金絲雀保護。
- 工程師凌晨三點被叫起來，切回穩定版時操作錯誤，導致事故擴大。
    - 問題：沒有快速回滾機制。
- 明明有金絲雀部署，但事故發生時沒人知道 canary 錯誤率已經高了 5 倍。
    - 問題：沒有可觀測性。

核心觀念是：

> 部署不是問題，看不見才是問題。

金絲雀部署本身並不困難，真正困難的是在流量逐步切換時，能不能清楚看見新版本是否正在造成錯誤、延遲或異常行為。

---

## 今天的三個核心命題

本場分享圍繞三個問題展開。

### 1. 如何讓「對的人」先測試新版本？

對應解法：

```text
OPA 策略驅動路由
```

也就是透過策略判斷哪些使用者、哪些請求、哪些服務鏈路應該進入 canary 版本。

### 2. 如何在事故發生前就「看到」問題？

對應解法：

```text
可觀測性：Metrics + Traces + Logs
```

透過指標、追蹤與日誌，在問題擴大前先觀察到異常。

### 3. 如何用數據驅動部署決策，而非靠直覺？

對應解法：

```text
漸進式金絲雀 + SLO Metrics
```

每個階段是否繼續擴大流量，不靠感覺，而是看 SLO、錯誤率、延遲與版本差異。

---

## 架構總覽

整體架構可概念化如下：

```text
Internet
  ↓
APISIX Gateway
  ├─ OPA：策略決策
  ├─ OTel：追蹤
  ├─ Prometheus：指標
  └─ Traffic Split：流量分配
        ├─ Spring Boot v1 stable
        └─ Spring Boot v2 canary

ArgoCD
  ↓
GitOps

Observability Stack
  Prometheus → Grafana ← Tempo ← OTel Collector

11 Dashboards
```

這個架構的重點是：

- APISIX 作為閘道層，負責流量控制與可觀測性資料產生。
- OPA 負責策略決策，判斷請求要進 stable 或 canary。
- OpenTelemetry 負責追蹤資料。
- Prometheus 負責指標收集。
- Grafana 負責視覺化。
- Tempo 負責分散式追蹤。
- ArgoCD 與 GitOps 負責配置版本化與自動同步。

---

## 為什麼選 APISIX？

APISIX 在這個架構中扮演 API Gateway 與流量控制核心。

選擇 APISIX 的原因包括：

### 高效能

APISIX 基於 Nginx / OpenResty，適合高併發、低延遲場景。

這對金絲雀部署很重要，因為閘道層位於所有流量入口，不能成為效能瓶頸。

### 原生外掛支援

APISIX 原生支援多種外掛，包括：

- `traffic-split`
- `prometheus`
- `opentelemetry`
- `opa`

因此可以在同一個閘道層完成：

- 流量分配
- 指標暴露
- 追蹤串接
- 策略判斷

### 動態配置

APISIX 可透過 Admin API 即時調整權重，不需要重新啟動服務。

例如：

- stable 從 100% 調整到 90%
- canary 從 0% 調整到 10%

調整後可以立即生效，並推送到 Git Repository 留下紀錄。

這對部署治理很重要，因為它同時滿足：

- 快速切換
- 不需重啟
- 可追溯
- 可審計
- 可回復

### 豐富生態

APISIX 有 80+ 個外掛，涵蓋：

- 安全
- 限流
- 認證
- 可觀測性
- 流量治理

---

## APISIX vs Nginx Ingress

簡報中比較了 APISIX 與 Nginx Ingress 在金絲雀部署場景中的差異。

| 功能       | APISIX             | Nginx Ingress         |
| ---------- | ------------------ | --------------------- |
| 金絲雀切換 | Admin API 一鍵切換 | annotation + re-apply |
| OPA 整合   | 原生外掛           | 需要額外 sidecar      |
| 動態配置   | 不需重啟           | 需要 reload           |

APISIX 的優勢在於它更適合需要即時調整、策略化控制與可觀測性整合的部署場景。

---

## OPA 在金絲雀部署中的角色

OPA 負責策略決策，決定流量要進入 stable 或 canary。

範例策略如下：

```rego
# canary.rego — 核心邏輯
package apisix.canary

# Header-based: QA 帶 x-canary
canary_by_header if {
  input.request.headers["x-canary"] == "true"
}

# 路由決策
route_target := "canary" if { canary_by_header }
route_target := "stable" if { not canary_by_header }
```

這代表當請求帶有：

```text
x-canary: true
```

就會被導向 canary。

否則預設導向 stable。

---

## OPA 策略使用場景

OPA 可以支援多種金絲雀流量控制情境。

| 使用場景   | 條件                 | 目標版本        |
| ---------- | -------------------- | --------------- |
| QA 測試    | `x-canary: true`     | canary          |
| 內部員工   | `canary=true` cookie | canary          |
| 微服務鏈路 | 服務間附帶 header    | 整條鏈路 canary |
| 一般使用者 | 無 header / cookie   | stable          |

這種做法讓金絲雀部署不只是百分比切流，而是可以精準控制：

- 哪些人先測
- 哪些請求先走新版本
- 哪些服務鏈路維持一致 canary
- 一般使用者是否仍留在 stable

也就是：

> OPA 讓你精準控制誰先走新路，而不是讓全部使用者一起賭。

---

## OPA Bundle 更新流程

OPA 策略透過 bundle 方式更新。

流程如下：

```text
Rego 更新
  ↓
CI/CD 打包 bundle.tar.gz
  ↓
Azure Blob Storage
  ↓
OPA 每 30-60 秒自動拉取
  ↓
策略即時生效
```

這樣的好處是：

- 策略可以版本化。
- 策略變更可以經過 CI/CD。
- 可以進行 review 與測試。
- 策略更新不需要重新部署應用程式。
- 能符合 Policy as Code 的治理方式。

---

## OpenTelemetry 在架構中的定位

OpenTelemetry 負責將 APISIX 與 Spring Boot 應用程式的追蹤資料串接起來。

架構如下：

```text
APISIX
  ↓ OTel Plugin

Spring Boot
  ↓ OTel Java Agent 自動注入

OTel Collector
  ├─ Receivers
  ├─ Processors
  └─ Exporters

Tempo
  ↓
Traces
```

APISIX 透過 OTel plugin 產生閘道層追蹤資料。

Spring Boot 則透過 OTel Java Agent 自動注入，不需要修改程式碼。

---

## 關鍵設計：用 service.name 區分版本

在追蹤資料中，使用 `service.name` 區分 stable 與 canary。

| 版本   | service.name      | env          |
| ------ | ----------------- | ------------ |
| Stable | `demo-api-stable` | `production` |
| Canary | `demo-api-canary` | `canary`     |

這樣在 Grafana / Tempo 中，就可以依照 `service.name` 查詢：

- stable 的請求鏈路
- canary 的請求鏈路
- 兩個版本的延遲差異
- 兩個版本的錯誤路徑差異

---

## Zero Intrusion：不需要改一行 Java 程式碼

Spring Boot 應用程式透過 OTel Java Agent 注入。

範例設定：

```bash
ENV JAVA_TOOL_OPTIONS="-javaagent:/otel-agent.jar"
ENV OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
```

這代表應用程式本身不需要修改 Java 程式碼，就能產生追蹤資料。

這種方式降低了導入可觀測性的成本，也更適合既有系統逐步導入。

---

## 雙版本並行：Stable / Canary 架構

Kubernetes namespace 為：

```text
Namespace: app
```

Stable 版本：

```text
Deployment: spring-boot-stable
replicas: 2
APP_VERSION=v1
OTEL_SERVICE_NAME=demo-api-stable
Service: spring-boot-stable:8080
```

Canary 版本：

```text
Deployment: spring-boot-canary
replicas: 2
APP_VERSION=v2
OTEL_SERVICE_NAME=demo-api-canary
Service: spring-boot-canary:8080
```

這種設計讓 stable 與 canary 兩個版本可以同時存在，並透過 APISIX 控制流量比例。

---

## 流量控制雙層架構

本架構使用雙層流量控制。

### Layer 1：OPA 精確路由

OPA 負責根據請求條件做精確路由。

```text
x-canary: true  → canary
canary cookie   → canary
預設            → stable
```

這一層處理的是「誰應該進 canary」。

### Layer 2：APISIX 權重路由

APISIX 負責權重式流量分配。

```text
stable: weight 100 → 100% 流量
canary: weight   0 →   0% 流量
```

權重可以透過 Admin API 動態調整，並即時生效。

範例：

```bash
curl http://$LB_IP/api/hello
# → {"version":"v1"}

curl -H "x-canary: true" http://$LB_IP/api/hello
# → {"version":"v2"}
```

雙層架構的優點是：

- OPA 控制精準對象。
- APISIX 控制整體比例。
- 可以同時支援測試人員導流與百分比切流。
- 可在不重新部署的情況下調整權重。

---

## 11 個 Grafana Dashboards：全方位監控

簡報中設計了 11 個 Grafana 儀表板，共 60+ 監控面板。

| Dashboard                | 用途       |
| ------------------------ | ---------- |
| APISIX Overview          | 總覽與健康 |
| APISIX Route Metrics     | 路由與流量 |
| Canary Traffic Split     | 金絲雀核心 |
| Spring Boot Metrics      | 應用層     |
| Node Metrics             | 基礎設施   |
| Upstream Health          | 後端服務   |
| Security & Rate Limiting | 安全防護   |
| SLI / SLO                | 穩定服務   |
| Plugin Metrics           | 外掛與擴充 |
| Nginx / OpenResty        | 底層系統   |
| Performance & Cache      | 效能與快取 |

這些 dashboard 的目的不是只提供視覺化，而是建立部署決策依據。

---

## Canary Traffic Split Dashboard

Canary Traffic Split Dashboard 是金絲雀部署的核心儀表板，也就是部署指揮中心。

主要面板包括：

### 流量分配

使用圓餅圖顯示：

```text
Stable vs Canary 即時百分比
```

這能確認實際流量是否符合預期權重。

### QPS 對比

使用雙軸折線圖比較：

- stable QPS
- canary QPS

用來觀察流量是否正常進入 canary。

### P95 延遲對比

比較：

- stable P95
- canary P95

並搭配閾值標線，判斷 canary 是否比 stable 慢太多。

### 5xx 錯誤率

監控：

```text
canary 5xx 是否 > stable × 5
```

如果 canary 錯誤率明顯高於 stable，代表新版本可能有問題。

### Tempo Traces

透過 `service.name` 查詢 trace：

- stable trace
- canary trace
- 版本間行為差異

### Traffic Anomaly

監控流量異常，例如：

- 流量突然暴增
- 流量突然歸零
- canary 沒有流量
- stable / canary 比例不符合預期

重要觀念是：

> Dashboard 不只是「看的」——每個面板都要有對應的操作 SOP。

---

## APISIX 核心監控指標

APISIX 提供多種核心指標。

### 請求指標

```text
apisix_http_status
apisix_http_latency_bucket
```

`apisix_http_latency_bucket` 可用於計算：

- P50
- P95
- P99

latency type 包括：

- request
- upstream
- apisix

### 連線指標

```text
apisix_nginx_http_current_connections
```

connection state 包括：

- active
- waiting
- reading
- writing

### 健康指標

```text
apisix_etcd_reachable
apisix_upstream_status
apisix_etcd_modify_indexes
```

其中：

```text
apisix_etcd_reachable = 1 代表 UP
apisix_etcd_reachable = 0 代表 DOWN
```

### 記憶體與資源指標

```text
apisix_shared_dict_capacity_bytes
apisix_shared_dict_free_space_bytes
apisix_node_info
```

### Prometheus Scrape 設定

```text
ServiceMonitor scrape interval: 15s
metric_prefix: apisix_
port: 9091
```

---

## 漸進式金絲雀擴散策略

金絲雀部署不應直接從 0% 切到 100%，而應該逐步擴散。

### Step 1：10%

```text
stable: 90%
canary: 10%
```

先部署並觀察。

### Step 2：50%

```text
stable: 50%
canary: 50%
```

觀察 5 到 10 分鐘，確認 dashboard 指標是否穩定。

### Step 3：100%

```text
stable: 0%
canary: 100%
```

全面切換，完成 full promotion。

### Rollback：0%

```text
stable: 100%
canary: 0%
```

任何時刻都可以一鍵回滾。

範例指令：

```bash
./scripts/canary-switch.sh --stable 90 --canary 10
```

特性：

- 即時生效。
- 無需 redeploy。
- 異常時可立即切回 stable。

操作原則：

```text
先看再切
異常就暫停
錯誤就回滾
每階段觀察 10 分鐘
```

---

## 決策指標：什麼時候該停？什麼時候該進？

是否擴大 canary 流量，應該由指標決定，而不是靠直覺。

| 指標                    | 綠燈：安全繼續 | 黃燈：謹慎觀察 | 紅燈：立即回滾 |
| ----------------------- | -------------: | -------------: | -------------: |
| P95 延遲                |        < 200ms | 200ms ~ 1000ms |       > 1000ms |
| 5xx 錯誤率              |           < 1% |        1% ~ 5% |           > 5% |
| Canary vs Stable 延遲差 |          < 20% |      20% ~ 50% |          > 50% |
| OPA 策略錯誤            |              0 |            > 0 |       持續增加 |

這就是 SLO-driven Rollout 的核心：

> 用 SLI / SLO 指標決定是否繼續部署，而不是讓人憑感覺判斷。

---

## 對應 Prometheus Alert Rules

簡報中提供了對應的 Prometheus 告警規則概念。

### CanaryHighErrorRate

```yaml
- alert: CanaryHighErrorRate
  expr: canary 5xx > 5%
  for: 2m
  severity: critical
  annotations:
    runbook: "./scripts/canary-switch.sh --stable 100 --canary 0"
```

當 canary 5xx 錯誤率超過 5%，持續 2 分鐘，就應觸發 critical 告警，並提供回滾 runbook。

### CanaryHighLatency

```yaml
- alert: CanaryHighLatency
  expr: P95 > 1s
  for: 3m
  severity: critical
```

當 canary P95 延遲超過 1 秒，持續 3 分鐘，即應視為嚴重異常。

---

## Alert 的角色：告警即行動指令

告警不應只是通知，而應該是可行動的指令。

主要告警包括：

| Alert                | 條件                       | 持續時間 | 等級     |
| -------------------- | -------------------------- | -------: | -------- |
| CanaryHighErrorRate  | canary 5xx > 5%            |   2 分鐘 | critical |
| CanaryHighLatency    | canary P95 > 1s            |   3 分鐘 | critical |
| CanaryLatencyDrift   | canary 延遲 > stable × 1.5 |   5 分鐘 | warning  |
| CanaryTrafficAnomaly | 流量突然歸零或暴增         |   1 分鐘 | warning  |
| UpstreamUnhealthy    | 健康檢查失敗               |   1 分鐘 | critical |

### 告警要 Actionable

一個好的告警必須清楚回答三個問題：

1. 什麼壞了？
    - 例如：Canary 延遲飆升。
2. 影響是什麼？
    - 例如：10% 使用者體驗下降。
3. 該怎麼做？
    - 例如：執行 `canary-switch.sh` 回滾。

告警管道可以包含：

- On-call
- 通訊軟體通知
- Webhook 自動回滾

這裡的重點是：

> 告警不是只把人叫醒，而是要讓人知道下一步要做什麼。

---

## 可觀測性三支柱在本架構的對應

本架構使用 Metrics、Traces、Logs 三支柱共同支撐金絲雀部署。

### Metrics

工具：

- Prometheus
- Grafana

資料來源：

- APISIX prometheus plugin
- Spring Boot Actuator
- OTel Collector

用途：

- QPS
- 延遲
- 錯誤率
- JVM 狀態

### Traces

工具：

- OpenTelemetry
- Tempo
- Grafana

資料來源：

- APISIX OTel plugin
- Spring Boot OTel Java Agent

用途：

- 請求鏈路追蹤
- 找出版本間的行為差異
- 比對 stable 與 canary 的呼叫路徑

### Logs

工具：

- EFK / ELK
- Kibana

資料來源：

- Spring Boot stdout
- APISIX access log

用途：

- 錯誤詳情
- 除錯分析
- Trace ID 關聯

在 Kibana 中，可以：

- 按 `x-route-to` header 區分 stable / canary 日誌。
- 按 `trace-id` 關聯完整鏈路。

---

## 資料流全景圖

整體資料流如下：

```text
APISIX Gateway
  ├─ prometheus plugin
  ├─ OTel plugin
  └─ access log

Spring Boot Stable / Canary
  ↓
OTel Collector
  ├─ Receivers
  │   ├─ gRPC 4317
  │   └─ HTTP 4318
  ├─ Processors
  │   ├─ batch
  │   └─ memory_limiter
  └─ Exporters
      ├─ Tempo
      └─ Prometheus

Tempo
  ↓
Traces

Prometheus
  ↓
Metrics

Grafana
  ↓
11 Dashboards

Prometheus AlertManager
  ├─ OnDuty
  ├─ 通訊軟體
  └─ 自動回滾腳本
```

Log pipeline：

```text
Filebeat / Fluentd
  ↓
Elasticsearch
  ↓
Kibana
```

日誌中可透過 `x-route-to` header 區分版本。

GitOps 流程：

```text
ArgoCD watches this repo
所有配置版本化，可審計、可回溯
```

---

## 金絲雀部署 + 可觀測性：8 大最佳實踐

### 1. GitOps First

所有配置都應版本化，並由 ArgoCD 自動同步。

好處包括：

- 可審計
- 可回溯
- 可重現
- 降低人工操作錯誤

### 2. Policy as Code

OPA 策略應走 CI/CD 流程。

這代表策略變更需要：

- review
- 測試
- 紀錄
- 版本化

策略不是臨時手動修改，而是正式治理的一部分。

### 3. 漸進式部署

永遠不要直接從 0% 切到 100%。

至少應有三階段：

```text
10% → 50% → 100%
```

每個階段都應觀察指標後再決定是否繼續。

### 4. SLO-driven Rollout

用 SLI / SLO 指標決定是否繼續部署。

關鍵指標包括：

- 延遲
- 錯誤率
- canary vs stable 差異
- OPA 策略錯誤
- 流量異常

### 5. 自動回滾

透過 Prometheus Alert + Webhook，在錯誤率超標時自動觸發回滾。

例如：

```bash
./scripts/canary-switch.sh --stable 100 --canary 0
```

自動回滾可以降低凌晨人工操作錯誤，也能縮短事故擴大時間。

### 6. Trace-based Testing

在 canary 階段利用 distributed tracing 比對新舊版本行為。

可觀察項目包括：

- 呼叫鏈是否改變
- 下游服務是否異常
- 某段 span 是否變慢
- canary 是否出現 stable 沒有的錯誤

這能補足單看 metrics 不容易看出的行為差異。

### 7. Dashboard as Runbook

Dashboard 不只是顯示資訊，而應該附帶 SOP。

每個面板都應回答：

- 指標異常代表什麼？
- 可能原因是什麼？
- 應該先查哪裡？
- 是否該暫停擴散？
- 是否該回滾？
- 對應指令是什麼？

也就是：

> Dashboard 是你的部署 Runbook。

### 8. Chaos Engineering

應定期注入故障，驗證可觀測性和告警是否正常運作。

目的包括：

- 驗證告警是否觸發。
- 驗證 dashboard 是否看得出異常。
- 驗證自動回滾是否正常。
- 驗證團隊是否知道如何處理。
- 驗證 runbook 是否有效。

---

## 與業界方案比較

簡報中比較了幾種常見方案。

| 方案          | 特色                                       | 可觀測性整合            | 複雜度 |
| ------------- | ------------------------------------------ | ----------------------- | ------ |
| Argo Rollouts | Kubernetes 原生，自動 progressive delivery | Prometheus Analysis     | 中     |
| Flagger       | 搭配 Istio / Linkerd，自動金絲雀           | 內建 metric check       | 中高   |
| Istio + Kiali | Service Mesh 完整方案                      | 完整但複雜度高          | 高     |
| 本架構        | 策略驅動、閘道層控制，無 Service Mesh      | Metrics + Traces + Logs | 中低   |

---

## 本架構的差異化優勢

本架構的主要優勢包括：

- 不需要 Service Mesh，降低複雜度與資源消耗。
- OPA 提供比 annotation 更靈活的策略控制。
- 閘道層觀測 + 應用層觀測，雙管齊下。
- Admin API 動態切換，不需重新部署。
- 11 個 Dashboard，提供企業級全方位監控。
- GitOps 讓所有操作可追溯、可重現。

這種設計適合想要導入金絲雀部署，但又不希望一開始就承擔完整 Service Mesh 複雜度的團隊。

---

## Key Takeaways

### 1. 金絲雀部署不難，難的是看見它

沒有可觀測性的金絲雀，就像盲人開車。

如果看不到 canary 的錯誤率、延遲、流量比例與 trace 差異，就很難判斷是否該繼續擴散或立即回滾。

### 2. OPA 讓你精準控制誰先走新路

透過 OPA，可以讓 QA、內部員工或特定服務鏈路先進入 canary。

這避免了所有使用者一起承擔新版本風險。

### 3. 漸進式擴散 + SLO 閾值 = 數據驅動部署決策

部署過程應讓指標說話，而不是靠直覺。

例如：

- P95 延遲是否超標？
- 5xx 是否增加？
- canary 是否比 stable 慢太多？
- OPA 策略是否出錯？

### 4. Metrics + Traces + Logs 三支柱缺一不可

三者各司其職：

- Prometheus / Grafana 看整體趨勢與 SLO。
- Tempo 看請求鏈路與版本差異。
- Kibana 看錯誤詳情與 trace-id 關聯。

### 5. Dashboard 是你的部署 Runbook

每個 dashboard 面板都要有對應操作 SOP。

當指標異常時，工程師應該知道：

- 看哪裡
- 查什麼
- 暫停或回滾的條件
- 執行哪個指令

### 6. GitOps 確保可重現性

所有操作都有紀錄，出事時能追溯。

這讓部署不只是快速，也能被審計、回放與復原。

---

## 參考資料

- APISIX：apisix.apache.org
- OPA：openpolicyagent.org
- OpenTelemetry：opentelemetry.io
- Grafana Tempo：grafana.com/oss/tempo

---

## 總結

這場分享展示了如何以 APISIX、OPA、OpenTelemetry、Prometheus、Grafana、Tempo 與 ArgoCD 建立一套安全的金絲雀部署架構。

核心精神不是單純把新版本切一小部分流量出去，而是建立一套可以「看見、判斷、暫停、回滾、追溯」的部署系統。

本架構的關鍵設計包括：

- 用 APISIX 作為高效能閘道與流量分配核心。
- 用 OPA 做策略驅動路由，精準控制誰進 canary。
- 用 OpenTelemetry 串接追蹤。
- 用 Prometheus 與 Grafana 監控 SLO 與核心指標。
- 用 Tempo 比對 stable 與 canary 的請求鏈路。
- 用 Kibana 透過 trace-id 與 header 查詢日誌。
- 用 ArgoCD 與 GitOps 保存所有配置紀錄。
- 用告警與 Webhook 建立自動回滾能力。

最重要的觀念是：

> 金絲雀部署不難，難的是看見它。
>
> Dashboard 不只是看的，而是部署決策與事故處理的 Runbook。
