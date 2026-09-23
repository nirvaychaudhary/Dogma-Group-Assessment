# Availability & Resilience

Short forms are written out the first time they appear. The full list is in the [glossary](GLOSSARY.md).

## 1. The Target, and What It Permits

The service level objective (SLO) is **99.9% monthly availability**, measured as the ratio of non-5xx responses to total requests on user-facing endpoints. That allows roughly **43 minutes of error budget per month**.

Stating it as a budget rather than a slogan changes behaviour in two useful ways.

It makes 99.99% an explicit *rejection* — four nines requires multi-region active-active, and the cost and complexity of cross-region write consistency is not justified by a task manager.

And it gives a rule for release pace: when the budget is healthy, ship; when it is exhausted, the team works on reliability instead of features until it recovers. Without that rule, reliability work always loses to feature work.

| Objective | Target |
|---|---|
| Availability | 99.9% monthly |
| recovery time objective (RTO) — restore service | ≤ 30 minutes |
| recovery point objective (RPO) — maximum data loss | ≤ 5 minutes |
| Mean time to detect | ≤ 2 minutes |
| Degraded-mode availability | 99.99% for read operations |

That last row is the design philosophy in one line: **the system should almost never be completely down, even when it cannot do everything**. Partial function is worth far more than a clean failure.

---

## 2. Single Points of Failure

Honest inventory. A design that claims none is a design that has not looked.

| Component | Is it a single point of failure (SPOF)? | Mitigation | Residual risk |
|---|---|---|---|
| Application replicas | No | ≥ 2 across ≥ 2 availability zones (AZs), autoscaled, stateless | None material |
| Load balancer | No | Managed, multi-availability zone (AZ) by construction | Provider-wide failure |
| **PostgreSQL primary** | **Yes, partially** | Multi-AZ synchronous standby, 60–120 s automatic failover | **60–120 s of write unavailability on failover** |
| Redis | No, by design | The system is built to survive total Redis loss — see section 5 | Degraded rate limiting |
| Object storage | No | Regionally replicated, off the request path | None material |
| Email provider | No | Async only, retried, circuit-broken | Delayed onboarding |
| Secrets manager | No at runtime | Cached in memory with time to live (TTL) | Blocks new deployments |
| **Single region** | **Yes** | **Accepted** — see below | Full outage in a regional failure |
| **Domain Name System (DNS)** | **Yes** | Multiple providers' nameservers, long TTLs on stable records | Rare but total when it happens |
| **continuous integration and continuous delivery (CI/CD) pipeline** | Yes for changes | Documented manual break-glass deploy | Cannot ship during an outage |

### The two single points of failure (SPOFs) I am accepting, and why

**The database primary.** Multi-AZ gives an automatic failover in 60–120 seconds with zero data loss, because the standby is synchronous. Eliminating that window entirely requires either multi-master — which brings write-conflict resolution into a system that has no need for it — or an application that can serve writes without a database, which is not meaningful here. A 1–2 minute write outage during an unplanned failover consumes a few minutes of a 43-minute monthly budget. That is the right trade. Crucially, **reads continue to work during failover** because the application degrades to read-only rather than returning errors (section 4).

**Single region.** A regional failure means a full outage until we restore from cross-region snapshots, which is hours, not minutes. Multi-region active-active would cost roughly double, introduce cross-region replication lag into the authorization path, and require conflict resolution. For a task manager at 99.9%, the honest answer is that a multi-hour regional outage every few years is acceptable. What I *do* maintain is a warm-standby runbook with cross-region snapshot copies and infrastructure-as-code that can rebuild the stack elsewhere, tested annually — so the recovery path exists and is rehearsed even though it is not automatic.

Naming these explicitly is the point. An unacknowledged SPOF is an outage waiting to surprise someone; an acknowledged one is a documented business decision with a rehearsed recovery path.

---

## 3. Application Instance Failure

Stateless replicas make this the easy case: the load balancer stops routing to a failed instance, the orchestrator replaces it, and users see nothing.

The detail that actually determines whether this works is the **separation of liveness and readiness**, and it is routinely conflated.

| Probe | Checks | Failure action | Interval |
|---|---|---|---|
| `/health/live` | Process is responsive. **No dependency checks** | Kill and restart the container | 10 s, 3 failures |
| `/health/ready` | Database reachable, migrations current, Redis reachable, secrets loaded | Remove from the load balancer, keep running | 5 s, 2 failures |
| `/health/startup` | Boot initialisation complete | Delay liveness enforcement | 5 s, 30 failures |

**If liveness checked the database, a database blip would kill every replica simultaneously.** The orchestrator would restart them all, they would all fail the check again, and a 30-second database hiccup becomes a total outage with a crash-loop that outlives the original fault. Liveness answers only "is this process wedged?" Readiness answers "can this process usefully serve traffic right now?" — and its correct response is to step out of rotation, not to die.

**Graceful shutdown** on `SIGTERM`: fail readiness immediately, keep liveness passing, continue serving in-flight requests for up to 25 seconds, close database and Redis pools, exit. The load balancer's draining period is 30 seconds, deliberately longer, so traffic stops arriving before the process stops accepting it. Without this ordering, every deployment drops a handful of in-flight requests — invisible on a dashboard, quietly annoying to users.

**Deployment** is rolling with surge, one batch at a time, gated on health checks with automatic rollback if the error rate rises above baseline within a 10-minute bake. Both old and new versions run simultaneously during the roll, which is exactly why migrations must be backward compatible ([Data Architecture section 10](DATA_ARCHITECTURE.md#10-migrations)).

---

## 4. Database Failure

| Failure | Detection | Response | Impact |
|---|---|---|---|
| Primary instance fails | Managed health check, ~30 s | Automatic failover to the synchronous standby | 60–120 s of write unavailability, no data loss |
| AZ failure | Managed | Failover to the standby in the surviving AZ | Same as above |
| Connection pool exhausted | Pool-saturation metric | Shed load, alert, scale PgBouncer | Elevated latency |
| Slow queries saturating the primary | `pg_stat_statements`, latency alerts | Kill long queries, statement timeout as backstop | Degraded latency |
| **Logical corruption from a bad migration** | Data validation, user reports | **Point-in-time recovery** | Minutes to hours; the worst realistic case |
| Total regional loss | Multiple alarms | Cross-region restore per runbook | Hours |

### Read-only degraded mode

This is the most valuable resilience feature in the design, and it is cheap to build.

When the application detects that writes are failing but reads succeed — during a failover, or a primary in trouble — it enters read-only mode instead of returning `500` to everything:

- `GET` requests continue to be served normally from the standby or a replica.
- Write requests return `503` with `Retry-After` and a clear `SERVICE_READ_ONLY` code.
- A banner flag in the response tells clients to surface "temporarily read-only" rather than a generic error.
- Recovery is automatic once write probes succeed.

Users can still *see* their tasks during a database failover. For a task manager, reading is the dominant use — most sessions are checking what is due, not creating.

Turning a 90-second total outage into a 90-second read-only window is a large user-visible improvement for a modest amount of code, and it is only possible because read and write paths were separated deliberately rather than incidentally.

**Logical corruption is the failure I plan for hardest**, because Multi-AZ does not help — a bad migration replicates to the standby instantly and faithfully. The controls are preventive (reviewed, backward-compatible, batch-tested migrations), detective (post-deploy data validation), and corrective (point-in-time recovery (PITR) with quarterly, timed restore drills). See [Data Architecture section 9](DATA_ARCHITECTURE.md#9-backup-and-recovery).

---

## 5. Redis Failure

**Redis holds no source-of-truth data, and total loss is survivable by design.** This constraint shaped what Redis is allowed to store, which is why it is listed as "no, by design" in the SPOF table rather than mitigated after the fact.

| Redis function | Behaviour when unavailable | Rationale |
|---|---|---|
| Rate-limit counters | **Fail open**, alert | Failing closed would reject all traffic — a cache outage becoming a total outage |
| Token denylist | **Fail open**, alert | See [Security section 4](SECURITY.md#4-token-revocation): bounded 15-minute exposure for explicitly revoked tokens, versus logging out every user |
| Job queue | Enqueue fails; outbox rows remain unpublished and are retried | The outbox is in PostgreSQL, so no event is lost |
| Cached lookups | Fall through to the source | Slower, still correct |

The two fail-open decisions are deliberate and narrow, and I want to be explicit that they are security trade-offs rather than oversights.

Both are bounded, both are alerted, and both are backstopped by another layer: the web application firewall (WAF) still enforces coarse rate limits at the edge, and the `token_version` check still runs against PostgreSQL.

Every *authorization* decision continues to fail closed.

Redis runs Multi-AZ with automatic failover and AOF persistence, so total loss is unlikely; the design simply does not depend on that being true.

---

## 6. External Service Failure

The only synchronous-path external dependency is the secrets manager at boot. Everything else is asynchronous by design, which is itself the primary resilience control.

**Email provider outage:** jobs retry with exponential backoff and full jitter across roughly 24 hours, a circuit breaker stops hammering a provider that is clearly down, and permanently failed messages land in a dead-letter queue for inspection and replay. Registration still returns `202` — the account exists, and the verification email arrives when the provider recovers. Core task operations are entirely unaffected. A second provider behind the same adapter interface is a Stage 2 addition if email reliability becomes a real problem.

**Secrets manager outage:** secrets are fetched at boot and cached in memory with a TTL, so running instances are unaffected. Only new deployments and scale-out events are blocked, which is acceptable for a control-plane service.

### Circuit breakers, used sparingly

A circuit breaker guards every outbound call: after 5 failures in 30 seconds the circuit opens for 60 seconds, then allows a single probe request before closing.

This prevents a slow dependency from consuming the worker pool — the classic cascading failure, where a 30-second timeout on a dead provider ties up every worker thread and takes down functionality that has nothing to do with email.

I have deliberately *not* wrapped internal calls in circuit breakers. Inside a monolith there is no network hop to protect, and a breaker between a service and its own repository would add a failure mode rather than remove one.

---

## 7. Network Failure

| Failure | Mitigation |
|---|---|
| Transient packet loss | TCP retransmission; connection-level retries with backoff |
| Slow network to the database | Connection and statement timeouts; pool health checks evict dead connections |
| Cross-AZ partition | Multi-AZ deployment; managed services handle quorum |
| DNS resolution failure | Short TTLs for failover records, longer for stable ones; multiple nameserver providers |
| Client network drop mid-write | Idempotency keys make the retry safe |

That last row matters more than it looks. A user on a mobile connection who loses signal after the server commits will retry, and without idempotency keys they get a duplicate task. Network failures are the everyday reason idempotency exists — not exotic distributed-systems scenarios.

---

## 8. Timeouts and Retries

### Timeout budget

Every timeout must be strictly tighter than the layer outside it. If an inner call can outlive its caller, the caller's resources are held hostage by something it has already given up on.

| Layer | Timeout |
|---|---|
| Client | 10 s |
| Load balancer idle | 30 s |
| Application request | 8 s |
| Database statement | 5 s |
| Database connection acquire | 2 s |
| Redis operation | 250 ms |
| External Hypertext Transfer Protocol (HTTP) | 3 s connect, 5 s total |

The 5-second `statement_timeout` is the most important entry: it is an absolute backstop against a pathological query pinning a connection indefinitely. Without it, one bad query plan can exhaust the pool and take the service down.

### Retry policy

Retries are only applied to **idempotent operations** experiencing **transient** failures: connection errors, serialisation failures, deadlocks, `503`, `429` with `Retry-After`. Never on `4xx` validation errors — the request will fail identically forever — and never on a non-idempotent write without an idempotency key.

Exponential backoff with **full jitter**, capped at 3 attempts. Jitter is not optional: synchronised retries from thousands of clients produce a thundering herd that re-kills a service the moment it recovers, and fixed-interval backoff guarantees synchronisation.

**A retry budget caps retries at 10% of total requests.** Without it, retries amplify load exactly when the system is least able to absorb it, and a partial outage becomes a self-inflicted total one. This is the single most commonly missing piece of retry logic.

---

## 9. Graceful Degradation

The ordered list of what to sacrifice, decided in advance so the decision is not made under pressure.

| Priority | Capability | Degrades to |
|---|---|---|
| 1 | Authentication | Never sacrificed — without it nothing else is safe |
| 2 | Reading tasks | Served from a replica if the primary is unavailable |
| 3 | Writing tasks | `503` + `Retry-After` in read-only mode |
| 4 | Admin operations | Shed before user operations |
| 5 | Search | Falls back to simple filtering if a search index is down |
| 6 | Email notifications | Queued and delayed |
| 7 | Analytics and exports | Disabled first |

**Feature flags** allow any non-core capability to be switched off without a deploy, which matters because during an incident a deploy is the slowest and riskiest tool available. A kill switch that takes effect in seconds is worth a great deal at 3 a.m.

---

## 10. Recovery and Operational Practice

| Scenario | Procedure | RTO |
|---|---|---|
| Instance failure | Automatic replacement | < 2 min |
| AZ failure | Automatic failover | < 2 min |
| Bad deployment | Automatic rollback on error-rate breach | < 5 min |
| Data corruption | PITR to just before the event | 30–60 min |
| Regional failure | Cross-region restore from snapshots per runbook | 2–4 hours |
| Security incident | Revoke tokens, rotate keys, suspend accounts | < 15 min to contain |

Every scenario has a **runbook** with detection signals, immediate containment, diagnostics, recovery steps, verification and communication. Runbooks are linked directly from the alerts that fire, because an alert at 3 a.m. that does not say what to do is only half a control.

**Chaos testing in staging, quarterly:** kill a replica during load, force a database failover, block Redis, make the email provider return errors, saturate the connection pool. Each exercise verifies the behaviour claimed in this document. Resilience that has never been exercised is a hypothesis — and the value of these drills is less in proving the system works than in discovering the specific ways it does not, while nobody is watching.
