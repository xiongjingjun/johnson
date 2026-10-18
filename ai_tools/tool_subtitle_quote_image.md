# Subtitle Quote Image Tool

**Subtitle Quote Image Tool** is a utility designed to merge video screenshots from different timestamps into a single layout with stacked subtitle quotes.

---

## 📦 Dependencies (`requirements.txt`)

* **Pillow**: Handles image cropping, stitching, and exporting (JPG format).
* **imageio-ffmpeg**: Reads video files and performs precise frame extraction.
* **yt-dlp**: Fetches metadata, downloads videos, and retrieves subtitle tracks from remote URLs (e.g., Bilibili).

---

## 🛠️ Useful Scripts

### 1. Sample Frames (挑帧)

Generate a candidate overview image containing frames around specified time points:

```bash
python skills/native-subtitle-quote-image/scripts/native_subtitle_stitch.py sample VIDEO -t 30 -t 60 -t 90 -t 120 --around 0.8 --out focused-candidates.jpg
```

### 2. Preview Subtitle Band (确认字幕裁切区域)

Inspect and verify the vertical range for cropping subtitles:

```bash
python skills/native-subtitle-quote-image/scripts/native_subtitle_stitch.py band D:\ai_workspace\native-subtitle-quote-image\media_lib\test1.mp4 -t 30 --band-top 0.9 --band-bottom 0.96 --out band-preview.jpg
```

### 3. Render with Native Subtitles (使用原生字幕产出)

Generate stitched quote images using the video's embedded subtitles as defined in a manifest:

```bash
python skills/native-subtitle-quote-image/scripts/native_subtitle_stitch.py render D:\ai_workspace\native-subtitle-quote-image\media_lib\test1.mp4 --manifest manifest.json --out-dir output-v1 --band-top 0.88 --band-bottom 0.96
```

### 4. Render with Script Subtitles (使用脚本字幕产出)

Generate quote images using custom subtitles defined in a script file:

```bash
python skills/native-subtitle-quote-image/scripts/native_subtitle_stitch.py render-script D:\ai_workspace\native-subtitle-quote-image\media_lib\test1.mp4 --script script.json --out output_by_script.jpg --aspect 3:4 --width 1440 --frame-bottom 0.96
```

---

## 💡 Tips

> This repository is forked from [chengyi-ai/native-subtitle-quote-image](https://github.com/chengyi-ai/native-subtitle-quote-image). It is used for personal practice in extracting video subtitle screenshots and merging them into stacked rows of quotes.