---
name: openclaw-linebot-deploy
description: 完整部署基於 OpenClaw 的 LINE Bot 服務。涵蓋 Docker 環境設定、LINE Developers Console 配置、Cloudflare Tunnel 固定網址、System Prompt 客製化、多種回覆格式（純文字、Flex Message、圖片）、圖片生成連續技、以及 Claude Code 協作開發。當用戶提到 OpenClaw、LINE Bot 部署、Cloudflare Tunnel、LINE 機器人架設、自架 AI 助理、或想把 OpenClaw 接上 LINE 時，請務必使用此 skill。也適用於已部署但需要排錯、升級回覆格式、或加入圖片生成功能的場景。
---

# OpenClaw LINE Bot 完整部署指南

將 OpenClaw AI 助理框架部署為可上線的 LINE Bot 服務，包含環境設定、Cloudflare 固定網址、多種回覆格式支援。

---

## 快速總覽

OpenClaw 是可自架的 AI 助理框架，核心能力是「通訊平台 + AI 模型 + 外掛」三層自由組合。本 skill 引導使用者從零到完成一個能在 LINE 上對話的 AI Bot。

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

### 設定 OpenRouter API Key

編輯 Shell 設定檔（Mac: `~/.zshrc`、Windows Git Bash: `~/.bashrc`）：

```bash
export OPENROUTER_API_KEY='貼上你的 OpenRouter API Key'
export ANTHROPIC_DEFAULT_SONNET_MODEL="openrouter/free"
export ANTHROPIC_DEFAULT_OPUS_MODEL="openrouter/free"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="openrouter/free"
```

讓設定生效：
```bash
# Mac
source ~/.zshrc
# Windows Git Bash
source ~/.bashrc
```

---

## Phase 2：啟動 OpenClaw 服務

確認 Docker Desktop 已啟動（綠色圖示），且終端機在 OpenClaw 專案資料夾內。

### 四行指令啟動

```bash
# 1) 設定 AI 模型
docker compose run --rm openclaw-cli models set openrouter/openrouter/free

# 2) 安裝 LINE 外掛
docker compose run --rm openclaw-cli plugins install extensions/line

# 3) 啟用 LINE 外掛
docker compose run --rm openclaw-cli plugins enable line

# 4) 啟動 Gateway
docker compose up -d openclaw-gateway
```

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
