> **參考附錄** — `references/levels-2-8-compose-branch.md`
>
> **檔名與分類：** 這是 **Docker Compose 支線** 的關卡 2～8 指令稿，**不是**全課程預設。只有安裝代號 **macOS 1.b**（刻意用容器跑 OpenClaw）會用到；**Windows 2.a／2.b** 與 **macOS 1.a** 佔多數情境，應走本機 **`openclaw …`**（見 `openclaw-cli-reference.md`、`level-1-install.md`），**勿**為了對齊本檔去裝 Docker。
>
> **適用範圍（必讀）：** 僅當學員**已經**用 **`docker compose` 跑起 OpenClaw Gateway** 時照抄下列 `docker compose run --rm openclaw-cli …`。其餘人把同一組子指令改成 **`openclaw <子指令>`**（拿掉 compose 前綴）。

### 關卡 2：完成模型設定

```bash
# Primary 模型
docker compose run --rm openclaw-cli models set openrouter/anthropic/claude-sonnet-4.5

# Fallback 模型
docker compose run --rm openclaw-cli models fallbacks clear
docker compose run --rm openclaw-cli models fallbacks add openrouter/moonshotai/kimi-k2.5
docker compose run --rm openclaw-cli models fallbacks add openrouter/deepseek/deepseek-chat
docker compose run --rm openclaw-cli models fallbacks add openrouter/openai/gpt-4.1
docker compose run --rm openclaw-cli models fallbacks add openrouter/openrouter/auto
docker compose run --rm openclaw-cli models fallbacks add openrouter/openrouter/free

# 解鎖工具權限（必做，否則 Bot 只能聊天不能做事）
docker compose run --rm openclaw-cli config set tools.profile full
docker compose run --rm openclaw-cli config set tools.sessions.visibility all
docker compose restart openclaw-gateway
```

驗收：
```bash
docker compose run --rm openclaw-cli models status --plain
docker compose run --rm openclaw-cli config get tools.profile          # 要是 full
docker compose run --rm openclaw-cli config get tools.sessions.visibility  # 要是 all
```

---

### 關卡 3：完成 LINE 外掛啟用

```bash
docker compose run --rm openclaw-cli plugins install extensions/line
docker compose run --rm openclaw-cli plugins enable line
docker compose up -d openclaw-gateway
```

驗收：`docker compose run --rm openclaw-cli plugins list` 看到 `line` 狀態為 enabled

---

### 關卡 4：完成帳號註冊確認（LINE Bot Webhook）

**還沒有 LINE Developers 帳號的先去開：**  
👉 https://developers.line.biz/console/

開好再回來。

到 [LINE Developers Console](https://developers.line.biz/console/) 取得：
- **Channel Access Token**：Messaging API → Issue
- **Channel Secret**：Basic settings

```bash
docker compose run --rm \
  -e LINE_CHANNEL_ACCESS_TOKEN='你的 Token' \
  -e LINE_CHANNEL_SECRET='你的 Secret' \
  openclaw-cli channels add --channel line --use-env
```

驗收：`docker compose run --rm openclaw-cli channels list` 看到 `line`

---

### 關卡 5：完成 Webhook 驗證

**啟動 Quick Tunnel**

macOS / Docker Desktop：
```bash
docker run -it --rm cloudflare/cloudflared:latest \
  tunnel --url http://host.docker.internal:18789
```

Linux / WSL2：
```bash
docker run -it --rm --network host cloudflare/cloudflared:latest \
  tunnel --url http://localhost:18789
```

從終端機輸出找到 `https://xxxx.trycloudflare.com`

**設定 LINE Webhook**

到 [LINE Developers Console](https://developers.line.biz/console/) → Messaging API：
1. Webhook URL：`https://xxxx.trycloudflare.com/line/webhook`
2. 按 **Verify** → 成功
3. 開啟 **Use webhook**
4. 關閉 **Auto-reply messages**

驗收：Verify 顯示成功 + LINE 傳一句話有回覆

> ⚠️ Quick Tunnel 每次重開電腦後網址會變，需要重新設定 Webhook。  
> 下一關（關卡 7）固定 Webhook 網址：見 **`references/line-webhook-fixed-url.md`**（與是否 Docker 無關）；本檔只補 Compose 用的 cloudflared 片段。

---

### 關卡 6：完成 Token 怪獸瘦身優化

把環境變數寫入 `.env` 檔（防止重開電腦後失效）：

```bash
# 在專案根目錄（和 docker-compose.yml 同層）
echo "OPENROUTER_API_KEY=你的Key" >> .env
echo "LINE_CHANNEL_ACCESS_TOKEN=你的Token" >> .env
echo "LINE_CHANNEL_SECRET=你的Secret" >> .env

# 確保 .env 在 .gitignore 裡（Token 不能推上 git）
grep '.env' .gitignore || echo '.env' >> .gitignore
```

驗收：`cat .env` 有三行設定值，且 `.gitignore` 包含 `.env`

---

### 關卡 7：固定 LINE Webhook（與本檔的關係）

**教學主檔（全路線共用）：** **`references/line-webhook-fixed-url.md`**  
內含：方案對照（Cloudflare／ngrok）、註冊與設定步驟、LINE Verify、陪跑提醒。**不論**本機 `openclaw` 或 Compose，都先讀該檔。

**本檔僅補以下段落** — 僅當學員走 **Compose 支線** 且選 **Cloudflare 方案 A** 時，在專案裡加上 `cloudflared` 容器。

#### 關卡 7｜Compose 專用：方案 A 的 cloudflared 服務

在 `docker-compose.yml` 增加（與 `openclaw-gateway` 同檔）：

```yaml
  cloudflared:
    image: cloudflare/cloudflared:latest
    restart: always
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    depends_on:
      - openclaw-gateway
```

在 `.env` 加入：`TUNNEL_TOKEN=你的Token`（來自 `line-webhook-fixed-url.md` 的 7A-1）。

```bash
docker compose up -d
```

Zero Trust 後台設定 Public Hostname 時，**Service URL** 常填 **`http://openclaw-gateway:18789`**（與上列服務名一致）。

**驗收：** 完成 **`line-webhook-fixed-url.md`** 內 LINE Webhook **Verify** 與收訊測試；並確認 `docker compose ps` 中 `openclaw-gateway` 與 `cloudflared` 皆 **Up**。

---

### 關卡 8：完成跨界操控（Remote CDP）驗證

目標：讓 OpenClaw 可以操控你桌機的真實 Chrome。

**開啟宿主機 Chrome 的 CDP 埠**

macOS：
```bash
open -na "Google Chrome" --args --remote-debugging-port=9222
```

Windows（PowerShell）：
```powershell
Start-Process "chrome.exe" -ArgumentList "--remote-debugging-port=9222"
```

**取得 CDP URL**

WSL2（取 Windows 宿主 IP）：
```bash
ip route show | grep default | awk '{print $3}'
# CDP URL: ws://<取得的IP>:9222
```

Docker Desktop：`ws://host.docker.internal:9222`

**在 OpenClaw 設定：** Browser 模式 = `remote`，CDP URL = 上面的值

驗收：對 LINE Bot 說「幫我查今天台北天氣」，看到 Chrome 自動開頁搜尋

