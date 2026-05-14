# Data Models

## Entity Relationship Diagram

```mermaid
erDiagram
    users ||--o{ wallets : owns
    users ||--o{ orders : places
    users ||--o{ trades : participates
    users ||--o{ ledger_entries : has
    wallets ||--o{ balances : contains
    assets ||--o{ balances : denominates
    trading_pairs ||--o{ orders : contains
    trading_pairs ||--o{ trades : executes
    orders ||--o{ order_fills : results_in
    orders ||--o{ trades : creates

    users {
        bigint id PK
        string email UK
        string password_hash
        timestamptz created_at
        timestamptz updated_at
        boolean is_active
    }

    wallets {
        bigint id PK
        bigint user_id FK
        string name
        timestamptz created_at
    }

    assets {
        smallint id PK
        string symbol UK
        string name
        smallint decimals
        boolean is_active
    }

    balances {
        bigint id PK
        bigint wallet_id FK
        smallint asset_id FK
        numeric available
        numeric locked
        numeric total
        timestamptz updated_at
        UK(wallet_id, asset_id)
    }

    trading_pairs {
        smallint id PK
        varchar base_asset FK
        varchar quote_asset FK
        numeric min_qty
        numeric max_qty
        numeric step_qty
        numeric min_price
        numeric max_price
        numeric step_price
        boolean is_active
    }

    orders {
        bigint id PK
        bigint user_id FK
        smallint pair_id FK
        varchar side
        varchar order_type
        numeric price
        numeric qty
        numeric filled_qty
        numeric avg_fill_price
        varchar status
        varchar time_in_force
        boolean post_only
        numeric fee_rate
        timestamptz created_at
        timestamptz updated_at
        timestamptz filled_at
    }

    trades {
        bigint id PK
        smallint pair_id FK
        bigint maker_order_id FK
        bigint taker_order_id FK
        bigint maker_user_id FK
        bigint taker_user_id FK
        numeric price
        numeric qty
        numeric maker_fee
        numeric taker_fee
        varchar side
        timestamptz created_at
    }

    order_fills {
        bigint id PK
        bigint order_id FK
        bigint trade_id FK
        numeric qty
        numeric price
        numeric fee
        boolean is_maker
        timestamptz created_at
    }

    ledger_entries {
        bigint id PK
        bigint user_id FK
        smallint asset_id FK
        numeric amount
        numeric balance_after
        varchar transaction_type
        bigint reference_id
        varchar reference_type
        timestamptz created_at
    }
```

## PostgreSQL Schema

### Core Tables

```sql
-- ============================================================================
-- USERS
-- ============================================================================
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created ON users(created_at);

-- ============================================================================
-- ASSETS
-- ============================================================================
CREATE TABLE assets (
    id SMALLSERIAL PRIMARY KEY,
    symbol VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    decimals SMALLINT NOT NULL CHECK (decimals >= 0 AND decimals <= 18),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Seed data
INSERT INTO assets (symbol, name, decimals) VALUES
    ('BTC', 'Bitcoin', 8),
    ('ETH', 'Ethereum', 18),
    ('USDT', 'Tether', 6),
    ('SOL', 'Solana', 9),
    ('ADA', 'Cardano', 6);

-- ============================================================================
-- WALLETS
-- ============================================================================
CREATE TABLE wallets (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL DEFAULT 'Primary Wallet',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, name)
);

CREATE INDEX idx_wallets_user ON wallets(user_id);

-- ============================================================================
-- BALANCES
-- ============================================================================
CREATE TABLE balances (
    id BIGSERIAL PRIMARY KEY,
    wallet_id BIGINT NOT NULL REFERENCES wallets(id) ON DELETE CASCADE,
    asset_id SMALLINT NOT NULL REFERENCES assets(id),
    available NUMERIC(30, 18) NOT NULL DEFAULT 0,
    locked NUMERIC(30, 18) NOT NULL DEFAULT 0,
    total NUMERIC(30, 18) GENERATED ALWAYS AS (available + locked) STORED,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(wallet_id, asset_id),
    CHECK (available >= 0),
    CHECK (locked >= 0)
);

CREATE INDEX idx_balances_wallet ON balances(wallet_id);
CREATE INDEX idx_balances_asset ON balances(asset_id);

-- Trigger to update updated_at
CREATE TRIGGER update_balances_updated_at
    BEFORE UPDATE ON balances
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();

-- ============================================================================
-- TRADING PAIRS
-- ============================================================================
CREATE TYPE order_side AS ENUM ('bid', 'ask');
CREATE TYPE order_status AS ENUM ('open', 'filled', 'partially_filled', 'cancelled', 'rejected', 'expired');
CREATE TYPE order_type AS ENUM ('limit', 'market', 'ioc', 'fok', 'post_only');
CREATE TYPE time_in_force AS ENUM ('gtc', 'ioc', 'fok');

CREATE TABLE trading_pairs (
    id SMALLSERIAL PRIMARY KEY,
    base_asset_id SMALLINT NOT NULL REFERENCES assets(id),
    quote_asset_id SMALLINT NOT NULL REFERENCES assets(id),
    symbol VARCHAR(20) NOT NULL UNIQUE,  -- e.g., 'BTC/USDT'

    -- Trading rules
    min_qty NUMERIC(30, 18) NOT NULL,
    max_qty NUMERIC(30, 18) NOT NULL,
    qty_step NUMERIC(30, 18) NOT NULL,

    min_price NUMERIC(30, 18) NOT NULL,
    max_price NUMERIC(30, 18) NOT NULL,
    price_step NUMERIC(30, 18) NOT NULL,

    min_notional NUMERIC(30, 18) NOT NULL,  -- min_qty * min_price

    -- Fee schedule (can be overridden per user)
    default_maker_fee NUMERIC(10, 6) NOT NULL DEFAULT 0.001,  -- 0.1%
    default_taker_fee NUMERIC(10, 6) NOT NULL DEFAULT 0.001,  -- 0.1%

    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CHECK (base_asset_id != quote_asset_id),
    CHECK (min_qty > 0),
    CHECK (max_qty > min_qty),
    CHECK (qty_step > 0),
    CHECK (min_price > 0),
    CHECK (max_price > min_price),
    CHECK (price_step > 0)
);

-- Seed data
INSERT INTO trading_pairs (base_asset_id, quote_asset_id, symbol, min_qty, max_qty, qty_step, min_price, max_price, price_step, min_notional)
SELECT
    btc.id AS base_asset_id,
    usdt.id AS quote_asset_id,
    'BTC/USDT' AS symbol,
    0.00001::NUMERIC(30,18) AS min_qty,
    1000::NUMERIC(30,18) AS max_qty,
    0.00001::NUMERIC(30,18) AS qty_step,
    0.01::NUMERIC(30,18) AS min_price,
    1000000::NUMERIC(30,18) AS max_price,
    0.01::NUMERIC(30,18) AS price_step,
    10::NUMERIC(30,18) AS min_notional
FROM assets btc, assets usdt
WHERE btc.symbol = 'BTC' AND usdt.symbol = 'USDT';

-- ============================================================================
-- ORDERS
-- ============================================================================
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    pair_id SMALLINT NOT NULL REFERENCES trading_pairs(id),

    side order_side NOT NULL,
    order_type order_type NOT NULL,
    time_in_force time_in_force NOT NULL DEFAULT 'gtc',

    price NUMERIC(30, 18),  -- NULL for market orders
    qty NUMERIC(30, 18) NOT NULL,
    filled_qty NUMERIC(30, 18) NOT NULL DEFAULT 0,
    avg_fill_price NUMERIC(30, 18),  -- NULL until filled

    status order_status NOT NULL DEFAULT 'open',
    post_only BOOLEAN NOT NULL DEFAULT false,

    -- Fee rate at order time (decimal, e.g., 0.001 = 0.1%)
    maker_fee_rate NUMERIC(10, 6),
    taker_fee_rate NUMERIC(10, 6),

    -- Client order ID for idempotency
    client_order_id VARCHAR(100),

    -- Engine metadata
    engine_sequence BIGINT,
    engine_shard_id SMALLINT,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    filled_at TIMESTAMPTZ,

    CHECK (qty > 0),
    CHECK (filled_qty >= 0),
    CHECK (filled_qty <= qty),
    CHECK (price IS NULL OR price > 0),
    CHECK (
        CASE WHEN order_type = 'limit' THEN price IS NOT NULL
        ELSE price IS NULL
        END
    )
);

CREATE INDEX idx_orders_user ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_pair ON orders(pair_id, status, created_at DESC);
CREATE INDEX idx_orders_user_pair ON orders(user_id, pair_id, status);
CREATE INDEX idx_orders_client_id ON orders(user_id, client_order_id) WHERE client_order_id IS NOT NULL;
CREATE INDEX idx_orders_engine_seq ON orders(engine_shard_id, engine_sequence);

-- Auto-update timestamp
CREATE TRIGGER update_orders_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();

-- ============================================================================
-- TRADES
-- ============================================================================
CREATE TABLE trades (
    id BIGSERIAL PRIMARY KEY,
    pair_id SMALLINT NOT NULL REFERENCES trading_pairs(id),

    maker_order_id BIGINT NOT NULL REFERENCES orders(id),
    taker_order_id BIGINT NOT NULL REFERENCES orders(id),
    maker_user_id BIGINT NOT NULL REFERENCES users(id),
    taker_user_id BIGINT NOT NULL REFERENCES users(id),

    side order_side NOT NULL,  -- From taker's perspective
    price NUMERIC(30, 18) NOT NULL,
    qty NUMERIC(30, 18) NOT NULL,

    -- Fees (positive = charged, negative = rebate)
    maker_fee NUMERIC(30, 18) NOT NULL,
    taker_fee NUMERIC(30, 18) NOT NULL,

    -- Engine metadata
    engine_sequence BIGINT NOT NULL,
    engine_shard_id SMALLINT NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CHECK (qty > 0),
    CHECK (price > 0)
);

CREATE INDEX idx_trades_pair ON trades(pair_id, created_at DESC);
CREATE INDEX idx_trades_user ON trades(maker_user_id, taker_user_id, created_at DESC);
CREATE INDEX idx_trades_order ON trades(maker_order_id, taker_order_id);
CREATE INDEX idx_trades_engine_seq ON trades(engine_shard_id, engine_sequence);

-- ============================================================================
-- ORDER FILLS (Bridge table for order->trade relationship)
-- ============================================================================
CREATE TABLE order_fills (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    trade_id BIGINT NOT NULL REFERENCES trades(id) ON DELETE CASCADE,

    qty NUMERIC(30, 18) NOT NULL,
    price NUMERIC(30, 18) NOT NULL,
    fee NUMERIC(30, 18) NOT NULL,
    is_maker BOOLEAN NOT NULL,

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE(order_id, trade_id),
    CHECK (qty > 0),
    CHECK (price > 0)
);

CREATE INDEX idx_order_fills_order ON order_fills(order_id, created_at DESC);
CREATE INDEX idx_order_fills_trade ON order_fills(trade_id);

-- ============================================================================
-- LEDGER ENTRIES
-- ============================================================================
CREATE TYPE transaction_type AS ENUM (
    'deposit',
    'withdrawal',
    'order_lock',
    'order_unlock',
    'trade_fill',
    'trade_fee',
    'fee_rebate',
    'adjustment'
);

CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    asset_id SMALLINT NOT NULL REFERENCES assets(id),

    amount NUMERIC(30, 18) NOT NULL,  -- Positive = credit, Negative = debit
    balance_after NUMERIC(30, 18) NOT NULL,

    transaction_type transaction_type NOT NULL,
    reference_id BIGINT,  -- order_id, trade_id, withdrawal_id, etc.
    reference_type VARCHAR(50),  -- 'order', 'trade', 'withdrawal', etc.

    metadata JSONB,  -- Additional context

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CHECK (asset_id IS NOT NULL)
);

CREATE INDEX idx_ledger_user ON ledger_entries(user_id, asset_id, created_at DESC);
CREATE INDEX idx_ledger_reference ON ledger_entries(reference_type, reference_id);
CREATE UNIQUE INDEX idx_ledger_idempotent ON ledger_entries(reference_type, reference_id) WHERE reference_type IS NOT NULL;

-- ============================================================================
-- USER FEE TIERS
-- ============================================================================
CREATE TABLE user_fee_tiers (
    user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    tier_id SMALLINT NOT NULL,
    maker_fee NUMERIC(10, 6) NOT NULL,
    taker_fee NUMERIC(10, 6) NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_user_fee_tiers_user ON user_fee_tiers(user_id);

-- ============================================================================
-- WITHDRAWALS
-- ============================================================================
CREATE TYPE withdrawal_status AS ENUM ('pending', 'processing', 'completed', 'failed', 'cancelled');

CREATE TABLE withdrawals (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    asset_id SMALLINT NOT NULL REFERENCES assets(id),
    amount NUMERIC(30, 18) NOT NULL,
    fee NUMERIC(30, 18) NOT NULL DEFAULT 0,

    address VARCHAR(255) NOT NULL,
    memo VARCHAR(255),
    network VARCHAR(50),

    tx_hash VARCHAR(255),  -- On-chain transaction hash
    status withdrawal_status NOT NULL DEFAULT 'pending',

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,

    CHECK (amount > 0),
    CHECK (fee >= 0)
);

CREATE INDEX idx_withdrawals_user ON withdrawals(user_id, created_at DESC);
CREATE INDEX idx_withdrawals_status ON withdrawals(status, created_at);
CREATE INDEX idx_withdrawals_tx_hash ON withdrawals(tx_hash) WHERE tx_hash IS NOT NULL;

-- ============================================================================
-- DEPOSITS
-- ============================================================================
CREATE TYPE deposit_status AS ENUM ('pending', 'confirming', 'completed', 'failed');

CREATE TABLE deposits (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    asset_id SMALLINT NOT NULL REFERENCES assets(id),
    amount NUMERIC(30, 18) NOT NULL,

    address VARCHAR(255) NOT NULL,
    tx_hash VARCHAR(255) NOT NULL,
    memo VARCHAR(255),

    confirmations_needed SMALLINT NOT NULL DEFAULT 3,
    confirmations_current SMALLINT NOT NULL DEFAULT 0,

    status deposit_status NOT NULL DEFAULT 'pending',

    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ,

    UNIQUE(asset_id, tx_hash),
    CHECK (amount > 0)
);

CREATE INDEX idx_deposits_user ON deposits(user_id, created_at DESC);
CREATE INDEX idx_deposits_status ON deposits(status, created_at);
CREATE INDEX idx_deposits_tx_hash ON deposits(tx_hash);
```

## TimescaleDB Schema (Market Data)

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- ============================================================================
-- OHLCV CANDLES
-- ============================================================================
CREATE TABLE candles (
    time TIMESTAMPTZ NOT NULL,
    pair_id SMALLINT NOT NULL REFERENCES trading_pairs(id),
    resolution VARCHAR(10) NOT NULL,  -- '1m', '5m', '15m', '1h', '4h', '1d'

    open NUMERIC(30, 18) NOT NULL,
    high NUMERIC(30, 18) NOT NULL,
    low NUMERIC(30, 18) NOT NULL,
    close NUMERIC(30, 18) NOT NULL,
    volume NUMERIC(30, 18) NOT NULL,
    trades_count BIGINT NOT NULL,

    PRIMARY KEY (time, pair_id, resolution)
);

-- Convert to hypertable
SELECT create_hypertable('candles', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Create indexes for common queries
CREATE INDEX idx_candles_pair_res ON candles(pair_id, resolution, time DESC);

-- Create continuous aggregate for hourly candles
CREATE MATERIALIZED VIEW candles_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS time,
    pair_id,
    resolution,
    first(open, time) AS open,
    max(high) AS high,
    min(low) AS low,
    last(close, time) AS close,
    sum(volume) AS volume,
    sum(trades_count) AS trades_count
FROM candles
WHERE resolution = '1m'
GROUP BY time_bucket('1 hour', time), pair_id, resolution;

-- Create continuous aggregate for daily candles
CREATE MATERIALIZED VIEW candles_1d
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS time,
    pair_id,
    resolution,
    first(open, time) AS open,
    max(high) AS high,
    min(low) AS low,
    last(close, time) AS close,
    sum(volume) AS volume,
    sum(trades_count) AS trades_count
FROM candles
WHERE resolution IN ('1m', '5m', '15m', '1h')
GROUP BY time_bucket('1 day', time), pair_id, resolution;

-- ============================================================================
-- ORDER BOOK SNAPSHOTS (Historical)
-- ============================================================================
CREATE TABLE order_book_snapshots (
    time TIMESTAMPTZ NOT NULL,
    pair_id SMALLINT NOT NULL REFERENCES trading_pairs(id),
    engine_sequence BIGINT NOT NULL,

    -- Compressed order book state
    bids JSONB NOT NULL,  -- [{"price": "50000.00", "qty": "1.5"}, ...]
    asks JSONB NOT NULL,

    PRIMARY KEY (time, pair_id)
);

-- Convert to hypertable
SELECT create_hypertable('order_book_snapshots', 'time',
    chunk_time_interval => INTERVAL '1 hour',
    if_not_exists => TRUE
);

-- ============================================================================
-- TICK DATA (Raw trades)
-- ============================================================================
CREATE TABLE tick_trades (
    time TIMESTAMPTZ NOT NULL,
    pair_id SMALLINT NOT NULL REFERENCES trading_pairs(id),
    trade_id BIGINT NOT NULL,

    price NUMERIC(30, 18) NOT NULL,
    qty NUMERIC(30, 18) NOT NULL,
    side order_side NOT NULL,

    -- Foreign keys to users (optional, for analytics)
    maker_user_id BIGINT,
    taker_user_id BIGINT,

    PRIMARY KEY (time, pair_id, trade_id)
);

-- Convert to hypertable
SELECT create_hypertable('tick_trades', 'time',
    chunk_time_interval => INTERVAL '1 day',
    if_not_exists => TRUE
);

-- Create index for time-series queries
CREATE INDEX idx_tick_trades_pair_time ON tick_trades(pair_id, time DESC);

-- Data retention policy: Keep raw ticks for 30 days
SELECT add_retention_policy('tick_trades', INTERVAL '30 days');
```

## Redis Data Structures

### Session Management

```
# User session
session:{user_id} = {
    "user_id": 12345,
    "email": "user@example.com",
    "tier": "VIP_2",
    "created_at": 1715635200,
    "last_seen": 1715671200,
    "ip": "192.168.1.1"
}
TTL: 3600 (1 hour, refresh on activity)
```

### Fee Tiers Cache

```
# User fee tier (cached from database)
fee_tier:{user_id} = {
    "tier_id": 2,
    "maker_fee": "0.0008",  # 0.08%
    "taker_fee": "0.0010"   # 0.10%
}
TTL: 300 (5 minutes)
```

### Order Book Snapshots

```
# Full order book snapshot (periodic)
orderbook:{pair_id} = {
    "bids": [[50000.00, 1.5], [49999.00, 2.0], ...],
    "asks": [[50001.00, 1.0], [50002.00, 3.0], ...],
    "sequence": 123456789,
    "timestamp": 1715671200000
}
TTL: 60 (1 minute)

# Delta updates (published via pub/sub)
orderbook:delta:{pair_id} = {
    "bids": {"50000.00": 1.5, "49999.00": 0},  # Update or delete
    "asks": {"50001.00": 2.5},
    "sequence": 123456790
}
No TTL (published, not stored)
```

### Rate Limiting

```
# Rate limit counters
rate_limit:{user_id}:orders = {
    "count": 95,
    "window_start": 1715671200
}
TTL: 60 (1 minute window)

rate_limit:{ip}:requests = {
    "count": 980,
    "window_start": 1715671200
}
TTL: 60
```

### Balances Cache

```
# User's balance (cached from database)
balance:{user_id}:{asset_id} = {
    "available": "1.5",
    "locked": "0.5",
    "total": "2.0"
}
TTL: 30 (30 seconds, invalidate on update)
```

### Trading Pair Configuration

```
# Trading pair rules (cached)
pair_config:{pair_id} = {
    "min_qty": "0.00001",
    "max_qty": "1000",
    "qty_step": "0.00001",
    "min_price": "0.01",
    "max_price": "1000000",
    "price_step": "0.01",
    "min_notional": "10"
}
TTL: 3600 (1 hour)
```

## NATS Subject Naming Convention

### Order Events

```
# Order lifecycle
orders.{pair_id}.{user_id}.{order_id}.placed    -> {"order": {...}, "timestamp": ...}
orders.{pair_id}.{user_id}.{order_id}.accepted  -> {"order_id": ..., "sequence": ...}
orders.{pair_id}.{user_id}.{order_id}.filled    -> {"order_id": ..., "filled_qty": ...}
orders.{pair_id}.{user_id}.{order_id}.cancelled -> {"order_id": ..., "reason": ...}
orders.{pair_id}.{user_id}.{order_id}.rejected  -> {"order_id": ..., "reason": ...}

# Wildcard subscriptions
orders.>                                    # All order events
orders.{pair_id}.>                          # All events for a pair
orders.{pair_id}.{user_id}.>                # All events for a user's orders on a pair
```

### Trade Events

```
# Trade executed
trades.{pair_id}.{trade_id}                -> {"trade": {...}, "timestamp": ...}

# User trades
trades.user.{user_id}                      -> {"trade": {...}}

# Wildcard subscriptions
trades.>                                   # All trades
trades.{pair_id}.>                         # All trades for a pair
```

### Market Data Events

```
# Order book updates
marketdata.orderbook.{pair_id}.snapshot    -> Full order book snapshot
marketdata.orderbook.{pair_id}.delta       -> Incremental update
marketdata.orderbook.{pair_id}.sequence    -> Sequence number

# Ticker data
marketdata.ticker.{pair_id}.24h            -> 24h ticker update
marketdata.ticker.{pair_id}.price          -> Latest price update

# Wildcard subscriptions
marketdata.>                               # All market data
marketdata.orderbook.>                     # All order book updates
```

### Balance Events

```
# Balance changes
balance.{user_id}.{asset_id}.updated       -> {"available": ..., "locked": ...}
balance.{user_id}.lock                     -> Funds locked for order
balance.{user_id}.unlock                   -> Funds unlocked
balance.{user_id}.trade                    -> Balance updated after trade

# Wildcard subscriptions
balance.>                                  # All balance events
balance.{user_id}.>                        # All events for a user
```

### System Events

```
# Engine events
engine.{shard_id}.started                  -> Shard started
engine.{shard_id}.snapshot.created         -> Snapshot created
engine.{shard_id}.snapshot.restored        -> Snapshot restored
engine.{shard_id}.error                    -> Error occurred

# Health checks
health.{service}.{instance_id}             -> Health status
```

## Message Schemas (JSON)

### Order Placed

```json
{
  "type": "order.placed",
  "order": {
    "id": "123456789",
    "user_id": "98765",
    "pair_id": "1",
    "symbol": "BTC/USDT",
    "side": "bid",
    "order_type": "limit",
    "time_in_force": "gtc",
    "price": "50000.00",
    "qty": "1.5",
    "post_only": false,
    "client_order_id": "client-123"
  },
  "fee_rates": {
    "maker": "0.0008",
    "taker": "0.0010"
  },
  "timestamp": 1715671200000
}
```

### Trade Executed

```json
{
  "type": "trade.executed",
  "trade": {
    "id": "987654321",
    "pair_id": "1",
    "symbol": "BTC/USDT",
    "maker_order_id": "123456789",
    "taker_order_id": "123456790",
    "maker_user_id": "11111",
    "taker_user_id": "22222",
    "side": "ask",
    "price": "50000.00",
    "qty": "1.0",
    "maker_fee": "0.04",
    "taker_fee": "0.05"
  },
  "engine_sequence": 123456789,
  "timestamp": 1715671200000
}
```

### Order Book Delta

```json
{
  "type": "orderbook.delta",
  "pair_id": "1",
  "symbol": "BTC/USDT",
  "sequence": 123456789,
  "bids": {
    "50000.00": "1.5",
    "49999.00": "0"
  },
  "asks": {
    "50001.00": "2.5"
  },
  "timestamp": 1715671200000
}
```

## Data Consistency Guarantees

| Data Store | Consistency Model | Use Case |
|------------|------------------|----------|
| PostgreSQL | ACID, Strong | Balances, Orders, Trades |
| TimescaleDB | Append-only, Strong | OHLCV, Historical data |
| Redis | Eventually Consistent | Session cache, Order book snapshots |
| NATS | At-least-once delivery | Event streaming |

## Data Retention Policy

| Data Type | Retention Period | Rationale |
|-----------|------------------|-----------|
| Order book snapshots | 7-30 days | Useful for debugging, analytics |
| Tick data | 30 days | Raw data expensive, aggregate for long-term |
| OHLCV (1m) | 1 year | Detailed charting |
| OHLCV (1h, 1d) | 5+ years | Historical analysis |
| Ledger entries | 7 years | Legal requirement |
| Audit logs | 7 years | Compliance |
| NATS events | 24-48 hours | Event replay window |
