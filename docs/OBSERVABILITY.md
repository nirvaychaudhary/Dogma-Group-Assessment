# Observability & Traceability

## 1. The Design Goal

Observability is not "install a logging library". The goal is a specific, testable property:

> **Given only a correlation ID from a user complaint, an engineer can reconstruct everything the system did for that request — across the application programming interface (API), the worker, and the database — in under five minutes, without adding code and without reproducing the issue.**

Everything below exists to make that sentence true. The binding mechanism is a **correlation ID generated at the edge and propagated everywhere**: every log line, every trace span, every audit row, every outbox event, and every error response body carries it.

That last one is the highest-leverage detail in this document. Because the correlation ID is in the error body, a user can paste one string into a support ticket and an engineer can retrieve the complete history of that exact request. Without it, support begins with "can you tell me roughly what time it happened?" — and most investigations never recover from that start.

---

## 2. Structured Logging

**JavaScript Object Notation (JSON) only, one event per line, no free-text interpolation.**

```json
{
  "timestamp": "2026-09-22T09:14:02.481Z",
  "level": "info",
  "event": "task.updated",
  "correlation_id": "01J8XQ2K9F3M7N4P",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "user_id": "018f0b3d-...",
  "user_role": "user",
  "service": "task-api",
  "version": "1.4.2",
  "route": "PATCH /api/v1/tasks/{task_id}",
  "task_id": "018f2c1a-...",
  "changed_fields": ["status", "priority"],
  "duration_ms": 47,
  "status_code": 200
}
```

**Why structured rather than formatted strings.** `logger.info(f"User {email} updated task {title}")` is unqueryable — finding every update by one user means a regex against terabytes — and it is a security problem, because it puts an email address and a task title, which may contain anything, into the log store. Structured fields are queryable, aggregatable, and individually redactable.

**Log injection is impossible by construction.** User input is always a JSON *value*, never concatenated into a message. An attacker cannot inject newlines to forge log entries, which is a real attack against text logs used as evidence.

### Levels, with an actual contract

| Level | Meaning | Action | Example |
|---|---|---|---|
| `DEBUG` | Developer detail | Off in production | Query parameters |
| `INFO` | Business event worth counting | None | Task created, login succeeded |
| `WARNING` | Unexpected but handled | Review in aggregate | Rate limit hit, retry succeeded |
| `ERROR` | Request failed, needs attention | Investigate | Unhandled exception, database error |
| `CRITICAL` | System-level failure | Page immediately | Cannot reach the database |

The failure mode this table prevents is log-level inflation, where everything becomes `ERROR`, the error dashboard is permanently red, and the team stops looking. `WARNING` means "handled, but I want to know if it becomes common"; `ERROR` means "a user got a bad response".

### What never goes into a log

Passwords, tokens, cookies, `Authorization` headers, task titles or descriptions, and full email addresses. Emails are masked (`a***@example.com`); Internet Protocol address (IP) addresses are truncated in the last octet; user content is referenced by ID only.

This is enforced by a redaction processor in the logging pipeline that strips known-sensitive key names, plus a rule that domain objects are never passed whole to a logger — only explicit field allowlists. Relying on developers to remember is not a control; it works until the one time someone logs an exception whose message contains the request body.

**Sampling:** all `WARNING` and above are retained; `INFO` on high-volume success paths is sampled at 10% once volume justifies it, with **all logs for a sampled trace retained together**. Partial sampling within a single request is worse than useless, because the investigation hits a gap exactly where it matters.

---

## 3. Metrics

Prometheus-compatible, scraped from `/metrics`. Cardinality is managed deliberately: route *templates* (`/api/v1/tasks/{task_id}`), never concrete paths. Putting a unique identifier (UUID) in a label name produces millions of time series and takes down the metrics backend — a self-inflicted outage caused by monitoring.

### RED metrics, per route

| Metric | Type | Labels |
|---|---|---|
| `http_requests_total` | counter | `method`, `route`, `status_class` |
| `http_request_duration_seconds` | histogram | `method`, `route` |
| `http_requests_in_flight` | gauge | `route` |

Histograms, not averages. An average latency of 120 ms is compatible with 5% of users waiting 4 seconds; only percentiles reveal that, and the 99th percentile (p99) is where users actually decide the product is broken.

### Resource and dependency metrics

| Metric | Why it matters |
|---|---|
| `db_pool_connections_in_use` / `_available` | **The first bottleneck** ([Scalability section 10](SCALABILITY.md#10-what-would-actually-break-first)). Saturation here presents as high latency with idle central processing unit (CPU), which is why it needs a dedicated metric rather than being inferred |
| `db_query_duration_seconds` | Per operation; catches a regressed plan |
| `db_replication_lag_seconds` | Correctness input for replica routing |
| `redis_operation_duration_seconds` | |
| `queue_depth` / `queue_job_duration_seconds` | Backlog detection |
| `queue_dead_letter_total` | Jobs that exhausted retries |
| `outbox_unpublished_age_seconds` | Dispatcher health; a rising value means events are stuck |
| `circuit_breaker_state` | Per dependency |

### Business and security metrics

These are what distinguish useful monitoring from infrastructure monitoring.

| Metric | Purpose |
|---|---|
| `tasks_created_total`, `tasks_completed_total` | Product health; also the fastest detector of a broken write path, because a drop to zero is unambiguous |
| `user_registrations_total` | Funnel health |
| `auth_login_failures_total` by reason | Credential-stuffing detection |
| **`authz_denials_total`** by role and route | **A primary security signal.** A spike means someone is enumerating object IDs |
| `auth_refresh_reuse_detected_total` | Token theft indicator; any non-zero value is investigated |
| `admin_cross_user_access_total` by admin | Insider-access monitoring |
| `privilege_changes_total` | Role escalation |
| `rate_limit_exceeded_total` | Abuse, or a misbehaving client |

`authz_denials_total` deserves emphasis. In normal operation this is near zero — users do not routinely request objects they do not own, because the UI does not offer them. A sudden rise is one of the earliest and clearest signals that someone is probing the system's primary risk surface, and it costs nothing to collect because the policy engine already audits every denial.

---

## 4. Distributed Tracing

OpenTelemetry, with auto-instrumentation for FastAPI, SQLAlchemy, Redis, and outbound Hypertext Transfer Protocol (HTTP), plus manual spans around business operations.

Even in a monolith, tracing earns its place: it shows the *decomposition* of a slow request. A `PATCH /tasks/{id}` taking 800 ms could be slow in policy evaluation, the load query, the update, or the audit insert, and a trace answers that in one glance rather than by adding timing logs and redeploying.

```
Trace 4bf92f35... (total 847 ms)
└── PATCH /api/v1/tasks/{task_id}                             847 ms
    ├── auth.verify_jwt                                         2 ms
    ├── ratelimit.check                              [redis]    1 ms
    ├── validate.request_body                                   1 ms
    ├── uow.begin                                     [db]      3 ms
    ├── repo.tasks.get_for_update                     [db]      8 ms
    ├── policy.authorize                                        0 ms
    ├── domain.task.apply                                       0 ms
    ├── repo.tasks.save                               [db]    790 ms   ← the answer
    ├── repo.audit.record                             [db]      6 ms
    ├── repo.outbox.add                               [db]      4 ms
    └── uow.commit                                    [db]     32 ms
```

Spans carry the Structured Query Language (SQL) *statement template* — never bound parameters, which would put user data into the tracing backend.

**Trace context is propagated into asynchronous work.** The outbox row stores the `traceparent`, and the worker continues the trace when it processes the event. Without this, the trace ends at the API response and "why did this user never get their verification email?" becomes a separate, disconnected investigation. With it, the email send is a child span of the registration request that caused it — which is exactly the link an engineer needs.

**Sampling:** 100% of errors and slow requests, 100% of security-relevant operations, 1–10% of successful requests by volume. Tail-based sampling so the decision is made after the outcome is known — head-based sampling discards the interesting traces before it knows they were interesting.

---

## 5. Audit Logging

The audit log is a **security and compliance control**, distinct from application logs, and the distinction is deliberate.

| | Application logs | Audit log |
|---|---|---|
| Purpose | Debugging | Accountability and forensics |
| Store | Log aggregation platform | PostgreSQL, partitioned |
| Written | Best effort, asynchronous | **In the same transaction as the change** |
| Mutability | Rotated and expired | **Append-only; the app role has no `UPDATE`/`DELETE`** |
| Retention | 30 days | 1 year hot, 7 years archived |

Two properties make it trustworthy.

**Transactional writes** mean a state change without a corresponding audit record is impossible, whereas a log line emitted after a commit is lost if the process dies in between — precisely when you most need it.

**Append-only privileges** mean an attacker who compromises the application cannot erase their own tracks; deleting audit history requires separate database credentials that the application does not hold.

Audited events include every authentication outcome, refresh reuse detection, all task mutations with a field-level before/after diff, **all administrator cross-user access including reads**, every privilege and status change, every authorization denial, and every data export or erasure.

**Auditing admin reads, not just writes,** is a deliberate inclusion. In a task system the sensitive administrative action is usually looking rather than changing, and an audit trail that only records mutations cannot answer the question asked after an insider incident: "did anyone read this user's tasks?"

---

## 6. Health Checks

Specified in [API Design section 5.11](API_DESIGN.md#511-health-and-operational-endpoints) and their reliability rationale in [Availability section 3](AVAILABILITY.md#3-application-instance-failure).

The short version: liveness checks nothing external, readiness checks dependencies. A liveness probe that touches the database turns a transient database fault into an orchestrator-driven crash loop across every replica.

Beyond probes, **synthetic monitoring** runs a full journey every minute from outside the virtual private cloud (VPC) — register, log in, create a task, list, update, delete — and alerts on failure or latency regression.

This catches what internal metrics cannot: Domain Name System (DNS) problems, certificate expiry, web application firewall (WAF) misconfiguration, and content delivery network (CDN) faults.

Every one of those produces a total user-facing outage while every internal dashboard stays green.

---

## 7. Alerting

**Alert on symptoms users feel, not on causes.** High CPU is not a problem if latency is fine; alerting on it trains people to ignore pages. Every alert must be actionable, have a linked runbook, and be one a human genuinely needs to see now.

| Alert | Condition | Severity |
|---|---|---|
| Error rate elevated | 5xx > 1% for 5 min | Page |
| Latency service level objective (SLO) breach | 95th percentile (p95) > 500 ms for 10 min | Page |
| Service down | Synthetic journey fails twice | Page |
| Database unreachable | Connection failures > 10 in 1 min | Page |
| **Connection pool saturated** | > 90% in use for 5 min | Page |
| Error budget burn | 2% of monthly budget in 1 hour | Page |
| **Refresh reuse detected** | Any occurrence | Page — security |
| Authorization denial spike | 10× baseline over 5 min | Page — security |
| Privilege change | Any occurrence | Notify — security |
| Replication lag | > 5 s for 5 min | Notify |
| Queue backlog | Depth > 1,000 or age > 15 min | Notify |
| Dead-letter queue | Any new job | Notify |
| Disk usage | > 80% | Notify |
| Certificate expiry | < 14 days | Notify |

**Multi-window burn-rate alerting** on the error budget: a fast-burn window catches sudden outages, a slow-burn window catches a persistent low-level error rate that would quietly consume the month's budget. A simple threshold misses the second case entirely, and the second case is the one that erodes reliability without anyone noticing.

---

## 8. Investigating a Failed or Slow Request

The concrete walkthrough this whole document exists to support.

### A user reports a failure

A user submits a ticket containing `correlation_id: 01J8XQ2K9F3M7N4P`, taken from the error message they were shown.

**Step 1 — Find every log line for the request.** One query: `correlation_id = "01J8XQ2K9F3M7N4P"`. This returns the full sequence across the API and any worker involved, including the exception with its stack trace, the route, the user, and the timing.

**Step 2 — Open the trace.** The log record carries `trace_id`. The trace shows exactly which span failed and how long each preceding step took, so the failing operation is identified without guessing.

**Step 3 — Establish scope.** Is this one user or everyone? Filter the same error signature across the window and group by route, version and user. This distinguishes "one malformed input" from "we shipped a bug 20 minutes ago", and it changes the response entirely.

**Step 4 — Correlate with changes.** Overlay deployment markers. A step change in error rate that begins at a deploy time is a rollback decision, not a debugging exercise.

**Step 5 — Confirm state.** If the request was a mutation, query the audit log by `correlation_id` to establish whether the change committed. This answers the question a user actually cares about — "did my task save?" — which logs alone cannot, because a failure after commit and a failure before commit look identical in an error log.

### A user reports slowness

Start from the latency histogram for the route to see whether the whole route regressed or only the tail.

Pull sampled slow traces for that route and look at the span breakdown; in practice the answer is nearly always one of a database query whose plan changed, connection-pool wait time, an N+1 pattern from a new relationship load, or a slow external call.

The span attribution distinguishes these immediately, and each has a different fix.

Then confirm with `pg_stat_statements` for the specific statement, check `db_pool_connections_in_use` for saturation, and compare against the deployment timeline.

**The property worth noting:** each of these steps is a *query*, not a code change. No reproduction, no added logging, no redeploy. That is the difference between a system that is monitored and one that is observable, and it is the reason the correlation ID is threaded through the audit log and the outbox rather than living only in application logs.

---

## 9. Dashboards

Four, each with one audience and one question.

| Dashboard | Question it answers | Key panels |
|---|---|---|
| **Service health** | Are users being served? | Request rate, error rate, p50/p95/p99, in-flight, error budget remaining |
| **Dependencies** | Which dependency is hurting us? | Pool utilisation, query latency, replication lag, Redis latency, circuit breaker states, queue depth |
| **Security** | Is anyone attacking us? | Login failures, authorization denials, refresh reuse, admin cross-user access, privilege changes, rate-limit rejections |
| **Business** | Is the product working? | Registrations, tasks created/completed, active users, funnel conversion |

The business dashboard is not a product-management nicety. `tasks_created_total` dropping to zero is the fastest and least ambiguous detector of a broken write path — often faster than the error-rate alert, because a write path can fail in ways that return `2xx`.

---

## 10. Cost and Retention

Observability data can cost more than the system it observes, so retention is explicit rather than default.

| Signal | Hot | Archive |
|---|---|---|
| Application logs | 30 days searchable | 90 days in object storage |
| Metrics | 15 days at full resolution | 13 months downsampled |
| Traces | 7 days | Sampled exemplars for 30 days |
| Audit log | 12 months | 7 years |

Controls that keep this affordable: sampling `INFO` on high-volume paths while retaining everything for a sampled trace, strict label-cardinality review before any new metric ships, tail-based trace sampling, and downsampling old metrics to 5-minute resolution — nobody investigates last quarter at 15-second granularity.

Audit retention is the outlier, and intentionally so: it is a compliance artefact, it is small, and it is archived to cheap storage by dropping monthly partitions.
