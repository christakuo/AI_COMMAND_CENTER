# Task 04H-2F-3：顧問獎金 read-only API Web route／token 安全測試總結

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04H-2F-3：顧問獎金 read-only API Web route／token 安全測試總結文件化 |
| 完成日期 | 2026-07-03 |
| 彙整範圍 | Task 04H-2F 規劃、mock route、真實 ID Token、Access Token 安全盤點與補強測試 |
| 本次工作 | 只整理 AI_COMMAND_CENTER Markdown 文件 |
| 階段結論 | 主要安全邊界已完成，可進入 LIFF 顧問獎金畫面規格階段 |

## 2. 任務目的

Task 04H-2F 的目的如下：

- 驗證顧問獎金 read-only API 在 Web route 與 LINE token 身分驗證層級的安全性。
- 確保安全判斷不只依賴繞過 Web 入口的 Helper 測試。
- 在正式 LIFF 前端串接前，確認後端已具備基本的身分驗證、越權阻擋及資料隔離邊界。
- 確保 read-only API 不寫入 Sheet、不自動建立分頁，也不產生獎金草稿。

## 3. 測試與修正總覽

| 階段 | 內容 | 結果 |
|---|---|---|
| Task 04H-2F 規劃 | 規劃 Web route、LINE token、越權、GET 拒絕、欄位白名單及不寫入測試 | 完成 |
| Task 04H-2F-1 mock route | GET 的 Dashboard／Records 均回傳 `LINE_AUTH_REQUIRED_USE_POST` | 通過 |
| Task 04H-2F-1 mock route | POST Dashboard／Records 缺 token 均回傳 `TOKEN_MISSING` | 通過 |
| Task 04H-2F-1 mock route | POST Dashboard／Records 帶 `consultantId` 均回傳 `ACCESS_DENIED` | 通過 |
| Task 04H-2F-1 mock route | 錯誤回應不含 records、summary 或獎金資料 | 通過 |
| Task 04H-2F-2C 真實 ID Token | 有效 ID Token 的 Dashboard 與 Records | 通過 |
| Task 04H-2F-2C 真實 ID Token | 後端由 token 對應 `HC9001`，且 `HC9001` 看不到 `HC9002` 測試資料 | 通過 |
| Task 04H-2F-2C 真實 ID Token | Records 欄位白名單、raw `lineUserId` 偽造防護 | 通過 |
| Task 04H-2F-2C 真實 ID Token | `consultantId` 越權、token 缺失、無效 token、Web route `limit=0` 均拒絕 | 通過 |
| Task 04H-2F-2C 真實 ID Token | Sheet 不寫入、不自動建表 | 通過 |
| Task 04H-2F-2D 安全盤點 | 發現 Access Token fallback 缺少 `client_id` Channel 綁定 | 發現缺口 |
| Task 04H-2F-2D 安全盤點 | 發現 verify 失敗後仍可能繼續呼叫 Profile | 發現缺口；判定需先修正再串接 LIFF |
| Task 04H-2F-2D-Fix1 | 修改 Apps Script `LineAuthService.gs` | 完成並合併 |
| Task 04H-2F-2D-Fix1 | 要求 `client_id` 等於 `LINE_LOGIN_CHANNEL_ID` | 已補強 |
| Task 04H-2F-2D-Fix1 | 要求 `expires_in > 0` 且 scope 包含 `profile` | 已補強 |
| Task 04H-2F-2D-Fix1 | verify 失敗立即阻斷 Profile | 已補強 |
| Task 04H-2F-2D-Fix1 | 錯誤回應不洩漏 token、`client_id`、LINE 完整 response、Profile 內容或 stack trace | 已補強 |
| Task 04H-2F-2D-Fix1 | ID Token 回歸、同 Channel Access Token only、Bearer 前綴拒絕 | 通過 |
| Task 04H-2F-2D-Fix1 | Sheet 不寫入、不自動建表 | 通過 |

## 4. 目前已完成的安全能力

### Web route 與身分驗證

- GET 呼叫敏感 action 會被拒絕，必須改用 POST。
- 缺少 token 時會拒絕，不回傳獎金資料。
- 無效 ID Token 會拒絕。
- 有效 ID Token 可通過驗證並對應本人顧問身分。
- 後端以驗證後的 LINE 身分為準，前端傳入的 raw `lineUserId` 無法偽造查詢身分。
- Request body 帶 `consultantId` 會拒絕，避免前端指定他人顧問 ID。
- 單帳號真實 token 測試中，`HC9001` 看不到 `HC9002` 的測試資料。
- `limit=0` 會回傳 `INVALID_ARGUMENT`，不回傳 records。

### 回傳資料最小化

Records 使用欄位白名單，確認不回傳：

- 完整會員姓名。
- 完整手機。
- LINE UID。
- 付款批次 ID。
- 內部備註。
- 顧問 ID。

### Access Token fallback

- 已要求 Access Token verify 回應的 `client_id` 必須與 `LINE_LOGIN_CHANNEL_ID` 一致。
- 已要求 `expires_in > 0`。
- 已要求 scope 包含 `profile`。
- verify 未完整通過前不得呼叫 Profile。
- Bearer 前綴等不符合預期的 token 格式會拒絕。
- 錯誤回應不揭露 token、Channel 資訊或 LINE 服務完整回應。

### Read-only 邊界

- API 不寫入 Sheet。
- API 不自動建立 `bonus_records_獎金明細表` 或其他分頁。
- API 不執行 `ensureBonusRecordsSheet()`。
- API 不執行 `generateConsultantBonusDrafts()`。
- API 不產生獎金草稿。

## 5. 目前尚未完成與保留限制

- 雙帳號真實 LINE ID Token 顧問隔離尚未完成。
- `HC9002` 真實 token 對稱驗證尚未完成。
- 跨 LINE Login Channel Access Token 的真實拒絕測試尚未完成。
- 正式 Apps Script 尚未同步。
- 正式 Apps Script 尚未部署。
- 正式 Google Sheet 尚未修改。
- 正式 `bonus_records_獎金明細表` 尚未建立。
- LIFF 顧問獎金畫面尚未實作。
- 顧問端目前尚未看得到獎金資料。
- 導師獎金、團隊獎金、月結、付款批次及獎金審核後台仍未實作。

目前完成的是測試環境中的 read-only API 主要安全邊界，不代表正式環境已上線，也不代表完整顧問獎金制度已完成。

## 6. 是否阻塞 LIFF 顧問獎金畫面規格

判斷：**不阻塞 Task 04H-2G「LIFF 顧問獎金畫面規格」**。

理由：

- read-only API 的查詢能力已完成。
- Web route 的 GET／POST 邊界、token 缺失、無效 token 及越權拒絕已驗證。
- 真實 ID Token 單帳號主路徑已通過。
- Access Token fallback 已補強 Channel 綁定與 verify 閘門，且同 Channel 正向測試通過。
- 欄位白名單與不寫入邊界已驗證。

因此可先定義 LIFF 畫面的資訊架構、空狀態、載入狀態、錯誤狀態及 API 串接方式，不必等待全部補充測試完成。

但正式串接與上線前仍需保留以下待辦：

- 雙帳號真實 token 隔離測試。
- 跨 Channel Access Token 真實拒絕測試。
- 正式 Apps Script 同步驗收。
- 正式環境 `not_initialized` 空狀態驗收。

## 7. 下一步建議

1. **Task 04H-2G：LIFF 顧問獎金畫面規格**  
   先定義顧問看到的獎金摘要、明細、狀態、空狀態及錯誤提示，不修改正式環境。
2. **Task 04H-2H：LIFF 顧問獎金畫面實作**  
   依 04H-2G 規格串接既有 read-only API，先使用測試環境驗證。
3. **Task 04H-2I：正式 Apps Script 同步與 `not_initialized` 空狀態驗收**  
   正式同步後先確認尚未建表時只回傳安全空狀態，不自動建立分頁。
4. **Task 04H-2J：正式 LIFF＋正式 Apps Script read-only 空狀態驗收**  
   驗證顧問端在正式尚無獎金資料時顯示正確，不誤顯示為已有獎金或系統錯誤。
5. 待有第二個測試帳號時，補做雙帳號真實 token 隔離測試。
6. 待有第二個 LINE Login Channel 時，補做跨 Channel Access Token 真實拒絕測試。

## 8. 安全聲明

- 本次只文件化既有測試與修正結果。
- 不記錄任何 LINE token。
- 不記錄完整 LINE UID。
- 不記錄 Channel ID。
- 不記錄測試 Web App URL。
- 未修改 Apps Script repo。
- 未修改 LIFF repo。
- 未部署。
- 未執行 Apps Script 函式。
- 未修改正式 Google Sheet。
- 未建立正式 `bonus_records_獎金明細表`。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。

## 9. 結論

Task 04H-2F 已完成 mock route、真實 LINE ID Token 單帳號、Access Token Channel 綁定補強與相關回歸測試。顧問獎金 read-only API 的主要 Web route、身分驗證、越權防護、資料最小化及不寫入安全邊界已具備，因此可進入 LIFF 顧問獎金畫面規格階段。

正式上線仍必須另行完成正式同步、`not_initialized` 空狀態及正式 LIFF 串接驗收；雙帳號隔離與跨 Channel Access Token 真實拒絕則列為後續補測，不應在文件中誤標示為已完成。
