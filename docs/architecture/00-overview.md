# CEX Matching Engine - Architecture Overview

## Executive Summary

This document outlines the architecture for a production-ready Centralized Exchange (CEX) Matching Engine supporting 500+ trading pairs with sub-millisecond latency and 100,000+ orders/second throughput.

## System Architecture

```mermaid
flowchart TB
    subgraph Clients["Client Layer"]
        REST["REST API"]
        WS["WebSocket"]
    end

    subgraph Gateway["API Gateway (Axum)"]
        Auth["Auth Service"]
        RateLimit["Rate Limiter"]
        Validation["Request Validator"]
        Router["Order Router"]
    end

    subgraph Engine["Matching Engine Cluster"]
        ME1["Shard 1<br/>BTC/USDT, ETH/USDT"]
        ME2["Shard 2<br/>SOL/USDT, ADA/USDT"]
        ME3["Shard N<br/>..."]
        Snapshot["Snapshot Service"]
    end

    subgraph Events["Event Bus (NATS JetStream)"]
        OrderEvents["orders.>"]
        TradeEvents["trades.>"]
        MarketEvents["market_data.>"]
    end

    subgraph Settlement["Settlement Layer"]
        Balancer["Balance Service"]
        WalletDB[("Wallet DB")]
        Ledger["Transaction Ledger"]
    end

    subgraph Persistence["Persistence Layer"]
        PG[("PostgreSQL")]
        TSDB[("TimescaleDB")]
        Redis[("Redis<br/>Sessions/Cache")]
    end

    subgraph Stream["Stream Processing"]
        Arroyo["Arroyo<br/>Aggregation"]
        CandleWriter["Candle Writer"]
    end

    subgraph MarketData["Market Data Distribution"]
        WSPub["WS Publisher"]
        SnapshotPub["Snapshot Service"]
    end

    REST --> Gateway
    WS --> Gateway

    Gateway --> Auth
    Auth --> Redis
    Gateway --> RateLimit
    RateLimit --> Redis

    Gateway --> Validation
    Validation --> Balancer
    Balancer --> WalletDB

    Gateway --> Router
    Router --> ME1
    Router --> ME2
    Router --> ME3

    ME1 --> OrderEvents
    ME2 --> OrderEvents
    ME3 --> OrderEvents

    OrderEvents --> TradeEvents
    TradeEvents --> Settlement

    Balancer --> PG
    Ledger --> PG

    OrderEvents --> Arroyo
    Arroyo --> TSDB
    CandleWriter --> TSDB

    OrderEvents --> MarketEvents
    TradeEvents --> MarketEvents
    MarketEvents --> WSPub

    ME1 --> Snapshot
    ME2 --> Snapshot
    ME3 --> Snapshot
    Snapshot --> PG

    ME1 --> WSPub
    ME2 --> WSPub
    ME3 --> WSPub

    classDef primary fill:#2563eb,stroke:#1e40af,color:#fff
    classDef storage fill:#059669,stroke:#047857,color:#fff
    classDef event fill:#7c3aed,stroke:#6d28d9,color:#fff
    classDef external fill:#dc2626,stroke:#b91c1c,color:#fff

    class REST,WS primary
    class PG,TSDB,Redis storage
    class OrderEvents,TradeEvents,MarketEvents event
```

## Architectural Patterns

### CQRS (Command Query Responsibility Segregation)

The system separates **write operations** (commands) from **read operations** (queries):

| Aspect | Command Side | Query Side |
|--------|-------------|------------|
| Purpose | Mutate state | Read state |
| Database | PostgreSQL (source of truth) | Read replicas + Redis cache |
| Model | Normalized, consistent | Denormalized, optimized |
| Latency | Higher (ACID guaranteed) | Lower (eventual consistency OK) |
| Examples | Place order, cancel order, deposit | Get order book, get trade history |

**Application**:
- Commands flow through Matching Engine → Settlement → PostgreSQL
- Queries read from Redis (cached order books) or TimescaleDB (historical data)
- Market data is pushed proactively via WebSocket

### Event Sourcing

State changes are represented as an immutable sequence of events:

```
Order Placed → Order Accepted → Partial Fill → Partial Fill → Fully Filled → Order Closed
```

**Benefits**:
1. **Audit Trail**: Complete history of all state transitions
2. **Replayability**: Engine can be rebuilt from event stream
3. **Debugging**: Exact reproduction of any market scenario
4. **Analytics**: Event stream feeds downstream systems

**Event Storage**:
- NATS JetStream: Hot events (last 24-48 hours)
- PostgreSQL: Persistent event log (all history)

## Architectural Decision Records

### ADR-001: Rust as Primary Language

**Decision**: Use Rust for all critical-path services.

**Rationale**:
- Memory safety without GC pauses (critical for low latency)
- Zero-cost abstractions enable high-level code with bare-metal performance
- `rust_decimal` provides exact decimal arithmetic (no floating point errors)
- Tokio async runtime is battle-tested for high-concurrency workloads

**Alternatives Considered**:
- Go: GC pauses unacceptable for p99 < 500μs
- C++: Safety concerns, longer development time
- Java: GC and warm-up time issues

### ADR-002: NATS JetStream for Message Broker

**Decision**: Use NATS JetStream as the primary message broker.

**Rationale**:
- Sub-millisecond publish latency
- Built-in persistence (JetStream)
- Simple subject-based routing (vs. Kafka topic complexity)
- Lower operational overhead than Kafka
- Native Rust client with excellent performance

**Alternatives Considered**:
- Kafka: Overkill for our scale, higher latency
- Redis Streams: Limited durability guarantees
- RabbitMQ: Higher latency, AMQP complexity

### ADR-003: In-Memory Order Books with Periodic Snapshots

**Decision**: Maintain order books purely in memory with WAL + periodic snapshots.

**Rationale**:
- Order books must be accessed in < 100μs (database too slow)
- Memory structure: `BTreeMap<Price, VecDeque<Order>>`
- Write-Ahead Log to NATS for durability
- Snapshots every 1000 events or 30 seconds
- Recovery: Load snapshot + replay WAL

**Alternatives Considered**:
- Pure in-memory: Unacceptable data loss risk
- Database-backed: 10-100x slower, not production viable

### ADR-004: Sharding by Symbol

**Decision**: Each symbol (or small symbol group) gets its own dedicated engine thread.

**Rationale**:
- No cross-symbol contention
- Simple assignment: symbol → thread via hash
- Natural scaling: add more shards = more symbols
- One thread per shard eliminates lock overhead

**Trade-offs**:
- ✅ Pro: Lock-free within shard
- ❌ Con: Cannot share liquidity across shards
- ❌ Con: Imbalanced if one symbol dominates volume

### ADR-005: Separate Settlement from Matching

**Decision**: Matching Engine does NOT touch balances. Settlement Service handles all money.

**Rationale**:
- Separation of concerns: matching = price discovery, settlement = funds movement
- Engine can focus purely on order book mechanics
- Settlement can be throttled/queued without affecting matching
- Easier to audit and test

## Component Interaction Flow

### Order Placement Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant R as Redis
    participant B as Balance Service
    participant E as Matching Engine
    participant N as NATS
    participant S as Settlement

    C->>G: POST /orders (REST/WS)
    G->>R: Validate session & rate limit
    G->>B: Check & lock balance
    B->>R: Update available/locked
    B-->>G: Balance locked
    G->>E: Submit order
    E->>E: Match against order book
    E->>N: Publish OrderAccepted
    E->>N: Publish Trades (if any)
    E->>N: Publish OrderBookDelta
    E-->>G: Order accepted
    G-->>C: OrderResponse (order_id, fills)

    N->>S: Consume Trade events
    S->>S: Update user balances
    S->>R: Update cache
```

### Market Data Flow

```mermaid
sequenceDiagram
    participant E as Matching Engine
    participant N as NATS
    participant A as Arroyo
    participant T as TimescaleDB
    participant P as WS Publisher
    participant C as Subscribed Clients

    E->>N: Publish OrderBookDelta
    E->>N: Publish Trade

    N->>P: Forward to WS layer
    P->>C: Push updates via WebSocket

    N->>A: Stream for aggregation
    A->>A: Compute OHLCV candles
    A->>T: Persist candles
```

## Non-Functional Requirements

| Requirement | Target | Strategy |
|------------|--------|----------|
| Latency (matching) | p99 < 500μs | In-memory order books, single-threaded shards |
| Throughput | 100K orders/sec | Horizontal scaling of shards |
| Availability | 99.99% | Active-active with hot failover |
| Durability | Zero data loss | WAL + snapshots, consensus replication |
| Consistency | Strong | ACID for balances, deterministic matching |

## References

- [Service Breakdown](./01-services.md)
- [Matching Engine Core](./02-matching-engine.md)
- [Data Models](./03-data-models.md)
- [Event Flow](./04-events.md)
- [Fault Tolerance](./05-fault-tolerance.md)
- [Security](./06-security.md)
- [Observability](./07-observability.md)
- [Deployment](./08-deployment.md)
