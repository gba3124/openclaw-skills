> **參考附錄** — `references/line-webhook-fixed-url.md`
>
> **分類：** **關卡 7** — 把 LINE Webhook 從「每次會變的 Quick Tunnel」升級成 **固定 HTTPS 網址**。  
> **與安裝路線無關**：**本機 `openclaw`、WSL、macOS 1.a、Windows 2.a／2.b、Compose 支線**皆可照本檔擇一方案；只有選 **Cloudflare 方案 A** 且用 **Docker Compose** 跑 Gateway 時，才需額外讀 **`levels-2-8-compose-branch.md`** 內的 cloudflared 片段。

# 關卡 7：LINE Webhook 固定對外網址（ngrok／Cloudflare）

關卡 5 的 Quick Tunnel（例如 `*.trycloudflare.com`）**每次重開常變**，不適合當長期 Webhook。這裡改成 **Webhook 填一次就不用再改**（在 tunnel 與 Gateway 有正常運作的前提下）。

下面兩種都是實務上常用的 **免費或極低成本** 做法，**擇一即可**。額度與條款以各服務**官網最新文件**為準。

---

## 固定 Webhook 免費／低成本方案對照

| 方案 | 要不要**自有網域** | 網址長相 | 優點 | 注意 |
|------|-------------------|----------|------|------|
| **A. Cloudflare Tunnel** | **要**（網域可買便宜 TLD，DNS 託管在 Cloudflare；Tunnel 本身免費） | 你自己定，例如 `https://bot.你的網域.com` | 正式感、可自訂 hostname、與 Cloudflare DNS／WAF 同一生態 | 需買網域＋把 NS 指到 Cloudflare、設定步驟較長 |
| **B. ngrok 免費** | **不要** | ngrok 配給帳號 **1 個**自動分配的 dev 網域（例如 `xxxx.ngrok-free.app`，**名稱不可自訂**） | 註冊後很快就能固定 Webhook；適合不想先買網域的學員 | 有每月流量／請求等上限；免費版**不可**自訂子網域或綁自有網域（要付費方案） |

**ngrok 免費（官方）重點**（詳 [Free Plan Limits](https://ngrok.com/docs/pricing-limits/free-plan-limits/)）：含 **1 個自動分配的 development domain**、可用 `ngrok http <埠號>` 掛上該網域；免費版**不能**自訂網域名字。每月約 **1 GB** 出站流量、**約 2 萬** HTTP 請求等配額，小型課程 Bot 通常夠用。

**LINE Webhook 與 ngrok 插入頁**：免費版會對**一般瀏覽器開 HTML** 插入警告頁；**LINE 伺服器打 Webhook 屬 API／程式請求**，依 ngrok 說明**通常不受該插入頁影響**。

---

## 方案 A：Cloudflare Tunnel（固定網址，自備網域）

適合已經有（或願意買）網域、DNS 在 Cloudflare 的學員。

### 7A-1 建立 Tunnel

1. [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → Networks → Tunnels  
2. Create a tunnel → Cloudflared → 命名（如 `openclaw-bot`）  
3. 複製並保存 **Tunnel Token**

### 7A-2 設定 Public Hostname

| 欄位 | 值 |
|------|-----|
| Subdomain | `bot`（或自訂） |
| Domain | 你的網域 |
| Service Type | HTTP |
| Service URL | **依你怎麼跑 Gateway 填**（見下段 7A-3） |

最終對外網址：`https://bot.你的網域.com`（範例）

### 7A-3 把 Tunnel 接到 Gateway（依學員路線擇一）

- **本機／WSL 只跑 `openclaw`（無 Compose）**  
  在本機安裝並執行 `cloudflared`（見 [Cloudflare Tunnel 入門](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/)）。Public Hostname 的 **Service** 指到 Gateway 可達位址，常見為 **`http://127.0.0.1:18789`**（若埠不同請改）。

- **已用 Docker Compose 跑 `openclaw-gateway`（Compose 支線）**  
  在專案內加上 `cloudflared` 服務、`.env` 的 `TUNNEL_TOKEN` 等，見 **`levels-2-8-compose-branch.md`** → **關卡 7｜Compose 專用：cloudflared 服務**。Dashboard 裡 Service URL 常填 **`http://openclaw-gateway:18789`**（與 compose 服務名一致）。

### 7A-4 更新 LINE Webhook

Webhook URL：

`https://bot.你的網域.com/line/webhook`（與你實際 hostname 一致，路徑預設為 `/line/webhook`）

到 [LINE Developers Console](https://developers.line.biz/console/) → Messaging API → 貼上 → **Verify** → 開啟 **Use webhook**。

**驗收：** Verify 成功；手機傳訊息給 Bot 有回覆；重開機後不必改 Webhook（tunnel 與 Gateway 需有常駐或自動啟動）。

> 沒有網域：可先在 [Namecheap](https://www.namecheap.com/) 或 Cloudflare Registrar 買便宜 TLD，再把網域加入 Cloudflare 並改 NS。

---

## 方案 B：ngrok 免費（固定 dev 網域，免買網域）

適合 **不想先買網域**、希望快點得到固定 `https://…/line/webhook` 的學員。

### 7B-1 註冊與安裝

1. 至 [ngrok](https://ngrok.com/) 註冊帳號。  
2. 從 [Dashboard](https://dashboard.ngrok.com/) 取得 **Authtoken**。  
3. 本機安裝 [ngrok Agent](https://ngrok.com/download)，執行一次：  
   `ngrok config add-authtoken <你的TOKEN>`

### 7B-2 確認 Gateway 位址

OpenClaw Gateway 需已在本機聽 **18789**（與課程一致）。常見情況：

- Gateway 跑在**本機**或 **WSL**：在**同一台機器上**對 `127.0.0.1:18789` 開 tunnel。  
- Gateway 在 **Docker Desktop**：若埠已對應到本機 `18789`，同樣 `ngrok http 18789`。

### 7B-3 啟動 Tunnel

```bash
ngrok http 18789
```

終端機 **Forwarding** 網址應為 **`*.ngrok-free.app`**（免費 dev domain；細節以當版 CLI／Dashboard 為準）。

### 7B-4 設定 LINE Webhook

`https://<ngrok-顯示的主機名>/line/webhook`

到 [LINE Developers Console](https://developers.line.biz/console/) → Messaging API → **Verify** → **Use webhook**。

**驗收：** Verify 成功、有回覆；ngrok 與 Gateway 有跑時不必改 Webhook。

### 7B-5 背景常駐（選做）

可參考 ngrok [background service](https://ngrok.com/docs/agent/#background-service) 或系統服務；以官網為準。

---

## 陪跑提醒（歐文）

- 「不想買網域」→ 優先 **方案 B**。  
- 「要正式網域、長期營運」→ **方案 A**。  
- **不要**同時讓兩個對外網址指向同一 Bot，以免 Webhook 混亂。

---

## Webhook 路徑共識

OpenClaw LINE 外掛預設路徑多為 **`/line/webhook`**（自訂見 [LINE channel 文件](https://docs.openclaw.ai/channels/line)）。
