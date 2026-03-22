# 新手概念速查表

給完全沒接觸過 AI 開發的使用者。agent 可以在使用者提問時引用這些說明，不需要每次都從頭解釋。

---

## LLM 基礎

### Token

模型讀字的最小單位。模型不是一次讀整句，而是把文字切成小片段處理。

直覺：字數越多，token 越多。中文大約 1 個字 ≈ 1～2 tokens。

影響：速度與費用都和 token 數量有關。

### 上下文（Context）

模型這一刻「看得到」的工作記憶。包含系統提示詞、你前幾輪的提問、工具回傳結果。

關鍵限制：容量有限。對話越長，越早的訊息可能被擠掉。模型不是故意忘記，是真的看不到了。

類比：想像白板空間固定，寫太多後舊內容會被擦掉。

### 計費公式

```
費用 = (輸入 tokens × 輸入單價) + (輸出 tokens × 輸出單價)
```

- 輸入：你送進去的 prompt + 歷史上下文
- 輸出：模型回覆的內容
- 不同模型單價不同

省錢三招：問題更精準、限制回覆長度、先用便宜模型驗證。

---

## 模型選型

| 等級 | 代表模型 | 單次費用 | 適用場景 |
|------|---------|---------|---------|
| 免費/便宜 | OpenRouter free、Gemini Flash | ≈ $0 | 先跑通流程、大量回覆 |
| 中價位 | Claude 3.5 Sonnet、GPT-4o mini | $0.005～$0.015 | 日常客服、內容任務 |
| 高價位 | Claude Opus、GPT-4o | $0.02～$0.05+ | 高風險、必須正確的任務 |

建議策略：**預設用免費模型**，只有重要回覆才升級。這叫分層路由。

### 訂閱 ≠ API

Claude Pro/Max、ChatGPT Plus 的月費訂閱是「App 使用權」，不等於 API 可用額度。接 OpenClaw 長期穩定請走 API Key。

---

## 工具概念

### 終端機（Terminal）

用文字跟電腦說話的介面。你打指令 → Terminal 交給 Shell 翻譯 → 系統執行並回報。

- Windows：搜尋 **Git Bash**
- Mac：搜尋 **Terminal**

### Git

程式碼的版本管理工具。像 Google Drive，但專為程式碼設計。每次修改留記錄，出問題可回退。

課程中主要用 `git clone`（下載專案）和 `git restore .`（還原所有改動）。

### Node.js

JavaScript 的執行引擎。Claude Code 是用 JavaScript 寫的，Node.js 讓它跑起來。

你不需要直接操作 Node.js，它是底層執行環境。

### Docker（選用，不是每個人都要）

**一種**把軟體包在容器裡跑的方式，好處是隔離、環境一致。本課程裡：

- **macOS**：只有選「用 Docker 包 OpenClaw」這條路才需要裝 Docker Desktop。
- **Windows**：課程兩條路（本機或 WSL）**都不必**為了 OpenClaw 先裝 Docker。

若你**有**用 Docker 跑 OpenClaw，才需要開 Docker Desktop（綠色圖示）再下 `docker compose` 類指令。

### Claude Code

住在終端機裡的 AI 助理。你用中文描述需求，它幫你讀/寫/改程式。

工作流程：你描述需求 → Claude Code 讀專案改檔 → 你驗收結果。

### OpenRouter

AI 模型的轉運站。一個 API Key 就能切換不同廠商的模型，不被綁定。

部分模型有免費額度。429 錯誤 = 免費額度用太快，等幾秒就好。

---

## OpenClaw 核心概念

### 定位

可自架的 AI 助理框架。不是封閉成品，而是通訊平台 + AI 模型 + 外掛三層自由組合。

### 六大組件

| 組件 | 職責 |
|------|------|
| Gateway | 接收訊息、驗證來源、分流 |
| Agent | 思考與決策 |
| Memory | 保存對話脈絡 |
| Skills | 外部能力擴充 |
| Heartbeat | 健康檢查 |
| Cron | 定時任務排程 |

### 設計原則

1. **模組化**：功能拆成外掛，只裝需要的能力
2. **可容器化**：可用 Docker 跑，也可本機直裝；依你選的路線而定
3. **開放擴充**：模型與平台可替換，不綁單一供應商

---

## 常用指令速查

**以下 `docker compose` 列僅在「用 Docker 跑 OpenClaw」時適用；本機 `openclaw` 請用官方 CLI 對應指令。**

| 指令 | 用途 |
|------|------|
| `docker compose ps` | 看所有服務狀態 |
| `docker compose up -d openclaw-gateway` | 在背景啟動 Gateway |
| `docker compose restart openclaw-gateway` | 重啟 Gateway |
| `docker compose logs -f openclaw-gateway` | 即時看日誌（Ctrl+C 離開） |
| `docker compose run --rm openclaw-cli plugins list` | 列出已安裝外掛 |
| `docker compose run --rm openclaw-cli channels list` | 列出已設定頻道 |
| `docker compose run --rm openclaw-cli channels status --probe` | 全面健康檢查 |

### 讀指令的技巧

從右往左讀：先看「做什麼」（set / up / install），再看「用哪個工具」（docker compose / npm），最後看旗標（--rm / -d / -g）。
