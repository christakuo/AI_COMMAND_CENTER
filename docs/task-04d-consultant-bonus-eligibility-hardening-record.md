# Task 04D：顧問獎金正式資格規則補強紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04D：顧問獎金正式資格規則補強 |
| 完成日期 | 2026-07-02 |
| 文件任務 | Task 04D-4：文件化顧問獎金正式資格規則補強 |
| Apps Script repo | `yuanxin-apps-script-webhook-clean` |
| 程式 PR 狀態 | 使用者已回報 Task 04D PR 已 merge，Apps Script repo `main` clean |
| 修改範圍 | `BonusService.gs` |

## 2. 補強目的

Task 04C 已證明顧問獎金 MVP 可在獨立測試 Sheet 完成 Preview、建立獎金明細表、產生草稿及防重驗證。Task 04D 進一步收緊正式獎金資格規則，避免不完整、官方歸屬或測試資料進入正式待審核獎金草稿。

## 3. 已完成範圍

- 僅修改 `BonusService.gs`。
- 收緊 `isEligiblePaidLeadForBonus()` 的資格規則。
- `generateConsultantBonusDrafts()` 同步套用相同資格規則。
- Preview 新增 `excludedCount` 與 `excludedSummary`。
- 新增明確排除原因，便於正式執行前人工核對。
- 原本以備註放行缺顧問、官方顧問 `HC0000` 或缺付款時間的行為已移除。
- Task 04C 測試資料仍符合新資格規則，可維持原測試邏輯通過。

## 4. 已收緊的資格規則

以下任一條件成立時，不得產生獎金草稿：

1. 名單 ID 空白。
2. 是否已付款不是「是」。
3. 付款完成時間、付款日期及成交日期全部空白。
4. 歸屬顧問 ID 空白。
5. 歸屬顧問 ID 為官方顧問 `HC0000`。
6. 是否測試資料為「是」。
7. 相同「名單 ID + 獎金類型」已有獎金紀錄。

只有通過上述條件的資料，才可列入 Preview 的預計建立清單，並在正式 Generate 時產生待審核獎金草稿。

## 5. Preview 新增欄位

| 欄位 | 用途 |
|---|---|
| `excludedCount` | 統計因資格規則或防重規則被排除的資料筆數 |
| `excludedSummary` | 依排除原因彙整筆數，供正式執行前人工審查 |

Preview 與 Generate 使用相同資格規則，避免 Preview 顯示可建立，但正式 Generate 卻以不同條件處理的落差。

## 6. 排除原因列表

| 排除原因 | 說明 |
|---|---|
| `MISSING_LEAD_ID` | 名單 ID 空白 |
| `NOT_PAID` | 名單尚未付款 |
| `MISSING_PAYMENT_COMPLETED_AT` | 付款完成時間、付款日期及成交日期全部空白 |
| `MISSING_CONSULTANT_ID` | 歸屬顧問 ID 空白 |
| `OFFICIAL_CONSULTANT_HC0000` | 歸屬顧問為官方顧問 `HC0000` |
| `TEST_DATA` | 是否測試資料為「是」 |
| `EXISTING_BONUS_RECORD` | 相同「名單 ID + 獎金類型」已有獎金紀錄 |

## 7. Preview 與 Generate 一致性

- `isEligiblePaidLeadForBonus()` 作為正式資格判斷基礎。
- Preview 使用收緊後的規則計算符合資格、預計建立及排除結果。
- `generateConsultantBonusDrafts()` 使用相同規則建立獎金草稿。
- 防重規則仍採用「名單 ID + 獎金類型」。
- 已有獎金紀錄會以 `EXISTING_BONUS_RECORD` 排除，不得重複建立草稿。

## 8. 安全確認

- 本次未部署 Apps Script。
- 本次未執行任何 Apps Script 函式。
- 本次未修改正式 Google Sheet。
- 本次未建立正式 `bonus_records_獎金明細表`。
- 本次未執行 `generateConsultantBonusDrafts()`。
- 本次文件任務未修改 Apps Script repo 或 LIFF repo。

## 9. 結論

Task 04D 已在正式 Generate 前完成獎金資格規則收緊。Preview 與 Generate 已使用相同資格規則，且 Preview 可透過 `excludedCount` 與 `excludedSummary` 呈現排除結果，降低不合格資料進入待審核獎金草稿的風險。

正式環境仍不可直接執行 `generateConsultantBonusDrafts()`。本次完成的是程式規則補強，不代表正式 Sheet 已建立、正式資料已完成核對或首批獎金草稿已獲准產生。

## 10. 後續待辦

1. 正式 Apps Script 編輯器同步最新 `BonusService.gs`。
2. 執行正式 Sheet 備份。
3. 正式 leads 補齊或確認付款完成時間欄位。
4. 單獨執行 `ensureBonusRecordsSheet()`，建立正式 `bonus_records_獎金明細表`。
5. 人工核對 19 欄 Header。
6. 執行正式 Preview，並逐筆人工核對 `excludedSummary` 與 `plannedCreateCount`。
7. 全部停止條件排除後，才可規劃首批待審核獎金草稿產生。
