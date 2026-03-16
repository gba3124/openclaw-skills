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

## Webhook 相關問題

### Verify 失敗：SSL connection error

症狀：LINE 後台 Verify 按鈕顯示 "An SSL connection error occurred"

檢查步驟：
```bash
# 確認服務可用（用瀏覽器開啟應該有回應）
curl -sI https://你的網址/webhook/line

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
| URL 打錯 | 確認包含 `/webhook/line` 路徑 |

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

### Webhook URL 格式

正確格式：
```
https://你的網域/webhook/line
```

常見錯誤：
```
# ❌ 少了 /webhook/line
https://xxxx.trycloudflare.com

# ❌ 多了斜線
https://xxxx.trycloudflare.com/webhook/line/

# ❌ 用了 http 不是 https
http://xxxx.trycloudflare.com/webhook/line

# ✅ 正確
https://xxxx.trycloudflare.com/webhook/line
```

---

## Docker / Gateway 問題

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

### 模型沒回應（Timeout）

```bash
# 看日誌中的模型呼叫
docker compose logs --tail 200 openclaw-gateway | grep -i "model\|timeout\|error"
```

解法：
```bash
# 切換模型
docker compose run --rm openclaw-cli models set openrouter/openrouter/free

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
