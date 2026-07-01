# 90P-7T Case 1 partial write failure 紀錄

## 1. 文件目的

- 本文件記錄 90P-7S Case 1 Apps Script Editor 驗收函式執行後的 partial write failure。
- 本文件不是 Case 1 驗收成功紀錄。
- 本文件不是 CRM 業務邏輯失敗定論。
- 本文件不代表需要立即回切 Version 51。

## 2. 當時正式狀態

- 正式 Web App：Version 52。
- CRM：gated true。
- Version 51、Version 50、Version 49：保留可回切。
- 不需立即回切 Version 51。

## 3. Case 1 執行結果

| 項目 | 結果 |
|---|---|
| status | `failed` |
| success | `false` |
| caseId | `90P-7S-CASE1` |
| submissionId | `HASUB-20260701-90p7rcase1` |
| responseStatus | `success` |
| duplicate | `false` |
| idempotentReplay | `false` |
| createdHealthAssessment | `true` |
| createdLead | `false` |
| updatedLead | `false` |
| createdReferralEvent | `false` |
| assessmentId | `HA000017` |
| leadId | 空白 |
| eventId | 空白 |
| leadConsultantId | 空白 |
| healthAssessmentsAdded | `1` |
| leadsAdded | `0` |
| referralEventsAdded | `0` |
| ha000016Unchanged | `true` |
| ld000011Unchanged | `true` |
| ld000012Unchanged | `true` |
| rollbackRequired | `true` |
| case2Executed | `false` |
| retryPerformed | `false` |
| routeError | 空白 |

## 4. Sheet 人工核對結果

- `health_assessments_健康評估紀錄` 最後一筆為 `HA000017`。
- `HA000017` 的 submissionId 為 `HASUB-20260701-90p7rcase1`。
- `leads_潛在會員名單` 最後一筆仍為 `LD000012`。
- `referral_events_推薦事件紀錄` 最後一筆仍為 `EV000098`。
- `HA000016` 未修改。
- `LD000011` 未修改。
- `LD000012` 未修改。

## 5. 判斷

- Case 1 未通過。
- 本次已新增 health_assessment，因此不得重跑同一 submissionId。
- Lead 未建立，因此 referral_event 未建立是合理的安全行為。
- 目前沒有證據顯示正式 CRM 業務邏輯壞掉。
- 最可能原因是驗收函式使用的白名單手機已存在於 leads，或測試姓名與該手機對應的既有 Lead 姓名不一致，因而觸發 CRM 安全阻擋，例如 `PHONE_NAME_MISMATCH` 或人工審核條件。
- 現有驗收摘要未完整保留 CRM 決策欄位，因此以上為初步根因判斷，不是業務邏輯錯誤定論。

## 6. 禁止事項

- 不要重跑 `HASUB-20260701-90p7rcase1`。
- 不要刪除 `HA000017`。
- 不要手動補 Lead。
- 不要手動補 referral_event。
- 不要進入 Case 2。
- 不要修改 `HA000016`。
- 不要修改 `LD000011`。
- 不要修改 `LD000012`。
- 不要未經任務回切 Version 51。

## 7. 下一步

- 下一步進入 90P-7T 第三階段。
- 只修改 Apps Script repo 的 `Test.gs`。
- 補完整 CRM 決策摘要。
- 增加前置檢查：Case 1B 手機不得存在於 leads。
- 使用新的 submissionId：`HASUB-20260701-90p7rcase1b`。
- 不需要 Version 53。
- 不需要切換 Web App deployment。
- 需要 PR；PR merge 後需要 clasp push，讓 Apps Script Editor 取得更新後的驗收函式。

## 8. 結論

- 90P-7S Case 1 結果為 partial write failure。
- 已新增 `HA000017`，但未新增 Lead 或 referral_event。
- 不可重跑同一 submissionId。
- 不需立即回切 Version 51。
- 需修正 `Test.gs` 後，以 Case 1B 完成驗收。
