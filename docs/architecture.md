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
[ Client (Browser/App) ] 
       │
       ▼
    [ DNS ] 
       │
       ▼
    [ CDN ] (Caches static assets/frontend)
       │
       ▼
[ Load Balancer ] (Distributes traffic across app servers)
       │
       ├─────────────────────────┐
       ▼                         ▼
[ App Server 1 ]         [ App Server 2 ] (Stateless Node.js API instances)
       │                         │
       ├───────────┬─────────────┘
       │           │
       ▼           ▼
[ Redis Cache ]  [ Primary Database (PostgreSQL Master - Writes) ]
                   │
                   ▼ (Streaming Replication)
                 [ Read Replica (PostgreSQL - Reads) ]
                   │
       ┌───────────┘
       ▼
[ Queue (RabbitMQ) ] -> [ Worker Server (Background Tasks) ]