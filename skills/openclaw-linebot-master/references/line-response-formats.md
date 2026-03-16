# LINE Bot 回覆格式完整指南

涵蓋 LINE Messaging API 所有主要訊息格式，以及 Flex Message 的設計規範與限制。

---

## 目錄

1. [訊息格式總覽](#訊息格式總覽)
2. [Flex Message 限制](#flex-message-限制)
3. [Flex Message 卡片結構](#flex-message-卡片結構)
4. [Carousel 多卡片設計](#carousel-多卡片設計)
5. [導航按鈕設計](#導航按鈕設計)
6. [圖片訊息規範](#圖片訊息規範)
7. [常見錯誤與修正](#常見錯誤與修正)

---

## 訊息格式總覽

### 純文字

最基本的回覆格式，適合一般對話。

```json
{
  "type": "text",
  "text": "你好！我是 OpenClaw 驅動的 AI 助理 🦞"
}
```

### 圖片訊息

```json
{
  "type": "image",
  "originalContentUrl": "https://example.com/image.png",
  "previewImageUrl": "https://example.com/image_preview.png"
}
```

圖片規格：
- 格式：JPEG, PNG（不支援 GIF 動圖）
- 大小：建議 < 300KB
- 來源：必須是 HTTPS URL
- 預覽圖：建議 240×240px

### Sticker 貼圖

```json
{
  "type": "sticker",
  "packageId": "446",
  "stickerId": "1988"
}
```

### 影片訊息

```json
{
  "type": "video",
  "originalContentUrl": "https://example.com/video.mp4",
  "previewImageUrl": "https://example.com/video_preview.png"
}
```

### 位置訊息

```json
{
  "type": "location",
  "title": "Anthropic 辦公室",
  "address": "San Francisco, CA",
  "latitude": 37.7749,
  "longitude": -122.4194
}
```

---

## Flex Message 限制

Flex Message 是 LINE 最強大的訊息格式，可以建立卡片式 UI。

### 硬性限制

| 項目 | 限制 |
|------|------|
| JSON 大小 | 最大 **50KB** |
| Carousel 卡片數 | 最多 **12 張** |
| Box 巢狀層級 | 最多 **10 層** |
| 單張卡片寬度 | 建議 **280px** |
| Hero 圖片比例 | 建議 **4:3** 或 **16:9** |
| Hero aspectMode | 只支援 `"cover"` 或 `"fit"`（不支援 `"contain"`） |
| Button label | 最多 **20 字元**（超過會截斷） |
| 圖片格式 | JPEG, PNG |
| 圖片大小 | 建議 < **300KB** |
| 圖片來源 | 必須是 **HTTPS** URL |

### Bubble Size

| Size | 最大高度 | 適用場景 |
|------|---------|---------|
| nano | ~120px | 極小卡片 |
| micro | ~165px | 小型通知 |
| kilo | ~300px | 一般用途 |
| mega | ~450px | 內容較多 |
| giga | ~600px | 最大尺寸 |

內容超過限制時會被截斷，按鈕可能無法顯示。

### 不支援的屬性

使用以下屬性會導致 HTTP 400 錯誤：

| 不支援屬性 | 解決方案 |
|-----------|----------|
| `borderWidth` | 移除，改用 `separator` |
| `borderColor` | 移除，改用 `backgroundColor` |
| `aspectMode: "contain"` | 改用 `"fit"` |
| CSS 動畫 | 不支援 |
| SVG 嵌入 | 不支援 |

---

## Flex Message 卡片結構

### 標準結構

```
┌─────────────────────────┐
│       Hero 區域          │  ← 圖片或漸層背景
│    （圖片/符號/標題）     │
├─────────────────────────┤
│       Body 區域          │
│  ┌───────────────────┐  │
│  │ 標題 + 副標題      │  │
│  ├───────────────────┤  │
│  │ 資訊表格/內文      │  │
│  ├───────────────────┤  │
│  │ 重點框/摘要        │  │
│  └───────────────────┘  │
├─────────────────────────┤
│       Footer 區域        │
│  ┌───────────────────┐  │
│  │ 按鈕/導航          │  │
│  └───────────────────┘  │
└─────────────────────────┘
```

### 基礎 Bubble 範例

```json
{
  "type": "flex",
  "altText": "資訊卡片",
  "contents": {
    "type": "bubble",
    "size": "mega",
    "hero": {
      "type": "image",
      "url": "https://example.com/hero.png",
      "size": "full",
      "aspectRatio": "16:9",
      "aspectMode": "cover"
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "主標題",
          "weight": "bold",
          "size": "xl"
        },
        {
          "type": "text",
          "text": "這是內容描述，支援自動換行。",
          "wrap": true,
          "size": "md",
          "color": "#666666",
          "margin": "md"
        }
      ]
    },
    "footer": {
      "type": "box",
      "layout": "horizontal",
      "contents": [
        {
          "type": "button",
          "action": {
            "type": "message",
            "label": "了解更多",
            "text": "了解更多"
          },
          "style": "primary"
        }
      ]
    }
  }
}
```

### Hero 區域配色（Material Design 推薦）

| 用途 | 淺色 | 深色 |
|------|------|------|
| 紅/粉 | #FFCDD2 | #EF9A9A |
| 橘 | #FFE0B2 | #FFCC80 |
| 藍 | #BBDEFB | #90CAF9 |
| 綠 | #C8E6C9 | #A5D6A7 |
| 紫 | #E1BEE7 | #CE93D8 |
| 青 | #B2EBF2 | #80DEEA |

### 色塊模擬技巧

用 Box + backgroundColor 模擬色塊圖示：

```json
{
  "type": "box",
  "layout": "vertical",
  "contents": [],
  "width": "28px",
  "height": "28px",
  "backgroundColor": "#FF5733",
  "cornerRadius": "6px"
}
```

---

## Carousel 多卡片設計

### 基礎 Carousel

```json
{
  "type": "flex",
  "altText": "多卡片訊息",
  "contents": {
    "type": "carousel",
    "contents": [
      { "type": "bubble", "body": { "..." : "卡片1" } },
      { "type": "bubble", "body": { "..." : "卡片2" } },
      { "type": "bubble", "body": { "..." : "卡片3" } }
    ]
  }
}
```

### 知識卡片導航系統

```
主選單
    │
    ▼
目錄頁（按分類 Carousel）
    │
    ├── 分類 A → [項目1] [項目2] [項目3] ...
    ├── 分類 B → [項目4] [項目5] [項目6] ...
    └── ...
          │
          ▼
    項目詳情（多卡片 Carousel）
          │
          ├── 卡片1: 基本資訊
          ├── 卡片2: 詳細特性
          ├── 卡片3: 延伸內容
          └── 卡片4: 導航按鈕
                      │
                      ├── [⬅️ 上一個] [➡️ 下一個]
                      ├── [📋 目錄]
                      └── [🏠 主選單]
```

---

## 導航按鈕設計

### 標準導航按鈕

```json
{
  "type": "separator",
  "margin": "lg"
},
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    {
      "type": "button",
      "action": {
        "type": "message",
        "label": "📋 目錄",
        "text": "目錄"
      },
      "style": "primary",
      "height": "sm",
      "flex": 1
    },
    {
      "type": "button",
      "action": {
        "type": "message",
        "label": "下一個 ➡️",
        "text": "下一個"
      },
      "style": "secondary",
      "height": "sm",
      "flex": 1,
      "margin": "sm"
    }
  ],
  "margin": "lg"
}
```

### 兩欄按鈕設計

當項目數量多時，用兩欄節省空間。

閱讀順序原則：必須是「由上到下」（先左欄再右欄），不是「由左至右」。

```
正確（由上到下）：        錯誤（由左至右）：
項目 1 | 項目 6          項目 1 | 項目 2
項目 2 | 項目 7          項目 3 | 項目 4
項目 3 | 項目 8          項目 5 | 項目 6
項目 4 | 項目 9
項目 5 | 項目 10
```

### 奇數項目處理

最後一個項目旁邊加 `filler` 佔位：

```json
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    {
      "type": "button",
      "action": {
        "type": "message",
        "label": "項目 11",
        "text": "項目 11"
      },
      "style": "secondary",
      "flex": 1
    },
    {
      "type": "filler",
      "flex": 1
    }
  ]
}
```

### Button label 太長的解法

label 最多 20 字元，超過要改用 Box + Text：

```json
{
  "type": "box",
  "layout": "vertical",
  "contents": [
    {
      "type": "text",
      "text": "這是一段很長的按鈕文字，可以自動換行",
      "wrap": true,
      "align": "center",
      "color": "#FFFFFF",
      "size": "sm"
    }
  ],
  "backgroundColor": "#1DB446",
  "cornerRadius": "md",
  "paddingAll": "md",
  "action": {
    "type": "message",
    "label": "action",
    "text": "觸發文字"
  }
}
```

---

## 圖片訊息規範

### LINE Bot 圖片限制

| 項目 | 限制 | 說明 |
|------|------|------|
| 檔案大小 | < 300KB | 超過會載入緩慢或失敗 |
| 圖片格式 | JPEG, PNG | 不支援 GIF 動圖 |
| 圖片來源 | HTTPS | 必須是安全連線 |
| Hero 比例 | 4:3 或 16:9 | Flex Message 推薦 |

### 手機閱讀限制

LINE Bot 圖片在手機上無法放大，必須確保：
- 字體夠大（手機直接可讀）
- 圖片不要太寬（手機螢幕約 360-400px）
- 重要資訊不要太密集

### 圖片快取問題

更新圖片後 LINE 仍顯示舊版時，URL 加版本參數：
```
https://example.com/images/card.png?v=2
```

---

## 常見錯誤與修正

### aspectMode 錯誤

LINE API 回傳 HTTP 400：
```json
{"message":"invalid property","property":"/contents/1/hero/aspectMode"}
```

原因：使用了 `"aspectMode": "contain"`。改為 `"cover"` 或 `"fit"`。

### Text 欄位不能為空

```json
// ❌ HTTP 400
{"type": "text", "text": ""}

// ✅ 用佔位符
{"type": "text", "text": "－"}
```

### 長文字被截斷

```json
// ❌ 長文字被截斷
{"type": "text", "text": "很長的內容..."}

// ✅ 允許折行
{"type": "text", "text": "很長的內容...", "wrap": true}
```

### 診斷 LINE API 錯誤

解讀 property 路徑：
- `/contents/0/body/contents/5/contents/1/text`
- → carousel 第 1 張卡片 → body → 第 6 個元素 → 第 2 個子元素 → text

### 常見 HTTP 狀態碼

| 狀態碼 | 原因 |
|--------|------|
| 400 | JSON 格式錯誤或欄位無效 |
| 401 | Channel Access Token 過期 |
| 429 | 請求次數超限 |

### 圖片不顯示

確認步驟：
1. 圖片 URL 是否可直接在瀏覽器開啟
2. URL 是否為 HTTPS
3. 程式碼是否重複加了 BASE_URL 前綴

判斷完整 URL 的邏輯：
```python
if url.startswith('http'):
    image_url = url
else:
    image_url = f"{IMAGE_BASE_URL}/{url}"
```
