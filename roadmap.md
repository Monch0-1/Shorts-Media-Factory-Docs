# Roadmap

## Current state (v0.2.1 — Local_Provider_Layer)

- Clerk JWT authentication (RS256 / JWKS, `email_verified` enforced, lazy user creation)
- Credit ledger — append-only `credits_ledger`, `SELECT FOR UPDATE` double-spend guard, 100k beta seed
- Per-plan LLM tier ceiling (free → haiku, pro → sonnet, enterprise → opus)
- Content-policy billing — `rejected` status, base-cost-only charge, bypasses fallback chain
- Per-user rolling rate limit (free: 10 jobs/hr, pro: 30, enterprise: 100) + `429 Retry-After`
- Route ownership guards (`status`, `download`, `jobs`) — `smf_system` god-view retained
- Clerk webhook (`user.created`, Svix HMAC, idempotent seeding)
- APScheduler hourly cleanup — `completed` → `expired`, file deleted, record kept permanently
- TTS provider abstraction — `ITTSProvider` ABC, `FallbackTTSProvider`, `build_tts_chain()` factory
- ElevenLabs → OpenAI TTS fallback chain (PROD default), Kokoro GPU-local tier (LOCAL mode)
- `APP_MODE` routing: LOCAL (Kokoro only) / PROD (ElevenLabs → OpenAI) / DEBUG (Kokoro prepended)
- 459 automated tests (10 skipped — GPU `local_models` marker, CI-safe)
- Docker deployment with PostgreSQL (jobs) + SQLite (users, dev)

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

## Next: VFX Character Overlay

PNG character sprites (Nina, Tina, Narrator) composited into the final video. Mood-driven switching — 6 moods derived from segment SFX category, no additional AI calls.

**Status:** BRD drafted (`vfx_brd_draft.md`). Initial test assets received (2026-04-23). Full asset set needed: 18 total (6 moods × 3 characters). Branch scope is wide — full BRD required before implementation start.

**Dependency note:** SFX 2.0 character-aware sound mixing may depend on VFX landing first. SFX 2.0 scope will be limited or sequenced after VFX.

---

## Near-term: Script Generator Consolidation

`create_script_monologue.py` and `create_script_debate.py` are near-identical wrappers — every new script style requires a new file, new wiring, new factory changes.

**Goal (`Script_Generator_Refactor` branch):**
- Single mode-aware `ScriptGeneratorService` driven by `ThemeManager` / `ThemeConfig`
- `RealScriptService` becomes a thin caller with no dispatch logic
- Centralized voice registry (Nina/Tina/Narrator + theme → provider voice ID) — fixes the Narrator-for-all voice mapping regression (Claude LLM + default theme + top-5)
- `refine_base_prompt.py` redesigned and renamed as part of this work

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
| SFX 2.0 | Per-theme mode tuning, weighted trait scoring, overuse thresholds. Sequenced after VFX. |
| Fish-Speech TTS | High-fidelity local TTS with voice cloning. No stable PyPI release yet. Evaluate when available. |
| MCP server | Read-only PostgreSQL access for Claude Code (`query_assets`, `query_tags`, `describe_table`). After auth sprint. |

---

## What will not be built

- **Video hosting** — this is not a platform. Files are delivered within a retention window and deleted.
- **Lip sync** — out of scope for the current character overlay feature.
- **Real-time generation** — the async polling model is intentional and sufficient.
