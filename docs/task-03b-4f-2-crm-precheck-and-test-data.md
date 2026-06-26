# Task 03B-4F-2：CRM 寫入開關前檢查文件與測試資料確認

## 1. 任務目標

本任務目標是在開啟健康評估 CRM 寫入前，先建立完整的檢查清單、測試資料清單、小流量驗收原則與回滾 SOP，作為後續小流量啟用前的共同驗收依據。

本任務安全邊界：

- 本任務不開 CRM。
- 本任務不部署 Apps Script。
- 本任務不寫入 Sheet。
- 本任務不新增 referral_events。

## 2. 目前系統狀態

- 線上 Apps Script：元馨醫管家_中央推薦碼Webhook_v1
- 目前已部署版本：第 20 版
- CRM 開關：HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=false
- 尚未部署第 21 版
- 尚未開啟正式 CRM 寫入
- 尚未正式建立／更新 leads
- 尚未新增健康評估 referral_events
- 前端與 Vercel 未修改

## 3. 已完成前置任務

### Task 03B-4E

- 推薦碼解析
- 顧問有效性判定
- HC0000 官方帳號例外
- 新建 lead 顧問歸屬
- 一般顧問 90 天保護
- 既有 lead 不自動改派
- 已付款／已成交保護
- 停止聯繫／放棄追蹤保護
- CRM false 不讀不寫
- 部署第 20 版並通過安全測試

### Task 03B-4F-1

- 新增 testTask03B4F1DryRunMatrixAll
- 覆蓋 A～W dry-run 案例矩陣
- A～U 實際驗證
- V／W LINE 案例列 futureCase / skipped
- 線上 Test.gs 已同步
- 三項安全測試通過
- 未部署第 21 版

## 4. CRM 開關前必查清單

| 類別 | 檢查項目 | 預期狀態 | 確認方式 | 是否通過 | 備註 |
|---|---|---|---|---|---|
| Config | Config.gs 中 HEALTH_ASSESSMENT_CRM_WRITE_ENABLED 仍為 false | 開關仍關閉 | 線上 Apps Script 唯讀確認 Config.gs | 待確認 | 開 CRM 前必須由使用者明確授權 |
| 部署 | 開 CRM 前必須明確建立新版本與更新現有 Web App 部署 | 有明確版本與部署步驟 | Apps Script 版本與部署畫面確認 | 待確認 | 不可只改程式不控管版本 |
| Web App | Web App URL 不可更換 | 正式 URL 維持不變 | 部署設定畫面確認 | 待確認 | 避免前端 submit endpoint 失效 |
| 前端 | 前端 submit endpoint 不變 | 不改前端 | yuanxin-liff-demo 不變更 | 待確認 | 本階段不操作前端 |
| Vercel | Vercel proxy 未修改 | 不改 Vercel | Vercel repo / 設定不變 | 待確認 | 本階段不操作 Vercel |
| leads 欄位 | leads_潛在會員名單 欄位符合 builder 寫入欄位 | 欄位存在且命名一致 | Sheet 第一列欄名唯讀確認 | 待確認 | 不新增欄位、不改欄名 |
| health_assessments 欄位 | health_assessments_健康評估紀錄 可承接 CRM link 回寫欄位 | 欄位存在且命名一致 | Sheet 第一列欄名唯讀確認 | 待確認 | 使用既有事件處理狀態與異常原因 |
| consultants 測資 | consultants_顧問主檔 有足夠測試顧問資料 | 測試顧問資料齊全 | Sheet 唯讀確認或受控建立測資 | 待確認 | 不破壞正式顧問資料 |
| referral_events | referral_events 本階段仍不新增 | 不寫入 | 測試結果確認 createdReferralEvent=false | 待確認 | 留待後續獨立任務 |
| 顧問通知 | 顧問通知本階段不啟用 | 不通知 | 流程與測試結果確認 | 待確認 | 避免測試觸發營運通知 |
| LINE 訊息 | LINE 主動訊息本階段不啟用 | 不發送 | 流程與測試結果確認 | 待確認 | 避免對會員發送測試訊息 |
| 測試防線 | isTestData 防線保留 | 測試資料不建單 | test / dry-run 結果確認 | 待確認 | 正式開關前仍需保留 |
| 回滾 | 回滾方式已確認 | 可快速關回 false | 回滾 SOP 演練確認 | 待確認 | 需知道誰執行、何時執行 |
| 小流量 | 小流量測試資料已確認 | 只用內部手機 | 測試資料清單確認 | 待確認 | 第一批 3～5 筆 |
| 監控 | 監控方式已確認 | 每筆後立即檢查 Sheet | 驗收紀錄表確認 | 待確認 | 需記錄最後一列與更新列 |
| 誤寫處理 | 誤寫資料處理方式已確認 | 先標記、不批次刪除 | 誤寫處理 SOP 確認 | 待確認 | 避免破壞正式資料 |

## 5. consultants 測試資料清單

| 測試類型 | 必要欄位狀態 | 建議測試碼 | 預期結果 | 是否已有資料 | 備註 |
|---|---|---|---|---|---|
| HC0000 官方帳號 | 顧問ID=HC0000、顧問狀態=系統帳號、推薦碼=HC0000 | HC0000 | 明確輸入 HC0000 時歸官方，leads 推薦碼可寫 HC0000 | 待確認 | 官方暫存池，不作為一般顧問保護 |
| 有效顧問 | 顧問狀態=已啟用、是否可推廣=是、停權日期空白、終止日期空白 | 內部有效測試碼 | 新建 lead 可歸屬該顧問並啟動 90 天保護 | 待確認 | 建議使用內部測試顧問 |
| 停用顧問 | 顧問狀態非已啟用 | 內部停用測試碼 | 不可歸屬，改歸 HC0000，leads 推薦碼空白 | 待確認 | 不應使用正式顧問碼做破壞性測試 |
| 不可推廣顧問 | 是否可推廣=否 | 內部不可推廣測試碼 | 不可歸屬，改歸 HC0000，leads 推薦碼空白 | 待確認 | 欄位空白時應採保守策略 |
| 停權顧問 | 停權日期有值 | 內部停權測試碼 | 不可歸屬，改歸 HC0000，leads 推薦碼空白 | 待確認 | 不可影響正式停權資料 |
| 終止顧問 | 終止日期有值 | 內部終止測試碼 | 不可歸屬，改歸 HC0000，leads 推薦碼空白 | 待確認 | 不可影響正式終止資料 |
| 重複推薦碼 | 兩筆測試顧問使用同一推薦碼 | 內部重複測試碼 | 不自動歸屬，標記人工審核 | 待確認 | 需特別小心，避免影響正式顧問 |

測試資料注意事項：

- 測試資料若需新增，應明確標示為內部測試資料。
- 不可破壞正式顧問主檔。
- 重複推薦碼測試需特別小心，避免影響正式顧問。

## 6. leads 測試資料清單

| 測試類型 | 必要欄位狀態 | 預期 CRM decision | 預期是否建立 lead | 預期是否更新 lead | 預期是否人工審核 | 是否需要正式寫入測試 | 備註 |
|---|---|---|---|---|---|---|---|
| 新手機 | 手機不存在於 leads，姓名與手機合法 | EXTERNAL_CREATE_ALLOWED | 是 | 否 | 否 | 是 | 第一批可先測無推薦碼與有效推薦碼 |
| 既有一般顧問 | 手機命中既有 lead，歸屬顧問ID 為一般顧問 | EXTERNAL_UPDATE_ALLOWED | 否 | 是 | 若新碼不同則是 | 是 | 不自動改派 |
| 既有 HC0000 | 手機命中既有 lead，歸屬顧問ID=HC0000 | EXTERNAL_UPDATE_ALLOWED | 否 | 是 | 是 | 是 | 不自動從官方池轉派 |
| 顧問欄位空白 | 手機命中既有 lead，歸屬顧問ID 空白 | EXTERNAL_UPDATE_ALLOWED | 否 | 是 | 是 | 是 | 第一版不自動補入顧問 |
| 已付款／已成交 | 是否已付款=是，或 CRM階段 / 名單狀態含成交 | PAID_OR_CONVERTED_LEAD_FOUND | 否 | 是 | 依歸屬情況 | 是 | 不更新最近互動、不改派 |
| 停止聯繫 | 是否停止聯繫=是 | STOP_CONTACT_LEAD_FOUND | 否 | 是 | 依歸屬情況 | 是 | 不更新聯絡權限、不更新最近互動 |
| 放棄追蹤 | CRM階段或名單狀態含放棄追蹤 | ABANDONED_LEAD_FOUND | 否 | 是 | 依歸屬情況 | 是 | 第一版比照停止聯繫 |
| 重複／已合併 | 是否重複資料=是，或合併至名單ID 有值 | EXISTING_LEAD_REVIEW_REQUIRED | 否 | 否 | 是 | 可先 dry-run | 不更新 leads |
| 姓名手機衝突 | 手機相同但姓名不同 | PHONE_NAME_MISMATCH | 否 | 否 | 是 | 可先 dry-run | 不更新 leads |
| 無效手機格式 | 手機非 09 開頭 10 碼 | INVALID_PHONE | 否 | 否 | 否 | 否 | 不應正式寫入 |
| isTestData=true | payload isTestData=true | TEST_DATA_ASSESSMENT_ONLY | 否 | 否 | 否 | 否 | 測試資料防線 |
| CRM false | HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=false | CRM_WRITE_DISABLED | 否 | 否 | 否 | 否 | 不讀 leads、不寫 leads |

## 7. 小流量啟用原則

- 第一批只用內部手機。
- 建議 3～5 筆，不超過 5 筆。
- 每筆測試後立即檢查 Sheet。
- 不通知顧問。
- 不發 LINE 主動訊息。
- 不新增 referral_events。
- 先驗證新建 lead，再驗證更新既有 lead。
- 每筆驗收後都要記錄測試結果。
- 若有異常，立即停止並回滾 CRM 開關。

## 8. 每筆驗收後檢查欄位

### health_assessments 檢查欄位

- 健康評估ID
- 身分狀態
- 名單ID
- 推薦碼
- 是否有效推薦碼
- 歸屬顧問ID
- 歸屬顧問姓名
- 事件處理狀態
- 異常原因
- 聯絡同意版本
- 聯絡同意時間

### leads 檢查欄位

- 名單ID
- 潛在會員姓名
- 手機號碼
- 電子信箱
- 推薦碼
- 歸屬顧問ID
- 歸屬顧問姓名
- 首次推薦來源
- 推薦保護開始日
- 推薦保護到期日
- 是否保護期內
- 歸屬爭議狀態
- 歸屬爭議備註
- CRM階段
- 是否已付款
- 是否停止聯繫
- 健康評估ID
- 初步風險等級
- 主要健康關注
- 健康評估摘要
- 是否允許LINE訊息
- 是否允許電話聯繫
- 是否允許行銷資訊
- 最近互動時間
- 最後更新時間

## 9. 回滾 SOP

1. 停止所有測試入口。
2. 將 Config.gs 的 HEALTH_ASSESSMENT_CRM_WRITE_ENABLED 改回 false。
3. 儲存。
4. 建立新 Apps Script 版本。
5. 更新現有 Web App 部署，不更換 URL。
6. 執行 testHealthAssessmentCrmWriteDisabled。
7. 確認 decisionCode=CRM_WRITE_DISABLED。
8. 確認 sheetReadPerformed=false、sheetWritePerformed=false。
9. 標記誤寫資料，不要直接批次刪除。
10. 記錄異常情境與回復方式。

## 10. 不在本階段範圍

- 不開 CRM
- 不部署第 21 版
- 不新增 referral_events
- 不發顧問通知
- 不發 LINE 主動訊息
- 不做 Production 正式送出
- 不改前端
- 不改 Vercel
- 不改 Sheet 欄位

## 11. 下一步建議

下一步建議：

Task 03B-4F-3：小流量 CRM 寫入驗收計畫與開關操作規格

03B-4F-3 應包含：

- 是否部署第 21 版
- 是否準備一版 CRM true 的 Apps Script
- 是否先只測內部手機
- 實際開關操作步驟
- 驗收案例順序
- 回滾判斷點
