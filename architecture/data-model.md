# Data Model

## Job Record

The central entity. Every video generation request creates one job record that persists forever — even after the video file is deleted.

```
JobRecord
├── id              uuid (PK)         — generated at submission
├── topic           str               — user-provided generation topic
├── user_id         str (nullable)    — null until auth sprint lands
├── status          enum              — see lifecycle below
├── output_path     str (nullable)    — local path; cleared on expiry
├── error_message   str (nullable)    — capped at 1000 chars
├── created_at      datetime (UTC)
└── updated_at      datetime (UTC)
```

**Planned additions (auth sprint):**

```
├── theme           str               — theme used at submission time
├── context_story   str               — context provided by user
├── options_json    str               — JSON snapshot of options at submission
└── expires_at      datetime          — created_at + VIDEO_RETENTION_HOURS
```

---

## Job Status Lifecycle

```
                    ┌─────────┐
   POST /generate   │ queued  │
   ──────────────►  └────┬────┘
                         │ background worker picks up
                         ▼
                    ┌─────────────┐
                    │ processing  │
                    └──────┬──────┘
               ┌───────────┴───────────┐
               │ success               │ failure
               ▼                       ▼
          ┌──────────┐           ┌────────┐
          │completed │           │ failed │
          └─────┬────┘           └────────┘
                │ after VIDEO_RETENTION_HOURS
                ▼
          ┌─────────┐
          │ expired │  ← file deleted, record stays
          └─────────┘
```

**Status semantics:**

| Status | File exists | Record exists | Download available |
|---|---|---|---|
| `queued` | No | Yes | No |
| `processing` | No | Yes | No |
| `completed` | Yes | Yes | Yes |
| `failed` | No | Yes | No |
| `expired` | No | Yes | No |

---

## Video Retention

Default retention window: **72 hours** (`VIDEO_RETENTION_HOURS` env var — tunable without code changes).

Rationale: 24h is too tight for users in different timezones or with busy schedules. 72h balances storage cost against user convenience.

A cleanup job (roadmap) runs hourly:
1. Finds all `completed` jobs where `expires_at < now()`
2. Deletes the video file from disk (or S3)
3. Transitions status to `expired`
4. The job record is never deleted — dashboard metadata lives forever

---

## Future schema: Users and Credits

Full user and credit system is designed and ready to implement. Building it before beta so accounting logic is validated before money touches it.

```
users
├── id          uuid PK
├── email       str (unique)
├── clerk_id    str (unique)     ← external auth provider
├── plan        enum (free | pro | enterprise)
└── created_at  datetime

credits_ledger                   ← append-only, never update rows
├── id          uuid PK
├── user_id     FK → users.id
├── delta       int              ← +N purchase, -1 per video generated
├── reason      str              ← "stripe_purchase" | "video_generation"
└── created_at  datetime
```

Credit balance is computed as `SELECT SUM(delta) FROM credits_ledger WHERE user_id = ?` — no mutable balance field that can drift out of sync.
