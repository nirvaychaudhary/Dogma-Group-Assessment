# Decision Log

Short forms are written out the first time they appear. The full list is in the [glossary](docs/GLOSSARY.md).

Architecturally significant decisions — those that are expensive to reverse, or that constrain later choices. Routine implementation choices are not recorded here.

Each entry states the decision, why it was made, what else was considered, and what it costs. **The trade-off column is the important one.** A decision with no stated cost has not been examined properly.

| # | Decision | Status |
|---|---|---|
| [D1](#d1--modular-monolith-rather-than-microservices) | Modular monolith rather than microservices | Accepted |
| [D2](#d2--python-with-fastapi) | Python with FastAPI | Accepted |
| [D3](#d3--postgresql-as-the-single-system-of-record) | PostgreSQL as the single system of record | Accepted |
| [D4](#d4--short-lived-jwt-access-tokens-with-opaque-rotating-refresh-tokens) | Short-lived JSON Web Token (JWT) access tokens with opaque rotating refresh tokens | Accepted |
| [D5](#d5--argon2id-for-password-hashing) | Argon2id for password hashing | Accepted |
| [D6](#d6--a-single-authorization-policy-engine) | A single authorization policy engine | Accepted |
| [D7](#d7--404-rather-than-403-for-object-level-denials) | `404` rather than `403` for object-level denials | Accepted |
| [D8](#d8--a-separate-admin-namespace) | A separate `/admin` namespace | Accepted |
| [D9](#d9--uuidv7-primary-keys) | time-sorted unique identifier (UUIDv7) primary keys | Accepted |
| [D10](#d10--keyset-pagination-rather-than-offset) | Keyset pagination rather than offset | Accepted |
| [D11](#d11--optimistic-concurrency-with-a-version-column) | Optimistic concurrency with a version column | Accepted |
| [D12](#d12--soft-delete-with-a-separate-hard-erasure-path) | Soft delete, with a separate hard-erasure path | Accepted |
| [D13](#d13--transactional-outbox-for-asynchronous-work) | Transactional outbox for asynchronous work | Accepted |
| [D14](#d14--audit-log-written-in-the-business-transaction-append-only) | Audit log written in the business transaction, append-only | Accepted |
| [D15](#d15--no-caching-of-user-task-data-at-launch) | No caching of user task data at launch | Accepted |
| [D16](#d16--redis-fails-open-for-rate-limiting-and-the-token-denylist) | Redis fails open for rate limiting and the token denylist | Accepted |
| [D17](#d17--single-region-deployment) | Single-region deployment | Accepted |
| [D18](#d18--ecs-fargate-rather-than-kubernetes) | Elastic Container Service (ECS) Fargate rather than Kubernetes | Accepted |
| [D19](#d19--idempotency-keys-on-post) | Idempotency keys on `POST` | Accepted |
| [D20](#d20--a-domain-layer-separate-from-orm-models) | A domain layer separate from object-relational mapping (ORM) models | Accepted |
| [D21](#d21--uniform-registration-responses-to-prevent-enumeration) | Uniform registration responses to prevent enumeration | Accepted |
| [D22](#d22--mfa-deferred-from-v1) | multi-factor authentication (MFA) deferred from v1 | Accepted with reservations |
| [D23](#d23--architecture-boundaries-enforced-in-ci) | Architecture boundaries enforced in CI | Accepted |
| [D24](#d24--rfc-9457-problemjson-for-errors) | Request for Comments (RFC) 9457 problem+json for errors | Accepted |
| [D25](#d25--mermaid-diagrams-in-markdown) | Mermaid diagrams in Markdown | Accepted |

---

### D1 — Modular monolith rather than microservices

**Decision.** One deployable service, internally partitioned into modules with CI-enforced boundaries, running as multiple stateless replicas. Workers run as a separate process sharing the codebase.

**Reason.** The domain has one aggregate root and one supporting aggregate. The most important invariant in the system — "this actor may touch this object" — would become a network call and a distributed consistency problem if split. For a team of 2–4, three pipelines and three on-call surfaces is a large cost with no corresponding benefit. Microservices solve organisational scaling and independent failure domains; we have neither problem.

**Alternatives.** Microservices from the start — rejected as premature. A plain layered monolith — rejected because boundaries decay without enforcement. Serverless functions — rejected for cold starts on a latency-sensitive path and awkward connection pooling.

**Trade-off.** Modules cannot be deployed or scaled independently, and one module's memory leak affects all of them. Mitigated by keeping module boundaries real, so extraction is a contained change rather than a rewrite. See [High-Level Design (HLD) section 3](docs/HLD.md#3-architecture-style-modular-monolith).

---

### D2 — Python with FastAPI

**Decision.** Python 3.12 with FastAPI, Pydantic v2, SQLAlchemy 2.0 async, and Alembic.

**Reason.** The assessment specifies Python. Within Python, FastAPI gives Pydantic validation at the boundary — which is a security control, not only ergonomics — OpenAPI generated from the code so documentation cannot drift, native async for an I/O-bound workload, and first-class dependency injection that makes the testing approach practical.

**Alternatives.** Django Representational State Transfer (REST) Framework — excellent batteries-included ecosystem and a ready-made admin, but the ORM and the framework are tightly coupled to a layering that fights the domain separation in D20, and async support is still partial. Flask — too little structure; we would rebuild validation and serialisation ourselves. Litestar — technically strong, smaller ecosystem and hiring pool.

**Trade-off.** Async Python demands discipline: one synchronous database call in an async handler blocks the event loop and stalls every concurrent request on that worker. Mitigated by an async-only I/O rule, a thread pool for processor-heavy hashing, and load testing that would surface it.

---

### D3 — PostgreSQL as the single system of record

**Decision.** PostgreSQL 16 holds users, tasks, audit log, tokens, idempotency keys, and the outbox. No second primary datastore.

**Reason.** The core invariant and the dominant query shape are both relational. Every write needs multi-row atomicity across the entity, its audit record, and its outbox event. 50 million rows is comfortably a single-instance workload. Constraints in the database are the only validation that cannot be bypassed by a migration, a bulk job, or a bug.

**Alternatives.** MongoDB — flexible schema is a liability for a stable, well-understood domain, and referential integrity would move into application code where the security boundary lives. DynamoDB — hostile to the ad-hoc filtering administrators need, and the deepest available lock-in. MySQL — reasonable; PostgreSQL wins on partial and expression indexes, richer types, and transactional database structure change (DDL).

**Trade-off.** Not optimal for full-text search at very large scale or for extreme write throughput. Both are far beyond the sizing baseline, and both have a documented path. See [Data Architecture section 1](docs/DATA_ARCHITECTURE.md#1-technology-choice-postgresql-16).

---

### D4 — Short-lived JWT access tokens with opaque rotating refresh tokens

**Decision.** 15-minute signed with the Edwards-curve Digital Signature Algorithm (EdDSA) JWT access tokens, not stored server-side. 30-day opaque refresh tokens stored as SHA-256 hashes, rotated on every use, with reuse detection that revokes the whole token family.

**Reason.** The two token types have genuinely different jobs. The access token is presented on every request, so it must be verifiable without I/O. The refresh token is presented rarely and lives for 30 days, so revocability matters far more than verification speed — making it a JWT would surrender that control for no gain. Rotation with reuse detection turns a stolen refresh token from a silent month-long foothold into a detectable, self-revoking event.

**Alternatives.** Server-side sessions — simplest and fully revocable, but a datastore lookup on every request and awkward for mobile clients. Long-lived access tokens — unacceptable revocation exposure. Both tokens as JSON Web Tokens (JWTs) — gives up revocation for a saving that does not matter at refresh frequency.

**Trade-off.** Access tokens cannot be revoked instantly; exposure is bounded to 15 minutes, narrowed further by a `jti` denylist and a `token_version` counter. Rotation also creates a race for clients that refresh concurrently, handled with an explicit 10-second grace window. See [System Design section 3](docs/SYSTEM_DESIGN.md#3-token-refresh-with-reuse-detection).

---

### D5 — Argon2id for password hashing

**Decision.** Argon2id at 64 MiB, 3 iterations, parallelism 4, tuned to ~200 ms per verification and re-benchmarked annually. Work factor versioned, with transparent rehash on login.

**Reason.** Memory-hard, so the GPU and ASIC parallelism that makes bcrypt and PBKDF2 attackable becomes expensive. The hybrid `id` variant resists both side-channel and time-memory trade-off attacks. It is the Open Worldwide Application Security Project (OWASP) first recommendation.

**Alternatives.** bcrypt — battle-tested but only CPU-hard, and silently truncates at 72 bytes. PBKDF2 — FIPS-approved but the weakest against dedicated hardware. scrypt — memory-hard and fine, but with less tuning guidance and library maturity.

**Trade-off.** ~200 ms and 64 MiB per verification is a real resource cost and a denial-of-service consideration under a login flood. Mitigated by capping password length at 128 characters, running verification outside the database transaction, and rate limiting on both Internet Protocol address (IP) and account axes.

---

### D6 — A single authorization policy engine

**Decision.** All authorization decisions flow through `security/policy.py`. It raises rather than returning a boolean, and it audits its own denials.

**Reason.** Broken object-level authorization is the system's dominant risk, and it is almost always caused by the check being scattered across handlers where one of them forgets. One decision point means the entire authorization model is reviewable in one sitting and testable as one unit. Raising instead of returning means a forgotten result cannot silently grant access — `policy.can(...)` on a line by itself would compile and pass review.

**Alternatives.** Decorators per route — easy to omit on a new endpoint, and cannot see the resource. Checks inline in each service — the failure mode this is designed to prevent. An external policy engine such as OPA — powerful, but a network hop and a second language for two roles.

**Trade-off.** A central module every feature depends on, which could become a bottleneck for change. Acceptable, and arguably desirable: changes to authorization *should* be deliberate and reviewed. See [Security section 5](docs/SECURITY.md#5-authorization).

---

### D7 — `404` rather than `403` for object-level denials

**Decision.** Requesting a task owned by someone else returns `404`, identical to a task that does not exist. Non-admins hitting `/admin` also get `404`.

**Reason.** A `403` confirms the resource exists. With enumerable access patterns that turns every endpoint into an oracle for mapping the object space and confirming relationships between accounts. The denial is logged internally at high severity, so we lose nothing in detection.

**Alternatives.** `403` for clarity — better developer experience, worse security. `403` only for authenticated users — inconsistent, and still leaks.

**Trade-off.** Genuinely harder to debug a legitimate permission problem, and support cannot immediately distinguish "your task was deleted" from "you're signed in as the wrong account". Mitigated by correlation IDs and internal denial logging that make the real cause a one-query lookup.

---

### D8 — A separate `/admin` namespace

**Decision.** Administrative operations live under `/api/v1/admin/*` with a namespace-wide guard, rather than the user endpoints widening their scope for admin callers.

**Reason.** Role-widening puts the privilege boundary inside a function that also serves unprivileged traffic, and the failure mode of a bug there is full cross-tenant disclosure. A separate namespace makes every privileged operation statically enumerable, privileged-by-default for new endpoints, and independently rate-limited and audited.

**Alternatives.** One endpoint with conditional scope — rejected on blast radius. A separate deployed admin service — stronger isolation, disproportionate for this scale.

**Trade-off.** Duplication between user and admin handlers. Kept small by sharing service logic; thin routing duplication is cheap insurance against a catastrophic authorization bug. See [System Design section 8](docs/SYSTEM_DESIGN.md#8-administrator-accessing-another-users-task).

---

### D9 — UUIDv7 primary keys

**Decision.** UUIDv7 for all entity identifiers.

**Reason.** Sequential integers are enumerable, which converts any authorization gap into a full dump and leaks business volume. random unique identifier (UUIDv4) fixes that but is random, so index inserts scatter across the B-tree, causing page splits and bloat on a high-insert table. UUIDv7 embeds a millisecond timestamp in its high bits, so inserts append at the index edge like an integer while remaining unguessable.

**Alternatives.** `BIGSERIAL` — fastest and smallest, but enumerable. UUIDv4 — safe but poor index locality. ULID — equivalent properties, less native database support.

**Trade-off.** 16 bytes rather than 8, and less readable web addresses (URLs). Both acceptable given that object-level authorization is the primary risk.

---

### D10 — Keyset pagination rather than offset

**Decision.** Opaque cursors over `(created_at, id)`, no `OFFSET`, no total count by default.

**Reason.** `OFFSET 10000` makes the database read and discard 10,000 rows, so cost grows linearly with depth. Worse, offset pagination is *incorrect* under concurrent writes: an insert while a user pages shifts every subsequent page, producing duplicates and skips. Keyset pagination maps directly onto the composite index and is O(log n) at any depth.

**Alternatives.** Offset — simpler, allows jumping to an arbitrary page, breaks at depth and under concurrency. A hybrid — added complexity for a feature nobody asked for.

**Trade-off.** No random page access and no cheap total count. For a task list, "next/previous" and infinite scroll are the real interaction patterns, and exact counts are available behind an explicit opt-in.

---

### D11 — Optimistic concurrency with a version column

**Decision.** An integer `version` per task, surfaced as an Hypertext Transfer Protocol (HTTP) `ETag`, checked via `If-Match` and enforced with `UPDATE ... WHERE version = :expected`.

**Reason.** Two clients editing the same task is the realistic conflict here, and without a guard the second write silently destroys the first — invisible data loss with no explanation for the user. Optimistic locking detects it and gives the client enough information to merge. Conflicts are rare, so the optimistic assumption holds.

**Alternatives.** Pessimistic `SELECT FOR UPDATE` — holds a row lock across human think-time, turning a rare conflict into a common stall and inviting deadlocks. Last-write-wins — silent data loss. CRDTs — vastly disproportionate.

**Trade-off.** Clients must handle `409` and `412`. That is a real client-side cost, and it is the correct place for the cost to land.

---

### D12 — Soft delete, with a separate hard-erasure path

**Decision.** `DELETE /tasks/{id}` sets `deleted_at`, recoverable for 30 days, then purged by a daily job. General Data Protection Regulation (GDPR) account erasure is a separate, genuinely destructive path that pseudonymises the subject in the audit log rather than deleting it.

**Reason.** Accidental deletion is the most common user-visible data-loss event, and administrators need to answer "what happened to this task?". But soft delete does **not** satisfy a right-to-erasure request — a record that is still queryable has not been erased. Conflating the two is a common and expensive compliance mistake.

**Alternatives.** Hard delete only — simpler, unrecoverable, and no audit trail. An archive table — equivalent benefit, more complex queries and migrations.

**Trade-off.** Every query carries a `deleted_at IS NULL` filter, and every unique constraint over soft-deletable rows must be partial or a deleted task permanently blocks re-creating one with the same title. Both are handled by partial indexes and are called out explicitly in [Data Architecture section 3](docs/DATA_ARCHITECTURE.md#3-schema-decisions-that-matter).

---

### D13 — Transactional outbox for asynchronous work

**Decision.** Services insert an `outbox_events` row inside the business transaction. A dispatcher polls and publishes to the queue.

**Reason.** Enqueuing directly to Redis inside a database transaction is a dual write across two systems with no shared commit. If the transaction then rolls back, an email goes out about a change that never happened. The outbox makes the event part of the same atomic commit.

**Alternatives.** Direct enqueue — simple, incorrect under failure. Change data capture via Debezium — robust and more infrastructure than this volume warrants. Two-phase commit — unavailable and undesirable.

**Trade-off.** An extra table, a dispatcher process, polling latency of a second or two, and at-least-once delivery that consumers must absorb by being idempotent. Correctness is worth all of that.

---

### D14 — Audit log written in the business transaction, append-only

**Decision.** Every state change writes an `audit_log` row in the same transaction. The application's database role has `INSERT` and `SELECT` only — no `UPDATE`, no `DELETE`. Administrator **reads** are audited, not just writes.

**Reason.** A log line emitted after commit is lost if the process dies in between — precisely when it matters most. Transactional writing makes a state change without an audit record impossible. Append-only privileges mean an attacker who compromises the application cannot erase their own tracks. Auditing admin reads matters because in this domain the sensitive administrative action is usually looking, not changing, and a mutation-only trail cannot answer "did anyone read this user's tasks?"

**Alternatives.** Application logs as the audit trail — not durable, not tamper-evident. Asynchronous audit writes — faster, loses records exactly when needed. Database triggers — invisible to application developers and hard to test.

**Trade-off.** An extra insert on every mutation, and the fastest-growing table in the system. Mitigated by monthly partitioning and archival by dropping partitions.

---

### D15 — No caching of user task data at launch

**Decision.** Cache JSON Web Key Set (JWKS), rate-limit counters, the token denylist, the breached-password corpus, and configuration. Do **not** cache task lists or task detail. Use HTTP `ETag`/`304` from day one.

**Reason.** Task data is per-user, so there is no reuse across users and the hit rate is bounded by how often one person reloads. It is write-heavy, so entries invalidate almost as fast as they populate. And a cache-key bug on per-user data serves one user's tasks to another — the exact failure the security model exists to prevent, now reachable with no authorization flaw at all. Improving 95th percentile (p95) by 20 ms on a query that already returns in under 10 ms is not worth that.

**Alternatives.** Cache-aside on task lists — the intuitive move, rejected above. Write-through — same risks plus more complexity.

**Trade-off.** Every read hits the database. That is fine at the sizing baseline, and [Scalability section 5](docs/SCALABILITY.md#5-caching) states the measured signal that would change the decision.

---

### D16 — Redis fails open for rate limiting and the token denylist

**Decision.** If Redis is unavailable, rate limiting allows requests and the denylist check is skipped. Both are alerted. Every authorization decision continues to fail closed.

**Reason.** Failing closed would mean a cache outage logs out every user and rejects all traffic — a self-inflicted total outage. Exposure is bounded: the web application firewall (WAF) still enforces coarse rate limits at the edge, the `token_version` check still runs against PostgreSQL, and denylisted tokens expire within 15 minutes anyway.

**Alternatives.** Fail closed — safer in theory, converts a degradation into an outage. In-memory fallback — per-replica limits that silently multiply by replica count and change when you autoscale.

**Trade-off.** A genuine, narrow security softening during a Redis outage. I would rather state it explicitly than have it emerge as a surprise. See [Availability section 5](docs/AVAILABILITY.md#5-redis-failure).

---

### D17 — Single-region deployment

**Decision.** One region, Multi-availability zone (AZ) within it. Cross-region snapshot copies and a rehearsed rebuild runbook, but no active standby.

**Reason.** The 99.9% target does not require multi-region. Active-active roughly doubles cost, introduces cross-region replication lag into the authorization path, and requires write-conflict resolution in a system that has no need for it.

**Alternatives.** Active-active — the right answer at 99.99% or with data-residency requirements. Warm standby — cheaper than active-active, still a significant cost for a rare event.

**Trade-off.** A regional failure means hours of downtime. This is an accepted, documented business decision with a tested recovery path, not an oversight. See [Availability section 2](docs/AVAILABILITY.md#2-single-points-of-failure).

---

### D18 — ECS Fargate rather than Kubernetes

**Decision.** Containers on ECS Fargate behind an Application Load Balancer (ALB).

**Reason.** Kubernetes is a platform to operate, not just a place to run containers — upgrades, networking, RBAC, ingress controllers, and operators all become the team's responsibility. For 2–4 engineers running one service, that cost buys flexibility we cannot yet use.

**Alternatives.** EKS/Kubernetes — the right answer with many heterogeneous services or a platform team. EC2 with systemd — more control, more undifferentiated work. Lambda — cold starts on a latency-sensitive path and awkward database connection pooling.

**Trade-off.** Less control over scheduling and networking, and a migration cost later if Kubernetes becomes justified. Because the artefact is a standard OCI container, that migration is a deployment change rather than an application change.

---

### D19 — Idempotency keys on `POST`

**Decision.** `POST /tasks` and `POST /auth/register` accept an `Idempotency-Key`. The key and a fingerprint of the body are inserted as the first statement of the transaction; the response is stored and replayed on retry. Keys expire after 24 hours.

**Reason.** The failure is mundane, not exotic: a client times out at 5 seconds, the server commits at 5.2 seconds, the client retries, and the user has two identical tasks. Mobile clients on unreliable networks make this routine. Inserting the key first lets the database's unique constraint — rather than application logic — arbitrate concurrent duplicates.

**Alternatives.** Client-generated task IDs — pushes the problem to clients and lets them choose IDs. Content-hash deduplication — breaks legitimate duplicate tasks. Nothing — accepts duplicates.

**Trade-off.** An extra table, an extra write per create, and cleanup. Small, and it eliminates a whole class of user-visible defects.

---

### D20 — A domain layer separate from ORM models

**Decision.** `domain/task.py` holds an entity with behaviour and invariants; `models.py` holds the SQLAlchemy table. Repositories map between them.

**Reason.** Business rules become testable in microseconds with no database, so the most correctness-critical code is also the cheapest to test exhaustively. It also prevents persistence concerns leaking into business logic — with ORM entities used directly, a lazy-loaded relationship inside a business method becomes a silent N+1 in production.

**Alternatives.** ORM models as domain objects — faster to write, standard in most Python codebases, and perfectly defensible for simple create, read, update, and delete (CRUD).

**Trade-off.** A mapping layer to maintain, and more code for simple operations. **This is the most debatable decision in the design.** It is justified here because the domain contains rules worth protecting — the status state machine, the ownership invariant, version-based concurrency — and those rules are exactly what needs fast, exhaustive testing. For an entity that was a bag of fields with no behaviour, I would not pay this cost.

---

### D21 — Uniform registration responses to prevent enumeration

**Decision.** `POST /auth/register` returns the same `202` and the same body whether or not the address is already registered. The difference is conveyed by email. A dummy hash equalises timing on the conflict branch.

**Reason.** Returning `409` for a taken address is a user-enumeration oracle: an attacker harvests which addresses hold accounts and uses the list for credential stuffing and phishing. The mailbox owner learns the truth; an attacker learns nothing. The dummy hash matters because Argon2id is deliberately slow, so without it the conflict branch would return measurably faster and leak exactly what the response body hides.

**Alternatives.** `409 Conflict` — better usability, leaks account existence. A CAPTCHA before registration — friction for every user, and enumeration is still possible at a slower rate.

**Trade-off.** A user who forgot they had an account gets a confusing flow. Mitigated by the email copy, which says so explicitly.

---

### D22 — MFA deferred from v1

**Decision.** No multi-factor authentication in v1. Password strength, breach-corpus rejection, dual-axis rate limiting, and refresh rotation instead.

**Reason.** MFA is substantially more than a time-based one-time password (TOTP) library: enrolment, recovery codes, device management, lost-device support, and the support load that follows. Within v1 scope the controls above address the dominant threat.

**Alternatives.** Mandatory MFA for everyone — best security, significant friction and delivery cost. Optional MFA — low adoption without enforcement, and still requires the full recovery machinery.

**Trade-off.** Materially weaker account-takeover resistance, and this is the largest open security gap in the design. **Status is "accepted with reservations": I would make MFA mandatory for administrators before general availability**, since an admin account is the highest-value target in the system. See [Security section 12](docs/SECURITY.md#12-what-i-deliberately-did-not-build).

---

### D23 — Architecture boundaries enforced in CI

**Decision.** `import-linter` contracts fail the build when the domain layer imports a framework, when modules reach past each other's `interface.py`, or when the layering is violated.

**Reason.** A modular monolith is only modular while the boundaries hold, and documented conventions erode — usually under deadline pressure, which is exactly when the boundary matters most. A red pipeline is the only architecture document that cannot be ignored.

**Alternatives.** Convention and code review — relies on every reviewer catching every violation. Physical separation into packages — heavier, and does not prevent the import anyway.

**Trade-off.** Occasional friction when a legitimate change needs a contract update. That friction is the feature: it forces the boundary change to be explicit and reviewed.

---

### D24 — RFC 9457 problem+json for errors

**Decision.** All errors use `application/problem+json` with a stable machine-readable `code`, a per-field `errors` array where relevant, and the `correlation_id` on every response.

**Reason.** A single error shape means clients write one error handler. Separating the stable `code` from the human-facing `title` lets copy be reworded without a breaking change. Including the correlation ID in the error body is the highest-leverage support decision in the design: a user pastes one string into a ticket and an engineer retrieves the entire request.

**Alternatives.** A bespoke error envelope — equivalent, without the benefit of a standard. Bare status codes — insufficient detail for clients.

**Trade-off.** Slightly more verbose responses. Negligible.

---

### D25 — Mermaid diagrams in Markdown

**Decision.** All diagrams are Mermaid embedded in Markdown, rendered natively by GitHub.

**Reason.** Diagrams stored as images drift from reality within weeks, because updating one means finding the source file, opening another application, and re-exporting — so nobody does. Mermaid is plain text, reviewed in the same pull request as the change it describes, with readable diffs. A wrong diagram is worse than no diagram, and the main defence is making updates cheap.

**Alternatives.** Draw.io or Lucidchart exports — prettier, drift immediately. PlantUML — similar benefits, needs a rendering step and does not display inline on GitHub. Structurizr — excellent for C4 at larger scale, more tooling than this warrants.

**Trade-off.** Less control over layout, and Mermaid's renderer occasionally places nodes awkwardly. Accuracy over aesthetics is the right trade for documentation that must stay current.
