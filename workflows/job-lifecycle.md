# Job Lifecycle

Every video generation request maps to a single `JobRecord` that persists indefinitely — even after the video file is gone.

---

## State Transitions

```
POST /v1/video/generate
         │  balance check passes
         ▼
      [queued]  ◄── Created synchronously. job_id returned to client.
         │
         │  Background worker starts
         ▼
    [processing]  ◄── Pipeline running. Client polls status.
         │
    ┌────┴──────────────┐
    │ success           │ content policy refusal   │ failure
    ▼                   ▼                           ▼
[completed]        [rejected]                   [failed]
    │               (base rate charged)
    │  VIDEO_RETENTION_HOURS elapsed
    │  Hourly cleanup job runs
    ▼
 [expired]  ◄── Record kept. File gone. Download returns 404.
```

---

## What each state means for the client

### `queued`
Job accepted, credits reserved. Pipeline has not started yet. Check back shortly.

### `processing`
Pipeline is running. Script is being generated, audio synthesized, video assembled. Typical duration: 30–120 seconds depending on script length and API response times.

### `completed`
Video is ready. `result_url` is present in the status response. Use the download endpoint to get the file URL.

### `rejected`
The topic was refused by the content policy system — all configured LLM providers declined to generate a script for this prompt. A base processing fee is charged. The `error` field contains the user-facing reason. The job record is permanent and appears in history.

### `failed`
Something went wrong during pipeline execution (infrastructure error, unexpected API failure). No credits are charged — the failure is on the platform, not the user. The `error` field contains a description.

### `expired`
The video was completed but the retention window has passed. The file has been deleted by the hourly cleanup job. The job record remains in history with metadata (topic, theme, timestamp) — only the file is gone.

---

## Polling recommendation

```
POST /v1/video/generate  →  job_id

loop:
    GET /v1/video/status/{job_id}
    if status == "completed":
        GET /v1/video/download/{job_id}  →  { url, expires_in }
        break
    if status in ("failed", "rejected"):
        handle error
        break
    wait 5s
```

No webhook is required. In-app notification is implemented by polling — no additional backend infrastructure needed.

---

## Retention and storage

- Default retention: **72 hours** from job completion (`VIDEO_RETENTION_HOURS` env var)
- Cleanup: hourly APScheduler job — finds completed jobs past `expires_at`, deletes file, transitions to `expired`
- Job records are never deleted — full history always available
- Storage is intentionally lightweight — this is not a video hosting platform