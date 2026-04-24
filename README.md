# LC Tracker

個人 LeetCode 間隔複習追蹤器。純前端單頁應用，資料存於瀏覽器 localStorage，可選擇同步至 GitHub。

## 功能

- **間隔複習排程**：根據每次練習結果自動計算下次複習日期（可自訂各熟練度的間隔天數）
- **今日練習**：顯示今日到期題目、記錄嘗試結果（寫出來 / 沒寫出來），並列出今日已完成題目
- **題目列表**：新增、編輯、刪除題目；支援搜尋、狀態與難度篩選
- **熱度圖**：近 18 週練習頻率視覺化，以及題目標籤分布統計
- **GitHub 同步**：透過 Personal Access Token 將 `data.json` 存於指定 repo，支援跨裝置同步
- **匯入 / 匯出**：JSON 格式匯入（支援貼上或上傳檔案）與匯出

## 使用方式

直接用瀏覽器開啟 `index.html`，不需任何建置步驟。

```bash
open index.html
```

或部署至任何靜態托管服務（GitHub Pages、Netlify 等）。

## 熟練度狀態機

```
todo
 └─(首次完成)→ first_time_done
               ├─(成功)→ confident_1
               │          ├─(成功)→ confident_2
               │          │          ├─(成功)→ confident_3
               │          │          │          ├─(成功)→ confident_4 ✓ 已掌握
               │          │          │          └─(失敗)→ unfamiliar
               │          │          └─(失敗)→ unfamiliar
               │          └─(失敗)→ unfamiliar
               └─(失敗)→ unfamiliar
                          └─(成功)→ confident_1
```

**預設複習間隔：**

| 狀態 | 下次間隔 |
|---|---|
| first_time_done | +1 天 |
| unfamiliar | +1 天 |
| confident_1 | +5 天 |
| confident_2 | +10 天 |
| confident_3 | +21 天 |
| confident_4 | 不再排程 |

間隔可於 ⚙ 設定 → 複習間隔 自訂，設定存於 `data.json`。

## 資料格式

本機儲存於 `localStorage`（key: `lc_db_v4`），GitHub 同步時寫入指定 repo 的 JSON 檔案。

```json
{
  "problems": [
    {
      "id": "unique_id",
      "num": 1,
      "title": "Two Sum",
      "diff": "Easy",
      "status": "confident_1",
      "tags": ["Array", "Hash Map"],
      "note": "解題思路筆記",
      "nextDate": "2026-04-27",
      "attempts": 2,
      "createdAt": "2026-04-21T00:00:00.000Z",
      "updatedAt": "2026-04-22T00:00:00.000Z",
      "log": [
        { "date": "2026-04-21", "status": "first_time_done", "note": "", "success": true }
      ]
    }
  ],
  "daily": {
    "2026-04-24": { "done": ["id1", "id2"] }
  },
  "log": [
    { "date": "2026-04-24", "pid": "id1" }
  ],
  "intervals": {
    "first_time_done": 1,
    "unfamiliar": 1,
    "confident_1": 5,
    "confident_2": 10,
    "confident_3": 21
  }
}
```

## 匯入格式

可將 agent 規劃好的排程貼入匯入對話框。題號相同的題目會更新（保留歷史紀錄），新題號則新增：

```json
{
  "problems": [
    {
      "num": 643,
      "title": "Maximum Average Subarray I",
      "diff": "Easy",
      "tags": ["Array", "Sliding Window"],
      "note": "固定視窗基本型",
      "nextDate": "2026-04-25"
    }
  ]
}
```

## GitHub 同步設定

1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained
2. 建立 token，給予目標 repo 的 **Contents: Read and write** 權限
3. 在 LC Tracker ⚙ 設定填入 Token、帳號、Repo 名稱、Branch、資料檔案路徑

Token 僅存於瀏覽器 localStorage，不會傳送至任何第三方服務。

## 鍵盤快捷鍵

| 快捷鍵 | 功能 |
|---|---|
| `Cmd/Ctrl + N` | 新增題目 |
| `Cmd/Ctrl + S` | 手動同步 GitHub |
| `Esc` | 關閉所有對話框 |
