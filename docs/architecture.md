# QuickNotes System Architecture

## 1. Requirements

### Functional Requirements
* **User Authentication**: Users can register, log in, and maintain a secure session.
* **Note Management**: Users can create, read, update, and delete notes (CRUD).
* **Tagging**: Users can assign tags to notes for organization and filtering.
* **Search**: Users can search notes by title and body content.

### Non-Functional Requirements
* **Availability**: 99.9% uptime for core note-reading and writing operations.
* **Latency**: Read requests must respond in under 100ms; write requests under 200ms.
* **Scalability**: Must seamlessly support up to 1 million active users with predictable performance.
* **Security**: Data encrypted in transit (HTTPS) and at rest (database encryption).

---

## 2. Load Estimates (1 Million Users)

* **Active Users**: Assume 1 million daily active users (DAU).
* **Read Load**: 
  * Each user reads notes 40 times a day = 40,000,000 reads/day.
  * **Reads per second (QPS)** = $40,000,000 / 86,400 \approx \mathbf{46.3\text{ QPS}}$ average (Peak QPS $\approx 5 \times 46.3 \approx \mathbf{231\text{ QPS}}$).
* **Write Load**: 
  * Each user creates/updates 10 notes a day = 10,000,000 writes/day.
  * **Writes per second (QPS)** = $10,000,000 / 86,400 \approx \mathbf{11.6\text{ QPS}}$ average (Peak QPS $\approx 5 \times 11.6 \approx \mathbf{58\text{ QPS}}$).
* **Storage per Year**: 
  * 10M notes/day $\times 365$ days = 3.65 billion notes/year.
  * Assuming an average note size of 200 bytes: $3.65\text{B} \times 200\text{ bytes} \approx \mathbf{730\text{ GB per year}}$ (plus metadata/indexes $\approx 1\text{ TB/year}$).

---

## 3. Architecture Diagram

```text
                 ┌──> [ CDN (static assets) ]
[ Client ] -> [ DNS ] -> [ Load Balancer ] -> [ App Servers ]
                                              ├──> [ Redis Cache ]
                                              ├──> [ PostgreSQL Primary (writes) ]
                                              │            │
                                              │            └── replication ──> [ Read Replica (reads) ]
                                              └──> [ RabbitMQ ] -> [ Workers ]
```

## 4. Component Responsibilities

* **DNS** resolves the public application and API hostnames to their service endpoints.
* **CDN** serves static frontend assets from edge locations. Authenticated API responses are not cached at the CDN.
* **Load balancer** forwards HTTPS requests to healthy application instances using health checks and distributes traffic across availability zones.
* **Stateless application servers** authenticate and authorize requests, validate input, implement the API, and coordinate cache/database access. Any healthy instance can serve any request.
* **Redis** caches frequently read, user-scoped note data to reduce latency and database load. Successful writes invalidate or refresh affected entries. If Redis is unavailable, the API falls back to PostgreSQL.
* **PostgreSQL primary** is the source of truth and accepts writes. Transactions keep note and tag updates consistent.
* **PostgreSQL read replica** handles eligible reads and receives changes through streaming replication. Requests requiring read-after-write consistency are routed to the primary.
* **RabbitMQ and workers** handle asynchronous tasks, such as notifications or indexing, outside the request path. Workers should be idempotent and messages retried safely.

## 5. Request Flows

### GET `/notes`

1. The client sends an HTTPS request with its authentication token to the API hostname.
2. The load balancer health-checks application servers and forwards the request to a healthy instance.
3. The application validates the token and derives the user ID; it does not trust a client-supplied owner ID.
4. The application checks Redis for that user's requested page. On a cache hit, it returns the user-scoped result. On a miss, it queries PostgreSQL, normally through the read replica, and caches the result with a bounded expiration.
5. The application returns the response through the load balancer. If Redis is unavailable, the database result is returned directly.

### POST `/notes`

1. The client sends an HTTPS request containing note fields and its authentication token.
2. The load balancer routes it to a healthy application instance, which authenticates the user and validates the note and tag ownership.
3. The application writes the note and its tag relationships in a transaction on the PostgreSQL primary.
4. After the transaction commits, the application invalidates affected Redis list/detail entries and returns the created note. It does not report success before the primary commit succeeds.
5. Work that is not required to complete the API response can be published to RabbitMQ for a worker to process asynchronously.

## 6. Availability and Failure Handling

* **Application instance failure:** Load-balancer health checks remove unhealthy instances from rotation. Multiple stateless instances across availability zones continue serving requests.
* **Redis failure:** The application bypasses the cache and reads from PostgreSQL. Correctness is preserved, though latency and database load may increase; connection limits and rate controls help protect the database during cache recovery.
* **Read-replica failure or lag:** Route reads to another healthy replica or temporarily to the primary. Read-after-write requests always use the primary to avoid stale results.
* **Primary database outage:** Run the primary and standby in separate failure zones, monitor replication health, and promote a healthy replica through a controlled failover process. Applications reconnect to the promoted primary and retry only operations safe to retry; writes during failover may receive a retriable service error rather than a success response. Asynchronous replication can lose the latest transactions that have not reached the standby, so replication mode determines the recovery point objective. Maintain backups and point-in-time recovery, and rejoin the repaired former primary as a replica; never allow two writable primaries.
* **Queue or worker outage:** Core note reads and writes remain available when queued work is not required for the response. Durable queues retain messages while workers recover; dead-letter handling surfaces messages that exceed retry limits.

These measures reduce single points of failure but cannot guarantee zero downtime or zero data loss. Multi-zone deployment, tested failover drills, verified backups, and explicit recovery objectives are necessary to support the availability target.