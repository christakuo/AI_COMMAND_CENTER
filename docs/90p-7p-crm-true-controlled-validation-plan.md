# 90P-7P CRM true 受控驗收計畫

## 1. 文件目的

- 本文件是未來開啟 `HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=true` 前的受控驗收計畫。
- 本文件不是開啟 CRM true 的指令。
- 本文件不是部署紀錄。
- 本文件不代表目前已開啟 CRM true。
- 目前正式狀態仍是 CRM false。

## 2. 目前正式狀態

- Web App：Version 50。
- CRM 寫入開關：`HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=false`。
- Version 49：保留作緊急回切。
- `health_assessments_健康評估紀錄` 已有「送出識別碼」欄位。
- `leads_潛在會員名單` 未新增 submissionId 欄位。
- `referral_events_推薦事件紀錄` 未新增 submissionId 欄位。
- 舊資料 submissionId 空白是既有狀態，不回填。
- 90P-7O 已完成 submissionId／idempotency 防重送並通過正式最小驗收。
- 90P-7P 第二階段已建立正式流量監控與 CRM true 前置盤點文件。

## 3. 開啟 CRM true 前置條件

以下條件必須全部確認完成，才可另案評估受控開啟 CRM true：

- Version 50 已完成 3～7 天正式流量觀察。
- Version 50 上線後的新資料 submissionId 沒有空白。
- submissionId 沒有重複寫入。
- duplicate／idempotentReplay 行為符合預期。
- ambiguous success 不造成重複寫入。
- CRM false 期間，leads 沒有因健康評估異常新增或更新。
- CRM false 期間，referral_events 沒有因健康評估異常新增。
- 沒有出現 `HEALTH_ASSESSMENT_SUBMISSION_ID_COLUMN_MISSING`。
- Lead 新增與更新規則已重新確認。
- 既有 Lead 不會被錯誤覆蓋。
- 顧問歸屬規則已確認。
- 90 天推薦保護補建邏輯已確認。
- referral event 建立條件已確認。
- 測試資料與內部測試案例已備妥，且不使用真實客戶資料。
- rollback SOP 已完成並指定執行人與決策人。
- 必須另開 Apps Script PR。
- 必須建立新的 Apps Script 版本。
- 必須受控部署，不可直接在 Version 50 全量開啟 CRM true。

## 4. 驗收目標

CRM true 受控驗收需要證明：

- 健康評估送出後可正確新增一筆 health assessment。
- 符合條件時可正確建立新 Lead。
- 符合條件時可正確更新既有 Lead。
- 有效顧問推薦碼可正確歸屬對應顧問。
- 無推薦碼時可正確歸屬官方帳號 `HC0000`。
- 既有 Lead 的顧問歸屬不會被錯誤覆蓋。
- 90 天推薦保護可依規則正確建立、維持或不變更。
- referral event 可依已確認規則建立，且不遺漏、不重複。
- duplicate replay 不會重複新增 health assessment。
- duplicate replay 不會重複建立或更新 Lead。
- duplicate replay 不會重複建立 referral event。
- 發生錯誤時能立即停止，並依 rollback SOP 回復安全狀態。

## 5. 驗收範圍

### 本計畫涵蓋

- 健康評估送出。
- submissionId idempotency。
- health assessment 寫入。
- Lead 新增。
- Lead 更新。
- 顧問歸屬。
- 90 天推薦保護。
- referral event。
- duplicate replay。
- rollback 條件。

### 本計畫不涵蓋

- 健檢報告分析系統重構。
- 顧問獎金制度。
- 顧問 CRM 聯繫管理。
- 顧問資源中心優化。
- LINE 官方帳號選單調整。
- 企業健檢／ESG 功能。
- 會員健康管理方案重構。

## 6. 建議受控驗收案例

以下均為未來驗收案例草案。姓名、手機、submissionId 與推薦碼應於正式執行任務中由負責人填寫，且只能使用已核准的內部測試資料。

| 案例編號 | 情境 | 輸入資料 | 預期 health assessment | 預期 Lead | 預期 referral event | 預期 90 天推薦保護 | 驗收重點 |
|---|---|---|---|---|---|---|---|
| C01 | 無推薦碼的新訪客 | 內部測試姓名／手機待填；新 submissionId 待填；推薦碼空白 | 新增一筆並回傳新 assessmentId | 建立一筆新 Lead，歸屬 `HC0000` | 依驗收前已確認規則建立一次或不建立；結果須與規格一致 | 依官方帳號規則處理，不得誤套一般顧問保護 | 新訪客建單及官方帳號歸屬正確 |
| C02 | 有有效推薦碼的新訪客 | 內部測試姓名／手機待填；新 submissionId 待填；已核准測試顧問碼待填 | 新增一筆並回傳新 assessmentId | 建立一筆新 Lead，歸屬推薦碼對應顧問 | 依已確認規則建立一次，不得重複 | 正確建立 90 天推薦保護及期間欄位 | 推薦碼、顧問歸屬及保護期限一致 |
| C03 | 既有 Lead 再次評估 | 已核准既有內部測試 Lead；新 submissionId 待填；推薦碼依案例設定 | 新增一筆並回傳新 assessmentId | 只更新允許欄位，不重設 CRM 階段，不錯誤覆蓋既有顧問歸屬 | 依已確認的既有 Lead 規則建立一次或不建立 | 既有有效保護應維持，不得被錯誤縮短、延長或改派 | 核對更新前後欄位差異及顧問歸屬 |
| C04 | 相同 submissionId duplicate replay | 使用 C01～C03 中已完成案例的相同 submissionId；不使用既有 90P-7O 測試值 | 不新增第二筆，回傳原 assessmentId | 不新增、不再次更新 | 不新增第二筆 | 不重算、不補建、不變更 | 防重送必須在 CRM linking 前停止 |
| C05 | 無效推薦碼或身分／名單衝突 | 內部測試姓名／手機待填；新 submissionId 待填；無效推薦碼或預先設計的衝突資料 | 健康評估仍依安全規則保存一筆 | 不得錯誤合併或改派；應依已確認規則轉官方歸屬、保持不變或標記人工審核 | 依安全規則不建立，或只建立明確允許的事件 | 不得錯誤建立或改寫保護 | 系統採安全處理，不自動猜測或合併資料 |

執行限制：

- 不使用真實客戶資料。
- 不使用 `HASUB-20260701-90p7otest1`。
- 本文件只保留範例與待填欄位，不產生正式測試資料。
- referral event 的預期結果必須在驗收前依最終核准規格填定，不得在測試後才解釋。

## 7. 每筆驗收前檢查表

每筆測試開始前，逐項記錄：

- [ ] 測試案例編號。
- [ ] 測試資料已確認為核准的內部資料，不是真實客戶資料。
- [ ] submissionId 是本案例專用的新值，且未使用過。
- [ ] 已確認 CRM true 只在受控的新版本中開啟。
- [ ] 測試前 `health_assessments_健康評估紀錄` 筆數。
- [ ] 測試前 `leads_潛在會員名單` 筆數。
- [ ] 測試前 `referral_events_推薦事件紀錄` 筆數。
- [ ] 測試前既有 Lead 狀態與允許更新欄位。
- [ ] 測試前顧問歸屬狀態。
- [ ] 測試前 90 天推薦保護開始日、到期日及保護狀態。
- [ ] 本案例預期是否建立 referral event 已明確填定。
- [ ] rollback 負責人與停止決策人在線可處理。

## 8. 每筆驗收後檢查表

每筆測試完成後，逐項記錄：

- [ ] 已新增正確的一筆 health assessment。
- [ ] submissionId 只出現一次。
- [ ] assessmentId 已正確回傳並與 Sheet 一致。
- [ ] Lead 符合預期新增、更新或保持不變。
- [ ] referral event 符合預期新增或不新增。
- [ ] 90 天推薦保護符合預期。
- [ ] duplicate replay 回傳原 assessmentId。
- [ ] duplicate replay 沒有新增第二筆 health assessment。
- [ ] duplicate replay 沒有重複建立或更新 Lead。
- [ ] duplicate replay 沒有重複建立 referral event。
- [ ] 回應及執行紀錄是否出現錯誤碼；若有，已完整記錄。
- [ ] 是否產生需要人工標記或復原的資料。
- [ ] 測試後三個 Sheet 的筆數差異與本案例預期一致。

## 9. 停止條件

發生下列任一情況，必須立即停止驗收，不得繼續下一案例：

- 新資料 submissionId 空白。
- 同一 submissionId 寫入兩筆以上。
- duplicate replay 新增第二筆 health assessment。
- duplicate replay 建立或更新 Lead。
- duplicate replay 建立 referral event。
- Lead 顧問歸屬錯誤。
- 既有 Lead 被錯誤覆蓋。
- 90 天推薦保護錯誤。
- referral event 遺漏或重複。
- 出現 `HEALTH_ASSESSMENT_SUBMISSION_ID_COLUMN_MISSING`。
- 使用者前端已顯示成功，但 Sheet 無資料且無法追蹤。
- Sheet 出現無法解釋的異常資料。
- 實際結果與案例預期不一致，且無法立即證明安全。
- 任何情況下無法明確 rollback。

## 10. rollback SOP 草案

以下只定義高層次原則，本文件不執行 rollback：

1. 立即停止繼續測試，不執行下一案例。
2. 停止新流量導入受控驗收路徑。
3. 記錄異常時間、案例編號、submissionId、assessmentId、Lead ID、referral event ID 與畫面結果。
4. 檢查當時使用的 Apps Script 版本、CRM 開關與 Web App deployment。
5. 視影響情況，將既有 Web App deployment 回切到前一個已確認安全的版本；不得建立新 Web App URL。
6. 若需回切 Version 49，必須先評估其不支援 submissionId 的風險，並由回復決策人核准。
7. 人工標記異常資料及影響範圍，不直接刪除或覆寫正式資料。
8. CRM true 回復為 false 後，執行既定安全檢查，確認不再寫入 Lead 或 referral event。
9. 完成異常紀錄後另開修復任務，不在驗收現場臨時擴大修改範圍。
10. 修復、複核及重新核准前，CRM true 不得重新開啟。

## 11. 需要另開任務處理的事項

- Apps Script CRM true 實作或設定切換 PR。
- Apps Script 新版本建立。
- Web App deployment 受控切版。
- 正式內部小流量驗收。
- 驗收結果文件。
- 若驗收通過，再建立 CRM true 正式完成紀錄。
- 若驗收失敗，建立異常與 rollback 紀錄。

## 12. 明確禁止事項

- 本文件不代表可以開 CRM true。
- 不要直接修改正式 Apps Script 設定。
- 不要直接部署。
- 不要建立新 Web App URL。
- 不要送未經計畫的正式健康評估測試。
- 不要使用 `HASUB-20260701-90p7otest1`。
- 不要刪除 `HA000016`。
- 不要回填舊 submissionId。
- 不要改名或移除「送出識別碼」欄位。
- 不要修改第 12 列 `LD000011`。
- 不要修改第 13 列 `LD000012`。
- 不要將 CRM true 與其他功能重構混在同一任務。

## 13. 後續建議

1. 先完成 Version 50 上線後 3～7 天正式流量觀察。
2. 若觀察穩定，再另開 CRM true 實作或設定切換 PR。
3. CRM true 實作任務必須小範圍、可停止、可 rollback。
4. 驗收通過後，再建立正式完成紀錄。
5. 不建議與健檢報告分析系統重構、顧問獎金制度或其他任務合併。
