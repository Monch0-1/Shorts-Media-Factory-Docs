# Authentication

## Overview

All API endpoints require a valid **Clerk JWT Bearer token** passed in the `Authorization` header.

```
Authorization: Bearer <clerk-session-token>
```

Tokens are short-lived (60 seconds). Clients must refresh before each request for long-polling workflows. The QA login bridge at `GET /qa-login` provides a browser-based Clerk sign-in flow to obtain tokens for testing.

---

## Operating Modes

The auth layer has two modes controlled by `APP_MODE`:

| Mode | Behaviour |
|---|---|
| `PROD` | RS256 JWT validation via Clerk JWKS endpoint. Real users, real credits. |
| `DEBUG` | Auth bypassed entirely. Fixed `smf_system` user returned on every request. |

PROD mode validates tokens against Clerk's JWKS endpoint (5-minute in-process cache). DEBUG mode is safe for local development and CI — no Clerk account or token required.

---

## User Resolution Flow (PROD)

```
Request arrives with Bearer token
          │
          ▼
    Decode + validate JWT (RS256, Clerk JWKS)
          │
          ├── invalid / expired ──► 401
          │
          ▼
    email_verified check
          │
          ├── not verified ──► 403 (configurable)
          │
          ▼
    Beta allowlist check (if BETA_ALLOWED_EMAILS configured)
          │
          ├── email not on list ──► 403
          │
          ▼
    Lazy user creation
          │
          ├── first request ──► create UserRecord + seed 100k beta credits
          │
          └── returning user ──► load existing UserRecord
                    │
                    ▼
             UserContext passed to route handler
```

---

## UserContext

Every authenticated route handler receives a `UserContext` object:

| Field | Description |
|---|---|
| `user_id` | Internal UUID — used for all ownership checks |
| `clerk_id` | Clerk's external user ID (`sub` claim from JWT) |
| `email` | Resolved from JWT claim or Clerk backend API |
| `plan` | `free` \| `pro` \| `enterprise` — read from JWT `public_metadata` |

---

## Beta Allowlist

When `BETA_ALLOWED_EMAILS` is set (comma-separated), only listed emails can access the API. Non-allowlisted requests receive `403` before any user record or credits are created.

To add a tester: append their email to `BETA_ALLOWED_EMAILS` in `.env` and restart the container (no rebuild needed).

---

## Clerk Webhook

`POST /v1/webhooks/clerk` handles Clerk lifecycle events. Currently only `user.created` is processed:

1. Svix signature verified (skipped in DEBUG mode)
2. User record created if not already present (idempotent — duplicate events are safe)
3. 100,000 beta credits seeded via `credits_ledger`

Unknown event types return `200` and are ignored.

---

## System User

`smf_system` is a reserved internal user ID. Routes that check ownership skip the ownership guard for `smf_system`, giving it god-view access across all jobs and users. Used for support tooling and internal pipeline operations.

In DEBUG mode, all requests are resolved as `smf_system` automatically.

---

## Plan Tiers

| Plan | Rate limit (jobs/hour) | Max LLM tier |
|---|---|---|
| `free` | 10 | Haiku |
| `pro` | 30 | Sonnet |
| `enterprise` | 100 | Opus |

Plan is read from the Clerk JWT `public_metadata.plan` field. Unknown values default to `free`.