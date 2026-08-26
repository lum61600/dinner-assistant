# 聚餐選店助手

客資部部門聚餐的線上投票工具。福委設定候選日期與餐廳，同事開連結投票，系統統計票數並產生公告。

**線上網址**：https://lum61600.github.io/dinner-assistant/

---

## 功能

### 福委端（設定）

- **本次福委**：Tobey、婧菱、Nana 三人輪值，選定後顯示下一棒
- **投票截止日 + 四個候選日期**：自製日曆彈窗，點欄位任一處即可開啟
- **四間候選餐廳**：三種找店方式
  - 搜尋捷徑：開新分頁查 Google 地圖／熱門推薦／部落格食記
  - `🎲 店家清單抽 4 間`：從自訂清單隨機抽，依地區篩選；清單不足時用內建的連鎖店建議名單補（依「想吃什麼」對應 11 種類型）
  - `🗺 從地圖抓 4 間`：Nominatim 地名查詢 + Overpass API 撈附近餐廳，可依料理類型篩選
- **店家清單**：可逐筆新增／刪除，跟著發布一起存進資料庫

### 投票端

- 選擇自己的名字（親親、維多、婧菱、Tobey、Nana），可隨時改票
- 日期複選、餐廳最多選兩間、攜伴打勾
- 頁面上方顯示投票截止日與剩餘天數

### 結果

- **票數矩陣**：人員 × 選項的表格，仿試算表格式，含 TOTAL 列，最高票整欄標示
- **摘要**：已回覆／攜伴／總人數，自動換算預算上限
- **最後結果**：福委確認後產生公告列（地點／日期時間／預算／福委），標示是否為最高票，地址可自動帶入
- 票數表與公告列都能匯出成 PNG

---

## 技術架構

| 項目 | 使用 |
|---|---|
| 前端 | 單一 `index.html`，純 HTML/CSS/JS，無框架 |
| 資料庫 | Supabase（PostgreSQL + REST API） |
| 靜態託管 | GitHub Pages |
| 找店資料 | OpenStreetMap（Nominatim + Overpass API） |
| 圖片匯出 | HTML Canvas 手繪，無外部套件 |
| 字型 | Inter + Noto Sans TC（Google Fonts） |

### 資料表

```sql
create table public.dinner_sessions (
  id text primary key,
  data jsonb not null,
  updated_at timestamptz not null default now()
);

alter table public.dinner_sessions enable row level security;

create policy "read" on public.dinner_sessions for select using (true);
create policy "insert" on public.dinner_sessions for insert with check (true);
create policy "update" on public.dinner_sessions for update using (true) with check (true);
```

整份狀態（福委、日期、餐廳、店家清單、所有人的票、最後結果）以單一 JSON 存在 `data` 欄位。寫入用 upsert（`Prefer: resolution=merge-duplicates`），送票前會先讀取最新資料再合併，降低互相覆蓋的機率。

### 連線設定

在 `index.html` 的 script 開頭：

```js
const SUPA_URL = 'https://huejxwapgrwwpeouetac.supabase.co';
const SUPA_KEY = 'sb_publishable_...';
const TABLE    = 'dinner_sessions';
```

用的是 Publishable key，設計上就是公開在前端，不可換成 Secret key。

### 一輪投票 = 一筆資料

福委按「發布投票」時產生一組 session id，寫進網址 `?s=xxxxxxxx`。所有人開同一個連結就是同一輪。要開新一輪，打開**不帶 `?s=` 的乾淨網址**重新設定即可，舊連結仍可查詢。

---

## 使用流程

1. 福委開 https://lum61600.github.io/dinner-assistant/
2. 選福委 → 填截止日與四個日期 → 設定四間餐廳 → **發布投票**
3. 按右上角**複製投票連結**，貼到群組
4. 同事開連結投票（每 15 秒自動同步，也可手動按重新整理）
5. 截止後福委到「最後結果」確認地點與日期 → **產生最後結果** → 存成圖片公告

---

## 已知限制

- **免費方案會暗停**：Supabase 免費專案閒置約一週自動暫停。你們單數月才聚餐一次，幾乎每輪都會遇到。程式偵測到連線失敗會明確提示「請福委登入 Supabase 按 Restore」，不會出現看不懂的錯誤。
- **權限是知道網址就能讀寫**：RLS policy 開放給所有人，適合內部小團隊，不要把連結貼到公開場合。
- **沒有 Google 星等**：OpenStreetMap 只有圖資沒有評價，無法依星等排序。要星等得接 Google Places API（需綁信用卡），目前未採用。每張餐廳卡片提供「去 Google 看」連結手動查詢。
- **地圖資料品質不一**：OSM 由網友維護，抓到的店可能已歇業或不適合聚餐，發布前務必確認。
- **地址自動查詢可能不完整**：Nominatim 有時只查得到路名沒有門牌，樓層需自行補。
- **建議名單不保證有分店**：內建的連鎖店清單是通用的，不保證每個地區都有。
- **同時編輯採後寫入者優先**：五人規模足夠，不做衝突處理。

---

## 開發過程踩過的坑

- **沙箱環境擋對外連線**：Anthropic API 與 Overpass API 在預覽環境都回 `Failed to fetch`，需下載或部署後才能測試。錯誤訊息因此改成顯示實際原因（HTTP 狀態碼／逾時／連線失敗），而非籠統的「失敗」。
- **`showPicker()` 在跨來源 iframe 會被擋**：原生日期欄無法用程式叫出日曆，最後改為自製日曆彈窗，順帶解決各瀏覽器外觀不一致的問題。
- **提示訊息顯示在頁面頂端但按鈕在底部**：使用者按了以為沒反應。改成固定浮動的提示卡，並在按鈕上顯示載入狀態。
- **檔名必須是 `index.html`**：Windows 下載容易產生 `index (1).html` 或 `index.html.html`，會導致 GitHub Pages 打不開或沒有覆蓋到原檔。
- **上傳後要按 Commit changes**：拖曳檔案後沒有 commit 等於沒上傳。
- **Supabase 新版介面**：Project URL 不在 API Keys 頁，要到 Integrations → Data API，或從 dashboard 網址列的專案代號組出來。

---

## 後續可以做的

- 接 Google Places API 取得星等與評論數，支援依評價排序
- 投票截止後自動鎖定，不再接受修改
- 歷史紀錄頁，列出過去幾輪去過哪些店，避免重複
