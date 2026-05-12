# Copilot Instructions

## 建置與驗證指令

此倉庫已整理成標準 Hugo 結構：站台內容放在 `content/`，靜態資源放在 `static/`，版型放在 `layouts/`。站名目前是 **Tech Events Notes**，在倉庫根目錄直接執行 `hugo` 就是完整建置驗證。

### 全站建置

```bash
hugo --minify
```

### 本地預覽

```bash
hugo server -D
```

### 單頁聚焦檢查

目前沒有既有的自動化測試框架；最接近「單一測試」的做法，是直接建置一次，再檢查目標頁面是否有產出對應 HTML：

```bash
hugo --minify && \
test -f public/events/cybersec_2026/beyond_prompt_attacks/index.html
```

### 測試與 lint

目前倉庫內沒有既有的測試或 lint 指令；內容與版型變更通常以 Hugo 建置成功作為主要驗證方式。

## 高階架構

- 內容來源分成四層：`content/_index.md` 是首頁、`content/events/_index.md` 是活動列表、`content/events/<活動>/_index.md` 是單一活動首頁、`content/events/<活動>/*.md` 是各議程／講題筆記。站台層級靜態資源放在 `static/` 根目錄（例如 `static/favicon.svg`），講題附檔（例如 PDF）則對應放在 `static/events/<活動>/`。
- GitHub Pages workflow 直接在倉庫根目錄建置 Hugo 站台，並在 workflow 內安裝最新版 Hugo Extended；實際建置指令是 `hugo --destination /tmp/public --minify --baseURL "${{ steps.pages.outputs.base_url }}/"`，因此 `content/`、`static/`、`layouts/` 的相對位置必須維持 Hugo 標準慣例。
- `layouts/_default/baseof.html` 提供全站共用外框與樣式，並負責頁面 title 組裝、載入 `favicon.svg`、設定 `theme-color`，以及透過 CDN 載入 MathJax v3；`layouts/_default/list.html` 負責首頁與 section 頁，`layouts/_default/single.html` 負責單篇講題頁，`layouts/index.html` 讓首頁沿用相同版型。共用行為拆在 partials：`page-toc.html` 顯示目錄、`page-back-link.html` 產生回上一層連結、`page-title-text.html` 統一標題解析邏輯。
- `layouts/_default/_markup/render-link.html` 是關鍵：它會把 Markdown 內連到其他 `.md` 的相對連結轉成 Hugo `RelRef`，並把 `./slides.pdf` 或 `../static/...` 這類相對附檔路徑轉成站內 URL。**來源檔請保留相對路徑寫法**，不要改成產出後的 URL。

## 關鍵慣例

- 活動與清單頁幾乎都用 `_index.md` 搭配 TOML front matter，並設定 `title` 與 `showList = false`；清單內容不是依賴 Hugo 自動列子頁，而是直接在 Markdown 內手動維護議程連結順序。
- 首頁 `content/_index.md` 目前只提供站台簡介與通往活動列表頁的入口，不直接列出各活動或各議程。
- 單篇講題頁通常不寫 front matter，直接用第一個 `# ` 標題當頁面標題；`page-title-text.html` 的標題優先序是：front matter `title` → 第一個 H1 → humanized 檔名。新增內容時，若不寫 front matter，至少要有明確的 H1。
- 目錄區塊不是手動寫的；只要頁面有 Hugo 解析出的 headings，就會由 `page-toc.html` 自動顯示目錄。寬螢幕時 TOC 會浮在主內容左側；長文筆記應維持穩定的 `##` / `###` 層級，避免破壞 TOC 與閱讀體驗。
- 返回連結不是手動寫的；非首頁頁面若有上層頁面，`page-back-link.html` 會在頁面底部自動顯示「回上一頁」。
- 全站語系與內容以繁體中文（台灣）為主，`hugo.toml` 設定 `locale = "zh-tw"`，版型根元素使用 `lang="zh-Hant"`。新增頁面、導覽文字與 commit message 都應維持同樣語系。
- `hugo.toml` 已開啟 `markup.goldmark.renderer.unsafe = true`、Goldmark passthrough 與語法高亮設定；其中數學公式分隔符為行內 `$...$`、`\(...\)` 與區塊 `$$...$$`、`\[...\]`。若調整這些設定，要先確認既有 HTML、數學公式與程式碼區塊不會因此失效。
- 若需要提交 commit，請遵循 `.github/git-commit-instructions.md`：commit message 使用繁體中文（台灣）的 Conventional Commits；變更較大時，在標題下補充條列摘要。

## 部署備註

- GitHub Pages workflow 會先用 `actions/configure-pages@v6` 產生部署用 `baseURL`，再把 Hugo 輸出寫到 `/tmp/public`，最後上傳 Pages artifact。
- `baseof.html` 內的 MathJax 設定已排除 `pre`、`code` 等標籤，程式碼區塊中的 `$` 不會被誤判為公式。
