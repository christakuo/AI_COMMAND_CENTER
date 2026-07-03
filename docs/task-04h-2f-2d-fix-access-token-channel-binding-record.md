# Task 04H-2F-2D-Fix1：Access Token Channel 綁定補強與測試結果紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04H-2F-2D-Fix1：Access Token Channel 綁定補強與測試結果文件化 |
| 完成日期 | 2026-07-03 |
| 程式修改範圍 | Apps Script `LineAuthService.gs` |
| 程式狀態 | 使用者回報修正 PR 已合併，Apps Script repo `main` clean |
| 本次工作 | 僅文件化既有修正與測試結果 |
| 結論 | Access Token fallback 已補上 Channel 綁定與完整 verify 閘門，既有測試通過 |

本文件不記錄任何 LINE token、完整 LINE UID、Channel ID 或測試 Web App URL。

## 2. 修正背景

Task 04H-2F-2D 安全盤點發現 Access Token fallback 原有以下缺口：

1. `verifyLineAccessTokenForPortal()` 未比較 Access Token verify 回應的 `client_id` 與系統設定的 `LINE_LOGIN_CHANNEL_ID`。
2. verify 失敗或 `expires_in` 無效時，流程仍可能繼續呼叫 LINE Profile endpoint。
3. 只要 Profile endpoint 成功回傳 `userId`，流程仍可能回傳 `LINE_AUTH_OK`。
4. 此缺口只影響 Access Token fallback；ID Token 驗證路徑不受影響。

風險是不同 LINE Login Channel 的 Access Token，或未完成有效性確認的 Access Token，可能被錯誤接受為顧問登入憑證。

## 3. 已完成的安全補強

使用者回報 Task 04H-2F-2D-Fix1 已只修改 Apps Script `LineAuthService.gs`，並完成以下補強：

1. Access Token 驗證前先讀取 `LINE_LOGIN_CHANNEL_ID`。
2. 未設定 `LINE_LOGIN_CHANNEL_ID` 時立即拒絕。
3. LINE verify endpoint HTTP 非 200 時立即拒絕。
4. verify 回應不是有效 JSON 時立即拒絕。
5. verify 回應缺少 `client_id` 時立即拒絕。
6. `client_id` 與 `LINE_LOGIN_CHANNEL_ID` 不一致時立即拒絕。
7. `expires_in` 缺少、無效或小於等於 0 時立即拒絕。
8. `scope` 不含 `profile` 時立即拒絕。
9. 只有全部 verify 條件通過後，才可呼叫 LINE Profile endpoint。
10. Profile 成功且取得有效 `userId` 後，才回傳 `LINE_AUTH_OK`。
11. 錯誤回應不洩漏 token、`client_id`、完整 verify／Profile 回應或 stack trace。

`verifyLineIdTokenForPortal_()` 未修改；驗證順序仍是優先使用 ID Token，只有 ID Token 缺失或驗證失敗時才進入 Access Token fallback。

## 4. Access Token 驗證結果代碼

| 代碼 | 意義 |
|---|---|
| `LINE_ACCESS_TOKEN_CONFIG_MISSING` | 系統缺少必要的 LINE Login Channel 設定 |
| `LINE_ACCESS_TOKEN_VERIFY_FAILED` | LINE verify endpoint 呼叫失敗 |
| `LINE_ACCESS_TOKEN_VERIFY_INVALID` | verify 回應格式無效 |
| `LINE_ACCESS_TOKEN_CLIENT_ID_MISSING` | verify 回應缺少 `client_id` |
| `LINE_ACCESS_TOKEN_CHANNEL_MISMATCH` | Access Token 所屬 Channel 與系統設定不一致 |
| `LINE_ACCESS_TOKEN_EXPIRED` | `expires_in` 缺少、無效或已失效 |
| `LINE_ACCESS_TOKEN_SCOPE_INVALID` | Access Token 未具備 `profile` scope |
| `LINE_PROFILE_FAILED` | LINE Profile endpoint 呼叫失敗 |
| `LINE_PROFILE_INVALID` | Profile 回應無有效 `userId` |
| `LINE_AUTH_OK` | Token 驗證與 Profile 身分取得均成功 |

## 5. 測試環境

- 使用獨立測試 Apps Script 專案及獨立測試 deployment。
- 使用測試 Sheet 副本：`元馨醫管家_Task04H-2F-2B_RealToken_Test`。
- 測試專案的 `APP_SPREADSHEET_ID` 指向測試 Sheet 副本。
- 正式 Apps Script 未同步、未部署。
- 正式 Google Sheet 未修改。
- 文件不保存任何測試憑證或完整身分識別值。

## 6. ID Token 回歸測試

### 測試條件

- 使用有效的 `liff.getIDToken()` 結果。
- `lineAccessToken` 留空。
- 查詢 Dashboard，月份為 `2026-07`。

### 測試結果

| 項目 | 結果 |
|---|---|
| 查詢狀態 | 成功，`ready` |
| 顧問 | `HC9001`／測試顧問A |
| 待審核 | 4,000 元／1 筆 |
| 已核准 | 4,000 元／1 筆 |
| 已付款 | 4,000 元／1 筆 |
| 合計 | 12,000 元／3 筆 |

判定：Access Token 修正未破壞既有 ID Token 路徑，且系統仍優先使用 ID Token。

測試過程曾因既有 ID Token 已過期而回傳 `LINE_ID_TOKEN_VERIFY_FAILED`；重新登入取得新 token 後通過。此結果符合 token 到期的預期安全行為，不判定為程式缺陷。

## 7. 同 Channel Access Token only 測試

### 測試條件

- `lineIdToken` 留空。
- 使用相同 LINE Login Channel 的 `liff.getAccessToken()` 結果。
- 查詢 Dashboard。

### 測試結果

| 項目 | 結果 |
|---|---|
| 查詢狀態 | 成功，`ready` |
| 顧問 | `HC9001`／測試顧問A |
| 合計 | 12,000 元／3 筆 |

判定：同一 LINE Login Channel 的 Access Token fallback 可正常運作；`client_id`、有效 `expires_in`、`profile` scope 與 Profile `userId` 均通過驗證。

## 8. Bearer 前綴拒絕測試

### 測試條件

將 `lineAccessToken` 傳成含 `Bearer ` 前綴的字串，而不是系統預期的原始 token 值。

### 實際結果

```json
{
  "success": false,
  "errorCode": "ACCESS_DENIED",
  "message": "無法確認顧問身分",
  "authCode": "TOKEN_FORMAT_INVALID"
}
```

判定：格式錯誤的 Access Token 正確遭拒，且 response 未回傳 summary、records 或其他獎金資料。

## 9. Sheet 不寫入／不自動建表確認

人工確認結果：

- 測試 Sheet 沒有新增分頁。
- 測試用 `bonus_records_獎金明細表` 仍維持原本 7 筆測試資料，沒有新增第 8 筆。
- 沒有修改獎金狀態、付款日期、備註或顧問 ID。
- Web route 沒有自動建立任何分頁。
- 未產生獎金草稿。

判定：Access Token fallback 修正後仍維持 read-only 邊界。

## 10. 測試結果總結

| 驗證項目 | 結果 |
|---|---|
| ID Token 回歸 | 通過 |
| ID Token 優先順序 | 通過 |
| 同 Channel Access Token only | 通過 |
| `client_id` 綁定 | 已補強；同 Channel 正向測試通過 |
| `expires_in > 0` 閘門 | 已補強 |
| `profile` scope 閘門 | 已補強 |
| verify 完成前不得呼叫 Profile | 已補強 |
| Bearer 前綴拒絕 | 通過 |
| 不回傳獎金資料於拒絕 response | 通過 |
| Sheet 不寫入 | 通過 |
| 不自動建表 | 通過 |

## 11. 尚未完成的驗證

- 雙帳號真實 LINE ID Token 資料隔離測試。
- `HC9002` 真實 token 的對稱驗證。
- 不同 LINE Login Channel 的真實 Access Token 拒絕測試。
- 正式 Apps Script 同步與部署。
- 正式 Google Sheet 建表。
- LIFF 顧問獎金畫面串接。

同 Channel 正向測試可證明 fallback 正常運作，但不能取代跨 Channel 的負向測試；因此目前不可宣稱跨 Channel 實測已完成。

## 12. 後續建議

1. Task 04H-2F-3：彙整 Web route、ID Token 與 Access Token 安全測試結果。
2. Task 04H-2G：完成 LIFF 顧問獎金畫面與資料狀態規格。
3. Task 04H-2H：實作 LIFF 顧問獎金查詢畫面。
4. Task 04H-2I：規劃正式同步，先驗證正式環境 `not_initialized` 狀態。
5. 可取得第二個測試帳號時，補做雙帳號真實 ID Token 隔離測試。
6. 可取得另一個 LINE Login Channel 測試 token 時，補做跨 Channel Access Token 拒絕測試。

正式同步、正式部署及正式建表應維持獨立任務，不因本次測試通過而直接執行。

## 13. 安全聲明

- 本次只文件化使用者已回報的程式修正與測試結果。
- 未在 AI_COMMAND_CENTER 修改任何程式碼。
- 未修改 Apps Script repo。
- 未修改 LIFF repo。
- 未同步或部署正式 Apps Script。
- 未執行 Apps Script 函式。
- 未修改正式 Google Sheet。
- 未建立正式 `bonus_records_獎金明細表`。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。
- 未在文件中記錄 token、完整 LINE UID、Channel ID 或測試 Web App URL。

## 14. 結論

Task 04H-2F-2D-Fix1 已補強 Access Token fallback 的 LINE Login Channel 綁定與完整 verify 閘門。既有 ID Token 回歸、同 Channel Access Token only、Bearer 前綴拒絕及 Sheet 不寫入均通過。正式環境仍未同步或部署；雙帳號隔離與跨 Channel Access Token 拒絕仍需在具備獨立測試條件後補驗。
