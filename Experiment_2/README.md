# Netflix System Design — High Level Design (HLD)
--
This README explains the full system design in detail. The accompanying file **Netflix_HLD_Complete.drawio** contains the same content laid out visually across six pages (open it in app.diagrams.net or the draw.io desktop app and use the page tabs at the bottom).
--
**Name: Aditya Kumar**
**UID: 24BCS10209**

---

## 1. Problem Statement

Design a video streaming platform like Netflix that allows users to browse a catalog, stream video content on multiple devices, get personalized recommendations, and manage subscriptions.

---

## 2. Functional Requirements

These are the features the system must actually do:

1. **User management** — sign up, log in (email/password or OAuth/social login), manage multiple profiles per account, parental controls.
2. **Content catalog** — browse movies/shows by genre, language, new releases; search by title, actor, or genre.
3. **Video streaming** — play video with adaptive bitrate streaming (quality adjusts automatically to network speed), resume playback from where the user left off.
4. **Content upload and encoding** (admin/studio side) — ingest raw video, transcode it into multiple resolutions, bitrates, and formats.
5. **Recommendations** — personalized homepage rows based on watch history, ratings, and trending content.
6. **Watch history** — track what each profile has watched and how far into it.
7. **Ratings and reviews** — thumbs up/down or star ratings, which feed back into recommendations.
8. **Subscription and billing** — plan selection, recurring payments, upgrade/downgrade/cancel.
9. **Multi-device support** — web, mobile, smart TV, with consistent state across devices.
10. **Download for offline viewing** — mobile devices only.
11. **Notifications** — new episode alerts, renewal reminders.

**Explicitly out of scope** for this assignment: live streaming/sports, detailed content-licensing business logic, and the internal ML model design for recommendations (only the interface and storage for recommendations is covered).

---

## 3. Non-Functional Requirements

These describe *how well* the system must perform, not *what* it does:

| Requirement | What it means here |
|---|---|
| **Availability** | 99.99% uptime — streaming and browsing should almost never go down. |
| **Low latency** | Video should start playing in under 2 seconds; API responses should return in under 200ms. |
| **Scalability** | Must support 250M+ registered users and tens of millions of concurrent streams at peak. |
| **Durability** | No loss of video assets, user data, or billing records, ever. |
| **Consistency** | Eventual consistency is fine for catalog browsing and recommendations; billing/payments need strong consistency (ACID transactions). |
| **Global reach** | Multi-region deployment, with content served from a location physically close to the user. |
| **Security** | Encrypted video delivery (DRM), secure payment processing (PCI-DSS compliant), encryption of data at rest and in transit. |
| **Fault tolerance** | No single point of failure; the system should degrade gracefully (e.g. drop to a lower video quality) instead of failing outright. |
| **Elasticity** | The system must auto-scale during predictable peak hours — evenings, weekends, and new-release days. |

Why this split matters: functional requirements tell you *what to build*; non-functional requirements tell you *how to build it* so it survives real-world load — and they directly drive the architecture decisions in Section 7 (e.g. the CDN exists purely to satisfy the low-latency and scalability requirements).

---

## 4. Capacity Estimation

Rough, order-of-magnitude estimation — the point is to justify architectural choices (e.g. "why do we need a CDN," "why NoSQL for watch history"), not to be exact.

**Assumptions**
- Registered users: 250 million
- Daily Active Users (DAU): 100 million
- Average watch time per DAU: 2 hours/day
- Peak concurrency: ~20% of DAU streaming at the same time → **20 million concurrent streams**
- Average streaming bitrate (blended across SD/HD/4K): ~5 Mbps
- Catalog size: ~15,000 titles (movies + episodes)

### 4.1 Bandwidth
```
Peak bandwidth = 20,000,000 concurrent streams × 5 Mbps
              = 100,000,000 Mbps
              = 100 Tbps
```
This number is why Netflix cannot realistically serve all traffic from its own origin servers — it relies on **CDN edge caching / ISP-embedded caches** (Netflix calls this "Open Connect") so that video bytes travel the shortest possible network path to the user.

### 4.2 Storage
```
Titles                     = 15,000
Renditions per title       = ~10 (different resolutions/bitrates/codecs)
Avg size per rendition     = ~7 GB (2-hour movie, blended average)
Total raw storage          ≈ 15,000 × 10 × 7 GB ≈ 1.05 PB
With 3x replication        ≈ ~3 PB total
```

### 4.3 Request Rate (API / metadata layer)
```
DAU                         = 100,000,000
Avg API calls per user/day  = 50 (browse, search, history update, recommendation fetch, etc.)
Total daily requests        = 5,000,000,000
Average RPS                 = 5B ÷ 86,400 ≈ 58,000 RPS
Peak RPS (3–5× average)     ≈ 175,000 – 290,000 RPS
```

### 4.4 Database Size (rough)
```
Users table:        250M rows × ~1 KB               ≈ 250 GB
Watch history:       100M DAU × 5 events/day × 365 days × 200 bytes ≈ 36.5 TB/year
                     (this grows continuously — needs partitioning and a TTL/archival policy)
Catalog metadata:    15,000 titles × ~50 KB          ≈ 750 MB
```

---

## 5. Database Structure

Netflix-style systems use **polyglot persistence** — different databases for different access patterns, rather than one relational database for everything.

| Data | Store type | Example tech | Why |
|---|---|---|---|
| User accounts, profiles, subscriptions, payments | Relational (SQL) | PostgreSQL / MySQL | Needs strong consistency and transactions (billing). |
| Video/title catalog metadata | Document / wide-column NoSQL | Cassandra / DynamoDB | High read throughput, flexible schema, horizontally scalable. |
| Watch history / viewing events | Wide-column, time-series style | Cassandra | Extremely high write volume, append-heavy, partitioned by user + time. |
| Recommendations (precomputed) | Key-value store | Redis / DynamoDB | Needs ultra-low-latency lookups to render the homepage instantly. |
| Search index | Search engine | Elasticsearch | Full-text search across titles, actors, genres. |
| Video files (actual bytes) | Object storage | S3 (or equivalent) + CDN | Cheap, durable, massive-scale blob storage. |
| Session/cache | In-memory cache | Redis / Memcached | Absorbs read traffic on hot paths (catalog pages, session tokens). |
| Analytics/event stream | Message queue → data lake | Kafka → Hadoop/Spark | Decouples producers (services) from consumers (analytics, ML training). |

### 5.1 Core entities (simplified schema)

- **Users** — id (PK), email, password_hash, country, created_at
- **Profiles** — id (PK), user_id (FK → Users), name, maturity_level
- **Subscriptions** — id (PK), user_id (FK → Users), plan_type, status, renewal_date
- **Titles** — id (PK), title, genre, description, release_year
- **VideoFiles** — id (PK), title_id (FK → Titles), resolution, bitrate, cdn_url
- **WatchHistory** — id (PK), profile_id (FK → Profiles), title_id (FK → Titles), position_seconds, watched_at
- **Reviews** — id (PK), profile_id (FK → Profiles), title_id (FK → Titles), rating, created_at

**Relationships:**
- One User has many Profiles (parental control model — one account, several viewer profiles).
- One User has one or more Subscriptions over time (plan history).
- One Title has many VideoFiles (one per resolution/bitrate rendition).
- One Profile generates many WatchHistory records; one Title is watched in many WatchHistory records.
- One Profile writes many Reviews; one Title receives many Reviews.

This exact structure is drawn as an ER diagram (with boxes and relationship lines) on **page 4** of the draw.io file.

---

## 6. API Design (REST-style)

### Auth Service
```
POST   /api/v1/auth/signup
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh-token
```

### User & Profile Service
```
GET    /api/v1/users/{userId}
PUT    /api/v1/users/{userId}
POST   /api/v1/users/{userId}/profiles
GET    /api/v1/users/{userId}/profiles
DELETE /api/v1/users/{userId}/profiles/{profileId}
```

### Catalog Service
```
GET    /api/v1/titles                     # list/browse with filters (genre, year)
GET    /api/v1/titles/{titleId}           # title details
GET    /api/v1/search?q={query}
```

### Streaming Service
```
GET    /api/v1/titles/{titleId}/manifest          # returns DASH/HLS manifest URL for adaptive streaming
POST   /api/v1/titles/{titleId}/playback-session  # start playback session, get signed CDN URL
```

### Watch History Service
```
POST   /api/v1/profiles/{profileId}/history       # update watch progress
GET    /api/v1/profiles/{profileId}/history        # fetch continue-watching list
```

### Recommendation Service
```
GET    /api/v1/profiles/{profileId}/recommendations
```

### Review Service
```
POST   /api/v1/titles/{titleId}/reviews
GET    /api/v1/titles/{titleId}/reviews
```

### Billing Service
```
POST   /api/v1/subscriptions
GET    /api/v1/subscriptions/{subId}
PUT    /api/v1/subscriptions/{subId}         # upgrade/downgrade
DELETE /api/v1/subscriptions/{subId}         # cancel
```

Each group of endpoints maps to one independent microservice — this is deliberate, so the API surface mirrors the architecture in the next section.

---

## 7. High-Level Architecture

**Flow, end to end:**

1. **Client apps** (Web, Mobile, Smart TV) talk to the backend through an **API Gateway**, which handles routing, rate-limiting, and auth-token validation.
2. The gateway routes requests to independent **microservices**: Auth Service, Catalog Service, Streaming Service, and Billing Service (the diagram keeps this to four services for clarity — Recommendation, Review, and Watch History services follow the same pattern and were folded into the API list above without cluttering the diagram).
3. Each microservice reads/writes to the shared **Databases and Cache layer** — SQL for accounts/billing, NoSQL for catalog/history, Redis for hot-path caching, Elasticsearch for search.
4. **Video bytes never pass through the app servers.** The Streaming Service only returns a signed manifest/URL; actual video segments are served directly from **CDN edge nodes**, which pull from **origin object storage (S3)** on a cache miss. This is shown as a separate arrow going straight from the CDN back to the client, bypassing the API gateway entirely.
5. **Event-driven pipeline**: user actions (play, pause, rating, search) are published to a **message queue (Kafka)**, consumed asynchronously by an **Analytics/ML pipeline** (e.g. Spark) that trains recommendation models and writes results back into the fast key-value recommendation store.

**Why it's built this way:**
- **CDN instead of serving video from your own servers** — bandwidth costs and latency both demand that video live physically close to the user; recall the 100 Tbps peak bandwidth figure from Section 4.1.
- **SQL for billing, NoSQL for catalog/history** — billing needs ACID transactions; catalog and watch history need horizontal read/write scale far more than strict consistency.
- **Separating the streaming/manifest service from the actual video delivery path** — this decouples the control plane (auth, session, quality selection) from the data plane (raw byte delivery via CDN), so application servers are never a bandwidth bottleneck.
- **Async event pipeline for analytics/recommendations** — this avoids blocking the user-facing playback path on analytics writes; recommendations can update near-real-time rather than strictly real-time.

This exact flow is drawn as a component diagram on **page 6** of the draw.io file, with color coding: gray = client-facing, blue = gateway/microservices, green = databases/cache, orange = storage/CDN, purple = async/analytics.

---

## 8. How to Use the draw.io File

1. Open **Netflix_HLD_Complete.drawio** in [app.diagrams.net](https://app.diagrams.net) or the draw.io desktop app.
2. Use the page tabs at the bottom to move between sections — they map directly to the numbered sections above:
   - Page 1 → Section 2 (Functional Requirements)
   - Page 2 → Section 3 (Non-Functional Requirements)
   - Page 3 → Section 4 (Estimation)
   - Page 4 → Section 5 (Database Structure)
   - Page 5 → Section 6 (API Endpoints)
   - Page 6 → Section 7 (Architecture)
3. Every box and text element is fully editable — reposition, recolor, or reword anything directly on the canvas.
