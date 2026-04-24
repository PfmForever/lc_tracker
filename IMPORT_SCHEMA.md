# LC Tracker 匯入 JSON 規格

給 LeetCode training agent 的格式說明。Agent 依你目前的學習狀態輸出符合此格式的 JSON，丟到 app 的「⬆ 匯入排程」即可。

---

## 頂層結構

```json
{
  "problems": [ /* 題目陣列，必填 */ ],
  "daily":    { /* 已完成紀錄，選填，通常只有 app 自己用 */ }
}
```

- 最少只需要 `problems`。`daily` 通常由 app 管，agent 不需要產。
- 也可以直接傳純陣列：`[{...}, {...}]`，等同 `{"problems":[...]}`。

---

## Problem 物件

```json
{
  "num":       1,                    // 必填，LeetCode 題號（整數）
  "title":     "Two Sum",            // 建議填
  "diff":      "Easy",               // "Easy" | "Medium" | "Hard"
  "status":    "todo",               // 見下方狀態機
  "tags":      ["Array", "Hash Map"],
  "note":      "解題思路或提醒",
  "nextDate":  "2026-04-25",         // YYYY-MM-DD，agent 排程的下次練習日
  "attempts":  0,                    // 通常新題填 0
  "createdAt": "2026-04-22T00:00:00.000Z",  // 可省略，會自動補 now
  "log":       []                    // 嘗試歷史，新題留空陣列
}
```

### 欄位說明

| 欄位        | 型別     | 必填 | 說明 |
|------------|----------|------|------|
| `num`      | int      | ✓    | 題號，匹配用的主 key（見下方「匹配規則」）|
| `title`    | string   |      | 題目英文名，app 會用來產 LeetCode 連結（slugify 後）|
| `diff`     | enum     |      | `Easy` / `Medium` / `Hard`，預設 `Medium` |
| `status`   | enum     |      | 見下方狀態機，預設 `todo` |
| `tags`     | string[] |      | 類型標籤 |
| `note`     | string   |      | 筆記（可放提示、易錯點）|
| `nextDate` | string   |      | `YYYY-MM-DD`，**agent 規劃的重點欄位** |
| `attempts` | int      |      | 嘗試次數，新題填 0 |
| `id`       | string   |      | 內部 ID，agent **不要產生**，讓 app 自己配。更新既有題目時可帶回原本的 id |
| `log`      | array    |      | 嘗試歷史，新題留 `[]` |

---

## 狀態機 (status)

狀態反映「掌握程度」，同時決定預設複習間隔（若未明確指定 `nextDate`，app 會用此間隔計算）：

| status           | label      | 語意                    | 預設間隔 |
|------------------|-----------|-------------------------|---------|
| `todo`           | 待寫       | 還沒寫過                | 無       |
| `first_time_done`| 初寫       | 第一次寫完，待隔天驗收   | +1 天    |
| `unfamiliar`     | 不熟       | 寫過但還卡住            | +1 天    |
| `confident_1`    | 熟練 L1    | 寫對一次                | +5 天    |
| `confident_2`    | 熟練 L2    | 寫對兩次                | +10 天   |
| `confident_3`    | 熟練 L3    | 寫對三次                | +21 天   |
| `confident_4`    | 已掌握 ✓   | 畢業，不自動排程        | 無       |

> **間隔可自訂**：⚙ 設定 → 複習間隔，修改後存於 `data.json` 的 `intervals` 欄位，並優先於上表預設值。Agent 規劃時建議先讀取 `data.json` 裡的 `intervals` 物件以確保計算正確。

### 狀態轉移（app 自己做，agent 規劃時參考）

```
todo ─── 寫了 ──▶ first_time_done
any ─── 寫出來 ──▶ 下一階 confident_n+1
any ─── 沒寫出來 ─▶ unfamiliar
```

---

## 匹配規則（既有題目 vs 新題）

App 匯入時對每個 entry：

1. 若 entry 有 `id` 且 DB 有同 id → **更新**
2. 否則用 `num` 找 → 有同 num 就 **更新**，沒有就 **新增**

**更新時 app 會保留的欄位（agent 不要覆蓋）：**
- `id`、`createdAt`、`attempts`、`log`

**更新時會被覆蓋的欄位（agent 可改）：**
- `title`、`diff`、`status`、`tags`、`note`、`nextDate`

所以 agent 要重新規劃排程時，**只需要傳 `num` + `nextDate`（+ 需要改的狀態）** 就好，不必也不該重傳 log / attempts。

---

## Agent Prompt 範本

```
You are my LeetCode training planner. I will give you my current tracker data (JSON) and you will design the next batch of problems + a daily schedule, then output a JSON I can import directly.

## My Tracker Data Schema

The data.json has this top-level structure:
  - problems[]: all tracked problems
  - daily{}: { "YYYY-MM-DD": { "done": [id, ...] } }
  - log[]: { "date": "YYYY-MM-DD", "pid": id }
  - intervals{}: review spacing in days — READ THIS before calculating nextDate
    e.g. { "first_time_done": 1, "unfamiliar": 1, "confident_1": 5, "confident_2": 10, "confident_3": 21 }

Status machine:
  todo → (first attempt) → first_time_done
  first_time_done → (success) → confident_1  |  (fail) → unfamiliar
  unfamiliar      → (success) → confident_1  |  (fail) → unfamiliar
  confident_1     → (success) → confident_2  |  (fail) → unfamiliar
  confident_2     → (success) → confident_3  |  (fail) → unfamiliar
  confident_3     → (success) → confident_4  |  (fail) → unfamiliar
  confident_4 = mastered, no more scheduling needed

nextDate = today + intervals[newStatus] after each attempt.

## My Current Data

[貼上 data.json]

## What I need from you

### 1. Analysis (brief)
- Topics covered and mastery level
- Weak spots (unfamiliar / repeated failures)
- Current daily load vs. capacity

### 2. New problems to add
Design the next 1–2 weeks. Rules:
- Build on existing topics progressively
- Easy warm-up → Medium(s) → 1 Hard per week
- Include: num, title, diff, tags, note (key insight), nextDate (first attempt day)
- Do NOT include id, status, attempts, log for new problems

### 3. Schedule adjustments for existing problems
Only include problems where you're actually changing nextDate (bring id for existing ones).

### 4. Output — single importable JSON
{
  "problems": [
    // new problems (no id) + existing problems with changed nextDate (include their id)
  ]
}
Do NOT re-output the full problem list — only new entries and changed entries.
```

---

## 範例

### 只改排程（常見用法）

```json
{
  "problems": [
    { "num": 1,   "nextDate": "2026-04-23" },
    { "num": 15,  "nextDate": "2026-04-24" },
    { "num": 200, "nextDate": "2026-04-28" }
  ]
}
```

### 新增題目 + 首刷排程

```json
{
  "problems": [
    {
      "num": 322,
      "title": "Coin Change",
      "diff": "Medium",
      "status": "todo",
      "tags": ["DP", "BFS"],
      "note": "經典 DP，注意 amount=0 edge case",
      "nextDate": "2026-04-25"
    },
    {
      "num": 300,
      "title": "Longest Increasing Subsequence",
      "diff": "Medium",
      "status": "todo",
      "tags": ["DP", "Binary Search"],
      "nextDate": "2026-04-27"
    }
  ]
}
```

### 綜合（新增 + 更新狀態/排程）

```json
{
  "problems": [
    { "num": 1,   "status": "confident_2", "nextDate": "2026-05-05" },
    { "num": 15,  "status": "unfamiliar",  "nextDate": "2026-04-23" },
    { "num": 322, "title": "Coin Change", "diff": "Medium", "status": "todo", "tags": ["DP"], "nextDate": "2026-04-25" }
  ]
}
```

---

## 注意事項

- `nextDate` 一定用 `YYYY-MM-DD`（ISO 日期），不要用時區時間。
- 日期比較是字串比較，所以格式務必嚴格。
- Agent 不必也不應該產生 `id`；留空讓 app 用既有的或自動生成。
- 不要塞 `log` 或覆蓋 `attempts`，那是使用者的歷史，app 會自動保留。
- `status` 若填不在上表的值，會被忽略（保留原狀態或預設 `todo`）。
