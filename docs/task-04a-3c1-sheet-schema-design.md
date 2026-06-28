# Task 04A-3C-1：正式 Sheet 實體結構與欄位映射設計

## 0. 文件資訊、範圍與安全邊界

| 項目 | 內容 |
|---|---|
| 文件狀態 | 實體結構設計草案，待審核；不代表已核准建表或遷移 |
| 文件版本 | v1.0-draft |
| 建立日期 | 2026-06-28 |
| 上游依據 | Task 04A-1、Task 04A-2、Task 04A-3A、Task 04A-3B |
| 適用範圍 | 現行 Google Sheet＋Apps Script 架構下的實體分頁、欄位、主鍵、關聯、版本、不可變性、敏感隔離與分階段落地設計 |
| 不在本輪 | 建立分頁、修改 Header、搬移或清理資料、執行正式 Sheet 診斷、實作 Apps Script、啟用功能、部署或發布 |

本文件只描述結構與後續控制責任，不讀寫正式 Google Sheet，也不把既有 Sheet 的任何摘要欄位直接升格為財務主帳。Apps Script 參考僅用於理解現有以中文 Header 映射物件、批次讀取及 `LockService` 包裝等結構；敏感設定值、正式顧問識別、個資、URL 與憑證均不記錄於本文件。

本文使用的敏感分級如下：

| 等級 | 定義 | 例示與控制 |
|---|---|---|
| S0 制度 | 非個資的方案、規則、狀態碼 | 授權規則管理者可維護；歷史版本鎖定 |
| S1 內部 | 內部 ID、資格、組織、營運狀態 | 依職務與管理範圍存取 |
| S2 個資 | 姓名、聯絡資訊、LINE 識別、會員關聯 | 最小蒐集、遮罩、白名單與存取稽核 |
| S3 財務 | 付款、退款、獎金收入、批次與撥款結果 | 管理／審核／財務分權；禁止顧問直接存取 Sheet |
| S4 高敏感 | 銀行資料、付款或撥款憑證、爭議佐證 | 建議隔離工作簿或受控儲存；Sheet 只留安全引用 |

---

## 1. 架構選擇

### 1.1 選項比較

| 選項 | 優點 | 主要風險 | 適用判斷 |
|---|---|---|---|
| 擴充既有六個分頁 | 分頁少、既有程式改動表面較小 | 訂單多會員、多筆付款、事件歷史、快照、批次與撥款 ledger 會被塞進寬表；易覆寫歷史且難防重 | 不建議作正式目標；只適合保留目前摘要與相容欄 |
| 新增正規化分頁 | 關聯、追溯、追加式事件與權限邊界較清楚 | 分頁及關聯增加；Apps Script 必須有明確 repository、索引與批次讀取策略 | 適合交易、事件、快照、批次、撥款與稽核事實 |
| 混合式設計 | 保留六個既有入口，將歷史與財務事實拆到正規化分頁；兼顧相容性與追溯 | 必須清楚標示「權威」與「衍生摘要」，避免雙主帳 | **目前建議** |
| 未來改用資料庫 | 可提供真正唯一鍵、外鍵、交易、索引、列級權限及較佳併發 | 建置與維運成本較高；需設計遷移、雙寫與切換 | 當 Sheet 鎖、配額、掃描、稽核量或多寫入者已無法安全承載時評估 |

### 1.2 建議

採混合式設計，但本輪不改變正式儲存方式：

1. 六個既有分頁全部保留，避免本設計直接破壞既有 Apps Script 與營運流程。
2. `consultants`、`plans`、`orders`、`bonus_records`、`bonus_campaigns` 五個既有分頁後續補強；`leads` 保留 CRM 與推薦來源責任邊界，不擴張為財務主帳。
3. 新增 17 個正規化分頁的設計能力；其中交易、快照、計算依據、批次明細、撥款事件及稽核不得為減少分頁而合併。
4. `payout_attempt`、`payout_result` 與 `payout_result_correction_or_decision` 可共用一張 typed append-only 事件簿，因三者具相同財務權限域與不可變性，並以事件類型及父事件鍵維持語意與防重。
5. 銀行帳戶與憑證是否拆至獨立財務工作簿仍待決策；本文件不建立或搬移。

### 1.3 Google Sheet 固有限制

- Google Sheet 沒有真正外鍵、唯一約束或跨分頁 transaction；不得把程式呼叫順序當成全有或全無提交。跨分頁重要作業須使用 `operation_id`、提交事件與正式讀取過濾，提供邏輯全有或全無可見性；實體列仍可能因中斷而部分存在。
- 所有主鍵、關聯存在性、唯一鍵及金額平衡未來須由服務層驗證，並以稽核事件記錄結果。
- 多個 Web App、排程或人工操作可能併發寫入；未來寫入須以 `LockService`、冪等鍵、鎖擁有者及提交前再次查驗降低競態，但鎖不能取代資料庫交易。
- 權限主要在檔案層級。隱藏或保護分頁只能降低誤操作，不能視為個資或財務隔離。
- 公式可以顯示或校驗摘要，但不得成為獎金、批次淨額、已發放或追回餘額的唯一權威來源；正式值須由版本化輸入與追加式 ledger 重建。
- 顧問及導師不得直接取得正式 Sheet 權限；只能經可信任身分驗證、欄位白名單、遮罩與授權範圍查詢。

### 1.4 跨分頁提交 journal 方案評估

| 方案 | 優點 | 風險／控制 | 結論 |
|---|---|---|---|
| A. `audit_events` 承載提交事件 | 既有 append-only、correlation、角色、時間、結果與安全差異能力可直接延伸；提交決議與業務稽核在同一鏈；不增加分頁 | 必須把 operation 事件視為正式可見性權威，而非可略過的附帶 log；需固定索引與 reconciliation | **推薦** |
| B. 獨立 operation journal | 職責名稱更直觀，可獨立調校查詢與保留 | 與 audit 的 ID、角色、時間、原因、correlation、結果大量重複；跨兩個 journal 反而新增一致性問題 | 目前無充分必要，不新增 |

本設計採 **方案 A**，分頁總數維持 23。`audit_events` 中 `target_entity_type=OPERATION` 且事件類型為 `OPERATION_PREPARED`、`OPERATION_COMMITTED`、`OPERATION_ABORTED` 或 `OPERATION_RECOVERED` 的列，構成 operation journal 邏輯能力。這四類不是一般資訊性記錄；正式讀取時，`OPERATION_COMMITTED` 是跨分頁權威列可見的必要條件。若未來 audit 量或保留政策導致無法穩定查詢 operation，再另案評估物理拆表，不得在未設計遷移與雙讀期間前自行拆分。

---

## 2. 分頁總覽

### 2.1 既有且保留、需補強與建議新增

| 類別 | 正式建議名稱 | 用途 | 權威／摘要 | 主鍵 | 關聯鍵 | 寫入角色 | 可變性 | 敏感 |
|---|---|---|---|---|---|---|---|---|
| 既有保留＋補強 | `consultants_顧問主檔` | 顧問身分與目前摘要 | 身分權威；狀態／資格／組織為事件衍生摘要 | `consultant_id` | 最新事件 ID | 管理者；摘要程序 | 受控可變 | S1/S2 |
| 既有保留 | `leads_潛在會員名單` | CRM、推薦來源、保護資料 | CRM 權威；付款／成交／獎金僅摘要 | `lead_id` | consultant、member、order | CRM／授權流程 | 受控可變 | S2 |
| 既有保留＋補強 | `plans_方案規則表` | 方案規則版本 | 核准版本權威 | `plan_rule_version_id` | previous version | 規則管理／核准 | 核准後不可變 | S0 |
| 既有保留＋補強 | `orders_訂單主檔` | 訂單核心與目前交易摘要 | 訂單權威；會員、付款、快照與獎金為摘要 | `order_id` | lead、plan version | 營運／交易服務 | 成交後核心鎖定 | S2/S3 |
| 既有保留＋補強 | `bonus_records_獎金明細表` | 一獎項一受益人正式明細 | 獎金權威；撥款摘要可重建 | `bonus_record_id` | source snapshot、rule version | 計算／營運／審核 | 原始試算與核准值不可覆寫 | S3 |
| 既有保留＋補強 | `bonus_campaigns_活動獎金規則表` | 活動與期別版本 | 核准版本權威 | `bonus_campaign_version_id` | campaign、previous version | 規則管理／核准 | 核准後不可變 | S0 |
| 新增 | `consultant_status_events_顧問狀態事件` | 啟用、停權、終止、恢復 | 權威事件 | `consultant_status_event_id` | consultant | 管理／核准 | 追加式不可變 | S1 |
| 新增 | `mentor_qualification_events_導師資格事件` | 晉升、考核、暫停、喪失、恢復 | 權威事件 | `mentor_qualification_event_id` | consultant | 營運／審核／核准 | 追加式不可變 | S1 |
| 新增 | `organization_relation_events_組織關係事件` | 加入、移轉、分支獨立、承接 | 權威事件 | `organization_relation_event_id` | consultant、parent | 管理／核准 | 追加式不可變 | S1 |
| 新增 | `members_會員主檔` | 正式會員最小身分 | 會員識別權威 | `member_id` | source lead | 會員服務／授權管理 | 依法更正；ID 不變 | S2 |
| 新增 | `order_members_訂單會員關聯` | 一訂單多會員 | 關聯權威 | `order_member_id` | order、member | 訂單服務 | 成交前可改；成交後事件化 | S2 |
| 新增 | `payments_付款入帳明細` | 分次付款及入帳確認 | 收款權威 | `payment_id` | order、external reference | 財務／收款服務 | 確認後不可覆寫 | S3/S4 |
| 新增 | `order_adjustments_訂單退款調整` | 退款、取消、降級、換方案 | 調整權威 | `order_adjustment_id` | order、payment、新 order | 營運／財務／核准 | 追加式不可變 | S3/S4 |
| 新增 | `bonus_rules_常態獎金規則表` | 常態獎金與組織規則版本 | 核准版本權威 | `bonus_rule_version_id` | previous version | 規則管理／核准 | 核准後不可變 | S0 |
| 新增 | `deal_snapshots_成交快照` | 全額入帳時固化成交輸入 | 不可變權威 | `deal_snapshot_id` | order、plan/rule versions | 成交快照程序 | 追加式不可變 | S1/S3 |
| 新增 | `organization_snapshots_組織快照` | 固化成交當下路徑與導師判定 | 不可變權威 | `organization_snapshot_id` | deal snapshot、consultant | 成交快照程序 | 追加式不可變 | S1 |
| 新增 | `bonus_calculations_獎金計算快照` | 期間彙總的一次計算版本 | 不可變權威 | `bonus_calculation_snapshot_id` | rule/campaign version | 計算程序 | 追加式不可變 | S3 |
| 新增 | `bonus_basis_items_獎金計算依據明細` | 月結逐筆納入、扣除、排除 | 不可變權威 | `bonus_basis_item_id` | calculation snapshot、source | 計算程序 | 追加式不可變 | S1/S3 |
| 新增 | `bonus_adjustments_獎金調整追回` | 人工調整、取消差額、追回與扣抵 | 權威 ledger | `bonus_adjustment_id` | original bonus、refund | 營運／審核／核准／財務 | 追加式不可變 | S3/S4 |
| 新增 | `settlement_batches_結算批次` | 週結／月結批次生命週期 | 批次權威 | `settlement_batch_id` | recognition period | 系統／審核／核准／財務 | 鎖定前版本化；核准後不可變 | S3 |
| 新增 | `settlement_batch_items_結算批次明細` | 每次批次不可變納入內容 | 財務核心權威 | `settlement_batch_item_id` | batch、bonus/adjustment | 組批程序 | 鎖定後不可變 | S3 |
| 新增 | `payout_events_撥款事件簿` | attempt、result、更正／決議 | 撥款 ledger 權威 | `payout_event_id` | batch item、parent event | 財務／覆核 | 追加式不可變 | S3/S4 |
| 新增 | `audit_events_稽核事件` | 敏感操作、狀態、規則、批次稽核及 operation 提交 journal | 稽核與提交可見性權威 | `audit_event_id` | operation、polymorphic target | 所有服務／授權角色 | 追加式不可變 | S1-S4 |

### 2.2 暫不建立但須保留的能力

| 能力 | 本輪保留方式 | 啟動條件 |
|---|---|---|
| 獨立財務工作簿 | 所有高敏感欄只定義安全引用，不在一般分頁複製憑證或完整帳戶 | 權限、營運與稽核決策完成後 |
| 資料庫 | ID、版本、外鍵與防重鍵採可遷移設計 | 併發、掃描、配額、資料量或列級權限需求超出 Sheet 安全邊界時 |
| 歷史封存工作簿 | 每張追加式表保留封存批次與來源 ID 能力 | 保留期限、查詢 SLA 及封存 SOP 核准後 |
| 付款方式／銀行帳戶專用主檔 | 目前只在 payment／payout 留 token 或受控引用 | 財務決定資料域、加密／權限與更新責任後 |

---

## 3. 核心實體映射

### 3.1 Task 04A-3A 的 20 個核心實體

| 邏輯實體 | 實體分頁 | 映射方式 |
|---|---|---|
| `consultant` | `consultants_顧問主檔` | 一列一顧問；只保留目前摘要，歷史由事件表承載 |
| `consultant_status_event` | `consultant_status_events_顧問狀態事件` | 一列一不可變狀態事件 |
| `mentor_qualification_event` | `mentor_qualification_events_導師資格事件` | 一列一不可變資格／考核事件 |
| `lead` | `leads_潛在會員名單` | 沿用 CRM 主檔；成交時只以快照固定其必要證據 |
| `member` | `members_會員主檔` | 一列一穩定會員 ID；健康資料不進本表 |
| `order` | `orders_訂單主檔` | 一列一案件；付款與成員拆表 |
| `order_member` | `order_members_訂單會員關聯` | 一列一 order-member 關聯 |
| `payment` | `payments_付款入帳明細` | 一列一收款／入帳事實；沖正另列 |
| `refund_or_adjustment` | `order_adjustments_訂單退款調整` | 一列一退款／取消／方案調整事件 |
| `plan_rule_version` | `plans_方案規則表` | 既有表改採一列一版本，不覆寫已引用版本 |
| `bonus_rule_version` | `bonus_rules_常態獎金規則表` | 一列一常態規則版本 |
| `bonus_campaign_version` | `bonus_campaigns_活動獎金規則表` | 既有表改採一列一活動版本／期別 |
| `deal_snapshot` | `deal_snapshots_成交快照` | 一列一成交快照版本 |
| `organization_snapshot` | `organization_snapshots_組織快照` | 一列一成交的組織判定；完整路徑可用穩定 ID 序列化欄 |
| `bonus_calculation_snapshot` | `bonus_calculations_獎金計算快照` | 一列一受益人×獎金類型×期間×計算版本 |
| `bonus_calculation_basis_item` | `bonus_basis_items_獎金計算依據明細` | 一列一來源及效果；包含排除項 |
| `bonus_record` | `bonus_records_獎金明細表` | 一列一獎項對一位受益顧問 |
| `bonus_adjustment_or_recovery` | `bonus_adjustments_獎金調整追回` | 一列一調整／追回／扣抵 ledger 事件 |
| `settlement_batch` | `settlement_batches_結算批次` | 一列一批次版本 |
| `audit_event` | `audit_events_稽核事件` | 一列一追加式稽核事件 |

`organization_relation_event` 是 20 個核心實體之外、但 Task 04A-3A 明確要求保留的必要能力，獨立映射到 `organization_relation_events_組織關係事件`；否則無法依指定時間還原成交前的組織。

### 3.2 Task 04A-3B 的四個實體

| 邏輯實體 | 實體分頁 | `事件類型` | 主／父鍵 |
|---|---|---|---|
| `settlement_batch_item` | `settlement_batch_items_結算批次明細` | 不共表 | batch item ID／batch ID |
| `payout_attempt` | `payout_events_撥款事件簿` | `ATTEMPT` | payout event ID／batch item ID |
| `payout_result` | `payout_events_撥款事件簿` | `RESULT` | payout event ID／attempt event ID |
| `payout_result_correction_or_decision` | `payout_events_撥款事件簿` | `CORRECTION_DECISION` | payout event ID／original result event ID |

三種 payout 邏輯實體共表不破壞控制，理由如下：

- 防重：`ATTEMPT` 以 `attempt_idempotency_key` 唯一；`RESULT` 另以 `result_idempotency_key` 驗證同一外部／人工結果的重複回傳；更正／決議以自身 ID 與原 result 關聯，不覆寫原列。
- 權限：三者均屬財務高敏感資料，使用相同財務寫入角色與讀取邊界；顧問 API 只讀推導摘要。
- 不可變：整張表只允許 append，事件類型一經建立不可改；不允許把 attempt 列原地補成 result。
- 語意：事件類型專屬必填欄與父事件存在性由未來服務層驗證；不適用欄統一使用空值或規格化 `NOT_APPLICABLE`，不得混用自由文字。

---

## 4. 欄位字典共通規則

1. Header 採既有「英文分頁名_中文名稱」分頁風格與中文 Header；程式以本節的英文鍵集中映射，業務程式不得散落硬編碼 Header。
2. ID 為大小寫固定的非語意字串。示意格式只描述 prefix 與不可重用原則，不代表本輪建立流水號。
3. `DATETIME_TZ` 表示 ISO 8601 時間並另有時區欄；週月結一律保存 `Asia/Taipei`，不得依試算表 locale 猜測。
4. `DECIMAL(…,2)` 用於金額，另存幣別；`DECIMAL(…,1)` 用於可含 0.5 的件數，禁止二進位浮點累加作財務權威。
5. `ID_LIST`／`JSON_CANONICAL` 僅用於不可變快照中的正規化 ID 或安全結構；寫入前須固定排序、schema version 與指紋，不得塞入完整個資或憑證。
6. 規則金額、門檻、活動期間與適用條件不得作欄位預設值；交易與獎金列必須引用已核准規則版本。
7. 下列字典是目標結構；本輪不修改任何現有 Header。
8. 參與跨分頁重要作業的權威列必須保存同一穩定 `operation_id`；`correlation_id` 用於串聯更廣的重試、重算、退款或對帳流程，兩者不得混為一個可變欄位。
9. `operation_id` 所屬作業只有在 `audit_events` 存在有效 `OPERATION_COMMITTED` 事件後才對正式計算、審核、批次與顧問查詢可見。PREPARED、ABORTED、無終局事件或事件衝突的列只能供 reconciliation／稽核讀取。
10. 摘要欄不保存提交權威；commit 後才可重建。摘要更新失敗不得撤銷或污染已提交權威列，也不得讓未提交列變得可見。

---

## 5. 每張分頁欄位字典

### 5.1 `consultants_顧問主檔`（既有，補強目標）

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|顧問ID|consultant_id|TEXT|建立|是|否|consultant|`CNS-...`；不可重用|S1|不使用姓名或外部 ID 當主鍵|
|2|顧問姓名|consultant_name|TEXT|建立|否|依法更正|consultant|非空；輸出須遮罩／授權|S2|不得寫入程式設定|
|3|顧問狀態|current_consultant_status|CODE|事件建立後|否|僅摘要程序|status event|由最新有效事件推導|S1|沿用既有 Header；非歷史權威|
|4|目前位階|current_rank_code|CODE|適用時|否|僅摘要程序|qualification event|核准代碼表|S1|顯示中文另映射|
|5|導師資格狀態|current_mentor_status|CODE|適用時|否|僅摘要程序|qualification event|由事件推導|S1|沿用既有 Header；不得單靠布林旗標|
|6|目前上層顧問ID|current_parent_consultant_id|ID|建立組織後|否|僅摘要程序|organization event|須存在 consultant|S1|不可作歷史快照|
|7|目前組織路徑|current_organization_path|ID_LIST|建立組織後|否|僅摘要程序|organization event|無循環、ID 存在|S1|查詢摘要|
|8|最新狀態事件ID|latest_status_event_id|ID|事件後|否|僅摘要程序|status event|須存在事件|S1|索引用|
|9|最新資格事件ID|latest_qualification_event_id|ID|事件後|否|僅摘要程序|qualification event|須存在事件|S1|索引用|
|10|最新組織事件ID|latest_organization_event_id|ID|事件後|否|僅摘要程序|organization event|須存在事件|S1|索引用|
|11|權限狀態|access_status_code|CODE|建立|否|授權流程|access policy|授權碼表|S1|停權應即時反映|
|12|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|不可覆寫|
|13|最後更新時間|updated_at|DATETIME_TZ|更新|否|系統|system|不得早於建立|S1|只表示摘要更新|

### 5.2 `leads_潛在會員名單`（既有保留）

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|名單ID|lead_id|TEXT|建立|是|否|lead|`LED-...`；不可重用|S1|既有 Header 優先沿用|
|2|潛在會員姓名|lead_name|TEXT|取得同意後|否|依法更正|lead|輸出遮罩／授權|S2|不得複製到快照|
|3|手機號碼|phone|TEXT|流程需要時|否|依法更正|lead|格式驗證；輸出遮罩|S2|不可作主鍵|
|4|推薦碼|referral_code|TEXT|推薦建立時|否|受控|lead/referral|不得在記錄輸出完整值|S2|外部識別不可取代內部 ID|
|5|首次歸屬顧問ID|initial_owner_consultant_id|ID|歸屬建立時|否|否|lead attribution|須存在 consultant|S1|歷史起點|
|6|目前歸屬顧問ID|current_owner_consultant_id|ID|歸屬建立時|否|決議流程|lead attribution|須有決議／事件|S1|成交以 deal snapshot 為準|
|7|推薦保護開始時間|protection_started_at|DATETIME_TZ|保護建立時|否|決議流程|lead attribution|開始不得晚於到期|S1|不寫死天數|
|8|推薦保護到期時間|protection_ends_at|DATETIME_TZ|保護建立時|否|決議流程|lead attribution|引用制度／設定|S1|HC9001 修正前不可自動信任|
|9|歸屬爭議狀態|attribution_dispute_status|CODE|建立|否|事件流程|lead attribution|既定爭議碼|S1|不直接等同獎金狀態|
|10|轉換會員ID|converted_member_id|ID|轉換後|否|系統|member|須存在 member|S2|可空|
|11|最近轉換訂單ID|latest_converted_order_id|ID|轉換後|否|系統|order|須存在 order|S1|同 lead 可有多案，僅摘要|
|12|CRM階段|crm_stage_code|CODE|建立|否|CRM流程|lead|CRM 允許值|S1|非付款／成交權威|
|13|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|14|最後更新時間|updated_at|DATETIME_TZ|更新|否|系統|system|不得早於建立|S1|—|

### 5.3 `plans_方案規則表`（既有，補強為版本表）

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|方案規則版本ID|plan_rule_version_id|TEXT|建立|是|否|plan rule version|`PRV-...`|S0|每列一版本|
|2|方案ID|plan_id|TEXT|建立|否|否|plan family|穩定方案族群 ID|S0|同方案可多版本|
|3|方案名稱|plan_name|TEXT|草稿|否|草稿可改|plan rule version|非空|S0|顯示用途|
|4|方案類型|plan_type_code|CODE|草稿|否|草稿可改|plan rule version|核准代碼表|S0|—|
|5|適用最少人數|min_member_count|INTEGER|草稿|否|草稿可改|plan rule version|大於 0|S0|不可在程式寫死|
|6|適用最多人數|max_member_count|INTEGER|草稿|否|草稿可改|plan rule version|空值代表無上限；不得小於最少|S0|—|
|7|金額計算模式|price_calculation_mode|CODE|草稿|否|草稿可改|plan rule version|固定／參數公式等核准碼|S0|不允許自由公式直接執行|
|8|金額參數|price_parameters|JSON_CANONICAL|草稿|否|草稿可改|plan rule version|符合版本化 schema|S0|交易只引用版本|
|9|基本獎金計算模式|base_bonus_mode|CODE|草稿|否|草稿可改|plan rule version|核准碼表|S0|—|
|10|基本獎金參數|base_bonus_parameters|JSON_CANONICAL|草稿|否|草稿可改|plan rule version|符合 schema|S0|不得作欄位預設值|
|11|件數計算模式|unit_calculation_mode|CODE|草稿|否|草稿可改|plan rule version|核准碼表|S0|支援 0.5|
|12|件數參數|unit_parameters|JSON_CANONICAL|草稿|否|草稿可改|plan rule version|精度 0.5|S0|—|
|13|幣別|currency_code|CODE|草稿|否|草稿可改|plan rule version|ISO 4217|S0|—|
|14|版本狀態|rule_version_status|CODE|建立|否|依狀態機|rule lifecycle|DRAFT/APPROVED/ACTIVE/DEACTIVATED|S0|核准後鎖定內容|
|15|生效時間|effective_at|DATETIME_TZ|核准前|否|核准後否|rule lifecycle|不得重疊|S0|—|
|16|失效時間|expires_at|DATETIME_TZ|適用時|否|核准後否|rule lifecycle|晚於生效|S0|—|
|17|前一版本ID|previous_version_id|ID|非首版|否|否|rule lifecycle|須存在同方案版本|S0|—|
|18|核准者安全識別|approved_by_actor_id|ID|核准時|否|否|audit|授權角色|S1|不放姓名|
|19|核准時間|approved_at|DATETIME_TZ|核准時|否|否|audit|有效時間|S1|—|

### 5.4 `orders_訂單主檔`（既有，補強目標）

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|訂單ID|order_id|TEXT|建立|是|否|order|`ORD-...`|S1|不可重用|
|2|來源名單ID|lead_id|ID|適用時|否|成交前|lead|須存在 lead|S2|可空；非付款依據|
|3|方案規則版本ID|plan_rule_version_id|ID|確認方案時|否|成交後否|plan version|須為唯一適用版本|S0|不得只存方案名稱|
|4|訂單狀態|order_status_code|CODE|建立|否|流程更新|order|核准交易碼表|S1|目前摘要|
|5|契約完成時間|contract_completed_at|DATETIME_TZ|成交前|否|確認後否|contract process|有效佐證引用|S2|三要件之一|
|6|應收金額|receivable_amount|DECIMAL(18,2)|確認方案時|否|成交後否|order/plan version|非負；依版本計算|S3|公式僅校驗|
|7|幣別|currency_code|CODE|確認方案時|否|成交後否|order|ISO 4217|S0|—|
|8|實際會員人數摘要|member_count_summary|INTEGER|成員建立後|否|僅摘要程序|order_member|大於 0|S1|非權威|
|9|累計確認入帳摘要|confirmed_amount_summary|DECIMAL(18,2)|付款後|否|僅摘要程序|payment|不得超出合法調整後應收|S3|非權威|
|10|未收餘額摘要|outstanding_amount_summary|DECIMAL(18,2)|付款後|否|僅摘要程序|payment/order adjustment|可重建|S3|非權威|
|11|付款狀態摘要|payment_status_summary|CODE|付款後|否|僅摘要程序|payment|由明細推導|S3|非財務主帳|
|12|成交快照ID|current_deal_snapshot_id|ID|成交後|否|僅指向後繼版|deal snapshot|須存在且屬同訂單|S1|原快照不可回寫|
|13|成交時間摘要|deal_confirmed_at|DATETIME_TZ|成交後|否|僅摘要程序|deal snapshot|三要件最後完成點|S1|—|
|14|退款調整狀態摘要|adjustment_status_summary|CODE|調整後|否|僅摘要程序|order adjustment|由事件推導|S3|—|
|15|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|16|最後更新時間|updated_at|DATETIME_TZ|更新|否|系統|system|不得早於建立|S1|—|

### 5.5 `bonus_records_獎金明細表`（既有，補強目標）

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|獎金ID|bonus_record_id|TEXT|建立|是|否|bonus record|`BON-...`|S3|一獎項一受益人|
|2|來源類型|source_type|CODE|建立|否|否|bonus record|DEAL_SNAPSHOT/CALCULATION_SNAPSHOT|S1|二擇一|
|3|來源快照ID|source_snapshot_id|ID|建立|否|否|source snapshot|須符合來源類型|S1|期間型不得用代表訂單|
|4|獎金類型|bonus_type_code|CODE|建立|否|否|rule version|Task 04A-1 既定類型|S0|不新增營運語意|
|5|受益顧問ID|beneficiary_consultant_id|ID|建立|否|否|consultant|須存在|S1|—|
|6|常態規則版本ID|bonus_rule_version_id|ID|適用時|否|否|bonus rule version|已核准且適用|S0|不適用採標準值|
|7|活動規則版本ID|bonus_campaign_version_id|ID|適用時|否|否|campaign version|已核准且適用|S0|—|
|8|認列期間|recognition_period|TEXT|建立|否|否|deal/calculation|標準週或月鍵|S1|含時區規則|
|9|有效件數|credited_units|DECIMAL(18,1)|試算|否|否|snapshot|0.5 精度|S1|—|
|10|原始試算金額|original_calculated_amount|DECIMAL(18,2)|試算|否|否|calculation|非負；幣別一致|S3|永不覆寫|
|11|核准金額|approved_amount|DECIMAL(18,2)|核准時|否|核准後否|approval|差異須走 adjustment|S3|—|
|12|幣別|currency_code|CODE|試算|否|否|rule/source|ISO 4217|S0|—|
|13|計算審核狀態|calculation_review_status|CODE|建立|否|狀態事件控制|state machine|CALCULATED/PENDING_REVIEW/RETURNED/APPROVED/CANCELLED|S1|目前摘要|
|14|撥款狀態摘要|payout_status_summary|CODE|建立|否|僅推導|payout ledger|既定 payout 狀態|S3|非權威|
|15|爭議狀態摘要|hold_dispute_status|CODE|建立|否|事件流程|audit/state event|NONE/ON_HOLD/DISPUTED/RESOLVED|S1|—|
|16|追回狀態摘要|recovery_status|CODE|建立|否|僅推導|adjustment/recovery|NONE/PENDING/PARTIAL/RECOVERED|S3|—|
|17|防重鍵|idempotency_key|TEXT|試算|是（有效結果範圍）|否|calculation|正規化＋安全摘要|S1|不可含可逆個資|
|18|計算版本|calculation_version|INTEGER|試算|否|否|calculation|範圍內遞增|S1|—|
|19|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|20|提交作業ID|operation_id|ID|建立|否|否|operation journal|須有 PREPARED；正式使用前須有有效 COMMITTED|S1|正式採用與來源列使用同一 operation|

### 5.6 `bonus_campaigns_活動獎金規則表`（既有，補強為版本表）

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|活動規則ID|bonus_campaign_version_id|TEXT|建立|是|否|campaign version|`BCV-...`|S0|沿用既有 Header；語意為活動規則版本 ID|
|2|活動ID|campaign_id|TEXT|建立|否|否|campaign family|穩定 ID|S0|同名不同期分開|
|3|活動期別|campaign_period_code|TEXT|草稿|否|草稿可改|campaign version|同活動內唯一|S0|—|
|4|活動名稱|campaign_name|TEXT|草稿|否|草稿可改|campaign version|非空|S0|顯示用|
|5|開始時間|starts_at|DATETIME_TZ|核准前|否|核准後否|campaign version|早於結束|S0|不得作程式預設|
|6|結束時間|ends_at|DATETIME_TZ|核准前|否|核准後否|campaign version|晚於開始|S0|—|
|7|時區|timezone|TEXT|核准前|否|核准後否|campaign version|IANA timezone|S0|—|
|8|計算週期|calculation_period_code|CODE|草稿|否|草稿可改|campaign version|既有週期碼|S0|—|
|9|是否每期歸零|resets_each_period|BOOLEAN|草稿|否|草稿可改|campaign version|TRUE/FALSE|S0|—|
|10|件數來源|unit_source_code|CODE|草稿|否|草稿可改|campaign version|核准碼表|S0|—|
|11|門檻級距設定|tier_parameters|JSON_CANONICAL|草稿|否|草稿可改|campaign version|schema 驗證、固定排序|S0|不在程式寫死|
|12|適用範圍設定|eligibility_parameters|JSON_CANONICAL|草稿|否|草稿可改|campaign version|角色／位階／方案 schema|S0|—|
|13|發放與擇優設定|award_selection_parameters|JSON_CANONICAL|草稿|否|草稿可改|campaign version|併領／擇優／優先 schema|S0|—|
|14|版本狀態|rule_version_status|CODE|建立|否|依狀態機|rule lifecycle|DRAFT/APPROVED/ACTIVE/DEACTIVATED|S0|—|
|15|前一版本ID|previous_version_id|ID|非首版|否|否|rule lifecycle|須存在同活動版本|S0|—|
|16|核准者安全識別|approved_by_actor_id|ID|核准時|否|否|audit|授權角色|S1|—|
|17|核准時間|approved_at|DATETIME_TZ|核准時|否|否|audit|有效時間|S1|—|

### 5.7 `consultant_status_events_顧問狀態事件`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|顧問狀態事件ID|consultant_status_event_id|TEXT|建立|是|否|status event|`CSE-...`|S1|追加式|
|2|顧問ID|consultant_id|ID|建立|否|否|consultant|須存在|S1|索引欄|
|3|事件類型|event_type_code|CODE|建立|否|否|status event|只用核准合作事件碼|S1|不自行新增制度語意|
|4|前狀態|previous_status_code|CODE|建立|否|否|prior event|首筆使用標準值|S1|—|
|5|新狀態|new_status_code|CODE|建立|否|否|status event|允許轉換|S1|—|
|6|生效時間|effective_at|DATETIME_TZ|建立|否|否|decision|有效時間|S1|與記錄時間分開|
|7|原因代碼|reason_code|CODE|人工事件|否|否|decision|核准原因碼表|S1|自由文字非唯一依據|
|8|決議稽核事件ID|decision_audit_event_id|ID|需決議時|否|否|audit|須存在|S1|—|
|9|操作者安全識別|actor_id|ID|建立|否|否|audit|授權角色|S1|不存姓名|
|10|覆核者安全識別|reviewer_id|ID|需覆核時|否|否|audit|不得空白或使用標準不適用值|S1|—|
|11|建立時間|recorded_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|永不覆寫|
|12|更正前事件ID|corrects_event_id|ID|更正時|否|否|status event|須存在同顧問事件|S1|以新事件更正|

### 5.8 `mentor_qualification_events_導師資格事件`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|導師資格事件ID|mentor_qualification_event_id|TEXT|建立|是|否|qualification event|`MQE-...`|S1|—|
|2|顧問ID|consultant_id|ID|建立|否|否|consultant|須存在|S1|—|
|3|事件類型|event_type_code|CODE|建立|否|否|qualification event|晉升／考核／暫停／喪失／恢復之核准碼|S1|不改 Task 04A-1 語意|
|4|考核期間|assessment_period|TEXT|季度事件|否|否|qualification event|標準季度鍵|S1|過渡季亦須明示|
|5|有效會員數|qualified_member_count|INTEGER|適用時|否|否|basis evidence|非負|S1|人數與件數分開|
|6|有效件數|qualified_units|DECIMAL(18,1)|適用時|否|否|basis evidence|0.5 精度、非負|S1|—|
|7|前資格狀態|previous_qualification_status|CODE|建立|否|否|prior event|首筆標準值|S1|—|
|8|新資格狀態|new_qualification_status|CODE|建立|否|否|decision|允許轉換|S1|—|
|9|生效時間|effective_at|DATETIME_TZ|建立|否|否|decision|有效時間|S1|—|
|10|依據安全引用|evidence_reference|TEXT|人工核准|否|否|controlled evidence|不可為公開 URL／憑證內容|S4|僅 token／受控 ID|
|11|核准者安全識別|approved_by_actor_id|ID|核准事件|否|否|audit|授權角色|S1|—|
|12|核准時間|approved_at|DATETIME_TZ|核准事件|否|否|audit|有效時間|S1|—|
|13|建立時間|recorded_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|14|更正前事件ID|corrects_event_id|ID|更正時|否|否|qualification event|須存在同顧問事件|S1|—|

### 5.9 `organization_relation_events_組織關係事件`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|組織關係事件ID|organization_relation_event_id|TEXT|建立|是|否|organization event|`ORE-...`|S1|—|
|2|顧問ID|consultant_id|ID|建立|否|否|consultant|須存在|S1|子節點|
|3|事件類型|event_type_code|CODE|建立|否|否|organization event|加入／移轉／分支獨立／承接等核准碼|S1|—|
|4|原上層顧問ID|previous_parent_id|ID|適用時|否|否|prior event|須存在或標準不適用值|S1|—|
|5|新上層顧問ID|new_parent_id|ID|適用時|否|否|decision|須存在、不得等於本人|S1|—|
|6|生效時間|effective_at|DATETIME_TZ|建立|否|否|decision|不可造成同時多個有效上層|S1|—|
|7|分支識別|branch_id|TEXT|分支事件|否|否|organization|穩定內部 ID|S1|不可用姓名|
|8|原因代碼|reason_code|CODE|人工事件|否|否|decision|核准碼表|S1|—|
|9|核准者安全識別|approved_by_actor_id|ID|核准時|否|否|audit|授權角色|S1|—|
|10|核准時間|approved_at|DATETIME_TZ|核准時|否|否|audit|有效時間|S1|—|
|11|建立時間|recorded_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|12|更正前事件ID|corrects_event_id|ID|更正時|否|否|organization event|須存在|S1|不得改舊列|

### 5.10 `members_會員主檔`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|會員ID|member_id|TEXT|建立|是|否|member|`MEM-...`|S2|不可重用|
|2|來源名單ID|source_lead_id|ID|適用時|否|否|lead|須存在|S2|可空|
|3|會員姓名|member_name|TEXT|建立|否|依法更正|member|非空；輸出遮罩／授權|S2|快照不複製完整值|
|4|手機號碼|phone|TEXT|需要時|否|依法更正|member|格式與遮罩|S2|不可作主鍵|
|5|電子郵件|email|TEXT|需要時|否|依法更正|member|格式與最小揭露|S2|—|
|6|會員狀態|member_status_code|CODE|建立|否|流程更新|member|核准會員狀態碼|S2|目前摘要|
|7|同意版本安全引用|consent_reference|TEXT|依法需要時|否|受控更新|consent system|受控 ID，不存文件內容|S4|—|
|8|遮罩顯示名稱|masked_display_name|TEXT|建立|否|系統|member|不可反推完整值|S2|顧問查詢用|
|9|遮罩手機|masked_phone|TEXT|建立|否|系統|member|核准遮罩規則|S2|顧問查詢用|
|10|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|11|最後更新時間|updated_at|DATETIME_TZ|更新|否|系統|system|不得早於建立|S1|—|

健康評估、健檢與報告不得進入本表或任何獎金快照。

### 5.11 `order_members_訂單會員關聯`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|訂單會員關聯ID|order_member_id|TEXT|建立|是|否|order member|`ORM-...`|S2|—|
|2|訂單ID|order_id|ID|建立|否|否|order|須存在|S1|索引|
|3|會員ID|member_id|ID|建立|否|否|member|須存在|S2|索引|
|4|成員角色|member_role_code|CODE|建立|否|成交前|order member|核准碼表|S1|例如主要聯絡／一般成員之語意須另核准|
|5|成員狀態|member_link_status|CODE|建立|否|成交前；後續事件化|order member|核准碼表|S1|不刪除歷史成員|
|6|加入時間|joined_at|DATETIME_TZ|建立|否|否|order member|有效時間|S1|—|
|7|退出時間|left_at|DATETIME_TZ|適用時|否|後續事件|order adjustment|晚於加入|S1|成交後不得直接補寫舊列|
|8|是否計入成交人數|counts_for_deal|BOOLEAN|成交前|否|成交後否|deal decision|TRUE/FALSE|S1|成交快照固化|
|9|成交快照ID|deal_snapshot_id|ID|成交後|否|否|deal snapshot|須屬同訂單|S1|不可回寫|
|10|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|

服務層另須保證同一訂單中的有效 `(order_id, member_id)` 不重複；退出或更正以新調整及快照版本表達。

### 5.12 `payments_付款入帳明細`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|付款ID|payment_id|TEXT|建立|是|否|payment|`PAY-...`|S3|—|
|2|訂單ID|order_id|ID|建立|否|否|order|須存在|S1|—|
|3|付款序號|payment_sequence|INTEGER|建立|組合唯一|否|payment|同訂單遞增、不可重用|S3|與 order ID 組合唯一|
|4|付款金額|payment_amount|DECIMAL(18,2)|建立|否|確認後否|payment|大於 0|S3|沖正另列|
|5|幣別|currency_code|CODE|建立|否|確認後否|payment|ISO 4217、與訂單一致|S0|—|
|6|付款方式代碼|payment_method_code|CODE|建立|否|確認後否|payment|核准碼表|S3|不存完整帳戶|
|7|外部交易安全引用|external_transaction_ref|TEXT|取得時|條件唯一|確認後否|payment provider|token／雜湊；不可為憑證內容|S4|外部 ID 不取代 payment ID|
|8|交易時間|transacted_at|DATETIME_TZ|建立|否|確認後否|payment source|有效時間|S3|—|
|9|入帳狀態|receipt_status_code|CODE|建立|否|事件流程|payment|核准收款狀態碼|S3|目前摘要；重大更正另事件|
|10|入帳確認時間|confirmed_at|DATETIME_TZ|確認時|否|否|finance confirmation|不得早於交易（合法補登另留原因）|S3|—|
|11|確認者安全識別|confirmed_by_actor_id|ID|確認時|否|否|finance|授權財務角色|S3|—|
|12|憑證安全引用|evidence_reference|TEXT|確認時依政策|否|否|controlled storage|不可為公開 URL 或內容|S4|建議隔離|
|13|沖正原付款ID|reverses_payment_id|ID|沖正時|否|否|payment|須存在且同訂單|S3|原付款不修改|
|14|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|

### 5.13 `order_adjustments_訂單退款調整`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|訂單調整ID|order_adjustment_id|TEXT|建立|是|否|order adjustment|`OAD-...`|S3|—|
|2|調整類型|adjustment_type_code|CODE|建立|否|否|order adjustment|退款／取消／降級／換方案等核准碼|S3|不新增制度結果|
|3|原訂單ID|order_id|ID|建立|否|否|order|須存在|S1|—|
|4|原付款ID|payment_id|ID|適用時|否|否|payment|須屬原訂單|S3|—|
|5|關聯新訂單ID|replacement_order_id|ID|換方案時|否|否|order|須存在且不得等於原訂單|S1|—|
|6|調整金額|adjustment_amount|DECIMAL(18,2)|金額調整時|否|否|decision|非負；方向由類型決定|S3|不直接覆寫訂單|
|7|調整前方案版本ID|before_plan_rule_version_id|ID|方案調整時|否|否|order/deal snapshot|須存在|S0|—|
|8|調整後方案版本ID|after_plan_rule_version_id|ID|方案調整時|否|否|decision|須存在|S0|—|
|9|調整前人數|before_member_count|INTEGER|人數調整時|否|否|snapshot|非負|S1|—|
|10|調整後人數|after_member_count|INTEGER|人數調整時|否|否|decision|非負|S1|—|
|11|原因代碼|reason_code|CODE|建立|否|否|decision|核准碼表|S3|—|
|12|狀態|adjustment_status_code|CODE|建立|否|事件流程|adjustment workflow|提出／審核／核准等流程碼|S3|正式碼待遷移設計核准|
|13|核准者安全識別|approved_by_actor_id|ID|核准時|否|否|audit|授權角色|S3|—|
|14|佐證安全引用|evidence_reference|TEXT|需佐證時|否|否|controlled storage|不存公開 URL／內容|S4|—|
|15|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|16|提交作業ID|operation_id|ID|建立|否|否|operation journal|未 COMMITTED 不得影響訂單或獎金|S1|同一調整鏈的重要列共用|

### 5.14 `bonus_rules_常態獎金規則表`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|常態獎金規則版本ID|bonus_rule_version_id|TEXT|建立|是|否|bonus rule version|`BRV-...`|S0|—|
|2|規則ID|bonus_rule_id|TEXT|建立|否|否|rule family|穩定 ID|S0|同規則多版本|
|3|獎金類型|bonus_type_code|CODE|草稿|否|草稿可改|Task 04A-1|既定類型|S0|—|
|4|計算週期|calculation_period_code|CODE|草稿|否|草稿可改|rule version|成交／週／月等核准碼|S0|—|
|5|資格條件設定|eligibility_parameters|JSON_CANONICAL|草稿|否|草稿可改|rule version|schema 驗證|S0|不寫死程式|
|6|計算參數|calculation_parameters|JSON_CANONICAL|草稿|否|草稿可改|rule version|schema 驗證|S0|含金額／門檻時由版本承載|
|7|組織與承接設定|organization_parameters|JSON_CANONICAL|適用時|否|草稿可改|rule version|schema 驗證|S0|保存既定承接語意|
|8|時區|timezone|TEXT|核准前|否|核准後否|rule version|IANA timezone|S0|—|
|9|版本狀態|rule_version_status|CODE|建立|否|依狀態機|rule lifecycle|DRAFT/APPROVED/ACTIVE/DEACTIVATED|S0|—|
|10|生效時間|effective_at|DATETIME_TZ|核准前|否|核准後否|rule lifecycle|同範圍不得重疊|S0|—|
|11|失效時間|expires_at|DATETIME_TZ|適用時|否|核准後否|rule lifecycle|晚於生效|S0|—|
|12|前一版本ID|previous_version_id|ID|非首版|否|否|rule lifecycle|須存在同規則版本|S0|—|
|13|核准者安全識別|approved_by_actor_id|ID|核准時|否|否|audit|授權角色|S1|—|
|14|核准時間|approved_at|DATETIME_TZ|核准時|否|否|audit|有效時間|S1|—|

### 5.15 `deal_snapshots_成交快照`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|成交快照ID|deal_snapshot_id|TEXT|成交建立|是|否|deal snapshot|`DLS-...`|S1|不可回寫|
|2|訂單ID|order_id|ID|成交建立|否|否|order|須存在|S1|—|
|3|快照版本|snapshot_version|INTEGER|成交建立|組合唯一|否|deal snapshot|同訂單遞增|S1|原版保留|
|4|取代前快照ID|supersedes_snapshot_id|ID|合法更正時|否|否|deal snapshot|須屬同訂單|S1|不代表刪除原版|
|5|方案規則版本ID|plan_rule_version_id|ID|成交建立|否|否|plan version|唯一適用且已核准|S0|—|
|6|常態規則版本ID清單|bonus_rule_version_ids|ID_LIST|成交建立|否|否|bonus rule versions|固定排序、均存在|S0|—|
|7|活動規則版本ID清單|campaign_version_ids|ID_LIST|適用時|否|否|campaign versions|固定排序、均適用|S0|—|
|8|實際會員人數|actual_member_count|INTEGER|成交建立|否|否|order_member|與有效成員一致|S1|—|
|9|有效付費會員數|effective_paid_member_count|INTEGER|成交建立|否|否|order_member/payment|非負、不大於實際人數|S1|—|
|10|確認成交金額|confirmed_deal_amount|DECIMAL(18,2)|成交建立|否|否|order/payment|應收與入帳證據平衡|S3|—|
|11|幣別|currency_code|CODE|成交建立|否|否|order|ISO 4217|S0|—|
|12|有效件數|effective_units|DECIMAL(18,1)|成交建立|否|否|plan version|0.5 精度|S1|—|
|13|付款ID清單|payment_ids|ID_LIST|成交建立|否|否|payment|固定排序、全部已確認|S3|只存 ID|
|14|訂單會員關聯ID清單|order_member_ids|ID_LIST|成交建立|否|否|order member|固定排序、須屬同訂單|S2|只存 ID|
|15|最終歸屬顧問ID|attributed_consultant_id|ID|成交建立|否|否|attribution decision|須存在 consultant|S1|—|
|16|推薦保護判定|protection_decision_code|CODE|成交建立|否|否|attribution decision|完整／人工決議／阻擋等核准碼|S1|異常未解不得送審|
|17|組織快照ID|organization_snapshot_id|ID|成交邏輯提交流程|一對一|否|organization snapshot|須存在且屬本快照|S1|operation COMMITTED 前不得產生可見正式獎金|
|18|成交確認時間|deal_confirmed_at|DATETIME_TZ|成交建立|否|否|three prerequisites|三要件最後完成點|S1|—|
|19|時區|timezone|TEXT|成交建立|否|否|system|Asia/Taipei|S0|明確保存|
|20|輸入指紋|input_fingerprint|TEXT|成交建立|否|否|system|正規化安全摘要|S1|不得含可逆個資|
|21|建立時間|created_at|DATETIME_TZ|成交建立|否|否|system|有效時間|S1|—|
|22|提交作業ID|operation_id|ID|成交建立|否|否|operation journal|須與 organization snapshot 共用；COMMITTED 後才可見|S1|取代跨分頁 transaction 假設|

### 5.16 `organization_snapshots_組織快照`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|組織快照ID|organization_snapshot_id|TEXT|成交建立|是|否|organization snapshot|`ORS-...`|S1|不可回寫|
|2|成交快照ID|deal_snapshot_id|ID|成交建立|一對一|否|deal snapshot|須存在|S1|—|
|3|成交顧問ID|deal_consultant_id|ID|成交建立|否|否|attribution|須存在 consultant|S1|—|
|4|組織路徑ID清單|ancestor_path_ids|ID_LIST|成交建立|否|否|organization events|固定順序、無循環、ID 存在|S1|不得存姓名|
|5|路徑資格事件ID清單|qualification_event_ids|ID_LIST|成交建立|否|否|qualification events|與路徑節點對齊|S1|時間點證據|
|6|最近有效導師ID|nearest_effective_mentor_id|ID|判定適用時|否|否|organization decision|須在有效路徑或標準不適用值|S1|—|
|7|跳過導師ID清單|skipped_mentor_ids|ID_LIST|適用時|否|否|organization decision|固定排序／路徑順序|S1|—|
|8|跳過原因清單|skip_reason_codes|JSON_CANONICAL|適用時|否|否|qualification evidence|每節點一原因碼|S1|—|
|9|指導獎受益候選顧問ID|guidance_candidate_id|ID|適用時|否|否|rule decision|須存在或標準不適用值|S1|候選不等同正式獎金|
|10|培育獎受益候選顧問ID|cultivation_candidate_id|ID|適用時|否|否|rule decision|須存在或標準不適用值|S1|—|
|11|培育期間判定|cultivation_period_decision|CODE|適用時|否|否|rule decision|核准碼表|S1|—|
|12|承接原因代碼|handover_reason_code|CODE|發生承接時|否|否|rule decision|核准碼表|S1|—|
|13|判定時間|evaluated_at|DATETIME_TZ|成交建立|否|否|system|不得晚於快照建立|S1|—|
|14|輸入指紋|input_fingerprint|TEXT|成交建立|否|否|system|正規化安全摘要|S1|—|
|15|建立時間|created_at|DATETIME_TZ|成交建立|否|否|system|有效時間|S1|—|
|16|提交作業ID|operation_id|ID|成交建立|否|否|operation journal|須與 deal snapshot 共用；COMMITTED 後才可見|S1|—|

### 5.17 `bonus_calculations_獎金計算快照`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|獎金計算快照ID|bonus_calculation_snapshot_id|TEXT|計算|是|否|calculation snapshot|`BCS-...`|S3|—|
|2|計算執行ID|calculation_run_id|ID|計算|否|否|calculation run|每次執行唯一|S1|retry/rerun 分開記錄|
|3|獎金類型|bonus_type_code|CODE|計算|否|否|rule version|期間彙總既定類型|S0|—|
|4|受益顧問ID|beneficiary_consultant_id|ID|計算|否|否|consultant|須存在|S1|—|
|5|認列期間|recognition_period|TEXT|計算|否|否|period rules|標準月／週鍵|S1|—|
|6|常態規則版本ID|bonus_rule_version_id|ID|適用時|否|否|bonus rule version|唯一適用|S0|—|
|7|活動規則版本ID|bonus_campaign_version_id|ID|適用時|否|否|campaign version|唯一適用或標準不適用值|S0|—|
|8|計算版本|calculation_version|INTEGER|計算|組合唯一|否|calculation|同業務範圍遞增|S1|—|
|9|冪等鍵|idempotency_key|TEXT|計算|是（相同輸入）|否|calculation|正規化安全摘要|S1|—|
|10|輸入指紋|input_fingerprint|TEXT|計算|否|否|calculation|不得含可逆個資|S1|—|
|11|總有效件數|total_effective_units|DECIMAL(18,1)|計算|否|否|basis items|0.5 精度、可重建|S1|—|
|12|適用門檻代碼|applied_tier_code|TEXT|計算|否|否|rule version|須存在版本參數|S0|不複製門檻為預設值|
|13|原始金額|gross_amount|DECIMAL(18,2)|計算|否|否|basis/rule|可重建|S3|—|
|14|扣除金額|deduction_amount|DECIMAL(18,2)|計算|否|否|basis items|非負、可重建|S3|—|
|15|擇優結果代碼|selection_result_code|TEXT|適用時|否|否|rule decision|須對應規則版本|S0|同額標示待決策|
|16|計算結果金額|calculated_amount|DECIMAL(18,2)|計算|否|否|calculation|公式可重建|S3|—|
|17|幣別|currency_code|CODE|計算|否|否|rule version|ISO 4217|S0|—|
|18|正式採用狀態|adoption_status_code|CODE|計算|否|事件流程|adoption event|未採用／已採用／已取代等技術碼|S1|正式碼於遷移設計確認|
|19|取代前計算快照ID|supersedes_calculation_snapshot_id|ID|重算時|否|否|calculation|同業務範圍|S3|—|
|20|建立時間|created_at|DATETIME_TZ|計算|否|否|system|有效時間|S1|—|
|21|提交作業ID|operation_id|ID|計算|否|否|operation journal|須與全部 basis items 共用；COMMITTED 後才可採用|S1|—|

### 5.18 `bonus_basis_items_獎金計算依據明細`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|獎金計算依據ID|bonus_basis_item_id|TEXT|計算|是|否|basis item|`BBI-...`|S3|—|
|2|獎金計算快照ID|bonus_calculation_snapshot_id|ID|計算|否|否|calculation snapshot|須存在|S3|—|
|3|來源實體類型|source_entity_type|CODE|計算|組合唯一|否|basis item|核准來源類型|S1|與來源 ID 組合防重|
|4|來源實體ID|source_entity_id|ID|計算|組合唯一|否|source entity|須存在且類型相符|S1|—|
|5|來源子項識別|source_subitem_key|TEXT|一來源多合法效果時|組合唯一|否|basis design|標準不適用值或可驗證子項|S1|不得繞過防重|
|6|效果類型|effect_type_code|CODE|計算|否|否|basis item|INCLUDE/DEDUCT/EXCLUDE|S1|不複製來源列表示多次影響|
|7|件數影響值|unit_effect|DECIMAL(18,1)|適用時|否|否|source/rule|0.5 精度、正負方向依效果|S1|—|
|8|金額影響值|amount_effect|DECIMAL(18,2)|適用時|否|否|source/rule|幣別一致|S3|—|
|9|納入判定|inclusion_decision_code|CODE|計算|否|否|calculation|核准碼表|S1|排除項也要保存|
|10|原因代碼|reason_code|CODE|扣除／排除時|否|否|calculation|核准原因碼表|S1|—|
|11|排序序號|sort_order|INTEGER|計算|否|否|calculation|快照內唯一或穩定排序|S1|可重現輸出|
|12|建立時間|created_at|DATETIME_TZ|計算|否|否|system|有效時間|S1|—|
|13|提交作業ID|operation_id|ID|計算|否|否|operation journal|須與 calculation snapshot 共用；COMMITTED 後才可見|S1|—|

唯一條件為 `(bonus_calculation_snapshot_id, source_entity_type, source_entity_id, source_subitem_key)`；一般情況 `source_subitem_key=NOT_APPLICABLE`。

### 5.19 `bonus_adjustments_獎金調整追回`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|獎金調整ID|bonus_adjustment_id|TEXT|建立|是|否|adjustment/recovery|`BAD-...`|S3|—|
|2|調整類型|adjustment_type_code|CODE|建立|否|否|adjustment/recovery|人工調整／取消差額／負項追回／扣抵等核准碼|S3|—|
|3|原獎金ID|original_bonus_record_id|ID|建立|否|否|bonus record|須存在|S3|—|
|4|來源訂單調整ID|order_adjustment_id|ID|退款相關時|否|否|order adjustment|須存在|S3|—|
|5|關聯前項調整ID|parent_adjustment_id|ID|後繼事件|否|否|adjustment/recovery|須存在同鏈|S3|—|
|6|調整金額|adjustment_amount|DECIMAL(18,2)|建立|否|否|decision|正負方向由類型決定|S3|不覆寫原獎金|
|7|應追回金額|recovery_due_amount|DECIMAL(18,2)|追回建立|否|否|approved decision|非負|S3|—|
|8|本次追回金額|recovered_amount|DECIMAL(18,2)|追回確認|否|否|finance result|不得超過當時未追回餘額|S3|一列一事件|
|9|未追回餘額摘要|remaining_recovery_amount|DECIMAL(18,2)|追回後|否|僅推導|recovery ledger|不得為負|S3|非唯一權威|
|10|追回狀態摘要|recovery_status|CODE|追回後|否|僅推導|recovery ledger|PENDING/PARTIAL/RECOVERED|S3|—|
|11|原因代碼|reason_code|CODE|建立|否|否|decision|核准碼表|S3|—|
|12|提出者安全識別|proposed_by_actor_id|ID|人工提出|否|否|audit|授權角色|S3|—|
|13|覆核者安全識別|reviewed_by_actor_id|ID|審核時|否|否|audit|授權角色|S3|—|
|14|核准者安全識別|approved_by_actor_id|ID|核准時|否|否|audit|授權角色|S3|—|
|15|佐證安全引用|evidence_reference|TEXT|需佐證時|否|否|controlled storage|不可為公開 URL／內容|S4|—|
|16|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|17|提交作業ID|operation_id|ID|建立|否|否|operation journal|未 COMMITTED 不得影響追回餘額或批次|S1|—|

### 5.20 `settlement_batches_結算批次`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|結算批次ID|settlement_batch_id|TEXT|建立|是|否|settlement batch|`STB-...`|S3|—|
|2|批次類型|batch_type_code|CODE|建立|否|建立階段|batch rules|週結基本類／月結類|S0|不自行新增類型|
|3|認列期間|recognition_period|TEXT|建立|否|建立階段|period rules|標準週／月鍵|S1|—|
|4|批次版本|batch_version|INTEGER|建立|組合唯一|否|settlement batch|同類型／期間遞增|S1|核准後變更新批次|
|5|批次序號|batch_sequence|INTEGER|多批時|組合唯一|否|batch policy|互斥範圍須核准|S1|是否允許多批待決策|
|6|時區|timezone|TEXT|建立|否|鎖定後否|period rules|Asia/Taipei|S0|—|
|7|批次狀態|batch_status_code|CODE|建立|否|依狀態機|batch lifecycle|既定批次狀態|S1|—|
|8|預定撥款日|scheduled_payout_date|DATE|試算|否|核准後否|calendar policy|營業日來源待決策|S3|不可由公式作唯一權威|
|9|總應付金額摘要|total_payable_amount|DECIMAL(18,2)|鎖定後|否|僅重建／版本化|batch items|須等於 item 合計|S3|—|
|10|總扣抵金額摘要|total_offset_amount|DECIMAL(18,2)|鎖定後|否|僅重建／版本化|batch items|須等於 item 合計|S3|—|
|11|總淨額摘要|total_net_amount|DECIMAL(18,2)|鎖定後|否|僅重建／版本化|batch items|應付減扣抵|S3|—|
|12|內容指紋|content_fingerprint|TEXT|候選鎖定|否|否|system|正規化安全摘要|S1|來源變更即阻擋|
|13|鎖擁有者執行ID|lock_owner_run_id|ID|鎖定中|否|鎖流程|system|唯一持有者|S1|含租約資訊可另由執行層管理|
|14|鎖定時間|locked_at|DATETIME_TZ|候選鎖定|否|否|system|有效時間|S1|—|
|15|核准者安全識別|approved_by_actor_id|ID|核准時|否|否|audit|授權角色|S3|—|
|16|核准時間|approved_at|DATETIME_TZ|核准時|否|否|audit|有效時間|S3|—|
|17|建立時間|created_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|
|18|提交作業ID|operation_id|ID|建立|否|否|operation journal|須與該版本全部 batch items 共用；COMMITTED 後才可送審／排款|S1|—|

### 5.21 `settlement_batch_items_結算批次明細`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|結算批次明細ID|settlement_batch_item_id|TEXT|候選鎖定|是|否|batch item|`SBI-...`|S3|—|
|2|結算批次ID|settlement_batch_id|ID|候選鎖定|否|否|settlement batch|須存在|S3|—|
|3|批次版本|batch_version|INTEGER|候選鎖定|否|否|settlement batch|與批次一致|S1|—|
|4|明細版本|item_version|INTEGER|候選鎖定|組合唯一|否|batch item|來源鏈內遞增|S1|—|
|5|來源類型|source_type_code|CODE|候選鎖定|組合唯一|否|bonus/adjustment|BONUS/ADJUSTMENT_RECOVERY|S3|—|
|6|來源ID|source_id|ID|候選鎖定|組合唯一|否|source entity|須存在、狀態已核准|S3|同 batch version＋納入類型唯一|
|7|原獎金ID|original_bonus_record_id|ID|候選鎖定|否|否|bonus record|須存在|S3|調整／追回也回連|
|8|受益顧問ID|beneficiary_consultant_id|ID|候選鎖定|否|否|source entity|須一致|S1|—|
|9|納入類型|inclusion_type_code|CODE|候選鎖定|組合唯一|否|batch item|正向應付／負向扣抵／追回／調整|S3|—|
|10|應付金額|payable_amount|DECIMAL(18,2)|候選鎖定|否|否|approved source|非負|S3|—|
|11|扣抵金額|offset_amount|DECIMAL(18,2)|候選鎖定|否|否|approved recovery|非負且不超過餘額|S3|—|
|12|淨額|net_amount|DECIMAL(18,2)|候選鎖定|否|否|batch calculation|應付減扣抵|S3|不可只存無來源淨額|
|13|幣別|currency_code|CODE|候選鎖定|否|否|source|ISO 4217|S0|—|
|14|認列期間|recognition_period|TEXT|候選鎖定|否|否|source|與來源一致|S1|—|
|15|計算／規則版本引用|version_references|JSON_CANONICAL|候選鎖定|否|否|source snapshots|固定 schema、ID only|S1|—|
|16|內容指紋|content_fingerprint|TEXT|候選鎖定|否|否|system|正規化安全摘要|S1|—|
|17|鎖定時間|locked_at|DATETIME_TZ|候選鎖定|否|否|system|有效時間|S1|財務核心自此不可變|
|18|提交作業ID|operation_id|ID|候選鎖定|否|否|operation journal|須與 settlement batch 共用；COMMITTED 後才可見|S1|—|

唯一條件為 `(settlement_batch_id, batch_version, source_type_code, source_id, inclusion_type_code)`。跨批次另檢查既有成功、處理中與結果不明金額；已成功正向金額不得再次納入。

### 5.22 `payout_events_撥款事件簿`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|撥款事件ID|payout_event_id|TEXT|建立|是|否|payout ledger|`PYE-...`|S3|全表 append-only|
|2|事件類型|event_type_code|CODE|建立|否|否|payout ledger|ATTEMPT/RESULT/CORRECTION_DECISION|S3|決定專屬必填欄|
|3|提交作業ID|operation_id|ID|建立|否|否|operation journal|須有 PREPARED；RESULT 影響摘要前須 COMMITTED|S1|attempt/result 作業可各有穩定 operation|
|4|結算批次明細ID|settlement_batch_item_id|ID|建立|否|否|batch item|須存在且已提交|S3|—|
|5|父撥款事件ID|parent_payout_event_id|ID|result／retry／更正|否|否|payout ledger|類型與父事件相容|S3|—|
|6|撥款嘗試識別|attempt_id|TEXT|ATTEMPT|條件唯一|否|payout attempt|穩定 ID|S3|便於外部對帳|
|7|嘗試冪等鍵|attempt_idempotency_key|TEXT|ATTEMPT|條件唯一|否|payout attempt|batch item＋範圍＋序號正規化|S3|不可含帳戶資料|
|8|結果冪等鍵|result_idempotency_key|TEXT|RESULT|條件唯一|否|payout result|attempt＋可重現外部結果識別或人工匯入批次／列鍵|S3|同一來源結果固定命中既有 RESULT|
|9|結果來源類型|result_source_type_code|CODE|RESULT|否|否|payout import|外部回傳／人工匯入等核准碼|S3|不保存敏感原始 payload|
|10|結果來源批次鍵|result_source_batch_key|TEXT|人工匯入 RESULT|組合唯一|否|payout import|可重現且不含姓名／帳號|S3|例如受控匯入批次安全 ID|
|11|結果來源列鍵|result_source_row_key|TEXT|人工匯入 RESULT|組合唯一|否|payout import|同來源批次內穩定唯一|S3|不得使用姓名或帳號|
|12|重試前嘗試事件ID|retry_of_event_id|ID|retry ATTEMPT|否|否|payout attempt|須指向已確認失敗 attempt|S3|未知或衝突結果不得重試|
|13|衝突結果事件ID|conflicting_result_event_id|ID|衝突 RESULT／決議|否|否|payout ledger|須指向同 attempt 既有 RESULT|S3|衝突後轉結果不明／爭議|
|14|關聯識別|correlation_id|TEXT|建立|否|否|workflow|同一處理鏈一致|S1|—|
|15|嘗試金額|attempt_amount|DECIMAL(18,2)|ATTEMPT|否|否|batch item balance|大於 0；不得超過可處理餘額|S3|—|
|16|結果狀態|result_status_code|CODE|RESULT|否|否|payout result|SUCCESS/FAILED/RESULT_UNKNOWN|S3|不新增顧問顯示狀態|
|17|成功金額|succeeded_amount|DECIMAL(18,2)|RESULT|否|否|payout result|非負；有效累計不得超過 attempt 或 item 待處理餘額|S3|—|
|18|失敗金額|failed_amount|DECIMAL(18,2)|RESULT|否|否|payout result|非負；未知可用標準不適用值|S3|—|
|19|失敗原因代碼|failure_reason_code|CODE|失敗 RESULT|否|否|payout result|核准碼表|S3|不記敏感文字|
|20|更正決議代碼|correction_decision_code|CODE|CORRECTION_DECISION|否|否|authorized decision|核准碼表|S3|重新推導，不改原 result|
|21|憑證安全引用|evidence_reference|TEXT|result／決議依政策|否|否|controlled storage|不可為公開 URL 或憑證內容|S4|建議隔離|
|22|財務操作者安全識別|finance_actor_id|ID|建立|否|否|audit|授權財務角色|S3|—|
|23|覆核者安全識別|reviewer_actor_id|ID|更正／決議|否|否|audit|授權角色|S3|—|
|24|事件時間|occurred_at|DATETIME_TZ|建立|否|否|payout source|有效時間|S3|—|
|25|記錄時間|recorded_at|DATETIME_TZ|建立|否|否|system|不早於事件（合法補登另留原因）|S1|—|
|26|結果內容指紋|result_content_fingerprint|TEXT|RESULT|否|否|payout result|對正規化狀態／金額／安全原因碼建立不可逆摘要|S3|同 key 不同指紋即衝突；不保存原始 payload|

### 5.23 `audit_events_稽核事件`

|序|Header|程式鍵|型別|必填時點|唯一|可修改|來源／權威|驗證／允許值|敏感|備註|
|---:|---|---|---|---|---|---|---|---|---|---|
|1|稽核事件ID|audit_event_id|TEXT|建立|是|否|audit event|`AUD-...`|S1|append-only|
|2|事件類型|event_type_code|CODE|建立|否|否|audit catalog|含 OPERATION_PREPARED/COMMITTED/ABORTED/RECOVERED 與 Task 04A-3B 事件目錄|S1|operation 類型是提交權威|
|3|提交作業ID|operation_id|ID|operation 事件及參與列事件|組合唯一|否|operation journal|穩定 `OPR-...`；不得重用|S1|同作業各事件共用|
|4|作業類型|operation_type_code|CODE|OPERATION_PREPARED|否|否|operation journal|成交快照／計算／採用／組批／調整追回／撥款結果等核准碼|S1|—|
|5|作業冪等鍵|operation_idempotency_key|TEXT|OPERATION_PREPARED|條件唯一|否|operation journal|相同業務請求必須解析到同一 operation|S1|不得含可逆個資|
|6|作業事件序號|operation_event_sequence|INTEGER|operation 事件|組合唯一|否|operation journal|同 operation 遞增|S1|與 operation ID 組合唯一|
|7|預期資料清單|expected_entity_manifest|JSON_CANONICAL|OPERATION_PREPARED|否|否|operation journal|實體類型、預期數量／穩定 ID 範圍；不含敏感值|S1|commit 完整性依據|
|8|完整性指紋|integrity_fingerprint|TEXT|COMMITTED／決議|否|否|operation journal|由已寫權威列 ID、內容指紋與檢查版本正規化產生|S1|摘要欄不納入|
|9|目標實體類型|target_entity_type|CODE|建立|否|否|audit event|operation 事件固定為 OPERATION|S1|—|
|10|目標實體ID|target_entity_id|ID|建立|否|否|target entity|operation 事件等於 operation_id|S1|—|
|11|父實體ID|parent_entity_id|ID|適用時|否|否|target entity|須存在|S1|—|
|12|關聯識別|correlation_id|TEXT|建立|否|否|workflow|同一較廣流程一致|S1|operation ID 不取代 correlation ID|
|13|執行識別|run_id|TEXT|系統流程|否|否|workflow|計算／批次／attempt run|S1|—|
|14|前狀態|before_status_code|CODE|狀態變更|否|否|target|核准碼或標準不適用值|S1|—|
|15|後狀態|after_status_code|CODE|狀態變更|否|否|target|允許轉換|S1|—|
|16|安全差異|safe_diff|JSON_CANONICAL|變更事件|否|否|audit|欄名、遮罩摘要或不可逆摘要|S1-S4|不得複製完整敏感值|
|17|操作者角色|actor_role_code|CODE|建立|否|否|authorization|核准角色碼|S1|—|
|18|操作者安全識別|actor_id|ID|建立|否|否|authorization|service／user 安全 ID|S1|—|
|19|覆核者角色|reviewer_role_code|CODE|需覆核時|否|否|authorization|核准角色碼|S1|—|
|20|覆核者安全識別|reviewer_id|ID|需覆核時|否|否|authorization|安全 ID|S1|—|
|21|原因代碼|reason_code|CODE|人工／阻擋／終局事件|否|否|audit|核准碼表|S1|ABORTED／RECOVERED 必填|
|22|佐證安全引用|evidence_reference|TEXT|需佐證時|否|否|controlled storage|不得為公開 URL／內容|S4|查閱佐證另留 audit|
|23|結果狀態|result_code|CODE|建立|否|否|audit|SUCCESS/FAILED/PARTIAL/BLOCKED/RESULT_UNKNOWN|S1|技術結果，不改業務狀態|
|24|安全錯誤代碼|safe_error_code|CODE|失敗／阻擋|否|否|system|不得含敏感值|S1|—|
|25|發生時間|occurred_at|DATETIME_TZ|建立|否|否|source|有效時間|S1|—|
|26|記錄時間|recorded_at|DATETIME_TZ|建立|否|否|system|有效時間|S1|—|

---

## 6. 必要資料能力覆蓋

| 必要能力 | 權威分頁與處理 |
|---|---|
| 正式會員主檔 | `members`；只存必要身分與遮罩欄，不混入健康資料 |
| 一筆訂單多位會員 | `orders` 1:N `order_members` N:1 `members`；有效組合防重 |
| 分次付款及入帳確認 | `payments` 一筆一收款／沖正；`orders` 只留可重建摘要；全額入帳才建快照 |
| 退款、取消及方案調整 | `order_adjustments` 追加式事件，連結原訂單、付款與替代訂單 |
| 顧問狀態事件 | `consultant_status_events`；`consultants` 只留目前摘要 |
| 導師資格事件 | `mentor_qualification_events`；保存考核期間、依據與生效點 |
| 組織關係歷史 | `organization_relation_events`；不得只靠目前上層或路徑 |
| 成交及組織快照 | `deal_snapshots`＋`organization_snapshots` 使用同一 operation；邏輯提交完成後才可見，且不可回寫 |
| 方案規則版本 | `plans` 一列一版本；核准／引用後鎖定 |
| 常態獎金規則版本 | `bonus_rules` 一列一版本；金額與條件由版本參數承載 |
| 活動獎金版本 | `bonus_campaigns` 一列一活動期別版本，含時區、級距、適用範圍與擇優設定 |
| 獎金明細 | `bonus_records` 一獎項一受益人，明確區分成交型與期間型來源 |
| 月結計算快照及 basis items | `bonus_calculations`＋`bonus_basis_items`；每筆納入、扣除及排除均可追溯 |
| 人工調整與追回 | `bonus_adjustments`；原試算、核准與已發放事實不改寫 |
| 結算批次及不可變批次明細 | `settlement_batches`＋`settlement_batch_items`；鎖定後財務核心不可變 |
| payout attempt、result 及更正事件 | `payout_events` typed append-only ledger；attempt 與 result 分別有穩定冪等鍵，原事件永不覆寫 |
| 稽核與跨分頁提交 | `audit_events`；保存安全差異、角色、時間、原因、受控佐證，以及 PREPARED／COMMITTED／ABORTED／RECOVERED operation journal |

---

## 7. 既有六個分頁處理矩陣

本節依 Task 04A-2 已安全揭露的 Header 與 Task 04A-3A 銜接原則分類。未在上游文件完整列出的現存 Header，一律歸入「尚待正式 Header 再次核對」，不可因本文件而自動刪除、改名或搬移。

### 7.1 `consultants_顧問主檔`

| 分類 | 現有欄位／欄位群 | 後續定位 |
|---|---|---|
| 保留為權威欄位 | 顧問ID、顧問姓名及必要基本身分欄 | 顧問 ID 與身分主檔；個資受控更正 |
| 保留但改為衍生摘要 | 目前位階、顧問狀態、合約狀態、是否具導師資格、導師資格狀態、推薦人ID、上層導師ID、組織路徑、組織層級、直推會員成交件數、團隊會員成交件數、本季直推會員數、連續未達標季數、直接下線／團隊總顧問數 | 由狀態、資格、組織事件與成交快照重建；不得作歷史唯一依據 |
| 後續停用但暫不刪除 | 重複的「是否可領獎金」及各類「是否可領／可領…」旗標、以姓名表示的推薦人／上層導師欄 | 相容顯示；不得控制正式計獎 |
| 需搬移至新分頁 | 狀態生效歷史、資格考核歷史、組織移轉／分支獨立歷史；任何銀行或撥款敏感欄位 | 分別進三種事件表；金融欄待隔離決策後遷移 |
| 需新增版本或快照引用 | 最新狀態事件ID、最新資格事件ID、最新組織事件ID | 只作目前摘要指標 |
| 尚待正式 Header 再次核對 | 未由 Task 04A-2 完整列出的聯絡、合約、權限、培育期間與金融欄位之精確 Header、重複語意及使用中狀況 | 遷移設計前逐欄核對；本輪不診斷正式 Sheet |

### 7.2 `leads_潛在會員名單`

| 分類 | 現有欄位／欄位群 | 後續定位 |
|---|---|---|
| 保留為權威欄位 | 名單ID、推薦碼來源、首次歸屬、推薦保護起訖、CRM 階段、來源頁面／管道、重複資料與合併關聯 | CRM 與推薦來源證據；HC9001 修正前不得自動當成完整歸屬事實 |
| 保留但改為衍生摘要 | 目前歸屬顧問、是否保護期內、轉換會員ID、轉換訂單ID | 供 CRM 顯示；成交以 deal snapshot 為準 |
| 後續停用但暫不刪除 | 是否已付款、成交金額、是否列入顧問獎金、獎金歸屬顧問等財務／獎金旗標 | 不得作付款、成交或獎金主帳 |
| 需搬移至新分頁 | 正式會員身分、每筆付款、最終成交歸屬決議、人工改派／爭議決議 | 分別進 members、payments、deal snapshot 與 audit/event |
| 需新增版本或快照引用 | 最近轉換訂單ID可保留摘要；正式規則版本只在訂單／快照引用 | lead 不承擔規則版本主責 |
| 尚待正式 Header 再次核對 | CRM 舊／新版狀態欄、保護欄、聯絡與健康相關欄之精確 Header、是否仍被前端相容讀取 | 健康資料不得搬進獎金域 |

### 7.3 `plans_方案規則表`

| 分類 | 現有欄位／欄位群 | 後續定位 |
|---|---|---|
| 保留為權威欄位 | 方案ID、方案名稱、方案類型、卡別、人數基準、幣別、各件數用途旗標 | 方案族群與規則版本基礎；需經正式版本治理才成權威 |
| 保留但改為衍生摘要 | 方案金額、推廣獎金、件數換算、是否目前版本 | 可顯示核准版本計算結果；不可取代版本參數與選版 |
| 後續停用但暫不刪除 | 無法表達完整動態公式的固定列或自由文字換算說明（若存在） | 相容參考；不得由程式解析為正式規則 |
| 需搬移至新分頁 | 無；方案版本仍由本分頁承載 | 若未來公式子表正規化，另案決定 |
| 需新增版本或快照引用 | 方案規則版本ID、狀態、生失效時間、前一版本、核准者／時間、公式 schema／參數 | 訂單與成交快照必須引用實際版本 ID |
| 尚待正式 Header 再次核對 | 現有 31 欄完整 Header、0.5 精度、客製金額及版本欄實際資料型別與下拉覆蓋 | 遷移前核對，不假設現況資料有效 |

### 7.4 `orders_訂單主檔`

| 分類 | 現有欄位／欄位群 | 後續定位 |
|---|---|---|
| 保留為權威欄位 | 訂單ID、來源 lead／會員關聯、會員合約ID、訂單建立事實、應收與幣別的核准值 | 訂單核心；成交後核心欄位鎖定 |
| 保留但改為衍生摘要 | 方案名稱／類型、實際人數、單一會員ID、付款狀態、付款日期、入帳確認日、推廣獎金、件數換算、是否退款、退款金額、是否已產生獎金、追回金額、結算月份／批次、預計撥款日、撥款狀態 | 分別由版本、order members、payments、adjustments、bonus、batch 及 payout ledger 重建 |
| 後續停用但暫不刪除 | 單一客戶姓名作多會員代表、產生獎金布林旗標作防重、單一批次或撥款文字作完整歷史 | 僅相容顯示，不得作控制 |
| 需搬移至新分頁 | 訂單成員、每筆付款、退款／方案調整、最終歸屬、成交與組織快照 | 正規化分頁承載 |
| 需新增版本或快照引用 | 方案規則版本ID、成交快照ID、成交時間摘要、調整狀態摘要 | 已成交列不得回填目前規則 |
| 尚待正式 Header 再次核對 | 現有 64 欄完整 Header、契約三要件、付款／入帳、退款、顧問角色與批次欄的實際語意 | 正式遷移 mapping 前逐欄核對 |

### 7.5 `bonus_records_獎金明細表`

| 分類 | 現有欄位／欄位群 | 後續定位 |
|---|---|---|
| 保留為權威欄位 | 獎金ID、獎金類型、受益顧問、來源訂單／活動／擇優骨架、原始試算、扣除、調整、件數換算 | 補齊來源快照與版本後作正式獎金明細 |
| 保留但改為衍生摘要 | 實際發放、審核／追回／撥款狀態、結算月份／批次、撥款批次 | 由 approval、adjustment、batch item、payout ledger 推導 |
| 後續停用但暫不刪除 | 舊狀態文字（已確認、待撥款、已撥款、暫停、取消、追回）與單一追回摘要；任何以布林或文字代表完整歷史的欄 | 遷移後只作相容讀取；不可直接字串批改 |
| 需搬移至新分頁 | 月結計算快照與 basis items、人工調整／追回事件、批次明細、撥款帳戶資訊、attempt/result、狀態稽核 | 分拆到對應表；銀行資料待隔離決策 |
| 需新增版本或快照引用 | 來源類型／快照ID、常態／活動規則版本、認列期間、防重鍵、計算版本、多維狀態摘要 | 成交型與期間型來源不可混淆 |
| 尚待正式 Header 再次核對 | 現有 64 欄完整 Header、審核者／日期、撥款帳戶後段資訊、退款／追回欄及下拉實際覆蓋 | 搬移前需最小揭露與財務確認 |

### 7.6 `bonus_campaigns_活動獎金規則表`

| 分類 | 現有欄位／欄位群 | 後續定位 |
|---|---|---|
| 保留為權威欄位 | 活動規則ID、活動ID、名稱、開始／結束日、週期、是否每期歸零、件數來源、門檻、擇優群組、前一版規則ID | 補齊核准與版本不可變後作活動版本權威 |
| 保留但改為衍生摘要 | 是否目前版本、活動狀態、顯示用門檻或實發說明 | 由版本狀態與參數推導 |
| 後續停用但暫不刪除 | 無法機器驗證的自由文字條件、程式依賴的硬編碼期間／金額對應欄（若存在） | 不得作自動選版權威 |
| 需搬移至新分頁 | 無；活動版本仍由本分頁承載 | 月結結果與 basis 不放在規則表 |
| 需新增版本或快照引用 | 活動規則版本ID、活動期別、時區、適用角色／位階／方案、級距 schema、併領／擇優 schema、核准者／時間 | calculation snapshot 引用實際版本 |
| 尚待正式 Header 再次核對 | 現有 35 欄完整 Header、已遮蔽下拉值、適用範圍與組織獎扣除欄實際語意 | 不以本輪推測取代正式核對 |

---

## 8. 主鍵、關聯與防重約束

### 8.1 主鍵格式與不可重用

| 分頁群 | ID prefix 示意 |
|---|---|
| consultants / leads / members / orders | `CNS-` / `LED-` / `MEM-` / `ORD-` |
| consultant／mentor／organization events | `CSE-` / `MQE-` / `ORE-` |
| order members / payments / order adjustments | `ORM-` / `PAY-` / `OAD-` |
| plan / bonus rule / campaign versions | `PRV-` / `BRV-` / `BCV-` |
| deal / organization snapshots | `DLS-` / `ORS-` |
| bonus calculation / basis / record / adjustment | `BCS-` / `BBI-` / `BON-` / `BAD-` |
| settlement batch / item / payout event / audit / operation | `STB-` / `SBI-` / `PYE-` / `AUD-` / `OPR-` |

- prefix 僅作人類辨識；正式 ID 產生演算法、長度與碰撞策略留待實作設計。
- ID 一經配發即不可重用，即使資料被取消、停用、匿名化或封存。
- 姓名、手機、LINE UID、推薦碼、銀行／金流交易號或其他外部 ID 不得取代內部主鍵。外部 ID 只能作可更換、可重複檢查的關聯欄。
- 所有外鍵在寫入前由未來 repository/service 層確認目標存在、類型相符、狀態允許且未封存；提交後再以稽核或一致性工作檢查孤兒列。

### 8.2 業務唯一與冪等條件

**成交型獎金：**

```text
deal_snapshot_id
+ bonus_type_code
+ beneficiary_consultant_id
+ applicable_rule_version_id
+ recognition_period_or_NOT_APPLICABLE
```

相同組合只能有一個有效正式結果。合法重算建立 calculation version 或 adjustment 關聯，不得建立無關聯同義獎金。

**期間彙總型獎金：**

```text
bonus_type_code
+ beneficiary_consultant_id
+ recognition_period
+ bonus_rule_version_id
+ campaign_period_or_version_or_NOT_APPLICABLE
```

同一業務範圍可以保留多個計算版本，但同一時間只能有一個正式採用 calculation snapshot 與一個有效正式 bonus result。

**Basis item：**

```text
bonus_calculation_snapshot_id
+ source_entity_type
+ source_entity_id
+ source_subitem_key
```

一般來源的 `source_subitem_key` 固定為 `NOT_APPLICABLE`；只有規格先定義可驗證子項時才可分拆。效果以 INCLUDE／DEDUCT／EXCLUDE 表達，不複製來源繞過唯一條件。

**Batch item：**

```text
settlement_batch_id
+ batch_version
+ source_type_code
+ source_id
+ inclusion_type_code
```

另須跨批次檢查同一來源的已成功、處理中、未知與尚未處理餘額。成功正向金額不可再次排款；合法差額必須使用 adjustment/recovery ID。

**Payout attempt：**

`attempt_idempotency_key` 應由 batch item ID、嘗試範圍／序號、嘗試金額與處理版本正規化後產生。相同請求重試命中同一 attempt；已確認失敗的餘額才能建立新 retry attempt；結果不明在對帳或人工決議前禁止重試。

**Payout result：**

`result_idempotency_key` 應由 `attempt_id`、結果來源類型，以及可重現的外部結果安全識別或人工匯入批次／列鍵產生；結果內容另存不可逆 `result_content_fingerprint`。同一外部／人工結果重複匯入時必須命中既有 RESULT，不新增第二筆有效結果。相同 key 卻有不同內容指紋即為衝突。人工匯入批次鍵與列鍵不得使用姓名、銀行帳號或其他個資。

同一 attempt 若收到不同 `result_idempotency_key` 且狀態、成功金額、失敗金額或來源識別互相衝突，服務層須：

1. 保留全部原始 RESULT 事件，但不把衝突成功額重複納入。
2. 追加安全 audit／RESULT_UNKNOWN 爭議事件，凍結該 attempt 與 batch item 的重試及完成推導。
3. 只接受授權者追加 `CORRECTION_DECISION` 指定哪些 result 有效或仍待確認；不得覆寫或刪除原 RESULT。
4. 驗證單一 attempt 的有效成功累計不超過 `attempt_amount`，且同一 batch item 的有效成功累計不超過其尚待處理餘額；超過即阻擋提交並進 reconciliation。

**追回上限：**

```text
本次可追回上限 = 已核准應追回金額 - 累計有效已追回金額 - 其他進行中鎖定扣抵金額
```

新追回或扣抵 item 必須大於零且不超過上限；同一餘額同時只能被一個批次鎖定。任何超扣、負餘額或幣別不一致均阻擋。

### 8.3 Sheet 無法原生強制時的責任層

| 約束 | 未來第一責任層 | 補強控制 |
|---|---|---|
| 主鍵／業務唯一 | repository/service 寫入層 | 有限範圍索引、鎖內重查、定期一致性 audit |
| 外鍵存在與狀態 | domain service | 批次前完整性檢查、孤兒列報告 |
| 規則版本唯一適用 | rule selection service | 重疊即阻擋與人工決議 |
| 跨分頁提交可見性 | operation coordinator＋正式 read repository | audit operation journal、commit predicate、reconciliation |
| 不可變欄位 | append-only repository＋欄位 allowlist | 保護分頁僅防誤操作；audit 指紋偵測 |
| 金額平衡／追回上限／result 防重 | settlement/payout service | LockService、attempt/result 冪等鍵、批次指紋、對帳事件 |
| 權限與遮罩 | API/service authorization | 可信任身分、欄位白名單、敏感查詢 audit |

本輪不實作以上控制。

### 8.4 跨分頁邏輯提交協定

#### 8.4.1 事件與狀態語意

| 事件 | 語意 | 正式可見性 |
|---|---|---|
| `OPERATION_PREPARED` | operation 已取得穩定 ID、冪等鍵、預期資料 manifest 與 correlation；可以開始追加權威列 | 不可見 |
| `OPERATION_COMMITTED` | manifest 中全部必要列存在、operation_id 一致、外鍵／唯一／金額／指紋檢查成功 | 可見；唯一可令權威列進入正式流程的事件 |
| `OPERATION_ABORTED` | 作業已確認失敗或不應採用；既有列保留供稽核 | 不可見且不得再續寫正式結果 |
| `OPERATION_RECOVERED` | reconciliation 已接手並保存復原檢查與決議過程 | 本身不可見；後續仍須追加 COMMITTED 或 ABORTED |

合法路徑為 `PREPARED → COMMITTED`、`PREPARED → ABORTED`，或 `PREPARED → RECOVERED → COMMITTED/ABORTED`。一旦 COMMITTED，後續業務更正必須建立新 operation 與反向／調整／取代列，不得對原 operation 追加 ABORTED 來抹除已提交事實。若同一 operation 出現互斥終局事件或不一致指紋，正式讀取須隔離該 operation 並轉人工 reconciliation。

#### 8.4.2 寫入協定

1. 依業務防重範圍解析 `operation_idempotency_key`。若已存在 operation，沿用其 `operation_id`；否則建立不可重用的穩定 `operation_id`。另建立或沿用較廣流程的 `correlation_id`。
2. 在 `audit_events` 先追加 `OPERATION_PREPARED`，保存 operation 類型、冪等鍵、預期資料 manifest、執行 ID、角色與時間。
3. 各參與分頁只追加或建立具有同一 `operation_id` 的權威列；建立前先查相同主鍵、業務唯一鍵及 operation 既有列，避免重試複製。
4. 作業未完成時，這些列即使實體存在，也不得被正式計算、送審、核准、組批、撥款摘要或顧問查詢使用。
5. coordinator 依 manifest 驗證必要列數、ID 清單、operation_id、外鍵、業務唯一鍵、內容指紋、金額與狀態一致性；摘要欄不納入 commit 判定。
6. 所有檢查成功後，最後追加 `OPERATION_COMMITTED`，保存完整性指紋。這是邏輯提交點，不代表 Sheet 具備實體 transaction。
7. 可預期失敗時追加 `OPERATION_ABORTED`。若程式在追加 ABORTED 前中斷，PREPARED 或帶 operation_id 的孤立列保留且不可見，等待 reconciliation。
8. reconciliation 不覆寫原列：追加 `OPERATION_RECOVERED` 記錄接手、檢查與安全差異，再依結果追加 COMMITTED 或 ABORTED。缺列可使用同一 operation 與預先分配 ID 補齊；不得建立第二套正式資料。
9. 相同 idempotency key 的 retry/rerun 必須命中同一 operation，先讀既有 manifest 與列，再只補安全缺項或回傳既有終局結果；不得配發新的正式 operation。
10. commit 後才非同步重建目前摘要。摘要更新失敗只建立重試／audit，不改 COMMITTED 權威事件，也不使未提交列可見。

#### 8.4.3 正式讀取條件

正式 repository 對參與 operation 的權威列一律套用下列條件；管理畫面、計算、審核、批次、顧問 API 不得自行繞過：

```text
row.operation_id is not empty
AND exactly one valid OPERATION_COMMITTED exists for row.operation_id
AND committed integrity_fingerprint matches the operation manifest
AND no terminal-event conflict or unresolved reconciliation flag exists
```

PREPARED、RECOVERED 尚未終局、ABORTED、缺 operation event、指紋不符或終局衝突的列，只能由受權稽核／reconciliation 查詢。非跨分頁歷史資料在遷移完成前不得以空 `operation_id` 自動視為 committed；須由遷移 operation 明確採用或列入相容讀取白名單，白名單策略列為待決策。

#### 8.4.4 適用作業

| 作業 | 同一 operation 的必要權威列 | COMMITTED 前禁止 |
|---|---|---|
| 全額入帳後成交 | deal snapshot、organization snapshot、必要關聯／指紋 | 計獎、送審、顧問顯示 |
| 期間計算 | calculation snapshot、完整 basis items | 正式採用、產生期間 bonus record |
| Bonus 正式採用 | adopted calculation 關係、bonus record、必要 audit | 審核、核准、組批 |
| Settlement 組批 | settlement batch version、全部 batch items | 送審、核准、排款 |
| Refund／adjustment／recovery | order adjustment、bonus adjustment/recovery、必要餘額／關聯 | 重算、取消、追回扣抵、顧問狀態變更 |
| Payout result | RESULT、result 防重／衝突檢查、operation commit；摘要於 commit 後重建 | 成功累計、已發放推導、retry |

單純摘要重建不建立新的業務權威 operation；它只讀已 committed 權威列並以獨立冪等工作更新摘要。

---

## 9. 狀態與下拉值

### 9.1 顯示中文與內部程式碼分離

Sheet 權威欄保存穩定內部 code；管理端與顧問大廳透過集中字典顯示中文。中文顯示名稱不得反向當作狀態機鍵，亦不得由前端自由傳入。狀態字典版本變更須留 audit event。

### 9.2 規則版本狀態

| 內部碼 | 管理端中文 | 可修改規則內容 | 可被新交易選用 |
|---|---|---:|---:|
| `DRAFT` | 草稿 | 是，限授權規則管理者 | 否 |
| `APPROVED` | 已核准 | 否 | 依生效時間 |
| `ACTIVE` | 生效中 | 否 | 是 |
| `DEACTIVATED` | 已停用 | 否 | 否；歷史引用保留 |

### 9.3 獎金內部多維狀態

| 維度 | 內部碼 | 中文語意 |
|---|---|---|
| 計算／審核 | `CALCULATED`、`PENDING_REVIEW`、`RETURNED`、`APPROVED`、`CANCELLED` | 已試算、待審核、已退回、已核准、已取消 |
| 撥款摘要 | `NOT_SCHEDULED`、`QUEUED`、`PROCESSING`、`PARTIALLY_SUCCEEDED`、`FAILED`、`RESULT_UNKNOWN`、`PAID_CONFIRMED` | 未排款、已排款、處理中、部分成功、失敗、結果不明、已確認發放 |
| 暫停／爭議 | `NONE`、`ON_HOLD`、`DISPUTED`、`RESOLVED` | 無、暫停、爭議中、已決議 |
| 追回 | `NONE`、`PENDING`、`PARTIAL`、`RECOVERED` | 無、待追回、部分追回、已追回 |

撥款摘要由不可變 batch item 與 payout ledger 推導，不是撥款歷史權威。

### 9.4 顧問可見狀態

顧問可見名稱維持 Task 04A-3B 已定義的：`試算`、`待審核`、`已核准`、`待發放`、`已發放`、`暫停／爭議`、`已取消`、`待追回`、`部分追回`、`已追回`。映射優先序為已追回、部分追回、待追回、有效暫停／爭議、已取消、已發放、待發放、已核准、待審核、試算；不得新增「部分發放」等營運意義不同的顧問主狀態。部分成功只在明細顯示成功累計與待處理餘額，主狀態維持待發放。

### 9.5 批次狀態

| 內部碼 | 中文顯示／語意 |
|---|---|
| `CREATED` | 建立 |
| `SIMULATED` | 已試算 |
| `CANDIDATE_LOCKED` | 候選已鎖定 |
| `PENDING_REVIEW` | 待審核 |
| `APPROVED` | 已核准 |
| `QUEUED` | 排款／待發放 |
| `PROCESSING` | 撥款處理中 |
| `COMPLETED` | 完成 |
| `PARTIALLY_FAILED` | 部分失敗／待處理 |
| `CANCELLED` | 已取消 |

### 9.6 Payout result 狀態

| 內部碼 | 中文語意 | 後續控制 |
|---|---|---|
| `SUCCESS` | 成功 | 成功金額不可再送 |
| `FAILED` | 失敗 | 查明後只對失敗且尚未成功餘額建立新 attempt |
| `RESULT_UNKNOWN` | 結果不明 | 對帳或追加決議前禁止重試、禁止顯示已發放 |

錯誤 result 不修改原列；以 `CORRECTION_DECISION` 事件保存正確判定或仍待確認結論。

### 9.7 批次與撥款結果控制

1. batch item 只有在其 settlement operation 已 COMMITTED 後才能送審、核准或建立 ATTEMPT。
2. 建立 RESULT 前，在同一 operation／鎖範圍內依 `result_idempotency_key`、`result_content_fingerprint`、人工匯入批次／列鍵及 attempt 既有結果重查。key 與指紋皆相同的重複回傳直接回傳既有 RESULT ID，不追加第二筆有效結果；key 相同但指紋不同立即視為衝突。
3. 外部回傳的穩定識別須保存為不可逆安全摘要或受控 token；人工匯入須有可重現的匯入批次鍵及列鍵。不得以姓名、帳號、列位置的臨時顯示值或上傳時間單獨作防重鍵。
4. 同一 attempt 的新結果若與既有有效 RESULT 衝突，該 operation 不得 COMMITTED；系統追加 RESULT_UNKNOWN／爭議 audit，凍結 retry、PAID_CONFIRMED 與批次完成推導。
5. 衝突只能由授權財務提出、覆核者追加 `CORRECTION_DECISION` 決議；原 RESULT 全數保留。決議 operation COMMITTED 後才重新推導有效結果集合與餘額。
6. 每次 commit 前同時驗證：單一 RESULT 金額平衡、attempt 有效成功累計不超過 attempt 金額、batch item 有效成功累計不超過尚待處理餘額、成功部分未被其他 attempt 重送。
7. `RESULT_UNKNOWN` 或衝突尚未完成 reconciliation 時禁止重送；未知不能當失敗，亦不能先顯示已發放。
8. payout result operation COMMITTED 後，才可非同步重建 bonus、batch item 與 batch 的撥款摘要。摘要更新失敗不改 payout ledger，後續以相同摘要工作冪等重建。
9. 一般執行記錄、RESULT、audit 與 operation manifest 均不得保存完整銀行帳號、公開憑證 URL 或敏感外部回傳內容；只留核准代碼、安全摘要及受控引用。

---

## 10. 不可變與可修改欄位

| 類別 | 適用資料 | 規則 |
|---|---|---|
| 建立後永不可覆寫 | 所有 ID、事件時間與原事件內容、deal snapshot、organization snapshot、calculation snapshot、basis item、batch item 鎖定財務核心、payout attempt/result/correction、audit event | 更正只能新增後繼／反向／決議事件；原列保留 |
| 核准後鎖定 | plan／bonus／campaign rule version、bonus record 核准值、settlement batch version、人工 adjustment/recovery | 草稿可編輯；核准後建立新版本或新調整，不回改 |
| 只能新增事件更正 | 顧問狀態、導師資格、組織關係、已確認 payment、order adjustment、追回、payout result | 以 `corrects_*`、parent 或 supersedes 關聯原事件 |
| 可作目前摘要更新 | consultants 目前狀態／資格／組織、orders 付款／成交／調整摘要、bonus_records 多維狀態摘要、batch totals、recovery balance | 必須可由權威事件或 ledger 重建；不反向覆蓋權威資料 |
| 可由公式顯示但不可作主帳 | 人數／件數／金額校驗、累計入帳、未收餘額、獎金合計、批次總額、成功累計與待處理餘額 | 公式錯誤或被修改不得改變正式財務事實 |
| 提交後才可正式讀取 | 所有帶 `operation_id` 的跨分頁權威列 | PREPARED／RECOVERED 未終局／ABORTED／孤立列永久保持不可見；只有有效 COMMITTED operation 可進正式流程 |

### 10.1 特別控制

- **deal snapshot**：方案、人數、付款 ID、最終歸屬、保護判定、規則版本與成交時間建立後不可改；合法更正建立新 snapshot version 與 `supersedes_snapshot_id`。
- **organization snapshot**：路徑、資格事件、最近有效導師、跳過原因、指導／培育候選不可因日後組織異動回寫。
- **calculation snapshot**：每次計算版本與 input fingerprint 不可改；正式採用是另行事件／狀態關係，不刪除未採用或被取代版本。
- **basis item**：納入、扣除、排除及理由均不可改；同來源不得重複影響同一 snapshot。
- **bonus record 原始試算**：`original_calculated_amount`、來源快照、規則版本、防重鍵與 calculation version 永不覆寫；核准差異、退款或人工調整另建紀錄。
- **settlement batch item**：候選鎖定後來源、受益人、應付、扣抵、淨額、幣別及版本指紋不可改；核准後更正使用新批次／反向或調整 item。
- **payout attempt/result**：attempt command 與 result 分列追加；retry 建新 attempt；錯誤 result 建 correction/decision；原事件永存。
- **operation journal event**：PREPARED、COMMITTED、ABORTED、RECOVERED 全部 append-only。COMMITTED 後不得以覆寫或追加 ABORTED 抹除；後續修正建立新 operation。未提交孤立列由 reconciliation 決議。
- **audit event**：append-only，不能記錄密碼、憑證內容或不必要個資；因 audit_events 同時承載提交 journal，若 PREPARED／COMMITTED 寫入失敗，尚未提交的業務列保持不可見並失敗關閉。

---

## 11. 敏感資料與工作簿隔離

### 11.1 可留在現有工作簿的資料

- S0 制度設定與版本：方案、常態規則、活動規則。
- S1 內部營運 ID、狀態、資格、組織事件與不含姓名的快照證據。
- 為訂單與獎金必要的最小 S2 關聯；顧問查詢只用遮罩欄。
- 不含完整帳戶或憑證內容的 S3 獎金、批次、金額與結果摘要，但仍只限管理／審核／財務透過服務層存取。

### 11.2 應隔離或只留安全引用的資料

- 完整銀行帳戶、付款工具、匯款檔、付款／退款／撥款憑證及對帳附件屬 S4，建議放在獨立財務工作簿或其他受控儲存；現有工作簿只留 token／受控 document ID。
- 顧問／會員完整聯絡資訊僅留於其主責分頁；deal、organization、bonus、batch 與 audit 不複製完整姓名、手機、Email、LINE UID 或地址。
- 健康資料完全排除於獎金工作簿與查詢 API。

### 11.3 同工作簿保護分頁的不足

保護範圍、隱藏分頁或 filter view 不能形成真正資料隔離：有檔案存取權者仍可能看見或匯出資料，Apps Script 也通常以部署者權限存取整個檔案。因此，若銀行與憑證留在同一工作簿，仍存在誤授權、整檔匯出、公式／外掛外洩與稽核不足風險。

### 11.4 查詢與角色控制

- Apps Script API 必須對每個 use case 建立明確欄位白名單，不得回傳整列後再由前端隱藏。
- 顧問大廳只回傳本人獎金及本人推薦會員的必要遮罩資料；有效導師只回傳當下授權團隊的必要件數與組織獎資料。
- 分支獨立後原導師只可查看案件安全識別、件數與培育獎；導師暫停或喪失時立即撤銷團隊資料權限。
- 管理者可提出、送審與處理例外；審核者核對；核准者核准；財務只處理已核准批次、payout result 與追回。即使同一自然人兼任，也必須分步、分角色、分時間留痕。
- 顧問不得直接取得正式 Sheet 權限；敏感查詢、匯出、被拒絕的越權嘗試都建立 audit event。

### 11.5 敏感設定

- Spreadsheet ID、憑證、秘密與其他敏感設定應由 Script Properties 或等效秘密管理取得，不得硬編碼於 `Config.gs` 或其他程式碼。
- 正式顧問識別、姓名與推薦識別不得作程式碼設定值、特殊分支常數或授權依據；正式身分應由受控資料與角色規則判定。
- 在敏感設定與正式顧問識別移除、`.gitignore` 補強及驗證完成前，Apps Script Repo 不得 Publish 或建立遠端。

### 11.6 是否拆成獨立財務工作簿的選項

| 選項 | 優點 | 風險／代價 | 建議 |
|---|---|---|---|
| 同一工作簿，分頁保護 | 關聯簡單、初期改動少 | 無真正隔離；整檔授權與匯出風險最高 | 僅可作過渡方案，且不可讓顧問直接存取 |
| 獨立財務工作簿 | 財務權限與憑證可與 CRM／營運分離 | 跨工作簿一致性、失敗復原與 ID 對帳更複雜 | 若保存完整帳戶／憑證，優先評估 |
| 受控物件儲存／資料庫＋Sheet 安全引用 | 權限、加密、稽核與生命週期較完整 | 建置與維運成本較高 | 中長期財務與憑證建議方向 |

本文件不替營運與財務做拆分決策，也不建立或搬移任何資料。

---

## 12. 資料量與效能

1. **避免全表掃描**：現有 `getAllRecords` 的整表讀取只適合小量與低頻查詢。正式交易查找不得每次掃描所有歷史事件、basis、batch item 或 audit rows。
2. **索引策略**：每張表把主鍵放在固定前段欄；另依 use case 維護受控索引或限定查詢鍵，例如 order ID、consultant ID、recognition period、status、batch ID、source ID 與 parent event ID。索引是可重建摘要，不是第二主帳。
3. **限定讀取範圍**：先取得 header map 與實際資料末列，再只讀必要欄與候選期間；API 不得為取得少數列而讀取整張寬表。
4. **批次分段**：計算、組批、稽核與封存按期間及穩定游標分段；每段保存 run ID、範圍、冪等鍵與完成點，失敗後從安全邊界重試。
5. **LockService**：計算鎖以獎金類型、受益人、期間與版本為範圍；批次鎖以類型、期間、版本與候選集為範圍。鎖內仍須重查唯一鍵、餘額與狀態；鎖逾時安全停止，不盲寫。
6. **Operation 查詢索引**：正式讀取應按候選 operation ID 批次取得其最新有效 journal 事件，不得每列重掃全部 audit。可維護 `operation_id → commit event／fingerprint` 的可重建快取或 Script Cache，但每次正式提交與高風險批次仍以 append-only audit journal 為權威；快取缺失或衝突時失敗關閉。
7. **Reconciliation**：排程以有限時間窗／游標檢查逾期 PREPARED、無 PREPARED 的 operation 列、manifest 缺列、終局事件衝突、指紋不符、result 衝突及摘要落後。只產生清單或追加 RECOVERED／COMMITTED／ABORTED 決議，不覆寫原列；自動 COMMITTED 只可在完整 manifest、冪等與金額檢查全部通過且政策明確允許時進行。
8. **歷史封存**：規則、快照、已核准／已發放、payout、operation commit 與 audit 不得刪除。達到經核准保留條件後可移至唯讀封存，並保留主鍵、來源、指紋、封存批次與可查鏈路；operation journal 與其權威列不可分離封存。
9. **公式限制**：避免整欄 volatile 公式、跨大量分頁的 `IMPORTRANGE` 或由公式作唯一狀態。顯示摘要採有限範圍與可重建寫回。
10. **評估資料庫時點**：不虛構正式資料量；當不可避免的全掃描導致配額／逾時、LockService 成為長期串行瓶頸、多寫入者需要真正跨表交易、孤兒／重複列難以控制、audit／basis 導致工作簿操作不穩、或需要可靠列級權限時，應正式評估資料庫。

---

## 13. 分階段落地順序

| 階段 | 內容 | 前置條件 | 本階段不得提前啟用 |
|---:|---|---|---|
| 1 | 會員、訂單及付款：members、order_members、payments、order adjustments 與 orders 摘要邊界 | 正式 Header mapping、ID 規則、個資／財務工作簿決策、遷移 dry-run；先完成 operation coordinator、四種 journal 事件與未 commit 過濾的共通元件 | 任何正式計獎；僅因部分付款建立件數或獎金；繞過 commit predicate 讀取 |
| 2 | 成交與組織快照：顧問狀態、導師資格、組織歷史、deal／organization snapshot | 階段 1 付款與成員可平衡；HC9001 修正並驗證；組織事件資料可還原；成交 operation 中斷／recovery 案例通過 | 導師、活動或組織獎自動成立；以目前組織回填歷史；使用未 COMMITTED 快照 |
| 3 | 規則版本及獎金明細：plans、bonus_rules、campaigns、bonus_records | 正式生效日與選版規則、版本重疊阻擋、規則核准角色；bonus 正式採用 operation 可重試且不重複 | 自動核准；覆寫已引用規則；以公式作獎金主帳；未提交 bonus 送審 |
| 4 | 月結計算與防重：calculation snapshots、basis items、idempotency、採用版本 | 階段 2 快照完整；階段 3 版本唯一適用；退款回溯及 calculation／basis operation reconciliation 案例通過 | 正式月結發放；沒有 basis items 的合計型獎金；部分寫入計算可見 |
| 5 | 結算、撥款及追回：batch、batch items、adjustment/recovery、payout ledger | 交易週／截單、營業日、財務 SOP、權限與對帳格式核准；operation、attempt、RESULT 三層冪等與衝突案例全部通過 | 自動撥款；修改 batch item 或 result；未知／衝突結果重試；result commit 前更新正式摘要 |
| 6 | 權限、稽核及顧問查詢：白名單、遮罩、即時撤權、audit、reconciliation 與大廳顯示 | 可信任身分驗證、角色矩陣、敏感隔離、operation journal 保留／索引、稽核與越權測試通過 | 顧問直接存取 Sheet；回傳整列；未授權團隊或財務資料；未提交孤立列外洩 |

每階段均先完成離線 mapping、測試資料、dry-run、差異報告、人工驗收與回復方案，再另案批准正式寫入；本文件本身不授權任何階段執行。

---

## 14. 驗收條件

1. Task 04A-3A 的 20 個核心實體及 Task 04A-3B 的四個撥款／批次實體均有明確儲存位置與主鍵。
2. 一筆訂單可關聯多位會員，同一會員關聯不重複；實際會員數、有效付費會員數與有效件數分開保存。
3. 一筆訂單可保存多筆付款、入帳確認與沖正；未全額入帳前不能產生可送審正式獎金。
4. deal、organization、calculation snapshots 與 basis items 建立後不可回寫；合法更正有新版本及原版關聯。
5. plan、bonus rule、campaign version 已核准或被引用後不可覆寫；重疊適用版本會阻擋。
6. 每筆成交型獎金可追到單一 deal snapshot；每筆期間彙總型獎金可逐筆追到完整 calculation snapshot 與 basis items。
7. 成交型、期間型、basis item、batch item、payout attempt 與 payout result 的防重條件明確；相同輸入或相同外部／人工結果重跑不產生同義正式結果。
8. 月結可還原納入、扣除、排除、件數、門檻／級距、擇優、規則版本及重算差異。
9. settlement batch item 鎖定後不可改；成功金額不重複發放；失敗只重試未成功餘額；未知或衝突結果先凍結、對帳與追加決議。
10. 退款、取消、方案調整、發放前差額、發放後負項追回、部分追回與跨期扣抵均保留完整鏈，且追回不超過未追回餘額。
11. 顧問只能看到本人與授權範圍資料；導師資格失效即撤權；所有會員資料遮罩且不回傳金融／健康資料。
12. 銀行、付款與撥款憑證只以受控安全引用出現在一般工作簿與 audit，不出現在一般執行記錄或顧問 API。
13. 任何獎金、批次、已發放或追回事實都不依賴公式作唯一財務主帳。
14. 系統、管理者、審核者、核准者與財務操作可分步驗證，並有 append-only audit／operation journal event。
15. 現有六個分頁每一欄都能在後續遷移 mapping 中分類為保留權威、衍生、暫停使用、搬移、增加版本／快照引用或待核對。
16. 完成正式 Header 核對、資料品質分析、遷移 dry-run 與回復方案後，才可安全進入下一階段遷移設計。
17. deal／organization、calculation／basis、bonus 正式採用、settlement／items、adjustment／recovery 及 payout result 作業皆先 PREPARED；COMMITTED 前所有實體列對正式流程與顧問查詢不可見。
18. 模擬在每個跨分頁寫入點中斷時，不會產生可送審、可組批、可撥款或可查詢的部分結果；孤立列可由 reconciliation 追加 RECOVERED 後 COMMITTED 或 ABORTED，且不覆寫原列。
19. 相同 operation idempotency key 重試命中同一 operation，不建立第二套正式資料；commit 後摘要重建失敗不影響權威可見性，後續可冪等補建。
20. 同一 RESULT 重複匯入命中相同 `result_idempotency_key`；人工匯入可由穩定批次／列鍵重現，且防重鍵不含姓名、帳號或敏感內容。
21. 同一 attempt 的互斥 RESULT 會使作業保持未提交／結果不明並停止重試；只有追加式 CORRECTION_DECISION 可解除，原 RESULT 不被改寫。
22. 任一 RESULT 提交前均驗證有效成功累計不超過 attempt 金額及 batch item 尚待處理餘額；超額會阻擋並進入 reconciliation。

---

## 15. 尚待決策與阻塞

| 類別 | 尚待決策／阻塞 | 影響 |
|---|---|---|
| 工作簿邊界 | 是否繼續使用同一工作簿 | 權限、跨表一致性、維運與封存 |
| 財務隔離 | 銀行、付款、退款、撥款憑證與對帳是否使用獨立財務工作簿或受控儲存 | S4 風險、財務角色與跨工作簿交易 |
| 正式結構 | 正式欄位名稱、分頁數、ID 格式、JSON schema、狀態碼與中文顯示字典 | 遷移 mapping 與 Apps Script repository |
| 保留期限 | 個資、交易、付款、獎金、憑證、快照與 audit 的法定保留／匿名化期限 | 封存、刪除與稽核設計 |
| 制度生效 | 新制度正式生效日與跨制度成交選版 | 規則版本選擇與歷史訂單 |
| 稅務財務 | 稅務、扣繳、匯款手續費、退匯與實發呈現 | batch item 淨額與對帳；系統不得自行推定 |
| 期間 | 交易週精確 timestamp、截單、延遲入帳／補登歸期 | 週結認列與防重 |
| 營業日 | 國定假日、補班、銀行非營業日與下一營業日正式來源 | 預定撥款日 |
| 既有資料 | 現有資料品質、空值、重複、格式、下拉覆蓋與遷移／修復策略 | 是否可建立可靠快照與版本 |
| 推薦保護 | HC9001 推薦保護寫入異常 | 修正並驗證前，推薦歸屬及衍生獎金不得自動成立 |
| Apps Script 安全 | 敏感設定與正式顧問識別尚未移除，`.gitignore` 仍待補強 | Apps Script Repo 不得 Publish 或建立遠端 |
| 職責分離 | 是否強制審核者與核准者不同、哪些情形需雙人覆核 | 核准與 payout 流程 |
| Payout SOP | 失敗、退匯、帳戶錯誤、部分成功、結果不明與更正的正式營運流程 | retry、decision、對帳與顧問文字 |
| Operation 協定 | PREPARED 逾期門檻、manifest schema、可自動 recovery 的範圍、人工 COMMITTED／ABORTED 權限及終局衝突 SOP | 孤立列偵測、失敗關閉與營運復原 |
| 舊資料可見性 | 既有無 operation_id 資料如何由遷移 operation 採用；過渡白名單是否允許及何時移除 | 不得把空 operation_id 默認為已提交 |
| Journal 索引與保留 | operation commit 快取／索引的重建頻率、audit 封存時 operation 與權威列的共同封存策略 | 正式讀取效能與歷史可見性 |
| 防重技術 | operation／attempt／result 正規化、正式雜湊、人工匯入批次／列鍵、`NOT_APPLICABLE`、版本序號與索引格式 | Sheet service 的一致實作 |
| 擇優稽核 | 條件同額時正式採用規則 ID 的標示方式 | 月結 calculation snapshot |

### 15.1 明確結論

完成本文件只代表具備實體儲存與欄位映射的設計基礎，**不代表可以建立正式分頁、修改或搬移正式資料、發布 Apps Script Repo、建立 Apps Script 遠端、部署、啟用獎金功能、自動核准、自動撥款或開放顧問查詢**。任何正式異動都必須另案完成 Header 核對、資料品質分析、遷移設計、敏感隔離、dry-run、人工驗收與明確核准。
