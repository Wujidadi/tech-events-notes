# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概觀

以 Hugo 建置的技術會議／活動筆記站，部署到 GitHub Pages（`.github/workflows/pages.yml`，push 到 `main` 即觸發建置與發佈）。

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

四層內容階層（皆放在 `content/` 下）：

1. `content/_index.md`：站台首頁
2. `content/events/_index.md`：活動列表
3. `content/events/<活動>/_index.md`：單一活動首頁（含議程列表）
4. `content/events/<活動>/<議程>.md`：各議程／講題筆記

對應的講題附件（PDF 等）放在 `static/events/<活動>/`，部署後對外路徑為 `/events/<活動>/<檔名>`。

## 版型結構

- `layouts/_default/baseof.html`：全站共用外框、內嵌全站 CSS（GitHub 風暗色主題）、頁尾版權聲明
- `layouts/_default/list.html`：list 頁版型（活動列表、單一活動首頁）
- `layouts/_default/single.html`：單篇講題頁版型
- `layouts/index.html`：首頁，沿用相同版型
- `layouts/partials/page-toc.html`：頁面有 headings 時自動產生目錄側欄
- `layouts/partials/page-back-link.html`：產生「回上一頁」連結
- `layouts/partials/page-title-text.html`：統一標題解析邏輯

### 關鍵：`layouts/_default/_markup/render-link.html`

這個 render hook 是內容寫作的基石，行為如下：

- Markdown 內連到其他 `.md` 的相對連結，會被轉成 Hugo `RelRef`（站內頁面導覽）
- `./slides.pdf` 這類相對附檔路徑，會被轉成站內 URL（指向 `static/` 內的對應檔案）
- 外部連結（`http://`、`https://`）與站內非 HTML 附件（如 PDF）會自動加上 `target="_blank"` 與 `rel="noopener noreferrer"`
- 站內 `.md` 頁面連結則維持在當前分頁開啟

**寫作時要保留相對路徑寫法**（例如 `./other_session.md`、`./slides.pdf`），不要改成建置後的 URL。

## 內容寫作慣例

- **活動／清單頁**用 `_index.md` 搭配 TOML front matter，並設定 `title` 與 `showList = false`；清單內容**不依賴 Hugo 自動列子頁**，而是直接在 Markdown 內**手動維護議程連結順序**。
- **單篇講題頁**通常**不寫 front matter**，直接用第一個 `# ` 標題當頁面標題。`page-title-text.html` 的標題解析優先序：front matter `title` → 第一個 H1 → humanized 檔名。新增內容時若不寫 front matter，至少要有明確的 H1。
- **目錄區塊（TOC）自動產生**：只要頁面有 Hugo 解析出的 headings 即會由 `page-toc.html` 自動顯示。長文筆記應維持穩定的 `##` / `###` 層級，避免破壞 TOC 與閱讀體驗。
- 全站語系與內容以**繁體中文（台灣）**為主（`hugo.toml` 設定 `locale = "zh-tw"`、版型根元素 `lang="zh-Hant"`）。新增頁面、導覽文字、commit message 都應維持相同語系。
- `hugo.toml` 已開啟 `markup.goldmark.renderer.unsafe = true`，代表 Markdown 中可用原生 HTML。調整這個設定前要先確認既有內容不會失效。

## Git Commit

- 一律使用**繁體中文（台灣）**並採台灣標準翻譯與慣用術語，不夾雜日語、韓語或其他非中文詞彙。
- 採 [Conventional Commits](https://www.conventionalcommits.org/zh-hant/) 標準格式。
- 變動較大時，於標題之外條列說明本次異動摘要與各檔案的異動原因。
- 詳見 `.github/git-commit-instructions.md`。
