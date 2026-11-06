# Movie Ticket Booking Platform — Architecture

This document complements [Requirements.md](Requirements.md) and [Implementation-Plan.md](Implementation-Plan.md). It shows **what talks to what, and in what order** — a static component view plus execution-flow (sequence) diagrams for every major scenario.

## 1. Component Architecture (High-Level, Numbered Request Flow)

Edge labels are numbered to show the typical order of execution for a request originating from the UI, not just static connectivity.

```mermaid
flowchart TB
    subgraph client ["Client Layer"]
        ui["React Web App"]
    end

    subgraph edge ["Edge"]
        gw["API Gateway"]
        fa["FusionAuth (OIDC/OAuth2)"]
    end

    subgraph services ["Core Microservices"]
        theatreSvc["Theatre Service"]
        movieSvc["Movie and Show Service"]
        invSvc["Inventory Service"]
        bookingSvc["Booking Service"]
        promoSvc["Promotion Service"]
        walletSvc["Wallet Service"]
    end

    subgraph integration ["Integration Layer"]
        adapter["Theatre Integration Adapter"]
        kafka[("Kafka Event Bus")]
    end

    subgraph data ["Data Stores"]
        pg[("PostgreSQL, one schema per service")]
        redis[("Redis")]
    end

    ui -->|"1 HTTPS request + JWT"| gw
    gw -->|"2 validate token"| fa
    gw -->|"3 route"| theatreSvc
    gw -->|"3 route"| movieSvc
    gw -->|"3 route"| bookingSvc
    bookingSvc -->|"4 lock seats"| invSvc
    bookingSvc -->|"5 price booking"| promoSvc
    bookingSvc -->|"6 debit wallet"| walletSvc
    invSvc -->|"uses"| redis
    theatreSvc --> pg
    movieSvc --> pg
    invSvc --> pg
    bookingSvc --> pg
    promoSvc --> pg
    walletSvc --> pg
    theatreSvc -->|"legacy sync"| adapter
    adapter -->|"7 publish events"| kafka
    kafka -->|"8 consume"| movieSvc
    kafka -->|"8 consume"| invSvc
```

## 2. Execution Flow Diagrams

### 2.1 Authentication Flow

```mermaid
sequenceDiagram
    actor User
    participant UI as ReactApp
    participant FA as FusionAuth
    participant GW as APIGateway

    User->>UI: Click Login
    UI->>FA: OIDC authorization request (PKCE)
    FA->>User: Show hosted login form
    User->>FA: Submit credentials
    FA-->>UI: Redirect with authorization code
    UI->>FA: Exchange code for tokens
    FA-->>UI: Access token, ID token, refresh token
    UI->>GW: API request with Bearer token
    GW->>FA: Validate token against JWKS
    FA-->>GW: Token valid, roles and claims
    GW-->>UI: Proxied response
```

### 2.2 Theatre Onboarding & Show Management (B2B Write)

```mermaid
sequenceDiagram
    actor Partner as TheatrePartner
    participant UI as ReactApp
    participant GW as APIGateway
    participant TS as TheatreService
    participant MS as MovieShowService
    participant DB as PostgreSQL

    Partner->>UI: Submit theatre and screen details
    UI->>GW: POST /theatres (JWT role=THEATRE_PARTNER)
    GW->>TS: Forward request
    TS->>TS: Validate role and payload
    TS->>DB: Insert theatre and screens
    DB-->>TS: Persisted
    TS-->>GW: 201 Created
    GW-->>UI: Theatre onboarded

    Partner->>UI: Create show for a screen
    UI->>GW: POST /shows
    GW->>MS: Forward request
    MS->>TS: Verify screen exists
    TS-->>MS: Screen details
    MS->>DB: Insert show record
    DB-->>MS: Persisted
    MS-->>GW: 201 Created
    GW-->>UI: Show created
```

### 2.3 Discovery / Browse Flow (B2C Read)

```mermaid
sequenceDiagram
    actor Customer
    participant UI as ReactApp
    participant GW as APIGateway
    participant MS as MovieShowService
    participant PR as PromotionService
    participant Cache as RedisCache

    Customer->>UI: Search movie in city on date
    UI->>GW: GET /movies/{id}/theatres?city=..&date=..
    GW->>MS: Forward request
    MS->>Cache: Check cached result
    alt Cache hit
        Cache-->>MS: Cached theatre and show list
    else Cache miss
        MS->>MS: Query shows by city, date, movie
        MS->>Cache: Store result with TTL
    end
    MS->>PR: Get active offers for city/theatres
    PR-->>MS: Offers list
    MS-->>GW: Theatres, showtimes, offers
    GW-->>UI: Render results
```

### 2.4 Booking Flow (Core Saga) — Critical Path

```mermaid
sequenceDiagram
    actor Customer
    participant UI as ReactApp
    participant GW as APIGateway
    participant BK as BookingService
    participant INV as InventoryService
    participant PR as PromotionService
    participant WAL as WalletService
    participant KFK as KafkaEventBus

    Customer->>UI: Select seats, confirm booking
    UI->>GW: POST /bookings (JWT)
    GW->>BK: Forward request
    BK->>INV: Lock seats (showId, seatIds, strategy)
    INV-->>BK: Seats locked (lockToken)
    BK->>PR: Calculate price (showId, seatIds, showTime)
    PR-->>BK: finalAmount with applied promotions
    BK->>WAL: Debit wallet (customerId, finalAmount, idempotencyKey)
    alt Sufficient balance
        WAL-->>BK: Debit success (transactionId)
        BK->>BK: Persist booking CONFIRMED + outbox event (local tx)
        BK->>INV: Confirm seats (lockToken)
        INV-->>BK: Seats marked BOOKED
        BK->>KFK: Outbox relay publishes booking.confirmed
        BK-->>GW: 201 Created (booking details)
        GW-->>UI: Booking confirmed
    else Insufficient balance
        WAL-->>BK: Debit failed (insufficient funds)
        BK->>INV: Release seat lock
        INV-->>BK: Lock released, seats AVAILABLE
        BK-->>GW: 402 Payment Required
        GW-->>UI: Booking failed, insufficient balance
    end
```

### 2.5 Cancellation & Refund Flow

```mermaid
sequenceDiagram
    actor Customer
    participant UI as ReactApp
    participant GW as APIGateway
    participant BK as BookingService
    participant INV as InventoryService
    participant WAL as WalletService
    participant KFK as KafkaEventBus

    Customer->>UI: Cancel booking
    UI->>GW: DELETE /bookings/{id}
    GW->>BK: Forward request
    BK->>BK: Load booking, validate cancellable status
    BK->>INV: Release seats (showId, seatIds)
    INV-->>BK: Seats released, marked AVAILABLE
    BK->>WAL: Credit wallet refund amount (idempotencyKey)
    WAL-->>BK: Credit success (transactionId)
    BK->>BK: Update booking status to CANCELLED
    BK->>KFK: Publish booking.cancelled event
    BK-->>GW: 200 OK
    GW-->>UI: Booking cancelled, refund credited
```

### 2.6 Legacy Theatre Integration Flow

```mermaid
sequenceDiagram
    participant Legacy as LegacyTheatreSystem
    participant Adapter as TheatreIntegrationAdapter
    participant KFK as KafkaEventBus
    participant MS as MovieShowService
    participant INV as InventoryService

    alt Push model (webhook)
        Legacy->>Adapter: POST /webhooks/theatres/{id}/shows (signed payload)
    else Pull model (scheduled poll)
        Adapter->>Legacy: GET /partner-api/shows (scheduled poll)
        Legacy-->>Adapter: Shows and inventory snapshot
    end
    Adapter->>Adapter: Normalize payload to internal schema
    Adapter->>KFK: Publish show.updated and inventory.updated
    KFK-->>MS: Consume show.updated
    MS->>MS: Upsert show record
    KFK-->>INV: Consume inventory.updated
    INV->>INV: Upsert seat availability
```

## 3. Deployment Topology (Single Region, Multi-AZ)

```mermaid
flowchart TB
    internet(("Internet")) --> lb["Load Balancer"]

    subgraph region ["Cloud Region"]
        subgraph az1 ["Availability Zone 1"]
            gw1["API Gateway Pod"]
            svc1["Service Pods — Replica Set A"]
        end
        subgraph az2 ["Availability Zone 2"]
            gw2["API Gateway Pod"]
            svc2["Service Pods — Replica Set B"]
        end
        pgPrimary[("PostgreSQL Primary (AZ1)")]
        pgReplica[("PostgreSQL Replica (AZ2)")]
        redisCluster[("Redis Cluster")]
        kafkaCluster[("Kafka Cluster, 3 brokers across AZs")]
    end

    lb --> gw1
    lb --> gw2
    gw1 --> svc1
    gw2 --> svc2
    svc1 --> pgPrimary
    svc2 --> pgPrimary
    pgPrimary -.->|replicates| pgReplica
    svc1 --> redisCluster
    svc2 --> redisCluster
    svc1 --> kafkaCluster
    svc2 --> kafkaCluster
```

Notes:
- Pods are spread across AZs via `topologySpreadConstraints`/pod anti-affinity so an AZ outage doesn't take the whole fleet down.
- PostgreSQL uses a primary/replica setup within the region; Redis and Kafka run as multi-AZ clusters.
- This topology targets the 99.99% single-region availability goal from [Requirements.md §5](Requirements.md#5-non-functional-requirements) without the added complexity of multi-region/multi-cloud (explicitly out of scope for now).
