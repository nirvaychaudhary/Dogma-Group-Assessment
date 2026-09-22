# Assumptions, Risks & Trade-offs

An architecture is only meaningful relative to what it assumes and what it accepts. This document states both, and is deliberately the least flattering document in the set.

---

## 1. Assumptions

Assumptions are ordered by how much of the design collapses if they are wrong.

### Load-bearing assumptions

| # | Assumption | What breaks if it is wrong |
|---|---|---|
| A1 | **A task has exactly one owner.** No sharing, no teams, no collaborators | The entire authorization model. Ownership becomes an ACL, and `WHERE owner_id = :principal` — the core query-scoping control — stops working. **This is the largest latent change in the design** |
| A2 | **Two roles are sufficient.** `user` and `admin`, with no hierarchy | The policy engine needs a permission model rather than a role map. Contained, but not trivial |
| A3 | **Scale stays within the [HLD §2](HLD.md#2-context-constraints-and-sizing-assumptions) baseline** — 50k users, 150 req/s, 2M tasks | The launch topology. Stages 1–2 of the [evolution path](SCALABILITY.md#2-the-evolution-path) absorb 10×, so this degrades gracefully |
| A4 | **Single region is acceptable** | Availability and disaster-recovery design. A data-residency requirement would force a material redesign |
| A5 | **Tasks contain ordinary business text**, not regulated data | Encryption and compliance posture. Health or payment data would require field-level encryption and a different audit regime |

### Supporting assumptions

| # | Assumption | Note |
|---|---|---|
| A6 | English-language, UTC-normalised; clients handle local presentation | Internationalisation is presentation-layer |
| A7 | Clients are a web SPA and a mobile app that we control | Shapes the token-storage decision |
| A8 | No file attachments in v1 | Object storage, virus scanning and signed URLs are absent |
| A9 | Email delivery is reliable enough for verification | Registration depends on it; SMS fallback would be needed otherwise |
| A10 | A team of 2–4 backend engineers | Drives the monolith and managed-services choices throughout |
| A11 | Users tolerate 15-minute access-token expiry with silent refresh | Standard practice |
| A12 | 30-day soft-delete retention meets user expectations | Configurable if wrong |

### If A1 is wrong

A1 is called out because it is the assumption most likely to change and the most expensive if it does. Product requirements for task management reach "share this with a colleague" quickly.

If sharing arrives, the migration is: a `task_permissions` table, ownership checks replaced by permission lookups in `security/policy.py`, and every scoped query rewritten to join against it. This is contained *because* the ownership check is centralised in the policy engine and the repository rather than scattered across handlers — which is a significant part of why it was centralised. It remains a multi-week change touching the most sensitive code in the system, so I would want the requirement confirmed before building anything speculative for it.

---

## 2. Risk Register

Scored as likelihood × impact. Only risks I consider real are listed; a register padded with implausible entries hides the ones that matter.

| # | Risk | L | I | Score | Mitigation | Residual |
|---|---|---|---|---|---|---|
| R1 | **Broken object-level authorization** leaks data across users | Med | Critical | **High** | Centralised policy engine, query-level scoping, `404` responses, generated authorization matrix, mutation testing, two-reviewer rule on `security/`, penetration test before GA | Low |
| R2 | **Account takeover via credential stuffing** | High | High | **High** | Argon2id, breach-corpus rejection, dual-axis rate limiting, uniform error responses, anomaly alerting | Med — **MFA is the real fix and is deferred** |
| R3 | **A bad migration corrupts data** | Med | High | **High** | Backward-compatible migrations, migration-safety linter, staging rehearsal with production-shaped data, PITR, quarterly restore drills | Low |
| R4 | **Database connection exhaustion** under scale-out | High | Med | **High** | Pool sizing arithmetic, saturation metric and alert from day one, PgBouncer at Stage 1, load shedding | Low |
| R5 | Malicious or compromised admin extracts data | Low | Critical | Med | Read auditing, self-modification prohibited, export thresholds, volume-anomaly alerting, last-admin protection | Med — **mostly detective, not preventive** |
| R6 | A regional outage takes the system down | Low | High | Med | Multi-AZ, cross-region snapshots, rehearsed rebuild runbook | **Accepted**: hours of downtime in a rare event |
| R7 | An unindexed query degrades as data grows | High | Med | Med | Load testing with realistic volumes, `pg_stat_statements`, slow-query alerting, the "will this work at 50M rows?" review question | Low |
| R8 | Supply-chain compromise via a dependency | Low | High | Med | Hash-pinned lockfile, `pip-audit`, Trivy, SBOM, dependency review | Med — an irreducible ecosystem risk |
| R9 | Refresh-token theft grants 30 days of access | Low | High | Med | Rotation with reuse detection, `HttpOnly` cookies, family revocation, user notification | Low |
| R10 | Email provider outage blocks all onboarding | Med | Med | Med | Retries with backoff, circuit breaker, dead-letter queue; core operations unaffected | Med — single provider in v1 |
| R11 | Audit log growth degrades write performance | Med | Med | Med | Monthly partitioning, archival, partial indexes | Low |
| R12 | Redis loss degrades security controls | Low | Med | Low | Documented fail-open behaviour, edge rate limiting as a backstop, Multi-AZ Redis | Low — **a conscious trade** |
| R13 | Modular boundaries erode over time | Med | Med | Med | `import-linter` contracts as a build gate | Low |
| R14 | Team turnover loses architectural context | Med | Med | Med | This documentation set, the decision log, runbooks | Low |

### The three I would actually lose sleep over

**R1** is the risk the whole design is organised around. It is high-likelihood because it is an easy mistake — one endpoint, one forgotten check — and critical-impact because it is a data breach. It is also the hardest to detect, since a successful exploit looks like a normal `200`. Hence four independent layers of mitigation plus generated tests plus mutation testing plus a review rule, and hence `authz_denials_total` being a paged security metric.

**R2** is high-likelihood because it requires no skill: credential-stuffing lists are commodity and the attack is automated. The mitigations are good but not sufficient, and I want to be explicit that **the honest fix is MFA**, which I deferred on cost grounds and would make mandatory for administrators before general availability. Deferring it is a decision, not an oversight.

**R4** is worth naming because it is the failure that *looks like* something else. Every dashboard is green — CPU low, queries fast — while requests queue for a connection. Teams respond by adding replicas, which makes it strictly worse. It is called out in [Scalability §10](SCALABILITY.md#10-what-would-actually-break-first) and given a dedicated metric and alert precisely so it is diagnosed rather than discovered.

---

## 3. Bottlenecks

Ranked by when they are likely to be hit, as detailed in [Scalability §10](SCALABILITY.md#10-what-would-actually-break-first).

| Rank | Bottleneck | Arrives at | Warning signal |
|---|---|---|---|
| 1 | Database connections | ~8–10 replicas | Pool utilisation > 70% |
| 2 | Unindexed queries | Data growth, not traffic | A single endpoint's p95 diverging |
| 3 | Autovacuum lag | Sustained high write rate | Table and index bloat |
| 4 | Redis contention | High rate-limit volume | Redis operation latency |
| 5 | Worker backlog | Provider outage | Queue depth, outbox age |
| 6 | Audit write amplification | ~50M audit rows | Write latency rising across all mutations |
| 7 | Primary write saturation | Well beyond the sizing baseline | Sustained write latency |

Each has a monitored leading indicator, which is the point of enumerating them. A bottleneck you have predicted is a capacity-planning task; one you have not is an incident.

---

## 4. Single Points of Failure

Full analysis in [Availability §2](AVAILABILITY.md#2-single-points-of-failure). Summary:

| SPOF | Status |
|---|---|
| Database primary | Mitigated to a 60–120 s write-unavailability window; reads continue in degraded mode |
| Single region | **Accepted** — documented, with a rehearsed rebuild runbook |
| DNS | Mitigated with multiple nameserver providers |
| CI/CD | Mitigated with a documented manual deploy path |
| Application replicas, Redis, object storage, email | Not SPOFs by design |

---

## 5. Security Risks

Covered in [Security §1](SECURITY.md#1-threat-model-first) and §12. The gaps I am consciously carrying:

| Gap | Why | Closing trigger |
|---|---|---|
| **No MFA** | Cost of enrolment, recovery and support in v1 | **Mandatory for admins before GA**; optional for users shortly after |
| Insider risk is detective, not preventive | Preventive controls (approval workflows, break-glass) need an operational maturity that does not exist yet | Compliance requirement, or team growth |
| No penetration test | Cannot test a design | **Before GA**, scoped at authorization boundaries |
| Anomaly detection is threshold-based | Behavioural baselines need production traffic | ~3 months of data |
| Single email provider | Simplicity | If delivery reliability becomes a measured problem |
| Fail-open rate limiting and denylist | Avoids a cache outage becoming a total outage | Reconsider if Redis proves unreliable |

---

## 6. Principal Trade-offs

Every one of these is a real cost accepted for a real benefit. Listing only the benefits would make this a sales document.

| Trade-off | Gained | Paid |
|---|---|---|
| **Modular monolith over microservices** | Operational simplicity, transactional integrity, fast delivery | Coupled deployment; one module cannot scale alone |
| **PostgreSQL only** | ACID, constraints, one operational surface | Not optimal for full-text search or very high write throughput |
| **JWT access tokens** | No database round trip on the hot path | Revocation is not instant; bounded to 15 minutes plus a denylist |
| **Opaque refresh tokens** | Immediate revocation, reuse detection | A database lookup on refresh, and rotation races needing a grace window |
| **Separate `/admin` namespace** | Auditable, reviewable privilege boundary | Some duplication between user and admin handlers |
| **`404` instead of `403`** | No existence oracle | Worse developer experience; harder to debug legitimate permission problems |
| **Uniform registration responses** | No user enumeration | A confusing flow for users who forgot they had an account |
| **No caching of task lists** | No invalidation complexity, no cross-user leakage risk | Every read hits the database |
| **Keyset pagination** | Constant-time deep pages, stable under writes | No random page access, no cheap total count |
| **Soft delete** | Recoverable from accidental deletion | Every query carries a filter; unique constraints must be partial; a separate hard-delete path for GDPR |
| **Optimistic concurrency** | No locks held across user think-time | Clients must handle `409`/`412` |
| **Domain layer separate from ORM models** | Fast, pure tests of the rules that matter | A mapping layer to maintain — **the most debatable choice in the design** |
| **Transactional outbox** | No dual-write inconsistency | An extra table, a dispatcher, and at-least-once semantics for consumers to absorb |
| **Fail-open rate limiting** | A Redis outage does not become a total outage | A window of unlimited requests, backstopped only at the edge |
| **Single region** | Half the cost, far less complexity | Hours of downtime in a regional failure |
| **No MFA in v1** | Faster delivery, simpler support | Meaningfully weaker account-takeover resistance |

---

## 7. What I Would Do Next

In priority order, and with reasons, because the sequence is itself a judgement.

| Priority | Action | Why now |
|---|---|---|
| 1 | **MFA, mandatory for admins** | The largest open security gap; an admin account is the highest-value target |
| 2 | **Penetration test scoped at authorization** | Verifies the controls for R1, the dominant risk, against an adversary who was not involved in the design |
| 3 | **Load test with production-shaped data** | Confirms the bottleneck prediction is right. If the first failure is not connection exhaustion, the model is wrong and so is the scaling plan |
| 4 | **Rehearse a PITR restore, timed** | An untested backup is a hypothesis. This is the only control for R3's worst case |
| 5 | Behavioural anomaly detection | Converts insider risk from threshold alerting to real detection |
| 6 | Second email provider | Removes the last meaningful single-provider dependency |
| 7 | Confirm or reject the sharing requirement (A1) | Determines whether the authorization model needs to change before it calcifies |

Items 1–4 are all *verification* rather than construction. At this stage that is the right emphasis: the design's claims are only worth what they can be demonstrated to be true, and the cheapest time to find out that one of them is wrong is now.
