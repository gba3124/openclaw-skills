# OpenClaw LINE Bot 排錯指南

從症狀出發，快速定位問題根因。

---

## 目錄

1. [排錯決策樹](#排錯決策樹)
2. [Webhook 相關問題](#webhook-相關問題)
3. [Docker / Gateway 問題](#docker--gateway-問題)
4. [LINE API 錯誤](#line-api-錯誤)
5. [Cloudflare Tunnel 問題](#cloudflare-tunnel-問題)
6. [AI 模型問題](#ai-模型問題)
7. [Flex Message 除錯](#flex-message-除錯)

---

## 排錯決策樹

```
LINE 傳訊息後沒回覆
    │
    ├── Webhook Verify 有成功嗎？
    │   ├── 沒試過 → 先去 LINE 後台 Verify
    │   ├── 失敗 → 看「Webhook 相關問題」
    │   └── 成功 → 繼續往下
    │
    ├── Gateway 有在跑嗎？
    │   ├── docker compose ps → 不是 Up → 看「Docker 問題」
    │   └── Up → 繼續往下
    │
    ├── Tunnel 有在跑嗎？
    │   ├── Quick Tunnel → 確認終端機還開著
    │   ├── 固定 Tunnel → docker compose logs cloudflared
    │   └── 都正常 → 繼續往下
    │
    ├── Token/Secret 正確嗎？
    │   ├── docker compose logs -f openclaw-gateway → 看有無 401
    │   ├── 有 401 → 重新 Issue Token 再綁定
    │   └── 沒 401 → 繼續往下
    │
    └── AI 模型有回應嗎？
        ├── 看 logs 有無 429 (rate limit) → 等幾秒重試
        ├── 看 logs 有無 timeout → 模型太慢，換一個
        └── 看 logs 有無其他錯誤 → 按訊息搜尋
```

---

## 先跑這 4 步（不要跳步）

```bash
# 1) Skills 路徑是否正確
ls -la ~/.openclaw/workspace/skills

# 2) 模型與認證是否正確
docker compose run --rm openclaw-cli models

# 3) 工具權限是否解鎖
docker compose run --rm openclaw-cli config get tools.profile
docker compose run --rm openclaw-cli config get tools.sessions.visibility

# 4) 版本與健康
docker compose run --rm openclaw-cli --version
curl -i http://127.0.0.1:18789/healthz
```

通過標準：
- Skills 可看到自訂 skill（來源為 workspace/managed）
- `Default` 是 `openrouter/anthropic/claude-sonnet-4.5`
- `tools.profile=full`、`tools.sessions.visibility=all`
- `healthz` 回 `200`

---

## Skills 路徑錯誤（最常見）

錯誤路徑（不會自動載入）：
```text
/app/skills/openclaw-skills/skills/<skill-name>/SKILL.md
```

正確路徑：
```text
~/.openclaw/workspace/skills/<skill-name>/SKILL.md
```
或：
```text
~/.openclaw/skills/<skill-name>/SKILL.md
```

修正後請重啟：
```bash
docker compose restart openclaw-gateway
docker compose run --rm openclaw-cli skills list
```

---

## Webhook 相關問題

### Verify 失敗：SSL connection error

症狀：LINE 後台 Verify 按鈕顯示 "An SSL connection error occurred"

檢查步驟：
```bash
# 確認服務可用（用瀏覽器開啟應該有回應）
curl -sI https://你的網址/line/webhook

# 檢查 SSL 憑證（若不用 Cloudflare Tunnel 時）
openssl s_client -connect your-domain.com:443 -servername your-domain.com 2>/dev/null | openssl x509 -noout -issuer
```

常見原因與解法：

| 原因 | 解法 |
|------|------|
| 自簽憑證 | 使用 Cloudflare Tunnel（自動處理 SSL） |
| 憑證鏈不完整 | 使用 fullchain.pem 而非 cert.pem |
| Tunnel 沒在跑 | 確認 cloudflared 容器或指令還活著 |
| Gateway 沒在跑 | `docker compose up -d openclaw-gateway` |
| URL 打錯 | 確認包含 `/line/webhook` 路徑 |

### Verify 成功但 Bot 不回覆

```bash
# 即時觀察 Gateway 日誌
docker compose logs -f openclaw-gateway
```

然後在 LINE 傳一句話，看日誌有沒有新的輸出：

- **沒有任何輸出** → LINE 沒有把訊息送到你的服務
  - 確認 Use webhook 已開啟
  - 確認 Auto-reply messages 已關閉
  - 確認 Tunnel 還在跑

- **有收到但報錯** → 看錯誤訊息對應下方章節

### Verify 失敗：`HTTP 404 Not Found`

症狀（LINE 後台）：
```
The webhook returned an HTTP status code other than 200. (404 Not Found)
```

高機率原因：Webhook 路徑填錯。  
OpenClaw LINE channel 常見正確路徑是：
```
/line/webhook
```
不是：
```
/webhook/line
```

快速判斷（本機先測）：
```bash
# 這條若回 404，代表你現在填錯路徑
curl -i -X POST http://127.0.0.1:18789/webhook/line \
  -H 'content-type: application/json' -d '{}'

# 這條若回 400 且顯示 Missing X-Line-Signature，代表路由存在
# （400 是正常的，因為這個手動測試沒有 LINE 簽章）
curl -i -X POST http://127.0.0.1:18789/line/webhook \
  -H 'content-type: application/json' -d '{}'
```

若你用 Quick Tunnel，請確認 LINE 後台填的是：
```
https://xxxx.trycloudflare.com/line/webhook
```

排查清單（照順序）：
1. Webhook URL 最後一定是 `/line/webhook`
2. `docker compose ps` 確認 `openclaw-gateway` 是 `Up`
3. Tunnel 有在跑（Quick Tunnel 終端機不能關）
4. 按 `Verify` 後再用 LINE 實際傳訊息驗收

### 常見誤解：換 Webhook 會不會讓 Token/Secret 一起變？

不會。  
Webhook URL、`Channel Access Token`、`Channel Secret` 是三個獨立項目：

- 換 Webhook URL：只改 LINE 要打到哪個網址
- Token/Secret：只有你在 LINE Developers 重新發行（Reissue/Reset）時才會改變

快速核對目前容器是否吃到正確值（遮罩顯示）：
```bash
docker inspect openclaw-openclaw-gateway-1 --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | awk -F= '/^LINE_CHANNEL_ACCESS_TOKEN=|^LINE_CHANNEL_SECRET=/{print $1"=<masked>"}'
```

當症狀是 `404` 時，優先修「路徑或網址」；  
當症狀是 `401/403` 時，再優先檢查「Token/Secret」。

### 收到 `access not configured` + `Pairing code`

症狀（LINE 訊息）：
```
access not configured.
Pairing code: XXXXXXXX
```

原因：Webhook 已通，但目前 LINE 使用者尚未完成配對授權（`dmPolicy=pairing` 預設行為）。

解法（擇一）：

```bash
# 直接核准當次配對碼
docker compose run --rm openclaw-cli pairing approve line <PAIRING_CODE>

# 查看是否還有待核准請求
docker compose run --rm openclaw-cli pairing list line
```

核准後，請使用者再傳一次訊息即可恢復正常回覆。

### Webhook URL 格式

正確格式：
```
https://你的網域/line/webhook
```

常見錯誤：
```
# ❌ 少了 /line/webhook
https://xxxx.trycloudflare.com

# ❌ 多了斜線
https://xxxx.trycloudflare.com/line/webhook/

# ❌ 用了 http 不是 https
http://xxxx.trycloudflare.com/line/webhook

# ✅ 正確
https://xxxx.trycloudflare.com/line/webhook
```

---

## Docker / Gateway 問題

### 刪掉 `~/.openclaw` 後 Gateway 一直重啟

症狀（logs）：
```
Missing config. Run `openclaw setup` or set gateway.mode=local
```

原因：主機上的 `~/.openclaw` 被移除後，容器掛載路徑內沒有完整設定。

解法：
```bash
# 1) 重建設定與工作區
docker compose run --rm openclaw-cli setup

# 2) 明確設定本機模式
docker compose run --rm openclaw-cli config set gateway.mode local

# 3) 重啟
docker compose up -d --force-recreate openclaw-gateway openclaw-office
```

補充：若 `openclaw-cli` 因 gateway 正在重啟而無法執行，先直接修主機檔案 `~/.openclaw/openclaw.json`，再重啟容器。

### `bind=lan` 啟動失敗（Control UI origins）

症狀（logs）：
```
Gateway failed to start: Error: non-loopback Control UI requires gateway.controlUi.allowedOrigins
```

原因：Gateway 綁定非 loopback（例如 `lan`）時，需要明確設定 Control UI 允許來源。

建議解法（優先）：
```json
{
  "gateway": {
    "mode": "local",
    "controlUi": {
      "allowedOrigins": [
        "http://127.0.0.1:18789",
        "http://localhost:18789"
      ]
    }
  }
}
```

若只是臨時救火，可加：
```json
"dangerouslyAllowHostHeaderOriginFallback": true
```
但此為降級安全選項，僅建議短期使用。

### `127.0.0.1:18789` 連不到的速查

先做三步：
```bash
# 1) 看容器狀態
docker compose ps

# 2) 看健康檢查
curl -i http://127.0.0.1:18789/healthz

# 3) 看 gateway logs
docker compose logs --tail 100 openclaw-gateway
```

判讀重點：
- `healthz` 回 `200` 但前端顯示離線：多半是配對或 WebSocket 權限問題（看下一節）。
- gateway 一直 restarting：先檢查 `gateway.mode` 與 `controlUi.allowedOrigins`。
- `OPENCLAW_GATEWAY_BIND=loopback` 時，容器內只聽本機，常導致主機映射路徑不可用；本機 Docker 開發通常建議用 `lan` 並搭配 `allowedOrigins`。

### Dashboard 顯示「pairing required」

症狀：UI 顯示離線、要求裝置配對批准。

解法：
```bash
# 列出待批准請求
docker compose run --rm openclaw-cli devices list

# 批准請求
docker compose run --rm openclaw-cli devices approve <requestId>
```

手機端補充：
```bash
docker compose run --rm openclaw-cli dashboard --no-open
```
使用完整 URL（含 `#token=...`）在手機瀏覽器開啟，否則常因 token 遺失無法完成握手。

### Docker Desktop 未啟動

症狀：
```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

解法：啟動 Docker Desktop 應用程式，等到圖示變綠色。

### Gateway 無法啟動

```bash
# 看狀態
docker compose ps

# 看詳細錯誤
docker compose logs openclaw-gateway

# 完全重建
docker compose down
docker compose up -d openclaw-gateway
```

### 常見 Docker 錯誤

| 錯誤訊息 | 原因 | 解法 |
|----------|------|------|
| port already in use | 8080 被佔用 | 停掉佔用 8080 的程式，或改 port |
| image not found | 映像檔不存在 | `docker compose pull` |
| no space left on device | 磁碟空間不足 | `docker system prune` 清理 |

### 重啟服務

```bash
# 重啟 Gateway（改完設定後必做）
docker compose restart openclaw-gateway

# 重啟全部（包含 Tunnel）
docker compose restart

# 砍掉重來
docker compose down && docker compose up -d
```

---

## LINE API 錯誤

### HTTP 400：JSON 格式錯誤

通常是 Flex Message 有不支援的屬性。

```bash
# 看具體錯誤欄位
docker compose logs --tail 100 openclaw-gateway | grep "400"
```

常見 400 錯誤：

| property 路徑 | 原因 | 修正 |
|--------------|------|------|
| `/hero/aspectMode` | 用了 `"contain"` | 改為 `"cover"` 或 `"fit"` |
| `/body/contents/N/text` | text 為空字串 | 改為 `"－"` 或移除 |
| `/borderWidth` | 不支援的屬性 | 移除，用 separator |

解讀 property 路徑：
```
/contents/0/body/contents/5/contents/1/text
│         │ │    │         │ │         └─ 是 text 屬性
│         │ │    │         │ └─ 第 2 個子元素
│         │ │    │         └─ 第 6 個元素
│         │ │    └─ contents 陣列
│         │ └─ body 區域
│         └─ carousel 第 1 張卡片
└─ contents（整個 carousel）
```

### HTTP 401：認證失敗

```json
{"message":"Authentication failed..."}
```

解法：
1. 到 LINE Developers Console → Messaging API
2. 點擊 **Reissue** 產生新 Token
3. 重新綁定：

```bash
docker compose run --rm \
  -e LINE_CHANNEL_ACCESS_TOKEN='新的 Token' \
  -e LINE_CHANNEL_SECRET='你的 Secret' \
  openclaw-cli channels add --channel line --use-env
```

### HTTP 429：超過請求限制

LINE 免費帳號有訊息數量限制。OpenRouter 免費模型也有 rate limit。

```
# 429 通常自動恢復，等幾秒重試即可
# 若頻繁遇到，考慮：
# 1. 升級 LINE 帳號方案
# 2. 切換到付費 AI 模型
```

---

## Cloudflare Tunnel 問題

### Quick Tunnel 網址失效

原因：關了終端機、電腦休眠、或 Cloudflare 回收。

解法：重新執行 Quick Tunnel 指令，拿新網址，更新 LINE Webhook。

### 固定 Tunnel 斷線

```bash
# 查看 Tunnel 日誌
docker compose logs cloudflared

# 常見錯誤
# "failed to connect" → 檢查網路連線
# "invalid token" → Token 過期或錯誤，到 Zero Trust 重新取得
# "connection refused" → Gateway 沒在跑
```

### Tunnel 連接但 Verify 失敗

Gateway 可能還在啟動中。等 10-15 秒再 Verify。

確認順序：
1. `docker compose ps` → Gateway 是 Up
2. `docker compose logs cloudflared` → 看到 "Connection registered"
3. 再到 LINE 後台 Verify

---

## AI 模型問題

### `No API key found for provider "anthropic"`

症狀：
```
Agent failed before reply: No API key found for provider "anthropic"
```

高機率原因：
1. 預設模型仍是 Anthropic 系列
2. 你想用 OpenRouter，但容器沒吃到 `OPENROUTER_API_KEY`

標準修復流程：
```bash
# 1) 套用 OpenRouter 六模型組
docker compose run --rm openclaw-cli models set openrouter/anthropic/claude-sonnet-4.5
docker compose run --rm openclaw-cli models fallbacks clear
docker compose run --rm openclaw-cli models fallbacks add openrouter/moonshotai/kimi-k2.5
docker compose run --rm openclaw-cli models fallbacks add openrouter/deepseek/deepseek-chat
docker compose run --rm openclaw-cli models fallbacks add openrouter/openai/gpt-4.1
docker compose run --rm openclaw-cli models fallbacks add openrouter/openrouter/auto
docker compose run --rm openclaw-cli models fallbacks add openrouter/openrouter/free

# 2) 檢查模型與認證
docker compose run --rm openclaw-cli models

# 3) 重新建立容器（確保新環境變數生效）
docker compose up -d --force-recreate openclaw-gateway openclaw-office
```

你應該看到：
- `Default: openrouter/anthropic/claude-sonnet-4.5`
- `Auth overview` 出現 openrouter 的有效來源（env 或 auth profile）

若還是失敗，檢查兩個服務都要有：
```yaml
environment:
  OPENROUTER_API_KEY: ${OPENROUTER_API_KEY:-}
```

### terminal 有 key，但容器仍顯示 missing auth

症狀：
- `printenv OPENROUTER_API_KEY` 有值
- 但 `docker compose run --rm openclaw-cli models` 仍顯示 `Missing auth - openrouter`

原因：
- key 只存在目前 shell session，沒有寫入專案 `.env`
- 或寫入後沒有重建容器

解法：
```bash
# 1) 先看 shell 是否有值
printenv OPENROUTER_API_KEY

# 2) 寫入 openclaw/.env
# OPENROUTER_API_KEY=sk-or-v1-...

# 3) 重建容器
docker compose up -d --force-recreate openclaw-gateway openclaw-office

# 4) 再驗證
docker compose run --rm openclaw-cli models
```

判斷成功：
- `Auth overview` 出現 openrouter 且來源為 env
- 不再出現 `No API key found for provider "anthropic"`

### 模型沒回應（Timeout）

```bash
# 看日誌中的模型呼叫
docker compose logs --tail 200 openclaw-gateway | grep -i "model\|timeout\|error"
```

解法：
```bash
# 切換模型
docker compose run --rm openclaw-cli models set openrouter/anthropic/claude-sonnet-4.5

# 或切到特定模型
docker compose run --rm openclaw-cli models set openrouter/moonshotai/kimi-k2.5

# 重啟
docker compose restart openclaw-gateway
```

### OpenRouter 429 (Rate Limit)

免費模型限流，通常等 3-5 秒會恢復。

Shell 設定檔中加入自動重試函式：
```bash
orfree() {
  for i in {1..5}; do
    echo "第 $i 次嘗試..."
    claude "$@" && break
    sleep 3
  done
}
```

---

## Flex Message 除錯

### 除錯步驟

1. 在 [LINE Flex Message Simulator](https://developers.line.biz/flex-simulator/) 測試 JSON
2. 逐步簡化 JSON 直到不報錯
3. 比對 Simulator 和實際 API 的差異

### 常見陷阱

| 問題 | 現象 | 解法 |
|------|------|------|
| JSON 超過 50KB | 訊息發不出去 | 拆分成多則訊息或減少內容 |
| Carousel 超過 12 張 | API 報錯 | 分頁或加目錄導航 |
| 巢狀超過 10 層 | API 報錯 | 簡化結構 |
| Bubble 內容超出高度 | 按鈕被截斷 | 用 `size: "giga"` 或減少內容 |

### 驗證 Flex Message JSON

送出前先驗證：
```python
import json

flex_json = { ... }
json_str = json.dumps(flex_json, ensure_ascii=False)
size_kb = len(json_str.encode('utf-8')) / 1024

if size_kb > 50:
    print(f"警告：JSON 大小 {size_kb:.1f}KB 超過 50KB 限制")
else:
    print(f"OK：JSON 大小 {size_kb:.1f}KB")
```

---

## 完整診斷指令清單

```bash
# 服務狀態
docker compose ps

# Gateway 日誌（即時跟蹤）
docker compose logs -f openclaw-gateway

# Tunnel 日誌
docker compose logs cloudflared

# 確認外掛安裝
docker compose run --rm openclaw-cli plugins list

# 確認頻道設定
docker compose run --rm openclaw-cli channels list

# 全面健康檢查
docker compose run --rm openclaw-cli channels status --probe

# 重啟所有服務
docker compose restart
```

---

## 版本升級排錯（Update available）

症狀：
- 介面顯示有新版本可更新
- 你已 `git pull`，但 runtime 還是舊版

原因：
- `openclaw:local` image 沒重建，容器仍跑舊映像

標準流程：
```bash
git pull --rebase --autostash
docker build -t openclaw:local .
docker compose up -d --force-recreate openclaw-gateway openclaw-office
docker compose run --rm openclaw-cli --version
curl -i http://127.0.0.1:18789/healthz
```

判斷成功：
- `--version` 為目標版本（或更高）
- `healthz` 回 `200 OK`

常見誤區：
- 只 `git pull` 沒 `docker build`
- 有更新但 UI 仍顯示舊版（瀏覽器快取，硬重新整理）

---

## API 費用控制（7 招落地 + 驗收）

### 1) 模型分層

設定範例（OpenRouter）：
```bash
docker compose run --rm openclaw-cli models set openrouter/anthropic/claude-sonnet-4.5
docker compose run --rm openclaw-cli models fallbacks clear
docker compose run --rm openclaw-cli models fallbacks add openrouter/moonshotai/kimi-k2.5
docker compose run --rm openclaw-cli models fallbacks add openrouter/deepseek/deepseek-chat
docker compose run --rm openclaw-cli models fallbacks add openrouter/openai/gpt-4.1
docker compose run --rm openclaw-cli models fallbacks add openrouter/openrouter/auto
docker compose run --rm openclaw-cli models fallbacks add openrouter/openrouter/free
```

驗收：
```bash
docker compose run --rm openclaw-cli models status --plain
docker compose run --rm openclaw-cli models fallbacks list
```

### 2) 控制上下文長度

執行原則：
- 只保留最近 5-10 則對話歷史
- 話題切換時重開 session
- 長對話改為「摘要記憶」而非完整歷史

驗收：
- 長對話每輪輸入 token 不再持續線性上升

### 3) FAQ 快取

執行原則：
- 先盤點高頻問題 Top 20-30
- 建立標準回覆，人工審核後入庫
- 相似度達門檻（例如 0.85）直接回快取

驗收：
- 高頻問題命中快取比例逐週提升
- 命中快取的請求不觸發模型呼叫

### 4) 本地模型分流（Ollama）

執行原則：
- 分類、擷取、格式化走本地
- 複雜推理與高品質生成走雲端

驗收：
- 雲端模型請求數下降
- 核心回覆品質維持可接受

### 5) Prompt 精簡

執行原則：
- 刪重複規則，保留核心約束
- 長描述改短規格 + 範例
- 不把可推導資訊重複寫進 system prompt

驗收：
- 每次請求的固定輸入 token 下降

### 6) 用量上限與告警

執行原則：
- 設日上限、月上限、50%/80% 告警
- dev/prod 分離 API key

驗收：
- 不會再出現單日異常爆量超支

### 7) 批次處理

執行原則：
- 可合併任務一次送出（分類、審核、翻譯）
- 降低每次請求的固定 overhead

驗收：
- 同量任務的 API 呼叫次數下降
- 同量任務的平均成本下降
