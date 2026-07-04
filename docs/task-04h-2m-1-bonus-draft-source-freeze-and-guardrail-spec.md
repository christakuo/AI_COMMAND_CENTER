# Task 04H-2M-1：正式獎金草稿來源封版與補強規格

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04H-2M-1：正式獎金草稿來源封版與補強規格 |
| 文件狀態 | 規格草案，供 Task 04H-2M-2 實作與驗收使用 |
| 建立日期 | 2026-07-04 |
| 前置盤點 | Task 04H-2L：三 repo 正式版本對齊與顧問獎金草稿產生器唯讀封版審查 |
| 本次修改範圍 | `AI_COMMAND_CENTER` 文件與 roadmap |
| 不在本次範圍 | Apps Script／前端實作、Google Sheet 異動、草稿產生、部署或正式開放 |

## 2. 任務背景與 04H-2L 結論

Task 04H-2L 已確認 `AI_COMMAND_CENTER`、`yuanxin-liff-demo` 與 `yuanxin-apps-script-webhook-clean` 的指定本機路徑、`main` 分支與正式版本已對齊，並完成現有顧問獎金草稿產生器、read-only API 與 LIFF 畫面的唯讀審查。

審查結論如下：

- `generateConsultantBonusDrafts()` 仍是 Task 04B-2 MVP，只能作為既有流程的技術雛形。
- 現行產生器不足以證明正式成交、全額入帳、推薦歸屬、90 天保護、顧問資格、退款／取消／爭議與規則版本均正確。
- 現行 `名單ID + 獎金類型` 防重鍵、固定 NT$4,000 金額及待審核列覆寫方式，不符合正式財務資料所需的版本、來源快照、稽核與回復能力。
- 正式資料來源及 guardrail 完成前，不得執行正式 Generate，不得產生正式獎金草稿，也不得將 `consultantBonusEnabled` 改為 `true`。

本文件的目的，是在任何後續程式補強或正式資料處理前，先封版資料責任、阻擋規則、preview／generate 合約、防重範圍及稽核要求。

## 3. 現行 MVP 產生器限制

| 限制 | 現況 | 正式風險 |
|---|---|---|
| 資料來源 | 只讀 `leads_潛在會員名單` 與既有 `bonus_records` | CRM 摘要可能被誤當財務與成交權威 |
| 獎金金額 | 程式固定 NT$4,000 | 規則異動、適用期間或例外無法追溯 |
| 付款／入帳 | 依 lead 的「是否已付款」及付款／成交日期備援 | 無法證明全額、正式確認入帳或分次付款已完成 |
| 退款／取消／爭議 | 未檢查 | 無效案件可能產生正向獎金 |
| 90 天保護 | 未檢查 | 推薦保護與最終歸屬可能錯誤 |
| 顧問資格 | 未向顧問主檔驗證 | 不存在、停權或不具資格者可能成為受益人 |
| 規則版本 | 無 | 無法重現計算依據或合法重算 |
| 認列月份 | 無正式參數；掃描全部 leads | 執行範圍不可控，月份可能依非權威日期判定 |
| 防重 | `名單ID + 獎金類型` | 未含訂單／成交、受益人、規則版本與認列月份 |
| 重跑 | 可更新未付款的待審核草稿 | 原始計算與修改歷程可能被覆寫 |
| 執行稽核 | 無 run ID、來源快照或計算版本 | 無法完整追蹤一次執行、輸入及部分失敗 |
| 失敗回復 | 每列捕捉錯誤後繼續，無 rollback | 可能部分成功，且無正式人工復原依據 |
| Preview 一致性 | Preview 對既有列採 skip，Generate 可能 update | 預覽結果不能精確預告實際寫入效果 |

## 4. 正式獎金草稿權威資料來源

### 4.1 資料責任原則

1. CRM 欄位只能作為候選索引與來源提示，不得單獨證明成交、全額入帳或正式獎金成立。
2. 同一事實只能指定一個權威來源；其他 Sheet 的同義欄位只作摘要，衝突時必須阻擋，不得自動擇一。
3. 正式草稿必須保存所採用的來源識別、規則版本、認列月份及不可逆的安全摘要，以便日後重現及稽核。
4. 權威資料缺漏、衝突或無法驗證時一律 fail-closed，不得以預設值補齊後產生草稿。

### 4.2 權威來源矩陣

| 資料域 | 權威來源／必要識別 | 草稿所需事實 | 不得作為唯一依據 |
|---|---|---|---|
| 顧問主檔 | `consultants_顧問主檔`、顧問ID | 顧問存在、合作／啟用狀態、獎金資格、資格生效區間 | lead 上的顧問姓名或顧問ID摘要 |
| Leads／推薦起點 | `leads_潛在會員名單`、名單ID | 推薦碼、初始推薦事件、候選歸屬、測試資料標記 | `是否已付款`、CRM 階段或成交金額摘要 |
| 推薦歸屬與 90 天保護 | 推薦事件／歸屬判定資料、名單ID、顧問ID、保護起訖與決議識別 | 推薦時點、有效推薦碼、是否在保護期、最終歸屬、人工改派及爭議結果 | 單一目前歸屬欄位且無判定證據 |
| 成交 | 訂單或成交權威資料、訂單／成交ID | 必要契約完成、最終方案、完整人數、成交有效狀態 | lead 的 CRM 成交狀態 |
| 付款／入帳 | 正式付款／入帳紀錄、付款ID、訂單ID、確認人與確認時間 | 應收總額、累計確認入帳、未收餘額、全額入帳時間、沖正狀態 | 單一「是否已付款」或付款日期 |
| 退款／取消 | 正式退款、沖正、訂單取消或財務決議資料 | 是否取消、退款金額、退款完成時間、對原付款／訂單的關聯 | lead 備註或未結構化文字 |
| 爭議 | 歸屬／付款／資格／合規爭議及決議紀錄 | 爭議類型、目前狀態、責任人、結案決議與時間 | 空白備註或口頭確認 |
| 獎金規則 | 已核准且生效的獎金規則版本 | 獎金類型、金額／公式、資格、適用方案、生效起訖、規則版本ID | 程式常數或未版本化 Sheet 值 |
| 認列月份 | 全額入帳確認時間＋`Asia/Taipei` 規則 | 標準 `YYYY-MM` 認列月份 | 建立時間、更新時間或無法判定的日期 |
| 輸出 | `bonus_records_獎金明細表` | 一個正式防重範圍的一筆有效草稿及其版本／稽核欄位 | 直接覆寫已審核、已核准或已付款列 |

### 4.3 成交與全額入帳最低判定

只有同時符合以下條件，才可成為可產生候選：

1. 有唯一且有效的來源名單ID及訂單／成交ID。
2. 必要契約及成交條件已完成。
3. 完整方案、人數、應收金額可確定。
4. 所有有效付款累計等於合法調整後應收金額，未收餘額為零。
5. 全額款項已有公司確認入帳時間。
6. 沒有未處理沖正、退款、取消或結果不明的付款。
7. 分次付款案件只能在全案全額確認入帳後一次認列，不得按已收比例分批計獎。

## 5. 正式阻擋規則

所有 preview 與 generate 必須共用同一套 eligibility evaluator 及 reason code；不得由兩條邏輯各自判定。

| Reason code | 阻擋條件 | 分類 |
|---|---|---|
| `SOURCE_ID_MISSING` | 名單、訂單或成交必要識別缺漏 | error |
| `SOURCE_CONFLICT` | 來源識別或關聯互相矛盾 | error |
| `NOT_FULLY_PAID` | 尚未全額付款或未收餘額不為零 | skip |
| `RECEIPT_NOT_CONFIRMED` | 尚無公司確認入帳事實／時間 | skip |
| `PAYMENT_RESULT_UNKNOWN` | 付款、沖正或對帳結果不明 | error |
| `ORDER_CANCELLED` | 訂單／成交已取消 | skip |
| `REFUND_EXISTS` | 已退款、退款中或退款金額未結清 | skip |
| `DISPUTE_OPEN` | 歸屬、付款、資格或合規爭議尚未結案 | skip |
| `CONSULTANT_NOT_FOUND` | 受益顧問不存在於顧問主檔 | error |
| `CONSULTANT_INACTIVE` | 適用時點顧問停權、終止或未啟用 | skip |
| `CONSULTANT_NOT_ELIGIBLE` | 不具該獎金類型的有效資格 | skip |
| `OFFICIAL_CONSULTANT_HC0000` | 最終受益顧問為官方顧問 `HC0000` | skip |
| `REFERRAL_CODE_INVALID` | 推薦碼缺漏、無效或與來源事件不一致 | error |
| `PROTECTION_STATUS_UNKNOWN` | 90 天保護起訖或判定無法確認 | error |
| `ATTRIBUTION_UNRESOLVED` | 最終推薦歸屬不明、衝突或待人工改派 | error |
| `RULE_VERSION_MISSING` | 找不到已核准且適用的獎金規則版本 | error |
| `RULE_VERSION_OVERLAP` | 同一適用時點命中多個規則版本 | error |
| `RECOGNITION_MONTH_UNKNOWN` | 無法由確認入帳時間判定 `YYYY-MM` | error |
| `RECOGNITION_MONTH_MISMATCH` | 候選認列月份與本次要求月份不同 | skip |
| `TEST_DATA` | 任一權威來源標記為測試資料 | skip |
| `EXISTING_LOCKED_RECORD` | 同防重範圍已有待審核後、已核准、待付款或已付款等不可覆寫紀錄 | skip |
| `DUPLICATE_ACTIVE_RESULT` | 同防重範圍存在多筆有效結果 | error |
| `DATA_VALIDATION_FAILED` | 金額、日期、狀態或必要欄位型別不合法 | error |

補充規則：

- 「停權／終止是否影響已在有效推薦保護內的基本推薦獎金」若依正式政策有例外，必須由規則版本及資格快照明確判定，不可在 evaluator 中自行猜測。
- `skip` 表示資料完整且可明確判定本次不產生；`error` 表示資料或規則不完整，必須人工處理後才能重跑。
- 同一筆候選可以回傳多個 reason code；摘要以主要原因計數時仍須保留完整原因陣列。

## 6. Dry-run／Preview 規格

### 6.1 安全邊界

- Preview 不得建立 Sheet、補表頭、append、update、寫入稽核列或修改任何業務資料。
- Preview 不得呼叫 `ensureBonusRecordsSheet()`。
- Preview 可產生僅存在於回應中的 `previewRunId`，但不得因此寫入正式 Sheet。
- Preview 與 Generate 必須使用相同正規化、權威來源讀取、資格判定、防重、金額及輸出 mapping；差異只能是 Generate 取得寫入鎖並提交資料。

### 6.2 必要輸入

| 欄位 | 規格 |
|---|---|
| `recognitionMonth` | 必填；格式 `YYYY-MM`；依 `Asia/Taipei` 驗證 |
| `requestedBy` | 受控操作角色的安全識別或系統角色，不得存 token |
| `requestReason` | 本次預覽目的／工單識別 |
| `scope` | 明確限定獎金類型及認列月份；不得預設掃描全部歷史月份 |

### 6.3 每筆分類

每筆候選必須且只能有一個 action：

- `create`：目前沒有同防重範圍的有效草稿，可建立。
- `update`：只有明確定義為「可更新的未送審草稿」才可更新；須回傳預計變更欄位及原計算版本。若後續採追加版本而非覆寫，應改為 `create_version`，不得假裝 update。
- `skip`：確定不需或不得產生，並附完整排除原因。
- `error`：來源、規則或資料驗證失敗，必須人工處理。

每筆至少回傳：

- 安全候選識別，不含完整個資。
- 來源名單／訂單／成交識別的遮罩或受控值。
- 獎金類型、受益顧問ID、規則版本及認列月份。
- action、主要 reason code、完整 reason codes。
- 預計金額、幣別與來源摘要雜湊。
- 現有 bonus record ID／狀態（適用時）。
- 可供人工核對的非敏感差異摘要。

### 6.4 彙總

Preview 回應至少包含：

- 候選總筆數 `candidateCount`
- 可新增筆數 `createCount`
- 可更新／建立新版本筆數 `updateCount`
- 跳過筆數 `skipCount`
- 錯誤筆數 `errorCount`
- 各 reason code 筆數 `reasonSummary`
- 輸入範圍、認列月份、規則版本集合及來源資料時間點
- `previewRunId` 與安全的輸入指紋 `inputFingerprint`

所有分類加總必須等於候選總筆數。若加總不平、資料讀取中途失敗、規則版本變動或來源指紋無法完成，整次 Preview 必須標記失敗，不得宣告可 Generate。

## 7. Generate 規格

### 7.1 必要輸入與前置條件

Generate 必須接收：

- 必填 `recognitionMonth`，不得預設當月或掃描全部歷史資料。
- 已人工覆核的 `previewRunId` 或等價受控核准識別。
- Preview 的 `inputFingerprint`、規則版本集合及候選統計。
- 操作角色安全識別與執行原因。

Generate 開始前必須重新讀取來源並重算指紋；若與已覆核 Preview 不一致，整次阻擋並要求重新 Preview。

### 7.2 執行要求

1. 每次實際執行產生唯一 `calculationRunId`／run ID。
2. 取得 Script Lock，鎖定範圍至少包含獎金類型與認列月份。
3. 共用 Preview eligibility evaluator，不得略過任何阻擋條件。
4. 寫入規則版本、認列月份、來源識別、安全來源摘要、防重鍵、計算版本及 run ID。
5. 新草稿初始狀態只能是「待審核」或正式狀態機定義的等價初始值。
6. 不得自動核准、不得自動標示可撥款、不得建立付款批次或付款結果。
7. 不得覆寫已送審、已審核、已核准、待付款、已付款、取消或追回等鎖定資料。
8. 未送審草稿若允許更新，必須保留原版本、前後差異與更新原因；優先採新計算版本而非原列靜默覆寫。
9. 任一候選的權威來源、規則或輸出驗證失敗時 fail-closed，不得以 0 元、當月、未知狀態或空值替代。
10. 結束時回傳 create／update／skip／error 與 Preview 相同的分類及 reason code。

### 7.3 部分成功與人工回復

- 執行須保存 run 層級狀態：`PREPARED`、`COMMITTED`、`PARTIAL_FAILURE`、`FAILED`。
- 每筆寫入必須關聯 run ID、operation item ID 及結果；不得只保留總數。
- 發生部分成功時，不得自動刪除已建立草稿，也不得直接重跑全部候選。
- 人工回復報告必須列出已建立、已建立新版本、跳過、失敗及狀態不明項目。
- 重試只能針對明確未提交且來源指紋未變的項目；已提交項目必須由正式防重鍵命中。
- 若無法確認某筆是否已提交，標記結果不明並阻擋自動重試，先進行人工對帳。

## 8. 正式防重鍵

### 8.1 最低組成

正式業務防重範圍至少包含：

```text
source_lead_id
+ source_order_or_deal_id
+ bonus_type
+ beneficiary_consultant_id
+ bonus_rule_version_id
+ recognition_month
```

若某獎金類型不是逐筆成交型，必須以正式期間計算快照ID取代單一訂單／成交ID，但仍須包含獎金類型、受益顧問、規則版本及認列期間。

### 8.2 產生與保存

- 各組成值先依封版格式正規化，再以固定欄位順序建立 canonical string。
- Sheet 保存不可逆安全摘要作為 `idempotency_key`，並分欄保存必要業務識別以利人工稽核。
- 防重摘要不得包含姓名、手機、LINE UID、token 或其他可逆敏感資料。
- 同防重範圍只能有一個有效正式結果；合法重算以遞增 calculation version、調整或追回關聯表示。
- 規則版本或合法來源更正造成防重範圍改變時，不得留下兩筆無關聯的有效同義獎金，必須建立 supersedes／adjustment 關聯並走人工審核。

## 9. `bonus_records_獎金明細表` 建議補強欄位

現有 19 欄可保留作 MVP 顯示相容欄；正式產生器至少應補強下列欄位。實際 Header 順序、型別、下拉值及遷移方式須在 Task 04H-2M-2 前另行核對，不得由 `ensureBonusRecordsSheet()` 未經審查直接追加至正式 Sheet。

| 建議 Header | 程式鍵 | 用途 |
|---|---|---|
| 計算Run ID | `calculation_run_id` | 關聯一次 Generate 執行 |
| Preview Run ID | `preview_run_id` | 關聯人工覆核的 Preview |
| 作業項目ID | `operation_item_id` | 追蹤部分成功及重試 |
| 來源訂單／成交ID | `source_order_or_deal_id` | 保存正式成交來源 |
| 來源類型 | `source_type` | 區分逐筆成交或期間快照 |
| 來源快照摘要 | `source_snapshot_hash` | 驗證輸入是否改變，不含可逆個資 |
| 獎金規則版本ID | `bonus_rule_version_id` | 重現適用規則 |
| 認列月份 | `recognition_month` | `YYYY-MM`，Asia/Taipei |
| 防重鍵 | `idempotency_key` | 正式冪等與重跑命中 |
| 計算版本 | `calculation_version` | 合法重算遞增版本 |
| 產生模式 | `generation_mode` | `GENERATE`；Preview 不得寫入 |
| 排除／錯誤原因 | `reason_codes` | 僅在正式設計允許保存失敗項目時使用；不可混入有效獎金列 |
| 計算輸入摘要 | `calculation_basis_summary` | 非敏感規則與來源摘要 |
| 審核狀態 | `review_status` | 正式狀態機目前摘要 |
| 送審人／時間 | `submitted_by`／`submitted_at` | 送審稽核 |
| 審核人／時間 | `reviewed_by`／`reviewed_at` | 審核稽核 |
| 核准人／時間 | `approved_by`／`approved_at` | 核准稽核 |
| 狀態流轉事件引用 | `status_event_ref` | 指向不可變狀態事件 |
| 前一計算版本／取代關聯 | `supersedes_bonus_record_id` | 關聯合法重算或更正 |

建議將 run、失敗 item、狀態事件與完整來源 basis 拆成獨立受控資料表；`bonus_records` 只保存必要摘要與引用，避免把執行日誌、敏感來源及正式獎金混為同一寬表。

## 10. API／UI 正式開放前補強

### 10.1 Read-only API

- 無法判定認列月份的資料不得保守納入任意月份；應排除並回報內部資料品質錯誤。
- 非數字、負數或不合法金額不得靜默轉為 0；應 fail-closed，且不得向顧問顯示錯誤金額。
- 未知狀態不得當成正常資料顯示；須有封版的內部狀態至顧問可見狀態映射，未知值應隔離並告警。
- 正式資料產生後必須重驗：顧問本人隔離、月份篩選、狀態篩選、limit 上限、empty、ready、token、`consultantId` 越權、GET 拒絕及欄位白名單。
- 必須再次確認 API 不建表、不補表頭、不更新資料、不呼叫 Preview／Generate，也不回傳來源快照、規則內部參數、備註、完整會員資料或財務敏感欄位。

### 10.2 LIFF UI

- `consultantBonusEnabled` 必須維持 `false`，直到 Task 04H-2N 正式資料與查詢驗收通過。
- 正式資料產生後須驗收 disabled、loading、`not_initialized`、`ready + empty`、content、token error、access denied、invalid argument 與 internal error。
- 須驗收月份切換、狀態切換、50 筆 limit、未知狀態隔離、長內容、手機 LIFF WebView 日期及金額格式。
- 顧問端不得顯示完整來源識別、防重鍵、run ID、規則內部內容、會員完整姓名／手機、其他顧問收入或內部錯誤原因。

## 11. 實作與驗收通過條件

Task 04H-2M-2 至少必須證明：

1. Preview 完全唯讀，且與 Generate 使用相同 evaluator／mapper。
2. recognition month 必填並限制執行範圍。
3. 正式權威來源缺漏或衝突時 fail-closed。
4. 固定 4,000 元已被核准規則版本取代。
5. 防重鍵、run ID、計算版本與來源摘要可追溯。
6. 已審核／已核准／已付款資料不可覆寫。
7. 部分成功、結果不明及重試有明確人工回復路徑。
8. Preview 與 Generate 的分類、reason code 及統計可逐筆對帳。
9. 不會因執行產生自動核准、排款或付款狀態。
10. 自動測試與測試副本驗收均通過後，才可規劃正式受控 Generate。

## 12. 後續任務切分

### Task 04H-2M-2：Apps Script dry-run／generate guardrail 實作

- 依本規格補強權威來源 adapter、共同 evaluator、月份參數、reason code、防重、run／item 稽核及部分失敗處理。
- 同步補強 API 對無法判月、非法金額及未知狀態的 fail-closed 行為。
- 不得直接對正式 Sheet 執行 Generate。

### Task 04H-2M-3：測試資料副本驗收

- 使用隔離測試 Sheet 副本驗證 Preview／Generate 一致性、阻擋規則、防重、重跑、部分失敗及回復。
- 覆蓋全額入帳、分次付款、退款、取消、爭議、90 天保護、顧問停權、規則重疊與跨月份案例。

### Task 04H-2I-Final：正式 `ready + empty` 驗收紀錄補件

- 補齊正式 Apps Script 同步、正式 Web route 與 `bonus_records` 已存在但無資料時的 `ready + empty` 證據。
- 確認 read-only route 未寫入、未產生草稿及未洩漏敏感欄位。

### Task 04H-2N：正式草稿受控產生與顧問查詢開放驗收

- 先受控 Preview、人工覆核，再限定月份執行 Generate。
- 逐筆核對正式草稿、重驗 read-only API 與 LIFF UI。
- 全部驗收通過後，才可另依明確核准評估將 `consultantBonusEnabled` 改為 `true`。

建議執行順序：`04H-2M-2` → `04H-2M-3` → `04H-2I-Final` → `04H-2N`。

## 13. 安全聲明

- 本文件不授權修改前端或 Apps Script。
- 本文件不授權建立、補欄、修改或清理任何正式 Google Sheet。
- 本文件不授權執行 `ensureBonusRecordsSheet()`、Preview 或 `generateConsultantBonusDrafts()`。
- 本文件不授權產生正式獎金草稿、審核、核准、排款或付款。
- 本文件不授權部署或開啟 `consultantBonusEnabled`。

