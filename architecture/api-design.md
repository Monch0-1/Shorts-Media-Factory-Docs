# API Design

## Authentication

All endpoints require an `X-API-Key` header. The current implementation uses a static key from environment configuration.

**Roadmap:** Clerk/Supabase JWT replaces the static key. The auth dependency is isolated to a single file — routes are untouched by this swap.

```
X-API-Key: <your-api-key>
```

---

## Endpoints

### POST /v1/video/generate
Submit a video generation job. Returns immediately with a `job_id`.

**Request**
```json
{
  "topic": "Why procrastination is actually rational",
  "theme": "default",
  "time_limit": 60,
  "context_story": "Optional background context for the script",
  "options": {
    "include_sfx": true,
    "include_subtitles": true
  }
}
```

**Response** `202 Accepted`
```json
{
  "job_id": "3f7a1c2e-...",
  "status": "queued"
}
```

**Available themes:** `default`, `horror`, `reddit`, `inspirational`, `story_formatter`

---

### GET /v1/video/status/{job_id}
Poll for job state. Call this until `status` is `completed` or `failed`.

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

**Response — failed**
```json
{
  "job_id": "3f7a1c2e-...",
  "status": "failed",
  "created_at": "2026-04-08T14:23:00Z",
  "error": "Pipeline error description"
}
```

**Job status values:** `queued` · `processing` · `completed` · `failed` · `expired`

---

### GET /v1/video/download/{job_id}
Returns a download URL for a completed video. The URL shape is designed for S3 pre-signed URL compatibility — clients are not broken when the backend switches from local file serving to cloud storage.

**Response** `200 OK`
```json
{
  "url": "http://localhost:8000/v1/video/file/3f7a1c2e-...",
  "expires_in": 3600
}
```

Returns `404` if the job is not completed, does not exist, or has expired.

---

### GET /v1/video/jobs
Paginated job history. Returns newest jobs first.

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

### GET /v1/health
Liveness check. Returns `200` when the service is running.

```json
{ "status": "ok" }
```

---

## Error handling

All error responses follow the same shape:

```json
{
  "detail": "Human-readable error description"
}
```

| Status | Meaning |
|---|---|
| `400` | Invalid request (validation error) |
| `401` | Missing or invalid API key |
| `404` | Job not found, not completed, or expired |
| `422` | Request body schema violation |
| `500` | Internal server error |

Path traversal attempts on the download endpoint return `404` — no information leakage about whether the path exists.
