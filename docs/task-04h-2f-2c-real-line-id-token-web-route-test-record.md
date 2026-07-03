# Task 04H-2F-2C：真實 LINE ID Token 單帳號 Web route 測試紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04H-2F-2C：真實 LINE ID Token 單帳號 Web route 測試結果文件化 |
| 前置實作 | Task 04H-2D、Task 04H-2D-Fix1 |
| 前置測試 | Task 04H-2E、Task 04H-2F-1、Task 04H-2F-2A、Task 04H-2F-2B |
| 測試性質 | 獨立測試環境、真實 LINE ID Token、Web App POST route |
| 結論 | 單帳號真實 ID Token 核心安全路徑通過 |

本文件只記錄使用者已完成的測試結果，不記錄 token、完整 LINE UID 或測試 Web App URL。

## 2. 測試環境與隔離狀態

- 使用獨立測試 Apps Script Web App deployment。
- 使用獨立測試 Sheet 副本：`元馨醫管家_Task04H-2F-2B_RealToken_Test`。
- 測試專案的 `APP_SPREADSHEET_ID` 指向測試 Sheet 副本。
- 測試使用真實 LINE ID Token。
- 測試顧問 `HC9001` 的 `LINE使用者ID` 已在測試 Sheet 改為測試者真實 LINE UID。
- 正式 Apps Script 未修改、未部署、未同步。
- 正式 Google Sheet 未修改。
- 正式 `bonus_records_獎金明細表` 未建立。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。

## 3. 有效 ID Token＋Dashboard 測試

### 測試條件

- 使用測試 LIFF 的 `liff.getIDToken()` 取得真實 ID Token。
- `action=getConsultantBonusDashboard`。
- `month=2026-07`。
- Token 對應測試顧問 `HC9001`。

### 測試結果

| 欄位 | 結果 |
|---|---|
| `success` | `true` |
| `setupStatus` | `ready` |
| `empty` | `false` |
| `consultantId` | `HC9001` |
| `consultantName` | 測試顧問A |
| `pendingReviewAmount` | `4000` |
| `approvedAmount` | `4000` |
| `paidAmount` | `4000` |
| `pendingReviewCount` | `1` |
| `approvedCount` | `1` |
| `paidCount` | `1` |
| `totalAmount` | `12000` |
| `totalCount` | `3` |
| `updatedAt` | `2026-07-20 15:30:00` |

### 判定

- 真實 LINE ID Token 驗證成功。
- 後端成功以 token 驗證結果對應到 `HC9001`。
- Dashboard 只統計 `HC9001` 在 2026-07 的獎金資料。

## 4. 有效 ID Token＋Records 測試

### 測試條件

- `action=getConsultantBonusRecords`。
- `lineIdToken` 使用 `liff.getIDToken()` 的結果。
- `lineAccessToken` 為空字串。
- `month=2026-07`。
- `status=all`。
- `limit=50`。

### 測試結果

- `success=true`。
- `setupStatus=ready`。
- `empty=false`。
- 共回傳 3 筆：`BN900003`、`BN900002`、`BN900001`。
- 未回傳 `HC9002` 的 `BN900006`、`BN900007`。
- `pagination.limit=50`。
- `nextPageToken=null`。

### Response 欄位白名單

每筆只包含：

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

### 判定

- 真實 ID Token＋Web App POST route＋本人獎金明細查詢通過。
- `HC9001` 未看到 `HC9002` 的資料。
- 欄位白名單正常。

## 5. Raw `lineUserId` 偽造測試

### 測試條件

- 使用 `HC9001` 的真實 ID Token。
- Request body 額外傳入另一個測試用 raw `lineUserId`。
- `action=getConsultantBonusDashboard`。
- `month=2026-07`。

### 測試結果

- `success=true`。
- 回傳顧問仍為 `HC9001`。
- `totalAmount=12000`。
- `totalCount=3`。

### 判定

- 後端沒有信任前端傳入的 raw `lineUserId`。
- 後端以真實 LINE ID Token 驗證後取得的 UID 為準。
- raw `lineUserId` 偽造無法改變查詢身分。

## 6. `consultantId` 越權拒絕測試

### 測試條件

- 使用 `HC9001` 的真實 ID Token。
- Request body 額外傳入 `consultantId=HC9002`。
- `action=getConsultantBonusDashboard`。
- `month=2026-07`。

### 實際結果

```json
{
  "success": false,
  "errorCode": "ACCESS_DENIED",
  "message": "無法查詢指定顧問的獎金資料"
}
```

### 判定

- 即使具有有效 LINE ID Token，只要前端指定 `consultantId`，後端即拒絕。
- Response 不包含 summary、records、顧問資料或獎金明細。

## 7. Token 缺失拒絕測試

### 測試條件

- 不傳 `lineIdToken`。
- 不傳 `lineAccessToken`。
- `action=getConsultantBonusDashboard`。
- `month=2026-07`。

### 實際結果

```json
{
  "success": false,
  "errorCode": "TOKEN_MISSING",
  "message": "無法確認顧問身分",
  "authCode": "TOKEN_MISSING"
}
```

### 判定

- 缺少 LINE ID Token 與 Access Token 時，Web route 正確拒絕。
- Response 未回傳任何顧問或獎金資料。

## 8. 無效 Token 拒絕測試

### 測試條件

- 傳入格式看似 token、但無效的測試 ID Token。
- `lineAccessToken` 為空字串。
- `action=getConsultantBonusDashboard`。
- `month=2026-07`。

### 實際結果

```json
{
  "success": false,
  "errorCode": "ACCESS_DENIED",
  "message": "無法確認顧問身分",
  "authCode": "LINE_ID_TOKEN_VERIFY_FAILED"
}
```

### 判定

- 無效 ID Token 被 LINE 驗證拒絕。
- Web route 未回傳任何顧問或獎金資料。

## 9. Web route `limit=0` 拒絕測試

### 測試條件

- 使用有效 ID Token。
- `action=getConsultantBonusRecords`。
- `month=2026-07`。
- `status=all`。
- `limit=0`。

### 實際結果

```json
{
  "success": false,
  "errorCode": "INVALID_ARGUMENT",
  "message": "查詢筆數必須是正整數"
}
```

### 判定

- Task 04H-2D-Fix1 不只在 Helper 測試通過，也在真實 Web route＋有效 ID Token 情境下通過。
- Response 不包含 records、summary 或任何獎金資料。

## 10. Sheet 不寫入／不自動建表確認

人工確認結果：

- 測試 Sheet 分頁沒有新增。
- 測試用 `bonus_records_獎金明細表` 仍維持原本 7 筆測試資料：`BN900001`～`BN900007`。
- 沒有新增第 8 筆資料。
- 沒有修改付款日期。
- 沒有修改獎金狀態。
- 沒有修改備註。
- 沒有修改顧問 ID。
- 沒有自動建立任何分頁。

判定：read-only Web route 沒有寫入 Sheet、沒有自動建表，也沒有產生獎金草稿。

## 11. 已通過項目總結

| 驗證項目 | 結果 |
|---|---|
| 真實 ID Token 驗證 | 通過 |
| Dashboard POST route | 通過 |
| Records POST route | 通過 |
| Token 對應本人顧問 | 通過 |
| 單帳號本人資料隔離 | 通過 |
| Raw `lineUserId` 偽造防護 | 通過 |
| `consultantId` 越權拒絕 | 通過 |
| Token 缺失拒絕 | 通過 |
| 無效 Token 拒絕 | 通過 |
| `limit=0` 拒絕 | 通過 |
| Response 欄位白名單 | 通過 |
| Sheet 不寫入 | 通過 |
| 不自動建表 | 通過 |

## 12. 尚未完成項目

- 雙帳號真實 LINE ID Token 隔離測試。
- `HC9002` 真實 token 對稱驗證。
- Access Token fallback channel 綁定補強。
- Access Token fallback 正式安全驗收。
- 正式 Apps Script 同步。
- 正式部署。
- 正式 Sheet 建表。
- LIFF 前端獎金畫面串接。

單帳號測試已證明 token 對應及偽造防護，但尚不能完全取代兩個真實 LINE 帳號互查隔離的對稱驗證。

## 13. 後續建議

1. Task 04H-2F-2D：Access Token fallback channel 綁定安全補強盤點。
2. Task 04H-2F-2E：雙帳號真實 ID Token 顧問隔離測試；可取得第二測試帳號時執行。
3. Task 04H-2F-3：完整文件化 Task 04H-2F Web route／token 測試結果。
4. Task 04H-2G：LIFF 顧問獎金畫面規格與串接。

在 Access Token fallback 安全邊界及必要隔離測試完成前，不應將本次單帳號測試視為正式環境全面開放核准。

## 14. 安全聲明

- 本次只文件化測試結果。
- 文件不記錄任何 LINE token。
- 文件不記錄完整 LINE UID。
- 文件不記錄測試 Web App URL。
- 未修改 Apps Script repo。
- 未修改 LIFF repo。
- 未部署。
- 未執行 Apps Script 函式。
- 未修改正式 Google Sheet。
- 未建立正式 `bonus_records_獎金明細表`。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。

## 15. 結論

Task 04H-2F-2C 已完成單帳號真實 LINE ID Token Web route 測試。有效 ID Token 的 Dashboard／Records、raw `lineUserId` 偽造防護、`consultantId` 越權拒絕、token 缺失／無效拒絕、`limit=0` 驗證、欄位白名單及 Sheet 不寫入均通過。正式環境仍未修改或部署，雙帳號及 Access Token fallback 驗證仍待後續任務完成。
