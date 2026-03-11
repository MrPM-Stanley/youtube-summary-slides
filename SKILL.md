---
name: youtube-summary-slides
description: "YouTube 影片摘要投影片產生器。從 YouTube 影片自動產生繁體中文摘要投影片（.pptx），包含影片關鍵畫面截圖。務必在以下情境觸發此 skill：當使用者提供 YouTube 連結並要求摘要、做筆記、做投影片；當使用者說「幫我看這個影片」「幫我做影片摘要」「把影片整理成投影片」「summarize this video」；當使用者提到 YouTube 影片並希望產出投影片、簡報、筆記；即使使用者只是貼了一個 YouTube 連結並說「幫我整理一下」也應觸發。"
---

# YouTube 影片摘要投影片產生器

將 YouTube 影片轉化為一份圖文並茂的繁體中文摘要投影片。投影片數量等於影片分鐘數（例如 20 分鐘影片 = 20 張投影片），但不是機械式地每分鐘一張，而是根據內容的資訊密度和重要性來分配。

## 工作流程概覽

```
YouTube URL → 下載影片+字幕 → (若無字幕) Whisper 語音辨識 → 分析內容 → 擷取關鍵截圖 → 產生投影片
```

## 第一步：環境準備

確保以下工具可用：

```bash
export PATH="$PWD/.local/bin:$PATH"
# 檢查並安裝
pip install yt-dlp openai-whisper python-pptx Pillow --break-system-packages -q
```

ffmpeg 應該已經預裝。若沒有：`apt-get install -y ffmpeg`

## 第二步：下載影片與字幕

使用 yt-dlp 下載影片和字幕。優先下載既有字幕，同時下載影片檔（後續截圖用）。

```bash
WORK_DIR="$PWD/yt-work"
mkdir -p "$WORK_DIR"

# 先嘗試取得影片資訊（標題、時長等）
# 同時從 URL 中解析出 VIDEO_ID（例如從 https://www.youtube.com/watch?v=dQw4w9WgXcQ 取得 dQw4w9WgXcQ）
# VIDEO_ID 後續每張投影片的時間戳連結都會用到
yt-dlp --print "%(title)s|||%(duration)s|||%(id)s|||%(subtitles)s|||%(automatic_captions)s" --no-download "VIDEO_URL"

# 下載影片（720p 以下即可，截圖不需要太高解析度）
yt-dlp -f "bestvideo[height<=720]+bestaudio/best[height<=720]" \
  --merge-output-format mp4 \
  -o "$WORK_DIR/video.mp4" "VIDEO_URL"

# 嘗試下載字幕（優先手動字幕，其次自動字幕）
yt-dlp --write-sub --write-auto-sub --sub-lang "zh-Hant,zh-Hans,zh,en,ja,ko" \
  --sub-format "vtt/srt/best" --skip-download \
  -o "$WORK_DIR/video" "VIDEO_URL"
```

## 第三步：取得文字內容

### 情況 A：有字幕檔

如果上一步成功下載了字幕（.vtt 或 .srt 檔），解析字幕檔取得帶時間戳的文字內容。

用 Python 解析字幕：

```python
import re

def parse_vtt(filepath):
    """解析 VTT 字幕檔，回傳 [(start_seconds, text), ...]"""
    with open(filepath, 'r', encoding='utf-8') as f:
        content = f.read()

    segments = []
    # 匹配時間戳和文字
    pattern = r'(\d{2}):(\d{2}):(\d{2})[.,](\d{3})\s*-->\s*\d{2}:\d{2}:\d{2}[.,]\d{3}\s*\n(.+?)(?=\n\n|\n\d{2}:|\Z)'
    for match in re.finditer(pattern, content, re.DOTALL):
        h, m, s, ms = int(match.group(1)), int(match.group(2)), int(match.group(3)), int(match.group(4))
        start = h * 3600 + m * 60 + s + ms / 1000
        text = re.sub(r'<[^>]+>', '', match.group(5)).strip()
        if text and not text.startswith('WEBVTT'):
            segments.append((start, text))

    return segments
```

### 情況 B：沒有字幕檔

使用 Whisper 進行語音辨識。這會需要較長時間，但能產出帶時間戳的逐字稿。

```bash
whisper "$WORK_DIR/video.mp4" --model base --output_dir "$WORK_DIR" --output_format json
```

Whisper 的 JSON 輸出中，每個 segment 都包含 `start`、`end` 和 `text` 欄位，可以直接使用。

對於較長的影片（>30分鐘），可以先提取音訊再辨識，會快一些：

```bash
ffmpeg -i "$WORK_DIR/video.mp4" -vn -acodec pcm_s16le -ar 16000 -ac 1 "$WORK_DIR/audio.wav"
whisper "$WORK_DIR/audio.wav" --model base --output_dir "$WORK_DIR" --output_format json
```

## 第四步：分析內容並規劃投影片

拿到完整的帶時間戳文字後，進行內容分析：

1. **計算影片時長**（分鐘數），這就是目標投影片數量
2. **閱讀完整逐字稿**，識別：
   - 主題轉換點
   - 關鍵概念和定義
   - 重要數據或案例
   - 結論和行動建議
3. **規劃每張投影片**，決定：
   - 該張投影片涵蓋的時間範圍
   - 摘要標題（繁體中文）
   - 3-5 個重點（繁體中文）
   - 是否需要截圖（只在畫面有視覺意義時截圖，例如圖表、示範畫面、重要文字畫面）
   - 截圖的精確時間戳

### 投影片分配原則

- 資訊密度高的段落多分配投影片，純閒聊或重複的段落少分配
- 開頭和結尾通常各佔 1-2 張（引言 + 總結）
- 中間根據主題切換來分段
- 不是每張都需要截圖 — 只有畫面真的能幫助理解時才截

## 第五步：擷取關鍵截圖

根據上一步決定的時間戳，用 ffmpeg 截圖：

```bash
# 在特定時間點截圖（秒數）
ffmpeg -ss 125.5 -i "$WORK_DIR/video.mp4" -frames:v 1 -q:v 2 "$WORK_DIR/screenshot_001.jpg"
```

截圖命名建議使用 `screenshot_NNN.jpg`（NNN 對應投影片編號），方便後續對應。

### 截圖在投影片中的呈現規範

影片截圖幾乎都是 16:9 比例，嵌入投影片時必須**嚴格維持 16:9 比例**（寬:高 = 16:9），否則畫面會變形。具體做法是：決定寬度 `imgW` 後，高度一律用 `imgH = imgW * 9 / 16` 計算，絕對不要手動指定不符比例的高度。

截圖建議尺寸為 4.8~5.0 英吋寬（約佔投影片寬度的一半），靠右對齊放置。

文字區和截圖之間需要留出至少 **0.2 吋**的間距，避免文字緊貼截圖邊緣。具體做法：

```
截圖起始 X = 10（投影片寬）- imgW - 0.25（右邊距）
文字區寬度 = 截圖起始 X - 文字起始 X - 0.2（間距）
```

例如：截圖寬 4.875 吋 → 截圖起始 X ≈ 4.875 → 文字起始於 X=0.5 → 文字寬度 = 4.875 - 0.5 - 0.2 = 4.175 吋。

## 第六步：建立投影片

使用 pptx skill 的方式建立投影片。讀取 pptx skill 的 `pptxgenjs.md` 來了解如何用 PptxGenJS 建立精美投影片。

### 投影片結構

1. **封面投影片**：影片標題（繁體中文翻譯）、頻道名稱、影片時長、連結
2. **內容投影片**（N 張）：每張包含：
   - 繁體中文標題（精簡的主題句）
   - 3-5 個繁體中文重點摘要
   - 對應的影片截圖（若該段有視覺意義的畫面）
   - **YouTube 時間戳連結**：顯示格式為 `MM:SS ~ MM:SS`，其中起始時間是可點擊的超連結，點擊後直接在瀏覽器開啟 YouTube 影片並跳到該時間點播放。連結格式為 `https://www.youtube.com/watch?v=VIDEO_ID&t=XmYs`（例如影片 03:55 處 → `&t=3m55s`）。記得在第二步取得影片資訊時就要保存 VIDEO_ID，後續每張投影片都會用到。
3. **總結投影片**：全影片的關鍵要點回顧

### 設計要求

- 所有文字必須使用繁體中文（即使原始影片是其他語言）
- 截圖必須維持 16:9 原始比例（`imgH = imgW * 9 / 16`），絕對不可變形
- 截圖建議寬 4.8~5.0 吋，靠右對齊放置
- 文字區與截圖之間至少留 0.2 吋間距
- 有截圖的投影片用雙欄佈局（左文右圖），沒有的用全寬文字佈局
- 配色要跟影片主題相關，不要用預設的無聊藍色
- 時間標記用小字放在投影片底部，起始時間必須是可點擊的超連結（連到 YouTube 該時間點）

### 使用 PptxGenJS 範例

```javascript
const pptxgen = require("pptxgenjs");
let pres = new pptxgen();
pres.layout = 'LAYOUT_16x9';

// ===== 有截圖的投影片 =====
let slide = pres.addSlide();

// 截圖尺寸計算（嚴格維持 16:9 比例）
const imgW = 4.875;
const imgH = imgW * 9 / 16;  // ≈ 2.742，不要手動寫死高度
const imgX = 10 - imgW - 0.25;  // 靠右對齊
const imgY = 0.4;

// 文字區寬度 = 截圖起始位置 - 左邊距 - 間距（至少 0.2 吋）
const textX = 0.5;
const textW = imgX - textX - 0.2;  // ≈ 4.175

// 左側文字
slide.addText("投影片標題", {
  x: textX, y: 0.3, w: textW, h: 0.7,
  fontSize: 24, bold: true, color: "2C3E50"
});
slide.addText([
  { text: "重點一：說明內容", options: { bullet: true, breakLine: true, fontSize: 13 } },
  { text: "重點二：說明內容", options: { bullet: true, breakLine: true, fontSize: 13 } },
  { text: "重點三：說明內容", options: { bullet: true, fontSize: 13 } }
], { x: textX, y: 1.2, w: textW, h: 3.5, valign: "top" });

// 右側截圖（白色背景 + 陰影框）
slide.addShape(pres.shapes.RECTANGLE, {
  x: imgX - 0.05, y: imgY - 0.05, w: imgW + 0.1, h: imgH + 0.1,
  fill: { color: "FFFFFF" },
  shadow: { type: "outer", color: "000000", blur: 4, offset: 2, opacity: 0.1 }
});
slide.addImage({ path: "screenshot_001.jpg", x: imgX, y: imgY, w: imgW, h: imgH });

// 底部 YouTube 時間戳連結
// VIDEO_ID 在第二步從 URL 中解析取得（例如 "dQw4w9WgXcQ"）
const startLink = `https://www.youtube.com/watch?v=${VIDEO_ID}&t=3m25s`;
slide.addText([
  { text: "03:25", options: { fontSize: 10, color: "3498DB", hyperlink: { url: startLink } } },
  { text: " ~ 07:10", options: { fontSize: 10, color: "999999" } }
], { x: 0.5, y: 5.1, w: 3, h: 0.4 });

pres.writeFile({ fileName: "summary.pptx" });
```

## 第七步：品質檢查

1. 確認投影片數量是否約等於影片分鐘數
2. 確認所有文字都是繁體中文
3. 確認截圖正確嵌入且不重疊文字
4. 確認每張內容投影片底部的時間戳連結可正確點擊，連結格式為 `https://www.youtube.com/watch?v=VIDEO_ID&t=XmYs`
5. 使用 pptx skill 的 QA 流程檢查視覺品質

## 注意事項

- 如果影片很長（>30 分鐘），Whisper 在純 CPU 環境下可能逾時。建議先提取音訊為 WAV（16kHz mono），再切成 10 分鐘的分段（chunk）分別辨識，最後合併並修正時間偏移（offset = chunk_index × 600 秒）。長影片優先使用 `tiny` 模型以避免逾時
- yt-dlp 有時需要更新：`pip install -U yt-dlp --break-system-packages`
- 截圖時避開轉場畫面，選擇穩定且有資訊量的畫面
- 如果影片是英文或其他語言，所有摘要文字都要翻譯成繁體中文
- 截圖不需要每張投影片都有 — 只在畫面真的有幫助理解的時候才放
