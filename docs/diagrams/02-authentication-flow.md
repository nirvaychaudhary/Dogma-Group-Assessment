# Diagram 2 — Authentication Flow

Short forms are written out the first time they appear. The full list is in the [glossary](../GLOSSARY.md).

The complete token lifecycle: registration, login, authenticated access, rotation, reuse detection, and revocation. Detailed discussion in [System Design sections 1 to 3](../SYSTEM_DESIGN.md#1-user-registration) and [Security sections 3 to 4](../SECURITY.md#3-authentication).

---

## 2.1 Token Lifecycle — State View

```mermaid
stateDiagram-v2
    [*] --> Pending: Register
    Pending --> Active: Email is verified
    Active --> SignedIn: Login succeeds
    SignedIn --> SignedIn: Normal requests
    SignedIn --> Active: Log out, or password changes
    SignedIn --> Locked: A used refresh token appears again
    Locked --> Active: Person logs in again
    Active --> [*]: Account is deleted
```

The two transitions worth noticing are `Refreshing → Compromised` and the `token_version` path. The first is the mechanism that turns a stolen refresh token from a 30-day foothold into a detectable, self-revoking event. The second is how a single database write invalidates every outstanding access token for a user without enumerating a denylist.

---

## 2.2 Registration and Verification

```mermaid
sequenceDiagram
    participant Person
    participant Application
    participant Database
    participant Email

    Person->>Application: Create an account
    Application->>Application: Check the password and the address
    alt Address is already registered
        Application->>Application: Spend the same time as a real hash
        Application->>Database: Queue a notice to the existing owner
    else Address is new
        Application->>Database: Save the account, a one-time link, and the history
    end
    Application-->>Person: The same answer in both cases
    Database-->>Email: Worker sends the right email later
    Person->>Application: Open the link from the email
    Application->>Database: Mark the account active and consume the link
    Application-->>Person: Account is ready
```

---

## 2.3 Login

```mermaid
sequenceDiagram
    participant Person
    participant Application
    participant Database

    Person->>Application: Email and password
    alt Too many recent attempts
        Application-->>Person: Wait, then try again
    else Password does not match, or no such account
        Application->>Database: Record the failure
        Application-->>Person: The same error either way
    else Password matches but the account cannot sign in
        Application-->>Person: Say why, only after the password was correct
    else Password matches and the account is active
        Application->>Database: Store the refresh token and the history
        Application-->>Person: Access token, plus a secure refresh cookie
    end
```

Note that **Argon2id verification happens outside the transaction**. It costs roughly 200 ms, and holding a database connection for that duration under a login burst is how a credential-stuffing attack becomes a connection-pool outage.

---

## 2.4 Authenticated Request — the Hot Path

```mermaid
flowchart TB
    request[Request arrives with an access token]
    signature{Signature, issuer, audience, and expiry are valid?}
    revoked{Token was logged out, or the password changed since?}
    refuse[Refuse the request]
    continue[Build the caller identity and continue]

    request --> signature
    signature -->|No| refuse
    signature -->|Yes| revoked
    revoked -->|Yes| refuse
    revoked -->|No| continue
```

The entire hot path is a signature verification plus two cached lookups. **No database round trip is required to authenticate**, which is the reason the access token is a JSON Web Token (JWT) at all.

---

## 2.5 Refresh with Reuse Detection

```mermaid
flowchart TB
    present[Refresh token is presented]
    valid{Still unused and unexpired?}
    recent{Replaced only seconds ago by this same login?}
    stolen[Revoke every token from that login]
    alert[Email the person and alert the on-call team]
    refuse[Refuse and end the sessions]
    rotate[Issue a new pair and retire the old refresh token]

    present --> valid
    valid -->|Already used| recent
    recent -->|Yes| rotate
    recent -->|No| stolen --> alert --> refuse
    valid -->|No| refuse
    valid -->|Yes| rotate
```

**Why the grace window exists.** A mobile client resuming with several queued requests can fire two refreshes concurrently and legitimately trigger reuse detection. Without the window, normal client behaviour would log users out. Ten seconds is short enough to be useless to an attacker — who has no reason to replay a token within seconds of the legitimate client — and long enough to absorb realistic races. It is a deliberate, bounded softening of the control rather than an unnoticed hole.

---

## 2.6 Revocation Paths

```mermaid
flowchart TB
    logout[Log out of this device] --> one[Revoke this refresh token and this access token]
    everywhere[Log out everywhere, change password, or change role] --> all[Increase the token version so every access token dies]
    theft[A refresh token is reused] --> family[Revoke the whole login family and alert the person]
```

`token_version` is the mechanism that makes global revocation cheap. One integer increment invalidates every outstanding access token for a user, with no need to enumerate or store them — which is what allows the access token to remain stateless without giving up the ability to revoke.

---

**Related:** [System Design section 2 Authentication](../SYSTEM_DESIGN.md#2-authentication) · [System Design section 3 Token Refresh](../SYSTEM_DESIGN.md#3-token-refresh-with-reuse-detection) · [Security section 3 Authentication](../SECURITY.md#3-authentication) · [Security section 4 Token Revocation](../SECURITY.md#4-token-revocation)
