# 90P-7Q Version 51 referral event linking 部署紀錄

## 1. 文件目的

本文件用來記錄：

- 90P-7Q referral event linking 程式已合併。
- Apps Script Version 51 已建立。
- 既有 Web App deployment 已切換到 Version 51。
- CRM 仍維持 false。
- 本文件不是 CRM true 開啟紀錄。
- 本文件不是正式健康評估測試紀錄。

## 2. 本次完成範圍

- PR：`#14 Add health assessment referral event linking`。
- Repo：`yuanxin-apps-script-webhook-clean`。
- Branch：`main`。
- Main 狀態：clean。
- Apps Script 版本：Version 51。
- Version 51 描述：`90P-7Q referral event linking CRM false`。
- Web App deployment：由 Version 50 切換到 Version 51。
- Web App URL：維持不變。
- CRM：`HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=false`。

## 3. 程式變更摘要

PR #14 修改檔案：

- `Config.gs`
- `EventService.gs`
- `HealthAssessmentCrmService.gs`
- `HealthAssessmentService.gs`

完成重點：

- 健康評估 CRM linking 成功建立或更新 Lead 後，已具備建立 referral event 的程式能力。
- `createdReferralEvent` 會反映事件實際建立結果。
- CRM false 時仍不建立 referral event。
- duplicate replay 仍在 CRM linking 前停止，不重複建立事件。
- LINE UID 自動建單未納入本階段。
- 未新增或修改 Google Sheet 欄位。

## 4. 部署狀態

- `.clasp.json` 為本機設定檔，已被 `.gitignore` 排除，不納入 Git。
- clasp push 檢查結果：遠端程式與本機一致，因此顯示 `Skipping push`，未重複上傳相同內容。
- Version 51 已建立。
- 既有正式 Web App deployment 已指向 Version 51。
- 未建立新 deployment。
- 未建立新 Web App URL。
- deployment 數量仍為 2。
- Version 50 保留，可作本次回切版本。
- Version 49 保留，作更早期緊急回切版本。

## 5. CRM false 下的正式行為

- 健康評估可以繼續寫入 `health_assessments_健康評估紀錄`。
- CRM false 時不建立或更新 Lead。
- CRM false 時不建立 referral event。
- CRM false 時不補建 90 天推薦保護。
- duplicate replay 不新增第二筆 health assessment。
- duplicate replay 在 CRM linking 前停止，不進入 Lead、推薦保護或 referral event 流程。

## 6. 未執行事項

- 未開 CRM true。
- 未修改 Google Sheet。
- 未送正式健康評估測試。
- 未建立新 Web App URL。
- 未建立新 deployment。
- 未切回 Version 50 或 Version 49。
- 未處理 LINE UID 自動建單。
- 未處理健檢報告分析系統。
- 未處理顧問獎金制度。

## 7. 回切資訊

- Version 50 保留，可作本次 Version 51 的回切版本。
- Version 49 保留，作更早期緊急回切版本。
- 若需回切，必須另開緊急回復任務。
- 不得在未記錄異常原因、影響範圍及回復決策的情況下自行回切。
- 回切後仍須確認沿用既有 deployment，且 Web App URL 不變。

## 8. 後續建議

- 下一階段若要開 CRM true，必須另開獨立任務。
- CRM true 任務應維持小範圍、可停止、可 rollback。
- 執行前必須確認內部驗收 token、手機白名單、官方顧問 `HC0000` 與核准測試案例。
- 必須依 `docs/90p-7p-crm-true-controlled-validation-plan.md` 執行。
- 不建議將 CRM true 與其他功能重構合併。

## 9. 禁止事項

- 不要再送 `HASUB-20260701-90p7otest1`。
- 不要刪除 `HA000016`。
- 不要改名或移除「送出識別碼」欄位。
- 不要回填舊 submissionId。
- 不要修改第 12 列 `LD000011`。
- 不要修改第 13 列 `LD000012`。
- 不要未經受控任務開 CRM true。
- 不要未經任務送正式健康評估測試。
- 不要建立新 Web App URL。
- 不要將 CRM true 與其他功能重構混在一起。
