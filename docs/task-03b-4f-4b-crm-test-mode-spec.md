# Task 03B-4F-4B：正式 CRM 小流量測試模式修正規格

## 1. 任務目標

本任務目標是釐清 Task 03B-4F-4 Case 1 未建立 lead 的原因，並定義正式 CRM 小流量測試模式的修正規格。

本任務只做文件與規格整理，不做以下操作：

- 不修改前端程式。
- 不修改 Vercel proxy。
- 不修改 Apps Script。
- 不部署。
- 不開 CRM。
- 不寫入 Google Sheet。
- 不新增 referral_events。

## 2. 03B-4F-4 Case 1 驗收結果

### Case 1 原始預期

- 使用新手機。
- 無推薦碼。
- 應新增 health_assessments 1 筆。
- 應新增 leads 1 筆。
- lead 應歸屬 HC0000。
- 不新增 referral_events。

### Case 1 實際結果

- health_assessments 新增 1 筆：HA000008。
- leads 未新增。
- 歸屬顧問ID 未寫入。
- referral_events 未新增。
- 前端成功頁顯示「健康評估測試資料已送出」。
- 前端成功頁顯示「資料已以測試資料保存於健康評估紀錄，目前尚未建立或更新CRM名單」。
- 因 Case 1 未通過，停止 Case 2 / Case 3。
- 已回滾至 Apps Script 第 23 版，CRM=false。
- Web App URL 維持不變。
- testHealthAssessmentCrmWriteDisabled 已通過。

## 3. 原因判斷

Case 1 未建立 lead 的最可能原因：

- `health-assessment.js` 目前固定送出 `isTestData=true`。
- `api/submit-health-assessment.js` 目前強制要求 payload 必須是 `isTestData=true`。
- `health-assessment.html` 明確標示目前是測試階段，送出資料不建立或更新 CRM 名單。
- Apps Script 在 CRM=true 且收到 `isTestData=true` 時，會依安全防線回傳 `TEST_DATA_ASSESSMENT_ONLY / assessment_only`，並跳過 leads 建立或更新。

因此，Case 1 的狀態判斷為：

- CRM true 已生效。
- health_assessments 寫入成功。
- CRM linking 因 `isTestData=true` 被安全跳過。
- leads 未新增是目前系統防線的預期結果。
- 目前沒有證據顯示是 Sheet 欄位錯位。
- 目前沒有證據顯示是 HC0000 顧問資料缺失。
- 目前沒有證據顯示是推薦碼或顧問歸屬邏輯造成。

## 4. 正式小流量測試模式需求

正式 CRM 小流量測試模式需要同時滿足以下條件：

- 內部測試可以送出 `isTestData=false` 的 production CRM payload。
- 一般公開訪客仍維持 `isTestData=true` 或既有測試保護。
- Vercel proxy 不能無條件放行 `isTestData=false`。
- Apps Script 必須有後端防線，不能只依賴前端判斷。
- CRM true 期間仍不得新增 referral_events。
- 顧問通知與 LINE 主動訊息本階段仍不啟用。
- Web App URL 不可更換。
- 若任一驗收異常，必須可快速回滾到 CRM=false。

## 5. 可行方案評估

| 方案 | 說明 | 安全性 | 是否需改前端 | 是否需改 Vercel proxy | 是否需改 Apps Script | 是否需部署 | 優點 | 風險 | 是否建議採用 |
|---|---|---|---|---|---|---|---|---|---|
| 方案 A | 新增內部測試 query mode，例如 `health-assessment.html?mode=crm_internal_test` | 中 | 是 | 是 | 建議搭配 | 是 | 可驗證真實前端送出流程 | 若只靠 query，容易被誤用 | 可作為入口設計，但需搭配後端防線 |
| 方案 B | 前端與 Vercel proxy 增加 internal validation token / secret | 中高 | 是 | 是 | 建議搭配 | 是 | 可避免一般訪客送出 production payload | token 管理需謹慎，不能外洩 | 建議採用 |
| 方案 C | Apps Script 增加 `allowCrmWriteTestRun`、內部手機白名單或 internal token 防線 | 高 | 視設計而定 | 視設計而定 | 是 | 是 | 後端可獨立阻擋誤寫入 | 需定義白名單與 decisionCode | 強烈建議採用 |
| 方案 D | 只新增 Apps Script 測試函式模擬 production payload | 中 | 否 | 否 | 是 | 是 | 可快速驗證後端 CRM 寫入邏輯 | 無法驗證前端與 Vercel proxy 真實路徑 | 可作輔助驗證，不建議作唯一驗收 |

## 6. 建議採用方案

建議採用「方案 A + 方案 B + 方案 C」的組合：

1. 前端新增內部小流量測試入口，例如：
   - `health-assessment.html?mode=crm_internal_test`
2. 前端只有在內部測試模式下，才允許送出 `isTestData=false`。
3. Vercel proxy 不再無條件要求所有 payload 都必須 `isTestData=true`，但必須驗證 internal validation token 或其他內部驗證條件。
4. Apps Script 增加後端防線，例如：
   - `allowedInternalTestPhones`
   - `internalValidationToken`
   - `allowCrmWriteTestRun`
5. 內部驗證未通過時，即使 CRM=true，也不得建立或更新 leads。
6. referral_events 仍維持 false，不新增。

此組合可以同時驗證真實前端流程、Vercel proxy、Apps Script 與 Sheet 寫入，同時保留多層安全防線。

## 7. 前端修正規格

前端修正需定義：

- query 參數名稱，例如 `mode=crm_internal_test`。
- 哪些條件下可送出 `isTestData=false`。
- 哪些內部測試手機可使用正式 CRM 小流量測試。
- 一般公開訪客是否一律維持 `isTestData=true`。
- 成功頁文案是否需要依 CRM 結果顯示不同訊息。
- `ref`、UTM、source 等來源參數仍需保留，不得影響推薦碼與來源追蹤。

## 8. Vercel proxy 修正規格

Vercel proxy 修正需定義：

- `api/submit-health-assessment.js` 如何判斷可接受 `isTestData=false`。
- 一般公開流量是否仍只能送 `isTestData=true`。
- `isTestData=false` 是否必須帶 internal validation token。
- token 是否應存放於 Vercel environment variable。
- token 不可寫死在前端公開程式碼中。
- 若驗證失敗，應回傳清楚錯誤，不可轉送 Apps Script。
- 不可讓一般訪客任意觸發 production CRM payload。

## 9. Apps Script 修正規格

Apps Script 修正需定義：

- `processHealthAssessmentCrmLinking` 在 CRM=true 時，如何再驗證內部小流量測試條件。
- internal validation token 與 phone whitelist 是否需要同時成立。
- `isTestData=false` 且內部驗證通過時，才允許進入正式 CRM linking。
- 驗證未通過時，仍只保存 health_assessments，不建立或更新 leads。
- 建議新增或保留 decisionCode，例如：
  - `INTERNAL_CRM_TEST_NOT_ALLOWED`
  - `CRM_INTERNAL_VALIDATION_FAILED`
  - `CRM_SMALL_BATCH_ALLOWED`
- referral_events 仍固定不新增。
- 顧問通知與 LINE 主動訊息仍不啟用。

## 10. 安全測試與回滾要求

正式 CRM 小流量測試模式修正後，必須先完成以下安全測試：

- CRM=false 時，不讀 leads、不寫 leads。
- 非內部測試條件下，即使有 `isTestData=false`，也不得寫入 leads。
- CRM=true 但 internal validation token 錯誤時，不得寫入 leads。
- CRM=true 但手機不在白名單時，不得寫入 leads。
- CRM=true 且內部驗證通過時，才允許小流量案例寫入 leads。
- 每筆正式小流量測試後，都需檢查 health_assessments、leads、referral_events。
- 任一異常需立即停止測試，將 CRM 開關回滾為 false。
- 回滾後必須執行 testHealthAssessmentCrmWriteDisabled。
- Web App URL 不得更換。

## 11. 後續任務拆分建議

建議拆成以下後續任務：

### Task 03B-4F-4C：CRM 小流量測試模式前端與 proxy 修正規格

範圍：

- 定義前端 query mode。
- 定義 `isTestData=false` 的前端條件。
- 定義 Vercel proxy token 驗證方式。
- 定義成功頁文案與 response 顯示方式。

### Task 03B-4F-4D：Apps Script 正式小流量防線修正規格

範圍：

- 定義 Apps Script 後端小流量驗證條件。
- 定義 phone whitelist。
- 定義 internal validation token。
- 定義新增 decisionCode。
- 定義安全測試函式。

### Task 03B-4F-4E：CRM true 小流量驗收重啟

範圍：

- 重新部署必要版本。
- CRM true 前安全檢查。
- 只用內部手機執行小流量驗收。
- 驗收後依結果決定是否繼續或回滾。

## 12. 不在本任務範圍

本任務不包含：

- 不修改程式。
- 不部署。
- 不開 CRM。
- 不寫入 Sheet。
- 不新增 referral_events。
- 不執行正式健康評估送出。
- 不通知顧問。
- 不發 LINE 主動訊息。

## 13. 下一步建議

建議下一步先進入：

**Task 03B-4F-4C：CRM 小流量測試模式前端與 proxy 修正規格**

再進入：

**Task 03B-4F-4D：Apps Script 正式小流量防線修正規格**

完成 4F-4C 與 4F-4D 規格確認後，才進入實作與重新部署規劃。
