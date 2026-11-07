# API Design

## Authentication

All endpoints require a valid **Clerk JWT Bearer token** in the `Authorization` header.

```
Authorization: Bearer <clerk-session-token>
```

Tokens are short-lived (60 seconds). In DEBUG mode (`APP_MODE=DEBUG`), auth is bypassed entirely — no token required. See [Auth Architecture](auth.md) for the full flow.

---

## Endpoints

### POST /v1/video/generate
Submit a video generation job. Returns immediately with a `job_id`. Deducts credits on success.

**Request**
```json
{
  "topic": "Why procrastination is actually rational",
  "theme": "default",
  "context_story": "Optional background context for the script",
  "llm_tier": "auto",
  "options": {
    "duration_seconds": 60,
    "enable_refiner": false,
    "use_script_template": false,
    "content": {
      "include_sfx": true
    },
    "subtitles": {
      "enabled": true,
      "position": "bottom",
      "color": "white",
      "size": "medium"
    }
  }
}
```

**Available themes:** `default`, `horror`, `reddit`, `inspirational`, `story_formatter`

**LLM tier:** `auto` (plan default), `gemini`, `haiku`, `sonnet`, `opus` — plan ceiling enforced (free → haiku max, pro → sonnet max, enterprise → opus max)

**Response** `202 Accepted`
```json
{
  "job_id": "3f7a1c2e-...",
  "status": "queued"
}
```

**Failure responses:**
- `402` — insufficient credits: `{ "detail": "insufficient_credits", "required": 800, "balance": 200 }`
- `403` — LLM tier exceeds plan ceiling
- `429` — rate limit exceeded: `{ "detail": "rate_limit_exceeded", "limit": 10, "retry_after_seconds": 1800 }`

---

### GET /v1/video/status/{job_id}
Poll for job state. Returns 404 for jobs belonging to other users (no existence leakage).

**Response — in progress**
```json
{
  "job_id": "3f7a1c2e-...",
  "status": "processing",
  "created_at": "2026-04-08T14:23:00Z"
}
```

**Response — completed**
```json
{
  "job_id": "3f7a1c2e-...",
  "status": "completed",
  "created_at": "2026-04-08T14:23:00Z",
  "result_url": "/v1/video/download/3f7a1c2e-..."
}
```

**Response — failed or rejected**
```json
{
  "job_id": "3f7a1c2e-...",
  "status": "failed",
  "created_at": "2026-04-08T14:23:00Z",
  "error": "Pipeline error description"
}
```

**Job status values:** `queued` → `processing` → `completed` | `failed` | `rejected` → `expired`

---

### GET /v1/video/download/{job_id}
Returns a download URL for a completed video. URL shape is compatible with S3 pre-signed URLs — clients are not broken when backend switches from local file serving to cloud storage.

**Response** `200 OK`
```json
{
  "url": "http://localhost:8000/v1/video/file/3f7a1c2e-...",
  "expires_in": 3600
}
```

Returns `404` if the job is not completed, does not exist, belongs to another user, or has expired.

---

### GET /v1/video/jobs
Paginated job history. Results are scoped to the authenticated user. Support accounts with system access see all jobs.

**Query parameters:** `limit` (1–100, default 20) · `offset` (default 0)

**Response**
```json
{
  "total": 47,
  "limit": 20,
  "offset": 0,
  "jobs": [
    {
      "job_id": "3f7a1c2e-...",
      "topic": "Why procrastination is actually rational",
      "status": "completed",
      "created_at": "2026-04-08T14:23:00Z"
    }
  ]
}
```

---

### GET /v1/user/balance/{user_id}
Returns the current credit balance for a user. Only the owning user may query their own balance.

**Response** `200 OK`
```json
{
  "user_id": "d82e61fc-...",
  "balance": 99200,
  "plan": "free"
}
```

---

### POST /v1/campaign/media
Upload media for a campaign. Multipart form, field `files` (images and/or videos). Pro and Enterprise plans only.

- Files are classified by **content**, never by filename or extension; client filenames are never stored
- Uploads are scoped to the caller — another user's asset ids are indistinguishable from unknown ones
- All-or-nothing per call: if any file is rejected, nothing is persisted
- Per-user storage quota; uploads expire automatically after the retention window

```json
{ "asset_ids": ["9f1c...", "a03b..."] }
```

Errors: `403` (plan / not enabled), `400 invalid_media` (the offending file index is reported; internal probe details are not), `413` (request too large or `upload_quota_exceeded`).

---

### POST /v1/campaign/generate
Queue a campaign video. Returns `202` with a `job_id`; poll `GET /v1/video/status/{job_id}` as for any other job.

```json
{
  "narration_style": "monologue",
  "theme": "default",
  "llm_tier": "auto",
  "sections": [
    { "context": "What this section is about...", "media": [{ "asset_id": "9f1c..." }] },
    { "context": "A section with no media.", "no_media": true }
  ],
  "options": { "content": { "include_sfx": true }, "vfx": {}, "subtitles": {} }
}
```

- Each section carries the client's own context and an ordered list of uploaded media (or `no_media`)
- Narration must stay faithful to the supplied context — sections that drift are reworked before audio is generated
- Video length is emergent (driven by the narration), not requested; a maximum applies
- Credits are charged upfront, then reconciled to the real cost after delivery (refund, or a top-up capped at the user's balance). Any failure on our side is a full refund
- Media may only reference the caller's own uploads (`asset_id`); server-side file references are rejected with `422 filename_refs_not_allowed`
- Every `asset_id` is checked at submit time, before rate limiting or any charge: an id that is not one of the caller's uploads returns `400 unknown_asset_id` (with the offending `asset_id` and `section` index)

Errors: `400 unknown_asset_id`, `402` insufficient credits, `403` plan / not enabled / tier ceiling, `422` invalid request, `429` rate limit (shared with `/v1/video/generate`).

---

### POST /v1/webhooks/clerk
Clerk lifecycle webhook. Handles `user.created` — idempotent user creation and credit seeding. Svix signature verified in PROD; skipped in DEBUG.

---

### GET /v1/health
Liveness check. Requires a valid token in PROD mode.

```json
{ "status": "ok" }
```

---

## Error Handling

All error responses follow the same shape:

```json
{
  "detail": "Human-readable error description"
}
```

| Status | Meaning |
|---|---|
| `400` | Invalid request (validation error) |
| `401` | Missing or invalid Bearer token |
| `402` | Insufficient credits |
| `403` | Forbidden — plan ceiling exceeded, email not verified, or beta allowlist rejection |
| `404` | Job not found, not completed, expired, or belongs to another user |
| `413` | Upload too large or per-user storage quota exceeded |
| `422` | Request body schema violation |
| `429` | Rate limit exceeded — `Retry-After` header included |
| `500` | Internal server error |

Path traversal attempts on the download endpoint return `404` — no information leakage about whether the path exists.