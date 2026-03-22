> **參考附錄** — `references/technical-facts.md`。速查條款；細節指令仍以各參考檔為準。

## 技術事實（避免教錯）

- **學員狀態檔**：與上層 `SKILL.md` 同目錄之 **`LEARNER_STATE.md`**（純 Markdown，單一檔案集中記錄勇者代碼、安裝路線、Q3 現況、8 關進度、足跡；觸發即更新）。結構範本見 **`LEARNER_STATE.example.md`**
- **安裝環境代號**：**macOS 1.a** 本機（**無** Docker）、**macOS 1.b** 容器包裝（**要** Docker）、**Windows 2.a／2.b** 皆**不要求** Docker；對照與 `openclaw` 替換規則見 **`level-1-install.md`** 開頭
- **Skills 路徑**：`~/.openclaw/workspace/skills/<name>/SKILL.md`（多一層目錄不會被掃到）
- **課程 skills 取得**：公開倉 `https://github.com/gba3124/openclaw-skills.git` → **HTTPS** `git clone` 或 **GitHub 網頁 Download ZIP** 皆可，**不需** GitHub 登入／Token；誤用 SSH 網址才會像要登入（詳 **`level-1-install.md` 的 1-1a** 與 **`troubleshooting-and-ops.md` 環境前置**）
- **預設模型**：`openrouter/anthropic/claude-sonnet-4.5`
- **工具權限**必須：`tools.profile=full` 且 `tools.sessions.visibility=all`
- **僅**走 **Docker 跑 OpenClaw** 時：只 `git pull` 不會升容器版本，須 `docker build` + `force-recreate`（見 `troubleshooting-and-ops.md`）；本機 `openclaw` 路線依官方更新方式
- 用 compose 時 `.env` 與 `docker-compose.yml` 同層並在 `.gitignore`；本機路線依官方設定路徑
- **固定 LINE Webhook**：全路線見 **`line-webhook-fixed-url.md`**（ngrok 免費免網域；Cloudflare Tunnel＋自備網域）。選 Cloudflare 且用 **Compose** 時再加讀 **`levels-2-8-compose-branch.md`** 的 cloudflared 片段。Quick Tunnel（`trycloudflare.com`）不適合當永久 Webhook
- **CLI 速查**：`openclaw` 子指令、本機／Docker 前綴、課程常用範本見 **`openclaw-cli-reference.md`**；權威文件 [docs.openclaw.ai/cli](https://docs.openclaw.ai/cli/index)

---

## 進階功能（完成 8 關後再看）

詳見 `openclaw-linebot-master` skill：
- Flex Message / Carousel 卡片式回覆
- Gemini + matplotlib 圖片生成
- 完整排錯決策樹
