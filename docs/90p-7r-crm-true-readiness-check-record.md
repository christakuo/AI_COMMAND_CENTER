# 90P-7R CRM true 受控開啟前設定與驗收資料盤點紀錄

## 1. 文件目的

- 本文件記錄 CRM true 受控開啟前的正式設定與驗收資料人工唯讀盤點結果。
- 本文件不是 CRM true 開啟紀錄。
- 本文件不是正式健康評估測試紀錄。
- 本文件不代表已經修改 Script Properties 或 Google Sheet。
- 目前 CRM 仍為 false。

## 2. 目前正式狀態

- 正式 Web App：Version 51。
- Web App URL：維持不變。
- CRM：`HEALTH_ASSESSMENT_CRM_WRITE_ENABLED=false`。
- Version 50：保留可回切。
- Version 49：保留作更早期緊急回切。
- 90P-7Q 已完成 referral event linking 與 Version 51 deployment 文件收尾。

## 3. 盤點方式

- Codex 原先嘗試唯讀盤點，但因 Chrome 外掛 Native Host 無法連線，未能安全讀取 Script Properties 與正式 Google Sheet。
- 因安全規則，未使用替代腳本繞過登入。
- 後續改由使用者人工唯讀檢查正式 Apps Script Project Settings 與正式 Google Sheet。
- 本次未修改 Script Properties。
- 本次未修改 Google Sheet。

## 4. Script Properties 盤點結果

| 設定項目 | 結果 | 備註 |
|---|---|---|
| `HEALTH_ASSESSMENT_INTERNAL_VALIDATION_TOKEN` | 存在 | 僅確認存在，未記錄或揭露 token 值 |
| `HEALTH_ASSESSMENT_INTERNAL_TEST_PHONES` | 存在，共 5 筆 | 其中包含同一手機的 09 開頭與去除開頭 0 格式，為避免驗收時格式判讀不同而刻意保留 |
| `OFFICIAL_CONSULTANT_ID` | `HC0000` | 可對應正式官方顧問設定 |

本紀錄未揭露 token 或任何完整手機號碼。

## 5. 官方顧問與推薦碼候選盤點

- 官方顧問 `HC0000` 存在。
- `HC0000` 只有唯一一筆。
- `HC0000` 狀態：系統帳號。
- 有效顧問推薦碼候選：`HC9001`。
- 候選顧問ID：`HC9001`。
- 候選狀態：已啟用。
- `HC9001` 可供後續「有效推薦碼新訪客」內部小流量驗收使用。

## 6. health_assessments 盤點

- `health_assessments_健康評估紀錄` 的「送出識別碼」欄位存在。
- `HA000016` 存在。
- `HASUB-20260701-90p7otest1` 只出現一次。
- 未刪除或修改 `HA000016`。
- 未新增健康評估測試資料。

## 7. leads 盤點

- `LD000011` 存在。
- `LD000012` 存在。
- 未修改 `LD000011`。
- 未修改 `LD000012`。
- 既有 Lead 更新候選：`LD000010`。
- `LD000010` 已避開不可動的 `LD000011` 與 `LD000012`。
- 本紀錄不揭露姓名、手機或其他個資。

## 8. referral_events 盤點

- `referral_events_推薦事件紀錄` 分頁存在。
- 現有資料可供後續比對新增事件。
- 目前最後一筆事件ID：`EV000098`。
- 未新增 referral event。
- 未修改既有事件資料。

## 9. CRM true 受控驗收可行性初判

| 驗收案例 | 初步判斷 | 依據 |
|---|---|---|
| 無推薦碼新訪客 → 官方 `HC0000` | 具備前置資料 | `OFFICIAL_CONSULTANT_ID=HC0000`，且顧問主檔有唯一系統帳號 `HC0000` |
| 有有效推薦碼新訪客 → `HC9001` | 具備前置資料 | `HC9001` 顧問及推薦碼候選存在，狀態為已啟用 |
| 既有 Lead 更新 → `LD000010` | 具備候選資料 | 已避開不可動的 `LD000011` 與 `LD000012` |
| duplicate replay | 具備驗收依據 | 依 90P-7O／90P-7Q 設計，duplicate 在 CRM linking 前停止 |
| referral event | 具備程式能力 | Version 51 已接入；CRM false 時不會建立，須待受控 CRM true 任務驗收 |
| rollback | 具備基本版本依據 | Version 50 與 Version 49 均保留 |

此初判不代表已開 CRM true，也不代表已送正式健康評估測試。

## 10. 仍需注意事項

- CRM true 開啟仍須另開任務。
- 驗收必須小範圍、受控並逐筆核對。
- 白名單手機格式包含 09 開頭與去除開頭 0 格式，後續驗收時須先確認使用哪一筆及前後端正規化結果。
- 不得使用 `HASUB-20260701-90p7otest1`。
- 不得刪除或修改 `HA000016`。
- 不得修改 `LD000011` 或 `LD000012`。
- 若 CRM true 驗收失敗，須依 rollback SOP 另案處理。
- 不得與健檢報告分析系統、顧問獎金制度或其他功能混在同一任務。

## 11. 後續建議

- 可進入 90P-7R 第二階段：CRM true 最小設定 PR／受控開啟準備。
- 第二階段仍應維持小範圍處理。
- 第二階段應先只處理 CRM 開關或受控條件相關設定，不應合併其他功能。
- 正式送測試前，必須逐項確認新 submissionId、內部 token、選定的白名單手機與核准測試案例。

## 12. 安全確認

- 未修改 Script Properties。
- 未修改 Google Sheet。
- 未送正式健康評估測試。
- 未開 CRM true。
- 未修改前端 repo。
- 未修改 Apps Script repo。
- 未部署。
- 未建立 Apps Script 新版本。
