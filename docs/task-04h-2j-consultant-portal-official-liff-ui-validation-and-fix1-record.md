# Task 04H-2J／04H-2J-Fix1：正式 LIFF 顧問大廳 UI 驗收與小修正紀錄

## 1. 任務資訊

| 項目 | 內容 |
|---|---|
| 任務名稱 | Task 04H-2J-Fix1-Doc：文件化正式 LIFF 顧問大廳 UI 上線驗收與小修正 |
| 完成日期 | 2026-07-03 |
| 相關 Repo | `yuanxin-liff-demo` |
| Task 04H-2J | 正式 LIFF 顧問大廳 UI 驗收完成 |
| Task 04H-2J-Fix1 | 顧問資源文字顏色與返回 LINE 行為修正完成 |
| PR／Merge | 使用者回報 Fix1 PR 已合併至 `main`；PR 編號未提供 |
| 本次工作 | 只修改 AI_COMMAND_CENTER Markdown 文件 |

## 2. 任務目的

本文件記錄：

- Task 04H-2J 正式 LIFF 顧問大廳新版品牌 UI 的實際驗收結果。
- Task 04H-2J-Fix1 顧問資源中心文字可讀性及返回 LINE 行為的小修正。
- `liff.closeWindow()` 返回位置與 LIFF 開啟來源的關係。
- 顧問獎金功能仍未開放的安全邊界及後續工作。

## 3. 正式 LIFF 顧問大廳驗收結果

| 驗收項目 | 結果 |
|---|---|
| 正式顧問入口顯示新版品牌 UI | 通過 |
| 顧問資料卡 | 正常 |
| 四項成果摘要 | 正常 |
| 「我的獎金」狀態 | 維持建置中／未開放 |
| 推薦／團隊 tabs | 正常 |
| 顧問資源中心 | 正常 |
| 返回入口 | 後續由 Fix1 調整為「返回官方 LINE」 |
| 會員首頁回歸確認 | 未受影響 |
| `consultantBonusEnabled` | 維持 `false` |
| 整體功能異常 | 未發現 |

驗收結論：正式 LIFF 顧問大廳品牌 UI 可正常使用，既有顧問資訊與會員首頁未受影響；獎金區仍依安全開關維持未開放狀態。

## 4. Task 04H-2J-Fix1 修改內容

### 顧問資源中心文字可讀性

- 公開連結提示文字改為較深青綠，提高淺色背景上的可讀性。
- 顧問限定提示維持香檳金棕風格。
- 權限類型與更新時間文字同步加深。
- 正式 LINE 環境驗收文字顏色正常。

### 返回 LINE 行為

- 最下方按鈕文案改為「返回官方 LINE」。
- 在 LIFF 內優先使用 `liff.closeWindow()`。
- 非 LIFF 環境或 `closeWindow` 不可使用時，保留原有 fallback。

### 未變更項目

- `consultantBonusEnabled` 仍為 `false`。
- `referral.js` 未修改。
- Apps Script 未修改。
- Google Sheet 未修改。
- Webhook URL 未修改。
- LIFF／LINE token 流程未修改。
- Bonus API 邏輯未修改。

## 5. `liff.closeWindow()` 行為說明

`liff.closeWindow()` 的作用是關閉目前 LIFF 視窗，關閉後回到使用者原本開啟 LIFF 的 LINE 畫面。返回位置由「開啟來源」決定，不是由按鈕文案強制指定聊天室。

| LIFF 開啟來源 | 按下「返回官方 LINE」後的預期結果 |
|---|---|
| 元馨醫管家官方 LINE 選單或官方 LINE 訊息 | 返回官方 LINE 畫面 |
| 私人 LINE、Notes 或其他非官方聊天室 | 返回原本開啟 LIFF 的私人或非官方 LINE 畫面 |

因此，從私人 LINE 或 Notes 開啟時沒有回到官方 LINE，屬於 `liff.closeWindow()` 的 LINE 預期行為，不是程式錯誤。目前不再修改此行為。

操作文案仍使用「返回官方 LINE」，其主要使用情境是顧問由官方 LINE 入口進入；營運上應優先從官方 LINE 選單或官方訊息提供顧問入口。

## 6. 安全限制

- `consultantBonusEnabled` 尚未開啟。
- 「我的獎金」仍顯示建置中／未開放狀態。
- 未因 UI 驗收建立或顯示正式顧問獎金資料。
- 未修改 Apps Script、Google Sheet、Webhook URL、token 流程或 bonus API 邏輯。
- 未執行 `ensureBonusRecordsSheet()`。
- 未執行 `generateConsultantBonusDrafts()`。
- 未建立正式 `bonus_records_獎金明細表`。
- 未產生正式獎金草稿。

正式 UI 驗收通過不等於顧問獎金資料與查詢功能已正式開放。

## 7. 未處理事項

- `consultantBonusEnabled` 尚未開啟。
- 顧問獎金正式資料尚未建立或顯示。
- 顧問獎金正式開放與真實資料驗收仍待後續任務。
- 正式 `bonus_records_獎金明細表` 尚未建立。
- 尚未產生正式獎金草稿。
- 示範報告頁仍維持舊深色風格。
- 身分安全驗證頁仍維持舊深色風格。
- 示範報告及身分安全驗證頁將於後續報告上傳與分析系統重做時統一調整。

## 8. 後續建議

1. 維持 `consultantBonusEnabled=false`，直到正式獎金 read-only 後端與資料狀態完成驗收。
2. 以官方 LINE 選單或官方 LINE 訊息作為正式顧問入口，讓「返回官方 LINE」符合主要使用流程。
3. 後續另開任務完成正式顧問獎金資料、空狀態及真實資料驗收。
4. 正式獎金驗收通過後，才評估開啟 `consultantBonusEnabled`。
5. 報告上傳與分析系統重做時，再統一示範報告及身分安全驗證頁的品牌視覺。

## 9. 本次文件化安全聲明

- 本次只修改 AI_COMMAND_CENTER 文件。
- 未修改 `yuanxin-liff-demo`。
- 未修改 Apps Script。
- 未修改 Google Sheet。
- 未修改任何部署設定。
- 未執行部署。
- 未記錄 token、完整 LINE UID、Channel ID 或 Web App URL。
- 未執行 commit、push 或建立 PR。

## 10. 結論

Task 04H-2J 正式 LIFF 顧問大廳新版 UI 已完成驗收，Task 04H-2J-Fix1 亦完成顧問資源文字可讀性與返回 LINE 行為修正。會員首頁未受影響，未發現功能異常。

`liff.closeWindow()` 會返回 LIFF 原始開啟來源；從非官方聊天室進入時返回該聊天室屬預期行為。目前顧問獎金仍維持未開放，後續必須完成正式資料與真實查詢驗收，才可評估開啟功能。
