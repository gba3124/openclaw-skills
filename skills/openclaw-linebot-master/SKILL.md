---
name: openclaw-linebot-deploy
description: 完整部署基於 OpenClaw 的 LINE Bot 服務。涵蓋 Docker 環境設定、LINE Developers Console 配置、Cloudflare Tunnel 固定網址、System Prompt 客製化、多種回覆格式（純文字、Flex Message、圖片）、圖片生成連續技、以及 Claude Code 協作開發。當用戶提到 OpenClaw、LINE Bot 部署、Cloudflare Tunnel、LINE 機器人架設、自架 AI 助理、或想把 OpenClaw 接上 LINE 時，請務必使用此 skill。也適用於已部署但需要排錯、升級回覆格式、或加入圖片生成功能的場景。
---

# OpenClaw LINE Bot 完整部署指南

將 OpenClaw AI 助理框架部署為可上線的 LINE Bot 服務，包含環境設定、Cloudflare 固定網址、多種回覆格式支援。

**新手提示**：如果使用者是完全零基礎，建議先使用 `openclaw-course-guide` skill 完成帳號註冊與工具安裝（Step ①～④），再回到本 skill 的 Phase 2 開始部署。

---

## 快速總覽

OpenClaw 是可自架的 AI 助理框架，核心能力是「通訊平台 + AI 模型 + 外掛」三層自由組合。本 skill 引導使用者從環境就緒到完成一個能在 LINE 上對話的 AI Bot。

### 先校正：哪些說法是錯的

以下是常見錯誤說法，教學時請直接更正：

- 「把 SKILL.md 移到 `/app/skills/` 根目錄才會生效」→ **錯**
- 「裝在 `/app/skills/openclaw-skills/skills/...` 就會自動載入」→ **錯**

正確做法：

- `~/.openclaw/workspace/skills/<skill-name>/SKILL.md`
- 或 `~/.openclaw/skills/<skill-name>/SKILL.md`

另外，若遇到「只會聊天不會做事」，優先檢查：
- `tools.profile=full`
- `tools.sessions.visibility=all`

### 架構圖

```
通訊入口              OpenClaw              AI 後端
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ 📱 LINE   │     │ 🦞 Gateway    │     │ 🤖 Claude │
│ ✈️ Telegram│ ──→ │  路由/權限/日誌 │ ──→ │ ⚡ GPT    │
│ 🌐 Web    │     │  外掛管理      │     │ 🆓 Free   │
└──────────┘     └──────────────┘     └──────────┘
                       ↑
                 Cloudflare Tunnel
                 (提供 HTTPS 固定網址)
```

### 完整部署流程

```
Phase 1: 環境準備          Phase 2: 啟動服務          Phase 3: 對外連線
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│ 安裝 Git/Node/ │     │ 設定 AI 模型    │     │ Cloudflare     │
│ Docker Desktop │ ──→ │ 安裝 LINE 外掛  │ ──→ │ Tunnel 打通    │
│ Clone OpenClaw │     │ 綁定 Token      │     │ Webhook 驗證   │
└────────────────┘     └────────────────┘     └────────────────┘
                                                      │
Phase 6: 進階功能          Phase 5: 回覆升級          Phase 4: 驗收
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│ 圖片生成連續技  │     │ Flex Message   │     │ LINE 傳訊息    │
│ Gemini+matplotlib│ ←── │ Carousel 卡片  │ ←── │ Bot 有回覆     │
│ 知識卡片系統    │     │ 導航按鈕       │     │ 流程跑通 🎉    │
└────────────────┘     └────────────────┘     └────────────────┘
```

---

## Phase 1：環境準備

### 前置條件

| 項目 | 說明 |
|------|------|
| Git | 下載 OpenClaw 專案原始碼 |
| Node.js | 執行 Claude Code（選用） |
| Docker Desktop | 執行 OpenClaw 容器（必要） |
| LINE Developers 帳號 | 建立 Messaging API Channel |
| OpenRouter 帳號 | 取得 AI 模型 API Key |
| Cloudflare 帳號 | 建立固定 Tunnel（Phase 3） |

### 安裝工具

**Windows（PowerShell 系統管理員）**：
```bash
winget install --id Git.Git -e --source winget
winget install OpenJS.NodeJS
winget install Docker.DockerDesktop
```

**Mac（Terminal）**：
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git node
brew install --cask docker
```

### 驗證安裝

```bash
git --version
node -v
docker --version
```

三行都有版本號才算通過。若 `docker --version` 有值但後面連不上，通常是 Docker Desktop 尚未啟動。

### Clone OpenClaw 專案

```bash
git clone <講師提供的 OpenClaw 專案網址>
cd <專案資料夾名稱>
```

agent 注意：如果使用者不知道專案網址，請詢問講師或課程提供的連結。切勿猜測。

### 設定 OpenRouter API Key

編輯 Shell 設定檔（Mac: `~/.zshrc`、Windows Git Bash: `~/.bashrc`）：

```bash
export OPENROUTER_API_KEY='貼上你的 OpenRouter API Key'
export ANTHROPIC_DEFAULT_SONNET_MODEL="openrouter/free"
export ANTHROPIC_DEFAULT_OPUS_MODEL="openrouter/free"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="openrouter/free"

# 備用：免費模型太慢時取消下行註解
# export ANTHROPIC_DEFAULT_SONNET_MODEL="moonshotai/kimi-k2.5"
```

選配：防 429 自動重試函式（貼在設定檔最底部）：
```bash
orfree() {
  for i in {1..5}; do
    echo "第 $i 次嘗試..."
    claude "$@" && break
    sleep 3
  done
}
```

讓設定生效：
```bash
# Mac
source ~/.zshrc
# Windows Git Bash
source ~/.bashrc
```

驗證設定：`echo $OPENROUTER_API_KEY` 有輸出值即通過。

### 學員「terminal 已有 key」的標準做法（必教）

很多學員會在 terminal 已經 `export OPENROUTER_API_KEY=...`，但 `.env` 仍是空值，導致重開機後失效。建議一律同步到專案 `.env`。

```bash
# 1) 確認目前 terminal 有值
printenv OPENROUTER_API_KEY

# 2) 開啟 openclaw/.env，填入這行（不要留空）
# OPENROUTER_API_KEY=sk-or-v1-...

# 3) 重建服務讓容器吃到新環境變數
docker compose up -d --force-recreate openclaw-gateway openclaw-office

# 4) 驗證生效
docker compose run --rm openclaw-cli models
```

驗收標準：
- `models` 輸出內 `Auth overview` 看到 openrouter 的 `source=env: OPENROUTER_API_KEY`
- Gateway logs 顯示 `agent model: openrouter/anthropic/claude-sonnet-4.5`

---

## Phase 2：啟動 OpenClaw 服務

**關鍵前置條件**：確認 Docker Desktop 已啟動（圖示為綠色），且終端機在 OpenClaw 專案資料夾內。如果 Docker Desktop 沒開，所有 `docker compose` 指令都會報 `Cannot connect to the Docker daemon` 錯誤。

### 先配置 OpenRouter 六模型組（建議）

```bash
# Primary
docker compose run --rm openclaw-cli models set openrouter/anthropic/claude-sonnet-4.5

# Fallbacks
docker compose run --rm openclaw-cli models fallbacks clear
docker compose run --rm openclaw-cli models fallbacks add openrouter/moonshotai/kimi-k2.5
docker compose run --rm openclaw-cli models fallbacks add openrouter/deepseek/deepseek-chat
docker compose run --rm openclaw-cli models fallbacks add openrouter/openai/gpt-4.1
docker compose run --rm openclaw-cli models fallbacks add openrouter/openrouter/auto
docker compose run --rm openclaw-cli models fallbacks add openrouter/openrouter/free
```

### 三行指令啟動

```bash
# 1) 安裝 LINE 外掛
docker compose run --rm openclaw-cli plugins install extensions/line

# 2) 啟用 LINE 外掛
docker compose run --rm openclaw-cli plugins enable line

# 3) 啟動 Gateway
docker compose up -d openclaw-gateway
```

### 避免 `No API key found for provider "anthropic"`（必教）

這是教學最常見卡點之一。根因通常是：
- 預設模型仍指向 Anthropic
- 或 OpenRouter API key 沒有注入到容器環境

建議在課程中固定加上這段檢查：

```bash
# 1) 檢查模型與認證狀態
docker compose run --rm openclaw-cli models
```

判斷標準：
- `Default` 應顯示 `openrouter/anthropic/claude-sonnet-4.5`
- `Auth overview` 需看到 openrouter 有有效來源（env 或 auth profiles）

建議 fallback 組（都走 OpenRouter）：
- `openrouter/moonshotai/kimi-k2.5`
- `openrouter/deepseek/deepseek-chat`
- `openrouter/openai/gpt-4.1`
- `openrouter/openrouter/auto`
- `openrouter/openrouter/free`

若還是報 Anthropic 無金鑰，先確認：
1. `OPENROUTER_API_KEY` 已設定
2. `docker-compose.yml` 的 `openclaw-gateway` 與 `openclaw-cli` 都有傳入 `OPENROUTER_API_KEY`
3. `docker compose up -d --force-recreate openclaw-gateway openclaw-office` 後再重試

### 2026.3.2+ 必做：工具權限改為 full

若學員反映「龍蝦只會回話、其他工具像壞掉」，先做這段：

```bash
docker compose run --rm openclaw-cli config set tools.profile full
docker compose run --rm openclaw-cli config set tools.sessions.visibility all
docker compose restart openclaw-gateway
```

驗收：
```bash
docker compose run --rm openclaw-cli config get tools.profile
docker compose run --rm openclaw-cli config get tools.sessions.visibility
```
應分別輸出 `full` 與 `all`。

### API 費用控制（7 招實作版）

以下 7 招可直接套用在 OpenClaw + OpenRouter：

1) **模型分層（最大降幅）**
- Primary 用高品質模型：`openrouter/anthropic/claude-sonnet-4.5`
- Fallback 混用：`kimi-k2.5`、`deepseek-chat`、`gpt-4.1`、`auto`、`free`
- 核心原則：簡單任務走便宜路由，複雜任務才走高成本模型

2) **限制上下文長度**
- 不要無限帶完整歷史，保留最近 5-10 則對話即可
- 話題切換時開新 session，避免無關歷史持續計費

3) **FAQ 快取**
- 高頻問題（如運費、退貨、地址）先建立標準回覆
- 相似度達門檻時直接回快取，不呼叫模型

4) **本地模型分流（Ollama）**
- 分類、擷取、格式化等低風險任務優先走本地模型
- 雲端模型只留給高品質生成與複雜推理

5) **Prompt 精簡**
- 刪除重複規則、合併語氣指令、用短範例取代長篇描述
- System prompt 每次都會重送，越短越省

6) **用量上限**
- 在 API 供應商後台設定每日/月度上限與 50%/80% 告警
- 開發與正式環境分開 key，避免測試燒掉正式預算

7) **批次處理**
- 可併的任務（分類、審核、翻譯）合併成一次請求
- 降低重複載入 system prompt 的成本

落地順序（課堂建議）：
1. 先設用量上限  
2. 再做模型分層  
3. 接著上 FAQ 快取  
4. 最後做 prompt 精簡與批次化

### 綁定 LINE Channel

前往 [LINE Developers Console](https://developers.line.biz/console/)，取得：
- **Channel Access Token**（Messaging API → Issue）
- **Channel Secret**（Basic settings）

```bash
docker compose run --rm \
  -e LINE_CHANNEL_ACCESS_TOKEN='貼上你的 Token' \
  -e LINE_CHANNEL_SECRET='貼上你的 Secret' \
  openclaw-cli channels add --channel line --use-env
```

---

## Phase 3：Cloudflare Tunnel 對外連線

LINE 必須能從外網打到你的 OpenClaw Gateway。Cloudflare Tunnel 是最穩定的方案。

有兩種模式，詳細設定請閱讀 `references/cloudflare-setup.md`：

| 模式 | 適用場景 | 網址穩定性 |
|------|---------|-----------|
| Quick Tunnel | 開發測試、課堂練習 | 每次重開都會變 |
| 固定 Tunnel | 正式上線、Demo Day | 永久固定網址 |

### Quick Tunnel（快速開始）

```bash
# Mac / Windows (Docker Desktop)
docker run -it --rm cloudflare/cloudflared:latest \
  tunnel --url http://host.docker.internal:8080

# Linux / VPS
docker run -it --rm --network host cloudflare/cloudflared:latest \
  tunnel --url http://localhost:8080
```

找到 `https://xxxx.trycloudflare.com`，Webhook URL 為：
```
https://xxxx.trycloudflare.com/webhook/line
```

### 設定 LINE Webhook

1. 到 [LINE Developers Console](https://developers.line.biz/console/) → 你的 Channel → **Messaging API**
2. 貼上 Webhook URL（`https://你的網域/webhook/line`）
3. 按 **Verify**，看到成功
4. 打開 **Use webhook**
5. 關閉 **Auto-reply messages**（避免重複回覆）

---

## Phase 4：驗收

```bash
docker compose run --rm openclaw-cli plugins list
docker compose run --rm openclaw-cli channels list
docker compose run --rm openclaw-cli channels status --probe
```

用你的 LINE 傳一句話，Bot 有回覆就代表部署成功。

---

## Phase 5：客製化 Bot

### 修改 System Prompt

使用 Claude Code 是最快的方式：

```bash
cd <openclaw-project-directory>
claude
```

貼這段給 Claude Code：
```
請幫我找到這個 Openclaw 專案裡設定系統提示詞（System Prompt）的位置。
把 LINE 機器人改成「<你想要的角色>」。
改完請告訴我需要重啟哪個服務，並給我指令。
```

改完重啟：
```bash
docker compose restart openclaw-gateway
```

### 多種回覆格式

OpenClaw 的 LINE 外掛支援多種訊息格式。詳細規格與範例請閱讀 `references/line-response-formats.md`。

| 格式 | 適用場景 | 複雜度 |
|------|---------|--------|
| 純文字 | 一般對話、Q&A | 低 |
| Flex Message | 卡片式資訊、導航按鈕 | 中 |
| Carousel | 多張卡片橫向滑動 | 中高 |
| 圖片訊息 | 圖表、知識卡片 | 中 |
| 圖片 + Flex 混合 | 教育類筆記、題庫 | 高 |

---

## Phase 6：進階功能

### 圖片生成連續技

結合 Gemini AI 生成底圖 + matplotlib 加中文標籤，專為 LINE Bot 設計。

詳細流程請閱讀 `references/image-generation.md`。

核心概念：
1. **Gemini 生成無標籤底圖**（避免中文亂碼）
2. **matplotlib 加上中文標籤**（確保正確顯示）
3. **壓縮到 < 300KB**（符合 LINE Bot 限制）

### 知識卡片系統

利用 Flex Message Carousel 建立知識卡片導航系統：

```
主選單
    ├── 分類目錄（Carousel）
    │   ├── 類別 A → [項目1] [項目2] ...
    │   ├── 類別 B → [項目3] [項目4] ...
    │   └── ...
    └── 項目詳情（多卡片 Carousel）
        ├── 卡片1: 基本資訊
        ├── 卡片2: 詳細內容
        └── 卡片3: 導航按鈕
```

---

## 排錯指南

常見問題的快速解法請閱讀 `references/troubleshooting.md`。

### 三大根因速查

| 症狀 | 最可能原因 | 第一步 |
|------|-----------|--------|
| Verify 失敗 | 網址不通 | 確認 Tunnel 在跑、URL 正確 |
| 只有已讀無回覆 | 服務沒跑 | `docker compose ps` 確認 Up |
| 401 / 403 | 金鑰錯 | 重新 Issue Token 再綁定 |

### 急救三步驟

```bash
# 1) 看 Gateway 狀態
docker compose ps

# 2) 看即時錯誤
docker compose logs -f openclaw-gateway

# 3) 快速重啟
docker compose restart openclaw-gateway
```

---

## 每日開機儀式（Quick Tunnel 模式）

使用 Quick Tunnel 時，每天開機要重做這 4 步：

1. 確認 `docker compose ps` 看到 `openclaw-gateway` 是 Up
2. 重跑 Quick Tunnel 指令，拿到新的 `https://xxxx.trycloudflare.com`
3. 到 LINE Developers 更新 Webhook URL（加上 `/webhook/line`）並按 Verify
4. 用自己的 LINE 傳一則訊息，確認有回覆再繼續開發

升級到固定 Tunnel 後就不需要每天重做，詳見 `references/cloudflare-setup.md`。

---

## 相關 References

| 檔案 | 內容 |
|------|------|
| `references/cloudflare-setup.md` | Cloudflare Quick Tunnel 與固定 Tunnel 完整設定 |
| `references/line-response-formats.md` | LINE 所有回覆格式規格、Flex Message 設計規範 |
| `references/image-generation.md` | Gemini + matplotlib 圖片生成連續技 |
| `references/troubleshooting.md` | 完整排錯指南（SSL、Webhook、API 錯誤） |

## 相關 Skills

| Skill | 用途 |
|-------|------|
| `openclaw-course-guide` | 零基礎新手從帳號註冊到工具安裝的課程導覽，完成後銜接本 skill |
