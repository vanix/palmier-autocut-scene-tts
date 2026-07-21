# palmier-autocut-scene-tts

AI Agent 技能（Skill）—— 將原始素材（MP4 + HEIC）自動切分成獨立場景，Qwen2.5VL 視覺描述 + 口語旁白 + TTS 語音合成，輸出可直接餵給 [Palmier Pro](https://palmier.pro) MCP 剪輯的完整影片。

## 功能

| Phase | 內容 |
|:-----:|:------|
| 1. 問卷 | 素材路徑、場景秒數、旁白風格、TTS 引擎、BGM 風格 |
| 2. 場景切割 | ffmpeg scene detect → `temp/scenes/` + `temp/manifest.json` |
| 3. 視覺描述 | Qwen2.5VL 批次描述每場景內容 |
| 3b. 過濾 | 晃動／重複／無關場景自動跳過 |
| 4. 旁白撰寫 | 15-25 字輕鬆通俗口語旁白 |
| 4b. TTS 合成 | Edge TTS（男/女聲）或 台灣藍鵲 TTS（李宏毅/女聲/自訂 .pt） |
| 4c. BGM | CC0 背景音樂搜尋與下載 → `temp/bgm/`（可選） |
| 5. 輸出 | `subtitles.srt` + `script_narrative.txt` + `temp/tts/` TTS 音檔 |
| 6. 剪輯 | 用 MCP 呼叫 Palmier Pro 自動上片、上字幕、上 TTS + BGM、輸出 MP4 |

## 安裝

```bash
# 1. 下載 skill
git clone https://github.com/vanix/palmier-autocut-scene-tts.git ~/.config/opencode/skills/palmier-autocut-scene-tts

# 2. 安裝 ffmpeg
brew install ffmpeg

# 3. 安裝 ollama 並下載視覺模型
brew install ollama
ollama pull qwen2.5vl:7b

# 4. 啟動 ollama
ollama serve &

# 5. Edge TTS（選用）
pip install edge-tts

# 6. 台灣藍鵲 TTS（選用）
pip install bluemagpie-tts soundfile
```

## 使用方式

在 opencode 中：

```
幫我剪片，素材在 ~/Desktop/日航商務艙初體驗/
```

依序回答問卷，過程中旁白寫完後會暫停讓您確認修改，確認後自動完成 TTS → BGM → Palmier Pro 剪輯 → 輸出 MP4。

## 互動問卷

| 代號 | 問題 | 預設值 |
|:----:|:-----|:------:|
| ANS_1 | 素材資料夾路徑？ | `~/Desktop/影片主題資料夾` |
| ANS_2 | 最大場景秒數？ | `5`（3~5s 動態） |
| ANS_3 | 旁白口吻偏好？ | `輕鬆通俗幽默` |
| ANS_4 | TTS 引擎？ | `Edge TTS` |
| ANS_4a | 語音性別？（Edge TTS） | `男聲` |
| ANS_4b | 聲音選擇？（藍鵲 TTS） | `李宏毅老師` |
| ANS_4c | .pt 路徑？（自訂聲音） | `~/Desktop/剪片ing/自己的聲音向量/my_voice.pt` |
| ANS_5 | 背景音樂風格？ | `輕鬆` |

## 輸出產物

| 檔案 | 用途 |
|:----|:------|
| `日航商務艙初體驗-v?.mp4` | 最終影片（經 Palmier Pro MCP 剪輯輸出） |
| `subtitles.srt` | 標準字幕檔 |
| `script_narrative.txt` | 給人看的剪輯腳本（若要修改旁白，直接編輯 `manifest.json` 的 `caption_short`） |
| `temp/manifest.json` | **管線核心資料檔**，各階段讀寫傳遞 |

## 關鍵更新

### v2 新增功能

- **旁白檢查點**：Phase 4 完成後暫停，輸出旁白表供使用者確認修改
- **音檔 loudnorm 統一音量**：所有 TTS 正規化至 -16 LUFS
- **TTS 淡入 0.5s**：每段旁白開頭平滑切入
- **台灣藍鵲 TTS + 自訂聲音向量**：支援 `my_voice.pt`
- **Palmier Pro 自動剪輯**：直接透過 MCP 建立專案、上片、上字幕、上音軌、輸出 MP4
- **TTS / BGM 分軌**：TTS 放 A1，BGM 放 A2（vol 0.08 + 淡入淡出）

## 工作流程

```
素材資料夾
  │
  ▼
Phase 1 ─ 問卷（路徑、秒數、風格、TTS 引擎、BGM 風格）
  │
  ▼
Phase 2 ─ ffmpeg 場景偵測 → temp/scenes/ + temp/manifest.json
  │
  ▼
Phase 3 ─ Qwen2.5VL 視覺描述 → 寫入 manifest.json
  │
  ▼
Phase 3b ─ 晃動／重複／無關場景自動跳過
  │
  ▼
Phase 4 ─ 撰寫 15-25 字輕鬆口語旁白 → 寫入 manifest.json
  │
  ▼
✋ 旁白檢查點 ─ 輸出旁白表，使用者確認後繼續
  │
  ▼
Phase 4b ─ TTS 合成 → loudnorm 統一音量 → 淡入 0.5s → temp/tts/
  │
  ▼
Phase 4c ─ CC0 BGM 搜尋下載 → temp/bgm/bgm.mp3（可選）
  │
  ▼
Phase 5 ─ 輸出 subtitles.srt + script_narrative.txt
  │
  ▼
Phase 6 ─ Palmier Pro MCP 自動剪輯
       ├─ 開專案、匯入素材
       ├─ add_clips 上片
       ├─ add_texts 上字幕（白字 48pt 置中下方）
       ├─ add_clips TTS → A1 音軌
       ├─ add_clips BGM → A2 音軌（vol 0.08 + 2s 淡入淡出）
       └─ export_project → MP4
```

## 依賴套件

| 套件 | 用途 |
|:----|:-----|
| ffmpeg | 場景偵測、影片切割、縮圖、loudnorm、afade |
| ollama + qwen2.5vl:7b | 視覺描述 |
| edge-tts | Edge TTS 語音合成 |
| bluemagpie-tts + soundfile | 台灣藍鵲 TTS（選用） |
| Python 3 | 執行腳本 |
| sips | HEIC 縮圖（macOS 內建） |

## 授權

MIT License
