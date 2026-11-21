# 我的日曆（my-calendar）

個人日曆 PWA，主要在 iPhone 主畫面使用。單一檔案 `index.html`（原生 JavaScript，無建置步驟、無外部套件），託管於 GitHub Pages：https://bonho513.github.io/my-calendar/

## 部署

- 推送到 `main` 即自動部署（GitHub Pages，根目錄）。
- **每次修改都要更新 `APP_VERSION`**（格式 `YYYY.MM.DD-n`，例如 `2026.11.20-3`）。App 會定時讀取網頁上的版本號，不同即自動重新載入；不更新版本號，使用者的 iPhone 不會取得新版。
- Google Apps Script 伺服器**不會**隨推送更新：伺服器原始碼在 `index.html` 的 `<script type="text/plain" id="tplServer">` 範本內（`__KEY__` 為佔位符）。修改後須由使用者在 Apps Script 專案「我的日曆同步」貼上並「管理部署作業 → 編輯 → 新版本」。

## 機密資料（絕對不可寫入此 repo）

此 repo 為公開。DeepSeek API Key、同步密碼、Apps Script 網址只存在使用者裝置的 `localStorage`（於 App 設定內輸入）。不要把任何金鑰寫入程式碼、範本、測試或此檔案。

## 架構

- 狀態：`events`（陣列，`localStorage` 鍵 `cal.events`）與 `settings`（`cal.settings`）。`persist()` 儲存事項，`saveSettings()` 儲存設定，`scheduleSync()` 排程同步。
- `render()` 依 `view` 重繪；今日頁由 `renderTodo()` 組成（進度卡 → 健身卡 → 清單卡 → 逾期 → 今日 / 明日 / 後續待辦）；`renderChat()` 繪畫 AI 頁。
- 記事及優惠券：今日頁 `noteCard()`、底部表單 `#ovNotes`（列表、搜尋、篩選）及 `#ovNote`（編輯）。點擊由 document 層的 `data-nt*` 處理器統一處理。
- 健身日曆：底部表單 `#ovFit`，`openFit()` / `renderFit()`，由今日頁健身卡及概覽「運動習慣」的 `data-fitcal` 按鈕開啟。是否算作健身由 `isWorkout(e)` 判斷（運動分類，並且有打卡、已完成或有動作紀錄）；顏色深淺按組數或時數。
- 點擊以主委派處理器（`data-*` 屬性）分派；底部表單為 `.overlay > .sheet`，以 `openOv('#id')` / `closeOv(el)` 開關。
- 新功能慣例：CSS 區塊加在 `</style>` 前；HTML 表單加在 `<!-- Code viewer -->` 前；JS 區塊加在檔尾 `autosize();\nrender();` 前，並以 `/* ---------- 名稱 ---------- */` 標示。

### 事項資料（`events` 內）

一般事項：`{id, title, date:'YYYY-MM-DD', allDay, start, end, location, notes, color, done, updatedAt, deleted?}`。`live()` 只回傳未刪除且非設定紀錄的事項。

附加欄位：
- 健身紀錄：`wk:[{n, s:[{w, r, t}]}]`、`wkAt`；地點打卡：`geo`、`geoOut`、`geoSeen`、`wkLive`。
- Google 日曆：`gcId`、`gcKey`、`gcAt`、`gcPushed`、`gcRec`（重複事項，App 內唯讀）、`src:'gcal'`。
- 清單連結：`li`（清單事項 id）；由清單產生的日曆事項 id 為 `'ev-' + 事項 id`。

設定紀錄（有 `cfg` 欄位，會經雲端同步，但不顯示於日曆，亦不推送到 Google）：
- `cfg-holidays`：`{cfg:'holidays', list:[{id, name, date, end?, yearly, type:'ph'|'comp'|'own'}]}`
- `cfg-gcal`：`{cfg:'gcal', on, tz}`
- 清單：`{id:'ls-…', cfg:'list', name, order, updatedAt}`
- 清單事項：`{id:'li-…', cfg:'li', list, text, cat, catSrc, done, doneAt, due, note, order, updatedAt}`（用 `due` 而非 `date`，小工具會忽略）。`cat` 是 `CATS` 的索引（0–6），`catSrc` 為 `'auto'`（關鍵字判斷）、`'ai'`（AI 判斷）或 `'user'`（使用者指定，永不自動覆蓋）。`listReconcile()` 在每次 `render()` 開頭，按 `updatedAt` 較新者雙向同步清單事項與日曆事項（包括分類顏色；日曆事項的 `licat` 記錄顏色來自哪個分類）。

### 自動分類

- `classify(text)` 以關鍵字規則（優先次序：截止 → 健康 → 運動 → 學習 → 工作 → 生活）即時判斷，找不到回傳 -1。
- `aiClassify()` 在背景以 AI 修正：清單事項中 `catSrc` 不是 `'ai'`/`'user'` 的，以及帶 `cq:1`（新增時關鍵字判斷不到）的日曆事項。AI 回答「其他」時不會覆蓋關鍵字已判斷的結果。
- 新增行程的編輯器在使用者未手動選分類前，會隨輸入自動選分類（`catAuto`）。

### AI 助手

- `chatPrompt()` 提供行程及清單內容；AI 回傳 `{reply, actions}`，動作有 `add`、`update`、`delete`、`todo`（加入清單）、`todo_done`（完成清單事項）。`applyActions()` 執行並回傳結果，`undo()` 可撤銷。
- 介面：`aiHead()`（連接狀態）、`aiWelcome()`（今日概況及建議）、`resCard()`（結果卡片）。

### 記事及優惠券

- 記事紀錄：`{id:'nt-…', cfg:'note', kind:'coupon'|'note', title, value, code, exp, body, pin, cal, used, usedAt, created, updatedAt}`（用 `exp` 而非 `date`）。
- `cal` 為真且有 `exp` 的優惠券，會由 `ntReconcile()` 建立全日事項 `'ev-' + 記事 id`（欄位 `nt`，生活分類），完成狀態與 `used` 雙向同步；這類事項不會列入逾期。
- 「貼上文字自動填寫」先用 AI（`ntParseAI`），失敗時用 `ntParseLocal()` 的規則。AI 對話動作另有 `note` 及 `note_used`。

刪除一律用墓碑（`deleted:true` 並更新 `updatedAt`），不可直接從陣列移除，否則同步會把它帶回來。

### 同步

- Google Drive：App 以 `text/plain` POST `{key, events, calForce}` 到 Apps Script；伺服器以 `updatedAt` 較新者合併（墓碑 90 日後清除），寫入 Drive 的 `my-calendar-data.json`，回傳 `{ok, events, more, calErr}`。
- Google 日曆（伺服器 v3）：推送 App 事項到預設日曆，每 90 秒（或手動同步時）拉取 −30 至 +365 日的事項。
- 另有舊的 GitHub Gist 同步模式。

## 介面與文字慣例

- 正式書面繁體中文；App 介面不用 emoji。
- 深藍主題，常用 CSS 變數：`--ink`、`--muted`、`--faint`、`--line`、`--cyan`、`--ok`、`--danger`、`--num`；週末 `--sat`、`--sun`。
- 區塊樣式：`.glass.sect` 卡片、`.sh` 標題列（`b`、`.cnt`、`.sd`、`.r` 按鈕）、`.chk` 圓形勾選（顏色用 `--c`）、`.sempty` 空白提示、`.donefold` 已完成摺疊、`.more-btn`。
- 類別顏色（`CATS`）：工作 #4F7BFF、截止 #FFC94D、生活 #FF6F9C、運動 #3FE0A5、學習 #A98BFF、健康 #3FD3F0、其他 #8C97B8。

## 本機測試

```
python -m http.server 8765
```

然後開啟 http://localhost:8765/index.html（建議用手機尺寸 390×844）。測試時不要連接真實的同步網址；如需測試同步，可攔截 `script.google.com` 的請求並以本機模擬伺服器回應。提交前檢查瀏覽器主控台沒有錯誤。
