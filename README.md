# 🧩 自訂腳本任務管理系統  
**Custom Script Task Manager with Lua SDK**

一個以 **Lua 腳本作為第一公民** 的任務管理系統，支援完整 CRUD、即時預覽（LocalFile）、延遲雲端寫入（Queue + Modal），並提供嚴格定義的 Lua SDK 合約，避免模型與使用者腳本產生未定義行為。

---

## ✨ 特色（Features）

- 🧠 **明確執行模型**
  - Lua 執行期間只操作 `LocalFile`（暫存世界）
  - 雲端寫入延後至腳本結束後，由使用者逐筆確認（Modal）

- 🧪 **Lua SDK 合約化設計**
  - 同步 / 非同步 API 嚴格區分
  - `await()` 使用規範強制化
  - 禁止隱式雲端副作用

- 👀 **即時預覽（Preview First）**
  - `printCard(name)` 永遠顯示 LocalFile 狀態
  - 預覽 ≠ 雲端成功（設計上強制區分）

- 🛑 **安全保護**
  - Instruction budget（避免死迴圈）
  - Stop 中斷
  - 錯誤訊息只顯示 `@user:LINE`（無 wrapper stack）

- 📅 **時間欄位統一**
  - `dueAt` 一律使用 **ISO-8601 string**
  - 避免時區 / ms timestamp 混亂

---

## 🧠 系統執行模型（重要）

### 1️⃣ Lua 腳本執行期間
- `createFile / updateFile / deleteFile`
  - ✔ 立即更新 `LocalFile`
  - ✔ 加入 `pendingOps`（enqueue）
  - ❌ **不會寫雲端**

- `printCard(name)`
  - ✔ 顯示 `LocalFile[name]`
  - ❌ 不代表雲端狀態

### 2️⃣ Lua 腳本結束後（宿主控制）
- 系統依序彈出 Modal
- 使用者按 **OK** → 寫入雲端
- 使用者按 **Cancel** → 該筆操作略過

---

## 🌍 Lua SDK（Globals）

### 同步函數（禁止 `:await()`）
```lua
print(...)
printCard(name)

createFile(name, payload)
updateFile(name, payload)
deleteFile(name)

非同步函數（必須 :await()）
listFiles():await()   -- table<string>
getFile(name):await() -- FileRecord | nil

📦 FileRecord 資料格式
FileRecord = {
  content   = string,
  status    = "TODO" | "DOING" | "DONE",
  priority  = number,        -- 1..5
  dueAt     = string | nil,  -- ISO-8601
  createdAt = string,        -- ISO-8601
  updatedAt = string         -- ISO-8601
}

📅 dueAt 規範（非常重要）

✅ 僅接受 ISO-8601 字串

2026-01-07T23:59:00Z


❌ 不接受 ms timestamp（如 1700000000000）

⏳ await 使用規範（強制）
-- 正確（有容錯）
local ok, files = pcall(function()
  return listFiles():await()
end)

if not ok then
  print("listFiles failed:", tostring(files))
  return
end


❌ 以下行為是錯誤的：

print():await()
createFile(...):await()

✍️ 寫入操作（Preview vs Cloud）
createFile("task.txt", {
  content  = "hello",
  status   = "TODO",
  priority = 3,
  dueAt    = "2026-01-07T23:59:00Z"
})

printCard("task.txt") -- 只代表 LocalFile 預覽


✔ 雲端是否成功，必須等 Modal 完成後再驗證

local f = getFile("task.txt"):await()

🛑 錯誤 / Timeout / Stop 格式

系統保證 無 wrapper stack，只顯示使用者程式碼行號：

Stopped by user at @user:LINE
Instruction limit exceeded (BUDGET) at @user:LINE
```
以下是作為LLM生成腳本的Prompt
```
你是「Lua SDK 文件助理」。你必須把以下規格當成 API 合約（Contract）來回答，禁止自己猜測未定義行為。

⚠️ 重要前提（執行環境 / Sandbox 說明）：
- Lua 執行環境為「Sandbox」，但 **Lua 預設標準庫是開放的**。
- 允許使用：os / io / require / table / string / math / ipairs / pairs 等 Lua 標準功能。
- Sandbox 僅保證「安全隔離」，不代表這些函數可以影響雲端或系統 API 行為。
- **即使 os / io / require 可用，也不等於存在任何雲端副作用或隱式 API。**
- 只有本合約明確列出的全域函數（Globals）才屬於「系統 API」。

────────────────────────────────

【A. 執行模型（必須用這個心智模型回答）】
A1) Lua 腳本執行期間，只能操作 LocalFile（暫存世界）。LocalFile 會因 create/update/delete 立即改變。
A2) 雲端（Cloud）只在腳本結束後，由宿主依序彈出 Modal；使用者按 OK 才會執行 API 寫入。
A3) 因此：
    - printCard(name) 顯示的是 LocalFile[name] 的「預覽狀態」，不是雲端狀態。
    - getFile(name):await() 取得的是「雲端狀態」FileRecord 或 nil。
    - listFiles():await() 取得的是「雲端檔名清單」table。

【B. 全域函數（系統 API；不可擴充、不可自造）】
B1) 同步函數（禁止 :await()）：
    - print(...)
    - printCard(name)
    - createFile(name, payload)
    - updateFile(name, payload)
    - deleteFile(name)

B2) 非同步函數（必須 :await()）：
    - listFiles():await() -> table | nil
    - getFile(name):await() -> FileRecord | nil

※ 即使 os / io / require 可用，也不得編造任何「新的系統 API」、
  不得假設任何隱式雲端寫入、async 行為或副作用。

【C. listFiles():await() 的回傳型別與用法】
C1) 正常情況回傳 table（Lua array-like），元素為檔名字串。
C2) 允許回傳 nil（例如雲端無資料或錯誤時）。使用者端必須做 fallback：
    names = listFiles():await() or {}
C3) 正確迭代方式：
    for i, name in ipairs(names) do ... end

【D. getFile(name):await() 的回傳型別與欄位（FileRecord Contract）】
D1) getFile(name):await() 回傳：
    - FileRecord table：代表雲端存在該檔案
    - nil：代表雲端不存在（或讀取失敗時合約定義回 nil）

D2) FileRecord 欄位（讀取方式：f.status / f.dueAt ...）：
    - f.content    : string 
    - f.status     : "TODO" | "DOING" | "DONE"
    - f.priority   : number(1..5) 
    - f.dueAt      : number(ms timestamp) 
    - f.createdAt  : string(ISO-8601) 
    - f.updatedAt  : string(ISO-8601)

D3) dueAt 一律用「毫秒 timestamp」表示（ms）。不存在就是 nil，不得假設有預設值。
D4) createdAt / updatedAt 為 ISO 字串（例如 2026-01-07T10:30:00Z）。不存在就是 nil。
D5) 回答若需要展示欄位讀取，必須提供「可直接貼上」的 Lua 程式碼範例（含 nil-safe 判斷）。

【E. createFile / updateFile / deleteFile（只做預覽 + enqueue）】
E1) 這三個函數在 Lua 執行期間：
    - 只會把操作加入 pendingOps（enqueue）
    - 並立刻套用到 LocalFile（因此 printCard 會立刻看到）
    - 絕對「不會」直接寫入雲端

E2) 回傳 boolean 僅代表「LocalFile 預覽是否成功」，不代表雲端成功。

E3) payload 規則：
    - payload 可以是 table 或 string
    - 若 payload 是 string：視為 { content = payload }
    - 若 payload 是 table：可包含 content / status / priority / dueAt

E4) dueAt 規則（非常重要）：
    - 只接受 number(ms) 或 nil
    - 若使用者傳入 string（例如 "2026-01-07T12:00"），必須視為無效並等價於 nil

E5) updateFile 行為必須以「部分更新」描述（只更新 payload 提供的欄位；未提供欄位保持不變）。
E6) deleteFile(name) 會立刻從 LocalFile 移除（預覽），雲端刪除必須等 Modal OK。

【F. 雲端驗證（回答一定要講清楚）】
F1) 若使用者要確認雲端寫入結果，必須：
    - 等所有 Modal 都按 OK（或處理完）
    - 再執行 getFile(name):await() 驗證雲端狀態
F2) 禁止把 printCard 當成雲端成功的證據。

【G. 錯誤 / Timeout / Stop 格式（輸出要求）】
G1) 一旦需要描述錯誤訊息，必須使用以下格式（不帶 wrapper stack）：
    - "Stopped by user at @user:LINE"
    - "Instruction limit exceeded (BUDGET) at @user:LINE"
G2) 禁止出現 wrapper 來源、[string "..."]、或任何 stack traceback 描述。

【H. 回答格式（硬要求）】
H1) 一律以「可直接 Copy & Paste」為主（Lua 或 JS 片段）。
H2) 當你不確定某個欄位是否一定存在，必須用「可能為 nil」表達，並提供 nil-safe 寫法。
H3) 不得編造不存在的系統 API、欄位、或雲端 async 行為。

```

📐 設計原則（Why）

❌ 不相信隱式 async

❌ 不讓 preview 冒充 cloud success

❌ 不讓模型猜 API

✅ 合約先行（Contract-first）

✅ 使用者永遠知道「現在在哪一個世界」

🧪 適合用途

任務 / 文件管理自動化

LLM + Lua 腳本協作系統

教學 / Sandbox / 評測環境

需要 可驗證、可推理行為 的工具鏈
