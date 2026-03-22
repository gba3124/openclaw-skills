> **參考附錄** — `references/troubleshooting-and-ops.md`。內容：版本更新、每日開機、**環境前置與常見坑**、排錯速查。

## 版本更新（看到 Update available 時）

**以下僅適用「用本機 clone 的 openclaw 專案 + Docker compose」路線：**

```bash
git pull --rebase --autostash
docker build -t openclaw:local .
docker compose up -d --force-recreate openclaw-gateway openclaw-office
docker compose run --rm openclaw-cli --version  # 確認新版本
curl -i http://127.0.0.1:18789/healthz          # 確認健康
```

> 只做 `git pull` 不會升容器映像，**一定要** `docker build` + `force-recreate`。**本機全域 `openclaw`** 請依 [官方文件](https://docs.openclaw.ai/install) 更新，勿硬抄上列。

---

## 每日開機確認

**固定 Webhook／Tunnel（關卡 7，見 `line-webhook-fixed-url.md`）完成後：** 若用 Compose 內 **cloudflared**，多數情況 `restart: always` 已處理。

**Quick Tunnel 模式（關卡 5，且用 Docker 跑 cloudflared 一次性容器）：**
1. 開 Docker Desktop
2. 在專案目錄 `docker compose up -d`
3. 重跑 Tunnel 指令，拿新網址
4. LINE Developers → 更新 Webhook URL → Verify

**本機 `openclaw` 路線：** 依官方方式確認 gateway／Tunnel 是否在跑，**不要**硬套「先開 Docker Desktop」。

---

## 環境前置與常見坑（GitHub、Windows、macOS）

**虛擬助手歐文**遇到學員卡關時，主動依作業系統對照下面條目排查；**不要假設**學員已裝好 Node、brew 或 PowerShell 已開權限。

### GitHub 與 clone（再強調一次）

| 狀況 | 處理 |
|------|------|
| 「沒 GitHub 不能 clone」 | 公開庫 **HTTPS** 與 **ZIP** 都不需要帳號；先換網址或改 ZIP（見 `level-1-install.md` 的 **1-1a**）。 |
| 只備份 SSH 網址 | 改 `https://github.com/使用者/倉庫.git`。 |
| 要 Token 才能下載 | 多為誤用私有庫網址、或憑證管理員搞錯；清憑證或換 ZIP。 |

### Windows（原生 PowerShell／CMD）

| 症狀 | 處理 |
|------|------|
| `irm ... \| iex` 被擋、`execution of scripts is disabled` | 以**目前使用者**開權限（不需全機強開）：`Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`，選 **Y**，**關閉 PowerShell 重開**再執行腳本。仍被擋再試：**以系統管理員開啟** PowerShell 執行同樣指令（僅在必要時）。 |
| `winget` 找不到 | 從 Microsoft Store 安裝 **App Installer**，或至 [WinGet 說明](https://learn.microsoft.com/windows/package-manager/winget/) 更新；暫時可改從官網下載 Git／Node 安裝包。 |
| 安裝程式說沒權限 | 對安裝檔或 PowerShell **右鍵 → 以系統管理員身分執行**（僅在該步需要）。 |
| `node`／`npm` 找不到（要走 **1-4d** npm 線） | 先裝 [Node.js LTS](https://nodejs.org/)，**重開終端**，再驗 `node -v`、`npm -v`；或**改走 1-4e** 官方 `install.ps1`（通常會處理 Node）。 |
| 把 **Mac 的 `brew`** 照抄到 Windows | **錯誤**：Windows 沒有 Homebrew 當預設；請用 **winget**、官網安裝檔，或 **WSL2** 內再用 `apt`。 |
| Git 在 Windows 裝了但指令仍沒有 | 重開終端；檢查安裝時是否勾選 **Add Git to PATH**；必要時重裝 Git for Windows。 |
| 路徑／反斜線搞混 | PowerShell 複製 skill 時用 `Copy-Item` 範例的路徑；在 **WSL** 裡請用 Linux 路徑與 `cp`，不要混用 `C:\...` 與 `~/` 同一條指令。 |

### macOS

| 症狀 | 處理 |
|------|------|
| 沒有 `brew` | Homebrew **不是**系統內建；依 [brew.sh](https://brew.sh) 一條指令安裝後再 `brew install ...`。 |
| 沒有 `git` | `xcode-select --install` 或 `brew install git`。 |
| 文件寫 Windows PowerShell | Mac 請用 **終端機**對應的 bash／zsh 區塊，不要執行 `irm \| iex`。 |

### 給歐文的帶跑順序（環境卡死時）

1. 請學員**貼完整錯誤原文**（或截圖裡逐字打出紅字）。
2. 對照上表**只做一項**修正，再試一次（符合「一次一題」精神，但技術上可一次只改一個變因）。
3. 通過後在 **`LEARNER_STATE.md`** 足跡記一筆（例：`排除：PowerShell ExecutionPolicy`）。

---

## 排錯速查

| 症狀 | 第一步 |
|------|--------|
| clone 要登入 GitHub／沒帳號 | 見 `level-1-install.md` 的 **1-1a** 與本檔 **GitHub 與 clone**；優先 HTTPS 或 ZIP |
| PowerShell 腳本不能執行 | 本檔 **環境前置與常見坑** → Windows → ExecutionPolicy |
| Windows 沒有 Node／npm | 裝 Node LTS 或改 **1-4e**（見 `level-1-install.md`）；見上表 |
| 指令找不到 | 重開終端 / `source ~/.zshrc`（Mac）；Windows 查 PATH |
| Docker daemon 錯誤 | **僅**在走 **macOS 1.b** 等「用 Docker 跑 OpenClaw」路線時：開 Docker Desktop；Windows／本機 `openclaw` 路線不適用 |
| Verify 失敗 | 若用容器：`docker compose ps` 確認 gateway Up；若本機 `openclaw`：查官方／`openclaw gateway status`。網址末端要加 `/line/webhook` |
| ngrok 顯示限流／額度用完 | 到 [ngrok Dashboard Usage](https://dashboard.ngrok.com/usage) 查配額；免費版有每月流量／請求上限，可等下月重置或改 **Cloudflare Tunnel**（`line-webhook-fixed-url.md` 方案 A；Compose 再加 `levels-2-8-compose-branch.md` cloudflared） |
| 有已讀無回覆 | 重啟 gateway + 重跑 Tunnel |
| 401 / 403 | 重新 Issue Token 再綁定 |
| 429 | 等幾秒讓限流恢復，或模型自動切 fallback |

**卡住急救三步（容器路線）：**
```bash
# 1) 先把錯誤訊息貼給 Claude Code 問他怎麼修
# 2) 還是壞掉（在 openclaw 專案根目錄）
git restore .
# 3) 重啟 gateway（僅 docker compose 路線）
docker compose restart openclaw-gateway
```
本機 `openclaw` 路線請改查 `openclaw doctor`／官方文件重啟 gateway，**不要**硬套 `docker compose`。
