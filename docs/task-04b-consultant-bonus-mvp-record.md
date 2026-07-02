# Task 04B：顧問獎金系統 MVP 完成紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04B：顧問獎金系統 MVP |
| 完成日期 | 2026-07-02 |
| Apps Script repo | `yuanxin-apps-script-webhook-clean` |
| Apps Script PR | #26（已 merge） |
| 本次文件任務 | Task 04B-5：顧問獎金 MVP 文件收尾 |

## 2. 已完成範圍

- 新增顧問獎金 MVP 後端基礎。
- 新增 `bonus_records_獎金明細表` 支援。
- 新增顧問推薦獎金常數：
  - 推薦獎金：4,000 元。
  - 獎金類型：會員推薦獎金。
  - 初始狀態：待審核。
  - 是否已付款：否。
  - 建立來源：`system_bonus_mvp`。
- 新增 BN 獎金 ID 支援，格式為 `BN000001`、`BN000002` 依序編號。
- 新增 Preview 唯讀函式。
- 新增正式產生待審核獎金草稿函式，但尚未執行。
- 防重規則採用「名單 ID + 獎金類型」。
- 手機號碼採遮罩處理，不寫入完整手機號碼。
- 官方顧問 `HC0000` 或未確認顧問歸屬不直接排除，但會加註備註供人工確認。
- `generateConsultantBonusDrafts()` 已具備 Script Lock 與防重機制。

## 3. Preview 驗收結果

本次只在 Apps Script 編輯器執行 `BonusService.gs` 的 `logPreviewConsultantBonusDrafts()`，結果如下：

```json
{
  "mode": "preview",
  "eligiblePaidLeadCount": 0,
  "plannedCreateCount": 0,
  "existingSkippedCount": 0,
  "createdCount": 0,
  "updatedCount": 0,
  "skippedCount": 0,
  "errorCount": 0,
  "errorSummary": {},
  "generatedAt": "2026-07-02 13:50:10"
}
```

驗收結論：本次 Preview 未發現符合條件的已付款名單、未規劃新增獎金草稿，且無錯誤；這是唯讀預覽結果，不代表正式獎金資料已建立。

## 4. 安全限制與未執行事項

- 未部署 Apps Script。
- 未執行正式寫入函式 `generateConsultantBonusDrafts()`。
- 未執行 `ensureBonusRecordsSheet()`。
- 未正式新增或寫入 `bonus_records_獎金明細表`。
- 未修改 Apps Script repo。
- 未修改 LIFF repo。
- 未修改健康評估文件。
- 未重開 90P-7T 測試。

## 5. 後續待辦

1. 正式 Sheet 確認或建立 `bonus_records_獎金明細表`。
2. 補明確付款完成時間／成交日期欄位。
3. 先用測試 Sheet 驗證 `ensureBonusRecordsSheet()` 與 `generateConsultantBonusDrafts()`。
4. 人工確認 Preview 結果後，再正式產生待審核獎金草稿。
5. 後續再做 `bonus_summary_顧問獎金彙總`。
6. 後續再做 `payout_records_獎金付款紀錄`。
7. 後續再做顧問大廳獎金顯示。
8. 團隊獎金、導師獎金、階層獎金暫不在 MVP。

## 6. 結論

Task 04B 已完成顧問獎金 MVP 後端基礎與唯讀 Preview 驗收。正式 Sheet 建立、正式獎金草稿產生及付款相關流程仍須依後續待辦逐步測試與人工確認，不得將本次 Preview 視為正式寫入或上線完成。
