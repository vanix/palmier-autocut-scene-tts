# palmier-autocut-scene-tts

AI Agent 技能（Skill）——將原始素材（MP4 + HEIC）自動切分成獨立場景，配上視覺描述和口語旁白，輸出可直接餵給 [Palmier Pro](https://palmier.pro) MCP 使用的腳本。
專為 OpenCode 設計。

## 功能

當使用者說出「幫我切場景剪片、切場景剪片、場景剪輯、剪片、開箱影片、旅遊影片剪輯」等關鍵字時自動觸發，執行完整場景式剪片流程：

| Phase | 內容 |
|:-----:|:------|
| 1. 問卷 | 素材路徑、跳過檔案、場景秒數、旁白風格、TTS 語音、BGM 風格 |
| 2. 場景切割 | ffmpeg scene detect → `temp/scenes/` + `temp/manifest.json` |
| 3. 視覺描述 | Qwen2.5VL 批次描述每場景內容 |
| 4. 旁白撰寫 | opencode 依篩選規則 + 口語風格寫旁白 |
| 4b. TTS 合成 | edge-tts 生成每場景旁白音檔 → `temp/tts/` |
| 4c. BGM | CC0 背景音樂搜尋與下載 → `temp/bgm/`（可選） |
| 5. 產出 | `palmier_script.json` + `subtitles.srt` + `script_narrative.txt` |

## 安裝

```bash
# 1. 下載 skill
git clone https://github.com/vanix/palmier-autocut-scene-tts.git ~/.config/opencode/skills/palmier-autocut-scene-tts

# 2. 安裝 ffmpeg
brew install ffmpeg

# 3. 安裝 ollama 並下載視覺模型
brew install ollama
ollama pull qwen2.5vl:7b

# 4. 啟動 ollama（背景執行）
ollama serve &
```

## 使用方式

在 opencode 中載入 skill：

```
幫我切場景剪片
```

依序回答問卷，剩下交給流程自動完成。

## 互動問卷（4 題）

| 代號 | 問題 | 預設值 |
|:----:|:-----|:------:|
| ANS_1 | 素材資料夾路徑？ | `~/Desktop/日航商務艙初體驗/` |
| ANS_2 | 跳過的檔案？（逗號分隔，或留空） | `商務艙開箱_1.MP4` |
| ANS_3 | 每個場景固定秒數？ | `5` |
| ANS_4 | 旁白口吻偏好？ | `輕鬆` |
| ANS_5 | TTS 語音？（男聲 / 女聲） | `男聲` |
| ANS_6 | 背景音樂風格？（不需要 / 輕鬆 / 活潑 / 寧靜 / 自訂） | `不需要` |

## 輸出產物

| 檔案 | 格式 | 用途 |
|:----|:----|:------|
| `palmier_script.json` | JSON | 餵給 Palmier Pro MCP（`add_clips` + `add_texts`） |
| `subtitles.srt` | SRT | 標準字幕檔，可匯入任何剪輯軟體 |
| `script_narrative.txt` | 純文字 | 給人看的剪輯腳本，含場景順序與旁白 |

## 工作流程

```
素材資料夾
  │
  ▼
Phase 1 ─ 問卷（路徑、跳過、秒數、風格、TTS 語音、BGM 風格）
  │
  ▼
Phase 2 ─ ffmpeg 場景偵測 → temp/scenes/ + temp/manifest.json
  │
  ▼
Phase 3 ─ Qwen2.5VL 視覺描述 → 寫入 temp/manifest.json
  │
  ▼
Phase 4 ─ opencode 篩選場景 + 撰寫口語旁白 → 寫入 temp/manifest.json
  │
  ▼
Phase 4b ─ edge-tts 語音合成 → temp/tts/*.mp3
  │
  ▼
Phase 4c ─ CC0 BGM 搜尋下載 → temp/bgm/bgm.mp3（可選）
  │
  ▼
Phase 5 ─ 輸出 palmier_script.json + subtitles.srt + script_narrative.txt
  │
  ▼
匯入 Palmier Pro → add_clips 上片 → add_texts 上字幕
→ import_media TTS → add_clips 旁白音軌
→ import_media BGM → add_clips 背景音軌（音量 0.08）→ 輸出成品
```

## 關鍵原則

- **HEIC 不轉檔**：Palmier Pro 原生支援 HEIC 匯入
- **場景檔名規則**：`scene_序號_原始檔名.mp4`
- **視覺模型誤判 → 跳過**：Qwen 描述與畫面不符時，不硬湊旁白
- **同來源多段重複 → 只留關鍵**：如同支手機錄影拍 10 段 App 畫面，只保留 2-3 個代表性片段
- **旁白輕鬆口語**：像在跟朋友聊天，不用書面語
- **暫存進 temp/**：過程中的場景影片、縮圖、腳本、TTS 音檔一律放 `temp/` 下，根目錄只留原始檔與最終產出

## 依賴套件

| 套件 | 用途 | 安裝方式 |
|:----|:-----|:---------|
| ffmpeg | 場景偵測、影片切割、縮圖 | `brew install ffmpeg` |
| ollama | 視覺模型（qwen2.5vl:7b） | `brew install ollama && ollama pull qwen2.5vl:7b` |
| edge-tts | 旁白語音合成 | `pip install edge-tts` |
| websearch | BGM/CC0 音樂搜尋 | opencode 內建 |
| Python 3 | 執行腳本 | 內建於 macOS，或 `brew install python` |
| sips | HEIC 縮圖（macOS 內建） | 已內建 |

## 產出檔案一覽

執行完成後，素材資料夾內會產生：

```
素材資料夾/
├── palmier_script.json        ← → 給 Palmier Pro MCP（含 tts_file + bgm）
├── subtitles.srt               ← → 字幕檔
├── script_narrative.txt       ← → 給人看的腳本
└── temp/                      ← 暫存檔（可清空）
    ├── manifest.json          ← 場景清單（含描述、旁白、TTS、BGM）
    ├── scripts/               ← Python 腳本
    │   ├── run_scene_detect.py
    │   ├── qwen_describe.py
    │   ├── generate_tts.py
    │   └── export_scripts.py
    ├── scenes/                ← 切割後的獨立場景 MP4
    ├── thumbs/                ← 480px 縮圖
    ├── tts/                   ← edge-tts 旁白音檔 MP3
    └── bgm/                   ← CC0 背景音樂 MP3
```

## 授權

MIT License
