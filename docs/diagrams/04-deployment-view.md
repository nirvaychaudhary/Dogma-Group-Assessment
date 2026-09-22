# Diagram 4 — Deployment & Infrastructure View

Physical topology, network segmentation, and the deployment pipeline. Reference implementation is AWS; §4.5 gives the equivalents on other providers. Discussion in [HLD §11](../HLD.md#11-infrastructure-view).

---

## 4.1 Production Topology

```mermaid
flowchart TB
    subgraph INTERNET["Internet"]
        USERS["Users"]
        ATTACK["Automated scanners<br/>and bots"]
    end

    subgraph GLOBAL["Global edge"]
        R53["Route 53<br/>health-checked DNS<br/>multiple nameserver providers"]
        CF["CloudFront<br/>TLS 1.3 termination<br/>static asset caching"]
        WAF["AWS WAF<br/>OWASP CRS · rate rules<br/>IP reputation · bot control"]
        SHIELD["Shield<br/>L3/L4 DDoS absorption"]
    end

    subgraph REGION["Region — eu-west-1"]
        subgraph VPC["VPC 10.0.0.0/16"]

            subgraph PUBLIC["Public subnets — 10.0.0.0/20"]
                ALB["Application Load Balancer<br/>TLS re-encryption to targets<br/>health checks on /health/ready<br/>30s connection draining"]
                NAT["NAT Gateway<br/>egress only"]
            end

            subgraph PRIVATE_APP["Private app subnets — 10.0.16.0/20 · no inbound internet"]
                subgraph AZ_A["Availability Zone A"]
                    API_A1["ECS Fargate<br/>task-api"]
                    API_A2["ECS Fargate<br/>task-api"]
                    WRK_A["ECS Fargate<br/>worker"]
                end
                subgraph AZ_B["Availability Zone B"]
                    API_B1["ECS Fargate<br/>task-api"]
                    API_B2["ECS Fargate<br/>task-api"]
                    WRK_B["ECS Fargate<br/>worker"]
                end
                PGB["PgBouncer<br/>transaction mode<br/>added at Stage 1"]
            end

            subgraph PRIVATE_DATA["Private data subnets — 10.0.32.0/20 · no internet route at all"]
                RDS_A[("RDS PostgreSQL 16<br/>PRIMARY — AZ A<br/>encrypted · PITR · 5 min RPO")]
                RDS_B[("Synchronous standby<br/>AZ B — automatic failover")]
                RDS_R[("Read replica<br/>Stage 2")]
                REDIS_A[("ElastiCache Redis<br/>PRIMARY — AZ A")]
                REDIS_B[("Redis replica<br/>AZ B — automatic failover")]
            end

            subgraph ENDPOINTS["VPC endpoints — traffic never leaves the AWS network"]
                VPE_S3["S3 gateway endpoint"]
                VPE_SM["Secrets Manager"]
                VPE_ECR["ECR"]
                VPE_LOG["CloudWatch Logs"]
            end
        end

        subgraph MANAGED["Regional managed services"]
            S3B[("S3<br/>backups · WAL archive<br/>archived audit partitions<br/>versioned + object lock")]
            SM["Secrets Manager<br/>automatic rotation"]
            KMS["KMS<br/>customer-managed keys"]
            ECR["ECR<br/>image scanning<br/>immutable tags"]
            CW["CloudWatch<br/>logs · metrics · alarms"]
        end
    end

    subgraph DR["Disaster recovery — second region"]
        S3DR[("Cross-region snapshot copies<br/>90 day retention")]
        IAC["Terraform state<br/>rebuild the stack from code"]
    end

    subgraph EXTERNAL["External SaaS"]
        MAIL["Email provider"]
        OBSV["Observability platform<br/>OTLP ingest"]
        PAGER["On-call paging"]
    end

    USERS --> R53 --> CF
    ATTACK -.->|"blocked at the edge"| SHIELD
    CF --> SHIELD --> WAF --> ALB

    ALB --> API_A1 & API_A2 & API_B1 & API_B2

    API_A1 & API_A2 & API_B1 & API_B2 --> PGB
    PGB --> RDS_A
    RDS_A -.->|"synchronous"| RDS_B
    RDS_A -.->|"asynchronous"| RDS_R

    API_A1 & API_B1 --> REDIS_A
    REDIS_A -.->|"replication"| REDIS_B

    WRK_A & WRK_B --> RDS_A
    WRK_A & WRK_B --> REDIS_A
    WRK_A & WRK_B --> NAT --> MAIL

    RDS_A -->|"continuous WAL + snapshots"| S3B
    S3B -->|"daily copy"| S3DR

    API_A1 --> VPE_SM --> SM
    API_A1 --> VPE_LOG --> CW
    WRK_A --> VPE_S3 --> S3B
    SM --> KMS
    RDS_A --> KMS
    S3B --> KMS

    CW --> PAGER
    API_A1 & WRK_A --> OBSV
```

---

## 4.2 Network Segmentation

Three tiers, each unable to reach further than it must. This is the control that limits blast radius when — not if — something in the application is compromised.

| Tier | Inbound | Outbound | Rationale |
|---|---|---|---|
| Public subnets | 443 from the internet, via WAF only | To private app subnets | Only the load balancer is exposed |
| Private app subnets | From the ALB security group only | To data subnets, plus internet via NAT | **No inbound internet route.** A compromised container cannot be reached directly |
| Private data subnets | 5432/6379 from the app security group only | **No internet route at all** | Even with credentials and code execution, data cannot be exfiltrated directly to the internet |

Security groups reference **other security groups, not CIDR ranges**, so rules stay correct as instances come and go — a CIDR-based rule silently grants access to whatever later occupies that address space.

VPC endpoints keep traffic to S3, Secrets Manager, ECR and CloudWatch inside the AWS network, so it never traverses the NAT gateway or the public internet. That improves security posture and removes NAT data-transfer cost at the same time.

---

## 4.3 Deployment Pipeline

```mermaid
flowchart TB
    DEV["Developer pushes a branch"] --> PR["Pull request opened"]

    subgraph CI["CI — target under 10 minutes to a merge decision"]
        direction TB
        FAST["Fast gates ~30s<br/>ruff format · ruff · mypy --strict<br/>import-linter · gitleaks"]
        UNIT["Unit tests ~3s<br/>domain + policy, no I/O"]
        INTEG["Integration tests ~2m<br/>real Postgres via Testcontainers"]
        APITEST["API + security suites ~2m<br/>including the authorization matrix"]
        SCAN["pip-audit · bandit · semgrep"]
        BUILD["Build image<br/>multi-stage · non-root · distroless<br/>+ Trivy scan + SBOM"]
        FAST --> UNIT --> INTEG --> APITEST --> SCAN --> BUILD
    end

    PR --> CI
    CI --> REVIEW["Review<br/>2 reviewers for security/,<br/>migrations, or admin routes"]
    REVIEW --> MERGE["Squash-merge to main"]

    MERGE --> PUSH["Push image to ECR<br/>tagged by immutable digest"]
    PUSH --> MIG_S["Run migrations — staging<br/>backward compatible, lock_timeout set"]
    MIG_S --> DEP_S["Deploy staging"]
    DEP_S --> SMOKE["Smoke + contract tests<br/>+ synthetic journey"]

    SMOKE --> GATE{"All green?"}
    GATE -->|"No"| STOP["Halt · notify"]
    GATE -->|"Yes"| MIG_P["Run migrations — production<br/>separate pipeline step, separate DB role"]

    MIG_P --> ROLL["Rolling deploy<br/>50% surge · one batch at a time<br/>gated on health checks"]
    ROLL --> BAKE["Bake 10 minutes<br/>watch error rate and p95"]
    BAKE --> OK{"Within<br/>baseline?"}
    OK -->|"No"| RB["Automatic rollback<br/>to the previous digest"]
    OK -->|"Yes"| DONE["Release complete<br/>deployment marker to dashboards"]
```

Three properties of this pipeline matter more than its shape:

**The same image digest is promoted from staging to production.** It is never rebuilt per environment. The artefact that passed the tests is bit-identical to the one released; only configuration differs.

**Migrations are a separate step, run before the application roll.** During a rolling deploy, old and new code execute simultaneously against the same schema, which is exactly why every migration must be backward compatible and why column removals span three releases ([Data Architecture §10](../DATA_ARCHITECTURE.md#10-migrations)).

**Rollback never requires a database rollback.** Because migrations are expand-only within a release, reverting the application to the previous digest is always safe. A deployment strategy whose rollback path depends on reversing a migration is not a rollback path at all.

---

## 4.4 Environment Sizing

| | Local | CI | Staging | Production |
|---|---|---|---|---|
| Compute | Docker Compose | Ephemeral containers | 1 task, 0.5 vCPU | 2–10 tasks, 1 vCPU / 2 GB |
| Database | Container | Testcontainers | db.t4g.medium, single AZ | db.r6g.large, **Multi-AZ** |
| Redis | Container | Container | cache.t4g.micro | cache.r6g.large, Multi-AZ |
| Data | Seeded fixtures | Ephemeral | Anonymised subset | Real |
| Backups | None | None | Daily | **Daily + continuous WAL** |
| Access | Developer | Pipeline | Team | **Break-glass, audited** |

Staging deliberately runs the **same Terraform modules** as production with smaller instance parameters. Hand-built, divergent staging environments are why "it worked in staging" is a familiar sentence. The one intentional difference is single-AZ, since staging does not need to survive an AZ failure — and that difference is documented rather than incidental.

---

## 4.5 Provider Portability

The design uses commodity primitives, so the mapping to other providers is direct.

| Capability | AWS | GCP | Azure | Self-hosted |
|---|---|---|---|---|
| Container runtime | ECS Fargate | Cloud Run | Container Apps | Nomad / K8s |
| Load balancer | ALB | Cloud Load Balancing | Application Gateway | HAProxy / Traefik |
| WAF | AWS WAF | Cloud Armor | Azure WAF | ModSecurity |
| PostgreSQL | RDS Multi-AZ | Cloud SQL HA | Database for PostgreSQL | Patroni |
| Redis | ElastiCache | Memorystore | Azure Cache | Redis Sentinel |
| Object storage | S3 | Cloud Storage | Blob Storage | MinIO |
| Secrets | Secrets Manager | Secret Manager | Key Vault | Vault |
| Registry | ECR | Artifact Registry | ACR | Harbor |
| Telemetry | CloudWatch + OTel | Cloud Ops + OTel | Monitor + OTel | Prometheus + Grafana + Tempo |

No proprietary programming or data model is embedded anywhere in the design — no Lambda-shaped handlers, no DynamoDB access patterns, no Cognito-specific token semantics. Those are the choices that make a migration a rewrite rather than a redeployment. The remaining lock-in is operational — IaC, IAM policies, runbooks — which is real work to change but is not architectural.

---

## 4.6 Cost Posture

Roughly \$700–900/month at the Stage 0 baseline, dominated by the Multi-AZ database and the always-on Fargate tasks.

Two of those costs are deliberate and I would not cut them: **Multi-AZ** is what makes the 60–120 second failover automatic rather than a manual, hour-long recovery, and **a minimum of two replicas across two AZs** is what makes a single instance failure invisible. Running single-AZ would save meaningful money and would convert every routine maintenance event into an outage.

Where cost is managed instead: Fargate Spot for workers, which are interruption-tolerant by design; scale-to-minimum overnight; aggressive log sampling and metric-cardinality control, since observability spend can quietly exceed compute spend; S3 lifecycle rules moving archived audit partitions to infrequent-access and then to Glacier; and reserved capacity once usage is predictable.

---

**Related:** [HLD §11 Infrastructure View](../HLD.md#11-infrastructure-view) · [HLD §13 Environments](../HLD.md#13-environments-and-promotion) · [Availability §2 SPOFs](../AVAILABILITY.md#2-single-points-of-failure) · [Development Practices §14 CI/CD](../DEVELOPMENT_PRACTICES.md#14-cicd)
