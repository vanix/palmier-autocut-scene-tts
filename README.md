# palmier-autocut-scene-tts

場景式剪片輔助工具。將原始素材（MP4 + HEIC）自動切分成獨立場景，配上視覺描述和口語旁白，輸出可直接餵給 Palmier Pro 使用的腳本。

## 適用情境

- 旅遊開箱 / 飯店體驗 / 活動記錄
- 多段零碎片段需要整理成連貫影片
- 需要旁白字幕但不想自己打逐字稿

## 輸出產物

| 檔案 | 格式 | 用途 |
|:----|:----|:------|
| `palmier_script.json` | JSON | 餵給 Palmier Pro MCP 工具 (`add_clips` + `add_texts`) |
| `subtitles.srt` | SRT | 標準字幕檔，可匯入任何剪輯軟體 |
| `script_narrative.txt` | 純文字 | 給人看的剪輯腳本，含場景順序與旁白 |

## 工作流程

```
素材資料夾
  │
  ▼
Phase 1 ─ ffmpeg 場景偵測 → 獨立場景檔案 + manifest.json
  │
  ▼
Phase 2 ─ Qwen2.5VL 視覺描述 → 寫入 manifest.json
  │
  ▼
Phase 3 ─ opencode 篩選場景 + 撰寫口語旁白 → 寫入 manifest.json
  │
  ▼
Phase 4 ─ 輸出 palmier_script.json + subtitles.srt + script_narrative.txt
  │
  ▼
匯入 Palmier Pro → add_clips 上片 → add_texts 上字幕 → 輸出成品
```

## 問卷

載入 skill 後會詢問：

1. **素材資料夾路徑？**（預設 `~/Desktop/日航商務艙初體驗/`）
2. **跳過的檔案？**（多檔逗號分隔，或留空）
3. **每個場景固定秒數？**（預設 `5`）
4. **旁白口吻偏好？**（預設 `輕鬆`）

## 依賴

- **ffmpeg** — 場景偵測與影片切割
- **ollama** — 視覺模型（預設 qwen2.5vl:7b）
- **Python 3** — 執行腳本

## 使用方式

在 opencode 中載入 skill：

```
請載入 palmier-autocut-scene-tts
```

依序回答問卷，剩下交給流程自動完成。
