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
export PATH="/sessions/optimistic-happy-goodall/.local/bin:$PATH"
# 檢查並安裝
pip install yt-dlp openai-whisper python-pptx Pillow --break-system-packages -q
```

ffmpeg 應該已經預裝。若沒有：`apt-get install -y ffmpeg`

## 第二步：下載影片與字幕

使用 yt-dlp 下載影片和字幕。優先下載既有字幕，同時下載影片檔（後續截圖用）。

```bash
WORK_DIR="/sessions/optimistic-happy-goodall/yt-work"
mkdir -p "$WORK_DIR"

# 先嘗試取得影片資訊（標題、時長等）
yt-dlp --print "%(title)s|||%(duration)s|||%(subtitles)s|||%(automatic_captions)s" --no-download "VIDEO_URL"

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

## 第六步：建立投影片

使用 pptx skill 的方式建立投影片。讀取 pptx skill 的 `pptxgenjs.md` 來了解如何用 PptxGenJS 建立精美投影片。

### 投影片結構

1. **封面投影片**：影片標題（繁體中文翻譯）、頻道名稱、影片時長、連結
2. **內容投影片**（N 張）：每張包含：
   - 繁體中文標題（精簡的主題句）
   - 3-5 個繁體中文重點摘要
   - 對應的影片截圖（若該段有視覺意義的畫面）
   - 時間標記（例如 `03:25 - 07:10`）
3. **總結投影片**：全影片的關鍵要點回顧

### 設計要求

- 所有文字必須使用繁體中文（即使原始影片是其他語言）
- 截圖放在投影片右側或下方，不要遮擋文字
- 有截圖的投影片用雙欄佈局（左文右圖）
- 沒有截圖的投影片用全寬文字佈局
- 配色要跟影片主題相關，不要用預設的無聊藍色
- 時間標記用小字放在投影片底部

### 使用 PptxGenJS 範例

```javascript
const pptxgen = require("pptxgenjs");
let pres = new pptxgen();
pres.layout = 'LAYOUT_16x9';

// 有截圖的投影片
let slide = pres.addSlide();
// 左側文字
slide.addText("投影片標題", { x: 0.5, y: 0.3, w: 5, h: 0.8, fontSize: 28, bold: true, color: "2C3E50" });
slide.addText([
  { text: "• 重點一：說明內容", options: { breakLine: true, fontSize: 14 } },
  { text: "• 重點二：說明內容", options: { breakLine: true, fontSize: 14 } },
  { text: "• 重點三：說明內容", options: { fontSize: 14 } }
], { x: 0.5, y: 1.3, w: 5, h: 3 });
// 右側截圖
slide.addImage({ path: "screenshot_001.jpg", x: 5.8, y: 0.5, w: 3.8, h: 2.85 });
// 底部時間標記
slide.addText("03:25 - 07:10", { x: 0.5, y: 5.1, w: 3, h: 0.4, fontSize: 10, color: "999999" });

pres.writeFile({ fileName: "summary.pptx" });
```

## 第七步：品質檢查

1. 確認投影片數量是否約等於影片分鐘數
2. 確認所有文字都是繁體中文
3. 確認截圖正確嵌入且不重疊文字
4. 使用 pptx skill 的 QA 流程檢查視覺品質

## 注意事項

- 如果影片很長（>60 分鐘），Whisper 辨識可能需要較長時間，使用 `base` 模型可以加速
- yt-dlp 有時需要更新：`pip install -U yt-dlp --break-system-packages`
- 截圖時避開轉場畫面，選擇穩定且有資訊量的畫面
- 如果影片是英文或其他語言，所有摘要文字都要翻譯成繁體中文
- 截圖不需要每張投影片都有 — 只在畫面真的有幫助理解的時候才放
