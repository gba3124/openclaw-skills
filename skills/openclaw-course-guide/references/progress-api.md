> **參考附錄** — 路徑：`skills/openclaw-course-guide/references/progress-api.md`。主 SKILL：`../SKILL.md`（核心規則、何時呼叫 API 以主檔為準）。

## 進度同步 API

**每次關卡狀態或進度改變，agent 絕對必須主動呼叫：**

```bash
curl -sS -X POST "https://fusion-openclaw-slide.vercel.app/api/update-progress" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 11db0d11612819151e4d2def6b424bdb96b2432360ef1f5b99f6d3863430401d" \
  -d '{
    "code": "<勇者代碼>",
    "step": "<關卡名>",
    "status": "<進行中|完成>",
    "progress": <0-100>,
    "note": "<選填>"
  }'
```

**觸發時機：**
- 進入新關 → `status: 進行中`
- 驗收通過 → `status: 完成`

**step 只能用以下 8 個原文，不可自創：**

| # | step 名稱 |
|---|-----------|
| 1 | 完成 OpenClaw 安裝 |
| 2 | 完成模型設定 |
| 3 | 完成 LINE 外掛啟用 |
| 4 | 完成帳號註冊確認（LINE Bot Webhook） |
| 5 | 完成 Webhook 驗證 |
| 6 | 完成 Token 怪獸瘦身優化 |
| 7 | LINE Bot 串接成功並驗證 |
| 8 | 完成跨界操控（Remote CDP）驗證 |

API 回傳 `"ok": true` = 成功。若失敗，告知學員手動到網站更新。
