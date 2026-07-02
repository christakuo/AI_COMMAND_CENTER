# Task 04E：顧問獎金正式 Preview 紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04E：顧問獎金正式 Preview |
| 完成日期 | 2026-07-02 |
| 文件任務 | Task 04E-4：文件化顧問獎金正式 Preview 結果 |
| 執行日期時間 | 2026-07-02 15:57:56 |
| 執行環境 | 正式 Apps Script 專案 |
| 執行性質 | 唯讀 Preview |

## 2. 正式環境準備狀態

- 正式 Google Sheet 已建立副本備份。
- 正式 `leads_潛在會員名單` 已新增「付款完成時間」欄位。
- 「付款完成時間」位於「是否已付款」右側。
- 正式 Apps Script 已同步最新版 `BonusService.gs`。
- 本次只執行唯讀 Preview，未進行正式獎金資料寫入。

## 3. Preview 執行資訊

| 項目 | 內容 |
|---|---|
| 函式所在檔案 | `BonusService.gs` |
| 執行函式 | `logPreviewConsultantBonusDrafts()` |
| 執行專案 | 正式 Apps Script 專案 |
| 執行模式 | Preview／唯讀 |

## 4. 正式 Preview JSON 結果

```json
{
  "mode": "preview",
  "eligiblePaidLeadCount": 0,
  "plannedCreateCount": 0,
  "existingSkippedCount": 0,
  "createdCount": 0,
  "updatedCount": 0,
  "skippedCount": 0,
  "excludedCount": 1000,
  "excludedSummary": {
    "NOT_PAID": 1000,
    "MISSING_PAYMENT_COMPLETED_AT": 1000,
    "MISSING_CONSULTANT_ID": 987,
    "OFFICIAL_CONSULTANT_HC0000": 9,
    "TEST_DATA": 2,
    "MISSING_LEAD_ID": 984
  },
  "errorCount": 0,
  "errorSummary": {},
  "generatedAt": "2026-07-02 15:57:56"
}
```

## 5. 結果判定

| 判定項目 | 結果 |
|---|---|
| Preview 執行 | 成功 |
| 錯誤數 | `errorCount = 0` |
| 符合資格名單 | `eligiblePaidLeadCount = 0` |
| 預計建立草稿 | `plannedCreateCount = 0` |
| 排除筆數 | `excludedCount = 1000` |
| 是否可正式 Generate | 不可 |
| 是否需要建立正式 bonus_records | 目前不需要 |

目前沒有可產生正式獎金草稿的名單。`plannedCreateCount = 0` 是停止條件，不可執行 `generateConsultantBonusDrafts()`；也不需要先執行 `ensureBonusRecordsSheet()`，正式 `bonus_records_獎金明細表` 暫不建立。

`excludedSummary` 的各項原因可能同時出現在同一筆名單，因此原因筆數合計不等同於 `excludedCount`。本次結果顯示正式 leads 尚未具備可產生獎金草稿的完整付款與顧問歸屬條件。

## 6. 安全確認

- 本次未部署 Apps Script。
- 本次未執行 `ensureBonusRecordsSheet()`。
- 本次未執行 `generateConsultantBonusDrafts()`。
- 本次未建立正式 `bonus_records_獎金明細表`。
- 本次未產生任何正式獎金草稿。
- 本次文件任務未修改 Apps Script repo 或 LIFF repo。
- 本次文件任務未執行任何 Apps Script 函式。

## 7. 後續重新 Preview 條件

待出現正式成交名單，且至少同時符合以下條件後，才重新執行 Preview：

1. 「是否已付款」為「是」。
2. 「付款完成時間」有值。
3. 「歸屬顧問 ID」非空白。
4. 「歸屬顧問 ID」不是官方顧問 `HC0000`。
5. 名單不是測試資料，且尚未存在相同類型的獎金紀錄。

## 8. 後續待辦

1. 有正式付款完成名單後重新執行 Preview。
2. 人工確認 `plannedCreateCount` 與符合資格名單逐筆一致。
3. 只有在 `plannedCreateCount > 0` 且 `errorCount = 0` 時，才能規劃執行 `ensureBonusRecordsSheet()`。
4. `ensureBonusRecordsSheet()` 應單獨執行，並人工核對 19 欄 Header。
5. 首批正式 Generate 必須另開任務，不可直接執行。

## 9. 結論

Task 04E 正式 Preview 執行成功且無錯誤，但目前沒有符合正式獎金資格的名單，因此本次正確停在唯讀 Preview 階段。正式 `bonus_records_獎金明細表` 暫不建立，也不得執行正式獎金草稿產生。
