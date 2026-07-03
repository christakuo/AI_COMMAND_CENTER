# Task 04H-2G-Doc：LIFF 顧問獎金畫面規格

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04H-2G-Doc：LIFF 顧問獎金畫面規格文件化 |
| 完成日期 | 2026-07-03 |
| 前置條件 | Task 04H-2D、04H-2D-Fix1、04H-2F 系列安全測試已完成 |
| 本次範圍 | 顧問獎金第一版畫面與前端串接規格 |
| 後續實作 | Task 04H-2H |

## 2. 任務目的

本文件用於：

- 鎖定 LIFF 顧問獎金畫面第一版 MVP 規格。
- 鎖定前端串接 `getConsultantBonusDashboard` 與 `getConsultantBonusRecords` 的欄位及狀態規則。
- 避免前端顯示 API 白名單以外的個資或內部欄位。
- 避免混淆 `not_initialized`、真正無資料的 `empty` 與查詢 `error`。
- 作為 Task 04H-2H 前端實作、測試及驗收依據。

本文件只定義畫面與串接方式，不代表正式 Apps Script 已同步，也不授權公開正式獎金入口。

## 3. `yuanxin-liff-demo` 顧問大廳現況

### Repo 狀態

| 項目 | 盤點結果 |
|---|---|
| Repo | `yuanxin-liff-demo` |
| Branch | `main` |
| HEAD | `9d7317ccff5ee7c9bc59201feebc49e5a3751cc3` |
| Working tree | clean |
| 本次是否修改或部署 | 否 |

### 顧問入口支援

現有顧問入口可由以下方式識別：

- `portal=consultant`
- `portal=portal`
- `portal=business`
- `portal=advisor`
- `mode` 或 `view` 使用上述值
- `#consultant-portal`
- `liff.state` 內的 `portal` 參數

### 顧問大廳 DOM

- `#screen-consultant-portal`
- `#consultantPortalLoading`
- `#consultantPortalError`
- `#consultantPortalContent`

### 現有內容順序

1. 顧問基本資料卡。
2. 四項成果摘要。
3. 推薦／團隊分頁。
4. 顧問資源中心。
5. 返回會員首頁。

### 現有功能與缺口

| 項目 | 現況 |
|---|---|
| 顧問姓名、ID、位階、狀態 | 已有 |
| 導師權限簡易徽章 | 已有 |
| 我的推薦會員 | 已有 |
| 我的推薦顧問 | 已有 |
| 團隊顧問數量 | 已有 |
| 團隊會員 | 已有 |
| 顧問資源中心 | 已有 |
| 獨立組織發展區塊 | 尚無 |
| 獎金總覽 | 尚無 |
| 獎金明細 | 尚無 |

## 4. 建議插入位置與畫面結構

建議在「四項成果摘要」後、「推薦／團隊分頁」前插入「我的獎金」區塊。

區塊依序包含：

1. 獎金總覽卡片。
2. 「查看獎金明細」入口。
3. 展開後的月份與狀態篩選。
4. 獎金明細列表。

採用此位置的原因：

- 顧問進入大廳後可優先看到收入成果。
- 不必先穿過推薦名單與資源中心才能查詢獎金。
- 獎金查詢應是獨立區塊；即使查詢失敗，也不應讓整個顧問大廳失效。

## 5. API 呼叫現況與串接限制

### 現有 Web App 設定

- Apps Script Web App URL 目前設定在 `index.html` 的 `window.YUANXIN_REFERRAL_CONFIG.webhookUrl`。
- 目前不是 Vercel 環境變數。
- 目前沒有正式／測試環境切換機制。
- 現有顧問 API 由 `YuanxinReferral.queryConsultantPortal(...)` 呼叫。

### 現有 request 方式

| 項目 | 設定 |
|---|---|
| Method | `POST` |
| Content-Type | `text/plain;charset=utf-8` |
| Body | `JSON.stringify(payload)` |
| Redirect | `follow` |

### 現有 LINE token 流程

1. 執行 `liff.init()`。
2. 未登入時執行 `liff.login()`。
3. 取得 `liff.getAccessToken()`。
4. 取得 `liff.getIDToken()`。
5. 取得 `liff.getProfile()`。
6. 將 `lineAccessToken`、`lineIdToken`、`lineUserId`、`lineDisplayName` 等資料送至 Apps Script。
7. Access Token 或 ID Token 任一存在即可送出；兩者都不存在時顯示 `TOKEN_MISSING`。

後端身分判斷必須以驗證後的 token 身分為準，不可信任前端 raw `lineUserId`。

### Task 04H-2H 必須修正的串接缺口

`queryConsultantPortal()` 目前會重新組裝 payload，不會自動保留呼叫端傳入的 `month`、`status`、`limit`、`nextPageToken`。

Task 04H-2H 必須加入明確查詢參數白名單：

- `month`
- `status`
- `limit`
- `nextPageToken`

不得使用任意 `...data` 將所有呼叫端欄位展開到 request。實際可傳欄位及格式仍以 Task 04H-2D API 契約為準。

## 6. 獎金總覽卡片規格

### API

`getConsultantBonusDashboard`

### 顯示欄位

| 畫面內容 | API 欄位 |
|---|---|
| 本月待審核金額 | `summary.pendingReviewAmount` |
| 本月已核准金額 | `summary.approvedAmount` |
| 本月已付款金額 | `summary.paidAmount` |
| 本月總獎金金額 | `summary.totalAmount` |
| 本月獎金筆數 | `summary.totalCount` |
| 最後更新時間 | `updatedAt` |
| 固定提示 | 獎金資料以後台審核結果為準 |

### 顯示規則

- 金額顯示格式為 `NT$4,000`。
- 前端只負責金額與日期顯示格式化。
- 前端不得自行重新計算獎金。
- 前端不得根據名單、推薦或付款資料推算獎金。
- Dashboard API 是總覽數字的唯一資料來源。
- `setupStatus=not_initialized` 時不得讀取或代填 `summary` 數字。

## 7. 獎金明細列表規格

### API

`getConsultantBonusRecords`

### 每筆顯示欄位

- `recognizedDate`
- `bonusType`
- `amount`
- `status`
- `paidStatus`
- `paidAt`
- `sourceSummary`
- `updatedAt`

`bonusId` 可作為前端列表識別鍵，但第一版不必直接顯示給顧問。

### 建議呈現順序

1. 日期。
2. 獎金類型。
3. 已遮罩的來源摘要。
4. 金額。
5. 審核狀態。
6. 付款狀態及付款日期。
7. 更新時間。

### 顯示規則

- 不得由前端推算付款日期。
- 不得由前端推算獎金認列規則。
- 不得顯示後端未列入白名單的欄位。
- `sourceSummary` 必須由後端完成遮罩；禁止前端先接收完整個資再自行遮罩。
- `paidAt` 無值時只顯示付款狀態，不推測或補日期。

## 8. 篩選規格

| 篩選項目 | 第一版規則 |
|---|---|
| 月份 | 預設當月 |
| 月份格式 | `YYYY-MM` |
| 狀態 | `all`、`pending_review`、`approved`、`paid` |
| `limit` | 固定 `50` |
| `nextPageToken` | 保留前端變數，但第一版不顯示分頁操作 |

補充規則：

- 不開放使用者自由輸入 `limit`。
- 第一版不做頁碼或「載入更多」。
- 若未來單月超過 50 筆，再以 `nextPageToken` 新增「載入更多」。
- 使用者變更月份或狀態後，重新查詢 Records；Dashboard 是否跟著月份更新，依 API 契約使用相同 `month`。

## 9. 狀態判斷順序與文案

前端必須依以下順序判斷，不可先看 `empty` 再看 `setupStatus`：

1. `loading`
2. `success=true` 且 `setupStatus=not_initialized`
3. `success=true`、`setupStatus=ready` 且 `empty=true`
4. `success=true` 且有資料
5. `success=false` 或有 `errorCode`

### Loading

- 只在「我的獎金」區塊顯示載入狀態。
- 不遮蔽既有顧問基本資料、成果摘要或推薦／團隊內容。

### `not_initialized`

條件：

- `success=true`
- `setupStatus=not_initialized`
- `empty=true`

文案：

> 獎金查詢功能建置中，資料開放後會在此顯示。

規則：

- 不使用紅字或錯誤警示樣式。
- 不要求使用者重新登入。
- 不顯示金額為 0。
- 不顯示「目前沒有獎金」。
- 避免讓顧問誤解為已完成計算但獎金為零。

### Ready＋empty

條件：

- `success=true`
- `setupStatus=ready`
- `empty=true`

文案：

> 目前查無本月獎金資料。

補充：

> 獎金資料以後台審核結果為準。

### Error

| `errorCode` | 顧問端文案 |
|---|---|
| `TOKEN_MISSING` | 需要完成 LINE 身分驗證，請重新從 LINE 開啟或重新登入。 |
| `ACCESS_DENIED` | 無法確認顧問身分，請確認是否使用顧問綁定的 LINE 帳號。 |
| `INVALID_ARGUMENT` | 查詢條件不正確，請重新選擇月份或狀態。 |
| `INTERNAL_ERROR` | 系統目前忙碌，請稍後再試。 |
| 其他錯誤 | 暫時無法取得獎金資料，請稍後再試。 |

錯誤處理規則：

- 只在獎金區局部顯示錯誤。
- 不得使整個顧問大廳失效。
- 不直接顯示 Apps Script 原始錯誤內容。
- 不顯示 stack trace 或內部服務資訊。

## 10. 隱私與欄位限制

### 前端不得顯示或保存

- 完整會員姓名。
- 完整手機。
- 會員 LINE UID。
- 顧問 LINE UID。
- 其他顧問 ID。
- 付款批次 ID。
- 內部備註。
- Access Token。
- ID Token。
- Channel ID。
- Apps Script 內部錯誤。
- Stack trace。
- Sheet 列號。
- 內部資料鍵值。
- 測試 Web App URL。

### Dashboard 可使用欄位白名單

- `success`
- `setupStatus`
- `empty`
- `consultant.consultantId`：僅限 token 驗證後的本人顧問 ID。
- `consultant.consultantName`
- `summary.month`
- `summary.pendingReviewAmount`
- `summary.approvedAmount`
- `summary.paidAmount`
- `summary.pendingReviewCount`
- `summary.approvedCount`
- `summary.paidCount`
- `summary.totalAmount`
- `summary.totalCount`
- `updatedAt`
- `message`

### Records 可使用欄位白名單

- `success`
- `setupStatus`
- `empty`
- `records[].bonusId`
- `records[].recognizedDate`
- `records[].bonusType`
- `records[].amount`
- `records[].status`
- `records[].paidStatus`
- `records[].paidAt`
- `records[].sourceSummary`
- `records[].updatedAt`
- `pagination.limit`
- `pagination.nextPageToken`
- `message`

前端不得因畫面需求要求 API 回傳完整 Sheet row，也不得透過瀏覽器記錄 token 或敏感 response。

## 11. 正式上線前限制與驗收順序

### 目前限制

- 正式 Apps Script 尚未同步 read-only bonus API。
- Access Token Fix1 尚未同步正式環境。
- 正式 Apps Script 尚未重新部署。
- 正式 `bonus_records_獎金明細表` 尚未建立。
- 尚未執行 `ensureBonusRecordsSheet()`。
- 尚未執行 `generateConsultantBonusDrafts()`。
- 尚未產生正式獎金草稿。
- 即使前端先完成，也不能立即向顧問正式開放真實獎金資料。
- 若前端早於後端部署完成，不應公開獎金入口。
- 不得將 `unknown action` 偽裝成 `not_initialized`；兩者代表不同狀態。

### 正式驗收順序

1. 同步正式 Apps Script read-only API 與 Access Token Fix1。
2. 建立新的正式 Web App deployment。
3. 確認正式環境已正確設定 `LINE_LOGIN_CHANNEL_ID`。
4. 驗收正式 `not_initialized` 空狀態。
5. 確認查詢不會自動建立 `bonus_records_獎金明細表`。
6. 確認查詢不會寫入任何 Sheet。
7. 完成正式 LIFF 串接驗收後，再開放正式獎金入口。

## 12. Task 04H-2H 實作邊界

### 可以做

- 新增獎金區 DOM 與畫面樣式。
- 串接既有 read-only API。
- 在 `queryConsultantPortal()` 加入查詢參數白名單。
- 實作 loading、empty、`not_initialized` 與 error 狀態。
- 實作金額及日期格式化。
- 實作手機版響應式畫面。
- 使用 mock response 做前端測試。

### 不可做

- 正式部署或公開入口。
- 建立或修改 Google Sheet。
- 自動建立 `bonus_records_獎金明細表`。
- 產生獎金草稿。
- 修改獎金狀態。
- 前端自行計算獎金。
- 顯示未列入白名單的個資。
- 將 `unknown action` 當成 `not_initialized`。
- 直接切換到尚未部署的新 Web App URL。
- 修改正式 Apps Script。
- 修改 Vercel 設定。

## 13. 後續任務建議

1. Task 04H-2H：LIFF 顧問獎金畫面實作。
2. Task 04H-2I：正式 Apps Script 同步與 `not_initialized` 空狀態驗收。
3. Task 04H-2J：正式 LIFF 顧問入口 read-only 驗收。
4. 待有第二測試帳號時，補做雙帳號真實 token 顧問隔離測試。
5. 待有第二 LINE Login Channel 時，補做跨 Channel Access Token 真實拒絕測試。

## 14. 安全聲明

- 本次只文件化 LIFF 顧問獎金畫面規格。
- 未記錄任何 token。
- 未記錄完整 LINE UID。
- 未記錄 Channel ID。
- 未記錄測試 Web App URL。
- 未修改 `yuanxin-liff-demo`。
- 未修改 Apps Script repo。
- 未部署。
- 未執行 Apps Script 函式。
- 未修改 Google Sheet。
- 未建立正式 `bonus_records_獎金明細表`。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。

## 15. 結論

Task 04H-2G-Doc 已鎖定 LIFF 顧問獎金第一版的插入位置、總覽卡片、明細列表、篩選、狀態文案、欄位白名單與上線邊界，可作為 Task 04H-2H 前端實作依據。

目前只可進行測試環境的畫面與 read-only 串接實作；在正式 Apps Script 同步、重新部署及 `not_initialized` 安全空狀態驗收完成前，不得向顧問正式開放獎金入口。
