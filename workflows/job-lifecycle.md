# Job Lifecycle

Every video generation request maps to a single `JobRecord` that persists indefinitely — even after the video file is gone.

---

## State Transitions

```
POST /v1/video/generate
         │
         ▼
      [queued]  ◄── Created synchronously. job_id returned to client.
         │
         │  Background worker starts
         ▼
    [processing]  ◄── Pipeline running. Client polls status.
         │
    ┌────┴────┐
    │         │
    ▼         ▼
[completed] [failed]
    │
    │  VIDEO_RETENTION_HOURS elapsed (default 72h)
    │  Cleanup job deletes file
    ▼
 [expired]  ◄── Record kept. File gone. Download returns 404.
```

---

## What each state means for the client

### `queued`
Job accepted. Pipeline has not started yet. Check back shortly.

### `processing`
Pipeline is running. Script is being generated, audio synthesized, video assembled. Typical duration: 30–120 seconds depending on script length and API response times.

### `completed`
Video is ready. `result_url` is present in the status response. Use the download endpoint to get the file URL.

### `failed`
Something went wrong during pipeline execution. The `error` field in the status response contains a description. The job record is permanent — failed jobs appear in job history.

### `expired`
The video was completed but the retention window has passed. The file has been deleted. The job record remains in history with metadata (topic, theme, timestamp) — only the file is gone.

---

## Polling recommendation

```
POST /v1/video/generate  →  job_id

loop:
    GET /v1/video/status/{job_id}
    if status == "completed":
        GET /v1/video/download/{job_id}  →  { url, expires_in }
        break
    if status == "failed":
        handle error
        break
    wait 5s
```

No webhook is required. In-app notification is implemented by polling — no additional backend infrastructure needed.

---

## Retention and storage

- Default retention: **72 hours** from job completion
- After expiry: video file deleted, `JobRecord` status set to `expired`
- Job records are never deleted — full history is always available
- Storage is intentionally lightweight — this is not a video hosting platform

The retention window is configurable via `VIDEO_RETENTION_HOURS` without a code change.
