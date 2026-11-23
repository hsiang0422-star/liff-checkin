# 🤖 AI 自動化佈署與系統規格書 (AI Deployment Specification)

> **給使用者的話**：如果您對程式或部署流程不熟悉，您可以將本儲存庫（或此規格書）提供給您的 AI 助手（如 Antigravity、Cursor、Claude、ChatGPT），並貼上下方的「一鍵啟動指令」，讓 AI 引導您完成客製化修改與部署！

---

## 📋 一、給人類使用者的「AI 一鍵佈署啟動指令」

請直接複製下方內容貼給您的 AI 助手：

```text
請幫我部署這套「學校智慧會議簽到系統 (liff-checkin)」。
請先閱讀專案中的 AI_DEPLOYMENT_SPEC.md、README.md、Code.gs 與 templates/ 資料夾內容，並嚴格遵守規格書中的所有不變量與欄位定義。
接下來請一步步引導我收集必要參數（試算表、LINE Channel Access Token、LIFF ID、GAS Web App URL），幫我修改或產生客製化的 index.html 與 Code.gs，並協助我完成 Google Apps Script、LINE Developers 與 GitHub Pages 的部署與驗證測試。
```

---

## 🏗️ 二、系統架構與核心檔案清單

| 檔案路徑 | 執行環境 | 角色與職責 |
| :--- | :--- | :--- |
| `Code.gs` | Google Apps Script (GAS) | 後端 API 路由 (`doPost`)、CacheService 快取管理、LINE Bot Webhook 處理與 Quick Reply、Google Sheets 擴充彈窗與報表產生器。 |
| `index.html` | GitHub Pages / 靜態託管 | LINE LIFF v2 前端 SPA；包含 LocalStorage 「0 秒秒開」快取機制、QR Code 掃描 (`liff.scanCodeV2`)、備援驗證碼、分權管理員看板與會議設定 UI。 |
| `TagDialog.html` | Google Apps Script (`HtmlService`) | 試算表內建選單彈窗：批次為教師加上職務/群組標籤。 |
| `QRCodeDialog.html` | Google Apps Script (`HtmlService`) | 試算表內建選單彈窗：產生並下載啟用中會議的現場 QR Code。 |
| `ReportDialog.html` | Google Apps Script (`HtmlService`) | 試算表內建選單彈窗：產生指定會議的出席統計工作表 (`報表_<會議名>`)。 |
| `templates/` | 下載範本資料夾 | 包含 `學校會議簽到系統_試算表範本.xlsx`、5 個獨立 UTF-8 BOM `.csv` 檔案，以及含佔位符的 `index.template.html`。 |

---

## 🗄️ 三、Google Sheets 資料庫精確規格 (Strict Schema)

所有試算表頁籤名稱（Tab Names）與欄位順序必須與 `Code.gs` 完全一致，否則查詢會失敗。

### 1. `教師主名單`
* **第 1 列為標題列，資料從第 2 列開始**。
* **欄位索引 (0-based in `Code.gs`)**：
  * `Col A (Index 0)`：`教師姓名` (String, 必填，trim 比對)
  * `Col B (Index 1)`：`薪資識別碼(末3碼)` (String/Number, 會自動 `padStart(3, '0')` 補滿 3 碼)
  * `Col C (Index 2)`：`LINE User ID` (String, 首次綁定時由系統自動寫入 `U...`)
  * `Col D (Index 3)`：`綁定狀態` (String, 預設 `"未綁定"`，綁定後寫入 `"已綁定"`)
  * `Col E (Index 4)`：`綁定時間` (String, `"yyyy/MM/dd HH:mm:ss"`)
  * `Col F (Index 5)`：`職務/群組標籤` (String, 多標籤以 `,` `，` `、` 或空白分隔；第 1 個詞會作為 LIFF 歡迎詞頭銜，如 `校長`、`導師`)

### 2. `會議地點與時間設定`
* **第 1 列為標題列，資料從第 2 列開始**。
* **欄位索引 (0-based in `Code.gs`)**：
  * `Col A (Index 0)`：`會議名稱` (String, 必填)
  * `Col B (Index 1)`：`簽到日期/星期` (String or Date, 支援 `"yyyy/MM/dd"` 或 `"週一"`~`"週日"`，使用 `ruleStr.includes(...)` 比對)
  * `Col C (Index 2)`：`開始時間` (String `"HH:mm"` or Date)
  * `Col D (Index 3)`：`結束時間` (String `"HH:mm"` or Date)
  * `Col E (Index 4)`：`現場驗證碼` (String/Number, 用於 QR Code 內容及手動備援輸入比對)
  * `Col F (Index 5)`：`啟用狀態` (String, 嚴格比對 `"啟用"` 或 `"停用"`)
  * `Col G (Index 6)`：`應到對象` (String, `"全體"` / `"全體教師"` / 空值代表全員；或填寫處室/標籤/姓名，逗號分隔)
  * `Col H (Index 7)`：`會議管理員` (String, 教師姓名，多人以逗號或空白分隔；擁有該場次管理看板權限與 LINE QR 索取權限)

### 3. `簽到紀錄`
* **第 1 列為標題列，由系統 `appendRow` 寫入**。
* **欄位索引 (0-based)**：
  * `Col A (0)`: 簽到時間 (`"yyyy/MM/dd HH:mm:ss"`)
  * `Col B (1)`: 會議名稱
  * `Col C (2)`: 教師姓名
  * `Col D (3)`: LINE User ID (手動代簽時為 `"-"`)
  * `Col E (4)`: 簽到狀態 (`"成功"` 或 `"管理員代簽"` 才會計入已簽到)
  * `Col F (5)`: 驗證訊息
  * `Col G (6)`: 備註1 (`"-"`)
  * `Col H (7)`: 備註2 (`"-"`)
  * `Col I (8)`: 簽到方式 (`"掃描簽到"` 或 `"代簽 (管理員姓名)"`)
* **效能特性**：`getInitialData` 與 `processQrCheckIn` 僅讀取 `A:E` 欄並由最後一列倒序掃描（超過 150/100 筆非當日資料即提前 `break`）。

### 4. `管理員名單`
* **第 1 列為標題列，資料從第 2 列開始**。
* **欄位索引 (0-based)**：
  * `Col A (0)`: 管理員姓名
  * `Col B (1)`: `LINE User ID`（最高系統管理員，可檢視與管理所有會議及使用 LIFF 會議設定介面）

### 5. `系統參數`（相容舊名 `系統參數與聯絡設定`）
* Key 值經 `.toUpperCase().replace(/[^A-Z0-9]/g, '')` 正規化比對：
  * `ADMIN_CONTACT_NAME` -> 未綁定畫面顯示的管理員聯絡人姓名
  * `ADMIN_CONTACT_PHONE` -> 未綁定畫面顯示的聯絡電話 (`tel:`)
  * `LINE_CHANNEL_ACCESS_TOKEN` -> LINE Messaging API Long-lived Token（用於 Reply / Push 訊息）
  * `LIFF_ID` -> 您的 LIFF ID（**必填**：用於 LINE Bot 回覆按鈕中的 LIFF 連結，也用來推得驗證 ID Token 所需的 Channel ID）
  * `LINE_LOGIN_CHANNEL_ID` -> （選填）LINE Login Channel ID；未設定時自動取 `LIFF_ID` 中「-」前面的數字

---

## ⚙️ 四、AI 客製化修改檢查清單 (Customization Points)

當 AI 協助新使用者從 `templates/` 或本專案複製部署時，必須協助檢查並替換以下設定：

1. **前端 (`index.html` / `templates/index.template.html`)**：
   * `const MY_LIFF_ID = "..."` -> 替換為使用者在 LINE Developers 建立的 **LIFF ID**。
   * `const GAS_WEB_APP_URL = "..."` -> 替換為使用者部署 GAS Web App 後取得的 **`/exec` 網址**。
   * （選填）`<title>` 與預設頭銜 `" 老師"`（若應用於非學校機構，可改為 `" 同仁"`）。
2. **後端 (`Code.gs`)**：
   * 確保 `系統參數` 分頁（或 Apps Script 指令碼屬性）有設定 `LIFF_ID`；後端會用它驗證前端送來的 LIFF ID Token，未設定時所有 API 請求都會被拒絕。
   * 若使用者所在時區非台灣 (UTC+8)，調整 `Utilities.formatDate(..., "Asia/Taipei", ...)`。
   * 建議設定 GAS 時間驅動觸發器（Time-driven Trigger）每 5~10 分鐘執行 `smartPing()`，在開會前 10 分鐘自動預熱試算表與快取，消除 GAS Cold Start 延遲。
3. **LINE Developers 設定**：
   * **Messaging API**：開啟 `Use webhook`，將 `Webhook URL` 設為 `GAS_WEB_APP_URL`；關閉「自動回應訊息 (Auto-reply messages)」以免干擾管理員選單。
   * **LIFF App**：`Endpoint URL` 設為前端 GitHub Pages 網址；`Scope` 勾選 `profile`、`openid`；`Scan QR` 設為 `ON`。

---

## 🔌 五、API 通訊規格 (`doPost`)

* **請求方式**：`POST`
* **Header**：`Content-Type: text/plain`（**極重要**：GAS Web App 不支援自訂 CORS preflight `OPTIONS`，必須使用 `text/plain` 避免瀏覽器阻擋請求！）
* **Body (JSON string)**：
  * 若含 `events` 陣列 -> 判定為 LINE Bot Webhook 事件 (`handleLineWebhook`)。
  * 若含 `action` 欄位 -> 路由至對應 API：
    * `getInitialData`: `{ myLineUserId }` -> 回傳 `{ isBound, userName, userTitle, isAdmin, isSysAdmin, activeMeetings, activeMeeting }`
    * `bindUser`: `{ myLineUserId, name, idCode }` -> 寫入綁定並呼叫 `clearTeacherCache()`
    * `processQrCheckIn`: `{ myLineUserId, scannedCode }` -> 比對開啟中會議 `passcode` 並防重複簽到
    * `getAdminDashboardData`: `{ targetMeetingName, myLineUserId }` -> 回傳應到/實到/未到列表與可管理會議選單
    * `adminManualCheckIn`: `{ name, myLineUserId, targetMeetingName }` -> 管理員代簽
    * `sendReminders`: `{ targetIds, meetingName }` -> 呼叫 LINE Push API 發送催簽訊息
    * `getMeetingSettingsData`: `{ myLineUserId }` -> 僅限最高管理員讀取所有會議
    * `saveMeetingSetting`: `{ myLineUserId, meetingData }` -> 新增/更新會議並呼叫 `clearMeetingCache()`
    * `toggleMeetingStatus`: `{ myLineUserId, rowIndex, newStatus }` -> 切換啟用/停用並呼叫 `clearMeetingCache()`

---

## ⚠️ 六、核心不變量與防呆提醒 (Critical Invariants for AI)

1. **GAS 部署權限**：Web App 部署時「執行身分」必須為 **我 (Me)**，「誰可以存取」必須為 **所有人 (Anyone)**，否則 LIFF 跨網域請求與 LINE Webhook 皆會回傳 401/302 失敗。
2. **GAS 更新版本**：每次修改 `Code.gs` 存檔後，必須點擊「部署 > 管理部署作業 > 編輯 > **新版本** > 部署」，舊版網址才會套用新程式碼。
3. **快取失效一致性**：任何會改寫 `教師主名單` 或 `會議地點與時間設定` 的後端函式，結束前必須呼叫 `clearTeacherCache()` 或 `clearMeetingCache()`；GAS CacheService 單一 key 上限為 100KB，目前已有長度檢查保護。
4. **如何取得第一個最高管理員的 LINE User ID？**
   * LINE Bot/LIFF 的 User ID 是針對單一 Provider 的 `U` 開頭 33 碼字串（不同 LINE Official Account 取得的 ID 不同）。
   * **最快取得方式**：先將自己加入 `教師主名單`，打開 LIFF 完成一次「首次身分綁定」，試算表 `教師主名單` C 欄就會自動出現您的 `LINE User ID`，將其複製貼到 `管理員名單` B 欄並點擊 `清除系統快取` 即可成為最高管理員！
5. **LocalStorage 帳號切換防護**：前端快取 key 為 `qrcode_checkin_user_profile`，當 `liff.getProfile()` 取回的 `userId` 與快取不一致時，必須自動清除快取並重新載入。

---

## ✅ 七、部署後驗收清單 (Verification Checklist)

AI 協助部署完成後，請引導使用者逐項確認：
- [ ] **GAS Web App 連線**：於瀏覽器或 curl 測試 `GAS_WEB_APP_URL` 是否不會導向 Google 登入頁。
- [ ] **試算表選單**：重新整理 Google 試算表後，上方工具列出現 `會議管理功能` 自訂選單。
- [ ] **LIFF 開啟與秒開**：從手機 LINE 開啟 LIFF 連結，首次綁定成功寫入 `教師主名單` C~E 欄；關閉重開第二次時在 0.1 秒內直接顯示姓名與問候語。
- [ ] **會議比對與簽到**：建立一筆涵蓋當下時間與日期（或星期）的「啟用」測試會議，掃描 QR Code 與測試手動輸入驗證碼皆能寫入 `簽到紀錄`。
- [ ] **LINE Bot 管理員選單**：於 LINE 官方帳號聊天室發送 `選單`，確認出現 Quick Reply 快捷按鈕且 `QR`、`簽到概況`、`未簽到名單`、`清除快取` 皆可正常運作。
