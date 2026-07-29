---
name: video-web-optimization
description: Compress and optimize video files for web delivery using FFmpeg. Use when the user wants to compress a video, shrink a video file, prepare video for a website or hero background, convert to WebM/VP9 or MP4/H.265, or asks about FFmpeg flags for web video. Triggers on mentions of .mov, .mp4, ProRes, hero video, background video, or "video is too big".
---

# Web Video Optimization

Compress video for web using VP9 (WebM) + H.265 (MP4) dual-source delivery.
FFmpeg writes new files and never modifies the input. Never set an output
path equal to the input path.

## Workflow

1. Verify FFmpeg exists: `ffmpeg -version`. If missing, stop and tell the user
   to install it (`winget install ffmpeg` on Windows, `brew install ffmpeg` on macOS).
2. Inspect the source:
   `ffprobe -v error -show_entries stream=width,height,codec_name,duration -of default=noprint_wrappers=1 <input>`
3. Ask target resolution if unclear. Default 1920 wide for hero backgrounds;
   3840 only if source is 4K+ and quality is critical.
4. Create a `Compressed Videos` folder in the same directory as the source
   (skip if it already exists) and write both outputs there, so the `.webm`
   and `.mp4` for a clip stay together instead of scattering next to the source.
5. Encode both formats using the software commands below, launching both as
   background jobs at the same time rather than one after the other — the
   codecs are independent, so wall-clock time is bounded by the slower job
   instead of the sum of both. Note the wall-clock time the encode takes — it
   goes in the report.
6. Report results using the format below. Target ~2MB max for hero/background
   video. If over, raise CRF by 4 and re-encode.

## Report format

Always close with this summary:

```
input.mp4 has been successfully compressed!

- Original -> 38.4 MB
- Compressed .mp4 -> 0.96 MB
- Compressed .webm -> 1.75 MB

You saved 95% (36.6 MB) in 11 minutes.
```

- Filename is the source file's name, not the output names.
- Sizes in MB to one decimal place.
- The savings percentage and MB figure compare the original against the
  **larger** of the two outputs, since that is the worst case a browser fetches.
- Round encode time to the nearest minute; use seconds if under a minute.
- State the output folder path after the summary.

## Path handling (Windows)

- Accept `C:\Users\Name\file.mov` or `/c/Users/Name/file.mov` (Git Bash).
- Quote any path containing spaces.
- Do not use forward-double-slash form (`C://`).
- If the user gives a folder, list video files in it and confirm which to encode.
- The `Compressed Videos` output folder contains a space — always quote paths
  that reference it.

## Commands — software encoding (default, all platforms)

WebM / VP9 (desktop, Android):

```bash
ffmpeg -i "input.mov" -c:v libvpx-vp9 -crf 40 -vf scale=1920:-2 -deadline best -row-mt 1 -tile-columns 2 -an "output.webm"
```

MP4 / H.265 (Safari, iOS, macOS):

```bash
ffmpeg -i "input.mov" -c:v libx265 -crf 32 -vf scale=1920:-2 -preset slow -tag:v hvc1 -movflags faststart -pix_fmt yuv420p -an "output.mp4"
```

These are the correct default. Prefer them for short hero loops — quality per
byte is meaningfully better than any hardware encoder.

## Commands — hardware acceleration (drafts / long clips only)

Only use when the user explicitly asks for a fast encode or the source is long
(>60s). Check encoder availability first:

- Windows: `ffmpeg -hide_banner -encoders | findstr "amf nvenc qsv"`
- macOS:   `ffmpeg -hide_banner -encoders | grep videotoolbox`

AMD (Radeon / AMF) — Windows:

```bash
ffmpeg -i "input.mov" -c:v hevc_amf -quality quality -rc cqp -qp_i 26 -qp_p 26 -vf scale=1920:-2 -tag:v hvc1 -movflags faststart -pix_fmt yuv420p -an "output.mp4"
```

Nvidia (NVENC) — Windows:

```bash
ffmpeg -i "input.mov" -c:v hevc_nvenc -preset p7 -rc vbr -cq 30 -b:v 0 -vf scale=1920:-2 -tag:v hvc1 -movflags faststart -pix_fmt yuv420p -an "output.mp4"
```

Intel (QSV) — Windows:

```bash
ffmpeg -i "input.mov" -c:v hevc_qsv -global_quality 30 -preset veryslow -vf scale=1920:-2 -tag:v hvc1 -movflags faststart -pix_fmt yuv420p -an "output.mp4"
```

Apple (VideoToolbox) — macOS:

```bash
ffmpeg -i "input.mov" -c:v hevc_videotoolbox -b:v 2M -vf scale=1920:-2 -tag:v hvc1 -movflags faststart -an "output.mp4"
```

Hardware notes:
- AMF/NVENC/QSV ignore `-crf`. Use the rate-control flags shown above.
- If `hevc_amf` is unavailable, the AMD Adrenalin driver is likely not installed.
  Fall back to `libx265` rather than failing.
- Integrated GPUs (Vega, UHD) produce visibly softer output than `libx265`.
  Warn the user before using them for final assets.
- VP9 has no meaningful hardware encode path. WebM always uses `libvpx-vp9`.

## Flags

| Flag | Purpose | Value |
| --- | --- | --- |
| `-crf` | Constant quality, software encoders only. Higher = smaller. | VP9 `40` (0–63), H.265 `32` (0–51) |
| `-vf scale=W:-2` | Resize; `-2` keeps even dimensions | `1920:-2` or `3840:-2` |
| `-vf unsharp=5:5:1.0` | Sharpen after downscale to ≤1080p | append to `-vf` chain |
| `-an` | Strip audio (smaller + autoplay-safe) | always |
| `-deadline best` | VP9 quality mode | always |
| `-row-mt 1` | Enables row-based multithreading in libvpx-vp9 | always (near-zero quality cost) |
| `-tile-columns 2` | Splits frame into parallel-encodable column tiles | always (small quality cost at high tile counts) |
| `-preset slow` | libx265 compression efficiency/speed balance | always (~40% faster than `veryslow` for ~4% larger file at same CRF) |
| `-tag:v hvc1` | Required for Safari/iOS H.265 playback | always |
| `-movflags faststart` | Moves moov atom to head for streaming | always on MP4 |
| `-pix_fmt yuv420p` | Fixes green/black screen on old devices | always on MP4 |
| `-g 1` | Every frame a keyframe | scroll-driven video only |
| `-frames:v 1` | Single-frame test render | quality checks |

## Rules

- Source should be a ProRes or high-bitrate master. Warn if the input is already
  a compressed web export — re-encoding compounds artifacts.
- Prefer downscaling over higher CRF when a file is too large.
- When scaling to 1080p or below, chain unsharp: `-vf scale=1920:-2,unsharp=5:5:1.0`
- Never suggest `poster` on an autoplay video — browsers fetch both assets.
- Use `-g 1` only for scroll-scrubbed video; it inflates file size significantly.

## Output HTML

```html
<video autoplay loop muted playsinline>
  <source src="output.webm" type="video/webm; codecs=vp9" />
  <source src="output.mp4" type="video/mp4; codecs=hevc" />
</video>
```
