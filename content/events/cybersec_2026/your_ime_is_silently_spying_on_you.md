# 你的輸入法正在背後偷偷監視你

> Nick Dai／趨勢科技 TrendAI 資深威脅研究員

> 2026-05-07 15:30 - 16:00 @ 7F 703, 南港展覽館 2 館

> [議程](https://cybersec.ithome.com.tw/2026/session/4376)

> 主題：TAOTH 攻擊活動、搜狗輸入法注音版更新伺服器遭接管、供應鏈攻擊與威脅狩獵流程

---

## 議程核心摘要

2025 年，趨勢科技在一起不尋常的 IR 事件中，發現受害主機的 `%temp%` 目錄被植入多個可疑檔案，例如 `sihost.exe`、`GTelemetry.exe`、`code.exe`，並伴隨 `SogouZhuyin.ini`、`SkinRecommend.ini` 等看似正常的更新設定檔。進一步調查後，確認這是名為 **TAOTH** 的攻擊活動：攻擊者濫用已被棄置的搜狗輸入法注音版網域與更新機制，透過合法輸入法的自動更新流程散布惡意程式。

---

## 1. 初始線索：不尋常的 IR 事件

調查起點是一批落在 `%temp%` 下的可疑二進位檔與設定檔：

- `sihost.exe`
- `sihost[1].exe`
- `GTelemetry.exe`
- `code.exe`
- `SogouZhuyin.ini`
- `SkinRecommend.ini`
- `urlguide50_20170927_201709271827.enc`
- `z.txt`

`SogouZhuyin.ini` 與 `SkinRecommend.ini` 看起來是格式完整的搜狗注音輸入法更新設定檔，內容包含：

- 版本資訊
- 更新 URL
- 檔案大小
- MD5
- 下載位置

其中更新 URL 指向：

- `srv-pc[.]sogouzhuyin[.]com/sihost.exe`
- 後續也出現 `sihostt.exe`

---

## 2. Log 分析：合法更新流程下載惡意程式

Detection logs 顯示，`.ini` 設定檔與執行檔之間存在關聯，並可看到：

- `sihost.exe`
- `GTelemetry.exe`
- `SogouZhuyin.ini`
- `SkinRecommend.ini`

XDR logs 進一步指出執行鏈：

```text
ZhuyinUp.exe → SGDownload.exe → sihost.exe → 可疑行為
```

其中 `ZhuyinUp.exe` 是搜狗注音輸入法的官方更新程式，具有有效簽章，但憑證狀態顯示為 revoked。也就是說，惡意程式不是一開始就來自偽造程式，而是透過原本的更新流程被帶入。

---

## 3. Post-exploitation：攻擊者進入後的行為

攻擊者成功植入後，會進行環境偵察與後續控制，例如執行：

- `tasklist.exe /svc`
- `quser.exe`
- `ipconfig.exe /all`
- `net.exe time /domain`
- `net.exe user`
- 查詢本機環境與使用者資訊

簡報中也指出攻擊者使用 **VS Code tunnel** 建立通道：

```text
code.exe tunnel
```

也就是濫用合法工具作為遠端存取與持久化通道，使惡意行為更不容易被單純以檔名或工具名稱判斷出來。

---

## 4. 多重受害者與植入惡意程式

進一步 threat hunting 發現，更多受害者的環境中也出現由 `ZhuyinUp.exe` 或 `SGDownload.exe` 落地的可疑檔案。

簡報列出的惡意程式家族包括：

- **TOSHIS**
- **DESFY**
- **GTELAM**
- **C6DOOR**

部分 C&C 或連線目標包含：

- `45[.]32[.]117[.]177`
- `154[.]90[.]62[.]210`
- `38[.]60[.]203[.]134`
- `192[.]124[.]176[.]51`
- `64[.]176[.]50[.]181`

這些證據顯示，事件並非單一主機感染，而是可擴展為一波供應鏈攻擊活動。

---

## 5. 假設 A：DNS Hijacking

研究人員一開始提出假設：是否為受害者環境的 DNS 被劫持，導致合法更新網域解析到惡意內容？

支持此假設的初步線索包括：

- `srv-pc[.]sogouzhuyin[.]com` 在 VirusTotal 上看似合法
- Wikipedia 上 `dl[.]sogouzhuyin[.]com` 也曾被列為官方下載網站
- `ZhuyinUp.exe` 內嵌的更新伺服器正是 `srv-pc[.]sogouzhuyin[.]com`
- 更新流程會：
    1. 連線下載 INI 設定檔
    2. 根據設定檔中的 `url` 下載更新安裝檔
    3. 依 `md5` 驗證檔案
    4. 執行更新安裝檔

但後續 telemetry 顯示受害者數量達數百，分布廣泛，若僅是單一環境 DNS 劫持，解釋力不足，因此需要重新評估。

---

## 6. 假設 B：Patched Updater

第二個假設是：`ZhuyinUp.exe` 是否被 patch 或遭竄改？

研究人員追蹤 VirusTotal 關聯與歷史樣本後發現：

- `ZhuyinUp.exe` 可追溯至 2017 年
- 安裝檔 `sogou_zhuyin_10n.exe` 曾於 2018 年由 `dl[.]sogouzhuyin[.]com` 散布
- 官方 Facebook 貼文也曾使用 `sogouzhuyin[.]com` 相關網域
- 憑證歷史顯示，該網域過去確實曾是合法基礎設施

但 crt.sh 資料顯示，`sogouzhuyin[.]com` 的憑證 CN 在 2020 年以前曾與 Amazon 相關，2024 年 10 月後轉為 Let’s Encrypt E6。這表示該網域基礎設施狀態發生變化，並不完全支持「更新程式本身遭 patch」的假設。

---

## 7. 假設 C：Domain Takeover

最終確認的方向是 **Domain Takeover**。

研究人員持續監控更新伺服器後，取得可存取的更新 URL，並確認其中的更新檔實際上是 **DESFY malware**。此外，簡報指出：

- 有人於 2025 年 3 月刻意修改 Wikipedia 頁面
- 將下載連結導向攻擊者可利用的安裝檔位置
- 棄置的搜狗注音相關網域被重新利用
- 合法輸入法安裝器本身無惡意，但自動更新流程會導向惡意部署

因此結論是：攻擊者濫用已棄置的搜狗注音網域與更新機制，形成供應鏈攻擊。

---

## 8. 攻擊鏈整理

簡報的 Infection Chain 顯示整體流程大致如下：

1. 受害者安裝搜狗注音輸入法
2. 輸入法透過合法更新程式連線到更新伺服器
3. 攻擊者控制或濫用已棄置的更新基礎設施
4. 更新設定檔導向惡意執行檔
5. 惡意程式落地並執行，包括：
    - TOSHIS
    - DESFY
    - GTELAM
    - C6DOOR
6. 後續行為包括：
    - 連線 C&C
    - 載入 shellcode
    - 部署 COBEACON / Merlin Agent
    - 蒐集檔名
    - 上傳至 Google Drive
    - 建立 VS Code tunnel

---

## 9. 受害者分析

受害者主要是全球各地的繁體中文使用者。簡報中的地理分布如下：

- 台灣：49%
- 柬埔寨：11%
- 美國：7%
- 香港：7%
- 日本：4%
- 泰國：4%
- 寮國：3%
- 其他：13%

GTELAM 會彙整受害主機上的文件檔名，形成獨立檔案後上傳到 Google Drive。檔名格式可包含：

- domain
- user
- host
- interface
- IP

這讓研究人員可以利用 GeoIP 與檔名內容進一步分析受害者輪廓。

---

## 10. 受害者輪廓

簡報指出，多數受害者是繁體中文使用者，並從檔名與使用情境歸納出幾類目標：

- 台商
- 博弈產業
- 詐騙相關使用者

例如簡報中出現的檔名包含：

- 博弈培訓簡報
- 體育投注紀錄
- 日本線上博彩調研
- Telegram Desktop 中與詐騙話術、投資包裝、開戶引導相關的文件

這顯示攻擊活動並非單純大規模灑網，而可能特別關注東亞與繁體中文圈中的高價值使用者。

---

## 11. 結論與啟示

本議程最終結論：

- 攻擊者濫用已棄置的搜狗注音網域部署惡意程式
- 搜狗注音輸入法安裝器本身不一定有惡意
- 真正造成感染的是自動更新流程導向遭濫用的更新基礎設施
- 多數受害者是全球各地的繁體中文使用者
- 單一 alert 若結合 XDR logs 與 OSINT，可擴展成完整攻擊活動分析
- 攻擊者的基礎設施本身，也可以成為調查線索
- 威脅狩獵應採用假設驅動方式，透過證據逐步驗證或排除
- 攻擊活動會持續演進，包括工具、基礎設施與投遞方式

## Takeaways

1. **供應鏈攻擊不一定來自惡意安裝器，也可能來自被棄置或接管的更新基礎設施。**
2. **合法簽章與正常檔名不足以保證安全，仍需檢查實際行為與下載來源。**
3. **XDR telemetry 結合 OSINT，可從單一事件擴展到完整 campaign 分析。**
4. **輸入法、更新器、遠端開發工具等常見軟體，都可能被攻擊者濫用成隱蔽入口。**
