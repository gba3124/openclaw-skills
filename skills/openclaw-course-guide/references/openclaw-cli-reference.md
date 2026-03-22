> **參考附錄** — `references/openclaw-cli-reference.md`。  
> 供 **虛擬助手歐文** 查指令、替學員產生可貼上的命令。**權威與更新**以官方為準：  
> - [CLI Reference 總覽](https://docs.openclaw.ai/cli/index)  
> - [CLI 索引（llms.txt）](https://docs.openclaw.ai/llms.txt)

# OpenClaw CLI 指令速查（課程陪跑用）

## 怎麼呼叫（本機 vs 容器）

| 情境 | 前綴 |
|------|------|
| 本機已安裝 `openclaw`（macOS 1.a、Windows 2.a、WSL 內直裝等） | 直接 `openclaw …` |
| 專案內用 Docker Compose，服務名為 `openclaw-cli` | `docker compose run --rm openclaw-cli …` |
| 容器內已進入 shell、CLI 條目名為 `openclaw` | 直接 `openclaw …` |

以下範例以 **`openclaw`** 為準；走 Docker 時把開頭換成上表第二列即可。

## 全域常用旗標

- `--json` / `--plain`：給腳本或貼日誌用（多數子指令支援其一）。
- `--dev`：狀態隔離在 `~/.openclaw-dev`，埠位可能偏移。
- `--profile <name>`：狀態隔離在 `~/.openclaw-<name>`。
- `-V` / `--version`：版本。
- 詳見官方 [CLI index — Global flags](https://docs.openclaw.ai/cli/index)。

---

## 官方指令樹（節錄）

行為若與下表不符，以 **`openclaw <cmd> --help`** 與官網為準。

```
openclaw [--dev] [--profile <name>] <command>
  setup
  onboard
  configure
  config
    get | set | unset | file | validate
  completion
  doctor
  dashboard
  backup (create | verify)
  security audit ...
  secrets reload | ...
  reset | uninstall | update
  channels
    list | status | logs | add | remove | login | logout
    capabilities | resolve
  skills
    list | info | check
  plugins
    list | info | install | enable | disable | doctor | update | uninstall
    marketplace list <marketplace>
  memory status | index | search
  message
  agent
  agents list | add | delete
  status
  health
  sessions
  gateway
    call | health | status | probe | discover
    install | uninstall | start | stop | restart | run
  daemon (legacy；服務類同 gateway)
  logs
  system ...
  models
    list | status | set | set-image
    aliases list|add|remove
    fallbacks list|add|remove|clear
    image-fallbacks ...
    scan
    auth add|setup-token|paste-token|login ...
  sandbox ...
  cron ...
  browser ...
  pairing list | approve
  hooks ...
  webhooks ...
  docs
  tui
```

外掛可註冊額外頂層指令（例如 `voicecall`）。完整樹狀圖見 [docs.openclaw.ai/cli/index](https://docs.openclaw.ai/cli/index)。

---

## 本課程最常幫學員下的指令

### 安裝與健康檢查

```bash
openclaw --version
openclaw doctor
openclaw status
openclaw status --deep
openclaw health
openclaw gateway status
openclaw gateway health
openclaw gateway probe
```

### 模型（對應關卡 2）

```bash
openclaw models status
openclaw models status --plain
openclaw models set openrouter/anthropic/claude-sonnet-4.5
openclaw models fallbacks clear
openclaw models fallbacks add openrouter/moonshotai/kimi-k2.5
# …其餘 fallback 同課程 reference
```

說明見 [models CLI](https://docs.openclaw.ai/cli/models)。OpenRouter 類模型 id 需帶 **`provider/`** 前綴。

### 工具權限（課程必做）

課程腳本使用（路徑若隨版本變更，以 `openclaw config get`／官方為準）：

```bash
openclaw config set tools.profile full
openclaw config set tools.sessions.visibility all
openclaw config get tools.profile
openclaw config get tools.sessions.visibility
```

`tools.profile` 為全域工具組合（見官方 [Tools](https://docs.openclaw.ai/tools)）。`tools.sessions.visibility` 若與你安裝的版本不符，請 `openclaw config get tools` 或 `openclaw doctor` 查實際鍵名。

改完通常需 **重啟 Gateway**（本機：`openclaw gateway restart`；Docker：`docker compose restart openclaw-gateway`）。

### LINE 外掛（對應關卡 3）

官方套件名可為（擇一，以 `plugins list` 與文件為準）：

```bash
openclaw plugins install @openclaw/line
# 或從 openclaw 原始碼目錄：
openclaw plugins install ./extensions/line
openclaw plugins enable line
openclaw plugins list
```

見 [LINE channel](https://docs.openclaw.ai/channels/line)、[plugins CLI](https://docs.openclaw.ai/cli/plugins)。

### LINE 通道與環境變數（對應關卡 4）

先匯出或寫入 `.env`：

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

再執行（與課程 Docker 範例等價）：

```bash
openclaw channels add --channel line --use-env
openclaw channels list
```

互動式精靈：無旗標的 `openclaw channels add` 可逐步問帳號與綁定。詳 [channels CLI](https://docs.openclaw.ai/cli/channels)。

### LINE 配對（DM 預設 pairing 時）

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

見 [LINE 說明 — Access control](https://docs.openclaw.ai/channels/line)。

### Gateway 生命週期（本機服務）

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway run
```

選項與 `gateway.mode` 等見 [gateway CLI](https://docs.openclaw.ai/cli/gateway)。預設 WebSocket 埠多為 **18789**。

### 設定檔路徑與驗證

```bash
openclaw config file
openclaw config validate
```

### Skills（與課程「放 SKILL.md 到 workspace」並行）

```bash
openclaw skills list
openclaw skills info <name>
openclaw skills check
```

見 [skills CLI](https://docs.openclaw.ai/cli/skills)。

### 更新

```bash
openclaw update
```

（來源安裝與 npm 安裝行為見官方 [update](https://docs.openclaw.ai/cli/update)。）

---

## Webhook 路徑提醒（LINE）

Gateway 對外 HTTPS 根網址設為 `https://<你的網域或 tunnel>` 時，LINE Webhook 通常為：

`https://<同上>/line/webhook`

固定網址（ngrok／Cloudflare）的完整步驟見 **`line-webhook-fixed-url.md`**。自訂路徑時見 [LINE — Configure](https://docs.openclaw.ai/channels/line)（`webhookPath`）。

---

## 陪跑注意

1. 下指令前先確認學員是 **本機 `openclaw`** 還是 **`docker compose run … openclaw-cli`**，不要混用路徑與工作目錄。  
2. **機密**不要貼進聊天；用 `--use-env`、SecretRef 或請學員本機貼 `.env`。  
3. 子指令或旗標不確定時，優先 **`openclaw <cmd> --help`**，再查官方連結。
