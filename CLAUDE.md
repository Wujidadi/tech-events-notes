# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概觀

以 Hugo 建置的技術活動筆記站 **Tech Events Notes**，部署到 GitHub Pages（`.github/workflows/pages.yml`，push 到 `main` 即觸發建置與發佈）。

## 常用指令

```bash
# 全站建置（主要驗證方式）
hugo --minify

# 本地預覽（含草稿）
hugo server -D

# 「單頁」聚焦檢查：建置後驗證目標頁面有產出對應 HTML
hugo --minify && test -f public/events/<活動 slug>/<議程 slug>/index.html
```

倉庫沒有自動化測試或 lint；以 `hugo` 建置成功作為主要驗證。

## 內容架構

目前已回到標準 Hugo 結構，主要分成兩塊：

- `content/`：站台內容（首頁、活動列表、各活動頁、各議程筆記）
- `static/`：靜態檔案，包含站台層級資源（如 `static/favicon.svg`）與講題附件（如 `static/events/<活動 slug>/*.pdf`）

內容階層如下（皆放在 `content/` 下）：

1. `content/_index.md`：站台首頁
2. `content/events/_index.md`：活動列表
3. `content/events/<活動>/_index.md`：單一活動首頁（含議程列表）
4. `content/events/<活動>/<議程>.md`：各議程／講題筆記

對應的講題附件（PDF 等）放在 `static/events/<活動>/`，部署後對外路徑為 `/events/<活動>/<檔名>`。

## 版型結構

- `layouts/_default/baseof.html`：全站共用外框、內嵌全站 CSS（GitHub 風暗色主題）、頁面 title 組裝邏輯，並於 `<head>` 載入 `static/favicon.svg`、設定 `theme-color`，以及以 CDN 載入 MathJax v3（`https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-chtml.js`）
- `layouts/_default/list.html`：list 頁版型（活動列表、單一活動首頁）
- `layouts/_default/single.html`：單篇講題頁版型
- `layouts/index.html`：首頁，沿用相同版型
- `layouts/partials/page-toc.html`：頁面有 headings 時自動產生 TOC
- `layouts/partials/page-back-link.html`：在頁面底部產生「回上一頁」連結
- `layouts/partials/page-title-text.html`：統一標題解析邏輯（用於 HTML title 與清單文字）

### 關鍵：`layouts/_default/_markup/render-link.html`

這個 render hook 是內容寫作的基石，行為如下：

- Markdown 內連到其他 `.md` 的相對連結，會被轉成 Hugo `RelRef`（站內頁面導覽）
- `./slides.pdf` 這類相對附檔路徑，會被轉成站內 URL（指向 `static/` 內的對應檔案）
- `../static/...` 這類明確指向 `static/` 的相對路徑，也會被轉成站內 URL
- 外部連結（`http://`、`https://`）與站內非 HTML 附件（如 PDF）會自動加上 `target="_blank"` 與 `rel="noopener noreferrer"`
- 站內 `.md` 頁面連結則維持在當前分頁開啟

**寫作時要保留相對路徑寫法**（例如 `./other_session.md`、`./slides.pdf`），不要改成建置後的 URL。

## 內容寫作慣例

- **首頁** `content/_index.md` 只放站台簡介與通往活動列表頁的入口，**不直接列出**各活動或各議程，避免與活動列表頁職責重疊。
- **活動／清單頁**用 `_index.md` 搭配 TOML front matter，並設定 `title` 與 `showList = false`；清單內容**不依賴 Hugo 自動列子頁**，而是直接在 Markdown 內**手動維護議程連結順序**。
- **單篇講題頁**通常**不寫 front matter**，直接用第一個 `# ` 標題當頁面標題。`page-title-text.html` 的標題解析優先序：front matter `title` → 第一個 H1 → humanized 檔名。新增內容時若不寫 front matter，至少要有明確的 H1。
- **目錄區塊（TOC）自動產生**：只要頁面有 Hugo 解析出的 headings 即會由 `page-toc.html` 自動顯示。寬螢幕時 TOC 會浮在主內容左側；長文筆記應維持穩定的 `##` / `###` 層級，避免破壞 TOC 與閱讀體驗。
- **返回連結自動產生**：非首頁頁面若有上層頁面，會由 `page-back-link.html` 在頁面底部自動顯示「回上一頁」。
- 全站語系與內容以**繁體中文（台灣）**為主（`hugo.toml` 設定 `locale = "zh-tw"`、版型根元素 `lang="zh-Hant"`）。新增頁面、導覽文字、commit message 都應維持相同語系。
- `hugo.toml` 目前站名為 **Tech Events Notes**，並已開啟 `markup.goldmark.renderer.unsafe = true` 與 Goldmark passthrough。調整這些設定前要先確認既有 HTML 與數學公式寫法不會失效。
- **數學公式**透過 MathJax v3 渲染（`baseof.html` 從 CDN 載入），可用的分隔符為：
  - 行內：`$...$` 或 `\(...\)`
  - 區塊：`$$...$$` 或 `\[...\]`
  - `MathJax.options.skipHtmlTags` 已排除 `pre`、`code` 等標籤，程式碼區塊中的 `$` 不會被誤判為公式。

## 部署備註

- GitHub Pages workflow 現在直接在 repo 根目錄執行 `hugo --destination /tmp/public --minify --baseURL "${{ steps.pages.outputs.base_url }}/"`，不再建立暫存站台或搬移首頁檔案。
- Workflow 會自行安裝最新版 Hugo Extended，再執行建置與 Pages 部署。

## Git Commit

- 一律使用**繁體中文（台灣）**並採台灣標準翻譯與慣用術語，不夾雜日語、韓語或其他非中文詞彙。
- 採 [Conventional Commits](https://www.conventionalcommits.org/zh-hant/) 標準格式。
- 變動較大時，於標題之外條列說明本次異動摘要與各檔案的異動原因。
- 詳見 `.github/git-commit-instructions.md`。
