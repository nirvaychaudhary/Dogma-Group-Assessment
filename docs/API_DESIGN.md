# API Design

REST over HTTPS. JSON only, under `/api/v1`. The OpenAPI document is generated from the Pydantic models, so the contract and the code are the same thing. Abbreviations used below are listed in the [glossary](GLOSSARY.md).

---

## 1. Conventions

| Concern | Convention | Reasoning |
|---|---|---|
| Base path | `/api/v1` | URI versioning is visible in logs, trivially routable at the load balancer, and cacheable. Header-based versioning is purer but harder to debug and to route on |
| Resource naming | Plural nouns, kebab-case paths, `snake_case` JSON fields | Consistency with Python on both sides of the wire |
| Identifiers | time-sorted unique identifier (UUIDv7) | Sortable by creation time like an integer, but non-enumerable. Sequential integer IDs let an attacker walk the object space and turn any authorization gap into a full dump |
| Timestamps | Request for Comments (RFC) 3339, UTC, `_at` suffix | Unambiguous |
| Partial updates | `PATCH` with merge semantics | Clients edit one or two fields; `PUT` forces read-modify-write |
| Pagination | Opaque cursor, `limit` capped at 100, default 20 | See [System Design section 4](SYSTEM_DESIGN.md#4-user-accessing-their-tasks) |
| Errors | RFC 9457 `application/problem+json` | A standard, machine-readable error shape |
| Correlation | `X-Correlation-ID` accepted and always echoed | Client-side support-ticket correlation |
| Concurrency | `ETag` on single resources, `If-Match` on mutations | Optimistic locking |
| Idempotency | `Idempotency-Key` on `POST` | Safe retries |
| Compression | Brotli, then gzip, on JSON larger than 1 KB | Smaller answers under load. See below |

### Response envelope

Collections are wrapped so pagination metadata has somewhere to live; single resources are returned bare.

```json
{
  "data": [ { "id": "018f...", "title": "Write the design doc" } ],
  "page": { "next_cursor": "eyJ0IjoiMjAyNi0wOS0yMi4uLiJ9", "has_more": true, "limit": 20 }
}
```

Returning a bare array at the top level forecloses ever adding metadata without a breaking change — a small decision that is expensive to reverse.

### Smaller answers

Two steps, in this order. Compression is wasted on a fat payload.

1. **Write less JSON.** Production responses are compact: no extra whitespace, and null fields are left out. A task list returns the summary fields only. The description is on the detail call. If the client already has the current version, `If-None-Match` returns `304` and an empty body, which is better than any compression.

2. **Compress what is left, at the edge.** The content delivery network compresses the response. The Python process does not. Under a spike, application CPU stays on login checks and database work.

| Rule | Choice |
|---|---|
| Client asks for Brotli (`br`) | Use Brotli. JSON compresses well with it |
| Client asks only for gzip | Use gzip |
| Body under 1 KB, or status `204` / `304` | Do not compress. The header would cost more than it saves |
| Login, refresh, or any body that contains a token | Do not compress. A compressed secret next to data the caller can influence can leak the secret |
| Request body | Not accepted in compressed form. A compressed upload is an easy way to exhaust memory |

The response sets `Content-Encoding` to `br` or `gzip`, and `Vary: Accept-Encoding`.

---

## 2. Authorization Model

Two roles. Deliberately not more: role proliferation without a driving requirement creates a permission model nobody can reason about, and an unreasonable model is an insecure one.

| Role | Capabilities |
|---|---|
| `user` | Full create, read, update, and delete (CRUD) on tasks they own. Read and update their own profile |
| `admin` | Everything a `user` can do, plus read/update/delete any user's tasks, manage user accounts, read the audit log |

Permissions are expressed as `resource:action` strings (`task:create`, `admin:task:update`, `audit:read`) and resolved through the policy engine. Roles map to permission sets in one table, so migrating to fine-grained or custom roles later is a data change rather than a redesign.

**Authorization requirement notation used below**

| Notation | Meaning |
|---|---|
| Public | No token |
| User | Valid access token, active and verified account |
| Owner | Authenticated **and** `task.owner_id == principal.user_id` |
| Admin | Authenticated **and** `role == admin` |

---

## 3. Endpoint Summary

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| `POST` | `/api/v1/auth/register` | Create an account | Public |
| `POST` | `/api/v1/auth/verify-email` | Confirm email ownership | Public (token) |
| `POST` | `/api/v1/auth/login` | Exchange credentials for tokens | Public |
| `POST` | `/api/v1/auth/refresh` | Rotate token pair | Refresh token |
| `POST` | `/api/v1/auth/logout` | Revoke current session | User |
| `POST` | `/api/v1/auth/logout-all` | Revoke every session | User |
| `POST` | `/api/v1/auth/password/forgot` | Begin password reset | Public |
| `POST` | `/api/v1/auth/password/reset` | Complete password reset | Public (token) |
| `POST` | `/api/v1/auth/password/change` | Change password when signed in | User |
| `GET` | `/api/v1/users/me` | Current profile | User |
| `PATCH` | `/api/v1/users/me` | Update own profile | User |
| `DELETE` | `/api/v1/users/me` | Request account deletion | User |
| `POST` | `/api/v1/tasks` | Create a task | User |
| `GET` | `/api/v1/tasks` | List own tasks | User |
| `GET` | `/api/v1/tasks/{task_id}` | Read one task | Owner |
| `PATCH` | `/api/v1/tasks/{task_id}` | Update a task | Owner |
| `DELETE` | `/api/v1/tasks/{task_id}` | Soft-delete a task | Owner |
| `POST` | `/api/v1/tasks/{task_id}/restore` | Restore within retention | Owner |
| `GET` | `/api/v1/admin/users` | List users | Admin |
| `GET` | `/api/v1/admin/users/{user_id}` | Read a user | Admin |
| `PATCH` | `/api/v1/admin/users/{user_id}` | Change role or status | Admin |
| `GET` | `/api/v1/admin/tasks` | List tasks across users | Admin |
| `GET` | `/api/v1/admin/tasks/{task_id}` | Read any task | Admin |
| `PATCH` | `/api/v1/admin/tasks/{task_id}` | Update any task | Admin |
| `DELETE` | `/api/v1/admin/tasks/{task_id}` | Delete any task | Admin |
| `GET` | `/api/v1/admin/audit-logs` | Query the audit trail | Admin |
| `GET` | `/health/live` | Liveness probe | Internal |
| `GET` | `/health/ready` | Readiness probe | Internal |
| `GET` | `/metrics` | Prometheus scrape | Internal |

---

## 4. Error Model

All errors use RFC 9457. A single shape means clients write one error handler.

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Request validation failed",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "/api/v1/tasks",
  "code": "VALIDATION_ERROR",
  "correlation_id": "01J8XQ2K9F3M7N4P",
  "errors": [
    { "field": "title", "code": "TOO_LONG", "message": "Must be at most 200 characters." },
    { "field": "due_at", "code": "IN_PAST", "message": "Must be in the future." }
  ]
}
```

`code` is the stable, machine-readable contract; `title` and `detail` are human-facing and may be reworded without a breaking change. `correlation_id` is present on **every** error, so a user can paste one string into a support ticket and an engineer can retrieve the entire request.

### Status code usage

| Status | Used for | Notably *not* used for |
|---|---|---|
| `400` | Malformed syntax, undecodable body | Business rule failures |
| `401` | Missing, expired or invalid token | Insufficient permissions |
| `403` | Authenticated but structurally forbidden — unverified email, suspended account | Object-level denials, which return `404` |
| `404` | Resource absent **or** not visible to this principal | — |
| `409` | State conflict — concurrent update lost the race, idempotent request in flight | Validation errors |
| `412` | `If-Match` did not match the current `ETag` | — |
| `422` | Semantically invalid: schema violations, illegal state transitions | Syntax errors |
| `429` | Rate limit exceeded; always carries `Retry-After` | — |
| `5xx` | Our fault. Never leaks a stack trace, Structured Query Language (SQL) fragment, or internal hostname | Anything the client caused |

**On `404` versus `403` for object-level denials.** Returning `403` when a user requests a task they do not own confirms that the task exists. Given enumerable access patterns, that turns the endpoint into an oracle for mapping the object space and for confirming relationships between accounts. Returning `404` costs some debuggability, which I recover by logging the denial internally at high severity with the full context.

**Internal errors never echo details.** A `500` returns only a generic message plus the correlation ID. Stack traces, driver messages, and constraint names go to logs, never to a client — they are a free map of the internals for an attacker.

---

## 5. Endpoint Detail

### 5.1 `POST /api/v1/auth/register`

Creates an unverified account and dispatches a verification email.

**Auth:** Public. Rate limited to 5/hour per Internet Protocol address (IP) and 3/day per email domain.

**Request**

```json
{
  "email": "ada@example.com",
  "password": "correct horse battery staple",
  "display_name": "Ada Lovelace"
}
```

| Field | Rules |
|---|---|
| `email` | Required, RFC 5322, ≤ 254 chars, normalised to lowercase, MX record checked asynchronously |
| `password` | Required, 12–128 chars, checked against a breached-password corpus, must not contain the email local part |
| `display_name` | Required, 1–100 chars, control characters stripped |

**Response — `202 Accepted`**

```json
{
  "message": "If the address is available, a verification email has been sent.",
  "email": "ada@example.com"
}
```

| Scenario | Status | Code |
|---|---|---|
| Email already registered | `202` | Identical body — see [System Design section 1](SYSTEM_DESIGN.md#1-user-registration) |
| Password fails policy | `422` | `WEAK_PASSWORD` |
| Password found in breach corpus | `422` | `PASSWORD_COMPROMISED` |
| Rate limited | `429` | `RATE_LIMITED` |
| Disposable email domain | `422` | `EMAIL_DOMAIN_NOT_ALLOWED` |

---

### 5.2 `POST /api/v1/auth/login`

**Auth:** Public. Rate limited to 10/15 min per IP and 5/15 min per account, with exponential backoff.

**Request**

```json
{ "email": "ada@example.com", "password": "correct horse battery staple" }
```

**Response — `200 OK`**

```json
{
  "access_token": "eyJhbGciOiJFZERTQSIsImtpZCI6IjIwMjYtMDktMDEifQ...",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": { "id": "018f...", "email": "ada@example.com", "display_name": "Ada Lovelace", "role": "user" }
}
```

```http
Set-Cookie: refresh_token=<opaque>; HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth; Max-Age=2592000
```

The refresh token is *not* in the JSON body for browser clients — placing it there would require JavaScript to store it, which reintroduces the cross-site scripting (XSS) exposure the `HttpOnly` cookie exists to prevent. Non-browser clients signal with `X-Client-Type: native` and receive it in the body.

| Scenario | Status | Code | Note |
|---|---|---|---|
| Wrong password or unknown email | `401` | `INVALID_CREDENTIALS` | Identical response and timing for both |
| Email not verified | `403` | `EMAIL_NOT_VERIFIED` | Only after the password verifies |
| Account suspended | `403` | `ACCOUNT_SUSPENDED` | Only after the password verifies |
| Too many attempts | `429` | `TOO_MANY_ATTEMPTS` | `Retry-After` in seconds |

---

### 5.3 `POST /api/v1/auth/refresh`

**Auth:** A valid, unrotated refresh token from the cookie or the body.

**Response — `200 OK`:** a new access token and a rotated refresh cookie. The old refresh token is invalid from this moment.

| Scenario | Status | Code | Note |
|---|---|---|---|
| Unknown or expired token | `401` | `INVALID_REFRESH_TOKEN` | |
| **Already-rotated token replayed** | `401` | `TOKEN_REUSE_DETECTED` | Revokes the entire token family and alerts the user |
| User suspended since issue | `401` | `ACCOUNT_INACTIVE` | The refresh boundary is where account state is re-checked |

---

### 5.4 `POST /api/v1/tasks`

**Auth:** User.

**Request**

```http
POST /api/v1/tasks
Authorization: Bearer <access_token>
Idempotency-Key: 8f3c1e2a-...
Content-Type: application/json
```

```json
{
  "title": "Write the architecture assessment",
  "description": "Cover HLD, API, security and data architecture.",
  "status": "todo",
  "priority": "high",
  "due_at": "2026-09-30T17:00:00Z",
  "tags": ["work", "urgent"]
}
```

| Field | Rules |
|---|---|
| `title` | Required, 1–200 chars, trimmed, must not be whitespace only |
| `description` | Optional, ≤ 10,000 chars |
| `status` | Optional, one of `todo` \| `in_progress` \| `done`, defaults to `todo`. Cannot be created as `done` |
| `priority` | Optional, one of `low` \| `medium` \| `high`, defaults to `medium` |
| `due_at` | Optional, RFC 3339, must be in the future |
| `tags` | Optional, ≤ 10 items, each ≤ 30 chars, deduplicated |

**There is no `owner_id` field.** It is derived from the access token. Sending one is rejected by the schema, because the model forbids unknown fields — silently ignoring unexpected input hides client bugs and, on a field like this one, hides attacks.

**Response — `201 Created`**

```http
Location: /api/v1/tasks/018f2c1a-...
ETag: "1"
```

```json
{
  "id": "018f2c1a-...",
  "owner_id": "018f0b3d-...",
  "title": "Write the architecture assessment",
  "description": "Cover HLD, API, security and data architecture.",
  "status": "todo",
  "priority": "high",
  "due_at": "2026-09-30T17:00:00Z",
  "tags": ["work", "urgent"],
  "version": 1,
  "created_at": "2026-09-22T09:14:02Z",
  "updated_at": "2026-09-22T09:14:02Z"
}
```

| Scenario | Status | Code |
|---|---|---|
| Validation failure | `422` | `VALIDATION_ERROR` with a per-field list |
| `owner_id` supplied | `422` | `UNKNOWN_FIELD` |
| Idempotency key replayed, same body | `201` | Original response replayed |
| Idempotency key replayed, different body | `422` | `IDEMPOTENCY_KEY_REUSED` |
| Idempotent request still in flight | `409` | `REQUEST_IN_PROGRESS` |
| Per-user task quota exceeded | `422` | `QUOTA_EXCEEDED` |

---

### 5.5 `GET /api/v1/tasks`

**Auth:** User. Always scoped to the caller.

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `status` | enum, repeatable | all | `?status=todo&status=in_progress` |
| `priority` | enum, repeatable | all | |
| `due_before` / `due_after` | RFC 3339 | — | |
| `q` | string, ≤ 100 chars | — | Substring match on title; parameterised, never interpolated |
| `tags` | string, repeatable | — | AND semantics |
| `sort` | enum | `-created_at` | Whitelisted: `created_at`, `due_at`, `priority`, each with `-` prefix. **Never a raw column name** |
| `cursor` | opaque string | — | From the previous page |
| `limit` | int | 20 | Capped at 100 |
| `include_deleted` | bool | false | Owner-visible within the retention window |

**Response — `200 OK`**

```json
{
  "data": [ { "id": "018f2c1a-...", "title": "Write the architecture assessment", "status": "todo", "priority": "high", "due_at": "2026-09-30T17:00:00Z", "version": 1, "created_at": "2026-09-22T09:14:02Z" } ],
  "page": { "next_cursor": "eyJjIjoiMjAyNi0wOS0yMlQwOToxNDowMloiLCJpIjoiMDE4ZjJjMWEifQ", "has_more": true, "limit": 20 }
}
```

| Scenario | Status | Code |
|---|---|---|
| Malformed or tampered cursor | `400` | `INVALID_CURSOR` |
| `limit` above the cap | `422` | `LIMIT_EXCEEDED` |
| `sort` not in the whitelist | `422` | `INVALID_SORT_FIELD` |

**`sort` is whitelisted, not passed through.** Mapping a client string onto a SQL fragment is a direct injection path, and even parameterised it lets a caller force an unindexed sort on a large table — a cheap denial-of-service. The whitelist maps to explicit, indexed column expressions.

---

### 5.6 `GET /api/v1/tasks/{task_id}`

**Auth:** Owner.

Returns the full representation with an `ETag` header. Supports `If-None-Match` and returns `304 Not Modified` when unchanged — a meaningful bandwidth saving for mobile clients polling a task detail view.

| Scenario | Status | Code |
|---|---|---|
| Task belongs to another user | `404` | `TASK_NOT_FOUND` — indistinguishable from genuine absence |
| Task soft-deleted | `404` | `TASK_NOT_FOUND` unless `?include_deleted=true` |
| `task_id` not a unique identifier (UUID) | `422` | `INVALID_UUID` |
| `If-None-Match` matches | `304` | Empty body |

---

### 5.7 `PATCH /api/v1/tasks/{task_id}`

**Auth:** Owner.

```http
PATCH /api/v1/tasks/018f2c1a-...
Authorization: Bearer <access_token>
If-Match: "3"
```

```json
{ "status": "in_progress", "priority": "medium" }
```

Only supplied fields change. `If-Match` is optional but strongly recommended; when present it is enforced. `id`, `owner_id`, `version`, `created_at` are immutable and rejected if sent.

**Response — `200 OK`** with the full updated representation and `ETag: "4"`.

| Scenario | Status | Code | Note |
|---|---|---|---|
| Not the owner | `404` | `TASK_NOT_FOUND` | Denial logged internally |
| `If-Match` stale | `412` | `PRECONDITION_FAILED` | Body carries the current representation so the client can merge |
| Concurrent write won the race | `409` | `CONCURRENT_MODIFICATION` | Client should re-read and retry |
| Illegal transition, e.g. `done → todo` | `422` | `INVALID_STATE_TRANSITION` | Response lists the allowed next states |
| Immutable field supplied | `422` | `FIELD_IMMUTABLE` | |

---

### 5.8 `DELETE /api/v1/tasks/{task_id}`

**Auth:** Owner. Soft delete, recoverable for 30 days.

**Response — `204 No Content`.** Deleting an already-deleted task also returns `204`: `DELETE` is idempotent and the caller's intent is satisfied. Returning `404` on the second call breaks naive retry logic for no security benefit — the caller already knew the resource existed.

---

### 5.9 `GET /api/v1/admin/tasks`

**Auth:** Admin.

Same filters as the user endpoint, plus `owner_id`, `created_before` / `created_after`, and `include_deleted`. Unlike the user endpoint, `owner_id` **is** accepted here — that is the entire purpose of the namespace, and it is why the namespace is separate.

**Response — `200 OK`.** Each item is enriched with an `owner` summary so an operator does not need an N+1 fan-out to see who owns what.

```json
{
  "data": [
    {
      "id": "018f2c1a-...",
      "title": "Write the architecture assessment",
      "status": "todo",
      "owner": { "id": "018f0b3d-...", "email": "ada@example.com", "display_name": "Ada Lovelace" },
      "version": 1,
      "created_at": "2026-09-22T09:14:02Z"
    }
  ],
  "page": { "next_cursor": null, "has_more": false, "limit": 20 }
}
```

```http
X-Admin-Action-Logged: true
```

| Scenario | Status | Code | Note |
|---|---|---|---|
| Caller is not an admin | `404` | `NOT_FOUND` | The admin namespace is invisible to non-admins; logged at high severity |
| `owner_id` refers to no user | `200` | — | Empty page, not `404`: the *collection* exists |
| Result set above the export threshold | `422` | `USE_EXPORT_ENDPOINT` | Bulk extraction is an async, audited job, not a pagination loop |

Every call writes an `admin.tasks_viewed` audit record including the target user, the filters, and the result count. The `X-Admin-Action-Logged` header makes the audit visible in the admin UI, which is a mild but effective deterrent against casual browsing.

---

### 5.10 `PATCH /api/v1/admin/users/{user_id}`

**Auth:** Admin. The most dangerous endpoint in the system, since it controls role assignment.

```json
{ "role": "admin", "status": "active" }
```

| Scenario | Status | Code | Note |
|---|---|---|---|
| Admin targets their own account | `422` | `SELF_MODIFICATION_FORBIDDEN` | Prevents self-escalation and accidental self-lockout |
| Demoting or suspending the last admin | `422` | `LAST_ADMIN_PROTECTED` | Also enforced by a database constraint |
| Role granted | `200` | — | `HIGH` severity audit entry, plus an email to both parties |

Privilege change is the step an attacker needs to convert a single compromised account into persistent control. It is audited at the highest severity, notifies out-of-band, and is a primary alerting signal.

---

### 5.11 Health and Operational Endpoints

| Endpoint | Checks | Notes |
|---|---|---|
| `GET /health/live` | Process is responsive | **No dependency checks.** A liveness probe that fails on a database blip makes the orchestrator kill every healthy replica during a database incident, converting a degradation into a total outage |
| `GET /health/ready` | Database reachable, migrations current, Redis reachable, secrets loaded | Removes the instance from the load balancer without killing it |
| `GET /health/startup` | Boot-time initialisation complete | Gives slow starts room before liveness applies |
| `GET /metrics` | Prometheus exposition | Bound to the internal network only |

The liveness/readiness distinction is one of the highest-leverage reliability details in the whole design, and it is routinely conflated. See [Availability section 3](AVAILABILITY.md#3-application-instance-failure).

---

## 6. Rate Limiting

Token-bucket counters in Redis, keyed by principal where available and by IP otherwise.

| Scope | Limit | Reasoning |
|---|---|---|
| Unauthenticated, per IP | 60/min | Protects the login and registration surface |
| `POST /auth/login` | 10/15 min per IP, 5/15 min per account | Two axes defeat both brute force and credential stuffing |
| `POST /auth/register` | 5/hour per IP | Limits automated account creation |
| `POST /auth/password/forgot` | 3/hour per account | Prevents mailbox flooding |
| Authenticated reads | 300/min per user | Comfortably above real usage |
| Authenticated writes | 60/min per user | |
| Admin endpoints | 120/min per admin | Separate bucket; a busy admin must not exhaust a user-facing limit |

Every response carries `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset`, so a well-behaved client can pace itself rather than discovering the limit by being rejected. `429` responses always include `Retry-After`.

---

## 7. Versioning and Deprecation

`/api/v1` is a stability contract. Within a major version, only additive changes ship: new optional fields, new endpoints, new enum values in fields documented as extensible. Clients are expected to ignore unknown response fields.

Breaking changes — removing or renaming a field, tightening validation, changing a status code, altering pagination semantics — require `/api/v2`, which runs alongside `v1`.

Deprecation is a defined process, not an announcement: advertise `Deprecation` and `Sunset` headers per RFC 9745, give a minimum 6-month window, instrument per-version and per-client usage so the decommission decision rests on data, and brown-out with brief scheduled `410`s before the final cut so silent integrations surface while someone is watching.

---

## 8. What I Left Out of v1

| Omitted | Reasoning |
|---|---|
| Bulk task operations | No evidence of need; a bulk endpoint materially complicates transaction and partial-failure semantics. Would return `207 Multi-Status` when added |
| Webhooks / subscriptions | Real value, real cost: delivery guarantees, retries, SSRF protection on customer-supplied web addresses (URLs), signature verification. Deserves its own design |
| WebSocket live updates | Stateful connections change the scaling and deployment model. Polling with `ETag`/`304` is adequate at this scale |
| GraphQL | See [High-Level Design (HLD) section 14](HLD.md#14-deliberate-non-goals) |
| Task sharing / collaboration | The requirement is explicitly single-owner. Sharing would replace the ownership model with an ACL model — the single biggest latent change in the design, and one I would not pre-build speculatively |
| API keys for machine clients | No stated requirement. The token model extends to them cleanly when there is one |
