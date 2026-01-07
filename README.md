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
'''
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
