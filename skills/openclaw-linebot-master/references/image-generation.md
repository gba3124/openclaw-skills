# LINE Bot 圖片生成連續技

結合 Gemini AI 生成高品質底圖 + matplotlib 加上正確中文標籤的連續技，專為 LINE Bot 圖片設計。

---

## 目錄

1. [為什麼需要連續技](#為什麼需要連續技)
2. [LINE Bot 圖片規格](#line-bot-圖片規格)
3. [連續技流程](#連續技流程)
4. [matplotlib 基礎設定](#matplotlib-基礎設定)
5. [壓縮與儲存](#壓縮與儲存)
6. [中文字體處理](#中文字體處理)
7. [Breaking Bad 風格範例](#breaking-bad-風格範例)
8. [常見問題](#常見問題)

---

## 為什麼需要連續技

| 方法 | 優點 | 缺點 |
|------|------|------|
| 純 matplotlib | 中文標籤正確 | 圖形簡陋、不夠精緻 |
| 純 Gemini | 圖片品質高 | 中文常出錯（亂碼） |
| **連續技** | 品質高 + 中文正確 | 需要兩步驟 |

---

## LINE Bot 圖片規格

### 硬性限制

| 項目 | 限制 | 說明 |
|------|------|------|
| **檔案大小** | **< 300KB** | 超過會載入緩慢或失敗 |
| 圖片格式 | JPEG, PNG | 不支援 GIF 動圖 |
| 圖片來源 | HTTPS | 必須是安全連線 |
| Hero 比例 | 4:3 或 16:9 | Flex Message 推薦 |

### 手機閱讀限制

LINE Bot 圖片在手機上無法放大。必須確保：
- 字體夠大（手機上直接可讀）
- 圖片不要太寬（手機螢幕約 360-400px）
- 重要資訊不要太密集

### 優化後的尺寸標準

| 用途 | figsize | DPI | 輸出尺寸 | 檔案大小 |
|------|---------|-----|----------|----------|
| **題庫圖片** | (8, 6) | 120 | ~960×720px | < 150KB |
| **筆記圖片** | (8, 8) | 120 | ~960×960px | < 200KB |
| **橫向圖表** | (10, 6) | 120 | ~1200×720px | < 180KB |
| 大型圖 | (10, 8) | 100 | ~1000×800px | < 250KB |

### 字體大小標準（手機可讀）

```python
FONT_TITLE = 24      # 圖片標題
FONT_LABEL = 18      # 標籤文字（主要內容）
FONT_SUBLABEL = 14   # 次要標籤
FONT_NOTE = 12       # 註解（最小值，謹慎使用）
```

---

## 連續技流程

### Step 1：Gemini 生成無標籤底圖

Prompt 關鍵技巧：加上 `WITHOUT any text labels` 避免中文亂碼。

```python
from google import genai
from google.genai import types

client = genai.Client(api_key=API_KEY)

prompt = """A detailed diagram of [主題].
Clean illustration style WITHOUT any text labels or annotations.
High quality educational diagram, white background."""

response = client.models.generate_content(
    model="gemini-2.0-flash-exp",
    contents=prompt,
    config=types.GenerateContentConfig(
        response_modalities=['image', 'text']
    )
)

for part in response.candidates[0].content.parts:
    if part.inline_data is not None:
        with open("base_image.png", 'wb') as f:
            f.write(part.inline_data.data)
```

Prompt 範例：

```
✅ 正確：
"A detailed anatomical diagram of the human heart showing four chambers.
 Clean medical illustration style WITHOUT any text labels.
 High quality, white background."

❌ 錯誤：
"心臟解剖圖，標示中文說明"  # 中文標籤會出錯
```

### Step 2：matplotlib 加中文標籤

```python
import matplotlib.pyplot as plt
import matplotlib.image as mpimg

plt.rcParams['font.sans-serif'] = ['Microsoft JhengHei', 'SimHei', 'Arial Unicode MS']
plt.rcParams['axes.unicode_minus'] = False

def add_labels_for_linebot(base_image_path, output_path, labels, title):
    """
    為 Gemini 底圖加上中文標籤（LINE Bot 優化版）

    Parameters:
    - base_image_path: Gemini 生成的底圖路徑
    - output_path: 輸出路徑
    - labels: [(標籤x%, 標籤y%, 文字, 箭頭x%, 箭頭y%), ...]
    - title: 圖片標題
    """
    img = mpimg.imread(base_image_path)
    fig, ax = plt.subplots(figsize=(8, 6))
    ax.imshow(img)
    ax.axis('off')

    h, w = img.shape[:2]

    for lx, ly, text, ax_x, ax_y in labels:
        bbox_props = dict(
            boxstyle='round,pad=0.3',
            facecolor='white',
            edgecolor='#333333',
            alpha=0.92,
            linewidth=1.5
        )
        ax.annotate(
            text,
            xy=(ax_x * w, ax_y * h),
            xytext=(lx * w, ly * h),
            fontsize=16,
            fontweight='bold',
            ha='center', va='center',
            bbox=bbox_props,
            arrowprops=dict(
                arrowstyle='-|>',
                color='#333333',
                lw=1.5,
                connectionstyle='arc3,rad=0.1'
            )
        )

    ax.set_title(title, fontsize=20, fontweight='bold', pad=10)
    save_for_linebot(fig, output_path)
    plt.close()
```

---

## matplotlib 基礎設定

```python
# -*- coding: utf-8 -*-
import matplotlib.pyplot as plt
import matplotlib.patches as patches
import os

plt.rcParams['font.sans-serif'] = ['Microsoft JhengHei', 'SimHei', 'Arial Unicode MS']
plt.rcParams['axes.unicode_minus'] = False
```

### 純 matplotlib 圖片尺寸

| 用途 | figsize | DPI | 輸出尺寸 |
|------|---------|-----|----------|
| 一般圖片 | (14, 10) | 150 | ~2100×1500px |
| 週期表 | (14, 7) | 120 | ~1680×840px |
| 小圖示 | (2, 2) | 150 | ~300×300px |

### 字體大小（純 matplotlib 用）

```python
FONT_TITLE = 30   # 主標題
FONT_LARGE = 22   # 雙字元符號
FONT_MEDIUM = 18  # 一般文字
FONT_SMALL = 14   # 次要資訊（最小值）
```

---

## 壓縮與儲存

### 自動壓縮函式

```python
from PIL import Image
import io

def save_for_linebot(fig, output_path, max_size_kb=280):
    """儲存圖片並確保 < 300KB"""
    buf = io.BytesIO()
    fig.savefig(buf, format='png', dpi=120, bbox_inches='tight',
                facecolor='white', pad_inches=0.05)
    buf.seek(0)

    size_kb = len(buf.getvalue()) / 1024

    if size_kb <= max_size_kb:
        with open(output_path, 'wb') as f:
            f.write(buf.getvalue())
        print(f"已儲存: {output_path} ({size_kb:.1f} KB)")
    else:
        img = Image.open(buf)
        if img.mode == 'RGBA':
            img = img.convert('RGB')

        for quality in [85, 75, 65, 55]:
            jpeg_buf = io.BytesIO()
            img.save(jpeg_buf, format='JPEG', quality=quality, optimize=True)
            jpeg_size = len(jpeg_buf.getvalue()) / 1024

            if jpeg_size <= max_size_kb:
                jpeg_path = output_path.rsplit('.', 1)[0] + '.jpg'
                with open(jpeg_path, 'wb') as f:
                    f.write(jpeg_buf.getvalue())
                print(f"已儲存: {jpeg_path} ({jpeg_size:.1f} KB, quality={quality})")
                return

        img.thumbnail((800, 800), Image.LANCZOS)
        jpeg_buf = io.BytesIO()
        img.save(jpeg_buf, 'JPEG', quality=70, optimize=True)
        jpeg_path = output_path.rsplit('.', 1)[0] + '.jpg'
        with open(jpeg_path, 'wb') as f:
            f.write(jpeg_buf.getvalue())
        print(f"已儲存（縮小）: {jpeg_path} ({len(jpeg_buf.getvalue())/1024:.1f} KB)")
```

### 壓縮策略（由輕到重）

1. 降低 DPI：`dpi=100` 或 `dpi=80`
2. 縮小 figsize：`figsize=(6, 4)`
3. 轉 JPEG：`quality=75`
4. 縮小尺寸：`img.thumbnail((800, 800))`

---

## 中文字體處理

### 標準中文字體

```python
plt.rcParams['font.sans-serif'] = ['Microsoft JhengHei', 'SimHei', 'Arial Unicode MS']
```

### Noto Sans CJK TC（罕見字支援）

微軟正黑體不支援某些罕見中文字（如化學元素 105-118）。改用 Noto Sans CJK TC：

```python
import matplotlib.font_manager as fm
import matplotlib.patheffects as path_effects

NOTO_FONT_PATH = "/path/to/NotoSansCJKtc-Regular.otf"
noto_font = fm.FontProperties(fname=NOTO_FONT_PATH)

ax.text(0.95, 0.08, "罕見字",
        fontsize=48, color='white', fontweight='bold',
        fontproperties=noto_font,
        path_effects=[
            path_effects.withStroke(linewidth=3, foreground='black')
        ])
```

重點：
- 使用 `fontproperties=` 參數，不是 `fontfamily=`
- `path_effects.withStroke()` 可加黑邊讓白字更清晰
- 字體檔約 16MB，不要放進 Git repo

### 特殊字元替換

某些 Unicode 字元在微軟正黑體中顯示為方框：

| 問題字符 | 解決方案 |
|----------|----------|
| − (U+2212) | 改用 `-` (普通連字符) |
| ₂ (U+2082) | 改用 `2` (普通數字) |
| ₃ (U+2083) | 改用 `3` (普通數字) |

---

## Breaking Bad 風格範例

### Prompt 範例

```python
def generate_prompt(symbol, rest_name, atomic_num, atomic_mass, color_hex):
    return f"""Create a Breaking Bad TV series style periodic table element card.

CRITICAL REQUIREMENTS:
1. BACKGROUND: Dark navy blue (#1a1a2e) with {color_hex} colored smoke/vapor.
2. ELEMENT BOX (center-left):
   - Green square box (#00A300) with white border (3px)
   - Large white letter '{symbol}' in the center (bold, sans-serif)
   - Small white number '{atomic_num}' in top-right corner
3. TEXT (to the right of the box):
   - '{rest_name}' in white, same baseline as the symbol
   - Below that: '{atomic_mass}' in smaller white text
4. LAYOUT: 2:1 aspect ratio (wide format)
5. STYLE: Cinematic, dramatic lighting

DO NOT add any other text, watermarks, or decorations."""
```

### 後處理：裁切 + 中文標籤 + 壓縮

```python
def post_process_image(input_path, output_path, name_zh, target_width=1200):
    """後處理：裁切成 2:1、加中文標籤、壓縮到 < 300KB"""
    from PIL import Image

    img = Image.open(input_path)
    width, height = img.size

    target_height = width // 2
    if height > target_height:
        top = (height - target_height) // 2
        img = img.crop((0, top, width, top + target_height))

    target_height = target_width // 2
    img = img.resize((target_width, target_height), Image.Resampling.LANCZOS)

    if img.mode != 'RGB':
        background = Image.new('RGB', img.size, (26, 26, 46))
        if img.mode == 'RGBA':
            background.paste(img, mask=img.split()[3])
        else:
            background.paste(img)
        img = background

    fig, ax = plt.subplots(figsize=(12, 6), dpi=100)
    ax.imshow(img)
    ax.axis('off')

    noto_font = fm.FontProperties(fname=NOTO_FONT_PATH)
    ax.text(0.95, 0.08, name_zh,
            transform=ax.transAxes,
            fontsize=48, color='white', fontweight='bold',
            ha='right', va='bottom',
            fontproperties=noto_font,
            path_effects=[
                path_effects.withStroke(linewidth=3, foreground='black')
            ])

    plt.tight_layout(pad=0)
    plt.savefig(output_path, dpi=100, bbox_inches='tight', pad_inches=0,
                format='jpeg', quality=85)
    plt.close()
```

輸出規格：
- 尺寸：1200×600px（2:1 比例）
- 檔案大小：~80-150KB（符合 LINE Bot 限制）

---

## 圖片命名規範

### 題庫圖片
```
ch{章}-{節}-q{題號}-{描述}.png      # 題目圖
ch{章}-{節}-a{題號}-{描述}.png      # 答案解析圖
```

### 筆記圖片
```
{主題}_{內容}.png
例：heart_anatomy.png, heart_conduction.png
```

### 元素圖片
```
periodic_table_{symbol}.png    # 週期表（高亮該元素）
electron_{symbol}.png          # 價電子圖
{element}_{topic}.png          # 內容圖片
```

---

## 常見問題

### 圖片太大（> 300KB）

依序嘗試：
1. 降低 DPI → `dpi=100`
2. 縮小 figsize → `figsize=(6, 4)`
3. 轉 JPEG → `quality=75`
4. 縮小尺寸 → `img.thumbnail((800, 800))`

### 字體太小

- 標籤字體至少 `fontsize=14`
- 標題字體至少 `fontsize=18`
- 避免密集標籤，必要時分多張圖

### Gemini 中文出錯

- Prompt 一律加 `WITHOUT any text labels`
- 所有中文用 matplotlib 後製

### LINE 圖片快取

更新圖片後 URL 加版本參數：
```
https://example.com/images/card.png?v=2
```

### Windows Console 編碼問題

```python
import sys
if sys.platform == 'win32':
    sys.stdout.reconfigure(encoding='utf-8', errors='replace')
```
