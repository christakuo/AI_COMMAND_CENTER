# 90P-7O 健康評估 submissionId / idempotency 防重送完成紀錄

## 1. 任務目的

建立健康評估送出的 `submissionId / idempotency` 防重機制，避免：

- 使用者重複點擊送出
- 回應不明確後再次送出
- 重複新增健康評估
- 重跑 CRM、Lead 更新及推薦保護流程

## 2. 修改範圍

### 前端

- PR：`yuanxin-liff-demo #43`
- 已 merge 至 main
- 每份評估產生非敏感 submissionId
- 使用 sessionStorage 保存
- sessionStorage 不可用時使用頁面記憶體備援
- 成功或 ambiguous success 後鎖定重送
- 一般失敗沿用原 submissionId
- Proxy 嚴格驗證並原樣轉送 submissionId
- 保留 PR #42 ambiguousSuccess 機制

### Apps Script

- PR：`yuanxin-apps-script-webhook-clean #13`
- main：`2e2cd27`
- 寫入前驗證及查詢 submissionId
- duplicate 在 CRM 流程前返回
- 不重複建立 health assessment
- 不重跑 Lead、90 天推薦保護或 referral event
- 缺少 Sheet 欄位時安全中止：
  `HEALTH_ASSESSMENT_SUBMISSION_ID_COLUMN_MISSING`

## 3. 部署資訊

- clasp push：成功，共 17 個檔案
- Apps Script Version：50
- 版本描述：`90P-7O submissionId idempotency CRM false`
- 既有 Web App deployment 已由 Version 49 切換至 Version 50
- Web App URL 維持不變
- 未建立新 deployment URL
- Version 49 保留作回切版本

## 4. Sheet 欄位變更

Sheet：

`health_assessments_健康評估紀錄`

欄位配置：

- A 欄：健康評估ID
- B 欄：送出識別碼
- C 欄：身分狀態

舊資料的送出識別碼保持空白，未進行回填。

以下 Sheet 未新增 submissionId 欄位：

- `leads_潛在會員名單`
- `referral_events_推薦事件紀錄`

## 5. 正式驗收結果

測試 submissionId：

`HASUB-20260701-90p7otest1`

正式最小驗收測試資料：

- assessmentId：`HA000016`
- isTestData：true
- 建議保留，不得任意刪除

### 第一次送出

- `status: success`
- `success: true`
- `duplicate: false`
- `idempotentReplay: false`
- `assessmentId: HA000016`
- `createdHealthAssessment: true`
- `createdLead: false`
- `updatedLead: false`
- `createdReferralEvent: false`
- `CRM_WRITE_DISABLED`
- CRM false 生效

### 第二次相同 submissionId replay

- `duplicate: true`
- `idempotentReplay: true`
- `assessmentId: HA000016`
- `createdHealthAssessment: false`
- `createdLead: false`
- `updatedLead: false`
- `createdReferralEvent: false`

### Sheet 驗收

- submissionId 只出現一次
- 未新增第二筆 health assessment
- 兩次回傳相同 assessmentId
- leads 未增加
- referral_events 未增加
- 第 12 列 LD000011 未修改
- 第 13 列 LD000012 未修改

驗收結論：通過。

## 6. Guardrails

- `HEALTH_ASSESSMENT_CRM_WRITE_ENABLED` 維持 false
- submissionId 不包含個資或健康答案
- Proxy 不記錄或回傳敏感 payload
- duplicate 在 CRM linking 前停止
- duplicate 不建立或更新 Lead
- duplicate 不補建 90 天推薦保護
- duplicate 不建立 referral event
- 缺少「送出識別碼」欄位時 fail closed
- Version 49 保留作緊急回切
- 不修改既有正式會員資料

## 7. 後續注意事項

- 持續監控重複送出及 ambiguous response 紀錄。
- 不應回填舊資料的 submissionId。
- 不得移除或更名「送出識別碼」欄位。
- CRM 開關若需改為 true，必須另開任務、PR 及受控驗收。
- 後續程式變更必須保留 Script Lock 內查重。
- 發生重複寫入、CRM 異常或欄位錯誤時，立即停止流量並回切 Version 49。
- 測試資料 HA000016 是正式最小驗收測試資料，`isTestData=true`；應依既定測試資料管理規範標記與保留，不得任意刪除。
