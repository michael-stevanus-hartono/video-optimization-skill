# video-web-optimization

A [Claude Code](https://claude.com/claude-code) / Agent skill that compresses and optimizes video files for web delivery using FFmpeg — producing dual-source **VP9 (WebM)** + **H.265 (MP4)** output for broad browser support.

## How it works

1. Add the skill
2. Download [FFmpeg](https://www.ffmpeg.org/download.html)
3. Copy the path of the video or a folder of videos, e.g. `C:\Users\Lenovo\Downloads\video.mp4`
4. Tell your agent to compress it, e.g. "compress this video: C:\Users\Lenovo\Downloads\video.mp4"
5. Review the estimate (original size, expected wait, rough size reduction) and confirm. If a file is estimated at 30+ minutes, you'll be offered faster alternatives (hardware encoding, faster VP9 settings, downscaling) before committing
6. Get back two web-ready files (`.webm` and `.mp4`) per video in a `Compressed Videos` folder, plus a drop-in `<video>` snippet

## What it does

Point the agent at a video file (or a folder of them) and it will:

1. Verify FFmpeg is installed and inspect the source(s) — resolution, codec, duration.
2. Show a per-file estimate (original size, expected wait, rough size reduction) and ask for
   confirmation before encoding anything. Files estimated at 30+ minutes are flagged explicitly,
   with faster alternatives offered instead of a silent long wait.
3. On multiple files, process them **one at a time**, not concurrently — a single video's
   WebM+MP4 pair already uses most available CPU threads, so running whole videos in parallel
   doesn't reliably speed things up.
4. Encode two web-ready versions into a `Compressed Videos` folder, without ever modifying the
   original (existing output is never overwritten — a repeat run creates `Compressed Videos - 1`,
   `- 2`, etc.):
   - `*.webm` — VP9 (Chrome, Firefox, Android)
   - `*.mp4` — H.265/HEVC with `hvc1` tag + faststart (Safari, iOS, macOS)
5. Report the actual results (size, % saved, time taken) and hand back a drop-in `<video>` snippet.

It defaults to high-quality software encoding (`libvpx-vp9`, `libx265`) and only reaches for
hardware encoders (AMD AMF / NVIDIA NVENC / Intel QSV / Apple VideoToolbox) for fast drafts, long
clips, or when a long-encode estimate prompts you to opt into a faster path.

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

## Update

`npx skills add` clones this repo once and copies the files in — it doesn't stay linked to
GitHub, so pushes here don't reach anyone who already installed it. Pull the latest version
with:

```bash
npx skills update michael-stevanus-hartono/video-optimization-skill
```

Or update every skill installed via the `skills` CLI at once:

```bash
npx skills update
```

## Usage

Ask your agent something like:

> compress this video: C:\path\to\input.mp4

or point it at a folder to batch-process every video inside:

> compress these videos: C:\path\to\video-folder

Either way, you'll see an estimate and a yes/no confirmation before anything encodes. It will
produce `input-web.webm` and `input-web.mp4` in a `Compressed Videos` folder next to the source.

## Output HTML

```html
<video autoplay loop muted playsinline>
  <source src="output.webm" type="video/webm; codecs=vp9" />
  <source src="output.mp4" type="video/mp4; codecs=hevc" />
</video>
```

## Credit

Workflow and FFmpeg parameter guidance adapted from PixelPoint. This repo packages that technique as an installable agent skill — see PixelPoint for the original source material.

- YouTube video: [https://youtu.be/DP5SoQRf2Gk](https://youtu.be/DP5SoQRf2Gk?si=uKROiAZ74P5j0AbX)
- Blog post: [https://pixelpoint.io/blog/web-optimized-video-ffmpeg/](https://pixelpoint.io/blog/web-optimized-video-ffmpeg/)

## License

No formal license is claimed over the underlying technique, which originates with PixelPoint (see Credit above). This packaging is shared openly for anyone to install, use, and adapt. If you're affiliated with PixelPoint and want different attribution or have this taken down, please open an issue.
