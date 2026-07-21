---
name: palmier-autocut-scene-tts
description: >
  Scene-based 剪片流程：ffmpeg 場景偵測 → Qwen2.5VL 視覺描述 →
  opencode 旁白撰寫 → Edge TTS / 台灣藍鵲 TTS 語音合成 →
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
| 7 | **最終產出三種格式** | `subtitles.srt`（字幕）+ `script_narrative.txt`（給人看的腳本）+ `temp/tts/*.wav`（旁白音檔） |
| 8 | **步驟失敗 → retry 一次，再失敗則告知使用者** | 不要靜靜卡住或假設成功 |
| 9 | **TTS 音檔全部 loudnorm 統一音量** | 不同 TTS 片段之間音量可能不一致，用 `ffmpeg -af loudnorm=I=-16:LRA=1:TP=-1` 對所有 WAV/MP3 做 EBU R128 正規化 |

## 預設值

| 項目 | 預設值 | 可調整 |
|:----:|:-------:|:------:|
| 解析度 | 1920×1080, 30fps | 否 |
| 場景偵測門檻 | `scene,0.15` | 否 |
| 場景長度 (MP4) | 3~5 秒（依場景動態） | 問卷 ANS_2 |
| 場景長度 (HEIC/圖片) | **固定 4 秒** | 否 |
| 旁白每句長度 | 15-20 字 | 否 |
| 旁白語言 | 正體中文 | 否 |
| 視覺模型 | `qwen2.5vl:7b` | 否 |
| Ollama API | `http://127.0.0.1:11434/api/generate` | 否 |
| TTS 引擎 | Edge TTS / 台灣藍鵲 TTS | 問卷 ANS_4 |
| Edge TTS 語音 | `zh-TW-YunJheNeural`（男聲） | 問卷 ANS_4a |
| 台灣藍鵲 TTS 聲音 | 李宏毅老師 / 通用女聲 / 自己的聲音 | 問卷 ANS_4b |
| BGM 風格 | 視 ANS_5（`不需要` / `輕鬆` / `活潑` / `寧靜` / 自訂） | 問卷 ANS_5 |
| BGM 來源 | Pixabay Music（CC0 免費授權） | 否 |
| 暫存目錄 | `temp/`（素材根目錄下） | 否 |
| 字幕字體大小 | 48（canvas points） | 否 |
| 字幕位置 | 螢幕中間下方（centerX: 0.5, centerY: 0.85） | 否 |
| 字幕顏色 | 白字 `#FFFFFF`，無背景 | 否 |

## Phase 1：互動問卷

依序詢問，答案記錄為 `ANS_X`。後續問題取決於前一個答案。

```
ANS_1  素材資料夾路徑？
        資料夾名稱建議符合影片主題
        範例: ~/Desktop/日航商務艙初體驗/
        預設: ~/Desktop/影片主題資料夾

ANS_2  最大場景秒數？（3~5s 動態，最小值固定 3）
        預設: 5

ANS_3  旁白口吻偏好？（輕鬆 / 活潑 / 通俗幽默 / 簡潔）
       預設: 輕鬆通俗幽默

ANS_4  TTS 引擎？（Edge TTS / 台灣藍鵲 TTS）
       預設: Edge TTS

       if ANS_4 == "Edge TTS":
         ANS_4a  語音性別？（男聲 / 女聲）
                 預設: 男聲
                 男聲 → edge-tts voice: zh-TW-YunJheNeural
                 女聲 → edge-tts voice: zh-TW-HsiaoChenNeural

       if ANS_4 == "台灣藍鵲 TTS":
         ANS_4b  聲音選擇？（李宏毅老師 / 通用女聲 / 自己的聲音）
                 預設: 李宏毅老師

         if ANS_4b == "自己的聲音":
           ANS_4c  聲音向量 .pt 路徑？
                   預設: ~/Desktop/剪片ing/自己的聲音向量/my_voice.pt

ANS_5  背景音樂風格？（不需要 / 輕鬆 / 活潑 / 寧靜 / 自訂描述）
       預設: 輕鬆
```

問完後格式化輸出給使用者確認，再開始執行。

```
=== 確認資訊 ===
素材路徑: ~/Desktop/日航商務艙初體驗/
場景秒數: 3~5 (動態)
旁白風格: 輕鬆
TTS 引擎: Edge TTS
  └ 語音性別: 男聲                # Edge TTS 時顯示
  └ 聲音選擇: 李宏毅老師           # 台灣藍鵲 TTS 時顯示
  └ 聲音向量: .../my_voice.pt     # 自己的聲音時才顯示
BGM 風格: 輕鬆
================
正確嗎？（回答 y / n）
```

---

## Phase 2：素材整理與場景切割

### Step 1 — 建立暫存目錄

```
TEMP_DIR = ANS_1 + "/temp/"
SCRIPTS_DIR = TEMP_DIR + "scripts/"
mkdir -p SCRIPTS_DIR TEMP_DIR/scenes TEMP_DIR/thumbs TEMP_DIR/tts TEMP_DIR/bgm
```

### Step 2 — 建立並執行 `run_scene_detect.py`

建立以下 Python 腳本到 `SCRIPTS_DIR/run_scene_detect.py`，然後執行。

```python
#!/usr/bin/env python3
"""Phase 2: Scene detection + clip extraction.
ffmpeg scene detect at 0.15 → merge <1s segments → one 3~6s clip per segment.
Images: fixed max_duration.

Usage: python3 run_scene_detect.py <素材路徑> [--skip 檔名,檔名] [--max-duration 秒]
"""
import json, subprocess, sys, re, argparse
from pathlib import Path


def natural_sort_key(s):
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
    parser.add_argument("--max-duration", type=int, default=6, help="Max clip duration in seconds (default 6)")
    args = parser.parse_args()

    source_dir = Path(args.source_dir).expanduser().resolve()
    temp_dir = source_dir / "temp"
    scenes_dir = temp_dir / "scenes"
    scenes_dir.mkdir(parents=True, exist_ok=True)

    skip_list = [s.strip() for s in args.skip.split(",") if s.strip()]

    all_files = []
    for ext in (".MP4", ".mp4", ".MOV", ".mov", ".HEIC", ".heic"):
        for f in sorted(source_dir.glob(f"*{ext}")):
            if f.name not in skip_list:
                all_files.append(f)

    if not all_files:
        print("No media files found!")
        sys.exit(1)

    all_files.sort(key=lambda f: natural_sort_key(f.stem))

    manifest = []
    scene_idx = 0
    fps = 30
    min_dur = 3
    max_dur = args.max_duration
    merge_threshold = 1.0  # merge segments shorter than this

    for filepath in all_files:
        ext = filepath.suffix.upper()
        stem = filepath.stem
        full_stem = re.sub(r'[^\w\s-]', '', stem).replace(' ', '_') or stem
        print(f"\n{'='*60}")
        print(f"Processing: {filepath.name}")

        source_dur = get_media_duration(filepath)

        if ext in (".HEIC", ".HEIF"):
            scene_idx += 1
            scene_name = f"scene_{scene_idx:04d}_{full_stem}"
            entry = {
                "scene": scene_idx,
                "source_file": filepath.name,
                "source_duration": round(source_dur, 1),
                "type": "image",
                "source_path": str(filepath),
                "display_name": scene_name,
                "duration": 4.0,  # images fixed at 4s
                "has_audio": False,
            }
            manifest.append(entry)
            print(f"  [{scene_idx:04d}] 📷 {filepath.name} ({max_dur}s)")
            continue

        # ffmpeg scene detection at 0.15
        r = subprocess.run([
            "ffmpeg", "-i", str(filepath),
            "-vf", "select='gt(scene,0.15)',showinfo",
            "-vsync", "vfr", "-an", "-f", "null", "-"
        ], capture_output=True, text=True, timeout=300)
        stderr = r.stderr

        # Parse scene change pts_times
        change_pts = []
        for line in stderr.split("\n"):
            m = re.search(r"pts_time:([\d.]+)", line)
            if m:
                t = float(m.group(1))
                if t > 0.1:
                    change_pts.append(t)

        # Build segments [start, end) from change points
        raw_segments = []
        prev = 0.0
        for t in change_pts:
            raw_segments.append((prev, t))
            prev = t
        raw_segments.append((prev, source_dur))

        # Merge segments shorter than merge_threshold — repeat until stable
        merged = list(raw_segments)
        changed = True
        while changed:
            changed = False
            i = 0
            while i < len(merged):
                s, e = merged[i]
                if e - s >= merge_threshold:
                    i += 1
                    continue
                if i + 1 < len(merged):
                    merged[i] = (s, merged[i + 1][1])
                    merged.pop(i + 1)
                    changed = True
                elif i > 0:
                    merged[i - 1] = (merged[i - 1][0], e)
                    merged.pop(i)
                    changed = True
                    i -= 1
                else:
                    i += 1

        # If only 1 segment after merge → whole file = one scene
        if len(merged) <= 1:
            clip_dur = min(max(source_dur, min_dur), max_dur)
            scene_idx += 1
            scene_name = f"scene_{scene_idx:04d}_{full_stem}"
            out_file = scenes_dir / f"{scene_name}.mp4"
            subprocess.run([
                "ffmpeg", "-y", "-ss", "0", "-i", str(filepath),
                "-t", f"{clip_dur:.2f}",
                "-c:v", "libx264", "-pix_fmt", "yuv420p", "-c:a", "aac",
                str(out_file)
            ], capture_output=True, text=True, timeout=60)
            entry = {
                "scene": scene_idx, "source_file": filepath.name,
                "source_duration": round(source_dur, 1), "type": "video",
                "source_path": str(filepath), "display_name": scene_name,
                "source_start": 0.0, "source_end": clip_dur,
                "duration": clip_dur, "has_audio": True,
            }
            manifest.append(entry)
            print(f"  [{scene_idx:04d}] 🎬 {filepath.name}  [0-{clip_dur:.1f}s] {clip_dur:.1f}s (1 scene)")
            continue

        # Extract one 3~6s clip from each merged segment (centered)
        for s, e in merged:
            seg_len = e - s
            clip_dur = min(max(seg_len, min_dur), max_dur)
            if seg_len > clip_dur:
                offset = (seg_len - clip_dur) / 2
                start_sec = s + offset
            else:
                start_sec = s

            scene_idx += 1
            scene_name = f"scene_{scene_idx:04d}_{full_stem}"
            out_file = scenes_dir / f"{scene_name}.mp4"
            subprocess.run([
                "ffmpeg", "-y",
                "-ss", f"{start_sec:.3f}", "-i", str(filepath),
                "-t", f"{clip_dur:.2f}",
                "-c:v", "libx264", "-pix_fmt", "yuv420p", "-c:a", "aac",
                str(out_file)
            ], capture_output=True, text=True, timeout=60)

            entry = {
                "scene": scene_idx, "source_file": filepath.name,
                "source_duration": round(source_dur, 1), "type": "video",
                "source_path": str(filepath), "display_name": scene_name,
                "source_start": round(start_sec, 2),
                "source_end": round(start_sec + clip_dur, 2),
                "duration": round(clip_dur, 2), "has_audio": True,
            }
            manifest.append(entry)
            print(f"  [{scene_idx:04d}] 🎬 {filepath.name}  [{start_sec:.1f}-{start_sec+clip_dur:.1f}s] {clip_dur:.1f}s  (seg={seg_len:.1f}s)")

    manifest_path = temp_dir / "manifest.json"
    with open(manifest_path, "w", encoding="utf-8") as f:
        json.dump(manifest, f, ensure_ascii=False, indent=2)

    total_time = sum(s.get("duration", 0) for s in manifest)
    print(f"\n{'='*60}")
    print(f"✅ {len(manifest)} scenes, {total_time:.1f}s total")
    print(f"📄 {manifest_path}")


if __name__ == "__main__":
    main()
```

**執行：**

```bash
python3 "$SCRIPTS_DIR/run_scene_detect.py" "ANS_1" \
  --max-duration ANS_2
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

## Phase 3b：場景過濾（opencode 負責）

### Step 6 — 以 Qwen 描述為基礎過濾場景

載入 `temp/manifest.json`，逐一檢視每個場景的 `qwen_desc`。

**過濾規則（依優先順序）：**

| # | 條件 | 處理 |
|:-:|:-----|:-----|
| 1 | `qwen_desc` 包含明顯的模糊/無法辨識關鍵字（「模糊」、「無法辨識」、「黑色背景」、「布料」等） | ✗ **晃動雜訊** → 跳過 |
| 2 | 同一個 `source_file` 有大量子場景（如手機錄影連拍 App 畫面），且 `qwen_desc` 內容高度重複 | ✗ **內容重複** → 只留開頭、中間、結尾各一，其餘跳過。App 螢幕錄影一類留 3-5 段足矣 |
| 3 | 畫面明顯與主線主題無關（拍到地板、褲子、鏡頭蓋、走道晃動） | ✗ **無關畫面** → 跳過 |
| 4 | `qwen_desc` 與 `source_file` 明顯矛盾（檔名「點餐櫃檯」但描述說「展覽館門口」） | → 以 `source_file` 為準，忽略 `qwen_desc`。若仍無法判斷則跳過 |
| 5 | 其他情況 | ✅ **保留** |

**實作方式：** 在 `temp/manifest.json` 中被跳過的場景加上 `"skip": true`。

```python
skip_scenes = set()
# 依上述規則決定要跳過的 scene 編號
for item in manifest:
    if item["scene"] in skip_scenes:
        item["skip"] = True
```

---

## Phase 4：旁白撰寫（opencode 負責）

### Step 7 — 載入過濾後的 manifest

讀取 `temp/manifest.json`，只處理沒有 `skip` 標記的場景。

### Step 8 — 撰寫旁白

為每個保留場景寫一句旁白，寫入 `caption_short` 欄位。

**核心要求：輕鬆、通俗、幽默。像在跟朋友邊看影片邊聊天，不是寫字幕標題。讓人看了會笑、覺得親切，不是讀說明書。**

**兩大資訊來源（都要看，互補）：**
| 來源 | 給你什麼 | 範例 |
|:-----|:---------|:------|
| **Qwen `qwen_desc`** | 畫面上實際有什麼（物體、場景、人物、文字） | 「窗外有停機坪」、「桌上有咖哩飯」、「手機畫面顯示 JAL App」 |
| **檔案名稱** | 拍攝者原本想記錄什麼（context、人物名、地點、趣味雙關） | 「躺著就到貴賓室的貴賓」、「芸希躺平」、「甜點有哈根達斯」 |

寫法：Qwen 告訴你畫面內容，檔名告訴你怎麼解讀它。例如 Qwen 說「小孩躺在椅子上」，檔名說「芸希躺平」→ 寫「女兒已經躺平在玩了」；Qwen 說「桌上有一個冰淇淋」，檔名說「甜點有哈根達斯」→ 寫「甜點是哈根達斯」。

**鐵則（照優先順序，每條都強制考慮 Qwen + 檔名兩個來源）：**

| # | 規則 | 雙重來源：Qwen 描述 ⨯ 檔名 |
|:-:|:-----|:---------------------------|
| 1 | **用 Qwen 描述的具體視覺細節 + 檔名資訊來寫**。Qwen 說有停機坪 → 寫「窗外就是停機坪」；檔名有「躺著就到貴賓室的貴賓」→ 寫「女兒躺著就到貴賓室」。**兩個來源都要看**，不能只看一邊 | |
| 2 | **寫完整一句話，不是標題**。要有主體 + 動作/描述 +（可選）humor punchline。不要低於 12 字。問自己：這句話拿掉畫面還能不能聽懂在講什麼？→ 不行就太短 | |
| 3 | **每句 15-25 字**，不要低於 12 字（太短像標題，沒畫面感）。寧可長一點有細節，也不要短到像字幕機跑馬燈 | |
| 4 | **嵌入具體名稱**：從 Qwen 擷取（Sakura Lounge、JAL、哈根達斯、停機坪、SkySuite）+ 從檔名擷取（芸希、咖哩飯、眼罩）。讓旁白有獨特性，不是套版。反面：如果旁白換到其他影片也能用 → 太抽象，重寫 | |
| 5 | **前後連貫像在講故事**。相鄰場景的旁白之間保有主題延續性。寫完後讀一遍，確認有「走進貴賓室 → 繞一圈 → 看窗外 → 坐下來吃」的敘事流，不是跳躍式 | |
| 6 | **輕鬆通俗幽默**。善用檔名玩梗（「躺著就到貴賓室的貴賓」→ 真不愧是貴賓）。用一般人會說的講法，不用書面語。問自己：這句拿去跟朋友聊天會很奇怪嗎？→ 怪就重寫 | |

**禁止詞：** 「我看到」、「我發現」、「映入眼簾」、「這裡有」、「這個是」

**語氣範例表（加強版，每句都有具體細節）：**

| 原始檔名 | 旁白範例 | 關鍵細節來源 |
|:---------|:---------|:-------------|
| `0_0貴賓室位置` | 走～來看看日航貴賓室在哪 | 檔名 hint |
| `0_1躺著就到貴賓室的貴賓` | 女兒躺著就到貴賓室，真不愧是貴賓 | 檔名梗 + 人物 |
| `1_進入貴賓室` | 一進 Sakura Lounge，氣氛就不一樣 | Qwen 描述 lounge 內部 |
| `3_0貴賓室view_2` | 窗外就是停機坪，陽光灑進來好美 | Qwen 看到停機坪 + 陽光 |
| `3_1貴賓室樓下view1` | 樓下座位區也好舒服，view 很讚 | Qwen 描述座位區 |
| `9_0貴賓室餐點線上預約` | 拿起手機打開 JAL App，準備點餐 | Qwen 看到手機畫面 |
| `9_1咖哩飯.HEIC` | 咖哩飯上桌，光看就餓了 | Qwen 看到咖哩飯照片 |
| `11_0商務艙簡易開箱` | 這螢幕也太大了吧，比我家電視還大 | Qwen 看到大螢幕 |
| `13_1芸希躺平.HEIC` | 女兒已經躺平在玩了，上面還有小飛機 | Qwen 看到人物 + 座椅 |
| `18_和式飛機餐.HEIC` | 和食便當超精緻，每道都像樣品 | Qwen 看到便當細節 |
| `22_到台灣囉.HEIC` | 吃飽睡飽，睜開眼就到台灣了 | 結尾場景 + 檔名梗 |

### Step 9 — 寫回 manifest.json

將所有場景的 `caption_short`（保留的）或 `"skip": true`（跳過的）寫入 `temp/manifest.json`。

```python
for item in manifest:
    if item.get("skip"):
        continue
    item["caption_short"] = "寫好的旁白"
```

### Step 9b — ✋ 使用確認：讓使用者檢查旁白

所有旁白寫入 manifest 後，**暫停管線**，輸出旁白表給使用者確認：

```
=== 旁白檢查表 ===
序號 │ 來源檔名            │ 秒數  │ 旁白
─────┼─────────────────────┼───────┼──────────────────────────
001  │ 0_0貴賓室位置       │  4.0  │ 走～來看看日航貴賓室在哪
002  │ 0_1躺著就到貴賓室的貴賓│ 4.0  │ 女兒躺著就到貴賓室...
...
共 N 段旁白，請確認。

要繼續生成 TTS 嗎？（y / n）
n → 讓使用者手動編輯 manifest.json 中的 caption_short，編輯完成後回答 y 繼續
```

使用者回答 `y` 才繼續 Phase 4b。回答 `n` 時停下來等使用者手動修改，他們自己說 OK 後再繼續。

---

## Phase 4b：語音合成（可選）

### Step 10 — 安裝依賴

```bash
# Edge TTS（選用 Edge TTS 引擎時需要）
pip install edge-tts

# 台灣藍鵲 TTS（選用台灣藍鵲 TTS 引擎時需要）
pip install bluemagpie-tts soundfile
# 首次執行會自動下載 ~2GB 模型權重，請確保網路穩定
```

### Step 11 — 建立並執行 `generate_tts.py`

讀取 `temp/manifest.json` 中所有保留場景的 `caption_short`，依 ANS_4 選擇的 TTS 引擎合成音檔到 `temp/tts/`。

```python
#!/usr/bin/env python3
"""Phase 4b: TTS generation (Edge TTS / 台灣藍鵲 TTS).
Reads temp/manifest.json (with caption_short),
generates audio for each scene to temp/tts/,
writes tts_file path back to manifest.

Usage:
  # Edge TTS
  python3 generate_tts.py <素材路徑> --tts-engine edge --voice zh-TW-YunJheNeural

  # 台灣藍鵲 TTS
  python3 generate_tts.py <素材路徑> --tts-engine bluemagpie --speaker hung_yi_lee
  python3 generate_tts.py <素材路徑> --tts-engine bluemagpie --speaker female_voice
  python3 generate_tts.py <素材路徑> --tts-engine bluemagpie --speaker custom --centroid-path /path/to/voice.pt
"""
import json, os, sys, argparse, asyncio, subprocess, warnings
from pathlib import Path

# ------------------------------------------------------------------ #
# Edge TTS
# ------------------------------------------------------------------ #
async def generate_edge_tts(text, output_path, voice):
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


# ------------------------------------------------------------------ #
# BlueMagpie-TTS
# ------------------------------------------------------------------ #
_BLUEMAGPIE_MODEL = None


def _load_bluemagpie():
    global _BLUEMAGPIE_MODEL
    if _BLUEMAGPIE_MODEL is not None:
        return _BLUEMAGPIE_MODEL

    print("  Loading BlueMagpie-TTS (~20s first time, then cached)...", file=sys.stderr)
    warnings.filterwarnings("ignore", message=".*weight_norm.*")
    import torch
    from transformers import PreTrainedTokenizerFast
    from bluemagpie import BlueMagpieModel

    model_dir = os.path.expanduser(
        "~/.cache/huggingface/hub/models--OpenFormosa--BlueMagpie-TTS/"
        "snapshots/aaf1a0878e37875382bb0e5c8a3a2ba43be67297"
    )
    tokenizer = PreTrainedTokenizerFast(tokenizer_file=os.path.join(model_dir, "tokenizer.json"))
    device = "mps" if torch.backends.mps.is_available() else "cpu"
    model = BlueMagpieModel.from_local(model_dir, tokenizer=tokenizer, training=False, device=device)
    _BLUEMAGPIE_MODEL = model
    return model


def generate_bluemagpie_tts(text, output_path, speaker, centroid_path):
    import torch
    import soundfile as sf

    model = _load_bluemagpie()
    sr = model.sample_rate

    # Determine speaker centroid
    centroid = None
    if speaker == "custom":
        if centroid_path and os.path.isfile(centroid_path):
            centroid = torch.load(centroid_path, weights_only=True)
        else:
            print(f"  Warning: .pt not found at {centroid_path}, fallback to hung_yi_lee",
                  file=sys.stderr)
            speaker = "hung_yi_lee"

    # Append 句號 for natural ending
    text_clean = text.strip()
    if text_clean and text_clean[-1] not in ".。!！?？":
        text_clean += "。"

    torch.manual_seed(42)
    try:
        audio = model.generate(
            target_text=text_clean,
            speaker_centroid=centroid,
            cfg_value=2.0,
            inference_timesteps=20,
            max_len=2000,
            retry_badcase=True,
        )
    except RuntimeError as e:
        if "out of memory" in str(e).lower():
            print("  OOM, retrying with timesteps=10...", file=sys.stderr)
            if hasattr(torch.mps, "empty_cache"):
                torch.mps.empty_cache()
            audio = model.generate(
                target_text=text_clean,
                speaker_centroid=centroid,
                cfg_value=2.0,
                inference_timesteps=10,
                max_len=2000,
                retry_badcase=True,
            )
        else:
            raise

    wav = audio.squeeze().cpu().numpy()
    duration = wav.size / sr
    sf.write(output_path, wav, sr)
    return True, duration


# ------------------------------------------------------------------ #
# Main
# ------------------------------------------------------------------ #
async def main():
    parser = argparse.ArgumentParser(description="TTS generation")
    parser.add_argument("source_dir", help="Path to source media folder")
    parser.add_argument("--tts-engine", default="edge",
                        choices=["edge", "bluemagpie"],
                        help="TTS engine (edge=Edge TTS, bluemagpie=台灣藍鵲 TTS)")
    # Edge TTS options
    parser.add_argument("--voice", default="zh-TW-YunJheNeural",
                        help="edge-tts voice name")
    # BlueMagpie-TTS options
    parser.add_argument("--speaker", default="hung_yi_lee",
                        choices=["hung_yi_lee", "female_voice", "custom"],
                        help="BlueMagpie-TTS speaker")
    parser.add_argument("--centroid-path", default="",
                        help="Path to .pt file for --speaker custom")
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

    pending = [item for item in active if not item.get("tts_file")]

    if not pending:
        print("All scenes already have TTS files.")
        sys.exit(0)

    engine_name = "Edge TTS" if args.tts_engine == "edge" else "台灣藍鵲 TTS"
    print(f"Generating TTS for {len(pending)} scenes ({engine_name})...")

    for i, item in enumerate(pending):
        sid = item["scene"]
        text = item["caption_short"]

        if args.tts_engine == "edge":
            out_name = f"scene_{sid:04d}.mp3"
            out_path = tts_dir / out_name
            success = await generate_edge_tts(text, str(out_path), args.voice)
            if success and out_path.exists():
                dur = len(text) / 4
                status = "✅"
            else:
                status = "⚠️"
                dur = 0
        else:
            out_name = f"scene_{sid:04d}.wav"
            out_path = tts_dir / out_name
            try:
                ok, dur = generate_bluemagpie_tts(text, str(out_path),
                                                  args.speaker, args.centroid_path)
                status = "✅" if ok else "⚠️"
            except Exception as e:
                print(f"  Error: {e}", file=sys.stderr)
                status = "⚠️"
                ok = False
                dur = 0

        if status == "✅":
            item["tts_file"] = str(out_path)
            item["tts_duration_sec"] = round(dur, 1)

        print(f"  [{sid:04d}/{len(active)}] {status} {text} -> {out_name}", flush=True)

        if (i + 1) % 5 == 0 or i == len(pending) - 1:
            with open(manifest_path, "w", encoding="utf-8") as f:
                json.dump(manifest, f, ensure_ascii=False, indent=2)

    print(f"\n✅ Done. {len(pending)} TTS files in {tts_dir}")


if __name__ == "__main__":
    asyncio.run(main())
```

**執行（依 ANS_4 選擇）：

```bash
# Edge TTS — 男聲（預設）
python3 "$SCRIPTS_DIR/generate_tts.py" "ANS_1" --tts-engine edge --voice zh-TW-YunJheNeural

# Edge TTS — 女聲（依 ANS_4a）
python3 "$SCRIPTS_DIR/generate_tts.py" "ANS_1" --tts-engine edge --voice zh-TW-HsiaoChenNeural

# 台灣藍鵲 TTS — 李宏毅老師（依 ANS_4b）
python3 "$SCRIPTS_DIR/generate_tts.py" "ANS_1" --tts-engine bluemagpie --speaker hung_yi_lee

# 台灣藍鵲 TTS — 通用女聲（依 ANS_4b）
python3 "$SCRIPTS_DIR/generate_tts.py" "ANS_1" --tts-engine bluemagpie --speaker female_voice

# 台灣藍鵲 TTS — 自己的聲音（依 ANS_4b + ANS_4c）
python3 "$SCRIPTS_DIR/generate_tts.py" "ANS_1" --tts-engine bluemagpie --speaker custom --centroid-path "ANS_4c"
```

**產出：**
- `temp/tts/scene_0001.mp3` ~ `scene_NNNN.mp3` — 每場景旁白音檔
- `temp/manifest.json` 更新，每個 entry 新增 `tts_file` 與 `tts_duration_sec`

### Step 11b — 音檔統一音量（loudnorm）

不同 TTS 片段間音量可能不一致，用 FFmpeg EBU R128 loudnorm 全部正規化到 -16 LUFS（spoken word 標準）。

**Edge TTS 產出是 MP3 → 先轉 WAV 再 normalize，最後轉回 MP3：**

```bash
TTS_DIR="ANS_1/temp/tts"

for f in "$TTS_DIR"/*.mp3; do
  tmp_wav="${f%.mp3}_tmp.wav"
  ffmpeg -y -v quiet -i "$f" -af loudnorm=I=-16:LRA=1:TP=-1 "$tmp_wav"
  ffmpeg -y -v quiet -i "$tmp_wav" "$f"
  rm "$tmp_wav"
done
```

**藍鵲 TTS 產出是 WAV → 直接 in-place：**

```bash
TTS_DIR="ANS_1/temp/tts"

for f in "$TTS_DIR"/*.wav; do
  tmp="${f%.wav}_norm.wav"
  ffmpeg -y -v quiet -i "$f" -af loudnorm=I=-16:LRA=1:TP=-1 "$tmp"
  mv "$tmp" "$f"
done
```

### Step 11c — 每段 TTS 淡入 0.5 秒

讓每段旁白開頭不會突兀切入：

```bash
TTS_DIR="ANS_1/temp/tts"

# WAV（藍鵲 TTS）
for f in "$TTS_DIR"/*.wav; do
  tmp="${f%.wav}_fade.wav"
  ffmpeg -y -v quiet -i "$f" -af afade=t=in:d=0.5 "$tmp"
  mv "$tmp" "$f"
done

# MP3（Edge TTS）
for f in "$TTS_DIR"/*.mp3; do
  tmp="${f%.mp3}_fade.mp3"
  ffmpeg -y -v quiet -i "$f" -af afade=t=in:d=0.5 "$tmp"
  mv "$tmp" "$f"
done
```

---

## Phase 4c：CC0 背景音樂（依 ANS_5）

**若 ANS_5 為「不需要」，跳過此階段。**

### Step — 搜尋 CC0 BGM

用 `websearch` 搜尋無版權音樂，關鍵字範例：

```
pixabay music [風格] free download royalty free background music
uppbeat [style] free background music
youtube audio library [genre] no copyright
```

例如 ANS_5 = 「輕鬆」：

```
pixabay music relaxing travel background music free download
```

### Step — 確認授權並下載

從搜尋結果中選一首：
- 確認標示 **CC0 / Royalty Free / No Copyright**
- 用 `webfetch` 或直接 `import_media` 下載 MP3 到 `temp/bgm/`

```bash
# 下載 BGM 到 temp/bgm/
import_media(source={url: "https://..."})
# 或手動下載
curl -L -o "ANS_1/temp/bgm/bgm.mp3" "https://..."
```

### Step — 記錄 BGM 資訊

在 `temp/manifest.json` 寫入 BGM 資訊：

```python
import json
with open("ANS_1/temp/manifest.json") as f:
    manifest = json.load(f)

# 視 manifest 為 dict，在最上層寫入 bgm
bgm_info = {
    "bgm_file": "ANS_1/temp/bgm/bgm.mp3",
    "bgm_name": "曲名",
    "bgm_duration": 120,  # 秒
    "bgm_license": "CC0 / Pixabay",
}

# 或存在 manifest 最尾端（若為 list 則用此方式）
with open("ANS_1/temp/manifest.json", "w") as f:
    json.dump(manifest, f, ensure_ascii=False, indent=2)
```

**產出：**
- `temp/bgm/bgm.mp3` — 下載的 CC0 背景音樂
- `temp/manifest.json` 記錄 BGM 路徑與授權

---

## Phase 5：輸出四種格式

### Step 12 — 建立並執行 `export_scripts.py`

```python
#!/usr/bin/env python3
"""Phase 5: Export output formats.
Reads temp/manifest.json (with caption_short + tts_file),
calculates cumulative timing,
outputs subtitles.srt + script_narrative.txt

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
    parser = argparse.ArgumentParser(description="Export subtitles + narrative script")
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

    # --- Output 1: subtitles.srt ---
    srt_path = source_dir / "subtitles.srt"
    with open(srt_path, "w", encoding="utf-8") as f:
        for i, item in enumerate(active):
            start = item["_start_sec"]
            end = item["_end_sec"]
            text = item["caption_short"]
            f.write(f"{i+1}\n{fmt_time(start)} --> {fmt_time(end)}\n{text}\n\n")
    print(f"📄 {srt_path} ({len(active)} entries)")

    # --- Output 2: script_narrative.txt ---
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
- `subtitles.srt` — 標準字幕
- `script_narrative.txt` — 腳本文字

確認 `temp/tts/` 下有對應的 `.mp3` 音檔。
確認 `temp/bgm/` 下有 `.mp3` BGM（如 ANS_6 有指定）。

---

## 輸出檔案一覽

執行完成後，素材資料夾內會產生：

```
素材資料夾/
├── subtitles.srt              ← → 字幕檔
├── script_narrative.txt      ← → 給人看的腳本
└── temp/                     ← 暫存檔（可清空）
    ├── manifest.json         ← 完整場景清單（含描述、旁白、TTS、BGM）
    ├── scripts/
    │   ├── run_scene_detect.py
    │   ├── qwen_describe.py
    │   ├── generate_tts.py
    │   └── export_scripts.py
    ├── scenes/               ← 切割後的獨立場景 MP4
    ├── thumbs/               ← 480px 縮圖（視覺描述用）
    ├── tts/                  ← TTS 旁白音檔（Edge TTS → MP3 / 藍鵲 → WAV）
    └── bgm/                  ← CC0 背景音樂 MP3
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

### BGM 音軌（Palmier Pro 側）

```python
# 1. 匯入 BGM
bgm_ref = import_media(source={path: "素材路徑/temp/bgm/bgm.mp3"}).mediaRef

# 2. 取得總長度（seconds 或 total_frames / fps）
total_sec = total_frames / fps

# 3. 放 BGM（與 TTS 不同軌）
#    ⚠️ 不可 omit trackIndex！BGM 會進 TTS 同一軌，蓋掉旁白！
#    ⭐ 指定 TTS 音軌的下一個 index（TTS 在 A1 → BGM 在 A2）
tl = get_timeline()
audio_indices = [t.index for t in tl.tracks if t.type == "audio"]
bgm_track_index = max(audio_indices) + 1

#    ⭐ 用 source 取代 endFrame（避免超過 BGM 真實長度）
bgm_clip_id = add_clips(entries=[{
    "mediaRef": bgm_ref,
    "startFrame": 0,
    "source": [0, total_sec],
    "trackIndex": bgm_track_index
}])[0].id  # 取 clips[0].id 需視實際 SDK 語法

# 4. 調整 BGM 音量（不要蓋過旁白）
set_clip_properties(clipIds=[bgm_clip_id], volume=0.08)

# 5. 淡入淡出
set_keyframes(clipId=bgm_clip_id, property="volume",
              keyframes=[[0, 0], [60, 0.08], [total_sec*fps-60, 0.08], [total_sec*fps, 0]])
```

## 經驗教訓

| # | 教訓 | 說明 |
|:-:|:----|:------|
| 1 | **場景偵測後先看 manifest 再寫旁白** | 先確認場景數量與內容，避免寫到一半發現場景順序不對 |
| 2 | **Qwen 描述僅供參考，檔名才是真的** | 視覺模型對手機螢幕錄影、模糊背景的場景正確率低 |
| 3 | **同一檔案多段子場景要先過濾再寫旁白** | 若先寫完再發現要跳過，會浪費時間 |
| 4 | **旁白每句控制 15-20 字** | 太長的旁白在 5 秒場景內唸不完 |
| 5 | **HEIC 照片直接給 Palmier 處理** | 不需預轉 JPG，Palmier 原生支援 |
| 6 | **先 remove_silence 再放 BGM** | silence 的 ripple 會切裂已定位的 BGM track |
| 7 | **TTS 音檔在 Palmier 側上旁白軌** | Edge TTS 產生的 MP3（或藍鵲 TTS 產生的 WAV）經 `import_media` 匯入後，用 `add_clips` 放到獨立的旁白音軌，不要混入原始素材音軌 |
| 8 | **BGM 音量要低於旁白** | 背景音樂設 volume 0.05-0.1，不要蓋過原片音軌或 TTS。淡入 2s、淡出 2s |
| 9 | **BGM 在 timeline 定位後不要 ripple** | `remove_silence()` 或 ripple 操作會切裂已定位的 BGM track。先定位素材、silence、TTS，最後才放 BGM |
| 10 | **確認 CC0 授權** | 搜到的 BGM 一定要確認是 CC0 / Royalty Free，避免版權爭議 |
| 16 | **BGM 用 `trackIndex` 放不同軌，不要 omit** | TTS 先放（omit trackIndex → A1），BGM 後放。若 BGM 也 omit trackIndex 會擠進 TTS 的 A1 軌，蓋掉旁白。解法：`get_timeline()` 找 A1 index → BGM 指定 `trackIndex: A1_index + 1`，Palmier 會開 A2 |
| 11 | **場景偵測用 `showinfo` 不是 `setpts`** | `select='gt(scene,0.15)',setpts=N/FRAME_RATE/TB` 無法輸出正確的來源影格編號。改用 `select='gt(scene,0.15)',showinfo` 並解析 `pts_time` 來取得場景切換時間點 |
| 12 | **合併 <1s 的走道晃動片段** | 走道拍攝、鏡頭快速移動時，ffmpeg 場景偵測會產生大量 <0.5s 的假陽性。合併門檻設 1 秒，低於此的片段往前/後合併 |
| 13 | **Qwen 描述出現「模糊」＝跳過** | `qwen_desc` 若包含「模糊」、「無法辨識」、「黑色背景」等關鍵字，代表該片段是走道晃動或無意義畫面，直接跳過 |
| 14 | **同來源 App 錄影超過 5 段 → 只留 3-5 段** | 手機螢幕錄影的每個畫面內容差異很小，留開頭、關鍵操作、結尾各一即可，中間步驟全跳過 |
| 15 | **用 `source: [0, dur]` 取代 `endFrame`** | `endFrame` 若超過音檔真實長度，clip 會被拒絕。用 `source` 指定來源區間秒數最安全 |
