# Technology Stack

## Why each technology was chosen

---

### AI Language Models (Gemini + Claude)
Script generation requires a model that can follow complex structured output schemas while maintaining creative quality and narrative consistency. The pipeline supports multiple providers behind a common interface — Gemini and Claude are both first-class citizens.

**Gemini** is the default primary provider. Its native JSON mode (`response_mime_type=application/json`) returns structured output without prompt-engineering workarounds, and its context window handles long-form multi-turn scripts reliably. The free tier makes it cost-effective for development and low-volume production.

**Claude (Anthropic)** serves as the fallback provider. When Gemini hits quota limits or experiences overload, the pipeline automatically retries via Claude without surfacing the error to the user. Claude is also available as the primary provider for higher-fidelity use cases (configurable per plan in a future release).

The provider chain is a single environment variable change — no code required to swap or reorder providers. See [LLM Providers](architecture/llm-providers.md).

---

### ElevenLabs
Two reasons: voice quality and sound effects. ElevenLabs produces the most natural-sounding AI voices currently available — a critical requirement when the output is a video people will watch on TikTok. It also provides an SFX generation API for cases where the local sound effects library does not have a match, giving the pipeline a reliable fallback with no manual intervention.

---

### FastAPI (Python 3.12)
The API layer needs to be async-native — video generation is a long-running background task, not a synchronous operation. FastAPI's `BackgroundTasks`, native Pydantic validation, and automatic OpenAPI documentation made it the clear choice over Flask or Django REST Framework. The Pydantic integration also enforces strict input validation at the boundary without boilerplate.

---

### MoviePy + FFmpeg
MoviePy provides a Pythonic API for video composition without requiring shell scripting around FFmpeg directly. The pipeline composites multiple layers — background video, master audio, SFX events, subtitle ImageClips — and MoviePy handles the timeline and render pipeline cleanly. FFmpeg handles the final encoding.

---

### PostgreSQL (via SQLModel)
Job state needs to survive container restarts and support concurrent workers. An in-process dictionary was the initial prototype; PostgreSQL was adopted before the first Docker deployment to avoid re-migration later. SQLModel provides a unified Pydantic + SQLAlchemy model that works across both SQLite (local dev/test) and PostgreSQL (production) with a single connection string change.

---

### Docker + docker-compose
The pipeline has hard system dependencies: FFmpeg, specific Python versions, Linux font paths. Docker eliminates "works on my machine" issues and provides a reproducible environment for both local development and cloud deployment. `docker-compose` orchestrates the app container and PostgreSQL with a health check dependency so the app never starts before the database is ready.

---

### PIL (Pillow) for subtitles
MoviePy's built-in `TextClip` depends on ImageMagick, which is an additional system dependency with licensing and configuration overhead. PIL renders subtitle frames directly as RGBA PNGs with full control over font, stroke, pill backgrounds, and multi-line layout. The rendered images are composited as `ImageClip` objects — no ImageMagick required.

---

## What was evaluated and not chosen

| Technology | Considered for | Decision |
|---|---|---|
| Celery + Redis | Background job queue | Deferred — `BackgroundTasks` is sufficient for single-server; Celery queues added when horizontal scaling is needed |
| Whisper | Subtitle generation from audio | Deferred — subtitle timing from synthesis timestamps is more precise than transcription |
| OpenAI GPT-4 | Script generation | Not adopted — Gemini native JSON mode and Claude's instruction-following quality covered the use case without adding a third provider dependency |
| S3 | Video file storage | Designed in (download endpoint returns URL shape compatible with S3 pre-signed URLs); swap is one file change |
