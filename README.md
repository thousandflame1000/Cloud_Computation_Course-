# 自訂腳本任務管理系統
Custom Script Task Manager with Lua SDK

這是一個 **以 UI / 宿主邏輯為核心** 的任務管理系統，  
並提供 **Lua 腳本作為受控的輔助工具**，用來進行批次操作、條件處理與自動化流程。

Lua **不是系統主控層**，也不會直接影響雲端狀態。  
所有雲端寫入都必須經由宿主確認後才會執行。

---

## 功能概述

- 基本任務 / 文件 CRUD
- Lua 腳本輔助操作（受限、合約化）
- 即時預覽（LocalFile）
- 延遲雲端寫入（Queue + Modal）
- 腳本執行限制（Stop / instruction budget）
- 明確的錯誤輸出格式（無 wrapper stack）

---

## 系統行為說明

### 1. Lua 腳本執行期間

Lua 腳本執行時：

- 只能操作 **LocalFile（預覽狀態）**
- 不會直接寫入雲端
- 所有寫入請求只會被加入佇列

#### 寫入相關 API 行為

- `createFile / updateFile / deleteFile`
  - 立即更新 LocalFile
  - 同時將操作加入 `pendingOps`
  - **不會寫入雲端**

- `printCard(name)`
  - 顯示 LocalFile 中的資料
  - 不代表雲端狀態

---

### 2. Lua 腳本結束後

當 Lua 腳本結束後：

- 系統會依序處理 `pendingOps`
- 每一筆操作都會顯示確認視窗（Modal）
- 使用者選擇：
  - **OK** → 寫入雲端
  - **Cancel** → 該筆操作不執行

---

## Lua SDK（可用全域函數）

### 同步函數

以下函數為同步執行，**不得使用 `:await()`**：

lua
print(...)
printCard(name)

createFile(name, payload)
updateFile(name, payload)
deleteFile(name)
非同步函數
以下函數會回傳 Promise-like userdata，必須使用 :await()：

lua
複製程式碼
listFiles():await()    -- table<string> | nil
getFile(name):await()  -- FileRecord | nil
await 使用規則
只有 listFiles() 與 getFile() 可以使用 :await()

對其他函數使用 :await() 沒有意義，屬於錯誤用法

正確用法
lua
複製程式碼
local ok, names = pcall(function()
  return listFiles():await()
end)

if not ok then
  print("listFiles failed:", tostring(names))
  return
end
錯誤用法
lua
複製程式碼
print():await()
createFile(...):await()
FileRecord 格式
lua
複製程式碼
FileRecord = {
  content   = string,
  status    = "TODO" | "DOING" | "DONE",
  priority  = number,        -- 1..5
  dueAt     = string,  -- ISO-8601
  createdAt = string,  -- ISO-8601
  updatedAt = string   -- ISO-8601
}

dueAt 規則
僅接受 ISO-8601 字串

範例：

text
複製程式碼
2026-01-07T23:59:00Z
不接受毫秒 timestamp（例如 1700000000000）

預覽與雲端驗證
建立或更新檔案
lua
複製程式碼
createFile("task.txt", {
  content  = "hello",
  status   = "TODO",
  priority = 3,
  dueAt    = "2026-01-07T23:59:00Z"
})

printCard("task.txt") -- 只代表 LocalFile 預覽
驗證雲端狀態（必須在 Modal 完成後）
lua
複製程式碼
local f = getFile("task.txt"):await()
錯誤與中斷行為
系統錯誤輸出只包含使用者腳本位置，不顯示 wrapper stack。

可能的錯誤格式：

text
複製程式碼
Stopped by user at @user:LINE
Instruction limit exceeded (BUDGET) at @user:LINE

#LLM 生成腳本用 Prompt
本系統提供專用 Prompt，用於限制 LLM 生成 Lua 腳本時的行為：
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
    - getFile(name):await() 取得的是「雲端狀態」FileRecord 。
    - listFiles():await() 取得的是「雲端檔名清單」table。

【B. 全域函數（系統 API；不可擴充、不可自造）】
B1) 同步函數（禁止 :await()）：
    - print(...)
    - printCard(name)
    - createFile(name, payload)
    - updateFile(name, payload)
    - deleteFile(name)

B2) 非同步函數（必須 :await()）：
    - listFiles():await() -> table 
    - getFile(name):await() -> FileRecord 

※ 即使 os / io / require 可用，也不得編造任何「新的系統 API」、
  不得假設任何隱式雲端寫入、async 行為或副作用。
  
【C. listFiles():await() 的回傳型別與用法】
C1) 正常情況回傳 table（Lua array-like），元素為檔名字串。
C2) ⚠️ 失敗時「不回傳 nil」，而是會直接 throw error（腳本會中斷）。
    因此使用者端若要容錯，必須用 pcall 包住 await：
    local ok, names = pcall(function() return listFiles():await() end)
    if not ok then
      print("listFiles failed:", tostring(names)) -- names 是錯誤訊息
      return
    end
C3) 正確迭代方式：
    for i, name in ipairs(names) do ... end

【D. getFile(name):await() 的回傳型別與欄位（FileRecord Contract）】
D1) getFile(name):await() 正常回傳：
    - FileRecord table：代表雲端存在該檔案
    - nil：代表雲端不存在（not found）
D2) ⚠️ 讀取失敗（例如網路/API 錯誤）「不回傳 nil」，而是會直接 throw error。
    因此需要容錯時，必須用 pcall 包住 await：
    local ok, f = pcall(function() return getFile(name):await() end)
    if not ok then
      print("getFile failed:", tostring(f)) -- f 是錯誤訊息
      return
    end
    -- ok == true 時，f 才可能是 FileRecord 或 nil（not found）
D3) FileRecord 欄位（讀取方式：f.status / f.dueAt ...）：
    - f.content    : string 
    - f.status     : "TODO" | "DOING" | "DONE"
    - f.priority   : number(1..5) 
    - f.dueAt      : string(ISO-8601) 
    - f.createdAt  : string(ISO-8601) 
    - f.updatedAt  : string(ISO-8601)
D4) dueAt 一律用「ISO-8601 字串」表示（例如 2026-01-07T23:59:00Z）
D5) createdAt / updatedAt 為 ISO 字串（例如 2026-01-07T10:30:00Z）。
D6) 回答若需要展示欄位讀取，必須提供「可直接貼上」的 Lua 程式碼範例（含 nil-safe 判斷 + pcall 容錯範例）。

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
    - 只接受 string(ISO-8601)，例如 "2026-01-07T23:59:00Z"
    - 若使用者傳入 number（例如 1700000000000）或其他格式，視為不符合合約（應提示使用者改成 ISO 字串）

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

設計取向
不嘗試自動同步 Local 與 Cloud

不假設預覽等於成功

不容許未定義 API 行為

所有狀態改變都必須可觀察、可中斷
