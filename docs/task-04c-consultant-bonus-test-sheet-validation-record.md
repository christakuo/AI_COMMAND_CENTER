# Task 04C：顧問獎金 MVP 測試 Sheet 實寫驗證紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04C：顧問獎金 MVP 測試 Sheet 實寫驗證 |
| 完成日期 | 2026-07-02 |
| 文件任務 | Task 04C-3：文件化顧問獎金 MVP 測試 Sheet 實寫驗證結果 |
| 驗證結論 | 測試 Sheet 實寫驗證通過 |

## 2. 測試環境說明

| 項目 | 內容 |
|---|---|
| 測試 Spreadsheet | `元馨醫管家_Task04C_顧問獎金MVP測試` |
| 測試 Apps Script 專案 | `元馨醫管家_Task04C_顧問獎金MVP測試` |
| Google Sheet | 獨立測試 Sheet |
| Apps Script | 獨立測試專案 |
| 正式環境 | 未修改、未部署、未建立觸發器 |

本次驗證只在獨立測試環境執行，不影響正式 Apps Script 專案及正式 Google Sheet。

## 3. 測試資料

| 欄位 | 測試值 |
|---|---|
| 名單 ID | `TEST-BONUS-001` |
| 潛在會員姓名 | 測試會員A |
| 手機號碼 | `0912345678` |
| 推薦碼 | `HC9999` |
| 歸屬顧問 ID | `HC9999` |
| 歸屬顧問姓名 | 測試顧問 |
| 是否已付款 | 是 |
| 是否測試資料 | 否 |
| 付款完成時間 | 2026-07-02 14:00:00 |

## 4. Preview 結果

```json
{
  "mode": "preview",
  "eligiblePaidLeadCount": 1,
  "plannedCreateCount": 1,
  "existingSkippedCount": 0,
  "createdCount": 0,
  "updatedCount": 0,
  "skippedCount": 0,
  "errorCount": 0,
  "errorSummary": {},
  "generatedAt": "2026-07-02 14:48:21"
}
```

Preview 正確辨識 1 筆符合條件的已付款名單，規劃建立 1 筆獎金草稿，且沒有錯誤。

## 5. Ensure 結果

| 項目 | 結果 |
|---|---|
| 執行函式 | `BonusService.gs`／`ensureBonusRecordsSheet()` |
| 執行狀態 | 完成，無錯誤 |
| 建立結果 | 成功建立 `bonus_records_獎金明細表` |
| Header | 19 欄，內容正確 |

## 6. 第一次 Generate 結果

執行 `BonusService.gs`／`generateConsultantBonusDrafts()` 完成且無錯誤，成功新增一筆獎金草稿 `BN000001`。

| 重點欄位 | 實際結果 |
|---|---|
| 獎金 ID | `BN000001` |
| 名單 ID | `TEST-BONUS-001` |
| 會員姓名 | 測試會員A |
| 會員手機遮罩 | `0912***678` |
| 顧問 ID | `HC9999` |
| 顧問姓名 | 測試顧問 |
| 推薦碼 | `HC9999` |
| 成交日期 | 2026-07-02 14:00:00 |
| 成交狀態 | 付款完成 |
| 獎金類型 | 會員推薦獎金 |
| 獎金金額 | 4,000 元 |
| 獎金狀態 | 待審核 |
| 是否已付款 | 否 |
| 建立來源 | `system_bonus_mvp` |

第一次 Generate 的獎金 ID、歸屬、成交資料、獎金內容及手機遮罩均符合預期。

## 7. 第二次 Generate 防重結果

- 再次執行 `BonusService.gs`／`generateConsultantBonusDrafts()`，執行完成且無錯誤。
- 未新增 `BN000002`。
- `bonus_records_獎金明細表` 仍只有 `BN000001` 一筆獎金草稿。
- 防重規則「名單 ID + 獎金類型」驗證通過。
- 最後更新時間可能更新，屬於預期行為，不影響防重判定。

## 8. 安全確認

- 未修改正式 Apps Script 專案。
- 未修改正式 Google Sheet。
- 未修改 LIFF repo。
- 未部署。
- 未建立觸發器。
- 未執行任何健康評估 Case 函式。
- 本次文件任務未執行任何 Apps Script 函式。

## 9. 結論

Task 04C 顧問獎金 MVP 測試 Sheet 實寫驗證通過。Preview、Sheet 建立、首次獎金草稿產生與第二次執行防重皆符合預期；測試成功建立 `BN000001`，且未重複建立 `BN000002`。

此結果只證明獨立測試環境的實寫流程通過，不代表正式環境已核准執行或完成上線。

## 10. 後續待辦

1. 正式環境仍不建議直接執行 `generateConsultantBonusDrafts()`。
2. 需先補正式 leads 的付款完成時間／成交日期欄位策略。
3. 正式執行前需先跑 Preview 並人工確認。
4. 再規劃正式 `bonus_records_獎金明細表` 建立與首批草稿產生流程。
5. `bonus_summary`、`payout_records`、顧問大廳獎金顯示後續另開任務。
