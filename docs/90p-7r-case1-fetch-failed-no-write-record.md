# 90P-7R Case 1 fetch failed 無寫入紀錄

## 1. 文件目的

- 本文件記錄 90P-7R 第五階段 Case 1 的一次受控送出嘗試。
- 本次嘗試沒有取得 HTTP response。
- 本次嘗試沒有新增 health_assessments、leads 或 referral_events。
- 本文件不是 Case 1 驗收成功紀錄。
- 本文件不是 CRM true 失敗判定。
- 本文件不代表需要回切 Version 51。

## 2. 當時正式狀態

- 正式 Web App：Version 52。
- Version 52 描述：`90P-7R gated CRM true`。
- Web App URL：維持不變。
- deployment ID：未改變。
- deployment 數量：仍為 2。
- CRM：gated true。
- Version 51：保留可回切。
- Version 50：保留可回切。
- Version 49：保留作更早期緊急回切。

## 3. Case 1 原定驗收目標

- 驗收情境：無推薦碼新訪客 → 官方 `HC0000`。
- submissionId：`HASUB-20260701-90p7rcase1`。
- 測試姓名：`90P7R測試官方歸屬`。
- 推薦碼：空白。
- `isTestData=false`。
- 同意聯絡：true。
- 使用內部驗收 token。
- 使用白名單手機。
- 不使用 `HC9001`。
- 不進入 Case 2。

本文件不記錄 token 或完整手機號碼。

## 4. 送出前人工基準

- `HASUB-20260701-90p7rcase1`：不存在。
- health_assessments 最後一筆：`HA000016`。
- leads 最後一筆：`LD000012`。
- referral_events 最後一筆：`EV000098`。

## 5. 實際結果

| 項目 | 結果 |
|---|---|
| 是否送出成功 | 否 |
| Response status | 無 HTTP 回應；`fetch failed` |
| duplicate | 無法取得 |
| idempotentReplay | 無法取得 |
| createdHealthAssessment | false；Sheet 未新增 |
| createdLead | false；Sheet 未新增 |
| updatedLead | false |
| createdReferralEvent | false；Sheet 未新增 |
| 新 assessmentId | 無 |
| 新 leadId | 無 |
| 新 eventId | 無 |
| 新 Lead 是否歸屬 `HC0000` | 未建立，無法驗證 |
| health_assessments 新增 | 0 筆 |
| leads 新增 | 0 筆 |
| referral_events 新增 | 0 筆 |
| 指定 submissionId | 仍不存在 |
| 是否送第二筆測試 | 否 |
| 是否進入 Case 2 | 否 |

## 6. Sheet 安全確認

- `HA000016` 未修改。
- `LD000011` 未修改。
- `LD000012` 未修改。
- health_assessments 沒有新增資料。
- leads 沒有新增資料。
- referral_events 沒有新增資料。
- 本次沒有資料污染。

## 7. 判斷

- 本次不是 Case 1 驗收成功。
- 本次也不能判定 CRM true 業務邏輯失敗。
- 因為沒有 HTTP response，也沒有 Sheet 寫入，較可能是送出端 fetch 環境無法連到正式 Web App endpoint。
- Version 52 gated CRM true 已部署，但 Case 1 尚未完成正式驗收。
- 因沒有資料污染，當下不需要立即回切 Version 51。

## 8. 後續建議

- 下一次 Case 1 必須另開受控驗收任務。
- 下一次可改用較穩定的送出方式，例如：
  - Apps Script 內部臨時驗收函式。
  - 已知可連線的前端端點。
  - 由具備可控網路環境的工具送出。
- 下一次送出前仍須重新確認：
  - submissionId 不存在。
  - health_assessments、leads、referral_events 的最後 ID。
  - 內部 token 與白名單手機仍有效。
- 不得直接進入 Case 2。
- 不得把本次 Case 1 視為已成功。

## 9. 禁止事項

- 不要再送 `HASUB-20260701-90p7otest1`。
- 不要刪除或修改 `HA000016`。
- 不要改名或移除「送出識別碼」欄位。
- 不要回填舊 submissionId。
- 不要修改 `LD000011`。
- 不要修改 `LD000012`。
- 不要未經受控任務重送 Case 1。
- 不要進入 Case 2。
- 不要未經任務修改 Google Sheet。
- 不要未經任務修改 Apps Script。
- 不要未經任務建立新 Version 或切換 deployment。
- 不要未經任務回切 Version 51。

## 10. 結論

- 90P-7R Case 1 目前狀態為「未完成、無寫入、無資料污染」。
- 正式 Web App 仍在 Version 52 gated CRM true。
- 不需立即回切。
- 需另開任務完成 Case 1 驗收。
