# Roadmap

## Current state (as of 2026-11-04 — Hardening_Lite merged)

- Clerk JWT authentication (RS256 / JWKS, `email_verified` enforced, lazy user creation)
- Credit ledger — append-only `credits_ledger`; credits are charged upfront, atomically with the balance check, and reconciled afterwards (full refund on failure); concurrent requests are serialized per user in PostgreSQL; 100k beta seed
- Per-plan LLM tier ceiling (free → haiku, pro → sonnet, enterprise → opus)
- Content-policy billing — `rejected` status, base-cost-only charge, bypasses fallback chain
- Per-user rolling rate limit (free: 10 jobs/hr, pro: 30, enterprise: 100) + `429 Retry-After`
- Route ownership guards (`status`, `download`, `jobs`) — `smf_system` god-view retained
- Clerk webhook (`user.created`, Svix HMAC, idempotent seeding)
- APScheduler hourly cleanup — `completed` → `expired`, file deleted, record kept permanently
- TTS provider abstraction — `ITTSProvider` ABC, `FallbackTTSProvider`, `build_tts_chain()` factory
- ElevenLabs → OpenAI TTS fallback chain (PROD default), Kokoro GPU-local tier (LOCAL mode)
- `APP_MODE` routing: LOCAL (Kokoro only) / PROD (ElevenLabs → OpenAI) / DEBUG (Kokoro prepended)
- 1,100+ automated tests (opt-in suites for GPU models, live LLM calls and Postgres-specific behaviour are skipped by default)
- Docker deployment with PostgreSQL for jobs and users (SQLite only for local dev and tests)
- Character overlays (Nina / Tina / Narrator), mood-driven switching, frame overlays and contextual pop-in effects (`SMF_VFX_Creativity`)
- Duration backstop — a script that runs long is trimmed so the rendered video stays within tolerance of the requested length
- **Campaign format** (`Marketing_Workflow`): sectioned, bring-your-own-media videos — per-section context and uploaded images/videos, narration kept faithful to what the client supplied, section media as the background, cost reconciled to real usage; gated to the Pro and Enterprise plans
- Narration is guaranteed plain text before it reaches text-to-speech, and subtitles are fitted to the frame by measurement (`Hardening_Lite`)
- Per-user upload quota with automatic expiry of uploaded media; campaign jobs share the per-user hourly rate limit
- Full dependency vulnerability audit and upgrade pass (image-decoding and HTTP stack), run before each merge

---

## Completed Sprints

### Auth Sprint — SMF_auth_and_users (merged 2026-04-22)

Multi-tenant SaaS infrastructure. Full auth flow live-tested end-to-end with Docker + Clerk JWT.

- `UserRecord` + `CreditLedgerEntry` — isolated `users.db`; no cross-service DB foreign keys
- `verify_jwt` dependency replaces static `X-API-Key` entirely
- `BETA_ALLOWED_EMAILS` allowlist — non-allowlisted emails blocked at 403 before any record is created
- `JobRecord` gains `theme`, `context_story`, `options_json`, `credits_cost`, `expires_at`
- Credit cost formula: duration-based (500–2000) + optional SFX/VFX/model addons
- `GET /v1/user/balance/{user_id}` — JWT ownership guard
- LLM tier selection: `llm_tier` request field, plan ceiling enforced at route level
- PROD startup validation — hard fail on missing `CLERK_SECRET_KEY` / `CLERK_WEBHOOK_SECRET`
- ngrok sidecar in `docker-compose.override.yml` — public tunnel on `docker compose up`

### Fast_Integration (merged 2026-04-12)

- FastAPI with full async job pipeline and background task runner
- 5 production themes (default, horror, reddit, inspirational, story_formatter)
- Docker deployment with PostgreSQL
- Local SFX library + ElevenLabs generation fallback
- DEBUG / PROD operating modes

### Local_Provider_Layer (merged 2026-04-25)

- `ITTSProvider` ABC — single contract for all TTS providers going forward
- `FallbackTTSProvider` — ordered provider chain, catches `TTSUnavailableError`, advances on transient errors
- `ElevenLabsTTSProvider`, `KokoroTTSProvider`, `OpenAITTSProvider` — three providers behind the interface
- Factory modes: LOCAL (Kokoro), PROD (ElevenLabs → OpenAI), DEBUG (Kokoro prepended)
- `TTS_PROVIDERS` env var controls external tier order — new cloud providers require only a factory registry entry

---

### Script_Generator_Refactor (merged 2026-04-30)

- One mode-aware script generator driven by theme configuration replaces per-style wrapper modules; new script styles no longer need new wiring

### Pipeline_Consolidation (merged 2026-05-07)

- Single pipeline service shared by the API paths; `GET /v1/user/me`; structured-output hardening for LLM providers

### SMF_VFX_Creativity (merged 2026-09-11)

- Character overlays with mood-driven switching, frame overlays and contextual pop-in effects, gated by plan tier

### Release_Hardening (merged 2026-10-18)

- Credit double-spend closed: upfront atomic charge + reconciliation, serialized per user in PostgreSQL; verified against the live Docker stack
- Duration backstop for scripts that overshoot the requested length
- Users database wired to PostgreSQL in deployments, with a startup guard that refuses to boot production on SQLite
- Output filename length cap; SFX database connectivity fix inside Docker

### Marketing_Workflow (merged 2026-11-01)

- Campaign format: sectioned bring-your-own-media videos with an upload endpoint, narration validated against the client's own context, per-section media backgrounds, and cost reconciliation (refund or capped top-up) after delivery
- Abuse limits: shared hourly rate limit, per-user upload quota, hourly cleanup of expired uploads
- Security pass: web-framework and image-library upgrades from a full dependency audit
- Live-tested end to end against Docker with a real Clerk login

### Hardening_Lite (merged 2026-11-04)

- Output quality: stray markup can no longer be read aloud by text-to-speech; subtitles always fit inside the frame (wrap, then shrink, then break as a last resort)
- Money-path safety: refunds are issued before a failed job's final status is written, so a status-write failure can no longer swallow a refund; unknown campaign media references are rejected at submit time, before any charge
- Database credentials are always redacted in logs
- Dead-code and unused-import cleanup; the test suite no longer leaves renders behind
- Live-tested end to end against Docker with a real Clerk login

---

## Next: Voice & SFX Improvements

**Investigation first:** a best-in-class **local** voice tier (cloned character voices at near-ElevenLabs quality) and **local sound-effect generation**, so the SFX layer no longer depends on a cloud provider or on library size. Licensing and GPU-in-Docker are gating questions.

---

## Next: Animated Backgrounds (investigation + proof of concept)

Replace stock background footage with rendered, context-driven animations (motion graphics, images and GIFs timed to the narration). Text-to-video generation is explicitly out of scope — cost and content policy. Built on a long-lived integration branch and merged only once the result is clearly better.

---

## Also planned: Security & data-integrity hardening

A dedicated hardening branch for security and data-integrity debt (credential rotation, auth-library migration, schema-migration tooling, and a few transactional-safety improvements around credit charging). Sequencing relative to the sprints above is not yet fixed.

---

## Medium-term: Cloud Migration

- GCP Cloud Run or AWS ECS for stateless app containers
- RDS PostgreSQL for managed database
- S3 for video file storage (download endpoint already returns URL shape compatible with pre-signed URLs — no client changes required)
- Redis for rate limiting sliding window (currently in-process counter) and Celery task queue

---

## Longer-term

| Feature | Notes |
|---|---|
| Video History | LLM script JSON + credits used stored per job; surfaced in `GET /v1/video/jobs`. Designed 2026-04-24 — deferred to own sprint. |
| User dashboard | Job history, status, download links, credit balance. Depends on Video History sprint. |
| Email notifications | Video ready + expiry reminder |
| Stripe integration | Credit purchases via webhook → credits_ledger |
| Subtitle karaoke | Character-level timing from ElevenLabs alignment API. Breaking change (TTS signature). Implement after VFX. |
| SFX 2.0 | Per-theme mode tuning, overuse thresholds. VFX has shipped; scope to be re-set alongside the Voice & SFX investigation. |
| Fish-Speech TTS | High-fidelity local TTS with voice cloning. No stable PyPI release yet. Evaluate when available. |
| MCP server | Read-only PostgreSQL access for Claude Code (`query_assets`, `query_tags`, `describe_table`). After auth sprint. |

---

## What will not be built

- **Video hosting** — this is not a platform. Files are delivered within a retention window and deleted.
- **Lip sync** — out of scope for the current character overlay feature.
- **Real-time generation** — the async polling model is intentional and sufficient.
