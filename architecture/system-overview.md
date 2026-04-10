# System Overview

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        Client                           │
│              (Swagger UI / Frontend / curl)             │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS + X-API-Key
                         ▼
┌─────────────────────────────────────────────────────────┐
│                     FastAPI Layer                        │
│                                                         │
│  POST /v1/video/generate   → enqueue job, return job_id │
│  GET  /v1/video/status     → poll job state             │
│  GET  /v1/video/download   → get download URL           │
│  GET  /v1/video/jobs       → paginated job history      │
│  GET  /v1/health           → liveness check             │
└────────────┬────────────────────────┬───────────────────┘
             │                        │
             ▼                        ▼
┌────────────────────┐   ┌────────────────────────────────┐
│   PostgreSQL       │   │        Background Worker        │
│                    │   │                                 │
│  JobRecord         │   │  queued → processing →          │
│  - id (uuid)       │   │  completed | failed             │
│  - topic           │◄──│                                 │
│  - status          │   │  Runs the video pipeline        │
│  - output_path     │   │  outside the request context   │
│  - created_at      │   └────────────┬───────────────────┘
│  - updated_at      │                │
└────────────────────┘                ▼
                          ┌───────────────────────────────┐
                          │       Video Pipeline           │
                          │                               │
                          │  1. Script Generation         │
                          │     └─ LLM Provider Chain     │
                          │        (Gemini → Claude)      │
                          │  2. Audio Synthesis           │
                          │     └─ ElevenLabs TTS         │
                          │  3. SFX Selection & Timing    │
                          │     └─ Local library + API    │
                          │  4. Video Assembly            │
                          │     └─ MoviePy + FFmpeg       │
                          │  5. Subtitle Rendering        │
                          │     └─ PIL (pillow)           │
                          └───────────────────────────────┘
```

---

## Key Design Decisions

### FastAPI as a thin HTTP wrapper
The API layer does not contain business logic. It enqueues jobs, tracks state, and serves results. The video pipeline is entirely decoupled — it can be moved to a Celery worker or a separate service without changing the API contract.

### Background tasks over synchronous processing
Video generation takes 30–120 seconds depending on script length and API latency. Synchronous endpoints would time out. The API returns a `job_id` immediately; clients poll `GET /v1/video/status/{job_id}` until the job completes.

### PostgreSQL for job state
In-process job dictionaries were the initial prototype. PostgreSQL was adopted early to:
- Survive container restarts without losing job state
- Support multi-worker deployments in the future
- Enable per-user job history once auth lands

### Deterministic output paths
Output files are named `{sanitized_topic}_{job_id}.mp4`. The `job_id` suffix eliminates race conditions when two users generate the same topic simultaneously.

### LLM Provider abstraction
Script generation is decoupled from any specific AI provider via a provider interface. The pipeline does not know which model it is calling — it calls the interface. A configurable fallback chain attempts providers in order, automatically recovering from transient quota or overload errors (429/503) without surfacing failures to the user. The default order is cost-optimised (cheaper providers first); the order is a single environment variable change.

See [LLM Providers](llm-providers.md) for the full design.

### Dual operating modes
`APP_MODE=DEBUG` swaps real AI APIs for local mock services — pre-recorded audio, pre-written scripts. This makes the full pipeline testable without incurring API costs or network dependency.

---

## Infrastructure

The project ships as two Docker images orchestrated with `docker-compose`:

| Service | Image | Purpose |
|---|---|---|
| `app` | `python:3.12-slim` + FFmpeg | FastAPI + video pipeline |
| `db` | `postgres:16` | Job state persistence |

Output videos are stored on a bind-mounted volume. The container itself holds no user data beyond the retention window.

### Deployment targets
- **Local dev:** `docker compose up` — SQLite or PostgreSQL, local file storage
- **Cloud (roadmap):** GCP Cloud Run or AWS ECS + RDS PostgreSQL + S3 for video storage
