---
name: openclaw-course-guide
description: 引導零基礎小白從一台空電腦開始，按課程順序完成帳號註冊、工具安裝、OpenClaw 架設、到 LINE Bot 上線的完整流程。當用戶提到「課程」、「從零開始」、「新手教學」、「小白安裝」、「第一次架 OpenClaw」、「按步驟裝」時使用此 skill。完成基礎建設後會銜接 openclaw-linebot-master skill 進入部署與進階功能。
---

# OpenClaw 課程導覽：從零到 LINE AI Bot

依照「創聚 × OpenClaw：AI 原型開發夜」課程順序，帶完全沒經驗的使用者完成整條路線。

---

## 全局路線圖

```
① 帳號註冊 → ② 工具安裝 → ③ 專案下載 → ④ 環境設定 → ⑤ 啟動服務 → ⑥ LINE Bot 上線 → ⑦ 固定 Tunnel
  (10 min)    (10 min)    (3 min)     (5 min)    (5 min)     (10 min)     (10 min)
```

每一步都有「做完才能往下」的驗收條件。agent 必須在每步驗收通過後才帶使用者進入下一步。

---

## 核心觀念（先給使用者校準）

開始動手前，用 30 秒跟使用者對齊三件事：

1. **程式碼是工具不是目的** — 今天不需要懂每行程式，只要會用工具把想法做出來
2. **先求有再求好** — 先讓 Bot 活著，再優化細節
3. **每步都驗收** — 做完一步就測，不要囤到最後才測

如果使用者想深入了解 LLM 基礎（Token、上下文、計費），請閱讀 `references/concept-glossary.md`。

---

## Step ①：帳號註冊

在安裝任何工具之前，先確保以下帳號都已註冊。

詳細圖文指引請閱讀 `references/prerequisite-accounts.md`。

### 必要帳號清單

| 帳號 | 用途 | 註冊連結 |
|------|------|----------|
| GitHub | 下載 OpenClaw 原始碼 | https://github.com/signup |
| Anthropic | Claude Code 登入授權 | https://console.anthropic.com/ |
| OpenRouter | 取得 AI 模型 API Key | https://openrouter.ai/keys |
| LINE Developers | 建立 Messaging API Channel | https://developers.line.biz/console/ |
| Cloudflare | 建立 Tunnel 對外連線 | https://dash.cloudflare.com/sign-up |

### 網域（固定 Tunnel 必備）

固定 Cloudflare Tunnel 需要一個你控制的網域，並將 DNS 託管到 Cloudflare。

沒有網域的人：
- 可先在 [Namecheap](https://www.namecheap.com/)、[Cloudflare Registrar](https://www.cloudflare.com/products/registrar/) 等購買便宜網域（`.com` 約 $10/年，`.xyz` 約 $1/年）
- 買完後到 Cloudflare Dashboard 新增網站，依指示將 DNS Nameserver 指向 Cloudflare
- 如果暫時不想買域名，Step ⑥ 可先用 Quick Tunnel 跑通流程，Step ⑦ 再升級固定 Tunnel

### 驗收條件

- [ ] 五個帳號都能登入
- [ ] OpenRouter 已建立 API Key（先複製保存）
- [ ] LINE Developers 已建立 Messaging API Channel
- [ ] （選配）已有一個網域在 Cloudflare DNS 管理

---

## Step ②：安裝工具

三個基礎工具：Git、Node.js、Docker Desktop。

### Windows（PowerShell 系統管理員）

```bash
winget install --id Git.Git -e --source winget
winget install OpenJS.NodeJS
winget install Docker.DockerDesktop
```

安裝完畢後**重新開機**，後續操作統一改用 **Git Bash**。

### Mac（Terminal）

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install git node
brew install --cask docker
```

### 驗收條件

三行都必須有版本號：

```bash
git --version
node -v
docker --version
```

如果 `docker --version` 有值但後續指令報錯，通常是 Docker Desktop 尚未啟動。
請先打開 Docker Desktop 應用程式，確認圖示變綠色再繼續。

---

## Step ③：下載專案 + 安裝 Claude Code

### 3-1：Clone OpenClaw 專案

agent 必須先確認使用者的 OpenClaw 專案來源。按優先順序詢問：

1. **講師有提供專案網址？** → 直接 `git clone <那個網址>`
2. **使用 OpenClaw 官方 repo？** → `git clone https://github.com/open-claw/openclaw.git`
3. **不確定？** → 請使用者先確認來源再繼續，不要猜測

```bash
git clone <確認後的專案網址>
cd <專案資料夾名稱>
```

clone 完後用 `ls` 確認資料夾內有 `docker-compose.yml`，這是 OpenClaw 專案的標誌檔案。如果沒有，代表 clone 的不是 OpenClaw 本體。

### 3-2：安裝 Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

### 3-3：Claude Code 登入

```bash
claude
# 跟著畫面完成瀏覽器授權
exit
```

### 驗收條件

```bash
claude --version
```

有版本號即通過。

---

## Step ④：環境設定（Shell 設定檔 + API Key）

### 4-1：打開設定檔

**Windows（Git Bash）**：
```bash
start notepad ~/.bashrc
```

**Mac**：
```bash
touch ~/.zshrc
open -e ~/.zshrc
```

### 4-2：貼入以下設定

```bash
# OpenRouter API 金鑰
export OPENROUTER_API_KEY='貼上你的 OpenRouter API Key'

# 綁定 OpenRouter 免費模型
export ANTHROPIC_DEFAULT_SONNET_MODEL="openrouter/free"
export ANTHROPIC_DEFAULT_OPUS_MODEL="openrouter/free"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="openrouter/free"

# 備用：免費模型太慢時取消下行註解
# export ANTHROPIC_DEFAULT_SONNET_MODEL="moonshotai/kimi-k2.5"

# 防 429 自動重試函式
orfree() {
  for i in {1..5}; do
    echo "第 $i 次嘗試..."
    claude "$@" && break
    sleep 3
  done
}
```

存檔後關閉編輯器。

### 4-3：讓設定生效

```bash
# Mac
source ~/.zshrc
# Windows Git Bash
source ~/.bashrc
```

### 驗收條件

```bash
echo $OPENROUTER_API_KEY
```

有輸出你的 Key 值即通過。如果空白，代表 source 沒成功或設定檔路徑不對。

---

## Step ⑤：啟動 OpenClaw 服務

確認 Docker Desktop 已啟動（綠色圖示），且終端機在 OpenClaw 專案資料夾內。

### 四行指令，依序執行

```bash
docker compose run --rm openclaw-cli models set openrouter/openrouter/free
docker compose run --rm openclaw-cli plugins install extensions/line
docker compose run --rm openclaw-cli plugins enable line
docker compose up -d openclaw-gateway
```

每行跑完才貼下一行。最後一行會在背景啟動 Gateway。

### 綁定 LINE Channel

到 [LINE Developers Console](https://developers.line.biz/console/) 取得兩把鑰匙：
- **Channel Access Token**：Messaging API → Issue
- **Channel Secret**：Basic settings

```bash
docker compose run --rm \
  -e LINE_CHANNEL_ACCESS_TOKEN='貼上你的 Token' \
  -e LINE_CHANNEL_SECRET='貼上你的 Secret' \
  openclaw-cli channels add --channel line --use-env
```

### 驗收條件

```bash
docker compose ps
```

看到 `openclaw-gateway` 狀態為 Up 即通過。

---

## Step ⑥：LINE Bot 上線（Quick Tunnel 先跑通）

先用 Quick Tunnel 快速打通外網，確認整條路線可用，再到 Step ⑦ 升級固定網址。

### 6-1：啟動 Quick Tunnel

```bash
# Mac / Windows (Docker Desktop)
docker run -it --rm cloudflare/cloudflared:latest \
  tunnel --url http://host.docker.internal:8080

# Linux / VPS
docker run -it --rm --network host cloudflare/cloudflared:latest \
  tunnel --url http://localhost:8080
```

終端機輸出找到 `https://xxxx.trycloudflare.com`，這就是臨時公開網址。

### 6-2：設定 LINE Webhook

1. 到 [LINE Developers Console](https://developers.line.biz/console/) → 你的 Channel → **Messaging API**
2. Webhook URL 貼上：`https://xxxx.trycloudflare.com/webhook/line`
3. 按 **Verify** → 成功
4. 打開 **Use webhook**
5. 關閉 **Auto-reply messages**

### 6-3：最終驗收

```bash
docker compose run --rm openclaw-cli plugins list
docker compose run --rm openclaw-cli channels list
docker compose run --rm openclaw-cli channels status --probe
```

用你的 LINE 傳一句話，Bot 有回覆 = 部署成功。

### 6-4：用 Claude Code 客製化 Bot

```bash
cd <openclaw-project-directory>
claude
```

貼給 Claude Code：
```
請幫我找到這個 Openclaw 專案裡設定系統提示詞（System Prompt）的位置。
把 LINE 機器人改成「<你想要的角色>」。
改完請告訴我需要重啟哪個服務，並給我指令。
```

改完重啟：
```bash
docker compose restart openclaw-gateway
```

回 LINE 傳一句話，語氣變了就代表「AI 協作開發」閉環跑通。

---

## Step ⑦：升級固定 Cloudflare Tunnel（永久網址）

Quick Tunnel 每次重開電腦都會變網址。固定 Tunnel 設定一次就永久有效，LINE Webhook 不用再改。

### 前置條件

- Cloudflare 帳號已登入
- 一個網域已在 Cloudflare DNS 管理（沒有的話先買一個，見 Step ① 說明）
- 已進入 [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)

### 7-1：建立 Tunnel

1. Zero Trust Dashboard → **Networks** → **Tunnels**
2. **Create a tunnel** → 選 **Cloudflared**
3. 命名（如 `openclaw-bot`）
4. 複製保存 **Tunnel Token**（一長串 base64 字串）

### 7-2：設定 Public Hostname

在 Tunnel 設定中 → **Public Hostname** → **Add a public hostname**：

| 欄位 | 值 |
|------|-----|
| Subdomain | `bot`（或你想要的名稱） |
| Domain | 選你的網域 |
| Service Type | HTTP |
| URL | `openclaw-gateway:8080` |

最終網址範例：`https://bot.yourdomain.com`

### 7-3：修改 docker-compose.yml

在 OpenClaw 專案的 `docker-compose.yml` 加入 cloudflared 服務：

```yaml
services:
  openclaw-gateway:
    # 原本的設定保留不動
    restart: always

  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: always
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    depends_on:
      - openclaw-gateway
```

建立 `.env` 檔案（和 docker-compose.yml 同層）：
```bash
TUNNEL_TOKEN=貼上你在 Cloudflare 拿到的 Token
```

確認 `.gitignore` 包含 `.env`，Token 不能推到 Git。

### 7-4：啟動固定 Tunnel

先停掉 Quick Tunnel（關掉那個終端機視窗），然後：

```bash
docker compose up -d
```

這會同時啟動 Gateway 和 Cloudflare Tunnel。

### 7-5：更新 LINE Webhook（最後一次）

到 LINE Developers Console → Messaging API：
- Webhook URL 改為：`https://bot.yourdomain.com/webhook/line`
- **Verify** → 成功
- 確認 **Use webhook** 開啟

此後不需要再更新 Webhook URL。重開機、休眠後 Tunnel 會自動恢復。

### 驗收條件

```bash
docker compose ps
docker compose logs cloudflared
```

- `openclaw-gateway` 和 `cloudflared` 都是 Up
- cloudflared 日誌看到 `Connection registered`
- LINE 傳一句話，Bot 有回覆
- 重開機後 LINE 再傳一句話，Bot 仍然有回覆（不需要手動操作任何東西）

---

## 每日開機儀式

### 固定 Tunnel 模式（Step ⑦ 完成後）

什麼都不用做。`restart: always` 會讓服務自動啟動。只要確認 Docker Desktop 有開。

### Quick Tunnel 模式（還沒做 Step ⑦）

每天開機要重做：
1. 確認 Docker Desktop 已啟動
2. `docker compose ps` 確認 `openclaw-gateway` 是 Up
3. 重跑 Quick Tunnel 指令，拿到新網址
4. 到 LINE Developers 更新 Webhook URL（加 `/webhook/line`）並 Verify
5. LINE 傳一句話確認有回覆

---

## 壞掉急救三步

```bash
# 1) 錯誤訊息貼給 Claude Code 先修一次

# 2) 還是壞掉就還原
git restore .

# 3) 重啟服務
docker compose restart openclaw-gateway
```

---

## 兩週任務地圖

| 時間 | 目標 | 驗收標準 |
|------|------|----------|
| 今天下課前 | Bot 活著 | LINE 能收發 + 至少改一次人設 |
| 第 1 週 | 可展示版本 | 固定一個主題（FAQ Bot、記帳助理等），每功能做完就測 |
| 第 2 週 | 凍結功能 | 停止加功能、穩定現有、準備 Demo Day |

Demo Day 交付最低標準：
1. 一人一個能對話的 LINE Bot
2. 一個明確使用情境
3. 一段你如何指揮 Claude Code 完成改動的說明

---

## 排錯速查

| 症狀 | 最可能原因 | 第一步 |
|------|-----------|--------|
| 工具裝完指令找不到 | 沒重開終端 / 沒 source | 重開終端機或 `source ~/.zshrc` |
| Docker 指令報 daemon 錯 | Docker Desktop 沒開 | 先開 Docker Desktop |
| Verify 失敗 | 網址不通或 Gateway 沒跑 | `docker compose ps` 確認 Up |
| 只有已讀無回覆 | 服務沒跑或 Tunnel 斷了 | 重啟 Gateway + 重跑 Tunnel |
| 401 / 403 | Token/Secret 貼錯 | 重新 Issue Token 再綁定 |
| 429 | 免費模型限流 | 等幾秒自動恢復，或切備援模型 |

完整排錯指南請閱讀 `openclaw-linebot-master` skill 的 `references/troubleshooting.md`。

---

## 進階功能

完成 Step ①～⑦ 後，Bot 已能穩定在 LINE 上對話。以下進階功能請閱讀 `openclaw-linebot-master` skill：

- Flex Message 卡片式回覆、Carousel 多卡片滑動
- Gemini + matplotlib 圖片生成連續技
- 知識卡片導航系統
- 完整排錯決策樹

---

## 相關 References

| 檔案 | 內容 |
|------|------|
| `references/prerequisite-accounts.md` | 帳號註冊指引 + 域名設定 |
| `references/concept-glossary.md` | Token、上下文、計費、模型選型速查 |
| `openclaw-linebot-master` skill | 進階回覆格式、圖片生成、完整排錯指南 |
