# 帳號註冊指引

課程開始前，請確保以下五個帳號都已註冊完畢。建議在有網路的環境提前完成，避免上課時卡在信箱驗證。

---

## 1. GitHub

用途：下載 OpenClaw 專案原始碼（`git clone`）。

註冊：https://github.com/signup

驗收：能登入 https://github.com/ 看到自己的 Dashboard。

常見問題：
- 信箱驗證信在垃圾郵件資料夾
- 建議使用個人信箱，公司信箱偶爾有防火牆擋驗證信

---

## 2. Anthropic（Claude Console）

用途：Claude Code 登入授權。Claude Code 是住在終端機裡的 AI 助理，用中文描述需求就能幫你改程式。

註冊：https://console.anthropic.com/

驗收：能登入 Console 首頁。

注意事項：
- Claude Pro/Max 月費訂閱與 API 是分開的計費體系
- 課堂主要用 Claude Code 的功能，不一定需要付費訂閱
- 如果你有 Claude Pro/Max，Claude Code 已包含在內

---

## 3. OpenRouter

用途：取得 AI 模型 API Key。OpenRouter 像是 AI 模型的轉運站——一個 Key 就能切換不同廠商的模型。

註冊：https://openrouter.ai/

取得 API Key：https://openrouter.ai/keys → **Create Key**

驗收：能看到自己的 API Key（格式通常是 `sk-or-v1-xxxx...`）。

注意事項：
- 先複製保存你的 Key，離開頁面就看不到完整值了
- 部分模型有免費額度，課堂就用這些
- 免費模型遇到 429 錯誤是正常限流，等幾秒重試

---

## 4. LINE Developers

用途：建立 Messaging API Channel，讓你的 Bot 能在 LINE 上收發訊息。

註冊/登入：https://developers.line.biz/console/

### 建立 Channel 步驟

1. 登入後點 **Create a new provider**（取個名稱，例如你的名字）
2. 在 Provider 下點 **Create a Messaging API channel**
3. 填寫必要資訊：
   - Channel name：你的 Bot 名稱
   - Channel description：簡短描述
   - Category / Subcategory：隨便選
   - Email：你的信箱
4. 同意條款後建立

### 取得兩把鑰匙

| 名稱 | 位置 | 用途 |
|------|------|------|
| Channel Access Token | Messaging API → Issue（長按鈕） | 給 OpenClaw 發送訊息的權限 |
| Channel Secret | Basic settings | 驗證訊息來源真偽 |

驗收：能在 Console 中看到你的 Channel，並且已取得 Token 和 Secret。

---

## 5. Cloudflare

用途：建立 Tunnel 讓外網能打到你的本機服務。LINE 的 Webhook 必須是 HTTPS 公開網址。

註冊：https://dash.cloudflare.com/sign-up

驗收：能登入 Cloudflare Dashboard。

注意事項：
- 課堂階段先用 Quick Tunnel 快速打通流程
- 升級固定 Tunnel 需要進入 Zero Trust Dashboard：https://one.dash.cloudflare.com/
- 固定 Tunnel 需要一個你控制的網域，且 DNS 託管在 Cloudflare

---

## 6. 網域（固定 Tunnel 必備）

用途：固定 Cloudflare Tunnel 需要一個你控制的域名。設定一次之後 Webhook URL 就永久固定，不用每天換。

Quick Tunnel 先跑通不需要域名，但要穩定上線就需要。

### 購買管道

| 服務 | 說明 |
|------|------|
| [Cloudflare Registrar](https://www.cloudflare.com/products/registrar/) | 直接在 Cloudflare 買，DNS 自動託管，最省事 |
| [Namecheap](https://www.namecheap.com/) | 便宜域名選擇多（`.xyz` 約 $1/年） |
| [Google Domains](https://domains.google/) | Google 帳號直接管理 |

### 設定步驟

1. 購買域名（例如 `mybot.xyz`）
2. 到 Cloudflare Dashboard → **Add a site** → 輸入你的域名
3. Cloudflare 會給你兩組 Nameserver（例如 `xxx.ns.cloudflare.com`）
4. 回到域名購買商的後台，把 Nameserver 改成 Cloudflare 給的那兩組
5. 等 DNS 生效（通常 5 分鐘～24 小時）
6. Cloudflare Dashboard 看到你的域名狀態為 **Active** 即完成

驗收：Cloudflare Dashboard 中域名顯示為 Active。

---

## 帳號完成度自檢

```
- [ ] GitHub 可登入
- [ ] Anthropic Console 可登入
- [ ] OpenRouter API Key 已建立並保存
- [ ] LINE Developers Channel 已建立，Token + Secret 已取得
- [ ] Cloudflare 可登入
- [ ] （選配）域名已購買且在 Cloudflare DNS 管理中顯示 Active
```

前五個都打勾才能進入工具安裝階段。域名可以後續再補，不阻擋 Quick Tunnel 先跑通。
