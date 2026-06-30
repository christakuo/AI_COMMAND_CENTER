# 90P-7：健康評估推薦保護正式驗收完成紀錄

完成日期：2026-06-30

## 任務背景

本輪完成健康評估與 CRM 名單銜接、推薦碼解析、顧問歸屬及 90 天推薦保護的正式驗收，並處理驗收期間發現的台灣手機格式比對、測試重複 Lead，以及前端對 Apps Script 非 JSON 回應的 ambiguous success 防重送問題。

本文件僅記錄已完成工作、正式驗收結果、最終安全狀態及後續待辦；不含正式會員個資、Web App URL、token、Email 或 LINE UID。

## 已完成項目

### 90P-7J：台灣手機 canonicalization 修復

- Apps Script repo：`yuanxin-apps-script-webhook-clean`
- PR：#12
- `main` HEAD：`ba8a0d2`
- 已將 `0935977113`、`935977113`、`+886935977113`、`886935977113` 視為同一支台灣手機；上述號碼僅為內部測試資料。
- HealthAssessment identity matching 已改用 canonical phone。
- canonical-equivalent duplicate 不再建立新 Lead。
- excluded candidate 不再建立新 Lead。
- 90P-1 保護補建條件未放寬。
- 修正曾同步至 Apps Script，並建立 Version 47（CRM false）。

### 90P-7K：第 13 列測試重複 Lead 清理

- 第 13 列 `LD000012` 確認為 90P-7E-3 測試期間因舊手機比對邏輯產生的重複 Lead。
- 已標記為無效、測試資料及重複資料，並合併至 `LD000011`。
- 手機與 LINE 使用者 ID 欄位已清空，備註已保留原始測試原因供稽核。
- 第 12 列 `LD000011` 保留為正式驗收目標 Lead。

### 90P-7L：最後正式寫入驗收

- 建立 Version 48：CRM true，包含 90P-7J 手機 canonicalization 修復。
- 建立 Version 49：CRM false 安全版。
- 驗收時僅短暫切換至 Version 48、送出一次內部健康評估測試，隨即切回 Version 49。
- 內部測試資料：姓名「張三豐」、手機 `0935977113`、`ref=HC9001`。
- 驗收完成後未再送出健康評估測試。

### 90P-7M：non-JSON response 唯讀追查

- Sheet 寫入已成功，但前端顯示 Apps Script returned non-JSON response。
- 問題來源為 Vercel proxy 對 Apps Script response 執行 `JSON.parse` 失敗後回傳 502。
- 前端原先將此情況視為一般失敗並允許重新送出，可能造成重複 health assessment。

### 90P-7N：non-JSON ambiguous success 防重送修復

- Frontend repo：`yuanxin-liff-demo`
- PR：#42
- Production 已完成部署。
- Apps Script 回傳非 JSON 或空回應時，proxy 改回傳安全 JSON，代碼為 `APP_SCRIPT_NON_JSON_RESPONSE` 或 `APP_SCRIPT_EMPTY_RESPONSE`，並設定 `ambiguousSuccess=true`。
- diagnostics 僅包含非敏感的 upstream 狀態、內容類型、redirect 狀態、body 長度與類型，以及 correlation ID。
- 不回傳 raw upstream body，也不記錄 payload、token、姓名、手機、Email 或 LINE UID。
- 前端改顯示「資料可能已送出，請勿重複送出」，停用立即重送，按鈕改為「送出狀態待確認」。

## 正式驗收結果

- `leads_潛在會員名單` 最後列為 13，未新增 Lead。
- `health_assessments_健康評估紀錄` 最後列為 16。
- `referral_events_推薦事件紀錄` 最後列為 38，未新增 referral event。
- 第 12 列 `LD000011` 成功更新：
  - 手機 canonical 值：`935977113`（內部測試資料）
  - 推薦碼：`HC9001`
  - 歸屬顧問 ID：`HC9001`
  - 歸屬顧問姓名：內部測試顧問
  - 推薦保護開始日：2026-06-30 18:40:26
  - 推薦保護到期日：2026-09-28 18:40:26
  - 是否保護期內：是
  - 是否已填健康評估：是
  - 健康評估 ID：`HA000015`
  - 最近互動時間：2026-06-30 18:40:26
- 驗收結論：
  - `935977113` 與 `0935977113` 成功視為同一支內部測試手機。
  - 既有 Lead `LD000011` 正確更新，未新增重複 Lead。
  - 90 天推薦保護期成功補建。
  - `referral_events` 未被誤新增。

## 最終系統狀態

- Frontend repo：`yuanxin-liff-demo`
- Frontend Production：PR #42 已部署。
- Apps Script Web App：Version 49。
- CRM flag：false。
- 尚未在 PR #42 部署後再次送出健康評估測試，且不需要再做 CRM true 正式測試。
- 不得再切換至 Version 48。
- 第 12 列與第 13 列均不再修改。

## 已知問題與處理

### Apps Script non-JSON／empty response

此情況可能代表上游已完成寫入，但 proxy 無法解析回應，屬於結果不明確而非可安全重試的一般失敗。PR #42 已將其改為 ambiguous success，提供非敏感 diagnostics，並在前端阻止使用者立即重送。

### submission 層級冪等性尚未完成

目前前端已降低 ambiguous response 導致的人為重送風險，但後端尚未以 submission ID 建立完整 idempotency 防線，列入 90P-7O。

## 後續待辦

1. 90P-7O：為健康評估加入 `submissionId`／idempotency 防重送機制。
2. create path header mapping／fail closed 另案檢查，確認缺欄或欄位映射異常時不會建立不完整資料。
3. 第 13 列 `LD000012` 保留為測試稽核資料，不再動作。
4. 未來若開啟 CRM true，必須使用最新版 Apps Script，不得回切舊版 Version 45。
5. 維持 Version 49 與 CRM false，除非另有正式、受控且完成前置檢查的啟用任務。

## 本輪文件收尾限制

- 僅修改 `AI_COMMAND_CENTER` 文件。
- 未修改 frontend repo 或 Apps Script repo。
- 未修改程式碼。
- 未呼叫正式 Web App、未讀寫 Sheet。
- 未設定環境變數或 Script Property。
- 未部署、未 Commit、未 Push。
