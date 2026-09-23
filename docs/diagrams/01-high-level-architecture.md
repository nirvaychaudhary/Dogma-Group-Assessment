# Diagram 1 — High-Level Architecture

The system is shown as three small pictures instead of one crowded drawing. Each picture is one idea.

The written design is in [High-Level Design, sections 4 to 6](../HLD.md#4-logical-architecture).

---

## 1. How a request travels

A person uses the web or mobile app. The request is checked at the edge, then handed to an application server.

```mermaid
flowchart TB
    person[Person using the web or mobile app]
    edge[Content delivery network and firewall]
    balancer[Load balancer]
    app[Application server]

    person --> edge --> balancer --> app
```

The edge blocks obvious attacks. The load balancer sends the request to a healthy server. Any server can answer, because the servers do not keep private memory of the user.

---

## 2. What the application does with it

Every request walks the same path. Cheap checks happen first, so a bad request is rejected before it does expensive work.

```mermaid
flowchart TB
    track[Give the request a tracking number]
    size[Reject anything oversized]
    limit[Limit repeated calls]
    who[Confirm who is calling]
    allow[Check permission for this action]
    save[Save the change and the history together]
    answer[Send the answer]

    track --> size --> limit --> who --> allow --> save --> answer
```

Permission is its own step. A user can only reach their own tasks. An administrator uses a separate admin area, and that access is recorded.

---

## 3. Where information is kept

```mermaid
flowchart TB
    app[Application server]
    database[(Main database)]
    standby[(Standby database)]
    redis[(Temporary store)]
    worker[Background worker]
    email[Email service]
    backups[(Backup storage)]

    app --> database
    app --> redis
    database --> standby
    database --> backups
    database --> worker
    worker --> email
```

The main database is the source of truth. The standby is a live copy used if the main database fails.

The temporary store holds rate limits, revoked login passes, and the job queue. It is not the source of truth. If it is lost, the service slows down. It does not lose tasks.

Email is sent by a background worker, after the database change is saved. A failed email never undoes the task, and a cancelled task never sends a stray email.

---

**Related:** [High-Level Design, section 4](../HLD.md#4-logical-architecture) · [High-Level Design, section 5](../HLD.md#5-component-responsibilities) · [High-Level Design, section 6](../HLD.md#6-request-lifecycle)
