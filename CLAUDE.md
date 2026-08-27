# Aiello SiteChat Settings（Chat Theme & Colors + Greeting + FAQ Cards，飯店端填寫表單）

單一 `index.html`（無框架、無 build step，`<script type="module">` 因為要 `import` Supabase JS）+
Tailwind Play CDN + Supabase。單人開發：Jim。GitHub：`aiellojim/SiteChat-Setting-Form`。
本機：`/Users/jim.chao/SiteChat settings`。部署：Vercel，正式網址
`https://sitechat-settings.aiello.dev/`（原本是 Vercel 預設網域 `site-chat-setting-form.vercel.app`，
2026-08-14 換成自訂網域；舊網址若還沒解除綁定，理論上仍可訪問，但不要再新產生指向舊網址的連結）。

## 動手前先讀

**這是 Aiello 表單家族的一員，header/sidebar、語言切換、banner 行為、自動存檔、Supabase 慣例等
共用規格寫在 `/Users/jim.chao/hotel-dashboard/docs/aiello-forms-spec.md` —— 動這些邏輯前先讀那份
文件，不要在這裡重新設計或憑印象照舊做法延續。這個表單目前是規格裡兩種語言切換實作方式中「原地
重繪、不整頁重載」那一派的範本（沒有嵌入即時載入語言的第三方元件，見規格第 2 節）。** 這份
CLAUDE.md 只放這個表單自己的專屬資訊。

其他待辦看 `/Users/jim.chao/hotel-dashboard/docs/todo.md`（待辦 #6 的 RLS 安全性缺口清單已包含這個
表單的兩張表）。

## 本表單專屬資訊

- 連結格式：`?p=<project_id>`（主要慣例，比照其他表單），另外自己也支援 `/form/<project_id>` 路徑
  （`vercel.json` rewrite 到 `/index.html`），兩種都會被 `getProjectId()` 認得。
- `form-submit-notify` Edge Function 的 `source` 標籤：`sitechat_settings`。
- 資料表：`sitechat_settings`（singleton，PK 是 `project_id`，存 `bot_name`／`bot_icon_url`／
  `theme` jsonb／`greeting` jsonb／`form_submitted_at`）、`sitechat_faq_cards`（可重複列表，
  `project_id`＋`sort_order`，存 `titles`／`questions` 兩個 jsonb）。
- `theme` jsonb 是純平鋪物件，key 就是 CSS 變數名稱（含開頭 `--`），value 幾乎都是 hex 色碼，只有
  `--welcome-bg` 是完整的 `linear-gradient(...)` 字串——這個形狀是刻意設計成可以直接匯入/匯出整包給
  外部系統對接用，改動前先確認不會破壞這個「key = CSS 變數名稱」的直接對應關係。
- **Quick Colors**（3 個給客戶用的主控色，一次改一個會連動更新一批底層 CSS 變數）跟底下完整的
  「Chat Theme & Colors」granular 卡片是分開的兩層——granular 卡片目前用 `SHOW_GRANULAR_COLORS = false`
  在前端隱藏（結構/資料/render function 都還在，之後要重新開放隨時可以，不要刪除相關程式碼）。
- FAQ 卡片、Greeting 的「內容語言」（en/zh/ja，填寫的實際內容）跟 header 的「UI 語言」是兩個獨立
  狀態（`currentLang` vs `uiLang`），不要混用同一個變數——這正是規格第 2 節特別強調要分開的原因，
  這個表單就是踩過這個坑之後才確立那條規則的。
- 手機預覽畫面固定顯示英文，不受 UI 語言切換影響（Jim 的明確決定：預覽是給填表人看效果，不需要
  跟著介面語言翻譯）。
- 2026-08-27 新增「KMS Permissions」分頁，讀寫既有的 `web_portal_users`（跟 AVA basic settings／
  ACA basic settings 完全共用同一張表），沒有新增資料表。顯示與編輯歸屬邏輯（`showKms()`／
  `kmsEditable()`／`kmsOwnerFormName()`，定義在 `BRAND` 常數之後）是全域優先序
  AVA/GW（AVA basic settings）> ACA basic settings > SiteChat Settings 在這份表單這端的實作：
  `showKms()` 只認專案是否明確掛了 `"KMS"` 標籤（跟 AVA/GW/ACA 無關），為 false 時整個分頁連同
  側邊欄項目一起隱藏；`showKms()` 為 true 之後，`kmsEditable()` 才決定「輪到誰」——SiteChat 排在
  優先序最後一位，所以要同時檢查 AVA、GW、ACA 三個標籤都不存在才可編輯，跟 ACA 只需要檢查
  AVA/GW 兩個不同（ACA 優先序比 SiteChat 高，不需要知道 SiteChat 存不存在）。唯讀鏡像時的提示
  文字會用 `kmsOwnerFormName()` 動態代入實際的主編輯表單名稱（AVA Basic Settings 或 ACA Basic
  Settings），不是寫死其中一個。**安全要求（Jim，2026-08-27 明確交代）**：唯讀狀態下不能有任何
  寫入 `web_portal_users` 的路徑——`renderKms()` 在不可編輯時整段不渲染 input/select 的事件綁定、
  也不渲染新增/刪除按鈕；`onKmsFieldInput`/`addKmsUser`/`removeKmsUser` 各自再檢查一次
  `kmsEditable()`；`saveToSupabase()` 送出前又再檢查一次 `kmsEditable()` 才會組出要送到 Supabase
  的 diff——三層防護，真正擋下寫入的是最後 `saveToSupabase()` 那一層（前兩層是 UI 層面，理論上
  防止不了刻意繞過的操作，但正常操作流程下不會走到）。

## 硬規則

1. 大改動（≥3 檔案、動 schema，或 Jim 明說是大改動）前先 `git tag` 打還原點。
2. 禁止把 secret（service_role key 等）寫進任何會 commit 的檔案；Supabase anon/publishable key 是
   公開金鑰，可以照現況寫在前端原始碼裡，不算 secret。
3. 完成的定義：`node --check` 語法過 **且** 實際頁面確認，「程式碼看起來對」不算完成——驗收流程見
   `aiello-forms-spec.md` 第 5 節。
4. 每次改完照 `aiello-forms-spec.md` 第 5 節跑完驗收流程，才能算改動結束。

## 維護協議

- 模型可自行更新這份檔案的「本表單專屬資訊」章節（例如新增資料表、新增 `source` 標籤）。
- 共用規格（`aiello-forms-spec.md`）的改動邏輯：先跟 Jim 確認，除非是修正明顯過時的既有描述。
