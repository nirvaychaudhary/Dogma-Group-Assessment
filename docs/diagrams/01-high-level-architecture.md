# Diagram 1 — High-Level Architecture

The system decomposed into layers, showing every component and the direction of data and control flow. Discussed in [HLD §4–§5](../HLD.md#4-logical-architecture).

```mermaid
flowchart TB
    subgraph CLIENTS["CLIENT LAYER"]
        SPA["Web SPA<br/>access token in memory<br/>refresh in HttpOnly cookie"]
        MOB["Mobile App<br/>tokens in OS keystore"]
        SVC_CLI["Service clients<br/>CI, integrations"]
    end

    subgraph EDGE["EDGE LAYER"]
        CDN["CDN<br/>static assets, TLS 1.3"]
        WAF["WAF<br/>OWASP CRS, IP reputation,<br/>coarse rate limits, L7 DDoS"]
        LB["Load Balancer<br/>multi-AZ, least-outstanding-requests,<br/>health checks, 30s drain"]
    end

    subgraph APP["APPLICATION LAYER — stateless, 2 to 10 replicas across 2+ AZs"]
        direction TB

        subgraph MIDDLEWARE["Middleware pipeline — ordered"]
            M1["1. Correlation ID"]
            M2["2. Body size guard"]
            M3["3. IP rate limit — pre-auth"]
            M4["4. Authenticate JWT"]
            M5["5. Identity rate limit"]
            M6["6. Access log + metrics"]
            M1 --> M2 --> M3 --> M4 --> M5 --> M6
        end

        subgraph ROUTERS["API layer — FastAPI"]
            R_AUTH["/api/v1/auth"]
            R_USER["/api/v1/users"]
            R_TASK["/api/v1/tasks"]
            R_ADMIN["/api/v1/admin<br/>namespace-wide admin guard"]
            R_HLTH["/health, /metrics"]
        end

        POLICY["AUTHORIZATION POLICY ENGINE<br/>single decision point<br/>can principal do action on resource<br/>raises, never returns a boolean"]

        subgraph SERVICES["Application services — transaction boundaries"]
            S_ID["Identity service"]
            S_TSK["Task service"]
            S_ADM["Administration service"]
            S_AUD["Audit service"]
        end

        subgraph DOMAIN["Domain layer — pure, no I/O"]
            D_USER["User entity<br/>invariants"]
            D_TASK["Task entity<br/>status state machine"]
        end

        subgraph DATAACCESS["Data access"]
            UOW["Unit of Work<br/>one transaction per use case"]
            REPOS["Repositories<br/>owner-scoped queries<br/>version-guarded updates"]
        end

        MIDDLEWARE --> ROUTERS
        ROUTERS --> POLICY
        POLICY --> SERVICES
        SERVICES --> DOMAIN
        SERVICES --> UOW
        UOW --> REPOS
    end

    subgraph ASYNC["ASYNCHRONOUS PROCESSING"]
        OUTBOX["Outbox dispatcher<br/>polls unpublished events<br/>carries traceparent"]
        QUEUE["Job queue<br/>Redis + arq"]
        WORKERS["Worker pool<br/>separate process, shared codebase"]
        SCHED["Leader-elected scheduler<br/>advisory lock, never per-replica"]
    end

    subgraph DATA["DATA LAYER"]
        PG[("PostgreSQL 16 — PRIMARY<br/>Multi-AZ synchronous standby<br/>users · tasks · audit_log<br/>outbox · idempotency · tokens")]
        STANDBY[("Synchronous standby<br/>automatic failover 60-120s")]
        REPLICA[("Read replica<br/>Stage 2 — analytics and audit only")]
        REDIS[("Redis — Multi-AZ<br/>rate limits · token denylist<br/>job queue · hot lookups<br/>NOT a source of truth")]
        S3[("Object storage<br/>WAL archive · snapshots<br/>archived audit partitions")]
    end

    subgraph EXT["EXTERNAL SERVICES"]
        EMAIL["Transactional email<br/>behind an adapter port"]
        SECRETS["Secrets manager<br/>cached in memory with TTL"]
        KMS["KMS<br/>encryption keys"]
    end

    subgraph OBSERVE["OBSERVABILITY"]
        OTEL["OpenTelemetry Collector"]
        LOGS["Log aggregation<br/>structured JSON, redacted"]
        METRICS["Metrics + dashboards<br/>RED · USE · security · business"]
        TRACES["Distributed tracing<br/>tail-based sampling"]
        ALERTS["Alerting → on-call<br/>runbook linked"]
        SYNTH["Synthetic monitoring<br/>full journey, every minute"]
    end

    SPA --> CDN
    MOB --> WAF
    SVC_CLI --> WAF
    CDN --> WAF
    WAF --> LB
    LB --> MIDDLEWARE

    M3 -.->|"counters"| REDIS
    M4 -.->|"denylist + JWKS"| REDIS
    M5 -.->|"counters"| REDIS

    REPOS --> PG
    REPOS -.->|"explicitly routed<br/>read-only queries"| REPLICA
    SERVICES -.->|"outbox row in the<br/>same transaction"| PG

    PG -->|"synchronous replication"| STANDBY
    PG -->|"asynchronous replication"| REPLICA
    PG -->|"continuous WAL archive"| S3

    PG --> OUTBOX
    OUTBOX --> QUEUE
    QUEUE --> WORKERS
    SCHED --> QUEUE
    WORKERS --> PG
    WORKERS --> EMAIL
    WORKERS --> S3

    APP --> SECRETS
    APP --> KMS
    APP --> OTEL
    WORKERS --> OTEL
    OTEL --> LOGS
    OTEL --> METRICS
    OTEL --> TRACES
    METRICS --> ALERTS
    LOGS --> ALERTS
    SYNTH --> ALERTS
    SYNTH -.->|"probes from outside the VPC"| CDN
```

## Reading the diagram

**The vertical axis is trust.** Everything above the application layer is untrusted input. The middleware pipeline is the boundary, and its ordering is deliberate — cheap, security-critical checks run before expensive ones, so a hostile request is rejected before it can consume signature-verification CPU.

**The policy engine sits between routing and services**, and every path through the application crosses it. It is drawn as a distinct box rather than folded into the services because object-level authorization is the system's primary risk and the design treats it as a component with an owner, not a convention.

**Redis is reachable only from the middleware and the queue.** It never appears on a path that produces an authoritative answer, which is what makes total Redis loss survivable.

**The outbox arrow points from PostgreSQL to the dispatcher, not from the services to the queue.** That is the transactional outbox pattern in one detail: services write events to the database inside the business transaction, and the dispatcher publishes them afterwards. There is no path by which a service writes directly to the queue, so a rolled-back transaction can never emit an event.

**Read replica arrows are dashed and labelled "explicitly routed."** Replica reads are a per-query decision, never a default, because a user's own data and all authorization lookups must come from the primary.

---

**Related:** [HLD §4 Logical Architecture](../HLD.md#4-logical-architecture) · [HLD §5 Component Responsibilities](../HLD.md#5-component-responsibilities) · [HLD §6 Request Lifecycle](../HLD.md#6-request-lifecycle)
