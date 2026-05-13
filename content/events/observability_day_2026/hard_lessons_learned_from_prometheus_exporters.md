# Hard Lessons Learned from Prometheus Exporters: Observability Challenges in Kubernetes Environments

> 邱宏瑋 (hwchiu)／台積電 經理

> 2026-04-23 10:15 - 10:55 @ 張榮發基金會國際會議中心 11F

> [議程](https://o11yday.ithome.com.tw/2026/session-page/4408)

> [簡報](https://drive.google.com/file/d/1fHZP7oUhXboG5_942W7h1gx112gRX_f_/view)

> 雖然 Prometheus 已成為現代雲原生環境中維護與可觀測性的業界標準，但在大規模 Kubernetes 生產叢集中，我們仍頻繁遭遇許多「指標滿天飛、Grafana 儀表板美輪美奐，問題卻始終找不到」的困境。
>
> 本次演講將分享過去在大型 Kubernetes Cluster 實際運維中遇到的真實觀測挑戰：這些隱藏的觀測盲點如何悄然侵蝕叢集穩定性、影響 Pod 排程、資源爭用與整體服務可用性，卻又為什麼無法被常見的 Alerting 規則、標準 Dashboard 與傳統排查流程所捕捉。
>
> 透過這些血淚教訓，我們將一起重新審視 Prometheus Exporter 的使用邊界，並探討如何突破傳統思維，打造更務實、更有效的可觀測性策略，讓團隊在大型叢集環境中真正做到「早發現、快定位、少 downtime」。

> **聽眾收穫：**
>
> 理解為什麼 Prometheus 已是業界標準，卻在大規模 Kubernetes 生產環境中仍會出現「指標滿滿、Grafana 很漂亮，問題卻找不到」的常見困境。
>
> 多個真實案例剖析：這些觀測盲點如何悄悄影響叢集穩定性、Pod 排程、資源爭用與整體可用性，卻被常規的 Alerting、Dashboard 與排查流程完全漏掉。
>
> 實戰層面的洞見與可立即落地的建議：如何超越標準 Prometheus Exporter 的思維，重新設計 observability 策略，避免同樣的「看不見的問題」再次發生。

---

## Prometheus 在 Kubernetes 中的 Scrape 拓撲

在 Kubernetes 環境中，Prometheus 通常透過 pull model 定期 scrape 各種 exporter 與應用程式端點，將資料寫入 TSDB，再透過 PromQL 查詢，並由 Grafana 呈現儀表板。

典型資料來源包括：

- **node-exporter**
    - 例如 `:9100/metrics`
    - 提供節點層級的 CPU、記憶體、磁碟、網路與程序狀態等指標。

- **cAdvisor**
    - 例如 `:8080/metrics`
    - 提供 container 層級的資源使用資訊。

- **Application Pods**
    - 每個服務或應用 Pod 暴露自己的 metrics endpoint。

- **kube-state-metrics**
    - 例如 `Deployment :8080/metrics`
    - 提供 Kubernetes 物件狀態，例如 Deployment、Pod、ReplicaSet、HPA 等資源狀態。

常見設定如：

```yaml
scrape_interval: 15s
```

也就是 Prometheus 每 15 秒抓取一次資料。

---

## Pull Model：你隱性簽下的取樣契約

Prometheus 的 pull model 本質上是一種取樣機制。

例如 scrape 時間點為：

```text
T=0s → T=15s → T=30s → T=45s
```

在兩次 scrape 之間發生、又在下一次 scrape 前消失的狀態，可能對 Prometheus 來說是不可見的。

換句話說，Prometheus 看到的是固定時間間隔上的切片，而不是完整連續的事件流。

這代表使用 Prometheus 時，其實隱含接受了以下限制：

- metrics 是週期性取樣，不是事件完整記錄。
- 短暫發生又消失的異常可能被漏掉。
- scrape interval 決定了你能看到的時間解析度。
- metrics 可以告訴你某個時間點的狀態，但不一定能完整還原事件過程。

---

## Metrics 告訴你發生了什麼，但不告訴你為什麼

Metrics 很擅長描述「發生了什麼」，例如：

- `container_cpu_throttled_seconds_total` 上升
- `http_requests_total` 的 p99 latency 上升
- `node_memory_MemAvailable_bytes` 下降

但它們通常無法直接回答「為什麼發生」。

例如：

- 是否因為同一個 node 上有 noisy neighbor？
- 是否因為 node 正處於 memory pressure？
- 是否因為錯誤的 CPU limit 導致 CFS bandwidth throttle？
- 是否因為 kernel scheduler、page reclaim 或 cgroup 決策造成等待？

因此，Prometheus metrics 能指出症狀，但根因往往需要額外上下文。

### 維運重點：解 Root Cause，而不是堆 Alert

一位好的 Kubernetes 管理者，應該專注於解決根因以降低告警，而不是不斷增加新的 alert。

如果每次遇到未知問題都只是新增一條告警規則，告警系統會越來越複雜，但問題本身不一定被真正理解。

更好的方向是：

- 先確認症狀。
- 找出缺少哪些上下文。
- 補足觀測資料。
- 理解根因。
- 針對根因修復系統或調整架構。

---

## 三種角色，三種可觀察性平面

Kubernetes 環境中的可觀察性，並不是所有人都看同一組 metrics。不同角色關心的問題不同，觀測平面也不同。

### App Owner：我的服務是否健康？

App Owner 通常關心應用程式本身是否正常。

主要觀察：

- Request rate
- Error rate
- p99 latency
- DB query time
- Cache hit ratio
- Downstream service health

盲點：

- Node-level pressure 對 App Owner 來說通常不可見。
- 服務 latency 上升時，不一定能從應用 metrics 直接看出是節點資源競爭造成。

### Service SRE：我的 SLO 是否正在燃燒？

Service SRE 關心服務是否違反 SLO，以及是否需要介入。

主要觀察：

- Pod availability
- Restart rate
- CPU / memory requests vs limits
- HPA scaling
- Deployment health

盲點：

- Platform incident 可能看起來像 application failure。
- 如果缺少 node 或 kernel 層級訊號，容易誤判問題來源。

### Platform Admin：整個叢集是否健康？

Platform Admin 關心 Kubernetes cluster 與 node 的整體健康。

主要觀察：

- Node conditions
- Disk pressure
- Memory pressure
- Pod evictions
- OOM events
- Scheduler queue depth
- Bind latency

盲點：

- Service metrics 通常不會自然浮現在 platform 層級。
- 平台問題可能已經影響應用，但不一定能從服務端 metrics 直接看出。

---

## App Owner 常用視角：RED、Four Golden Signals 與 USE

不同觀測模型適合不同層級的問題。

### RED：Rate、Errors、Duration

RED 主要適合 App Owner 與 Service Engineer，尤其是 HTTP microservice。

- **Rate**
    - 每秒服務處理多少 request。

- **Errors**
    - request 失敗比例。

- **Duration**
    - 每個 request 花費的時間分布。

對 HTTP microservice 來說，RED 通常是最好的起點。

### Four Golden Signals

Four Golden Signals 來自 Google SRE 實務，常作為 SLO 的基礎錨點。

四個訊號包括：

- **Latency**
    - 處理 request 所需時間，需區分成功與失敗情境。

- **Traffic**
    - 系統需求量，例如 RPS、QPS、stream 數量。

- **Errors**
    - 明確失敗的 request 比率。

- **Saturation**
    - 系統有多滿，例如 queue depth、lag 等。

### USE：Utilization、Saturation、Errors

USE 比較適合 infrastructure context，也就是 Platform 或 Node Owner。

- **Utilization**
    - 資源忙碌時間比例，例如 CPU 使用率、disk busy percentage。

- **Saturation**
    - 額外排隊工作，例如 run queue length、memory swap。

- **Errors**
    - 錯誤事件，例如硬體故障、packet drop。

USE 是為資源設計的模型，不是為服務本身設計。

---

## Service-Level 模型的盲點

RED、Four Golden Signals 與 USE 都有價值，但單一模型無法回答所有問題。

服務層級模型常見盲點包括：

- 看得到 latency 變高，但看不到同 node 的資源競爭。
- 看得到 CPU usage 正常，但看不到 kernel scheduler waiting time。
- 看得到 memory working set 沒超標，但看不到 page reclaim pressure。
- 看得到 node aggregate metrics，但無法知道是哪個 pod 或 tenant 造成壓力。

AI 可以協助快速取得與比較資料，但真正困難的問題，往往不是「怎麼分析」，而是「應該提供哪些特定資料讓 AI 分析與比較」。

如果不知道該提供哪些 context，AI 也只能在不完整的資料上做推論。

---

## Case Study 01：etcd — Control Plane Database

### etcd 的角色

etcd 是整個 Kubernetes cluster 的 durable state machine，也就是控制平面的核心資料庫。

如果 etcd 發生問題，會連帶影響：

- API Server latency
- Leader election
- Kubernetes control plane 穩定性
- 甚至整個 cluster 的操作能力

例如：

```text
Slow fsync → leader elections → API server latency → cluster halts
```

### 你會看到的症狀

在儀表板上可能看到：

- API server latency 上升。
- etcd DB size 接近 8 GiB quota。
- leader changes 增加。
- cluster 開始不穩定。

Prometheus 可能提供的 metrics 包括：

- `etcd_db_total_size_in_bytes`
- `etcd_server_leader_changes_seen_total`
- `etcd_disk_backend_commit_duration_seconds`

---

## etcd Storage Pressure 在 Prometheus 中的樣子

當 etcd storage pressure 出現時，Prometheus 可能顯示：

- `etcd_db_total_size_in_bytes`
    - DB size 達到 8 GiB quota 的 80%。

- `etcd_server_leader_changes_seen_total`
    - leader election 頻繁發生，代表 cluster instability。

這些 alert 能告訴你「什麼東西滿了」，但不能告訴你：

- 為什麼 etcd DB 變大？
- 是哪種 Kubernetes resource 消耗最多空間？
- 是否會在下週再次發生？
- 是短期事件還是結構性問題？

---

## etcd Defrag：壓力釋放，不是根因修復

etcd defrag 可以釋放空間壓力，但它不是根因修復。

典型 runbook 可能是：

1. Alert 觸發：DB size 超過 80%。
2. 執行：
   ```bash
   etcdctl defrag --cluster
   ```
3. Alert 消失。
4. Ticket 關閉。

> `etcdctl defrag --cluster` 雖然可以釋放碎片化造成的空間壓力，但在高負載或大型叢集環境中，defrag 本身也可能影響 etcd 的回應能力，進而讓 API Server latency 上升，甚至造成控制平面操作短暫卡住。
>
> 因此，這類操作應搭配維護時機、etcd 健康檢查與風險評估，而不是在告警發生時直接執行後就關單。

但這樣仍然不知道：

- 哪一種 resource type 消耗了空間？
- 是不是某個 operator 高頻更新 object？
- 是否有大量短生命週期 pod churn？
- 下週是否會再次發生？

Defrag 比較像 pressure-release valve。它可以讓告警暫時消失，但底層問題仍可能存在。

---

## etcd：Prometheus 無法提供的上下文

Prometheus 可以提供 etcd size 類 metrics，例如：

- `etcd_db_total_size_in_bytes`
- `etcd_mvcc_db_total_size_in_bytes`
- `etcd_mvcc_db_total_size_in_use_in_bytes`

但要診斷根因，還需要：

- key-space breakdown by resource type
- API audit logs：誰寫入最多？
- compaction frequency
- revision lag
- object 更新頻率
- 特定 resource 的大小與變化趨勢

這些資訊通常不是標準 Prometheus exporter 直接能提供的。

### 可能根因

etcd storage pressure 的潛在根因包括：

- Oversized objects
    - 例如過大的 ConfigMaps、Secrets、Custom Resources。

- High-frequency object update
    - 例如 operator 高頻更新 Kubernetes object。

- Rapid pod churn
    - 例如 CronJobs、Airflow 等大量短生命週期工作。

- Unused ReplicaSet revision history
    - 歷史版本未清理，造成資源累積。

### 簡報中的測試情境

簡報中列出多個可能造成 etcd 壓力的情境：

- API Server 每 5 分鐘 compact。
- 定期 defrag。
- CronJobs 每 3 分鐘產生約 700 個 pods。
- Pod 執行 sleep 10 秒後完成。
- 10 個 1 MiB ConfigMap，每分鐘更新 label。
- 500 個 Deployment 搭配 HPA，並定期更新 HPA objects。
- 10 個 1 MiB ConfigMap，每 10 秒更新 label。

這些例子說明：etcd 壓力不只是「資料庫變大」，背後可能是物件大小、更新頻率、短生命週期資源與控制器行為共同造成。

相關參考：

- https://github.com/etcd-io/etcd/issues/19806
- https://github.com/etcd-io/bbolt/issues/694

---

## Case Study 02：Zombie Processes 作為平台健康訊號

### Zombie Process 是什麼？

Zombie process 是已經結束，但 parent process 沒有呼叫 `wait()` 回收的 defunct process。

它們雖然不再執行，但仍占用 PID slot。

如果 zombie processes 快速累積，通常代表某個 parent process 行為異常。

### 觀察訊號

可以觀察：

```promql
node_processes_state{state="Z"}
```

如果這個數值穩定上升，尤其超過 10 到 15，就可能是某個 stuck 或 crashing containerized parent 的早期警訊。

### Prometheus 的限制

node-exporter 可以統計 node 上的 zombie process 數量，例如：

```text
node_processes_state{state="Z"} = 14
instance = "node-03"
```

但它無法回答：

- 哪個 pod 正在產生 zombies？
- 哪個 container 擁有這些 processes？
- 這種情況已經持續多久？
- parent process 是誰？

也就是說，node-exporter 能提供 node-level count，但沒有 pod 或 container label。

### 需要額外工具補足上下文

要真正完成診斷，仍需要 process-level tooling，例如：

- `ps`
- `/proc`
- container runtime 層級資訊
- node 上的即時 process tree
- pod / container 對應關係

`node_processes_state{state="Z"}` 是 platform health signal，不是 app metric。

### 可能根因

簡報提到一個可能原因：

- 不適當的 probe timeout 可能導致 silent zombie processes。

例如：

1. Kubelet 執行 exec probe。
2. timeout 設定為 5 秒。
3. command 實際執行 6 秒。
4. Kubelet 在 timeout 後 kill exec。
5. 若 parent process 未正確回收，可能產生 zombie。

---

## Case Study 03：Noisy Neighbor 是結構性排程問題

### 問題設定

在多團隊共用 Kubernetes nodes 的環境中，Kubernetes 會透過 cgroup limits 做基本隔離。

但 Linux kernel 並不會預設完整隔離所有共享資源，例如：

- page cache
- IRQ
- network backplane
- disk I/O
- kernel scheduler
- memory reclaim
- 部分 cgroup throttle decision

因此，兩個看似互不相關的 workload，如果被 scheduler 排到同一個 node 上，仍可能互相影響。

### 你看到的症狀

常見情境：

- Team A 的 service latency 突然升高。
- CPU 使用率只有 35%。
- Memory 看起來正常。
- Team A 自己的 metrics 沒有明顯異常。
- 但同一個 node 上的其他 workload 正在消耗共享資源。

這就是 noisy neighbor 問題。

### 為什麼難找？

難點在於：

- node-exporter 回報的是 aggregate node metrics。
- cAdvisor 回報的是 per-container usage。
- 但兩者都不直接揭露 cross-tenant resource contention。
- service metrics 看不到 shared kernel resource contention。

Noisy neighbor 是結構性問題：scheduler 把會互相競爭的 workloads 放到了同一個 node 上。

---

## 為什麼 CPU 與 Memory 儀表板會誤導你？

典型情境如下：

```text
Dashboard green. Latency 5× above SLO.
```

儀表板可能顯示：

- `container_cpu_usage_seconds_total`：32%
- `container_memory_working_set_bytes`：420 Mi
- Pod Ready
- CPU / Memory 看起來都正常

但 p99 latency 卻是 SLO 的 5 倍。

CPU 與 memory dashboards 無法顯示的問題包括：

- Disk I/O bandwidth contention
- Memory reclaim pressure
- Kernel scheduler waiting time
- shared resource contention
- cgroup 之間的間接干擾

因此，CPU 32%、Memory 420 Mi 並不代表服務沒有受到資源競爭影響。

---

## Noisy Neighbor：超越 CPU 與 Memory 的訊號

在 node 上，可能同時存在多個 pods，共享：

- `/var`
- `/var/lib`
- container image storage
- log storage
- ephemeral storage
- disk I/O
- host kernel resource

可能觀察到的訊號包括：

- KubeNodeDiskPressure alerts
- Kubelet node disk pressure
- disk performance downgrade
- system file overconsumption
- container image and log accumulation
- application-generated ephemeral files

除此之外，也可能需要觀察：

- CPU usage
- CPU throttling
- process / thread spawn 行為
- kernel scheduler waiting time

例如：

```promql
node_schedstat_waiting_seconds_total
```

這類訊號可以協助發現 workload 雖然 CPU usage 不高，但實際上正在 scheduler queue 中等待。

### node-exporter 本身也可能被影響

在 noisy neighbor 問題中，node-exporter 本身也可能受到影響，造成：

- metrics 隨機遺失
- scrape 不穩定
- troubleshooting 更困難

更嚴重時，甚至可能影響 node 上的關鍵 control plane daemons，例如：

- Kubelet
- Kube-proxy
- CRI-O

### 可能的緩解方向

對於關鍵 workload 或 system daemons，可以考慮更強的 CPU isolation。

簡報中提到的方向包括：

- Pod 使用 Static CPU Manager
- Non-Pod 使用 `cpuset`、`isolcpus`
- cgroup v2
- boot-time 與 run-time 層級的資源隔離

相關參考：

- https://github.com/kubernetes/kubernetes/issues/87862
- https://github.com/kubernetes/kubernetes/issues/115994

---

## Service-Level 模型再次失明的地方

Containers 提供輕量級虛擬化，但並沒有提供完整的資源競爭隔離。

因此，noisy neighbor 問題可能以標準 metrics 不容易看見的方式影響其他 containers。

這再次說明：

- service metrics 可能只看到 latency 變差。
- container CPU / memory metrics 可能看起來正常。
- node-level aggregate metrics 可能不足以定位來源。
- 真正的問題可能藏在 kernel scheduling、I/O、page reclaim 或 cgroup decision 裡。

---

## Kernel Blind Spots：node-exporter 活在 User Space

node-exporter 提供大量 user-space 可觀察資料，但某些核心層級的細節仍不可見。

### User-space metrics 可以告訴你的事

例如：

- `node_cpu_seconds_total`
    - CPU 使用狀態，例如 35%。

- `node_memory_MemAvailable_bytes`
    - 可用記憶體，例如 3.2 Gi。

- `node_disk_read_bytes_total`
    - 磁碟讀取 bytes，可能看起來正常。

- `node_network_receive_bytes_total`
    - 網路接收 bytes，可能看起來正常。

### 隱藏在 kernel space 的資訊

但以下資訊常常是標準 metrics 看不到的：

- kernel scheduler internals
- IRQ / softirq per CPU distribution
- page reclaim / kswapd pressure
- cgroup hierarchy throttle decisions

這些正是許多 Kubernetes 平台疑難雜症的根因所在。

---

## Kernel Forensics：當你想看時，證據已經消失

很多 kernel 或 process 層級的鑑識資料是短暫的。

等到工程師開始排查時，證據可能已經消失。

### Stack traces / core dumps

Container runtime 預設可能丟棄 core dumps。

你可能只得到 crash timestamp，但 call stack 已經不存在。

### Node reboot

kernel panic 或 node reboot 會清掉：

- `/proc` 狀態
- in-memory counters
- 當下 process 狀態
- transient kernel evidence

Prometheus 可能只看到 counter reset，卻沒有解釋為什麼 reset。

### AI 與自學習

簡報以 AI、Node Exporter 與 Self-Learning 表達一個方向：未來 AIOps 可以協助從既有 metrics 中學習模式，但前提仍是資料與上下文足夠完整。

如果底層證據在觀測前已經消失，AI 也很難從缺失資料中還原真相。

---

## PSI：Pressure Stall Information 是更好的 Operator 維度

一個重要觀念是：

```text
Usage ≠ Pressure
```

CPU 使用率 70% 不代表 workload 沒有被餓死。

如果某個 cgroup 長時間支配 scheduler queue，其他 workload 即使在 CPU usage 看起來不滿的情況下，仍可能被迫等待。

PSI 測量的是 tasks 實際等待資源的時間，而不是資源看起來有多忙。

### 傳統 Usage Metrics

傳統 metrics 包括：

- CPU
    - 0 到 100% allocated core time。

- Memory
    - `working_set_bytes` vs limit。

- Disk
    - 每秒 read / write bytes。

問題是：70% usage 仍可能讓 workloads starve。

### PSI 能回答什麼？

PSI 衡量 tasks 等待資源而 stalled 的時間。

主要指標概念：

- **some**
    - 至少一個 task stalled，但其他 task 仍有進展。

- **full**
    - 所有 tasks 都 stalled，在該時間區間內沒有進展。

PSI 直接回答：

> 這個資源是否正在成為瓶頸？

這比單純觀察 usage 更接近 operator 真正需要的答案。

---

## Pod-Level PSI：Kubernetes 1.34 的新能力

### Kubernetes 1.34 以前：只有 Node-Level PSI

在 Kubernetes 1.34 以前：

- `/proc/pressure/` 是 per-node kernel interface。
- PSI values 聚合整個 node 上所有 cgroups。
- 無法 attribution 到特定 pod 或 namespace。
- 如果 Pod A 讓 Pod B starve，node PSI 會上升，但你不知道是誰造成的。

### Kubernetes 1.34：Pod-Level PSI Beta

Kubernetes 1.34 提供 Pod-Level PSI beta 能力，透過 feature gate：

```text
KubeletPSI
```

其能力包括：

- per-pod cgroup PSI files
- Prometheus labels 可包含 pod 與 namespace
- noisy neighbor attribution 更容易
- 可將壓力歸因到特定 pod

需求包括：

- cgroup v2
- Linux kernel 4.20+

這對 noisy neighbor 與平台層級 root cause analysis 很重要。

---

## Takeaway：Prometheus Setup 應該補上的能力

Prometheus 與 exporters 是 Kubernetes observability 的重要基礎，但不是完整答案。

從簡報案例可以歸納出幾個應補強的方向。

### 1. 不只看 What，也要設計 Why 的資料路徑

Metrics 可以回答：

- 什麼變高？
- 什麼變低？
- 哪個 alert 觸發？
- 哪個時間點異常？

但 root cause 需要：

- audit logs
- key-space breakdown
- process-level tooling
- kernel-level signals
- PSI
- scheduler waiting time
- pod / namespace attribution
- cgroup 資訊
- runtime 與 node context

### 2. 對不同角色提供不同觀測平面

App Owner、Service SRE、Platform Admin 需要的資訊不同。

好的觀測系統應該能串起：

- service-level metrics
- Kubernetes object state
- node-level metrics
- process-level evidence
- kernel pressure signals
- audit 與事件資料

### 3. 避免用更多 alert 取代理解根因

告警應該協助行動，而不是掩蓋未知。

如果一個告警只能讓人執行固定操作，但無法理解問題是否會再發生，就代表仍缺少觀測上下文。

### 4. 補足 kernel 與 cgroup 層級的盲點

Kubernetes 平台問題常常不是 app 層問題，而是：

- node pressure
- scheduler wait
- page reclaim
- disk I/O contention
- noisy neighbor
- cgroup throttle
- process leak
- zombie processes

這些都需要超越標準 exporter 的資料。

### 5. 為 AIOps 準備正確上下文

AI 可以幫助快速搜尋、整理、比較資料，但前提是你知道該給它哪些資料。

AIOps 的難點不只是模型能力，而是觀測資料是否能完整描述問題脈絡。

---

## 總結

Prometheus exporters 很適合作為 Kubernetes observability 的基礎，但它們不是萬能的。

這場分享透過 etcd、zombie processes 與 noisy neighbor 三個案例說明：

- Prometheus 常能告訴你發生了什麼。
- Prometheus 不一定能告訴你為什麼發生。
- node-exporter 與 cAdvisor 各自有觀測邊界。
- service-level metrics 可能看不到 platform-level pressure。
- kernel-space 與 process-level evidence 很容易消失。
- noisy neighbor 問題需要更細緻的 attribution。
- PSI 是比 usage 更接近資源壓力真相的維度。
- Kubernetes 1.34 的 Pod-Level PSI 有助於改善 noisy neighbor 歸因問題。

最後的核心訊息是：

> As much as the world pushes for AIOps, the most "human" way to work in this AI era is still to invest time in understanding the "why" to find a better "how".

即使世界正在走向 AIOps，在 AI 時代最「人性」也最重要的工作，仍然是投入時間理解問題背後的「為什麼」，才能找到更好的「怎麼做」。
