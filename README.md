# Shorts Media Factory

An agentic AI pipeline that transforms a topic into a fully produced short-form video — script, voiceover, sound effects, and final render — with a single API call.

The human acts as Creative Director. The AI handles the production friction.

---

## What it does

1. User submits a topic and theme via API
2. Gemini generates a themed, structured script
3. ElevenLabs synthesizes character voiceovers and sound effects
4. The pipeline assembles audio, background video, subtitles, and SFX into a final MP4
5. The video is available for download within the retention window

One API call. One video. No editing software required.

---

## Market validation

One of the early test videos generated with this engine hit **23k views and 1k likes on TikTok** at the proof-of-concept stage — before any user-facing product existed.

The market is not looking for AI-generated noise. It is looking for high-quality, themed content where the AI handles production and the human controls the creative direction.

---

## Technology stack

| Layer | Technology |
|---|---|
| API | FastAPI (Python 3.12) |
| Script generation | Google Gemini |
| Voice synthesis | ElevenLabs |
| Video assembly | MoviePy + FFmpeg |
| Job state | PostgreSQL (SQLModel) |
| Infrastructure | Docker + docker-compose |
| Auth (roadmap) | Clerk / Supabase JWT |

---

## Repository contents

| Document | Description |
|---|---|
| [System Overview](architecture/system-overview.md) | Component diagram and architecture decisions |
| [API Design](architecture/api-design.md) | Endpoints, request models, job lifecycle |
| [Data Model](architecture/data-model.md) | Job states, retention, schema evolution |
| [Video Generation Workflow](workflows/video-generation.md) | End-to-end pipeline walkthrough |
| [Job Lifecycle](workflows/job-lifecycle.md) | State transitions from queued to expired |
| [Tech Stack Rationale](tech-stack.md) | Why each technology was chosen |
| [Roadmap](roadmap.md) | What is being built next |

---

## What this repository is

Public architecture and design documentation. The production codebase is private.

This repository exists to demonstrate engineering depth and system design decisions — not to expose implementation details or IP.
