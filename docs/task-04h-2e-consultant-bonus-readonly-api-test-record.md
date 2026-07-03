# Task 04H-2E：顧問獎金 read-only API 測試環境驗證紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04H-2E：顧問獎金 read-only API 測試環境驗證結果文件化 |
| 前置實作 | Task 04H-2D：Apps Script 顧問獎金 read-only API |
| 修正任務 | Task 04H-2D-Fix1：invalid limit 驗證修正 |
| 測試方式 | 獨立測試環境＋測試 Helper |
| 本階段結論 | Helper 路線核心邏輯測試通過；Web route／LINE token 尚未驗證 |

Task 04H-2D 與 Task 04H-2D-Fix1 的程式 PR 均已 merge，Apps Script repo `main` clean。本文件只記錄使用者回報的測試結果，不修改任何程式或正式環境。

## 2. 測試環境

| 項目 | 測試設定 |
|---|---|
| 測試 Spreadsheet | `元馨醫管家_Task04H-2E_顧問獎金API測試` |
| 測試 Apps Script | `元馨醫管家_Task04H-2E_Bonus_ReadOnly_API_Test` |
| `APP_SPREADSHEET_ID` | 指向獨立測試 Spreadsheet |
| `OFFICIAL_CONSULTANT_ID` | `HC9000` |
| 正式 Spreadsheet | 未使用、未修改 |
| 正式 Apps Script | 未部署、未同步、未執行 |

## 3. Helper 測試路線與邊界

本次使用測試專案內的 `TestBonusReadOnlyHelper.gs`：

- Helper 只存在獨立測試 Apps Script 專案。
- Helper 不加入 Apps Script repo。
- Helper 不部署至正式環境。
- Helper 不寫入 Sheet。
- Helper 不呼叫 `ensureBonusRecordsSheet()`。
- Helper 不呼叫 `generateConsultantBonusDrafts()`。
- Helper 直接測試 read-only 核心服務，會繞過 Web route 與真實 LINE token 驗證入口。

因此本次可證明查詢、統計、隔離、過濾與白名單核心邏輯，但不能取代 Web App 路由及 LINE 身分安全測試。

## 4. `not_initialized` 測試

### 測試條件

- 測試 Spreadsheet 起初只建立 `consultants_顧問主檔`。
- 未建立 `bonus_records_獎金明細表`。
- 執行 `run04H2E_NotInitialized_All_HC9001`。

### 通過結果

| 驗證項目 | 結果 |
|---|---|
| Dashboard `success` | `true` |
| Dashboard `setupStatus` | `not_initialized` |
| Dashboard `empty` | `true` |
| 顧問 ID | `HC9001` |
| Dashboard summary | 所有金額與筆數為 `0` |
| Dashboard message | 獎金資料建置中，尚未開放查詢 |
| Records `success` | `true` |
| Records `setupStatus` | `not_initialized` |
| Records `empty` | `true` |
| Records | `[]` |
| `pagination.limit` | `50` |
| `nextPageToken` | `null` |
| 測試後 bonus Sheet 是否存在 | `false` |

### 實際回報重點

```json
{
  "caseName": "Task 04H-2E not_initialized",
  "passed": true,
  "dashboard": {
    "success": true,
    "setupStatus": "not_initialized",
    "empty": true,
    "consultant": {
      "consultantId": "HC9001",
      "consultantName": "測試顧問A"
    },
    "summary": {
      "month": "2026-07",
      "pendingReviewAmount": 0,
      "approvedAmount": 0,
      "paidAmount": 0,
      "pendingReviewCount": 0,
      "approvedCount": 0,
      "paidCount": 0,
      "totalAmount": 0,
      "totalCount": 0
    },
    "updatedAt": "",
    "message": "獎金資料建置中，尚未開放查詢"
  },
  "records": {
    "success": true,
    "setupStatus": "not_initialized",
    "empty": true,
    "records": [],
    "pagination": {
      "limit": 50,
      "nextPageToken": null
    },
    "message": "獎金資料建置中，尚未開放查詢"
  },
  "bonusSheetExistsAfterRun": false
}
```

`not_initialized` 雖回傳全 0 summary，前端仍必須優先判斷 `setupStatus`，顯示「獎金資料建置中，尚未開放查詢」，不得把這些 0 顯示成已完成計算的 0 元。

## 5. 測試資料統計、顧問隔離與白名單測試

### 測試資料

- 人工建立測試用 `bonus_records_獎金明細表`。
- 人工貼入 19 欄 MVP Header。
- 人工貼入 7 筆測試資料：
  - `HC9001`：`BN900001`、`BN900002`、`BN900003`、`BN900004`、`BN900005`。
  - `HC9002`：`BN900006`、`BN900007`。
- 測試資料刻意包含完整會員姓名、完整或遮罩手機、付款批次 ID 與內部備註，以驗證 API 不外洩敏感欄位。
- 執行 `run04H2E_Data_All`。

### `HC9001`／2026-07 統計結果

| 統計項目 | 結果 |
|---|---:|
| 待審核金額 | 4,000 |
| 已核准金額 | 4,000 |
| 已付款金額 | 4,000 |
| 待審核筆數 | 1 |
| 已核准筆數 | 1 |
| 已付款筆數 | 1 |
| 總金額 | 12,000 |
| 總筆數 | 3 |

```json
{
  "success": true,
  "setupStatus": "ready",
  "empty": false,
  "consultant": {
    "consultantId": "HC9001",
    "consultantName": "測試顧問A"
  },
  "summary": {
    "month": "2026-07",
    "pendingReviewAmount": 4000,
    "approvedAmount": 4000,
    "paidAmount": 4000,
    "pendingReviewCount": 1,
    "approvedCount": 1,
    "paidCount": 1,
    "totalAmount": 12000,
    "totalCount": 3
  },
  "updatedAt": "2026-07-20 15:30:00",
  "message": ""
}
```

### 顧問隔離結果

- `HC9001` 只看到 `BN900001`、`BN900002`、`BN900003`。
- `HC9001` 看不到 `HC9002` 的 `BN900006`、`BN900007`。
- `HC9002` 只看到 `BN900006`、`BN900007`。
- `HC9002` 看不到 `HC9001` 的 `BN900001`、`BN900002`、`BN900003`。

### Response 白名單

每筆紀錄只包含：

- `bonusId`
- `recognizedDate`
- `bonusType`
- `amount`
- `status`
- `paidStatus`
- `paidAt`
- `sourceSummary`
- `updatedAt`

確認未回傳：

- 完整會員姓名。
- 完整手機。
- LINE UID。
- 付款批次 ID。
- 內部備註。
- 顧問 ID。
- 其他非白名單 Sheet 欄位。

## 6. Status／Month／Limit filter 測試

執行 `run04H2E_Filters_All`，結果如下：

| 測試條件 | 通過結果 |
|---|---|
| `pending_review` | 只回傳 `BN900001` |
| `approved` | 只回傳 `BN900002` |
| `paid` | 只回傳 `BN900003` |
| `month=2026-06` | 只回傳 `BN900004` |
| `month=2026-05` | 只回傳 `BN900005` |
| 成交日期空白 | 使用建立時間作為認列日期來源 |
| `month=2026-08` | 無資料，`empty=true` |
| 未傳 `limit` | `pagination.limit=50` |
| `limit=20` | `pagination.limit=20` |
| `limit=150` | 上限縮為 `pagination.limit=100` |
| 分頁 token | `nextPageToken=null` |

### `pending_review` 實際回報重點

```json
{
  "success": true,
  "setupStatus": "ready",
  "empty": false,
  "records": [
    {
      "bonusId": "BN900001",
      "recognizedDate": "2026-07-01",
      "bonusType": "會員推薦獎金",
      "amount": 4000,
      "status": "待審核",
      "paidStatus": "否",
      "paidAt": "",
      "sourceSummary": "會員推薦案（尾碼111）",
      "updatedAt": "2026-07-01 11:00:00"
    }
  ],
  "pagination": {
    "limit": 50,
    "nextPageToken": null
  },
  "message": ""
}
```

## 7. Invalid limit 問題與 Fix1 驗證

### 原始問題

- 執行 `run04H2E_InvalidLimit_HC9001`。
- 傳入 `limit=0`。
- 預期回傳 `INVALID_ARGUMENT`。
- 實際曾回傳 `success=true` 並包含 records。
- 判定為 Task 04H-2D 的 limit 驗證問題。

### 原因

原本使用 `normalizeText(options.limit)`，會把數值 `0` 視為空值，導致 `limit=0` 被誤認為未填並套用預設值 50。

### 修正

- Task 04H-2D-Fix1 新增 `normalizeConsultantBonusLimit_()`。
- 明確區分未填、有效正整數及無效數值。
- 修正 PR 已 merge，Apps Script repo `main` clean。

### 修正後結果

```json
{
  "success": false,
  "errorCode": "INVALID_ARGUMENT",
  "message": "查詢筆數必須是正整數"
}
```

- Response 不包含 records。
- Response 不包含 summary。
- Response 不包含任何獎金資料。
- Data／Filters 回歸測試通過。

## 8. 尚未驗證項目

本階段採用 Helper 測試，會繞過 Web Token 入口，因此尚未驗證：

- 真實 LINE ID Token 驗證。
- 真實 LINE Access Token 驗證。
- Web App `POST` route。
- `GET` 請求拒絕。
- Request body 帶入 `consultantId` 時是否回傳 `ACCESS_DENIED`。
- Token 缺失時是否由 Web route 回傳 `TOKEN_MISSING`。
- Web route 是否完整移除非白名單欄位。
- Web route 的錯誤是否安全處理且不洩漏內部資訊。

這些項目涉及正式安全邊界，需在下一階段使用 Web App＋LINE token，或先以 mock route 進行受控測試。

## 9. 結論

Task 04H-2E Helper 路線測試通過：

- read-only API 核心邏輯通過。
- `not_initialized` 行為通過。
- 不自動建立 `bonus_records` 行為通過。
- 顧問資料隔離通過。
- 金額統計通過。
- status／month／limit filter 通過。
- Response 白名單及敏感資料阻擋通過。
- invalid limit 問題已發現，並由 Task 04H-2D-Fix1 修正與回歸驗證通過。

尚未完成 Web route／LINE token 測試，因此不能把本次結果視為完整端到端身分驗證通過或正式部署核准。

目前仍維持：

- 未部署正式 Apps Script。
- 未同步正式 Apps Script。
- 未修改正式 Google Sheet。
- 未建立正式 `bonus_records_獎金明細表`。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。

## 10. 後續建議

### Task 04H-2F：Web route／LINE token 測試

建議先做 04H-2F，再進 LIFF 個人獎金畫面實作。原因是 LINE token、顧問身分推導、POST route 與越權拒絕是個人獎金資料的核心安全邊界；若先做前端，可能在串接階段才發現 route 合約或權限處理不一致。

建議至少驗證：

1. `POST`＋有效 LINE token 可取得本人資料。
2. 缺 token 回傳 `TOKEN_MISSING`。
3. 無對應顧問回傳 `CONSULTANT_NOT_FOUND`。
4. Request body 帶 `consultantId` 時拒絕或忽略，且不能改變查詢對象；依既定合約驗證 `ACCESS_DENIED`。
5. `GET` 被拒絕。
6. Web Response 維持欄位白名單。

若取得真實 LINE token 太困難，可先使用 mock route 驗證路由、方法限制、錯誤碼與「不得接受 consultantId」。但 mock route 不能證明 LINE token 真實驗證完成；在正式同步或開放 LIFF 前，仍須至少完成一次受控的真實 token 端到端測試。

### 後續順序

1. Task 04H-2F：顧問獎金 read-only API Web route／LINE token 測試規劃或 mock 測試。
2. Task 04H-2G：LIFF 顧問獎金卡片與獎金明細畫面實作規格。
3. Task 04H-2H：LIFF 顧問獎金卡片與明細畫面實作。
4. Task 04H-2I：正式 Apps Script 同步與 `not_initialized` 空狀態驗收。

## 11. 安全聲明

- 本次只文件化測試結果。
- 未修改 Apps Script repo。
- 未修改 LIFF repo。
- 未部署。
- 未執行 Apps Script 函式。
- 未修改正式 Google Sheet。
- 未建立正式 `bonus_records_獎金明細表`。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。
