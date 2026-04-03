---
name: video-feedback
description: >
  Analyze video feedback recordings (Loom or local MP4/MOV/WebM) to extract visual
  and audio insights, then produce actionable reports. Use when the user shares a Loom
  URL, provides a video file path, or says "analyze this video", "here's my feedback
  recording", "review this Loom", or any request to understand and act on video feedback.
  Supports screen recordings, UI walkthroughs, tutorial reviews, and general video feedback.
---

# Video Feedback Analyzer

Analyze video recordings to extract what the user said (audio transcription) and what they showed (frame analysis), then produce an actionable feedback report tied to the relevant project context.

## Architecture: Context-Efficient Pipeline

```
Main Agent                          Sub-Agents (disposable context)
──────────                          ──────────────────────────────
1. Download video (yt-dlp)   ───>
2. Extract metadata (ffprobe) ──>
3. Extract frames (ffmpeg)   ───>
4. Extract audio (ffmpeg)    ───>
5. Transcribe audio (whisper) ──>
6. Split frames into batches ───>  7. Read images (vision)
                                      Write text descriptions
                                      to batch_N_analysis.md
8. Read text files only      <──   (context discarded)
9. Sync transcript + visual  ───>
10. Resolve project context  ───>
11. Produce actionable report
```

Images only ever exist inside sub-agent contexts. The main agent only reads lightweight text files. This cuts context usage by ~90%.

## 1. Prerequisites

```bash
which ffmpeg && which ffprobe && which yt-dlp && python3 -c "import whisper" 2>/dev/null && echo "All prerequisites OK" || echo "MISSING DEPENDENCIES"
```

If any are missing, tell the user what to install and STOP:
- **ffmpeg/ffprobe**: `sudo apt install ffmpeg` (Linux) or `brew install ffmpeg` (macOS)
- **yt-dlp**: `pip install yt-dlp` or `brew install yt-dlp`
- **whisper**: `pip install openai-whisper` (local transcription, no API key needed)

## 2. Setup

Use a working-directory-relative temp folder so sub-agents have permission to read/write files.

```bash
TMPDIR="$(pwd)/.video-feedback-tmp"
mkdir -p "$TMPDIR"
```

Previous files in this directory will be overwritten. No need to clean up first.

**Important:** This directory must be inside the current working directory (not `/tmp/`) because sub-agents are sandboxed to the project directory and cannot access `/tmp/`.

## 3. Acquire Video

Determine the input type:

**If Loom URL** (matches `loom.com/share/`):

```bash
yt-dlp --force-overwrites --merge-output-format mp4 -o "$TMPDIR/video.mp4" "LOOM_URL"
```

If yt-dlp fails (private video, auth required, extractor broken):
1. Tell the user: "Loom download failed. Please download the MP4 manually from Loom and give me the file path."
2. STOP and wait for a local file path.

**If other video URL** (YouTube, Vimeo, etc.):

```bash
yt-dlp --force-overwrites --merge-output-format mp4 -o "$TMPDIR/video.mp4" "VIDEO_URL"
```

**If local file path** (ends in .mp4, .mov, .webm, .avi, .mkv):

```bash
cp "FILE_PATH" "$TMPDIR/video.mp4"
```

If the file doesn't exist, tell the user and STOP.

Set `VIDEO="$TMPDIR/video.mp4"` for all subsequent steps.

## 4. Extract Video Metadata

```bash
ffprobe -v quiet -print_format json -show_format -show_streams "$VIDEO"
```

Extract and note: duration, resolution (width x height), fps, codec, file size, whether audio is present.

If no video stream is found, report "audio-only file" and STOP.
If file size > 2GB, warn the user and suggest analyzing a time range.

## 5. Extract Frames

Choose strategy based on duration:

| Duration | Strategy | Command |
|----------|----------|---------|
| 0-60s | 1 frame every 2s | `ffmpeg -hide_banner -y -i "$VIDEO" -vf "fps=1/2,scale='min(1280,iw)':-2" -q:v 5 "$TMPDIR/frame_%04d.jpg"` |
| 1-10min | Scene detection (threshold 0.3) | `ffmpeg -hide_banner -y -i "$VIDEO" -vf "select='gt(scene,0.3)',scale='min(1280,iw)':-2" -vsync vfr -q:v 5 "$TMPDIR/scene_%04d.jpg"` |
| 10-30min | Keyframe extraction | `ffmpeg -hide_banner -y -skip_frame nokey -i "$VIDEO" -vf "scale='min(1280,iw)':-2" -vsync vfr -q:v 5 "$TMPDIR/key_%04d.jpg"` |
| 30min+ | Thumbnail filter | `ffmpeg -hide_banner -y -i "$VIDEO" -vf "thumbnail=SEGMENT_FRAMES,scale='min(1280,iw)':-2" -vsync vfr -q:v 5 "$TMPDIR/thumb_%04d.jpg"` |

For thumbnail filter, calculate `SEGMENT_FRAMES = total_frames / 60` to cap output at ~60 frames.

**Fallbacks:**
- Scene detection yields 0 frames -> retry with interval at 1 frame/5s
- More than 100 frames extracted -> subsample evenly to 80
- Frame extraction fails -> try the next simpler strategy (scene -> interval, keyframe -> interval)

After extraction, list all frame files and calculate each frame's timestamp from its sequence number and the extraction rate.

## 6. Extract Audio

If metadata shows audio is present:

```bash
ffmpeg -hide_banner -y -i "$VIDEO" -vn -acodec pcm_s16le -ar 16000 -ac 1 "$TMPDIR/audio.wav"
```

If no audio stream exists, skip transcription entirely and note "No audio track -- visual analysis only."

## 7. Transcribe Audio

```bash
python3 -c "
import whisper, json
model = whisper.load_model('base')
result = model.transcribe('$TMPDIR/audio.wav', word_timestamps=True)
with open('$TMPDIR/transcript.json', 'w') as f:
    json.dump(result, f, indent=2)
print('Transcription complete:', len(result['segments']), 'segments')
"
```

Then read `$TMPDIR/transcript.json`. Each segment has `start`, `end`, and `text` fields.

Format transcript segments for reference:

```
[0:00-0:05] "First thing I notice is the header is too crowded"
[0:05-0:12] "If you look at the spacing between these nav items..."
```

**Quality note:** If transcript quality is poor (garbled, missing words), the user can re-run with a larger model by setting `WHISPER_MODEL=small` or `WHISPER_MODEL=medium`. The base model is fastest but least accurate.

## 8. Delegate Frame Analysis to Sub-Agents

**Do NOT read frame images in the main conversation.** Split frames into batches and delegate to sub-agents.

### 8a. Prepare Batch Manifest

Split extracted frame files into batches of 8-10 frames each. For each batch, record:
- Batch number (1, 2, 3, ...)
- Frame file paths (absolute)
- Frame timestamps
- Output file path: `$TMPDIR/batch_N_analysis.md`

### 8b. Spawn Sub-Agents

For each batch, spawn a sub-agent with this prompt. **Launch all batches in parallel** -- they are fully independent.

#### Sub-Agent Prompt Template

```
You are analyzing frames extracted from a video feedback recording. The user was narrating feedback while navigating a UI or document. Pay special attention to:
- Where the mouse cursor is pointing (this indicates what the user is talking about)
- Any UI elements that appear highlighted, selected, or interacted with
- Visible text, labels, URLs, file names, error messages
- Changes between consecutive frames (what the user navigated to)

VIDEO: {filename}
DURATION: {duration}
BATCH: {batch_number} of {total_batches}

Read each frame image listed below using the Read tool. For each frame, write a structured description.

FRAMES:
{for each frame in batch}
- {absolute_path_to_frame} (timestamp: {MM:SS})
{end for}

For each frame, describe:
1. SCENE: What is visible (layout, UI elements, page/document structure)
2. CURSOR: Where the mouse cursor is positioned and what it's near/on
3. CONTENT: Text, code, labels, menus, URLs, or data visible on screen
4. ACTION: What changed since the likely previous frame
5. DETAILS: Error messages, button states, form values, anything notable

After describing all frames, add a BATCH SUMMARY section with:
- Content type (Screencast, UI Review, Document Review, Presentation, Tutorial, Other)
- Key UI elements or pages shown in this batch
- Any text/URLs/file names visible (quote exactly)

Write the complete analysis to: {TMPDIR}/batch_{N}_analysis.md
```

Use the Agent tool to spawn each sub-agent. If the tool supports parallel launches, launch all batches simultaneously.

**Fallback:** If sub-agents fail (permission errors, timeouts), read frames directly in the main context. Subsample to **20 frames maximum** -- pick every Nth frame to get even coverage. Warn about context usage for longer videos.

### 8c. Collect Results

After all sub-agents complete, read all batch analysis files in order:

```bash
ls "$TMPDIR"/batch_*_analysis.md
```

Read each file. These contain only text -- no images enter the main context.

## 9. Synthesize: Sync Transcript with Visual Analysis

This is the core value step. Merge the Whisper transcript (what the user said) with the frame analysis (what was on screen) into a unified timeline.

For each transcript segment:
1. Find the frame(s) closest in timestamp
2. Pair the spoken words with the visual context
3. Identify what the user was referring to based on cursor position + speech

Produce a **Feedback Extract** -- timestamped entries like:

```
[0:32] "This spacing is way too tight"
  -> Screen shows: Card grid with ~4px gap between items. Cursor positioned between two cards.
  -> Feedback: Increase spacing between grid items.

[1:15] "And this button should be more prominent"
  -> Screen shows: Muted gray CTA button, bottom-right of hero section. Cursor on the button.
  -> Feedback: Make CTA button more visually prominent (size, color, or contrast).
```

## 10. Resolve Project Context

Before generating the action plan, determine what project/files this feedback applies to:

1. **Check conversation context** -- if already working on a project in this session, map feedback to those files.
2. **Check user-provided context** -- the user may have prefixed their message with context like "Feedback on the event calendar mobile view."
3. **Use clues from the video** -- visible URLs, page titles, project names mentioned in narration, file paths shown on screen.
4. **Search if needed** -- use Glob/Grep to find relevant files based on clues from steps 1-3.
5. **If unclear** -- ask the user which project/files this applies to before proceeding.

**Important:** Not all feedback maps to code. It could be about a document, email draft, marketing plan, design, or anything else. Don't assume it's always a codebase change.

## 11. Produce Action Plan

Based on the feedback extract and resolved context, generate one of two outputs:

**For complex feedback** (multiple changes, new features, unclear scope, or new task kickoff):

```markdown
# Video Feedback Report: [brief description]

## Feedback Extract
[timestamped entries from step 9]

## Identified Changes
1. [Specific change] -- [which file/location if known]
2. [Specific change] -- [which file/location if known]
...

## Recommended Next Step
[Suggest: invoke brainstorming skill, create a plan, or list specific changes for approval]
```

**For simple feedback** (clear, small changes on a known project):

```markdown
# Video Feedback: [brief description]

## Changes to Make
1. [File: path] -- [specific change]
2. [File: path] -- [specific change]
...

Ready to implement these changes?
```

Choose simple vs. complex based on:
- Number of distinct changes (>3 = complex)
- Whether project context is clear (unclear = complex)
- Whether changes require design decisions (yes = complex)

## 12. Cleanup

The `.video-feedback-tmp/` directory is gitignored and will be overwritten on the next run. No cleanup needed -- leave the files in place. If the user explicitly asks to clean up, tell them to run `rm -rf .video-feedback-tmp` manually.
