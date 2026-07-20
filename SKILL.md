---
name: palmier-autocut-scene-tts
description: >
  Scene-based 剪片流程：ffmpeg 場景偵測 → Qwen2.5VL 視覺描述 →
  opencode 旁白撰寫 → edge-tts 語音合成 →
  輸出 Palmier Script JSON + SRT + 腳本文字檔 + TTS 音檔。
  適用於旅遊、開箱、活動記錄等場景式素材。不需使用者具備技術知識。
  Trigger keywords: 幫我切場景剪片, 切場景剪片, 場景剪輯, 剪片, scene editing,
  開箱影片, 旅遊影片剪輯, 自動旁白
---

# palmier-autocut-scene-tts

## 鐵則（違反會產出爛結果，使用者會生氣）

| # | 規則 | 說明 |
|:-:|:----|:------|
| 1 | **場景檔名格式固定** | `scene_三位數序號_原始檔名（去副檔名）.mp4`，例如 `scene_001_0_0貴賓室位置.mp4` |
| 2 | **HEIC / 靜態圖不轉檔** | 直接記錄來源路徑。Palmier Pro 原生支援 HEIC 匯入 |
| 3 | **視覺模型明顯誤判 → 跳過** | Qwen 描述與畫面不符時（例如「展覽館門口」實際是點餐櫃檯），不硬湊旁白，以檔名為準重寫。仍無法判斷則跳過 |
| 4 | **旁白輕鬆口語** | 像在跟朋友聊天，不用書面語或旅遊雜誌體。善用檔名梗 |
| 5 | **同一來源多段子場景內容重複 → 只留關鍵** | 如同支手機錄影拍 10 段 App 畫面，只保留 2-3 個代表性片段 |
| 6 | **暫存檔全進 `temp/`** | 場景影片、縮圖、腳本、manifest 都放在 `temp/` 下，素材根目錄只留原始檔與最終產出 |
| 7 | **最終產出四種格式** | `palmier_script.json`（給 MCP）+ `subtitles.srt`（字幕）+ `script_narrative.txt`（給人看的腳本）+ `temp/tts/*.mp3`（旁白音檔） |
| 8 | **步驟失敗 → retry 一次，再失敗則告知使用者** | 不要靜靜卡住或假設成功 |

## 預設值

| 項目 | 預設值 | 可調整 |
|:----:|:-------:|:------:|
| 解析度 | 1920×1080, 30fps | 否 |
| 場景偵測門檻 | `scene,0.15` | 否 |
| 場景長度 (MP4) | 5 秒 | 問卷 ANS_3 |
| 場景長度 (HEIC) | 等長於旁白語音 (~4s) | 否 |
| 旁白每句長度 | 15-20 字 | 否 |
| 旁白語言 | 正體中文 | 否 |
| 視覺模型 | `qwen2.5vl:7b` | 否 |
| Ollama API | `http://127.0.0.1:11434/api/generate` | 否 |
| TTS 引擎 | edge-tts | 否 |
| TTS 語音 | `zh-TW-YunJheNeural`（男聲） | 問卷 ANS_5 |
| 暫存目錄 | `temp/`（素材根目錄下） | 否 |

## Phase 1：互動問卷

依序詢問，答案記錄為 `ANS_X`：

| 代號 | 問題 | 預設值 |
|:----:|:-----|:------:|
| ANS_1 | **素材資料夾路徑？**（例如 `~/Desktop/日航商務艙初體驗/`） | `~/Desktop/` |
| ANS_2 | **跳過的檔案？**（多檔以逗號分隔，例如 `商務艙開箱_1.MP4,IMG_001.MOV`；留空則全部採用） | 空 |
| ANS_3 | **每個場景固定秒數？** | `5` |
| ANS_4 | **旁白口吻偏好？**（輕鬆 / 活潑 / 簡潔） | `輕鬆` |
| ANS_5 | **TTS 語音？**（`男聲` / `女聲`） | `男聲` |

問完後格式化輸出給使用者確認，再開始執行。

```
=== 確認資訊 ===
素材路徑: ~/Desktop/日航商務艙初體驗/
跳過檔案: 商務艙開箱_1.MP4
場景秒數: 5
旁白風格: 輕鬆
TTS 語音: 男聲
================
正確嗎？（回答 y / n）
```

---

## Phase 2：素材整理與場景切割

### Step 1 — 建立暫存目錄

```
TEMP_DIR = ANS_1 + "/temp/"
SCRIPTS_DIR = TEMP_DIR + "scripts/"
mkdir -p SCRIPTS_DIR TEMP_DIR/scenes TEMP_DIR/thumbs TEMP_DIR/tts
```

### Step 2 — 建立並執行 `run_scene_detect.py`

建立以下 Python 腳本到 `SCRIPTS_DIR/run_scene_detect.py`，然後執行。

```python
#!/usr/bin/env python3
"""Phase 2: Scene detection + clip extraction.
Scans ANS_1 for MP4/MOV/HEIC files, runs ffmpeg scene detection,
extracts clips, writes temp/manifest.json.

Usage: python3 run_scene_detect.py <素材路徑> [--skip 檔名,檔名] [--duration 秒]

Args:
    素材路徑: Source folder path
    --skip: Comma-separated filenames to skip
    --duration: Clip duration in seconds (default 5)
"""
import json, os, subprocess, sys, re, argparse
from pathlib import Path


def natural_sort_key(s):
    """Sort by numeric prefix then string."""
    m = re.match(r'(\d+)', s)
    return (int(m.group(1)) if m else 999, s)


def get_media_duration(path):
    r = subprocess.run(["ffprobe", "-v", "quiet", "-show_entries", "format=duration",
                        "-of", "csv=p=0", path], capture_output=True, text=True)
    try:
        return float(r.stdout.strip())
    except (ValueError, TypeError):
        return 0


def main():
    parser = argparse.ArgumentParser(description="Scene detection & clip extraction")
    parser.add_argument("source_dir", help="Path to source media folder")
    parser.add_argument("--skip", default="", help="Comma-separated filenames to skip")
    parser.add_argument("--duration", type=int, default=5, help="Clip duration in seconds")
    args = parser.parse_args()

    source_dir = Path(args.source_dir).expanduser().resolve()
    temp_dir = source_dir / "temp"
    scenes_dir = temp_dir / "scenes"
    scenes_dir.mkdir(parents=True, exist_ok=True)

    skip_list = [s.strip() for s in args.skip.split(",") if s.strip()]

    # Collect files
    all_files = []
    for ext in (".MP4", ".mp4", ".MOV", ".mov", ".HEIC", ".heic"):
        for f in sorted(source_dir.glob(f"*{ext}")):
            if f.name not in skip_list:
                all_files.append(f)

    if not all_files:
        print("No media files found!")
        sys.exit(1)

    # Sort by numeric prefix
    all_files.sort(key=lambda f: natural_sort_key(f.stem))

    manifest = []
    scene_idx = 0

    for filepath in all_files:
        ext = filepath.suffix.upper()
        stem = filepath.stem
        print(f"\n{'='*60}")
        print(f"Processing: {filepath.name}")

        if ext in (".HEIC", ".HEIF"):
            # Still image: record as single scene
            scene_idx += 1
            scene_name = f"scene_{scene_idx:04d}_{stem}"
            entry = {
                "scene": scene_idx,
                "source_file": filepath.name,
                "type": "image",
                "source_path": str(filepath),
                "display_name": scene_name,
                "duration": args.duration,
                "has_audio": False,
            }
            manifest.append(entry)
            print(f"  [{scene_idx:04d}] 📷 {filepath.name} ({args.duration}s)")
            continue

        # Video: run ffmpeg scene detection
        scene_file_escaped = str(scenes_dir / f"scene_%04d_{stem}.mp4")

        # First pass: detect scene changes
        detect_args = [
            "ffmpeg", "-i", str(filepath),
            "-vf", f"select='gt(scene,0.15)',setpts=N/FRAME_RATE/TB",
            "-vsync", "vfr",
            "-an",
            "-f", "null",
            "-"
        ]
        result = subprocess.run(detect_args, capture_output=True, text=True, timeout=300)
        stderr = result.stderr

        # Parse scene change frames from output
        scene_frames = [0]
        for line in stderr.split("\n"):
            m = re.search(r"Parsed_select_0.*n:\s*(\d+)", line)
            if m:
                frame = int(m.group(1))
                if frame > scene_frames[-1]:
                    scene_frames.append(frame)

        if len(scene_frames) < 2:
            # No scene change detected, use whole video
            dur = get_media_duration(filepath)
            scene_idx += 1
            scene_name = f"scene_{scene_idx:04d}_{stem}"
            out_file = scenes_dir / f"{scene_name}.mp4"

            target_dur = min(args.duration, dur)
            subprocess.run([
                "ffmpeg", "-y", "-ss", "0",
                "-i", str(filepath),
                "-t", str(target_dur),
                "-c:v", "libx264", "-pix_fmt", "yuv420p",
                "-c:a", "aac",
                str(out_file)
            ], capture_output=True, text=True, timeout=60)

            entry = {
                "scene": scene_idx,
                "source_file": filepath.name,
                "type": "video",
                "source_path": str(filepath),
                "display_name": scene_name,
                "source_start": 0.0,
                "source_end": target_dur,
                "duration": target_dur,
                "has_audio": True,
            }
            manifest.append(entry)
            print(f"  [{scene_idx:04d}] 🎬 {filepath.name} (no cuts, {target_dur:.1f}s)")
            continue

        # Extract each scene as a clip
        total_dur = get_media_duration(filepath)
        fps = 30  # assume 30fps

        for si in range(len(scene_frames)):
            start_frame = scene_frames[si]
            end_frame = scene_frames[si + 1] if si + 1 < len(scene_frames) else int(total_dur * fps)

            start_sec = start_frame / fps
            # Clip length: min(args.duration, segment_length)
            seg_len = (end_frame - start_frame) / fps
            clip_dur = min(args.duration, seg_len)

            if clip_dur < 0.5:
                continue  # skip too-short segments

            scene_idx += 1
            scene_name = f"scene_{scene_idx:04d}_{stem}"
            out_file = scenes_dir / f"{scene_name}.mp4"

            subprocess.run([
                "ffmpeg", "-y",
                "-ss", f"{start_sec:.3f}",
                "-i", str(filepath),
                "-t", f"{clip_dur:.2f}",
                "-c:v", "libx264", "-pix_fmt", "yuv420p",
                "-c:a", "aac",
                str(out_file)
            ], capture_output=True, text=True, timeout=60)

            entry = {
                "scene": scene_idx,
                "source_file": filepath.name,
                "type": "video",
                "source_path": str(filepath),
                "display_name": scene_name,
                "source_start": round(start_sec, 2),
                "source_end": round(start_sec + clip_dur, 2),
                "duration": round(clip_dur, 2),
                "has_audio": True,
            }
            manifest.append(entry)
            print(f"  [{scene_idx:04d}] 🎬 {filepath.name}  [{start_sec:.1f}s-{start_sec+clip_dur:.1f}s] {clip_dur:.1f}s")

    # Write manifest
    manifest_path = temp_dir / "manifest.json"
    with open(manifest_path, "w", encoding="utf-8") as f:
        json.dump(manifest, f, ensure_ascii=False, indent=2)

    print(f"\n{'='*60}")
    print(f"✅ Done! {len(manifest)} scenes extracted")
    print(f"📄 Manifest: {manifest_path}")
    print(f"📁 Scenes: {scenes_dir}")


if __name__ == "__main__":
    main()
```

**執行：**

```bash
python3 "$SCRIPTS_DIR/run_scene_detect.py" "ANS_1" \
  ${ANS_2:+--skip "$ANS_2"} \
  --duration ANS_3
```

**產出：**
- `temp/scenes/scene_0001_*.mp4` — 切割後的場景影片
- `temp/manifest.json` — 場景清單

### Step 3 — 確認輸出

確認 `temp/manifest.json` 已產生，且 `temp/scenes/` 目錄有對應的場景檔案。

---

## Phase 3：視覺描述

### Step 4 — 縮圖提取

對每個 video 場景提取 480px 縮圖到 `temp/thumbs/`。HEIC 直接從原檔提取。這步由 `qwen_describe.py` 自動處理，不需手動執行。

### Step 5 — 建立並執行 `qwen_describe.py`

```python
#!/usr/bin/env python3
"""Phase 3: Visual description via Qwen2.5VL.
Reads temp/manifest.json, extracts thumbnails to temp/thumbs/,
calls ollama, writes qwen_desc back to temp/manifest.json.

Usage: python3 qwen_describe.py <素材路徑>
"""
import json, os, subprocess, sys, requests, time, base64
from pathlib import Path


def get_thumbnail(manifest_entry, source_dir):
    """Get path to 480px thumbnail, creating if needed."""
    sid = manifest_entry["scene"]
    name = manifest_entry["display_name"]
    thumbs_dir = Path(source_dir) / "temp" / "thumbs"
    thumbs_dir.mkdir(parents=True, exist_ok=True)

    thumb_path = thumbs_dir / f"{name}.jpg"
    if thumb_path.exists():
        return thumb_path

    if manifest_entry["type"] == "image":
        src = Path(manifest_entry["source_path"])
        if src.suffix.upper() in (".HEIC", ".HEIF"):
            subprocess.run([
                "sips", "-Z", "480", "--setProperty", "format", "jpeg",
                str(src), "--out", str(thumb_path)
            ], capture_output=True, text=True, timeout=30)
        else:
            subprocess.run([
                "ffmpeg", "-y", "-i", str(src), "-vf", "scale=-1:480",
                "-vframes", "1", str(thumb_path)
            ], capture_output=True, text=True, timeout=30)
    else:
        scene_file = Path(source_dir) / "temp" / "scenes" / f"{name}.mp4"
        subprocess.run([
            "ffmpeg", "-y", "-ss", "1", "-i", str(scene_file),
            "-vframes", "1", "-vf", "scale=-1:480", str(thumb_path)
        ], capture_output=True, text=True, timeout=30)

    return thumb_path if thumb_path.exists() else None


def describe_image(image_path, api="http://127.0.0.1:11434/api/generate"):
    """Send image to Qwen2.5VL and get description."""
    if not image_path:
        return ""

    with open(image_path, "rb") as f:
        img_b64 = base64.b64encode(f.read()).decode("utf-8")

    prompt = "請用繁體中文描述這張圖片中的場景，2-3 句。只輸出描述，不要額外說明。注意這是某個影片或照片的畫面，請描述你在畫面中看到什麼。"

    try:
        resp = requests.post(api, json={
            "model": "qwen2.5vl:7b",
            "prompt": prompt,
            "images": [img_b64],
            "stream": False,
            "options": {"temperature": 0.3, "num_ctx": 2048}
        }, timeout=60)

        if resp.status_code == 200:
            text = resp.json().get("response", "").strip()
            # Clean up
            text = text.replace("\n", " ").replace("  ", " ")
            return text[:200]
        else:
            return f"[API Error: {resp.status_code}]"
    except Exception as e:
        return f"[Error: {e}]"


def main():
    parser = argparse.ArgumentParser(description="Qwen2.5VL visual description")
    parser.add_argument("source_dir", help="Path to source media folder")
    args = parser.parse_args()

    source_dir = Path(args.source_dir).expanduser().resolve()
    manifest_path = source_dir / "temp" / "manifest.json"

    if not manifest_path.exists():
        print(f"Error: {manifest_path} not found. Run run_scene_detect.py first.")
        sys.exit(1)

    with open(manifest_path) as f:
        manifest = json.load(f)

    # Check which entries already have descriptions
    pending = [item for item in manifest if not item.get("qwen_desc")]

    if not pending:
        print("All scenes already have descriptions.")
        sys.exit(0)

    print(f"Describing {len(pending)}/{len(manifest)} scenes...")

    for i, item in enumerate(pending):
        sid = item["scene"]
        thumb = get_thumbnail(item, source_dir)
        desc = describe_image(thumb)

        item["qwen_desc"] = desc
        status = "✅" if desc and not desc.startswith("[") else "⚠️"
        print(f"  [{sid:04d}/{len(manifest)}] {status} {desc[:60]}...", flush=True)

        # Save progress every 5
        if (i + 1) % 5 == 0 or i == len(pending) - 1:
            with open(manifest_path, "w", encoding="utf-8") as f:
                json.dump(manifest, f, ensure_ascii=False, indent=2)
            time.sleep(0.5)

    print(f"\n✅ Done. {len(pending)} descriptions written to manifest.json")


if __name__ == "__main__":
    main()
```

**執行：**

```bash
python3 "$SCRIPTS_DIR/qwen_describe.py" "ANS_1"
```

---

## Phase 4：場景篩選與旁白撰寫（opencode 負責）

### Step 6 — 讀取 manifest.json

載入 `temp/manifest.json`，逐一檢視每個場景的資訊：

- `source_file` — 原始檔名（最可靠）
- `qwen_desc` — Qwen 視覺描述（僅供參考）
- `type` — `video` 或 `image`
- `duration` — 場景長度

### Step 7 — 依照篩選規則決定保留或跳過

| 條件 | 處理 |
|:-----|:-----|
| `qwen_desc` 與 `source_file` 明顯不符（例如「展覽館門口」vs 檔名「點餐櫃檯」） | ✗ **視覺描述誤判** → 忽略 `qwen_desc`。以 `source_file` 為主改寫旁白。若 `source_file` 也無法判斷則跳過 |
| `qwen_desc` 內容模糊無意義（「模糊的黃色布料」、「黑色的背景」） | ✗ **無法辨識** → 跳過該場景 |
| 同一個 `source_file` 有多個子場景，且內容高度重複（如同支手機錄影連續拍 App 畫面） | ✗ **內容重複** → 只保留最具代表性的 2-3 段，其餘跳過。保留規則：開頭、中間、結尾各一 |
| 畫面與主線主題無關（拍到地板、晃動模糊、鏡頭蓋） | ✗ **無關畫面** → 跳過 |
| 其他情況 | ✅ **保留**。以 `source_file` 為準撰寫旁白 |

### Step 8 — 撰寫旁白

為每個「保留」的場景寫一句旁白，寫入 `caption_short` 欄位。

**寫作原則：**

- **輕鬆口語**，像在跟朋友聊天
  - 不要「我看到」、「我發現」、「映入眼簾」
  - 可以用「哇」、「耶」、「來～」、「～耶」、「～喔」等口語語氣詞
  - 參考口吻：`「走～來看看貴賓室長怎樣」`、`「冰箱裡直接擺兩瓶酒，誠意滿滿」`
- **善用檔名梗**：如果檔名有趣（「躺著就到貴賓室的貴賓」），可以玩文字遊戲
- **每句 15-20 字**，不超過 25 字
- **前後連貫**：相鄰場景的旁白之間保有主題的延續性
- **風格統一**：整份旁白的口吻要一致

**語氣範例表：**

| 原始檔名 | 場景類型 | 輕鬆口吻範例 |
|:---------|:--------:|:-------------|
| `0_0貴賓室位置` | 影片 | 走～來看看日航貴賓室長怎樣 |
| `3_0貴賓室view_2` | 影片 | 陽光灑進來，整間餐廳都亮了 |
| `9_0貴賓室餐點線上預約` | 影片 | 拿起手機打開 JAL App，準備點餐 |
| `9_1咖哩飯.HEIC` | 照片 | 咖哩飯上桌，光看就餓了 |
| `11_0商務艙簡易開箱` | 影片 | 這螢幕也太大了吧，比我家電視還大 |
| `13_1芸希躺平.HEIC` | 照片 | 女兒已經躺平在玩了，上面還有小飛機 |
| `18_和式飛機餐.HEIC` | 照片 | 和食便當超精緻，每道都像樣品 |
| `22_到台灣囉.HEIC` | 照片 | 吃飽睡飽，睜開眼就到台灣了 |

### Step 9 — 寫回 manifest.json

將所有場景的 `caption_short`（保留的）或 `"skip": true`（跳過的）寫入 `temp/manifest.json`。

```python
for item in manifest:
    if scene_should_skip(item):
        item["skip"] = True
    else:
        item["caption_short"] = "寫好的旁白"
```

---

## Phase 4b：edge-tts 語音合成（可選）

### Step 10 — 安裝 edge-tts

```bash
pip install edge-tts
```

### Step 11 — 建立並執行 `generate_tts.py`

讀取 `temp/manifest.json` 中所有保留場景的 `caption_short`，用 edge-tts 合成 MP3 到 `temp/tts/`。

```python
#!/usr/bin/env python3
"""Phase 4b: edge-tts TTS generation.
Reads temp/manifest.json (with caption_short),
generates MP3 for each scene to temp/tts/,
writes tts_file path back to manifest.

Usage: python3 generate_tts.py <素材路徑> [--voice zh-TW-YunJheNeural]
"""
import json, os, sys, argparse, asyncio, subprocess
from pathlib import Path


async def generate_tts(text, output_path, voice):
    cmd = [
        sys.executable, "-m", "edge_tts",
        "--voice", voice,
        "--text", text,
        "--write-media", str(output_path),
    ]
    proc = await asyncio.create_subprocess_exec(
        *cmd, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE
    )
    await proc.communicate()
    return proc.returncode == 0


async def main():
    parser = argparse.ArgumentParser(description="edge-tts TTS generation")
    parser.add_argument("source_dir", help="Path to source media folder")
    parser.add_argument("--voice", default="zh-TW-YunJheNeural",
                        help="edge-tts voice name")
    args = parser.parse_args()

    source_dir = Path(args.source_dir).expanduser().resolve()
    manifest_path = source_dir / "temp" / "manifest.json"
    tts_dir = source_dir / "temp" / "tts"
    tts_dir.mkdir(parents=True, exist_ok=True)

    if not manifest_path.exists():
        print(f"Error: {manifest_path} not found")
        sys.exit(1)

    with open(manifest_path) as f:
        manifest = json.load(f)

    active = [item for item in manifest
              if not item.get("skip") and item.get("caption_short")]

    if not active:
        print("Error: no active scenes with captions found")
        sys.exit(1)

    # Check if all already have TTS
    pending = [item for item in active if not item.get("tts_file")]

    if not pending:
        print("All scenes already have TTS files.")
        sys.exit(0)

    print(f"Generating TTS for {len(pending)} scenes (voice: {args.voice})...")

    for i, item in enumerate(pending):
        sid = item["scene"]
        text = item["caption_short"]
        out_name = f"scene_{sid:04d}.mp3"
        out_path = tts_dir / out_name

        # Get voice: check ANS_5 for gender preference
        voice = args.voice

        success = await generate_tts(text, out_path, voice)
        if success and out_path.exists():
            item["tts_file"] = str(out_path)
            dur = len(text) / 4  # rough estimate
            item["tts_duration_sec"] = round(dur, 1)
            status = "✅"
        else:
            status = "⚠️"

        print(f"  [{sid:04d}/{len(active)}] {status} {text} -> {out_name}", flush=True)

        # Save progress every 5
        if (i + 1) % 5 == 0 or i == len(pending) - 1:
            with open(manifest_path, "w", encoding="utf-8") as f:
                json.dump(manifest, f, ensure_ascii=False, indent=2)

    print(f"\n✅ Done. {len(pending)} TTS files in {tts_dir}")


if __name__ == "__main__":
    asyncio.run(main())
```

**執行：**

```bash
# 男聲（預設）
python3 "$SCRIPTS_DIR/generate_tts.py" "ANS_1" --voice zh-TW-YunJheNeural

# 女聲（依問卷 ANS_5）
python3 "$SCRIPTS_DIR/generate_tts.py" "ANS_1" --voice zh-TW-HsiaoChenNeural
```

**產出：**
- `temp/tts/scene_0001.mp3` ~ `scene_NNNN.mp3` — 每場景旁白音檔
- `temp/manifest.json` 更新，每個 entry 新增 `tts_file` 與 `tts_duration_sec`

---

## Phase 5：輸出四種格式

### Step 12 — 建立並執行 `export_scripts.py`

```python
#!/usr/bin/env python3
"""Phase 5: Export output formats.
Reads temp/manifest.json (with caption_short + tts_file),
calculates cumulative timing,
outputs palmier_script.json + subtitles.srt + script_narrative.txt

Usage: python3 export_scripts.py <素材路徑> [--fps 30]
"""
import json, os, sys, argparse, math
from pathlib import Path


def fmt_time(secs):
    """Format seconds to SRT timestamp."""
    h = int(secs // 3600)
    m = int((secs % 3600) // 60)
    s = int(secs % 60)
    ms = int((secs - int(secs)) * 1000)
    return f"{h:02d}:{m:02d}:{s:02d},{ms:03d}"


def estimate_tts_duration(text):
    """Rough estimate: ~4 chars/sec for Mandarin Chinese."""
    return max(1.5, len(text) / 4)


def main():
    parser = argparse.ArgumentParser(description="Export scenes to Palmier scripts")
    parser.add_argument("source_dir", help="Path to source media folder")
    parser.add_argument("--fps", type=int, default=30, help="Timeline FPS")
    args = parser.parse_args()

    source_dir = Path(args.source_dir).expanduser().resolve()
    manifest_path = source_dir / "temp" / "manifest.json"

    if not manifest_path.exists():
        print(f"Error: {manifest_path} not found")
        sys.exit(1)

    with open(manifest_path) as f:
        manifest = json.load(f)

    # Filter: keep non-skipped scenes with captions
    active = [item for item in manifest
              if not item.get("skip") and item.get("caption_short")]

    if not active:
        print("Error: no active scenes with captions found")
        sys.exit(1)

    fps = args.fps

    # Calculate cumulative timing
    cumulative = 0.0
    for item in active:
        dur = item.get("duration", 5.0)
        item["_start_sec"] = round(cumulative, 2)
        item["_end_sec"] = round(cumulative + dur, 2)
        item["_start_frame"] = round(cumulative * fps)
        item["_end_frame"] = round((cumulative + dur) * fps)
        text = item.get("caption_short", "")
        item["_tts_sec"] = round(estimate_tts_duration(text), 1)
        cumulative += dur

    total_sec = cumulative
    print(f"Active scenes: {len(active)}/{len(manifest)}")
    print(f"Total duration: {total_sec:.1f}s ({total_sec/60:.1f} min)")

    # --- Output 1: palmier_script.json ---
    script = {
        "project": {
            "name": source_dir.name,
            "fps": fps,
            "resolution": "1920x1080",
            "total_scenes": len(active),
            "total_duration_sec": round(total_sec, 1),
        },
        "scenes": []
    }

    for item in active:
        scene_entry = {
            "scene": item["scene"],
            "source_file": item["source_file"],
            "type": item["type"],
            "source_path": item.get("source_path", ""),
            "source_start_sec": item.get("source_start"),
            "source_end_sec": item.get("source_end"),
            "duration_sec": item.get("duration"),
            "start_frame": item["_start_frame"],
            "end_frame": item["_end_frame"],
            "subtitle": item["caption_short"],
            "tts_approx_sec": item["_tts_sec"],
            "tts_file": item.get("tts_file"),
        }
        # Remove None values
        scene_entry = {k: v for k, v in scene_entry.items() if v is not None}
        script["scenes"].append(scene_entry)

    palmier_path = source_dir / "palmier_script.json"
    with open(palmier_path, "w", encoding="utf-8") as f:
        json.dump(script, f, ensure_ascii=False, indent=2)
    print(f"📄 {palmier_path} ({len(active)} scenes)")

    # --- Output 2: subtitles.srt ---
    srt_path = source_dir / "subtitles.srt"
    with open(srt_path, "w", encoding="utf-8") as f:
        for i, item in enumerate(active):
            start = item["_start_sec"]
            end = item["_end_sec"]
            text = item["caption_short"]
            f.write(f"{i+1}\n{fmt_time(start)} --> {fmt_time(end)}\n{text}\n\n")
    print(f"📄 {srt_path} ({len(active)} entries)")

    # --- Output 3: script_narrative.txt ---
    txt_path = source_dir / "script_narrative.txt"
    with open(txt_path, "w", encoding="utf-8") as f:
        f.write(f"{'='*60}\n")
        f.write(f"{source_dir.name} · 剪輯腳本\n")
        f.write(f"{'='*60}\n")
        f.write(f"共 {len(active)} 場景 · 總長 ~{total_sec:.0f} 秒 ({total_sec/60:.1f} 分)\n\n")

        # Detect chapter breaks by source file grouping
        current_section = ""
        for item in active:
            sf = item["source_file"]
            # Check for section change
            first_digit = re.search(r'^(\d+)', sf)
            if first_digit:
                prefix = int(first_digit.group(1))
                # Major section breaks: 0-9 = lounge, 10+ = cabin
                if prefix <= 9:
                    section_label = "貴賓室"
                else:
                    section_label = "商務艙"
            else:
                section_label = "其他"

            if section_label != current_section:
                current_section = section_label
                f.write(f"\n{'─'*60}\n")
                f.write(f"【{section_label}】\n")
                f.write(f"{'─'*60}\n\n")

            f.write(f"{item['scene']:02d} │ {item['source_file']:<30s} │ {item['duration']:.1f}s")
            if item.get("type") == "image":
                f.write(" 📷")
            f.write(f"\n   │ {item['caption_short']}\n\n")

        f.write(f"{'='*60}\n")
        f.write("腳本結束\n")
        f.write(f"{'='*60}\n")

    print(f"📄 {txt_path} ({len(active)} entries)")


if __name__ == "__main__":
    main()
```

**執行：**

```bash
python3 "$SCRIPTS_DIR/export_scripts.py" "ANS_1" --fps 30
```

### Step 13 — 驗證輸出

確認根目錄下有：
- `palmier_script.json` — 含每個場景的 **tts_file** 路徑
- `subtitles.srt` — 標準字幕
- `script_narrative.txt` — 腳本文字

確認 `temp/tts/` 下有對應的 `.mp3` 音檔。

---

## 輸出檔案一覽

執行完成後，素材資料夾內會產生：

```
素材資料夾/
├── palmier_script.json       ← → 給 Palmier Pro MCP（含 tts_file 路徑）
├── subtitles.srt              ← → 字幕檔
├── script_narrative.txt      ← → 給人看的腳本
└── temp/                     ← 暫存檔（可清空）
    ├── manifest.json         ← 完整場景清單（含描述、旁白、TTS）
    ├── scripts/
    │   ├── run_scene_detect.py
    │   ├── qwen_describe.py
    │   ├── generate_tts.py
    │   └── export_scripts.py
    ├── scenes/               ← 切割後的獨立場景 MP4
    ├── thumbs/               ← 480px 縮圖（視覺描述用）
    └── tts/                  ← edge-tts 旁白音檔 MP3
```

## Palmier Script JSON 規格

`palmier_script.json` 是核心輸出，每個 scene entry 直對應到 Palmier Pro MCP 工具：

### `add_clips` 對應

```json
{
  "scene": 1,
  "source_file": "0_0貴賓室位置.MP4",
  "type": "video",
  "source_path": "/Users/xxx/0_0貴賓室位置.MP4",
  "source_start_sec": 0.5,
  "source_end_sec": 5.5,
  "duration_sec": 5.0,
  "start_frame": 0,
  "end_frame": 150,
  "subtitle": "走～來看看日航貴賓室長怎樣",
  "tts_approx_sec": 2.5,
  "tts_file": "/path/to/temp/tts/scene_0001.mp3"
}
```

使用方式：

```python
# 1. import_media(source={path: "素材路徑/"}) → 取得 mediaRefs
# 2. 逐場景 add_clips:
#    video:  add_clips(entries=[{
#        mediaRef: ref_of_source_file,
#        source: [source_start_sec, source_end_sec],
#        startFrame: start_frame
#    }])
#    image:  add_clips(entries=[{
#        mediaRef: ref_of_source_file,
#        startFrame: start_frame,
#        endFrame: end_frame  # duration_sec * fps
#    }])
# 3. 累積 startFrame 依序上片
```

### `add_texts` + TTS 音檔

```python
# 上完所有 clip 後，一次 add_texts 上字幕
add_texts(entries=[{
    "startFrame": item.start_frame,
    "endFrame": item.end_frame,
    "content": item.subtitle,
    "style": {
        "color": "#FFFFFF",
        "fontSize": 48,
        "alignment": "center"
    },
    "transform": {
        "centerX": 0.5,
        "centerY": 0.85
    }
} for item in script.scenes])

# 如有 TTS 音檔，匯入並上旁白音軌
for item in script.scenes:
    if item.get("tts_file"):
        import_media(source={path: item.tts_file})
        add_clips(entries=[{
            "mediaRef": ref_of_tts,
            "startFrame": item.start_frame,
            "endFrame": item.end_frame,
            "trackIndex": audio_track_index
        }])
```

## MCP 工具速查（Palmier 側）

| 工具 | 用途 |
|:----:|:------|
| `manage_project` | 開新專案 |
| `import_media` | 匯入整個素材資料夾 |
| `get_media` | 取得 mediaRef（對應來源檔名） |
| `add_clips` | 依序上片（不用 insert_clips，會 ripple） |
| `manage_tracks` | 調整音軌／字幕軌 |
| `add_texts` | 一次匯入全部字幕 |
| `update_text` | 修正字幕內容或樣式 |
| `export_project` | 輸出成品 |

## 經驗教訓

| # | 教訓 | 說明 |
|:-:|:----|:------|
| 1 | **場景偵測後先看 manifest 再寫旁白** | 先確認場景數量與內容，避免寫到一半發現場景順序不對 |
| 2 | **Qwen 描述僅供參考，檔名才是真的** | 視覺模型對手機螢幕錄影、模糊背景的場景正確率低 |
| 3 | **同一檔案多段子場景要先過濾再寫旁白** | 若先寫完再發現要跳過，會浪費時間 |
| 4 | **旁白每句控制 15-20 字** | 太長的旁白在 5 秒場景內唸不完 |
| 5 | **HEIC 照片直接給 Palmier 處理** | 不需預轉 JPG，Palmier 原生支援 |
| 6 | **先 remove_silence 再放 BGM** | silence 的 ripple 會切裂已定位的 BGM track |
| 7 | **TTS 音檔在 Palmier 側上旁白軌** | edge-tts 產生的 MP3 經 `import_media` 匯入後，用 `add_clips` 放到獨立的旁白音軌，不要混入原始素材音軌 |
