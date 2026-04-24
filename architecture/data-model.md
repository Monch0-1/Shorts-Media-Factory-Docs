# Data Model

## Job Record

The central entity. Every video generation request creates one job record that persists forever — even after the video file is deleted.

```
JobRecord
├── id              uuid (PK)         — generated at submission
├── topic           str               — user-provided generation topic
├── theme           str               — theme used at submission time
├── context_story   str (nullable)    — optional background context
├── options_json    str (nullable)    — JSON snapshot of VideoOptions at submission
├── user_id         str (nullable)    — null for legacy rows; always set on new jobs
├── status          enum              — see lifecycle below
├── credits_cost    int               — credits charged for this job (0 on failure)
├── output_path     str (nullable)    — local path; cleared on expiry
├── error_message   str (nullable)    — capped at 1000 chars
├── created_at      datetime (UTC)
├── updated_at      datetime (UTC)
└── expires_at      datetime (UTC)    — created_at + VIDEO_RETENTION_HOURS
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
          ┌────────────────┼────────────────┐
          │ success        │ policy refusal  │ failure
          ▼                ▼                 ▼
     ┌──────────┐   ┌──────────┐      ┌────────┐
     │completed │   │ rejected │      │ failed │
     └─────┬────┘   └──────────┘      └────────┘
           │ after VIDEO_RETENTION_HOURS
           ▼
     ┌─────────┐
     │ expired │  ← file deleted, record stays
     └─────────┘
```

**Status semantics:**

| Status | File exists | Record exists | Download available | Credits charged |
|---|---|---|---|---|
| `queued` | No | Yes | No | No |
| `processing` | No | Yes | No | No |
| `completed` | Yes | Yes | Yes | Yes (full cost) |
| `rejected` | No | Yes | No | Yes (base rate only) |
| `failed` | No | Yes | No | No |
| `expired` | No | Yes | No | No (charged at completion) |

**`rejected`** — topic refused by the content policy system. All configured LLM providers declined the prompt. A base rate (duration tier only, no SFX/model addons) is charged to discourage spam. The `error` field in the status response contains the reason shown to the user.

---

## Video Retention

Default retention window: **72 hours** (`VIDEO_RETENTION_HOURS` env var — tunable without code changes).

A cleanup job runs hourly via APScheduler:
1. Finds all `completed` jobs where `expires_at <= now()`
2. Deletes the video file from disk (FileNotFoundError silenced — already cleaned)
3. Clears `output_path`, transitions status to `expired`

Job records are never deleted — dashboard metadata lives forever.

---

## Users

One record per authenticated user. Created lazily on first API request (or via Clerk webhook on account creation — whichever comes first).

```
UserRecord
├── id          uuid (PK)
├── email       str (unique)
├── clerk_id    str (unique)     — Clerk's external user ID
├── plan        enum             — free | pro | enterprise
├── is_active   bool             — default true
└── created_at  datetime (UTC)
```

---

## Credits Ledger

Append-only. Rows are never updated or deleted. Balance is always computed as `SUM(delta)` — no mutable field that can drift.

```
CreditLedgerEntry
├── id          uuid (PK)
├── user_id     FK → UserRecord.id
├── delta       int              — positive for credits in, negative for usage
├── reason      str              — "beta_seed" | "video_generation" | "content_policy_rejection" | "stripe_purchase"
├── job_id      str (nullable)   — FK → JobRecord.id; set for generation/rejection entries
└── created_at  datetime (UTC)
```

**Balance query:** `SELECT SUM(delta) FROM credits_ledger WHERE user_id = ?`

Beta users are seeded with **100,000 credits** on account creation. No Stripe integration yet — purchases are manual ledger entries during beta.

---

## Storage Separation

Jobs and users live in separate databases with no cross-database foreign keys. This reflects the microservices target architecture — the user service can be extracted independently without schema migrations.

| Database | Contents | Dev default |
|---|---|---|
| Jobs DB | `JobRecord` | PostgreSQL (`jobs` service in Docker) |
| Users DB | `UserRecord`, `CreditLedgerEntry` | SQLite (`users.db`) |