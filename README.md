# CHAT · 仿 LINE 的家族聊天室 💚

部署在 **Cloudflare** 上、介面仿 **LINE** 的網頁聊天室。
親友在手機上把它「加到主畫面」就像一個 App；在電腦上用瀏覽器開也能用。
資料存在你自己的 Cloudflare 帳號裡，**免費方案就能跑**。

| | |
|---|---|
| 💬 **1 對 1 與群組聊天** | 訊息永久保存，換裝置、重新整理都不會消失 |
| ✅ **已讀回條** | 跟 LINE 一樣顯示「已讀」（群組顯示「已讀 N」） |
| ⌨️ **即時體驗** | WebSocket 即時推播、「正在輸入…」、新訊息通知與音效 |
| 🖼️ **豐富訊息** | 照片（自動壓縮）、語音訊息、貼圖、**自訂貼圖**（把家人照片做成貼圖）、表情符號、收回訊息（24 小時內） |
| 💛 **互動** | 表情回應、回覆引用、投票、群組公告、訊息搜尋、**連結預覽卡片**、未讀分隔線、聊天室置頂 |
| 🔔 **離線推播** | 網頁沒開也收得到通知（Web Push，金鑰自動產生、零設定） |
| ✅ **待辦清單／📅 行事曆** | 全家共用，行事曆到時間自動通知所有人。在聊天室右上角 ☰ 裡 |
| 💚 **LINE 風格** | 藍色聊天底、綠色泡泡、第一則訊息帶小尾巴、回覆引用放在泡泡裡、「Aa」輸入框（沒打字時是麥克風，打字後變送出） |
| 🎨 **外觀** | 六種聊天背景（LINE 藍、簡約白、薄荷、蜜桃、奶茶、夜空）、多種字型、深色模式、四段字體大小（長輩友善）、上班低調模式 |
| 🔐 **邀請碼註冊** | 只有拿到邀請碼的人能註冊；第一位註冊者自動成為管理員 |
| 🙈 **成員隱私** | 管理員看得到所有人；其他人只看得到管理員和被拉進同一個群組的人 |
| 📝 **我的記事本** | 跟自己聊天，當備忘錄用 |

---

## 📖 手冊

| 文件 | 給誰看 |
|---|---|
| [docs/部署手冊.md](docs/部署手冊.md) | **要架站的人**。註冊 Cloudflare、拿 token、設定 GitHub Secrets、部署、備份、疑難排解，每一步都有說明，不需要會寫程式 |
| [docs/使用手冊.md](docs/使用手冊.md) | **所有使用者（含長輩）**。加到主畫面、聊天、照片語音、通知、外觀、隱私 |

---

## 🚀 部署（約 20 分鐘，完整步驟見部署手冊）

共同前提：一個 [Cloudflare 帳號](https://dash.cloudflare.com/sign-up)（免費，不用信用卡），
並且在 **Workers & Pages** 頁面設定好 `*.workers.dev` 子網域。

### 方法 A：只用瀏覽器（GitHub Actions，推薦）

1. Cloudflare：複製 **Account ID**；建立 API Token（範本選 **Edit Cloudflare Workers**）
2. 這個 repo → **Settings → Secrets and variables → Actions**，新增兩個 secret：
   `CLOUDFLARE_API_TOKEN`、`CLOUDFLARE_ACCOUNT_ID`
3. **Actions** 分頁 → 左邊選「**部署到 Cloudflare**」→ **Run workflow**
4. 約一分鐘後完成，網址是 `https://chat.<你的子網域>.workers.dev`

### 方法 B：用自己的電腦

需要 [Node.js](https://nodejs.org/) 18 以上。

```bash
git clone https://github.com/yuyangmoney-stack/chat.git
cd chat
npm install
npx wrangler login     # 開瀏覽器授權 Cloudflare
npx wrangler deploy
```

### ⚠️ 部署完立刻做

**馬上打開網址註冊。** 第一個註冊的人會自動成為管理員，而在那之前任何人都能註冊。
註冊完成後，沒有邀請碼的人就無法再註冊。

**不需要**建資料庫、**不需要**設定其他金鑰 —— 資料都存在 Worker 附帶的
Durable Object（SQLite）裡；重新部署不會弄丟資料。

---

## 👨‍👩‍👧 開始使用

1. **你先註冊**：打開網址 → 註冊 → 你就是管理員 👑
2. **設定邀請碼**：設定 → 管理員 → 填邀請碼 → 儲存 → 按「**複製邀請訊息**」傳給親友
3. **親友註冊**：打開網址 → 註冊時輸入邀請碼。註冊完會自動有一個跟你的聊天室

家人都進來之後，把邀請碼清空再儲存，就等於關閉註冊（已註冊的人不受影響）。

---

## 🧱 技術架構

```
瀏覽器（前端 SPA，public/）
   │  HTTPS（REST API：登入、傳訊息、已讀…）
   │  WebSocket（即時接收：新訊息、已讀、輸入中…）
   ▼
Cloudflare Worker（src/index.js）── 靜態檔由 Cloudflare Assets 供應
   ▼
ChatServer Durable Object（src/chatserver.js）
   ├─ SQLite 儲存：使用者、聊天室、訊息、已讀進度、邀請碼、待辦、行事曆
   └─ WebSocket Hibernation：所有線上成員的即時連線
```

設計重點：

- **單一 Durable Object 實例**承載全部狀態。親友規模（數十人）下這是最簡單、
  最一致的架構：沒有分散式競態、免設定 D1/KV/R2、部署零設定。
- **密碼**以 PBKDF2-SHA256（10 萬次迭代＋隨機鹽）雜湊儲存；登入與邀請碼嘗試都有次數鎖定，
  失敗次數存在資料庫裡，Durable Object 被回收也不會歸零。
- **Session** 為隨機 token 存伺服器端，60 天未使用自動失效；改密碼會讓其他裝置登出。
- **成員可見性在伺服器端強制**：帳號清單、建立私訊／群組、邀請進群、成員資料的即時推播，
  都以「管理員 or 同聊天室」規則過濾，不是只靠前端隱藏。
- WebSocket 使用 **Hibernation API**，沒人講話時 Durable Object 休眠，幾乎不耗用量。
- **照片**在瀏覽器端壓縮成 JPEG（最長邊 1280px、約 450KB 內）後存進資料庫，
  但不會夾在訊息清單裡送出來：訊息只帶一個 `/api/media/<訊息編號>` 的簽章網址，
  瀏覽器捲到才載入、可以長時間快取。簽章裡綁了收件者，而且**每次取用都重新檢查授權**
  （帳號還在、沒被停用、仍是該聊天室成員），退出聊天室後舊網址立刻失效。
- **連結預覽**的外部抓取會自己處理轉址、每一跳都重新檢查，不會被外站導去內網（SSRF）。

### 免費方案夠用嗎？

夠。Workers 免費方案每天 10 萬次請求、Durable Objects（SQLite 版）含 5GB 儲存。
超過免費額度時 Cloudflare 只會讓操作暫時失敗，不會自動扣款。

---

## 🛠️ 常用操作

| 想做什麼 | 指令／位置 |
|---|---|
| 本機開發預覽 | `npm run dev` → http://localhost:8787 |
| 跑 API 測試 | 另開終端機 `npm test`（隱私規則、權限、聊天核心功能、待辦行事曆、管理員；建議先 `rm -rf .wrangler/state` 用乾淨資料） |
| 跑資安測試 | `npm run test:security`（攻擊者視角：越權、假 token、XSS、SQL injection、SSRF、照片網址授權、暴力破解；務必用乾淨資料） |
| 部署／更新 | Actions → 部署到 Cloudflare → Run workflow，或 `npx wrangler deploy` |
| 重新產生 App 圖示 | `npm run icons`（改 `scripts/gen-icons.mjs` 裡的 `BRAND` 顏色後執行） |
| 備份聊天資料 | 網頁「設定」→「管理員」→「下載聊天備份」 |
| 換自己的網域 | Cloudflare Dash → Workers → chat → Settings → Domains & Routes |

## 📁 檔案結構

```
wrangler.jsonc        Cloudflare 設定（Worker、Durable Object、靜態資源）
src/index.js          Worker 入口（路由轉送、安全標頭）
src/chatserver.js     聊天伺服器：帳號、訊息、群組、WebSocket（Durable Object）
public/index.html     前端頁面骨架
public/app.js         前端邏輯
public/style.css      LINE 風格樣式（全尺寸響應式）
public/manifest.webmanifest + sw.js + icons/   PWA（加到主畫面）
public/fonts/         自架手寫字型（含授權檔）
scripts/gen-icons.mjs 產生 App 圖示（零相依）
scripts/test-api.mjs  API 測試
scripts/test-security.mjs  資安測試
.github/workflows/deploy.yml  GitHub Actions 部署
```

## 🔔 離線推播小抄

在「設定 → 通知 → 離線推播」開啟後，網頁完全關閉也會收到「新訊息」提醒
（內容一律不顯示聊天文字）：

- **Android／電腦 Chrome、Edge**：直接開啟即可。
- **iPhone**：iOS 16.4 以上，需先用 Safari「**加入主畫面**」，再從主畫面圖示開啟，
  才能開啟離線推播（這是 Apple 的限制）。

## 字型來源與授權

字型只在選用時才下載，選擇器裡的預覽只抓幾十個字的子集。

| 字型 | 來源 | 授權 |
|------|------|------|
| 思源黑體／宋體 TC、粉圓（Huninn）、芫荽（Iansui）、霞鶩文楷 TC、霞鶩漫黑（LXGW Marker Gothic）、Klee One、Yomogi、Hachi Maru Pop、Yuji Syuku | Google Fonts（按需載入） | SIL OFL 1.1 |
| 辰宇落雁體 | `public/fonts/chenyuluoyan/`，取自 npm `@fontpkg/chenyuluoyan`，依 Google Fonts 的繁中分片切成 100 個 woff2 | SIL OFL 1.1（見 `public/fonts/LICENSE-chenyuluoyan.txt`） |
| 清松手寫體 1 | `public/fonts/jason-handwriting/`，取自 npm `@fontpkg/qing-song-shou-xie-ti-1`，同上切成 99 個 woff2 | SIL OFL 1.1（見 `public/fonts/LICENSE-jason-handwriting.txt`） |

## ⚠️ 已知限制

- 沒有語音／視訊通話。
- 尚未支援傳一般檔案（文件、影片）。
- 訊息以明文存在**你自己**的 Cloudflare 帳號裡（跟 LINE 一樣是伺服器可讀的模式），
  沒有端對端加密。請替 Cloudflare 帳號開啟兩步驟驗證。
- 本專案與 LINE 公司無關，只是介面風格參考。
