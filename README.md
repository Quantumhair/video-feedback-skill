# Video Feedback Skill for Claude Code

A [Claude Code](https://claude.ai/claude-code) skill that analyzes video recordings (Loom, local files, or any yt-dlp-supported URL) to extract what was said and what was shown on screen, then produces actionable feedback reports.

Record a Loom walkthrough of a UI bug, paste the link, and get back a timestamped report mapping your narration to specific screen elements -- ready to act on.

## How it works

1. Downloads the video (yt-dlp)
2. Extracts frames at smart intervals (ffmpeg)
3. Transcribes audio locally (OpenAI Whisper -- no API key needed)
4. Analyzes frames in parallel sub-agents (Claude vision)
5. Syncs transcript with visual analysis into a unified timeline
6. Maps feedback to your project files and produces an action plan

The architecture is context-efficient: images are only read inside disposable sub-agents. The main conversation only sees lightweight text descriptions, cutting context usage by ~90%.

## Install

```bash
# 1. Copy the skill file
mkdir -p ~/.claude/skills/video-feedback
curl -o ~/.claude/skills/video-feedback/SKILL.md \
  https://raw.githubusercontent.com/Quantumhair/video-feedback-skill/main/SKILL.md

# 2. Install dependencies
pip install yt-dlp openai-whisper    # or: brew install yt-dlp && pip install openai-whisper
sudo apt install ffmpeg              # Linux -- macOS: brew install ffmpeg
```

That's it. Claude Code will auto-detect the skill when you mention video feedback.

## Dependencies

| Tool | Purpose | Install |
|------|---------|---------|
| [ffmpeg](https://ffmpeg.org/) + ffprobe | Frame/audio extraction | `apt install ffmpeg` or `brew install ffmpeg` |
| [yt-dlp](https://github.com/yt-dlp/yt-dlp) | Video download (Loom, YouTube, etc.) | `pip install yt-dlp` |
| [openai-whisper](https://github.com/openai/whisper) | Local audio transcription | `pip install openai-whisper` |

All free, open-source, runs locally. No API keys required.

## Usage

Just share a video with Claude Code:

```
> Here's feedback on the dashboard: https://www.loom.com/share/abc123

> Analyze this video: /path/to/recording.mp4

> Review this Loom and fix what I'm talking about: https://www.loom.com/share/xyz789
```

The skill triggers automatically when Claude Code detects a Loom URL, video file path, or phrases like "analyze this video" or "review this Loom."

### What it supports

- **Loom recordings** (most common use case)
- **Local video files** (.mp4, .mov, .webm, .avi, .mkv)
- **Any yt-dlp-supported URL** (YouTube, Vimeo, etc.)
- **Videos up to ~30 minutes** (longer videos work but use more context)
- **With or without audio** (visual-only analysis if no audio track)

## Example output

```
[0:17] "There's uneven spacing throughout the page"
  -> Screen shows: Submit Event form, cursor between form sections.
  -> Feedback: Inconsistent vertical gaps between form section cards.

[0:28] "The date and time block and location block are up against each other"
  -> Screen shows: Date & Time and Location sections with zero gap.
  -> Feedback: Add consistent spacing between all form sections.
```

## How Claude Code skills work

Skills are markdown instruction files that teach Claude Code new capabilities. They live in `~/.claude/skills/<name>/SKILL.md` and are automatically invoked when relevant based on their description.

No compilation, no plugins, no package manager -- just a text file.

## License

MIT
