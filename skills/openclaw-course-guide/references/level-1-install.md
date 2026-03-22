> **參考附錄** — `references/level-1-install.md`。內容：**安裝環境代號對照**、**關卡 1**（含 1-1a Git／skills、1-2～1-4 各線）。**關卡 7**（固定 LINE Webhook，ngrok／Cloudflare）全路線見 **`line-webhook-fixed-url.md`**。關卡 **2～6、8** 若走 **Compose 支線**（多為 **macOS 1.b**）見 **`levels-2-8-compose-branch.md`**；**未**用 compose 者勿照抄，改 `openclaw …`。

### 安裝環境差異總覽（陪跑前先對齊）

**先講清楚，避免 AI 誤以為人人都要 Docker：**

- **Windows（2.a 與 2.b）**：課程**不要求**安裝 Docker Desktop。兩條路都用 **`openclaw` CLI**（本機 PowerShell 或 WSL 內）完成後續關卡即可。
- **macOS**：**只有**選 **1.b**（容器包 OpenClaw）才需要 Docker Desktop；選 **1.a** 本機直裝則**全程不必** Docker。

同一份 **`levels-2-8-compose-branch.md`** 為了講義好寫，把 **`docker compose run --rm openclaw-cli …`** 寫得很細——**只**給 **Compose 支線**（已用 compose 跑 Gateway）的人用。**多數學員**走本機／WSL `openclaw`，請把同一組子指令改成 **`openclaw <子指令>`**（拿掉 `docker compose run --rm openclaw-cli` 前綴），其餘盡量維持；細節以 [OpenClaw 官方文件](https://docs.openclaw.ai/install) 為準。

實際安裝代號與 **Q2a／Q2b** 對照：

| 代號 | 環境 | 白話 | 對應小節 | 要不要 Docker |
|------|------|------|----------|----------------|
| **macOS 1.a** | macOS 本機 | OpenClaw 直接跑在 macOS | **1-4b** | **不要** |
| **macOS 1.b** | macOS + 容器 | 用 Docker 包一層（想隔離或對齊課堂 compose 示範時再選） | **1-4a** | **要**（Docker Desktop） |
| **Windows 2.a** | Windows 本機 | npm 或官方 PowerShell 腳本 | **1-4d**／**1-4e** | **不要** |
| **Windows 2.b** | WSL2 Ubuntu | OpenClaw 裝在子系統裡 | **1-4c** | **不要**（除非學員自己再加 Docker 包一層，非本課程預設） |

---

### 關卡 1：完成 OpenClaw 安裝

**1-1 安裝 Claude Code**

macOS：
```bash
brew install --cask claude-code
# 或
curl -fsSL https://claude.ai/install.sh | bash
```

Windows PowerShell：
```powershell
irm https://claude.ai/install.ps1 | iex
# 或
winget install Anthropic.ClaudeCode
```

WSL2 Ubuntu：
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

驗收：`claude --version` 有版本號 = 通過

---

**1-1a 安裝 Git 與課程 Skills（公開 `git clone`，免 GitHub 登入）**

課程陪跑會用到 `openclaw-course-guide`、`openclaw-linebot-master` 等 skill。  
取得方式：**不用登入 GitHub、不需要申請帳號**——公開倉庫用 **HTTPS** 讀取即可（不需 `gh auth`、不需 Personal Access Token、不需輸入密碼；**除非你誤用 SSH 網址或公司網路攔截**，見下方「被要求登入時」）。

**陪跑原則（歐文必做）：** 學員一說「clone 要登入／沒 GitHub」——**先判斷是誤會還是真的環境擋**，依下方順序帶排錯；不要直接叫他「去辦 GitHub」當成唯一解。

**步驟 1：確認本機有 Git**

```bash
git --version
```

- 有顯示版本號 → 直接進步驟 2。
- **macOS** 顯示找不到指令：
  - 優先：終端機執行 `xcode-select --install`，依彈窗裝完 **Xcode Command Line Tools**（內含 `git`）。
  - 或已用 Homebrew：`brew install git`。
- **Windows（PowerShell／CMD）** 沒有 `git`：`winget install --id Git.Git -e --source winget`，裝完重開終端機。
- **WSL2 Ubuntu**：若尚未裝過，之後在 Ubuntu 內執行 `sudo apt-get update && sudo apt-get install -y git`（與下方 1-4b 可併成一次做完）。

**步驟 2：Clone 課程 skills 倉庫（公開網址）**

在**你想放專案的上層目錄**（例如家目錄或 `~/Projects`）執行：

```bash
git clone https://github.com/gba3124/openclaw-skills.git
```

可改成自訂資料夾名，例如：`git clone https://github.com/gba3124/openclaw-skills.git ~/openclaw-skills`。

**務必使用 HTTPS 網址**（以 `https://github.com/` 開頭）。若學員貼的指令是 `git@github.com:gba3124/openclaw-skills.git`，那是 **SSH**，本機沒放 SSH 金鑰時就會卡住或跑去要登入——請改成上面這行 HTTPS。

**被要求登入 GitHub、或跳出帳密／Token 視窗時（公開倉不該需要）：**

1. **先看網址**：是否為 `https://github.com/gba3124/openclaw-skills.git`。不是就改回這條再試。
2. **拒絕用 SSH 混過去**：不要把「去辦 GitHub + 建 SSH key」當預設答案；先換 HTTPS。
3. **Windows 若跳出 Git Credential Manager**：多半是先前存過錯誤憑證或誤連到私有庫。可試：關掉視窗 → 用 **全新資料夾**再執行 clone；仍不行則在「認證小幫手」裡**清除 github.com 的舊憑證**後重試（不必為了公開 clone 新建 PAT）。
4. **公司／校園網路**：若 HTTPS 被攔截，可換手機熱點試；或改用下方 **ZIP 備援**。
5. **仍失敗**：請學員把**完整終端機錯誤原文**貼上（含紅字），再對症處理。

**ZIP 備援（完全不用 Git、也不用 GitHub 帳號）：**

1. 瀏覽器開：`https://github.com/gba3124/openclaw-skills`
2. 綠色 **Code** → **Download ZIP**
3. 解壓縮後會得到資料夾（常名 `openclaw-skills-main`），其中已有 `skills/openclaw-course-guide`、`skills/openclaw-linebot-master`
4. **步驟 3** 複製時，把路徑裡的資料夾名改成你解壓後的實際名稱（例如 `openclaw-skills-main\skills\...`）

**OpenClaw 主程式**（`open-claw/openclaw`）同樣是公開倉，原則相同：HTTPS clone 或 GitHub 網頁 Download ZIP，**不需**為了下載原始碼而登入。

**步驟 3：複製到 OpenClaw 會掃描的 skills 目錄**

OpenClaw 只認：`~/.openclaw/workspace/skills/<技能資料夾名>/SKILL.md`（多一層目錄不會被掃到）。

macOS / Linux / WSL2（Bash）：

```bash
mkdir -p ~/.openclaw/workspace/skills
cp -R openclaw-skills/skills/openclaw-course-guide ~/.openclaw/workspace/skills/
cp -R openclaw-skills/skills/openclaw-linebot-master ~/.openclaw/workspace/skills/
```

若 clone 時用的是自訂路徑，把第一段的 `openclaw-skills` 改成你的資料夾名稱。

Windows **原生 PowerShell**（未進 WSL、且 OpenClaw 讀的是使用者家目錄下同一路徑時）：

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.openclaw\workspace\skills" | Out-Null
Copy-Item -Recurse -Force "openclaw-skills\skills\openclaw-course-guide" "$env:USERPROFILE\.openclaw\workspace\skills\"
Copy-Item -Recurse -Force "openclaw-skills\skills\openclaw-linebot-master" "$env:USERPROFILE\.openclaw\workspace\skills\"
```

**驗收：**

```bash
test -f ~/.openclaw/workspace/skills/openclaw-course-guide/SKILL.md && test -f ~/.openclaw/workspace/skills/openclaw-linebot-master/SKILL.md && echo OK
```

兩個路徑都存在且印出 `OK` = 通過。

> **陪跑提示（歐文口語可講）：** 這步是「把講義 clone 下來放進助理會讀的抽屜」，跟等等要裝的 OpenClaw **主程式**是不同資料夾；先做好，後面問我課程步驟我才對得上 skill。

---

**1-2 Clone OpenClaw 專案**

先問講師是否有提供專案網址。有 → `git clone <那個網址>`。否則：

```bash
git clone https://github.com/open-claw/openclaw.git
cd openclaw
ls  # 確認有 docker-compose.yml
```

---

**1-3 登入 Claude Code**

```bash
claude
# 照畫面完成瀏覽器授權，完成後 exit
```

---

**1-4a（macOS 1.b｜Docker 包 OpenClaw）**

確認 Docker Desktop 已開（右上角綠色圖示）：

```bash
export OPENCLAW_EXTENSIONS="nostr"
./docker-setup.sh
docker compose up -d
docker compose ps  # openclaw-gateway 狀態要是 Up
```

---

**1-4b（macOS 1.a｜本機直裝 OpenClaw，非 Docker）**

需已通過 Q2a 風險確認。此路線**不必**先做下方 **1-2**（clone `openclaw` 專案）與 **1-4a**；本機 Gateway 由安裝腳本／`openclaw` CLI 管理即可。

官方推薦一鍵腳本（會處理 Node 需求）：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

若你堅持自己管 Node，可改用（須 Node 24 建議或 Node 22.16+，詳見[官方安裝頁](https://docs.openclaw.ai/install)）：

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

驗收：

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

> 關卡 2 起請依 **安裝環境差異總覽** 改用本機 `openclaw`，勿照抄 `docker compose run --rm openclaw-cli`。

---

**1-4c（Windows 2.b｜WSL2：OpenClaw 裝在 Ubuntu 內）**

以下為 **Ubuntu 內本機直裝**（與官方 Linux 流程相同）。若你之後改在 WSL 裡用 **Docker 包 OpenClaw**，再補做 **1-2** 與 **1-4a** 類步驟即可。

```powershell
# PowerShell：安裝 WSL2 + Ubuntu（若尚未裝）
wsl --install -d Ubuntu
```

重開機後進入 Ubuntu，執行：

```bash
sudo apt-get update && sudo apt-get install -y git
curl -fsSL https://claude.ai/install.sh | bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

若你跳過腳本裡的引導、改用自己管好的 Node，可改裝：

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

驗收：`claude --version` 與 `openclaw --version` 皆有版本號；若用 daemon，`systemctl status` 可確認 systemd 是否正常（依你的 WSL 發行版而定）。

> WSL 內路線若後續也用 Docker 包 OpenClaw，則與 macOS 1.b 類似，改在專案目錄用 `docker compose`；若純本機 `openclaw` daemon，關卡 2+ 同樣用 `openclaw` CLI 對照替換。

---

**1-4d（Windows 2.a｜本機 npm 套件裝 OpenClaw）**

在 **PowerShell** 或 **CMD**（非 WSL）。此路線**不必**先做 **1-2**（clone 專案）與 Docker 相關小節。須 **Node 24（建議）或 Node 22.16+**、`npm` 可用；若還沒有 Node，請先安裝 [Node.js LTS](https://nodejs.org/) 或改用 **1-4e** 腳本。

```powershell
node -v
npm -v
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

驗收：

```powershell
openclaw --version
openclaw doctor
openclaw gateway status
```

裝完必做安全體檢：確認 Gateway 未誤曝到公網（`openclaw doctor` 與官方文件）。

> 關卡 2 起請依 **安裝環境差異總覽** 用本機 `openclaw`，勿照抄 `docker compose run --rm openclaw-cli`。

---

**1-4e（Windows 2.a｜本機 PowerShell 官方安裝腳本）**

在已取得 Q2b 方式 B 同意、且使用者選「官方腳本」時使用。此路線**不必**先做 **1-2**（clone 專案）與 Docker 相關小節。以系統管理員身份開啟 **PowerShell**：

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
winget install --id Git.Git -e --source winget
```

裝完必做安全體檢（**PowerShell** 或新安裝程式附的終端機皆可）：

```powershell
openclaw doctor
# 確認 Gateway Port 未直接外露到公網
```

> 關卡 2 起請依 **安裝環境差異總覽** 用本機 `openclaw`，勿照抄 `docker compose run --rm openclaw-cli`。

---
