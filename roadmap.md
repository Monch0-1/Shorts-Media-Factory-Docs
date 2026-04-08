# Roadmap

## Current state (v0.1.0 — Fast_Integration)

- FastAPI with full async job pipeline
- 5 production themes (default, horror, reddit, inspirational, story_formatter)
- Docker deployment with PostgreSQL
- Local SFX library + ElevenLabs generation fallback
- DEBUG / PROD operating modes
- 203 automated tests

---

## Next: Auth Sprint

**Goal:** Multi-tenant SaaS infrastructure. No user-facing product ships before this is in place.

### User accounts
- Clerk / Supabase JWT authentication replacing static API key
- `User` model with plan tiers (free, pro, enterprise)
- Auth dependency is isolated — routes are untouched by the JWT swap

### Credit system
- Append-only `credits_ledger` table — balance computed from `SUM(delta)`, never a mutable field
- Beta users seeded with starting credits — no Stripe yet
- Deduction at job submission, after balance check, before job enqueue
- `SELECT FOR UPDATE` prevents double-spend on concurrent submissions

### Multi-tenancy
- `GET /v1/video/jobs` filtered by authenticated user
- `GET /v1/video/status/{job_id}` ownership check
- `JobRecord` gains `theme`, `context_story`, `options_json`, `expires_at` fields

### Video expiry
- Hourly cleanup job: `completed` → `expired` after retention window
- File deleted on expiry; record and metadata kept permanently

---

## Near-term: VFX Character Overlay

- PNG character sprites with mood-driven switching (neutral, amused, shocked, thinking, tense, emotional)
- Mood derived from the script's SFX category per segment — no additional AI calls
- Composited alongside subtitles at render time — zero impact on audio pipeline

---

## Medium-term: Cloud Migration

- GCP Cloud Run or AWS ECS for stateless app containers
- RDS PostgreSQL for managed database
- S3 for video file storage (download endpoint already returns URL shape compatible with pre-signed URLs — no client changes required)
- Redis for rate limiting and future Celery task queue

---

## Longer-term

| Feature | Notes |
|---|---|
| User dashboard | Job history, status, download links, credit balance |
| Email notifications | Video ready + expiry reminder |
| Stripe integration | Credit purchases via webhook → credits_ledger |
| Rate limiting | Per-user (5 jobs/hour), per-IP, global |
| Subtitle karaoke | Character-level timing from ElevenLabs alignment API |
| Script generator consolidation | Single mode-aware service replacing theme-specific generators |
| SFX weighted trait scoring | Primary vs. secondary trait weights for more precise SFX selection |

---

## What will not be built

- **Video hosting** — this is not a platform. Files are delivered within a retention window and deleted.
- **Lip sync** — out of scope for the current character overlay feature.
- **Real-time generation** — the async polling model is intentional and sufficient.
