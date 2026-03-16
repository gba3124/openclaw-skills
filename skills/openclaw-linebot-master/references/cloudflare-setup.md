# Cloudflare Tunnel 完整設定指南

LINE Bot 的 Webhook 要求 HTTPS 公開網址。Cloudflare Tunnel 讓你的本機服務安全暴露到外網，不需要自己處理 SSL 憑證。

---

## 方案比較

| 項目 | Quick Tunnel | 固定 Tunnel |
|------|-------------|-------------|
| 需要帳號 | 不需要 | 需要 Cloudflare 帳號 |
| 需要網域 | 不需要 | 需要（可用免費子網域） |
| 網址穩定性 | 每次重開都會變 | 永久固定 |
| 適用場景 | 開發測試、課堂練習 | 正式上線、Demo Day |
| 設定複雜度 | 一行指令 | 需要 Zero Trust 後台設定 |
| 重開機後 | 要重新貼 Webhook URL | 自動恢復，不用動 |

---

## Quick Tunnel（開發用）

### 啟動

確保 OpenClaw Gateway 已在跑（`docker compose ps` 看到 Up）。

```bash
# Mac / Windows (Docker Desktop)
docker run -it --rm cloudflare/cloudflared:latest \
  tunnel --url http://host.docker.internal:8080

# Linux / VPS（直接用 host network）
docker run -it --rm --network host cloudflare/cloudflared:latest \
  tunnel --url http://localhost:8080
```

### 取得網址

終端機輸出中找到這一行：
```
+--------------------------------------------------------------------------------------------+
|  Your quick Tunnel has been created! Visit it at (it may take some time to be reachable):  |
|  https://random-words-here.trycloudflare.com                                               |
+--------------------------------------------------------------------------------------------+
```

### 設定 Webhook

Webhook URL 格式：
```
https://random-words-here.trycloudflare.com/webhook/line
```

到 LINE Developers Console → Messaging API → Webhook URL 貼上 → Verify → 開啟 Use webhook。

### Quick Tunnel 限制

- 關掉終端機或電腦休眠後，網址立即失效
- 每次重開 Tunnel 網址都不同
- 必須手動到 LINE 後台更新 Webhook URL
- 不適合長時間運行的正式服務

### 每天開機儀式

1. 啟動 Docker Desktop
2. `docker compose up -d openclaw-gateway`
3. 重跑 Quick Tunnel 指令
4. 複製新的 `trycloudflare.com` 網址
5. 到 LINE Developers 更新 Webhook URL（加 `/webhook/line`）
6. 按 Verify，確認成功
7. LINE 傳一句話測試

---

## 固定 Tunnel（正式上線用）

### 前置準備

1. Cloudflare 帳號：[dash.cloudflare.com](https://dash.cloudflare.com/)
2. 一個網域（在 Cloudflare DNS 管理）
3. 進入 [Cloudflare Zero Trust](https://one.dash.cloudflare.com/)

### Step 1：建立 Tunnel

1. 進入 Zero Trust Dashboard → **Networks** → **Tunnels**
2. 點擊 **Create a tunnel**
3. 選 **Cloudflared**
4. 命名（如 `openclaw-bot`）
5. 記下 **Tunnel Token**（一長串 base64 字串）

### Step 2：設定 Public Hostname

在 Tunnel 設定中 → **Public Hostname** → **Add a public hostname**：

| 欄位 | 值 |
|------|-----|
| Subdomain | `bot`（或你想要的名稱） |
| Domain | 選你的網域 |
| Service Type | HTTP |
| URL | `openclaw-gateway:8080` |

最終網址範例：`https://bot.yourdomain.com`

### Step 3：修改 docker-compose.yml

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
      - TUNNEL_TOKEN=貼上你在 Cloudflare 拿到的 Token
    depends_on:
      - openclaw-gateway
```

### Step 4：啟動

```bash
docker compose up -d
```

這會同時啟動 OpenClaw Gateway 和 Cloudflare Tunnel。

### Step 5：設定 LINE Webhook（只需一次）

Webhook URL：
```
https://bot.yourdomain.com/webhook/line
```

到 LINE Developers Console 貼上 → Verify → Use webhook 開啟。

此後不需要再更新 Webhook URL，即使重開機 Tunnel 也會自動恢復。

---

## 安全注意事項

### Tunnel Token 管理

- Token 等同服務密碼，不要提交到 Git
- 建議用 `.env` 檔案管理：

```bash
# .env
TUNNEL_TOKEN=eyJhIjoixxxx...
```

```yaml
# docker-compose.yml
services:
  cloudflared:
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
```

- 確保 `.gitignore` 包含 `.env`

### SSL 憑證

使用 Cloudflare Tunnel 時，SSL 由 Cloudflare 自動處理：
- 外部連線（LINE → Cloudflare）：Cloudflare 提供有效 SSL
- 內部連線（Cloudflare → OpenClaw）：走 Tunnel 加密通道
- 不需要自己申請 Let's Encrypt 或處理憑證

### LINE 對 SSL 的要求

LINE Webhook 驗證時會檢查：
1. 憑證必須是有效的 CA 簽發（Cloudflare 自動處理）
2. 憑證鏈必須完整
3. 不接受自簽憑證

使用 Cloudflare Tunnel 完全繞過這些問題。

---

## Tunnel 狀態檢查

### 確認 Tunnel 運行中

```bash
# 檢查所有服務狀態
docker compose ps

# 查看 Cloudflare Tunnel 日誌
docker compose logs cloudflared

# 只看最近 50 行
docker compose logs --tail 50 cloudflared
```

### 常見問題

| 症狀 | 原因 | 解法 |
|------|------|------|
| Tunnel 連不上 | Token 錯誤 | 到 Zero Trust 重新複製 Token |
| 間歇性斷線 | 網路不穩 | `restart: always` 會自動重連 |
| Webhook Verify 失敗 | Gateway 未啟動 | 先 `docker compose up -d openclaw-gateway` |
| 502 Bad Gateway | Gateway 啟動中 | 等 10 秒再 Verify |

### 從 Quick Tunnel 升級到固定 Tunnel

1. 停掉 Quick Tunnel 的終端機
2. 按照上方「固定 Tunnel」步驟設定
3. 到 LINE 後台改 Webhook URL 為固定網址
4. Verify 成功後就完成切換
