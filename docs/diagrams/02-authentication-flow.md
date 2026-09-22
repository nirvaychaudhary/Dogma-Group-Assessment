# Diagram 2 — Authentication Flow

The complete token lifecycle: registration, login, authenticated access, rotation, reuse detection, and revocation. Detailed discussion in [System Design §1–§3](../SYSTEM_DESIGN.md#1-user-registration) and [Security §3–§4](../SECURITY.md#3-authentication).

---

## 2.1 Token Lifecycle — State View

```mermaid
stateDiagram-v2
    [*] --> Unregistered

    Unregistered --> PendingVerification: POST /auth/register<br/>202 Accepted
    PendingVerification --> Unregistered: token expires after 24h

    PendingVerification --> Verified: POST /auth/verify-email<br/>single-use token
    Verified --> Authenticated: POST /auth/login<br/>access 15m + refresh 30d

    Authenticated --> Authenticated: authenticated request<br/>signature check only, no I/O
    Authenticated --> Refreshing: access token expires
    Refreshing --> Authenticated: POST /auth/refresh<br/>rotates both tokens

    Refreshing --> Compromised: rotated token replayed<br/>REUSE DETECTED
    Compromised --> Verified: entire family revoked<br/>user notified, must log in again

    Authenticated --> Verified: POST /auth/logout<br/>jti denylisted
    Authenticated --> Verified: password or role changed<br/>token_version incremented
    Authenticated --> Suspended: admin suspends account

    Suspended --> Verified: admin reactivates
    Verified --> [*]: account deleted
```

The two transitions worth noticing are `Refreshing → Compromised` and the `token_version` path. The first is the mechanism that turns a stolen refresh token from a 30-day foothold into a detectable, self-revoking event. The second is how a single database write invalidates every outstanding access token for a user without enumerating a denylist.

---

## 2.2 Registration and Verification

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant W as WAF
    participant A as API
    participant S as Identity Service
    participant DB as PostgreSQL
    participant OB as Outbox Dispatcher
    participant WK as Worker
    participant M as Email Provider

    C->>W: POST /auth/register
    W->>W: IP rate limit — 5/hour
    W->>A: forward + X-Correlation-ID
    A->>A: Validate: email format,<br/>password 12-128 chars, breach corpus
    A->>S: register(email, password)

    alt Email already registered
        S->>S: Dummy Argon2id hash<br/>(equalise response timing)
        S->>DB: Queue "account exists" notice
    else New account
        S->>S: Argon2id hash — 64MiB, t=3, p=4
        rect rgb(238, 245, 255)
        S->>DB: BEGIN
        S->>DB: INSERT users (status=pending)
        S->>DB: INSERT email_tokens (SHA-256, 24h)
        S->>DB: INSERT audit_log (user.registered)
        S->>DB: INSERT outbox_events (verification)
        S->>DB: COMMIT
        end
    end

    Note over A,C: Identical response either way —<br/>no user enumeration
    A-->>C: 202 Accepted

    OB->>DB: Poll unpublished events
    OB->>WK: Dispatch, carrying traceparent
    WK->>M: Send email
    WK->>DB: Mark published

    C->>A: POST /auth/verify-email {token}
    A->>S: verify(token)
    S->>DB: SELECT by SHA-256 hash,<br/>unconsumed and unexpired
    rect rgb(238, 245, 255)
    S->>DB: BEGIN
    S->>DB: UPDATE users SET status=active,<br/>email_verified_at=now()
    S->>DB: UPDATE email_tokens SET consumed_at<br/>(single use)
    S->>DB: INSERT audit_log
    S->>DB: COMMIT
    end
    A-->>C: 200 OK — account active
```

---

## 2.3 Login

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant A as API
    participant R as Redis
    participant S as Identity Service
    participant DB as PostgreSQL
    participant K as Key Store

    C->>A: POST /auth/login {email, password}
    A->>R: GET failure counters<br/>per-IP AND per-account
    alt Either threshold exceeded
        A-->>C: 429 + Retry-After
    else Allowed
        A->>S: authenticate(...)
        S->>DB: SELECT user WHERE email = lower(:email)

        alt User not found
            S->>S: Dummy Argon2id verify<br/>— constant-time behaviour
            S->>R: INCR both counters
            S->>DB: INSERT audit_log (auth.login_failed)
            A-->>C: 401 INVALID_CREDENTIALS
        else Password mismatch
            S->>R: INCR both counters
            S->>DB: INSERT audit_log (auth.login_failed)
            A-->>C: 401 INVALID_CREDENTIALS
        else Password valid but account not usable
            A-->>C: 403 EMAIL_NOT_VERIFIED<br/>or ACCOUNT_SUSPENDED
        else Valid
            S->>S: Rehash if work factor is outdated
            S->>K: Fetch current Ed25519 signing key by kid
            rect rgb(238, 245, 255)
            S->>DB: BEGIN
            S->>S: Mint access JWT — 15 min<br/>claims: sub, role, jti, tv, iss, aud
            S->>S: Generate 256-bit opaque refresh token
            S->>DB: INSERT refresh_tokens<br/>(SHA-256 hash, family_id, ip, user agent)
            S->>DB: UPDATE users SET last_login_at
            S->>DB: INSERT audit_log (auth.login_succeeded)
            S->>DB: COMMIT
            end
            S->>R: DEL failure counters
            A-->>C: 200 {access_token, expires_in: 900}<br/>Set-Cookie: refresh_token<br/>HttpOnly · Secure · SameSite=Strict<br/>Path=/api/v1/auth
        end
    end
```

Note that **Argon2id verification happens outside the transaction**. It costs roughly 200 ms, and holding a database connection for that duration under a login burst is how a credential-stuffing attack becomes a connection-pool outage.

---

## 2.4 Authenticated Request — the Hot Path

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant MW as Middleware
    participant R as Redis
    participant DB as PostgreSQL
    participant H as Handler

    C->>MW: GET /api/v1/tasks<br/>Authorization: Bearer eyJ...
    MW->>MW: Parse header, read kid
    MW->>R: GET jwks:{kid} — 5 min TTL
    MW->>MW: Verify Ed25519 signature<br/>algorithm PINNED, never read from the token
    MW->>MW: Validate exp, nbf, iss, aud<br/>60s clock leeway

    alt Signature or claims invalid
        MW-->>C: 401
    else Valid
        MW->>R: EXISTS denylist:{jti}
        alt Denylisted
            MW-->>C: 401
        else Redis unavailable
            Note over MW,R: FAIL OPEN on the denylist only —<br/>bounded to 15 min, alerted.<br/>Every authorization check still fails closed.
        end
        MW->>DB: Cached lookup of users.token_version
        alt token.tv != user.token_version
            MW-->>C: 401 — password, role or status changed
        else Current
            MW->>MW: Build Principal<br/>{user_id, role, jti, tv}
            MW->>H: Proceed
            H-->>C: 200
        end
    end
```

The entire hot path is a signature verification plus two cached lookups. **No database round trip is required to authenticate**, which is the reason the access token is a JWT at all.

---

## 2.5 Refresh with Reuse Detection

```mermaid
flowchart TB
    START["POST /auth/refresh"] --> HASH["SHA-256 the presented token"]
    HASH --> LOOKUP{"Found in<br/>refresh_tokens?"}

    LOOKUP -->|"No"| R401["401 INVALID_REFRESH_TOKEN"]
    LOOKUP -->|"Yes"| EXPIRED{"Expired?"}
    EXPIRED -->|"Yes"| R401
    EXPIRED -->|"No"| REVOKED{"Already<br/>revoked?"}

    REVOKED -->|"Yes"| GRACE{"Revoked within<br/>the 10s grace window<br/>AND is the immediate<br/>predecessor?"}
    GRACE -->|"Yes — client race"| SAME["Return the same successor<br/>that was already issued"]
    GRACE -->|"No — genuine reuse"| BREACH

    REVOKED -->|"No"| ACTIVE{"User still<br/>active?"}
    ACTIVE -->|"No"| R401B["401 ACCOUNT_INACTIVE"]
    ACTIVE -->|"Yes"| ROTATE

    subgraph ROTATE_BOX["Rotation — single transaction"]
        ROTATE["UPDATE old token:<br/>revoked_at, replaced_by"] --> NEWTOK["INSERT new token,<br/>same family_id"]
        NEWTOK --> NEWJWT["Mint a new access token"]
    end

    subgraph BREACH_BOX["Reuse detected — the token was stolen"]
        BREACH["Revoke the ENTIRE token family"] --> DENY["Denylist every live<br/>access token for this user"]
        DENY --> BUMP["Increment users.token_version"]
        BUMP --> AUDIT["audit_log: refresh_reuse_detected<br/>severity HIGH"]
        AUDIT --> NOTIFY["Email the user +<br/>page the security on-call"]
    end

    NEWJWT --> OK["200 + rotated cookie"]
    NOTIFY --> R401C["401 — every session terminated"]
    SAME --> OK
```

**Why the grace window exists.** A mobile client resuming with several queued requests can fire two refreshes concurrently and legitimately trigger reuse detection. Without the window, normal client behaviour would log users out. Ten seconds is short enough to be useless to an attacker — who has no reason to replay a token within seconds of the legitimate client — and long enough to absorb realistic races. It is a deliberate, bounded softening of the control rather than an unnoticed hole.

---

## 2.6 Revocation Paths

```mermaid
flowchart LR
    subgraph TRIG["Trigger"]
        T1["Logout"]
        T2["Logout everywhere"]
        T3["Password changed"]
        T4["Role changed"]
        T5["Account suspended"]
        T6["Reuse detected"]
    end

    subgraph MECH["Mechanism"]
        M1["Denylist jti<br/>TTL = remaining lifetime"]
        M2["Revoke refresh token"]
        M3["Increment token_version"]
        M4["Revoke the token family"]
    end

    subgraph EFF["Effect"]
        E1["This session ends immediately"]
        E2["Every session ends immediately"]
    end

    T1 --> M1 --> E1
    T1 --> M2
    T2 --> M3 --> E2
    T3 --> M3
    T4 --> M3
    T5 --> M3
    T6 --> M4 --> E2
    T6 --> M3
```

`token_version` is the mechanism that makes global revocation cheap. One integer increment invalidates every outstanding access token for a user, with no need to enumerate or store them — which is what allows the access token to remain stateless without giving up the ability to revoke.

---

**Related:** [System Design §2 Authentication](../SYSTEM_DESIGN.md#2-authentication) · [System Design §3 Token Refresh](../SYSTEM_DESIGN.md#3-token-refresh-with-reuse-detection) · [Security §3 Authentication](../SECURITY.md#3-authentication) · [Security §4 Token Revocation](../SECURITY.md#4-token-revocation)
