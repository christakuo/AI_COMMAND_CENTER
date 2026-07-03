# Task 04H-2H-Doc：LIFF 顧問獎金畫面實作結果紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04H-2H-Doc：LIFF 顧問獎金畫面實作結果文件化 |
| 完成日期 | 2026-07-03 |
| 實作任務 | Task 04H-2H：LIFF 顧問獎金 read-only 畫面第一版 |
| 實作狀態 | 使用者回報 PR 已合併，`yuanxin-liff-demo` main clean |
| 正式開放狀態 | 尚未開放；`consultantBonusEnabled` 仍為 `false` |
| 本次工作 | 只文件化既有實作與測試結果 |

## 2. 任務目的

Task 04H-2H 的目的如下：

- 在 LIFF 顧問大廳加入「我的獎金」read-only 第一版畫面。
- 預備串接 `getConsultantBonusDashboard` 與 `getConsultantBonusRecords`。
- 實作獎金總覽、獎金明細及月份／狀態篩選。
- 保留獎金區自己的 loading、`not_initialized`、empty、error 及 content 狀態。
- 在正式 Apps Script 同步及 `not_initialized` 驗收前維持功能關閉，不正式向顧問開放。

## 3. 修改 Repo 與檔案

| 項目 | 結果 |
|---|---|
| Repo | `yuanxin-liff-demo` |
| 修改檔案 | `index.html`、`referral.js` |
| 新增檔案 | 無 |
| PR | 使用者回報已合併 |
| Main | clean |
| Vercel 部署 | 未執行 |
| Web App URL | 未修改 |
| Apps Script | 未修改 |
| Google Sheet | 未修改 |

本文件只記錄 Task 04H-2H 已完成的前端程式結果，本次未再次修改 `yuanxin-liff-demo`。

## 4. 前端新增內容

### `index.html`

已新增：

- 「我的獎金」DOM 區塊。
- 暖象牙白、深青綠與香檳金的視覺樣式。
- 獎金總覽卡片。
- 月份與狀態篩選。
- 獎金明細列表。
- Loading 狀態。
- `not_initialized` 狀態。
- Ready＋empty 狀態。
- Error 狀態。
- Content 狀態。
- 金額格式化。
- 日期格式化。
- 狀態格式化。
- 已遮罩來源摘要呈現。
- `nextPageToken` 暫存；第一版不顯示分頁操作。

### 主要函式

- `initializeConsultantBonus()`
- `loadConsultantBonus()`
- `setConsultantBonusState()`
- `renderConsultantBonusDashboard()`
- `renderConsultantBonusRecords()`
- `createConsultantBonusRecord()`

## 5. API 串接與 Payload 白名單

### API actions

| 用途 | Action |
|---|---|
| 獎金總覽 | `getConsultantBonusDashboard` |
| 獎金明細 | `getConsultantBonusRecords` |

### 查詢規則

- `limit` 固定為 `50`。
- 月份格式為 `YYYY-MM`。
- `status` 支援：
  - `all`
  - `pending_review`
  - `approved`
  - `paid`

### `referral.js` 白名單

`queryConsultantPortal()` 僅新增以下明確白名單欄位：

- `month`
- `status`
- `limit`
- `nextPageToken`

安全邊界：

- 沒有使用任意 `...data` 展開。
- `consultantId` 不會穿透到 bonus API payload。
- `internalNote` 不會穿透。
- `paymentBatchId` 不會穿透。
- 其他未列入白名單的呼叫端欄位不會穿透。

顧問身分仍由後端驗證 LINE token 後決定，前端不得指定查詢其他顧問。

## 6. 獎金總覽顯示

已實作顯示：

- 待審核金額。
- 已核准金額。
- 已付款金額。
- 本月總獎金。
- 本月筆數。
- 更新時間。
- 「獎金資料以後台審核結果為準」提示。

顯示規則：

- 金額格式為 `NT$4,000`。
- 前端只做顯示格式化。
- 前端沒有從會員名單自行推算獎金。
- 前端沒有自行計算獎金。
- 所有獎金數字以 Dashboard API 回傳為準。

## 7. 獎金明細顯示

每筆只使用以下白名單欄位：

- `recognizedDate`
- `bonusType`
- `amount`
- `status`
- `paidStatus`
- `paidAt`
- `sourceSummary`
- `updatedAt`

前端未顯示：

- 內部備註。
- 付款批次。
- LINE 身分資料。
- 完整會員姓名。
- 完整手機。
- 其他顧問 ID。

`sourceSummary` 只呈現後端已完成遮罩的摘要，前端不接收完整個資後再自行遮罩。

## 8. 狀態處理

| 狀態 | 顯示方式 |
|---|---|
| `not_initialized` | 中性顯示「獎金查詢功能建置中」 |
| Ready＋empty | 顯示「目前查無本月獎金資料」 |
| `TOKEN_MISSING` | 提示重新由 LINE 開啟或登入 |
| `ACCESS_DENIED` | 提示確認是否使用顧問綁定帳號 |
| `INVALID_ARGUMENT` | 提示重新選擇查詢條件 |
| `INTERNAL_ERROR` | 提示稍後再試 |
| 未知錯誤 | 使用安全通用文案，不顯示 Apps Script 原始錯誤 |

隔離規則：

- 獎金 API 失敗只影響「我的獎金」區塊。
- 不會讓整個顧問大廳失效。
- 不使用整個顧問大廳的紅色錯誤區呈現獎金查詢問題。
- 不顯示 stack trace 或後端內部資訊。

## 9. 暫時保護機制

已加入：

```text
consultantBonusEnabled: false
```

目前行為：

- 只顯示建置中文案。
- 不會呼叫正式環境尚未支援的 bonus action。
- 前端尚未正式向顧問開放獎金查詢。

開啟條件：

1. 正式 Apps Script 已同步 read-only bonus API。
2. Access Token Fix1 已同步正式環境。
3. 正式 Apps Script 已重新部署。
4. 正式 `not_initialized` 空狀態驗收通過。
5. 已確認查詢不寫入 Sheet、不自動建立 `bonus_records_獎金明細表`。

在上述條件完成前，不得將 `consultantBonusEnabled` 改為 `true`。

## 10. 測試與檢查結果

依 Task 04H-2H 實作結果回報，Codex 已完成：

| 檢查項目 | 結果 |
|---|---|
| `referral.js` 語法檢查 | 通過 |
| `index.html` 三段 inline script 語法檢查 | 通過 |
| Git diff 格式檢查 | 通過 |
| 55 個 DOM ID 重複檢查 | 無重複 |
| API payload 白名單 mock | 通過 |
| `consultantId` 等敏感欄位阻擋 | 通過 |
| `not_initialized` mock | 通過 |
| Ready＋empty mock | 通過 |
| Dashboard success mock | 通過 |
| Records success mock | 通過 |
| `TOKEN_MISSING` mock | 通過 |
| `ACCESS_DENIED` mock | 通過 |
| `INVALID_ARGUMENT` mock | 通過 |
| `INTERNAL_ERROR` mock | 通過 |
| Token／UID console 輸出掃描 | 未發現不當輸出 |
| 寫入型 bonus action 掃描 | 未發現 |
| 本機 HTTP 頁面 | 成功回應 |

補充限制：自動瀏覽器 DOM 快照逾時，因此本次未將視覺截圖列為驗收證據。上述檢查證明語法、狀態邏輯及資料邊界，但正式開放前仍需完成瀏覽器與正式環境驗收。

## 11. 尚未完成

- 未部署 Vercel。
- 未正式開放顧問獎金入口。
- 正式 Apps Script 尚未同步 read-only bonus API。
- Access Token Fix1 尚未同步正式 Apps Script。
- 正式 Apps Script 尚未重新部署。
- 正式 `not_initialized` 尚未驗收。
- 正式 Google Sheet 尚未修改。
- 正式 `bonus_records_獎金明細表` 尚未建立。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。
- `consultantBonusEnabled` 尚未開啟。

## 12. 後續任務建議

1. Task 04H-2I：正式 Apps Script 同步 read-only API 與 Access Token Fix1。
2. Task 04H-2I-1：建立正式 Apps Script deployment。
3. Task 04H-2I-2：正式 `not_initialized` 空狀態驗收。
4. Task 04H-2J：正式 LIFF 顧問入口 read-only 驗收。
5. Task 04H-2K：全部驗收通過後開啟 `consultantBonusEnabled`。
6. 後續另開任務處理正式 `bonus_records_獎金明細表` 建表與正式獎金草稿產生。

前五項只處理 read-only 查詢與安全空狀態，不授權提前執行正式 Generate。

## 13. 安全聲明

- 本次只文件化已完成的前端實作結果。
- 未記錄任何 token。
- 未記錄完整 LINE UID。
- 未記錄 Channel ID。
- 未記錄測試 Web App URL。
- 未修改 `yuanxin-liff-demo`。
- 未修改 Apps Script repo。
- 未部署。
- 未執行 Apps Script 函式。
- 未修改 Google Sheet。
- 未建立正式 `bonus_records_獎金明細表`。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未產生正式獎金草稿。

## 14. 結論

Task 04H-2H 已完成 LIFF 顧問獎金 read-only 第一版畫面，包含「我的獎金」區塊、總覽、明細、篩選、局部狀態處理及查詢參數白名單。`consultantBonusEnabled=false` 持續阻擋正式 API 呼叫與入口開放。

下一步應先同步及部署正式 Apps Script read-only API，完成 `not_initialized` 安全空狀態驗收；在此之前不得開啟正式顧問獎金入口。
