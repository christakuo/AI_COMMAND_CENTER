# Task 04H-2C：顧問獎金／組織發展 read-only API 與畫面欄位規格

## 1. 任務名稱與目的

本任務定義顧問端第一版可安全顯示的獎金與組織發展欄位，以及 Apps Script read-only API 合約，讓後續實作可沿用既有顧問身分驗證與顧問大廳能力，避免前端先放入假資料、固定 0 元或未經驗證的獎金金額。

本規格只定義查詢與顯示，不建立正式獎金資料、不修改現有獎金計算規則，也不包含核准、付款或顧問自行修改功能。

## 2. 目前完成與未完成狀態

### 已完成或可沿用

- 已有顧問入口與顧問大廳。
- 已有 LINE 身分驗證與顧問身分解析。
- 已有 `getConsultantDashboard`。
- 已有 `getMyLeads`、`getMyConsultants`、`getTeamLeads` 與 `getConsultantResources`。
- 已有顧問推薦會員、推薦顧問、團隊顧問及團隊會員的基礎資料能力。
- 前端已有顧問基本資料、顧問 ID、姓名、位階、狀態、推薦會員、推薦顧問、團隊會員及顧問資源中心。
- 組織發展第一版可先整理 `getConsultantDashboard` 已回傳的摘要資料。

### 尚未完成

- 尚無 `getConsultantBonusDashboard`。
- 尚無 `getConsultantBonusRecords`。
- 尚無獨立 `getConsultantOrgDashboard`。
- 尚無獎金總覽或獎金明細畫面。
- 尚無待審核、已核准、已付款、月結或付款紀錄畫面。
- 尚無獨立組織發展畫面或組織圖。
- 正式 `bonus_records_獎金明細表` 尚未建立。
- `ensureBonusRecordsSheet()` 尚未正式執行。
- `generateConsultantBonusDrafts()` 尚未正式執行。
- 尚未產生正式獎金草稿。

因此第一版必須同時支援正常空資料及「後台獎金資料尚未建立」兩種不同狀態，不能一律顯示為 0 元。

## 3. 顧問端第一版畫面範圍

### 第一版納入

1. 我的獎金卡片。
2. 獎金明細列表。
3. 組織發展卡片。
4. 直推顧問／團隊成果摘要。
5. 顧問資源中心新增「獎金制度與明細」入口。

### 第一版暫不納入

- 自動核准。
- 自動撥款。
- 月結付款批次。
- 導師／團隊獎金計算。
- 顧問推薦顧問獎金計算。
- 複雜組織樹。
- 排行榜。
- 顧問自行申請付款。
- 顧問查看下層顧問獎金金額。
- 管理端財務後台。
- 顧問修改獎金、組織或會員資料。

## 4. 我的獎金卡片 UI 欄位

| 畫面欄位 | API 欄位 | 顯示規則 |
|---|---|---|
| 本月待審核金額 | `summary.pendingReviewAmount` | 金額格式化；只有 `setupStatus=ready` 才顯示 |
| 本月已核准金額 | `summary.approvedAmount` | 金額格式化 |
| 本月已付款金額 | `summary.paidAmount` | 只計已付款紀錄，不以「是否已付款＝否」的草稿推算 |
| 全部待審核筆數 | `summary.pendingReviewCount` | 非負整數 |
| 全部已核准筆數 | `summary.approvedCount` | 非負整數 |
| 全部已付款筆數 | `summary.paidCount` | 非負整數 |
| 最近更新時間 | `updatedAt` | `yyyy-MM-dd HH:mm:ss`；無資料可顯示 `—` |
| 查看獎金明細 | 按鈕／路由 | 進入獎金明細列表 |

固定說明文字：

> 實際獎金、審核結果與付款日期以元馨後台審核為準。

### 空狀態

| API 狀態 | 畫面文案 | 金額卡片 |
|---|---|---|
| API 正常、`setupStatus=ready`、`empty=true` | 目前尚無獎金紀錄 | 可顯示空狀態；不得暗示已有計算結果 |
| `setupStatus=not_initialized` | 獎金資料建置中，尚未開放查詢 | 不顯示固定 0 元 |
| API 錯誤 | 獎金資料暫時無法載入，請稍後再試 | 不保留舊金額假裝成功 |

## 5. 獎金明細列表 UI 欄位

### 顧問可見白名單

| API 欄位 | 畫面用途 | 資料來源／處理 |
|---|---|---|
| `bonusId` | 獎金編號 | 獎金 ID |
| `recognizedDate` | 認列日期 | 成交日期或認列日期；只回傳日期 |
| `transactionDate` | 交易／成交時間 | 若第一版需要顯示時間，可由成交日期安全格式化；沒有資料時為空字串 |
| `bonusType` | 獎金類型 | 例如會員推薦獎金 |
| `amount` | 獎金金額 | 數值，不回傳格式化字串 |
| `status` | 審核狀態 | 後端集中映射顯示值 |
| `paidStatus` | 是否付款 | `是`／`否` 或穩定 code 映射 |
| `paidAt` | 付款日期 | 無正式付款日期時為空字串，不可用更新時間代替 |
| `sourceSummary` | 安全來源摘要 | 例如「會員推薦案」，不得含完整會員個資 |
| `updatedAt` | 最近更新時間 | 安全格式化日期時間 |

第一版畫面可優先顯示 `recognizedDate`；`transactionDate` 僅在後端有可靠來源時使用，兩者不得用同一個不明欄位重複包裝。

### 禁止顯示或回傳

- 完整會員姓名。
- 完整手機號碼。
- LINE UID。
- 內部備註。
- 付款批次內部 ID。
- 審核人姓名或內部識別。
- 其他顧問 ID。
- 其他顧問獎金。
- 後台風控原因細節。
- Sheet 完整資料列。
- 金融帳號、憑證、退款或稽核內部資料。

### 狀態文案

| 穩定狀態語意 | 顧問端文案 | 第一版處理 |
|---|---|---|
| `pending_review` | 待審核 | 支援 |
| `approved` | 已核准 | 支援；有正式核准資料才顯示 |
| `paid` | 已付款 | 支援；有正式付款事實才顯示 |
| `rejected` | 已駁回 | 預留；制度與後端採用後才啟用 |
| `needs_information` | 待補件 | 預留，不先顯示假狀態 |
| `pending_confirmation` | 待確認 | 預留，不先顯示假狀態 |

「待補件／待確認」若尚未正式納入狀態制度，只能保留在規格，不得由前端自行推導。

## 6. 組織發展卡片 UI 欄位

| 畫面欄位 | 建議 API 欄位 | 說明 |
|---|---|---|
| 顧問 ID | `consultant.consultantId` | 由可信任身分解析 |
| 顧問姓名 | `consultant.consultantName` | 本人資料 |
| 目前位階 | `consultant.rank` | 只表示目前位階 |
| 導師資格／導師權限 | `consultant.mentorStatus` | 無資料時顯示未設定，不自行推論 |
| 我的推薦顧問數 | `counts.directConsultantCount` | 本人直接推薦顧問數 |
| 團隊顧問數 | `counts.teamConsultantCount` | 目前可授權查看的團隊摘要 |
| 我的推薦會員數 | `counts.directLeadCount` | 本人推薦會員／名單數 |
| 團隊會員數 | `counts.teamLeadCount` | 目前團隊會員／名單摘要 |
| 直接會員成交件數 | `counts.directDealUnits` | 只有可靠欄位時顯示；否則暫不顯示 |
| 團隊會員成交件數 | `counts.teamDealUnits` | 只作目前摘要，不回算歷史獎金 |
| 資料截至時間 | `dataAsOf` | 告知使用者資料時間點 |

固定說明文字：

> 此區顯示目前組織現況，不代表歷史獎金歸屬。

### 第一版不做

- 複雜組織樹。
- 歷史組織回溯。
- 依現況組織回算過去獎金。
- 顯示下層顧問收入。
- 顯示團隊會員完整個資。
- 將目前上層關係當成歷史獎金快照。

## 7. `getConsultantBonusDashboard` API 規格

### Action

`getConsultantBonusDashboard`

### 安全原則

- 只能使用 `POST`。
- 必須使用 LINE ID token 或 access token 驗證。
- 後端由 token 推導 LINE 使用者，再解析顧問 ID。
- 不接受前端傳入 `consultantId` 查詢。
- 只彙總該顧問本人的獎金資料。
- `bonus_records` 不存在時不得自動建立，回傳 `setupStatus=not_initialized`。
- 分頁存在但該顧問無資料時，回傳 `setupStatus=ready`、`empty=true`。
- 不得在此查詢 API 內呼叫 `ensureBonusRecordsSheet()` 或 Generate。

### Request

```json
{
  "action": "getConsultantBonusDashboard",
  "lineIdToken": "...",
  "lineAccessToken": "...",
  "pageName": "consultant_bonus_dashboard"
}
```

`lineIdToken` 與 `lineAccessToken` 的實際必填／備援規則沿用既有可信任 LINE 驗證服務；不可因其中一項空白就接受前端傳入顧問 ID 代替。

### Success Response

```json
{
  "success": true,
  "setupStatus": "ready",
  "empty": true,
  "consultant": {
    "consultantId": "HCxxxx",
    "consultantName": "顧問姓名"
  },
  "summary": {
    "month": "YYYY-MM",
    "pendingReviewAmount": 0,
    "approvedAmount": 0,
    "paidAmount": 0,
    "pendingReviewCount": 0,
    "approvedCount": 0,
    "paidCount": 0,
    "totalAmount": 0,
    "totalCount": 0
  },
  "updatedAt": "yyyy-MM-dd HH:mm:ss",
  "message": "目前尚無獎金紀錄"
}
```

### 尚未建表 Response

```json
{
  "success": true,
  "setupStatus": "not_initialized",
  "empty": true,
  "consultant": {
    "consultantId": "HCxxxx",
    "consultantName": "顧問姓名"
  },
  "summary": null,
  "updatedAt": "",
  "message": "獎金資料建置中，尚未開放查詢"
}
```

`not_initialized` 時 `summary` 使用 `null`，避免前端把尚未建立的資料誤顯示為已計算的 0 元。

### Error Response

```json
{
  "success": false,
  "errorCode": "TOKEN_MISSING",
  "message": "無法確認顧問身分"
}
```

允許錯誤碼：

| 錯誤碼 | 使用時機 | 前端處理 |
|---|---|---|
| `TOKEN_MISSING` | 缺少可驗證 LINE token | 顯示登入／身分驗證提示 |
| `CONSULTANT_NOT_FOUND` | LINE 使用者未解析到有效顧問 | 顯示無法確認顧問身分 |
| `ACCESS_DENIED` | 顧問停權、權限不足或越權請求 | 不顯示任何獎金資料 |
| `INTERNAL_ERROR` | 安全處理後的非預期錯誤 | 顯示稍後再試，不回傳內部細節 |

## 8. `getConsultantBonusRecords` API 規格

### Action

`getConsultantBonusRecords`

### 安全原則

- 只能使用 `POST`。
- 必須使用 LINE token 驗證。
- 後端由 token 推導顧問 ID。
- 不接受前端傳入 `consultantId`。
- 只能回傳該顧問本人的獎金紀錄。
- 第一版不允許導師查詢下層顧問獎金。
- `bonus_records` 不存在時回傳 `not_initialized`，不得自動建立。
- Response 必須使用明確欄位白名單，不得直接序列化完整 Sheet row。
- 月份、狀態、limit 與 page token 必須由後端驗證。

### Request

```json
{
  "action": "getConsultantBonusRecords",
  "lineIdToken": "...",
  "lineAccessToken": "...",
  "month": "YYYY-MM",
  "status": "all",
  "limit": 50,
  "pageToken": null
}
```

允許的 `status`：`all`、`pending_review`、`approved`、`paid`。第一版 `limit` 建議上限為 50；超過時由後端縮限或回傳安全參數錯誤，不得無限制讀取整張 Sheet。

### Response

```json
{
  "success": true,
  "setupStatus": "ready",
  "empty": false,
  "records": [
    {
      "bonusId": "BN000001",
      "recognizedDate": "yyyy-MM-dd",
      "transactionDate": "yyyy-MM-dd HH:mm:ss",
      "bonusType": "會員推薦獎金",
      "amount": 4000,
      "status": "待審核",
      "paidStatus": "否",
      "paidAt": "",
      "sourceSummary": "會員推薦案",
      "updatedAt": "yyyy-MM-dd HH:mm:ss"
    }
  ],
  "pagination": {
    "limit": 50,
    "nextPageToken": null
  },
  "message": ""
}
```

### Response 欄位白名單

只允許：

- `success`
- `setupStatus`
- `empty`
- `records[].bonusId`
- `records[].recognizedDate`
- `records[].transactionDate`
- `records[].bonusType`
- `records[].amount`
- `records[].status`
- `records[].paidStatus`
- `records[].paidAt`
- `records[].sourceSummary`
- `records[].updatedAt`
- `pagination.limit`
- `pagination.nextPageToken`
- `message`

禁止將 Sheet row 先整列轉成物件後直接回傳；應由服務層逐欄建立安全 DTO。

## 9. `getConsultantOrgDashboard` API 規格草案

### 第一版是否需要新 API

第一版可先沿用 `getConsultantDashboard` 已有摘要，減少後端與前端同時改動。若既有 Response 欄位名稱混雜、含不必要資料，或需要新增乾淨的直推顧問列表，再新增 `getConsultantOrgDashboard`。

無論沿用或新增，都只能表示「目前組織現況」，不能表示歷史獎金歸屬，也不能用來回算過去團隊獎金。

### Action

`getConsultantOrgDashboard`

### Response 草案

```json
{
  "success": true,
  "consultant": {
    "consultantId": "HCxxxx",
    "consultantName": "顧問姓名",
    "rank": "目前位階",
    "mentorStatus": "導師資格狀態"
  },
  "counts": {
    "directConsultantCount": 0,
    "teamConsultantCount": 0,
    "directLeadCount": 0,
    "teamLeadCount": 0,
    "directDealUnits": null,
    "teamDealUnits": null
  },
  "directConsultants": [
    {
      "consultantId": "HCxxxx",
      "consultantName": "顧問姓名",
      "rank": "目前位階",
      "status": "有效"
    }
  ],
  "teamConsultants": null,
  "dataAsOf": "yyyy-MM-dd HH:mm:ss",
  "warningText": "此區顯示目前組織現況，不代表歷史獎金歸屬"
}
```

第一版建議只回傳 `directConsultants`；`teamConsultants` 可先為 `null`，只顯示團隊總數。待後端完成層級、權限、停權及最小揭露驗證後，再開放分頁式團隊清單。

安全原則同樣包含：POST、LINE token 驗證、後端推導顧問 ID、不接受前端傳入查詢對象、不回傳會員完整個資、不回傳下層收入。

## 10. 19 欄 `bonus_records` MVP 是否足夠

### 第一版獎金明細

依 Task 04B／04C 已文件化的 MVP 寫入結果，現有 19 欄設計至少包含或可支援：獎金 ID、名單 ID、會員遮罩資料、顧問資料、推薦碼、成交日期、成交狀態、獎金類型、金額、獎金狀態、是否付款、建立來源與更新資訊。因此原則上足以支援第一版會員推薦獎金唯讀明細。

正式 API 實作前仍須從 `BonusService.gs` 的 Header 常數逐欄核對 19 欄實際名稱；本規格不以推測取代程式映射。

### 第一版獎金總覽

可由 API 衍生：

- 當月各狀態金額合計。
- 各狀態筆數。
- 全部筆數與總金額。
- 最近更新時間。
- `empty`。
- 安全的 `sourceSummary`。
- 顧問端狀態 code／中文文案映射。

### 可能缺少但不阻塞第一版查詢

- 正式審核者、審核事件與核准時間。
- 正式付款日期／付款完成事件。
- 月結認列期間及計算快照。
- 付款批次與批次明細。
- 調整、退款、追回及完整狀態歷程。
- 規則版本、組織快照與防重版本鍵。
- 導師、團隊及期間型獎金計算依據。

`paidAt` 若 19 欄中沒有可靠欄位，API 第一版應回傳空字串，畫面顯示 `—`；不可用最後更新時間或成交日期假裝付款日期。

### 是否應在正式建表前調整 Header

第一版 read-only 會員推薦獎金查詢不建議先擴張 19 欄 Header，理由如下：

1. 測試 Sheet 已驗證現有 Header 與 Generate／防重流程。
2. 正式表尚未建立，先改 Header 會同時改變既有寫入合約與測試基準。
3. 第一版缺少的月結、審核事件及付款批次，本來就不應全部塞回單一 MVP 表。
4. API 可先用衍生欄位與空值處理，不製造不存在的財務事實。

但 Task 04H-2D 實作前，必須唯讀核對實際 19 欄 Header 與 DTO 映射。如果連 `bonusId`、顧問 ID、獎金類型、金額、狀態、付款狀態、認列／成交日期及更新時間都無法可靠取得，應先停下並另開 Header 修訂任務。

## 11. 資料狀態與前端文案規則

### `not_initialized`

- 意義：正式 `bonus_records` 尚未建立。
- API：`success=true`、`setupStatus=not_initialized`、`empty=true`、`summary=null` 或 `records=[]`。
- 文案：**獎金資料建置中，尚未開放查詢**。
- 禁止顯示固定 0 元。

### `ready + empty`

- 意義：系統已啟用且分頁存在，但該顧問目前沒有獎金紀錄。
- API：`setupStatus=ready`、`empty=true`。
- 文案：**目前尚無獎金紀錄**。
- 可顯示空列表；若顯示 0 元，須明確表示這是已啟用系統的查詢結果。

### `ready + records`

- 意義：系統已啟用，且該顧問有可查詢紀錄。
- API：`setupStatus=ready`、`empty=false`。
- 畫面：正常顯示獎金總覽及白名單明細。

前端必須先判斷 `setupStatus`，再判斷 `empty`；不能把 `not_initialized` 當成 `ready + empty`。

## 12. 前端整合策略

1. 不先建立假資料。
2. 不先顯示固定 0 元。
3. 先完成 API 規格與 Apps Script read-only API。
4. 完成 `not_initialized`、空資料、正常資料與錯誤狀態測試後，前端才新增我的獎金卡片與明細列表。
5. 組織發展第一版先整理既有 `getConsultantDashboard` 資料；不足時才新增乾淨 API。
6. 顧問資源中心可先新增獎金制度文件入口。
7. 個人獎金查詢不能用靜態制度文件或固定文案代替正式 API。
8. 畫面不應根據推薦會員數自行估算獎金。
9. 前端不得自行將「待審核」改成「已核准」或「已付款」。

## 13. 後續任務建議

### Task 04H-2D：Apps Script read-only API 實作

- 實作 `getConsultantBonusDashboard`。
- 實作 `getConsultantBonusRecords`。
- `bonus_records` 不存在時安全回傳 `not_initialized`。
- 建立 Response 欄位白名單／DTO。
- 沿用 LINE token 驗證及顧問身分解析。
- 不接受 `consultantId`。

### Task 04H-2E：Apps Script read-only API 測試

- 正式 `bonus_records` 不存在的空狀態。
- 測試 Sheet 有資料狀態。
- 無資料狀態。
- token 缺失、無顧問身分及越權測試。
- 顧問只能查本人資料。
- 欄位白名單及敏感欄位未洩漏測試。

### Task 04H-2F：LIFF 顧問獎金卡片與明細畫面實作

- 串接兩個 read-only API。
- 完成三種 setup／empty 狀態與錯誤畫面。
- 不加入假資料或固定金額。

### Task 04H-2G：LIFF 組織發展卡片／清單整理

- 優先整理 `getConsultantDashboard` 既有資料。
- 需要乾淨欄位時，再實作 `getConsultantOrgDashboard`。
- 第一版先做卡片與直推顧問列表，不做複雜組織樹。

### Task 04H-2H：文件化與正式 Preview／空狀態驗收

- 文件化 API、前端狀態及權限驗收結果。
- 驗證正式尚未建表時只顯示 `not_initialized`。
- 不因查詢畫面而建立 Sheet 或產生獎金草稿。

## 14. 安全限制

- 本規格只定義 read-only 查詢。
- 不建立正式 `bonus_records_獎金明細表`。
- 不執行 `ensureBonusRecordsSheet()`。
- 不執行 `generateConsultantBonusDrafts()`。
- 不新增自動核准。
- 不新增自動付款。
- 不讓顧問修改資料。
- 不讓顧問查詢他人獎金。
- 不顯示完整會員個資。
- 不顯示下層顧問收入。
- 不修改 Apps Script repo。
- 不修改 LIFF repo。
- 不修改 Google Sheet。
- 不部署或執行 Apps Script 函式。

## 15. 結論

Task 04H-2C 已完成顧問獎金／組織發展第一版 read-only API 與畫面欄位規格。下一步應先實作並測試 Apps Script read-only API，再修改 LIFF；正式 `bonus_records` 尚未建立，正式 Generate 仍未執行，前端尚未修改。
