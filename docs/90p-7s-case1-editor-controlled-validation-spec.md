# 90P-7S Case 1 Apps Script Editor 一次性受控驗收函式規格

## 1. 文件目的

- 本文件定義 90P-7S 後續要實作的一次性 Apps Script Editor 驗收函式。
- 目標是完成 90P-7R 尚未完成的 Case 1 驗收。
- 本文件不是 Case 1 驗收成功紀錄。
- 本文件不代表已修改 Apps Script。
- 本文件不代表已送出測試資料。

## 2. 背景

- 正式 Web App 已切換到 Version 52。
- Version 52 為 gated CRM true。
- 外部 Codex fetch 嘗試失敗，未取得 HTTP response。
- PowerShell 手動方式因本機腳本解析問題放棄。
- 正式 Sheet 未新增資料，沒有資料污染。
- Case 1 尚未完成。
- 不需立即回切 Version 51。

## 3. 原定 Case 1 驗收目標

- 無推薦碼新訪客 → 官方 `HC0000`。
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

## 4. 送出前基準

- `HASUB-20260701-90p7rcase1`：不存在。
- health_assessments 最後一筆：`HA000016`。
- leads 最後一筆：`LD000012`。
- referral_events 最後一筆：`EV000098`。
- `HA000016`、`LD000011`、`LD000012` 不得修改。

## 5. 建議函式

函式名稱：

```javascript
run90P7RCase1ControlledValidation()
```

- 函式放在正式 Apps Script repo 的 `Test.gs`。
- 函式由 Apps Script Editor 手動選擇並執行。
- 函式內部呼叫 `routeRequest(payload)`。
- 函式不經由外部 Web App fetch。
- 函式只允許執行 Case 1。
- 函式不自動重試。

## 6. 函式安全需求

1. 固定 submissionId：`HASUB-20260701-90p7rcase1`。
2. 執行前必須確認 submissionId 不存在。
3. 執行前必須確認目前最後 ID 仍為：
   - `HA000016`
   - `LD000012`
   - `EV000098`
4. 若任一最後 ID 已變動，必須停止，不得送出。
5. 若 submissionId 已存在，必須停止，不得重送。
6. 從 Script Properties 讀取：
   - `HEALTH_ASSESSMENT_INTERNAL_VALIDATION_TOKEN`
   - `HEALTH_ASSESSMENT_INTERNAL_TEST_PHONES`
7. 不得在程式中硬寫 token。
8. 不得在程式中硬寫完整手機。
9. 手機應取白名單中的第一筆，或由函式明確選擇一筆白名單測試手機；不得將完整手機寫入文件、Logger 或回傳摘要。
10. payload 的推薦碼必須固定為空白。
11. payload 不得使用 `HC9001`。
12. payload 的 `isTestData` 必須為 false。
13. payload mode 必須符合 gated CRM true 條件，例如 `internal_small_batch`。
14. 僅可呼叫 `routeRequest(payload)` 一次。
15. 不得自動 retry。
16. 不得修改既有資料；只允許正式流程新增 Case 1 預期資料。
17. 執行後必須核對 health_assessments、leads、referral_events 各只新增一筆。
18. 必須確認新 Lead 歸屬 `HC0000`。
19. 必須確認 `HA000016`、`LD000011`、`LD000012` 未修改。
20. 必須回傳完整但不含 token、完整手機或健康答案的驗收摘要。

## 7. 預期成功結果

- response status：success。
- duplicate：false。
- idempotentReplay：false。
- createdHealthAssessment：true。
- createdLead：true。
- updatedLead：false。
- createdReferralEvent：true。
- 新 assessmentId 預期為 `HA000017`。
- 新 leadId 預期為 `LD000013`。
- 新 eventId 預期為 `EV000099`。
- 新 Lead 顧問ID：`HC0000`。
- health_assessments 新增 1 筆。
- leads 新增 1 筆。
- referral_events 新增 1 筆。
- submissionId 只出現一次。
- 不需回切 Version 51。

上述新 ID 是依送出前基準推算的預期值；實作函式仍須以送出後正式 Sheet 的實際新增資料核對，不得只依推算判定成功。

## 8. 異常處理

- 任一前置條件不符，必須在寫入前停止。
- 若 `routeRequest` 回傳錯誤，不得再次送出。
- 若 response 不明，不得重試。
- 若 Sheet 寫入數量不符，立即停止並建議人工盤點。
- 若 Lead 未歸屬 `HC0000`，停止並建議評估是否回切 Version 51。
- 若 referral event 未建立，停止，不得補送相同 submissionId。
- 若 `HA000016`、`LD000011` 或 `LD000012` 被修改，停止並建議立即人工盤點。
- 不得進入 Case 2。

## 9. 實作範圍

- 下一階段只允許修改 Apps Script repo 的 `Test.gs`。
- 不修改 `Config.gs`。
- 不修改 `HealthAssessmentService.gs`。
- 不修改 `HealthAssessmentCrmService.gs`。
- 不修改 `ReferralService.gs`。
- 不修改 `EventService.gs`。
- 不修改 `LeadService.gs`。
- 不修改前端 repo。
- 不修改 Google Sheet 欄位。
- 不建立 Version 53。
- 不切換 Web App deployment。

## 10. 部署與執行方式

- 下一階段需要在 Apps Script repo 開 PR。
- PR merge 後需要 clasp push，讓 `Test.gs` 函式進入 Apps Script Editor。
- 不需要建立 Version 53。
- 不需要切換 Web App deployment。
- 執行方式是在 Apps Script Editor 手動選擇 `run90P7RCase1ControlledValidation()` 並執行一次。
- 執行後人工核對 Sheet 與 response。
- 執行後應另開文件化任務記錄驗收結果。

## 11. 禁止事項

- 不要再送 `HASUB-20260701-90p7otest1`。
- 不要刪除或修改 `HA000016`。
- 不要改名或移除「送出識別碼」欄位。
- 不要回填舊 submissionId。
- 不要修改 `LD000011`。
- 不要修改 `LD000012`。
- 不要重複執行 `run90P7RCase1ControlledValidation()`。
- 不要進入 Case 2。
- 不要未經任務修改 Google Sheet。
- 不要未經任務修改 Apps Script 業務邏輯。
- 不要未經任務建立新 Version 或切換 deployment。
- 不要未經任務回切 Version 51。

## 12. 結論

- 90P-7S 建議以 Apps Script Editor 一次性驗收函式完成 Case 1。
- 這是目前最安全、最可控的驗收方式。
- 下一階段可進入 `Test.gs` 最小實作。
- Version 52 保持 gated CRM true，不需立即回切。
