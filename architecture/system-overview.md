# System Overview

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        Client                           │
│              (Swagger UI / Frontend / curl)             │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS + Bearer JWT (Clerk)
                         ▼
┌─────────────────────────────────────────────────────────┐
│                     FastAPI Layer                        │
│                                                         │
│  POST /v1/video/generate   → balance check, enqueue     │
│  GET  /v1/video/status     → poll job state             │
│  GET  /v1/video/download   → get download URL           │
│  GET  /v1/video/jobs       → paginated job history      │
│  GET  /v1/user/balance     → credit balance             │
│  POST /v1/webhooks/clerk   → user lifecycle events      │
│  GET  /v1/health           → liveness check             │
└────────┬───────────────────────────┬────────────────────┘
         │                           │
         ▼                           ▼
┌─────────────────┐      ┌───────────────────────────────┐
│   PostgreSQL    │      │       Background Worker        │
│   (Jobs DB)     │      │                               │
│  JobRecord      │      │  queued → processing →         │
│                 │◄─────│  completed | rejected | failed │
└─────────────────┘      │                               │
                         │  Runs the video pipeline       │
┌─────────────────┐      │  outside the request context  │
│   SQLite        │      └───────────┬───────────────────┘
│   (Users DB)    │                  │
│  UserRecord     │                  ▼
│  CreditLedger   │◄────  ┌──────────────────────────────┐
└─────────────────┘       │       Video Pipeline          │
                          │                              │
┌─────────────────┐       │  1. Script Generation        │
│   APScheduler   │       │     └─ LLM Provider Chain    │
│                 │       │        (Gemini → Claude)     │
│  Hourly cleanup │       │  2. Audio Synthesis          │
│  completed →    │       │     └─ TTS Provider Chain    │
│  expired        │       │        (ElevenLabs → OpenAI) │
└─────────────────┘       │  3. SFX Selection & Timing   │
                          │     └─ Local library + API   │
                          │  4. Video Assembly           │
                          │     └─ MoviePy + FFmpeg      │
                          │  5. Subtitle Rendering       │
                          │     └─ PIL (pillow)          │
                          └──────────────────────────────┘
```

---

## Key Design Decisions

### Clerk JWT authentication
Auth is isolated to a single dependency (`verify_jwt`). Routes are untouched by auth changes. In PROD mode, tokens are validated against Clerk's JWKS endpoint with a 5-minute in-process cache. In DEBUG mode, a fixed system user is returned unconditionally — no tokens required. See [Auth Architecture](auth.md).

### FastAPI as a thin HTTP wrapper
The API layer does not contain business logic. It enqueues jobs, tracks state, and serves results. The video pipeline is entirely decoupled — it can be moved to a Celery worker or a separate service without changing the API contract.

### Background tasks over synchronous processing
Video generation takes 30–120 seconds depending on script length and API latency. Synchronous endpoints would time out. The API returns a `job_id` immediately; clients poll `GET /v1/video/status/{job_id}` until the job completes.

### Append-only credit ledger
There is no mutable credit balance field. Balance is always computed as `SUM(delta)` from an append-only ledger table. This eliminates balance drift from concurrent updates and provides a full audit trail. A `SELECT FOR UPDATE` guard prevents double-spend on concurrent submissions.

### Storage separation (microservices target)
Jobs and users live in separate databases with no cross-database foreign keys. This is a deliberate architectural constraint — the user service can be extracted as a standalone microservice without schema migrations. See [Data Model](data-model.md).

### LLM and TTS provider abstraction
Both script generation and audio synthesis go through provider interfaces with configurable fallback chains. The pipeline does not know which model or service is underneath. Adding or reordering providers is a single environment variable change. See [LLM Providers](llm-providers.md).

### Dual operating modes
`APP_MODE=DEBUG` swaps real AI APIs for local mock services — pre-recorded audio, pre-written scripts, no auth. This makes the full pipeline testable without API costs or network dependency.

---

## Infrastructure

The project ships as two Docker images orchestrated with `docker-compose`:

| Service | Image | Purpose |
|---|---|---|
| `app` | `python:3.12-slim` + FFmpeg | FastAPI + video pipeline |
| `db` | `postgres:16` | Job state persistence |
| `ngrok` (override) | `ngrok/ngrok` | Public tunnel for Clerk webhooks (dev only) |

Output videos are stored on a bind-mounted volume. The container holds no user data beyond the retention window.

### Deployment targets
- **Local dev:** `docker compose up` — PostgreSQL for jobs, SQLite for users, local file storage
- **Cloud (roadmap):** GCP Cloud Run or AWS ECS + RDS PostgreSQL + S3 for video storage