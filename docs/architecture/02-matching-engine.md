# Matching Engine Core Design

## Overview

The matching engine is the heart of the exchange. It maintains in-memory order books and executes trades based on price-time priority.

## Data Structures

### Order Book Representation

```rust
use rust_decimal::Decimal;
use std::collections::{BTreeMap, BTreeSet, VecDeque};
use std::time::{Instant, UNIX_EPOCH};

/// Order book with separate bid and ask sides
pub struct OrderBook {
    pub symbol: String,
    /// Bids: sorted descending (best bid = highest price)
    pub bids: BTreeMap<Decimal, PriceLevel>,  // Use reverse() for max-first
    /// Asks: sorted ascending (best ask = lowest price)
    pub asks: BTreeMap<Decimal, PriceLevel>,
    /// For O(1) order lookup: order_id -> (side, price)
    pub order_index: HashMap<u64, (Side, Decimal)>,
    /// Sequence number for each event
    pub sequence: u64,
}

/// Orders at a specific price level
#[derive(Debug, Clone)]
pub struct PriceLevel {
    pub price: Decimal,
    /// FIFO queue of orders at this price
    pub orders: VecDeque<Order>,
    pub total_qty: Decimal,
}

/// Individual order
#[derive(Debug, Clone)]
pub struct Order {
    pub id: u64,
    pub user_id: u64,
    pub side: Side,
    pub qty: Decimal,
    pub filled_qty: Decimal,
    pub price: Option<Decimal>,  // None for market orders
    pub order_type: OrderType,
    pub time_in_force: TimeInForce,
    pub is_maker_only: bool,
    pub created_at: Instant,
    pub updated_at: Instant,
    pub fee_rate: Decimal,  // User's fee rate (decimal, e.g., 0.001 = 0.1%)
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Side {
    Bid,
    Ask,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OrderType {
    Limit,
    Market,
    IOC,     // Immediate-or-Cancel
    FOK,     // Fill-or-Kill
    PostOnly,// Maker-Only (reject if would cross spread)
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TimeInForce {
    GTC,     // Good-Til-Cancelled
    IOC,     // Immediate-or-Cancel
    FOK,     // Fill-or-Kill
}
```

### Trade Structure

```rust
#[derive(Debug, Clone)]
pub struct Trade {
    pub id: u64,
    pub symbol: String,
    pub maker_order_id: u64,
    pub taker_order_id: u64,
    pub maker_user_id: u64,
    pub taker_user_id: u64,
    pub price: Decimal,
    pub qty: Decimal,
    pub maker_fee: Decimal,
    pub taker_fee: Decimal,
    pub side: Side,  // From taker's perspective
    pub sequence: u64,
    pub timestamp: i64,
}
```

## Matching Algorithm

### Price-Time Priority (FIFO)

The matching engine follows strict price-time priority:

1. **Price priority**: Better prices execute first
   - Bids: Higher price is better
   - Asks: Lower price is better
2. **Time priority**: At same price, earlier orders execute first

### Main Matching Loop

```rust
impl OrderBook {
    /// Main matching function: submit order and return trades
    pub fn match_order(&mut self, mut order: Order) -> MatchingResult {
        let mut trades = Vec::new();
        let mut events = Vec::new();
        let original_qty = order.qty;

        match order.side {
            Side::Bid => self.match_bid(&mut order, &mut trades, &mut events),
            Side::Ask => self.match_ask(&mut order, &mut trades, &mut events),
        }

        // Handle remaining quantity (if any)
        if order.qty > Decimal::ZERO {
            match order.order_type {
                OrderType::Market | OrderType::IOC | OrderType::FOK => {
                    // Cancel remaining
                    events.push(OrderEvent::Cancelled {
                        order_id: order.id,
                        reason: if order.order_type == OrderType::Market {
                            CancelReason::MarketNoLiquidity
                        } else {
                            CancelReason::TimeInForce
                        },
                        remaining_qty: order.qty,
                    });
                }
                OrderType::Limit | OrderType::PostOnly => {
                    // Add to book
                    if let Err(e) = self.add_order(order.clone()) {
                        events.push(OrderEvent::Rejected {
                            order_id: order.id,
                            reason: e.to_string(),
                        });
                    } else {
                        events.push(OrderEvent::Accepted {
                            order_id: order.id,
                            side: order.side,
                            price: order.price.unwrap(),
                            qty: order.qty,
                        });
                    }
                }
            }
        } else {
            // Fully filled
            events.push(OrderEvent::Filled {
                order_id: order.id,
                filled_qty: original_qty,
            });
        }

        MatchingResult { trades, events }
    }

    fn match_bid(
        &mut self,
        taker: &mut Order,
        trades: &mut Vec<Trade>,
        events: &mut Vec<OrderEvent>,
    ) {
        let taker_price = taker.price;

        while taker.qty > Decimal::ZERO {
            // Get best ask
            let best_ask = match self.asks.keys().next().copied() {
                Some(p) => p,
                None => break,  // No asks remaining
            };

            // Check price compatibility
            match taker_price {
                Some(limit) if limit < best_ask => break,  // Price too low
                None => {},  // Market order: always accepts best price
                Some(_) => {},
            }

            // Get the price level
            let level = self.asks.get_mut(&best_ask).unwrap();
            let maker = level.orders.front_mut().unwrap();

            // Calculate fill quantity
            let fill_qty = taker.qty.min(maker.qty - maker.filled_qty);

            // Execute trade
            let trade = self.execute_trade(taker, maker, best_ask, fill_qty);
            trades.push(trade.clone());

            // Update quantities
            taker.qty -= fill_qty;
            maker.filled_qty += fill_qty;
            level.total_qty -= fill_qty;

            events.push(OrderEvent::Trade {
                order_id: taker.id,
                trade_id: trade.id,
                price: best_ask,
                qty: fill_qty,
                is_maker: false,
            });

            // Check if maker order is filled
            if maker.filled_qty >= maker.qty {
                level.orders.pop_front();
                events.push(OrderEvent::Filled {
                    order_id: maker.id,
                    filled_qty: maker.qty,
                });

                // Remove empty price level
                if level.orders.is_empty() {
                    self.asks.remove(&best_ask);
                }
            }
        }
    }

    fn match_ask(
        &mut self,
        taker: &mut Order,
        trades: &mut Vec<Trade>,
        events: &mut Vec<OrderEvent>,
    ) {
        let taker_price = taker.price;

        while taker.qty > Decimal::ZERO {
            // Get best bid
            let best_bid = match self.bids.keys().next().copied() {
                Some(p) => p,
                None => break,  // No bids remaining
            };

            // Check price compatibility
            match taker_price {
                Some(limit) if limit > best_bid => break,  // Price too high
                None => {},  // Market order: always accepts best price
                Some(_) => {},
            }

            // Get the price level
            let level = self.bids.get_mut(&best_bid).unwrap();
            let maker = level.orders.front_mut().unwrap();

            // Calculate fill quantity
            let fill_qty = taker.qty.min(maker.qty - maker.filled_qty);

            // Execute trade
            let trade = self.execute_trade(taker, maker, best_bid, fill_qty);
            trades.push(trade.clone());

            // Update quantities
            taker.qty -= fill_qty;
            maker.filled_qty += fill_qty;
            level.total_qty -= fill_qty;

            events.push(OrderEvent::Trade {
                order_id: taker.id,
                trade_id: trade.id,
                price: best_bid,
                qty: fill_qty,
                is_maker: false,
            });

            // Check if maker order is filled
            if maker.filled_qty >= maker.qty {
                level.orders.pop_front();
                events.push(OrderEvent::Filled {
                    order_id: maker.id,
                    filled_qty: maker.qty,
                });

                // Remove empty price level
                if level.orders.is_empty() {
                    self.bids.remove(&best_bid);
                }
            }
        }
    }

    fn execute_trade(
        &mut self,
        taker: &Order,
        maker: &Order,
        price: Decimal,
        qty: Decimal,
    ) -> Trade {
        self.sequence += 1;

        let trade_value = price * qty;
        let taker_fee_rate = taker.fee_rate;
        let maker_fee_rate = maker.fee_rate;

        Trade {
            id: self.next_trade_id(),
            symbol: self.symbol.clone(),
            maker_order_id: maker.id,
            taker_order_id: taker.id,
            maker_user_id: maker.user_id,
            taker_user_id: taker.user_id,
            price,
            qty,
            maker_fee: trade_value * maker_fee_rate,
            taker_fee: trade_value * taker_fee_rate,
            side: taker.side,
            sequence: self.sequence,
            timestamp: now_ms(),
        }
    }
}
```

### Order Placement

```rust
impl OrderBook {
    pub fn add_order(&mut self, order: Order) -> Result<(), EngineError> {
        let price = order.price.ok_or(EngineError::MarketOrderCannotBeAdded)?;

        // Post-only check: would cross spread?
        if order.is_maker_only {
            match order.side {
                Side::Bid => {
                    if let Some(&best_ask) = self.asks.keys().next() {
                        if price >= best_ask {
                            return Err(EngineError::WouldCrossSpread);
                        }
                    }
                }
                Side::Ask => {
                    if let Some(&best_bid) = self.bids.keys().next() {
                        if price <= best_bid {
                            return Err(EngineError::WouldCrossSpread);
                        }
                    }
                }
            }
        }

        // Add to appropriate side
        let levels = match order.side {
            Side::Bid => &mut self.bids,
            Side::Ask => &mut self.asks,
        };

        let level = levels.entry(price).or_insert_with(|| PriceLevel {
            price,
            orders: VecDeque::new(),
            total_qty: Decimal::ZERO,
        });

        level.orders.push_back(order.clone());
        level.total_qty += order.qty;

        // Update index for O(1) lookup
        self.order_index.insert(order.id, (order.side, price));

        Ok(())
    }
}
```

### Order Cancellation

```rust
impl OrderBook {
    pub fn cancel_order(&mut self, order_id: u64) -> Result<Order, EngineError> {
        // Find order
        let (side, price) = self.order_index
            .get(&order_id)
            .ok_or(EngineError::OrderNotFound)?;

        let levels = match side {
            Side::Bid => &mut self.bids,
            Side::Ask => &mut self.asks,
        };

        let level = levels.get_mut(price)
            .ok_or(EngineError::PriceLevelNotFound)?;

        // Find and remove order from queue
        let pos = level.orders
            .iter()
            .position(|o| o.id == order_id)
            .ok_or(EngineError::OrderNotFound)?;

        let order = level.orders.remove(pos).unwrap();
        level.total_qty -= order.qty;

        // Remove empty level
        if level.orders.is_empty() {
            levels.remove(price);
        }

        self.order_index.remove(&order_id);

        Ok(order)
    }

    /// Cancel all orders for a user (emergency)
    pub fn cancel_user_orders(&mut self, user_id: u64) -> Vec<Order> {
        let mut cancelled = Vec::new();

        // Cancel from bids
        self.bids.retain(|price, level| {
            level.orders.retain(|order| {
                if order.user_id == user_id {
                    cancelled.push(order.clone());
                    self.order_index.remove(&order.id);
                    false
                } else {
                    true
                }
            });
            !level.orders.is_empty()
        });

        // Cancel from asks
        self.asks.retain(|price, level| {
            level.orders.retain(|order| {
                if order.user_id == user_id {
                    cancelled.push(order.clone());
                    self.order_index.remove(&order.id);
                    false
                } else {
                    true
                }
            });
            !level.orders.is_empty()
        });

        cancelled
    }
}
```

## Order Type Handling

### Limit Orders

```rust
// Standard limit order: GTC by default
Order {
    order_type: OrderType::Limit,
    time_in_force: TimeInForce::GTC,
    is_maker_only: false,
    // ... rest of fields
}
```

**Behavior**:
1. Try to match immediately at existing prices
2. Remaining quantity added to book
3. Stays on book until filled or cancelled

### Market Orders

```rust
Order {
    order_type: OrderType::Market,
    price: None,
    time_in_force: TimeInForce::IOC,  // Always IOC
    // ... rest of fields
}
```

**Behavior**:
1. Cross the book immediately
2. Fill as much as possible
3. Unfilled portion is cancelled (never goes on book)
4. No partial fills if book is empty

### Immediate-or-Cancel (IOC)

```rust
Order {
    order_type: OrderType::IOC,
    time_in_force: TimeInForce::IOC,
    // ... rest of fields
}
```

**Behavior**:
1. Try to match immediately
2. Any unfilled portion is cancelled
3. Never goes on book

### Fill-or-Kill (FOK)

```rust
Order {
    order_type: OrderType::FOK,
    time_in_force: TimeInForce::FOK,
    // ... rest of fields
}
```

**Behavior**:
1. Must fill ENTIRE quantity immediately
2. If cannot fill completely, order is rejected
3. No partial fills

```rust
fn match_fok_order(&mut self, mut order: Order) -> MatchingResult {
    let original_qty = order.qty;

    // First pass: check if we can fill entirely
    let available_qty = self.calculate_available_qty(order.side, order.price);
    if available_qty < order.qty {
        return MatchingResult {
            trades: vec![],
            events: vec![OrderEvent::Rejected {
                order_id: order.id,
                reason: "Insufficient liquidity for FOK".to_string(),
            }],
        };
    }

    // Second pass: execute matches
    // ... (same as normal matching)
}
```

### Post-Only (Maker-Only)

```rust
Order {
    order_type: OrderType::PostOnly,
    is_maker_only: true,
    // ... rest of fields
}
```

**Behavior**:
1. Never crosses the spread
2. If order would match immediately, it's rejected
3. Only goes on book as maker

```rust
fn check_post_only(&self, side: Side, price: Decimal) -> bool {
    match side {
        Side::Bid => {
            if let Some(&best_ask) = self.asks.keys().next() {
                return price < best_ask;  // Must be below best ask
            }
        }
        Side::Ask => {
            if let Some(&best_bid) = self.bids.keys().next() {
                return price > best_bid;  // Must be above best bid
            }
        }
    }
    true
}
```

### Iceberg Orders (Hidden Liquidity)

```rust
pub struct IcebergOrder {
    pub visible_qty: Decimal,
    pub hidden_qty: Decimal,
    pub total_qty: Decimal,
    pub peak_size: Decimal,
}

impl OrderBook {
    /// Handle iceberg order: only show `peak_size` at a time
    pub fn add_iceberg_order(&mut self, order: IcebergOrder) {
        // Add visible portion to book
        let visible_qty = order.visible_qty.min(order.peak_size);

        // When visible portion is fully filled, automatically replenish
        // from hidden quantity
    }
}
```

**Behavior**:
1. Only `peak_size` visible on order book
2. When filled, automatically replenish from hidden qty
3. Prevents large orders from moving the market

## Memory Management

### Memory Pools

```rust
use std::alloc::{Allocator, Global};

/// Custom allocator for order objects
struct OrderPool {
    // Pre-allocated pool of Order objects
    orders: Vec<Order>,
    free_list: Vec<usize>,
}

impl OrderPool {
    fn new(capacity: usize) -> Self {
        Self {
            orders: (0..capacity).map(|_| Order::default()).collect(),
            free_list: (0..capacity).collect(),
        }
    }

    fn allocate(&mut self) -> Option<&mut Order> {
        let idx = self.free_list.pop()?;
        Some(&mut self.orders[idx])
    }

    fn deallocate(&mut self, order: &mut Order) {
        // Find index and add to free list
    }
}
```

### Snapshotting

```rust
impl OrderBook {
    /// Serialize order book state for snapshot
    pub fn to_snapshot(&self) -> OrderBookSnapshot {
        OrderBookSnapshot {
            symbol: self.symbol.clone(),
            sequence: self.sequence,
            bids: self.bids.iter()
                .map(|(price, level)| (*price, level.clone()))
                .collect(),
            asks: self.asks.iter()
                .map(|(price, level)| (*price, level.clone()))
                .collect(),
            timestamp: now_ms(),
        }
    }

    /// Restore from snapshot
    pub fn from_snapshot(snapshot: OrderBookSnapshot) -> Self {
        let mut book = OrderBook::new(snapshot.symbol);
        book.bids = snapshot.bids;
        book.asks = snapshot.asks;
        book.sequence = snapshot.sequence;
        book
    }
}
```

## Performance Optimizations

### 1. Avoid HashMap Iteration

```rust
// BAD: O(n) lookup
for order in orders {
    if order.id == target_id { ... }
}

// GOOD: O(1) lookup via index
if let Some(order) = self.order_index.get(&target_id) { ... }
```

### 2. Batch Price Level Updates

```rust
// Aggregate multiple updates at same price level
// before publishing to reduce NATS messages
```

### 3. Lock-Free Single Thread

```rust
// Each shard runs on a single thread
// No locks needed within shard
// Only cross-thread communication via channels
```

### 4. Cache-Friendly Data Layout

```rust
// Use Vec instead of HashMap for hot data
// Store orders contiguously in memory
// Use u64 instead of String for IDs
```

## Sharding Strategy

### Symbol Assignment

```rust
pub struct ShardConfig {
    pub shards: Vec<Shard>,
}

pub struct Shard {
    pub id: u32,
    pub symbols: Vec<String>,
    pub thread_handle: Option<JoinHandle<()>>,
}

impl ShardConfig {
    /// Assign symbol to shard (consistent hashing)
    pub fn get_shard_for_symbol(&self, symbol: &str) -> u32 {
        let hash = crc32::checksum_ieee(symbol.as_bytes());
        (hash as u32) % self.shards.len() as u32
    }
}
```

### Hot Symbol Handling

For extremely high-volume symbols (e.g., BTC/USDT), consider:

1. **Dedicated shard**: Single symbol per shard
2. **Multiple instances**: Run multiple engine instances with same symbol (need leader election)
3. **Order splitting**: Split order book by price ranges (advanced)

## Latency Targets

| Operation | Target | 50th | 95th | 99th |
|-----------|--------|------|------|------|
| Add limit order | < 100μs | 30μs | 60μs | 90μs |
| Market order match | < 200μs | 50μs | 120μs | 180μs |
| Cancel order | < 50μs | 15μs | 30μs | 40μs |
| Snapshot serialize | < 1ms | 300μs | 600μs | 900μs |

## Testing Strategy

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_price_time_priority() {
        // Add multiple orders at same price
        // Verify FIFO execution
    }

    #[test]
    fn test_market_order_no_liquidity() {
        // Submit market order to empty book
        // Verify order is cancelled, no trades
    }

    #[test]
    fn test_fok_partial_fill_rejection() {
        // Submit FOK larger than available
        // Verify complete rejection
    }

    #[test]
    fn test_post_only_cross_spread() {
        // Submit post-only that would cross
        // Verify rejection
    }
}
```

### Property-Based Tests

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_matching_invariants(
        bids in vec(order_strategy(), 1..100),
        asks in vec(order_strategy(), 1..100),
        market_order in order_strategy(),
    ) {
        // Invariant 1: Total qty preserved
        // Invariant 2: No negative balances
        // Invariant 3: Price never worse than limit
    }
}
```

### Fuzz Testing

```rust
// Use libFuzzer to find edge cases
// Random order sequences
// Crash on assertion failure
```
