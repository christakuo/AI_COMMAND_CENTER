# Task 04H-2M-3｜隔離測試資料 dry-run 驗收設計

## 1. 任務背景

- Task 04H-2M-2 已完成並 merge。
- 本階段只設計隔離測試驗收，不執行實測。
- 不碰正式 Sheet。
- 不執行 Apps Script。
- 不部署。
- 不產生任何獎金資料，包括正式與測試獎金資料。

本文件用於規劃後續在隔離環境驗證 BonusService 的 preview（dry-run）與 generate 行為。所有操作均須等後續實測任務另行授權後，才可在測試副本執行。

## 2. 隔離測試原則

- 不使用正式 Sheet 直接測試。
- 僅使用副本 Sheet 或測試專用 Spreadsheet。
- 測試前須確認 Apps Script 的 Spreadsheet ID、資料來源與所有相關設定均指向測試資料。
- 不部署正式 Web App。
- 不修改正式 deployment，包括版本、部署設定與 Web App URL。
- 不開啟 `consultantBonusEnabled`。
- 測試資料須可辨識、可重建，且不得與正式資料混用。
- 任一設定無法確認為隔離環境時，立即停止，不執行 preview 或 generate。

## 3. 測試資料最小案例

每個案例應使用明確且唯一的名單 ID、顧問 ID、來源識別與日期；除衝突或重複案例外，不共用識別值。

| 類別 | 最小案例 | 預期結果 |
| --- | --- | --- |
| create | 有效名單、有效顧問、已付款，且付款／成交日期符合固定 `recognitionMonth` | `create` |
| update | 已存在狀態為待審核且未付款的草稿 | `update` |
| skip | 顧問為 `HC0000` 官方顧問 | `skip`，不得產生草稿 |
| skip | 測試資料 | `skip`，不得產生草稿 |
| skip | 未付款 | `skip`，不得產生草稿 |
| skip | 無付款日期且無成交日期 | `skip`，不得產生草稿 |
| skip | 日期不符合 `recognitionMonth` | `skip`，不得產生草稿 |
| skip | 已存在且已審核或已付款 | `skip`，不可覆寫 |
| error | 缺名單 ID | `error` |
| error | 缺顧問 ID | `error` |
| error | 日期無法解析 | `error` |
| error | 金額不是有效正數 | `error` |
| duplicate | 同一批次出現重複來源 | 僅允許一筆有效決策，其餘須以重複原因阻擋 |
| source conflict | 同一名單對應不同顧問，或同一名單出現不同來源衝突 | 阻擋並回報來源衝突，不得自行選擇或寫入 |

實測紀錄須保存每個案例的預期分類、實際分類、reason code、`dedupeKey` 與必要的差異說明；reason code 的實際常數名稱以 Task 04H-2M-2 已 merge 程式為準，不在本文件另造常數。

## 4. dry-run 驗收項目

preview／dry-run 必須同時符合以下條件：

- 全程不寫入 Sheet，執行前後資料列、儲存格值與工作表清單均無變化。
- 不呼叫 `ensureBonusRecordsSheet()`，且不得因工作表不存在而自動建表。
- 每筆輸入均回傳 `create`、`update`、`skip` 或 `error` 其中一種分類。
- 每筆均有 reason code；阻擋與錯誤案例不得只有自由文字。
- `reasonSummary` 的各 reason 筆數與逐筆結果一致，總數也須與輸入及分類統計相符。
- 相同標準化輸入與相同規則版本重跑時，`inputFingerprint` 必須穩定；輸入內容改變時應能反映差異。
- preview 的 `runId` 格式必須為 `BPREVIEW-...`。
- `dedupeKey` 必須依已封版規則產生；相同業務來源應一致，不同認列月份或不同合法來源應依規格正確區隔。
- `recognitionMonth` 缺漏時必須拒絕。
- `recognitionMonth` 不是 `YYYY-MM` 格式或不是有效月份時必須拒絕。

建議在 dry-run 前後保存測試 Spreadsheet 的工作表清單、各表列數及目標範圍快照，作為「未寫入、未建表」的比對證據。

## 5. generate 測試設計

generate 不屬於本次執行範圍；後續只有在另行授權的隔離實測任務中，才可依下列設計執行：

- 只允許在測試副本或測試專用 Spreadsheet 執行。
- 必須使用固定且明確的 `recognitionMonth`，不得使用系統當月或空值推定。
- generate 前先執行 preview，保存完整結果；generate 後再以相同輸入比對結果與寫入列。
- 實際 `create`／`update` 數量應與 generate 前的 preview 一致。
- `skip`／`error` 不得寫入任何獎金列。
- 寫入列須包含 `runId`、`dedupeKey`、`reason`、`recognitionMonth` 等 metadata，並符合 Task 04H-2M-2 的欄位與格式規則。
- 以相同輸入重跑同一月份時不得新增重複資料；既有可更新草稿也不得被無意義地重複更新。
- 已審核或已付款資料不得覆寫，包括金額、顧問、來源、狀態與 metadata。
- 部分失敗須可由 `runId`、逐筆結果、reason 與錯誤資訊追蹤；成功列與失敗列須可明確對應，不得整批失去稽核線索。
- generate 完成後再次 preview，應能以既有紀錄或防重 reason 正確反映已處理項目。

## 6. 驗收通過標準

全部條件同時成立才算通過：

- dry-run 未寫入任何 Sheet，也未建立工作表。
- generate 僅在副本寫入預期的 create／update 列，數量與 preview 一致。
- 相同月份與相同來源 rerun 具冪等性，不新增重複資料。
- 所有阻擋案例均有正確 reason，分類與 `reasonSummary` 一致。
- 已審核／已付款資料維持不變，衝突與錯誤資料沒有寫入。
- 無正式 Sheet、正式 deployment 或前端變更。
- `consultantBonusEnabled` 維持關閉。

## 7. 驗收失敗處理 SOP

任一案例與預期不符時：

1. 立即停止測試，不再執行後續 preview、generate 或 rerun。
2. 保留當下測試副本，不清除、不修飾失敗現場。
3. 記錄完整輸入、`recognitionMonth`、`runId`、`reasonSummary`、逐筆結果及預期／實際差異。
4. 標記是否有非預期寫入、重複列、覆寫或 metadata 缺漏，並保存前後快照。
5. 回到 Task 04H-2M-2 修正程式，重新走 code review 與 merge 流程。
6. 修正後使用新的隔離副本重新驗收，不沿用已受污染的測試結果。
7. 未完成修正與重新驗收前，不得進入正式資料、正式 Sheet 或正式 deployment。

## 8. 後續任務建議

- Task 04H-2M-4：隔離測試資料 dry-run／generate 實測紀錄。
- Task 04H-2I-Final：正式 `ready + empty` 驗收紀錄補件。
- Task 04H-2N：正式草稿受控產生與顧問查詢開放驗收。

建議執行順序為 04H-2M-4 → 04H-2I-Final → 04H-2N；任一前置驗收未通過時，不得推進正式草稿或顧問查詢開放。

## 本次變更邊界

- 僅新增本驗收設計文件並同步 `AI_COMMAND_CENTER` roadmap。
- 未修改 `yuanxin-liff-demo`。
- 未修改 `yuanxin-apps-script-webhook-clean`。
- 未執行 Apps Script、未修改 Google Sheet、未部署、未產生任何正式或測試獎金資料。
- 未修改 `consultantBonusEnabled`。

