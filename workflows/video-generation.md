# Video Generation Workflow

End-to-end walkthrough of what happens between `POST /v1/video/generate` and a downloadable MP4.

---

## Overview

```
User Request
     │
     ▼
┌─────────────────────────────────┐
│  1. Job Creation                │
│     Create JobRecord (queued)   │
│     Return job_id immediately   │
└──────────────┬──────────────────┘
               │ async
               ▼
┌─────────────────────────────────┐
│  2. Script Generation           │
│     Topic + theme → Gemini      │
│     Structured script returned  │
│     (speakers, lines, SFX cues) │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  3. Audio Synthesis             │
│     Each line → ElevenLabs TTS  │
│     Character voices per theme  │
│     Audio segments assembled    │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  4. SFX Selection & Timing      │
│     Script cues → SFX library   │
│     Local assets first          │
│     ElevenLabs generation       │
│     as fallback                 │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  5. Video Assembly              │
│     Background video + audio    │
│     mix → master track          │
│     Subtitle rendering (PIL)    │
│     SFX timing applied          │
│     FFmpeg final render         │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  6. Job Completion              │
│     File written to output/     │
│     JobRecord → completed       │
│     Download URL available      │
└─────────────────────────────────┘
```

---

## Step 1 — Job Creation

The API endpoint creates a `JobRecord` with status `queued` and returns the `job_id` to the client in `202 Accepted`. The heavy work is handed off to a background task immediately — the HTTP connection is not held open.

The output path is computed deterministically at this point: `output/{sanitized_topic}_{job_id}.mp4`. The `job_id` suffix ensures two concurrent jobs on the same topic never write to the same file.

---

## Step 2 — Script Generation

The topic and theme are passed to the script generator. The theme drives:
- Which AI model configuration is used
- The speaker configuration (dialogue vs. monologue)
- The tone, pacing, and structural arc of the content
- The SFX vocabulary available for the script

The generator returns a structured script: a sequence of segments, each with a speaker, line of dialogue, and an optional sound effect cue.

---

## Step 3 — Audio Synthesis

Each segment is sent to ElevenLabs for text-to-speech synthesis. Speakers are assigned distinct voices per theme. The audio pipeline is sequential — ElevenLabs rate limits are respected via a semaphore, not unbounded parallelism.

Each segment's real audio duration is measured after synthesis and stored — this duration drives subtitle timing and SFX placement in subsequent steps.

---

## Step 4 — SFX Selection and Timing

Sound effect cues from the script are resolved against the SFX library. The selection process uses a two-tier approach:

1. **Local library first** — scored against the cue's requirements
2. **ElevenLabs generation** — used as fallback when no local asset meets the quality threshold

Selected SFX assets are timed precisely to the script — placement at the start or end of a segment, with fine-grained offset control and volume adjustment per cue.

---

## Step 5 — Video Assembly

All assets are composited into the final video:

- Background video is selected from the theme's asset pool and trimmed or looped to match the total audio duration
- Background music is mixed with the voice track at the theme's configured volume level
- SFX events are mixed into the master audio at their computed timestamps
- Subtitles are rendered as PIL images and composited as timed ImageClips
- FFmpeg renders the final MP4 at target quality settings

---

## Step 6 — Job Completion

The output file is verified to exist. If it does, the `JobRecord` is updated to `completed` with the output path stored. If it does not exist (pipeline returned without writing), the job transitions to `failed`.

The download URL becomes available. The video is retained for `VIDEO_RETENTION_HOURS` (default 72h), after which the file is deleted and the job transitions to `expired`.

---

## Operating Modes

| Mode | Script source | Audio source | Use case |
|---|---|---|---|
| `DEBUG` | Pre-written mock JSON | Pre-recorded MP3 files | Development, CI, integration testing |
| `PROD` | Google Gemini API | ElevenLabs API | Real content generation |

Mode is controlled by the `APP_MODE` environment variable. The pipeline code is identical — only the service implementations swap.
