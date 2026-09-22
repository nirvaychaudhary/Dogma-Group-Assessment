# Diagram 3 — Task Management Flow

Task lifecycle, the authorization decision, write-path transaction structure, and administrator cross-user access. Detailed discussion in [System Design §4–§8](../SYSTEM_DESIGN.md#4-user-accessing-their-tasks).

---

## 3.1 Task Status State Machine

```mermaid
stateDiagram-v2
    [*] --> todo: POST /tasks<br/>version = 1

    todo --> in_progress: start work
    todo --> done: complete directly
    in_progress --> done: complete<br/>sets completed_at
    done --> in_progress: reopen<br/>clears completed_at
    todo --> todo: edit fields
    in_progress --> in_progress: edit fields

    todo --> deleted: DELETE — soft
    in_progress --> deleted: DELETE — soft
    done --> deleted: DELETE — soft

    deleted --> todo: POST /restore<br/>within 30 days
    deleted --> in_progress: POST /restore
    deleted --> done: POST /restore

    deleted --> [*]: purged by the<br/>daily job after 30 days

    note right of done
        done to todo is REJECTED — 422.
        Reopening moves to in_progress,
        which keeps the history coherent.
    end note

    note right of deleted
        Every read path filters
        deleted_at IS NULL, served by
        a partial index.
    end note
```

Every transition increments `version`, which is the value surfaced as the HTTP `ETag`. The state machine lives in the domain entity, so it holds for every caller — HTTP handlers, background jobs, and future admin tooling alike.

---

## 3.2 The Authorization Decision

This is the single most important control in the system, so it is drawn on its own.

```mermaid
flowchart TB
    REQ["Request for a task resource"] --> AUTHN{"Valid access<br/>token?"}
    AUTHN -->|"No"| E401["401"]
    AUTHN -->|"Yes"| PRINCIPAL["Build Principal from token claims<br/>— NEVER from the request body"]

    PRINCIPAL --> NS{"Which<br/>namespace?"}

    NS -->|"/api/v1/tasks"| USERPATH
    NS -->|"/api/v1/admin/tasks"| ADMINPATH

    subgraph USERPATH["User namespace — scoped by construction"]
        U1["Repository query always applies<br/>WHERE owner_id = principal.user_id"]
        U2{"Row returned?"}
        U1 --> U2
        U2 -->|"No"| U404["404 TASK_NOT_FOUND<br/>— identical whether the task<br/>is absent or owned by someone else"]
        U2 -->|"Yes"| U3["policy.authorize(principal,<br/>action, task)"]
        U3 --> U4{"Permitted?"}
        U4 -->|"No"| U404
        U4 -->|"Yes"| UOK["Proceed"]
    end

    subgraph ADMINPATH["Admin namespace — privileged by default"]
        A1{"principal.role<br/>== admin?"}
        A1 -->|"No"| A404["404 — the namespace does<br/>not acknowledge itself.<br/>Denial logged HIGH severity."]
        A1 -->|"Yes"| A2["Query WITHOUT an owner filter"]
        A2 --> A3{"Row returned?"}
        A3 -->|"No"| A404B["404 TASK_NOT_FOUND"]
        A3 -->|"Yes"| A4["Write an audit record —<br/>INCLUDING for reads"]
        A4 --> AOK["Proceed"]
    end

    UOK --> HANDLE["Execute the use case"]
    AOK --> HANDLE
```

Three properties this diagram is meant to make obvious:

**The user path narrows at the query, before any check runs.** If someone later adds an endpoint and forgets to call `policy.authorize`, the query still returns nothing for another user's task. The system fails toward disclosing less.

**Both denial paths converge on `404`.** A `403` would confirm the resource exists and turn the endpoint into an existence oracle.

**The admin path audits reads.** In this domain the sensitive administrative action is usually looking, not changing.

---

## 3.3 List Own Tasks — Keyset Pagination

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant A as API
    participant S as Task Service
    participant RP as Repository
    participant DB as PostgreSQL

    C->>A: GET /api/v1/tasks?status=todo&limit=20&cursor=eyJ...
    A->>A: Validate: limit <= 100,<br/>sort against a whitelist,<br/>decode and verify the cursor
    A->>S: list_tasks(principal, filters, page)
    S->>RP: find_by_owner(principal.user_id, ...)

    Note over RP,DB: owner_id comes from the Principal.<br/>The request schema has no such field.

    RP->>DB: SELECT id, title, status, priority,<br/>due_at, version, created_at<br/>FROM tasks<br/>WHERE owner_id = :principal_id<br/>AND deleted_at IS NULL<br/>AND status = ANY(:statuses)<br/>AND (created_at, id) < (:cur_ts, :cur_id)<br/>ORDER BY created_at DESC, id DESC<br/>LIMIT 21

    Note over DB: Served by<br/>ix_tasks_owner_status —<br/>index range scan, O(log n)<br/>at ANY page depth

    DB-->>RP: up to 21 rows
    RP->>RP: has_more = (count == 21)<br/>— no second COUNT(*) query
    RP->>RP: next_cursor from row 20
    RP-->>S: Page
    S-->>A: Page
    A-->>C: 200 {data: [20 items],<br/>page: {next_cursor, has_more: true}}
```

Two details carry real weight. The `SELECT` names its columns and **omits `description`**, so a 50-item list does not transfer hundreds of kilobytes and evict hot pages from the buffer cache. And `LIMIT 21` for a page of 20 determines `has_more` without a `COUNT(*)`, which on a large filtered set would be the most expensive part of the request.

---

## 3.4 Write Path — One Transaction

All three mutations share the same structure, which is the point of drawing them together.

```mermaid
flowchart TB
    subgraph CREATE["CREATE — POST /tasks"]
        C1["Validate payload"] --> C2["policy.authorize task:create"]
        C2 --> C3["BEGIN"]
        C3 --> C4["INSERT idempotency_keys<br/>— the unique constraint arbitrates races"]
        C4 --> C5["Task.create(owner_id from Principal)<br/>— owner_id is NEVER client-supplied"]
        C5 --> C6["INSERT tasks — version = 1"]
        C6 --> C7["INSERT audit_log"]
        C7 --> C8["UPDATE idempotency_keys with the response"]
        C8 --> C9["COMMIT"]
        C9 --> C10["201 + Location + ETag: 1"]
    end

    subgraph UPDATE["UPDATE — PATCH /tasks/{id}"]
        U1["Validate partial payload"] --> U2["BEGIN"]
        U2 --> U3["SELECT the task"]
        U3 --> U4["policy.authorize — AFTER the load,<br/>INSIDE the transaction"]
        U4 --> U5{"If-Match matches<br/>version?"}
        U5 -->|"No"| U6["ROLLBACK → 412<br/>+ current representation"]
        U5 -->|"Yes"| U7["task.apply(changes)<br/>— domain validates the transition"]
        U7 --> U8["UPDATE ... SET version = version + 1<br/>WHERE id = :id AND version = :expected"]
        U8 --> U9{"Rows<br/>affected?"}
        U9 -->|"0 — lost the race"| U10["ROLLBACK → 409"]
        U9 -->|"1"| U11["INSERT audit_log with a<br/>before/after field diff"]
        U11 --> U12["COMMIT → 200 + new ETag"]
    end

    subgraph DELETE["DELETE — soft"]
        D1["BEGIN"] --> D2["SELECT + authorize"]
        D2 --> D3["UPDATE SET deleted_at = now(),<br/>deleted_by, version = version + 1"]
        D3 --> D4["INSERT audit_log"]
        D4 --> D5["COMMIT → 204<br/>— idempotent: 204 again if already deleted"]
    end
```

The invariant every branch preserves: **the business mutation, its audit record, and any outbox event commit together or not at all.** There is no path by which a task changes without an audit row, and no path by which an event is published for a change that rolled back.

---

## 3.5 Concurrent Update — Why the Version Guard Exists

```mermaid
sequenceDiagram
    autonumber
    participant A as Client A
    participant B as Client B
    participant API as API
    participant DB as PostgreSQL

    A->>API: GET /tasks/T1
    API-->>A: 200, ETag: "3"
    B->>API: GET /tasks/T1
    API-->>B: 200, ETag: "3"

    Note over A,B: Both clients now hold version 3.

    A->>API: PATCH /tasks/T1<br/>If-Match: "3" {status: done}
    API->>DB: UPDATE ... SET status='done', version=4<br/>WHERE id=T1 AND version=3
    DB-->>API: 1 row
    API-->>A: 200, ETag: "4"

    B->>API: PATCH /tasks/T1<br/>If-Match: "3" {priority: low}
    API->>DB: SELECT — version is now 4
    Note over API: If-Match "3" != 4 → precondition failed
    API-->>B: 412 + the current representation

    B->>B: Merge against the current state
    B->>API: PATCH /tasks/T1<br/>If-Match: "4" {priority: low}
    API->>DB: UPDATE ... version=5 WHERE version=4
    DB-->>API: 1 row
    API-->>B: 200, ETag: "5"
```

Without the version guard, Client B's write would silently overwrite Client A's completion. The user would mark a task done, watch it revert, and have no explanation. Optimistic concurrency turns that invisible data loss into an explicit `412` the client can resolve.

Pessimistic locking would also prevent it, at the cost of holding a database row lock across human think-time — converting a rare conflict into a common stall.

---

## 3.6 Administrator Cross-User Access

```mermaid
sequenceDiagram
    autonumber
    participant AD as Administrator
    participant MW as Middleware
    participant AR as Admin Router
    participant P as Policy
    participant S as Admin Service
    participant DB as PostgreSQL
    participant SEC as Security Monitoring

    AD->>MW: PATCH /api/v1/admin/tasks/T9<br/>{status: done}
    MW->>MW: Verify JWT → Principal
    MW->>MW: Admin rate-limit bucket<br/>— separate from the user bucket
    MW->>AR: Routed into the admin namespace
    AR->>P: Namespace-wide guard:<br/>authorize(principal, "admin:task:update")

    alt role != admin
        P->>DB: audit_log (authz.denied) severity HIGH
        AR-->>AD: 404 — the namespace is invisible
        DB-->>SEC: Denial spike feeds authz_denials_total
    else role == admin
        rect rgb(238, 245, 255)
        S->>DB: BEGIN
        S->>DB: SELECT task T9 — no owner filter
        S->>DB: UPDATE ... version = version + 1
        S->>DB: INSERT audit_log<br/>(admin.task_updated, actor_id,<br/>target_user_id, before/after diff,<br/>correlation_id, ip)
        S->>DB: INSERT outbox_events<br/>(notify the owner)
        S->>DB: COMMIT
        end
        AR-->>AD: 200 + X-Admin-Action-Logged: true
        DB-->>SEC: admin_cross_user_access_total
        SEC->>SEC: Alert on anomalous volume<br/>or off-hours access
    end
```

Note that the owner is **notified** when an administrator modifies their task. Silent administrative modification is how users lose trust in a system, and the notification costs one outbox row.

---

## 3.7 Delete, Restore and Purge

```mermaid
flowchart LR
    ACTIVE["Active task<br/>deleted_at IS NULL"]
    SOFT["Soft-deleted<br/>deleted_at = T<br/>invisible to all reads"]
    GONE["Purged<br/>row removed"]

    ACTIVE -->|"DELETE /tasks/{id}<br/>owner or admin"| SOFT
    SOFT -->|"POST /tasks/{id}/restore<br/>within 30 days"| ACTIVE
    SOFT -->|"daily purge job<br/>after 30 days"| GONE
    ACTIVE -->|"GDPR account erasure<br/>— a separate, genuinely<br/>destructive path"| GONE
```

Soft delete and GDPR erasure are **deliberately different paths**. Soft delete serves user convenience and is reversible; erasure must be irreversible to satisfy a data-subject request. Conflating them is a common and expensive compliance mistake — a "deleted" record that is still queryable does not satisfy a right-to-erasure obligation.

The erasure job also pseudonymises the subject's identifier in the audit log rather than deleting the rows, preserving the integrity of a security control while removing the personal data.

---

**Related:** [System Design §4–§8](../SYSTEM_DESIGN.md#4-user-accessing-their-tasks) · [Data Architecture §6 Transaction Boundaries](../DATA_ARCHITECTURE.md#6-transaction-boundaries) · [API Design §5](../API_DESIGN.md#5-endpoint-detail)
