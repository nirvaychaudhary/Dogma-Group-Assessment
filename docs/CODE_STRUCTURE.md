# Proposed Code Structure

## 1. The Organising Idea

The package layout is **modular by domain at the top level, layered within each module**.

The common alternative — top-level `routers/`, `services/`, `models/`, `schemas/` — groups files by technical role.

It looks tidy on day one and degrades predictably: adding one feature touches five directories, every directory eventually contains everything, and nothing tells you where a domain boundary lies.

Grouping by domain first means a feature is largely contained in one directory, and the boundary that would become a service boundary is already visible in the filesystem.

Four rules hold the structure together:

1. **Dependencies point inward.** `api → services → domain`, and `services → repositories → domain`. The domain layer imports nothing from the layers outside it.
2. **The domain layer has no I/O.** No SQLAlchemy, no Hypertext Transfer Protocol (HTTP), no `datetime.now()` — time is injected. This is what makes business rules testable in microseconds and portable across delivery mechanisms.
3. **Modules communicate through published interfaces.** `modules/tasks` may import `modules.identity`'s public surface, never its repositories or object-relational mapping (ORM) models.
4. **Infrastructure is injected, not imported.** Services depend on protocols; concrete adapters are wired at the composition root.

These rules are enforced in CI by an import-linter contract, not by convention. A convention that is only documented is a convention that erodes — usually under deadline pressure, which is exactly when the boundary matters most.

---

## 2. Directory Layout

```text
task-management-system/
│
├── src/
│   └── taskapp/
│       │
│       ├── main.py                     # ASGI app factory, composition root, lifespan
│       ├── worker.py                   # Background worker entry point
│       │
│       ├── core/                       # Cross-cutting, domain-agnostic
│       │   ├── config.py               # Pydantic Settings, validated at boot
│       │   ├── errors.py               # Exception hierarchy + RFC 9457 mapping
│       │   ├── logging.py              # structlog config, redaction processors
│       │   ├── telemetry.py            # OpenTelemetry setup, metrics registry
│       │   ├── context.py              # Request-scoped context: correlation ID, principal
│       │   ├── pagination.py           # Cursor encode/decode, page envelopes
│       │   ├── clock.py                # Injectable time source
│       │   └── types.py                # Shared value types, UUIDv7 generation
│       │
│       ├── security/                   # Authentication and authorization primitives
│       │   ├── principal.py            # Principal value object
│       │   ├── passwords.py            # Argon2id hashing, policy, breach checks
│       │   ├── tokens.py               # JWT mint/verify, refresh generation
│       │   ├── keys.py                 # JWKS, key rotation
│       │   ├── policy.py               # ← THE authorization decision point
│       │   ├── permissions.py          # Role → permission mapping
│       │   └── ratelimit.py            # Token bucket over Redis
│       │
│       ├── db/
│       │   ├── base.py                 # Declarative base, naming conventions
│       │   ├── session.py              # Engine, async session factory, pooling
│       │   ├── unit_of_work.py         # Transaction boundary
│       │   ├── repository.py           # Generic repository base
│       │   └── migrations/             # Alembic
│       │
│       ├── modules/
│       │   │
│       │   ├── identity/
│       │   │   ├── domain/
│       │   │   │   ├── user.py         # User entity, invariants
│       │   │   │   ├── enums.py        # UserRole, UserStatus
│       │   │   │   └── events.py       # UserRegistered, PasswordChanged
│       │   │   ├── models.py           # SQLAlchemy ORM tables
│       │   │   ├── schemas.py          # Pydantic request/response models
│       │   │   ├── repository.py       # Persistence for users, tokens
│       │   │   ├── service.py          # Registration, login, refresh, reset
│       │   │   ├── router.py           # /auth and /users routes
│       │   │   └── interface.py        # ← Public surface for other modules
│       │   │
│       │   ├── tasks/
│       │   │   ├── domain/
│       │   │   │   ├── task.py         # Task entity, status state machine
│       │   │   │   ├── enums.py        # TaskStatus, TaskPriority
│       │   │   │   ├── filters.py      # Filter/sort value objects
│       │   │   │   └── events.py
│       │   │   ├── models.py
│       │   │   ├── schemas.py
│       │   │   ├── repository.py       # Always scoped by owner
│       │   │   ├── service.py          # Use cases, transaction boundaries
│       │   │   ├── router.py           # /tasks routes
│       │   │   └── interface.py
│       │   │
│       │   ├── administration/
│       │   │   ├── schemas.py
│       │   │   ├── service.py          # Cross-user use cases
│       │   │   └── router.py           # ← Every /admin route lives here
│       │   │
│       │   └── audit/
│       │       ├── domain/event.py
│       │       ├── models.py           # Partitioned audit_log
│       │       ├── repository.py       # Insert + query only
│       │       ├── service.py
│       │       └── router.py           # /admin/audit-logs
│       │
│       ├── api/
│       │   ├── router.py               # Versioned router assembly
│       │   ├── dependencies.py         # get_current_user, require_admin, uow
│       │   ├── middleware/
│       │   │   ├── correlation.py
│       │   │   ├── authentication.py
│       │   │   ├── ratelimit.py
│       │   │   ├── logging.py
│       │   │   └── errors.py
│       │   ├── health.py
│       │   └── openapi.py              # Spec customisation
│       │
│       ├── integrations/               # Anti-corruption layer
│       │   ├── email/
│       │   │   ├── port.py             # EmailSender protocol
│       │   │   ├── provider.py         # Concrete adapter
│       │   │   └── fake.py             # Test double
│       │   ├── secrets/
│       │   └── cache/
│       │
│       └── jobs/
│           ├── queue.py                # JobQueue port + Redis adapter
│           ├── outbox.py               # Outbox dispatcher
│           ├── scheduler.py            # Leader-elected cron
│           └── handlers/
│               ├── send_email.py
│               ├── purge_deleted.py
│               ├── cleanup_tokens.py
│               └── archive_audit.py
│
├── tests/
│   ├── conftest.py
│   ├── factories/                      # Test data builders
│   ├── unit/                           # Domain + policy, no I/O
│   ├── integration/                    # Real Postgres via Testcontainers
│   ├── api/                            # Full stack through HTTP
│   ├── security/                       # ← Authorization matrix, abuse cases
│   ├── contract/                       # OpenAPI conformance
│   └── load/                           # k6 / Locust scenarios
│
├── docs/                               # This documentation set
├── scripts/                            # Dev tooling, seeding, migration helpers
│
├── pyproject.toml                      # Dependencies, tool config
├── uv.lock                             # Hash-pinned lockfile
├── .importlinter                       # Architecture contracts enforced in CI
├── Makefile
├── docker-compose.yml                  # Local Postgres, Redis, mail catcher
├── Dockerfile                          # Multi-stage, non-root, distroless
├── README.md
└── DECISION_LOG.md
```

---

## 3. Layer Responsibilities

| Layer | Knows about | Must not know about | Test style |
|---|---|---|---|
| `router.py` | HTTP, schemas, services | Structured Query Language (SQL), domain internals | application programming interface (API) tests |
| `schemas.py` | Wire format, validation | Persistence | Unit |
| `service.py` | Use cases, transactions, policy, audit | HTTP, SQL dialect | Integration |
| `domain/` | Business rules, invariants | **Everything external** | Unit, fast |
| `repository.py` | Queries, mapping, unit of work | Business rules | Integration |
| `models.py` | Table definitions | Business rules | — |

### Why the domain layer is separate from the ORM models

`domain/task.py` holds a `Task` entity with behaviour and invariants; `models.py` holds the SQLAlchemy table. They are different objects, and repositories map between them.

This costs a mapping layer, which is real overhead. What it buys:

- **Business rules are testable without a database.** The status state machine is tested in microseconds with no fixtures, so those tests are run constantly rather than avoided.
- **Persistence concerns cannot leak into business logic.** With ORM entities used directly, a lazy-loaded relationship inside a business method becomes a silent N+1 in production, and the business layer starts caring about session lifetimes and detached instances.
- **The schema can change independently of the rules.** Splitting a column does not ripple into domain logic.

I want to be honest that this is the most debatable choice in the structure. For a genuinely simple create, read, update, and delete (CRUD) service, using ORM models directly is faster and perfectly defensible.

I chose separation here because the domain contains rules that are worth protecting — the status state machine, the ownership invariant, version-based concurrency — and because those rules are exactly what needs exhaustive, fast testing. If the entity were a bag of fields with no behaviour, I would not pay this cost.

### Why `interface.py` exists in each module

A module's public surface is explicit and small:

```python
# modules/identity/interface.py  — illustrative
class IdentityInterface(Protocol):
    async def get_user_summary(self, user_id: UUID) -> UserSummary | None: ...
    async def user_exists(self, user_id: UUID) -> bool: ...
```

`modules/tasks` depends on this protocol, never on `identity.repository` or `identity.models`. The import-linter contract makes a violation a build failure. This is the mechanism that keeps the monolith modular: the boundary that would become a network boundary during extraction already exists, and extracting identity later means replacing one implementation of a protocol with an HTTP client.

### Why `security/policy.py` is at the top level, not inside a module

Authorization is not the property of any one module — it spans all of them, and it is the system's primary risk ([Security section 1](SECURITY.md#1-threat-model-first)).

Placing it at the top level means the entire authorization model is one file, reviewable in one sitting, with one owner and one test suite. Scattering ownership checks across module services is how one of them ends up missing.

---

## 4. Composition Root

Everything is wired in exactly one place — `main.py` — so that dependency graph, configuration and lifecycle are visible in a single file rather than discovered through import side effects.

```python
# main.py — illustrative

def create_app(settings: Settings | None = None) -> FastAPI:
    settings = settings or Settings()                # validated at boot; fail fast
    configure_logging(settings)
    configure_telemetry(settings)

    app = FastAPI(
        title="Task Management API",
        version=settings.app_version,
        lifespan=lifespan,
        docs_url="/docs" if settings.expose_docs else None,
    )

    # Order matters: cheap and security-critical work first.
    app.add_middleware(CorrelationIdMiddleware)
    app.add_middleware(RequestLoggingMiddleware)
    app.add_middleware(AuthenticationMiddleware)
    app.add_middleware(RateLimitMiddleware)

    app.include_router(health_router)
    app.include_router(api_v1_router, prefix="/api/v1")

    register_exception_handlers(app)
    return app
```

`create_app()` is a factory rather than a module-level singleton specifically so tests can build an app with overridden settings and fake adapters. A module-level `app = FastAPI()` forces tests to share global state, which is the root of the flaky, order-dependent test suites that teams eventually stop trusting.

---

## 5. How the Structure Supports the Three Goals

### Maintainability

A change is localised by the dimension it varies along. Adding a task field touches `tasks/`; changing the token lifetime touches `security/tokens.py`; swapping the email provider touches one adapter. New engineers can be given a module rather than a codebase. The pattern is identical across modules, so understanding one means understanding all of them.

### Testing

The layering was chosen partly *because* of what it makes cheap to test:

| Target | Needs | Speed |
|---|---|---|
| Domain rules | Nothing | Microseconds |
| Authorization policy | Nothing | Microseconds |
| Services | Real database, fake adapters | Milliseconds |
| Routers | Full app, real database | Tens of milliseconds |

Because the domain and the policy engine are pure, the two most correctness-critical parts of the system are also the fastest and cheapest to test exhaustively. That is not a coincidence — it is the main reason for keeping them free of I/O. Test suites that are slow get run less, and the tests that matter most are the ones you can afford to run on every save.

### Future development

| Likely change | Where it lands |
|---|---|
| New task field | `tasks/` + one migration |
| Task sharing / ACLs | `security/policy.py` + `tasks/repository.py` — the ownership check is already centralised |
| Third role | `security/permissions.py` — a data change |
| single sign-on (SSO) | New adapter in `integrations/`, plus `identity/service.py` |
| Extract workers | Already a separate process; no refactor |
| Extract identity | Replace `interface.py` implementation with an HTTP client |
| Swap the email provider | One file in `integrations/email/` |

The test worth applying to any structure is whether the *expected* changes are cheap. Here they are, because the structure was laid out against a prediction of what would change rather than against a template.

---

## 6. Conventions

**Naming:** `snake_case` for modules and functions, `PascalCase` for classes, a `_` prefix for private helpers. Schemas are suffixed by intent — `TaskCreateRequest`, `TaskResponse`, `TaskSummaryResponse` — so a request model can never be accidentally returned, which is how internal fields leak into API responses.

**Typing:** full annotations, `mypy --strict`, no bare `Any`. Domain identifiers are `NewType`-wrapped (`UserId`, `TaskId`) rather than raw `UUID`, so passing a task ID where a user ID is expected is a type error rather than a security incident.

**Async:** async all the way down for I/O; processor-heavy work such as Argon2id verification goes to a thread pool so it cannot block the event loop. A single synchronous database call in an async handler stalls every concurrent request on that worker — one of the easiest and most damaging mistakes to make in an async Python service, and one that load testing catches only if you look for it.

**Imports:** absolute only, no wildcards, no circular imports — enforced by lint rather than by discipline.
