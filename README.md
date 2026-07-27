# video-web-optimization

A [Claude Code](https://claude.com/claude-code) / Agent skill that compresses and optimizes video files for web delivery using FFmpeg — producing dual-source **VP9 (WebM)** + **H.265 (MP4)** output for broad browser support.

## What it does

Point the agent at a video file and it will:

1. Verify FFmpeg is installed and inspect the source (resolution, codec, duration).
2. Encode two web-ready versions without ever modifying the original:
   - `*.webm` — VP9 (Chrome, Firefox, Android)
   - `*.mp4` — H.265/HEVC with `hvc1` tag + faststart (Safari, iOS, macOS)
3. Strip audio, target ~2 MB for hero/background loops, and report final file sizes.
4. Hand back a drop-in `<video>` snippet.

It defaults to high-quality software encoding (`libvpx-vp9`, `libx265`) and only reaches for hardware encoders (AMD AMF / NVIDIA NVENC / Intel QSV / Apple VideoToolbox) for fast drafts or long clips.

## Requirements

- [FFmpeg](https://ffmpeg.org/) on your PATH (`ffmpeg -version` should work)
  - Windows: `winget install ffmpeg`
  - macOS: `brew install ffmpeg`

## Install

With the [`skills`](https://github.com/) CLI:

```bash
npx skills add michael-stevanus-hartono/video-optimization-skill
```

Or manually, copy `SKILL.md` into your skills directory:

```
~/.claude/skills/video-web-optimization/SKILL.md
```

## Usage

Ask your agent something like:

> compress this video: C:\path\to\input.mp4

It will produce `input-web.webm` and `input-web.mp4` alongside the source.

## Output HTML

```html
<video autoplay loop muted playsinline>
  <source src="output.webm" type="video/webm; codecs=vp9" />
  <source src="output.mp4" type="video/mp4; codecs=hevc" />
</video>
```

## License

MIT
