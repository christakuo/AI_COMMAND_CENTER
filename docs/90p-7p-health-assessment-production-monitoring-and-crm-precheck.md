# 90P-7P 健康評估正式流量監控與 CRM true 前置盤點

## 1. 文件目的

本文件用途如下：

- 記錄 Version 50 上線後的正式流量監控方式。
- 確認 CRM false 期間健康評估資料流是否安全。
- 作為未來開啟 `HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=true` 前的前置檢查依據。
- 本文件不是 CRM true 開啟指令，也不是部署紀錄。

## 2. 目前正式狀態

- Web App：Version 50。
- CRM 寫入開關：`HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=false`。
- Version 49：保留作緊急回切。
- `health_assessments_健康評估紀錄` 已有「送出識別碼」欄位。
- Version 50 上線前的舊資料 submissionId 空白是正常狀態，不回填。
- `leads_潛在會員名單` 未新增 submissionId 欄位。
- `referral_events_推薦事件紀錄` 未新增 submissionId 欄位。

## 3. 每日人工監控清單

建議每天固定時間檢查一次，並留下檢查日期、檢查人及異常說明。

| 檢查項目 | 正常狀況 | 異常狀況 | 建議處理方式 |
|---|---|---|---|
| 健康評估新增筆數 | 與近期一般流量接近，且每次送出只新增一筆 | 短時間突然大量增加，或與實際活動量不符 | 先記錄異常時段，檢查是否集中於相同來源或 submissionId；未釐清前不開 CRM true |
| 新資料 submissionId 是否空白 | Version 50 上線後的新資料都有值 | 新資料的 submissionId 空白 | 立即停止 CRM true 後續作業，確認欄位與送出流程；不要人工補值 |
| submissionId 是否重複 | 每個非空白 submissionId 只出現一次 | 同一 submissionId 出現兩筆以上健康評估 | 保留現場資料並停止後續 CRM true 作業，查明重複寫入原因；不要刪改資料掩蓋問題 |
| duplicate / idempotentReplay 是否異常集中 | 偶發重送可被辨識，且不新增第二筆資料 | 數量突然增加，或集中在同一 submissionId | 記錄時間、submissionId 與 assessmentId，檢查前端提示及網路回應狀況 |
| ambiguous success 是否造成重複資料 | 前端提示與 Sheet 實際寫入一致，重試不產生第二筆 | 前端顯示失敗或不明確，但 Sheet 出現重複資料 | 暫停 CRM true 規劃，核對首次送出與重試結果，確認防重送機制 |
| CRM false 下 leads 是否異常新增／更新 | 健康評估可新增，但 leads 不因健康評估新增或更新 | 出現由健康評估觸發的 Lead 新增或更新 | 立即記錄受影響 Lead，停止後續 CRM true 作業並調查開關與資料路徑 |
| CRM false 下 referral_events 是否異常新增 | 不因健康評估新增 referral event | 出現由健康評估觸發的新 referral event | 立即記錄事件，停止後續 CRM true 作業並確認事件建立條件 |
| 是否出現 `HEALTH_ASSESSMENT_SUBMISSION_ID_COLUMN_MISSING` | 未出現此錯誤碼 | 出現此錯誤碼 | 視為欄位或表頭異常；不要改名、移除或自行重建欄位，先停止後續作業並查明原因 |
| 是否有單一 submissionId 短時間大量 replay | 同一 submissionId 最多只有合理的偶發重試 | 同一 submissionId 在短時間被重送多次 | 記錄次數與時間，檢查前端鎖定、防連點及 ambiguous success 行為；未釐清前不開 CRM true |

## 4. submissionId 檢查規則

- Version 50 上線前舊資料的 submissionId 空白：正常。
- Version 50 上線後新資料的 submissionId 空白：異常。
- 同一個非空白 submissionId 出現兩筆以上：異常。
- 空白值不應納入重複判斷，避免把所有舊資料誤判成重複。
- 不應人工修改新資料的 submissionId。
- 不應回填舊資料的 submissionId。
- 不應改名或移除「送出識別碼」欄位。

## 5. CRM false 期間資料關係

```text
送出健康評估
  → 可以新增 health assessment
  → CRM false
  → 不新增 Lead
  → 不更新 Lead
  → 不建立 referral event
  → 不補建 90 天推薦保護
```

相同 submissionId 發生 duplicate replay 時：

- 不新增第二筆 health assessment。
- 回傳原本的 assessmentId。
- 不觸發 CRM linking。
- 不建立或更新 Lead。
- 不建立 referral event。

## 6. 正常／注意／停止條件

### 正常狀況

- 每份新評估只寫入一筆 health assessment。
- Version 50 上線後的新資料都有 submissionId。
- CRM false 下，leads 與 referral_events 沒有因健康評估異常新增或更新。
- duplicate replay 回傳原 assessmentId，不新增第二筆健康評估。
- ambiguous success 後的重試沒有造成重複寫入。

### 需要注意

- duplicate 或 idempotentReplay 數量突然增加。
- 同一 submissionId 短時間發生多次 replay。
- 前端顯示失敗或結果不明確，但 Sheet 已經寫入。
- 健康評估新增數量異常升高，且無法由活動或流量說明。

### 必須暫停 CRM true

- Version 50 上線後的新資料出現 submissionId 空白。
- 同一 submissionId 被重複寫入兩筆以上。
- 出現 `HEALTH_ASSESSMENT_SUBMISSION_ID_COLUMN_MISSING`。
- duplicate replay 觸發 CRM linking。
- CRM false 下，leads 或 referral_events 出現由健康評估造成的異常新增或更新。
- ambiguous success 造成重複寫入。
- 發現無法說明、無法界定影響範圍或無法安全復原的異常資料。

## 7. CRM true 前置條件

未來開啟 `HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=true` 前，至少需要完成：

- Version 50 穩定觀察完成。
- Version 50 上線後的新資料 submissionId 無空白。
- submissionId 無重複寫入。
- duplicate／idempotentReplay 行為符合預期。
- ambiguous success 不造成重複寫入。
- Lead 新增與更新規則重新確認。
- 既有 Lead 不會被錯誤覆蓋。
- 顧問歸屬規則確認。
- 90 天推薦保護補建邏輯確認。
- referral event 建立條件確認。
- 測試資料與內部測試案例備妥。
- rollback SOP 完成，包含停止條件、回復步驟與負責人。
- CRM true 受控驗收計畫完成並經人工確認。
- 必須另開任務、PR、Apps Script 新版本與受控部署，不得直接在本文件階段開啟。

## 8. CRM true 受控驗收計畫建議

建議另開文件或任務，建立 CRM true 受控驗收計畫。計畫至少應包含：

- 驗收目標。
- 明確的通過標準。
- 3～5 筆內部小流量案例。
- 新 Lead 案例。
- 更新既有 Lead 案例。
- 有推薦碼、無推薦碼及資料衝突案例。
- duplicate replay 案例。
- 90 天推薦保護欄位與期限檢查。
- referral event 是否建立及建立條件檢查。
- 每筆測試前後的 Sheet 核對表。
- 立即停止驗收的條件。
- rollback SOP。
- Apps Script 新版本號與部署紀錄。

**本文件不代表已開啟 CRM true。**

## 9. 禁止事項

- 不要再送 `HASUB-20260701-90p7otest1`。
- 不要刪除 `HA000016`。
- 不要改名或移除「送出識別碼」欄位。
- 不要回填舊 submissionId。
- 不要修改第 12 列 `LD000011`。
- 不要修改第 13 列 `LD000012`。
- 不要開 CRM true，除非另開任務、PR、部署與受控驗收。
- 不要切回 Version 49，除非發生異常且另有回復決策。
- 不要建立新 Web App URL。
- 不要未經受控計畫就送正式健康評估測試。

## 10. 後續建議

1. 先由人工觀察正式 Sheet 3～7 天，留下每日檢查紀錄。
2. 若觀察期間沒有新資料 submissionId 空白、重複寫入或 CRM false 異常，再建立 CRM true 受控驗收計畫。
3. 不建議直接開 CRM true。
4. CRM true 應另開任務處理，並包含 PR、Apps Script 新版本、受控部署、小流量驗收及 rollback SOP。
