# Security & Risk Management

## Overview

Security is paramount when handling real user funds. This document outlines the security measures across all system layers.

## Threat Model

| Threat | Vector | Impact | Mitigation |
|--------|--------|--------|------------|
| Unauthorized access | Stolen API keys, session hijacking | Fund theft | JWT, rate limiting, IP allowlist |
| Insider attack | Malicious employee | Fund theft, data theft | Multi-party approval, audit logs |
| DDoS | Flood of requests | Service unavailability | Rate limiting, auto-scaling, CDN |
| Market manipulation | Spoofing, wash trading | Market distortion | Surveillance, circuit breakers |
| Double spend | Race conditions | Fund loss | Database constraints, idempotency |
| Price manipulation | Oracle attacks | Incorrect pricing | Price bands, circuit breakers |
| Self-trading | User trades against self | Fee circumvention | Self-trade prevention |

## Authentication & Authorization

### JWT-Based Authentication

```rust
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Validation};

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,  // User ID
    pub exp: usize,   // Expiration
    pub iat: usize,   // Issued at
    pub jti: String,  // JWT ID (for revocation)
    pub tier: String, // Fee tier
}

pub struct AuthService {
    encoding_key: EncodingKey,
    decoding_key: DecodingKey,
    redis: RedisPool,
}

impl AuthService {
    pub async fn generate_token(&self, user_id: &str, tier: &str) -> Result<String, Error> {
        let now = Utc::now();
        let exp = now + Duration::hours(24);

        let claims = Claims {
            sub: user_id.to_string(),
            exp: exp.timestamp() as usize,
            iat: now.timestamp() as usize,
            jti: Uuid::new_v4().to_string(),
            tier: tier.to_string(),
        };

        let token = encode(&jsonwebtoken::Header::default(), &claims, &self.encoding_key)?;

        // Store JTI in Redis for revocation
        self.redis
            .set(format!("jti:{}", claims.jti), user_id, 86400)
            .await?;

        Ok(token)
    }

    pub async fn validate_token(&self, token: &str) -> Result<Claims, Error> {
        let claims = decode::<Claims>(token, &self.decoding_key, &Validation::default())?;

        // Check if token is revoked
        let revoked = self
            .redis
            .exists(format!("revoked:{}", claims.claims.jti))
            .await?;

        if revoked {
            return Err(Error::TokenRevoked);
        }

        // Refresh TTL
        self.redis
            .expire(format!("jti:{}", claims.claims.jti), 86400)
            .await?;

        Ok(claims.claims)
    }

    pub async fn revoke_token(&self, jti: &str) -> Result<(), Error> {
        self.redis
            .set(format!("revoked:{}", jti), "1", 86400)
            .await?;
        Ok(())
    }
}
```

### API Key Authentication

```rust
#[derive(Debug, Clone)]
pub struct ApiKey {
    pub id: String,
    pub user_id: u64,
    pub key_hash: String,
    pub secret_hash: String,
    pub permissions: Vec<Permission>,
    pub ip_whitelist: Vec<IpAddr>,
    pub rate_limit: u32,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Permission {
    PlaceOrder,
    CancelOrder,
    ViewBalance,
    ViewOrders,
    ViewTrades,
    Withdraw,
}

pub struct ApiKeyService {
    db: PgPool,
    redis: RedisPool,
}

impl ApiKeyService {
    pub async fn authenticate(
        &self,
        key_id: &str,
        signature: &str,
        timestamp: i64,
        payload: &[u8],
    ) -> Result<ApiKey, Error> {
        // Check timestamp (prevent replay attacks)
        let now = Utc::now().timestamp();
        if (now - timestamp).abs() > 30 {
            return Err(Error::TimestampOutOfRange);
        }

        // Get API key from cache or DB
        let api_key = self.get_api_key(key_id).await?;

        // Verify signature
        let expected = hmac_signature(&api_key.secret_hash, timestamp, payload);
        if !constant_time_eq(&expected, signature) {
            return Err(Error::InvalidSignature);
        }

        // Check IP whitelist
        if !api_key.ip_whitelist.is_empty() {
            let client_ip = self.get_client_ip().await?;
            if !api_key.ip_whitelist.contains(&client_ip) {
                return Err(Error::IpNotAllowed);
            }
        }

        Ok(api_key)
    }
}
```

## Pre-Trade Risk Checks

### Order Validation

```rust
pub struct RiskChecker {
    db: PgPool,
    redis: RedisPool,
    config: RiskConfig,
}

#[derive(Debug, Clone)]
pub struct RiskConfig {
    pub max_order_value: Decimal,
    pub max_orders_per_second: u32,
    pub max_orders_per_minute: u32,
    pub max_open_orders: u32,
    pub max_price_deviation_percent: Decimal,
}

impl RiskChecker {
    pub async fn validate_order(&self, user_id: u64, order: &Order) -> Result<(), RiskError> {
        // 1. Check user exists and active
        self.check_user_active(user_id).await?;

        // 2. Check trading pair is active
        self.check_pair_active(order.pair_id).await?;

        // 3. Validate order parameters
        self.validate_order_params(order).await?;

        // 4. Check balance availability
        self.check_balance(user_id, order).await?;

        // 5. Check rate limits
        self.check_rate_limits(user_id).await?;

        // 6. Check price bands
        self.check_price_bands(order).await?;

        // 7. Check open order count
        self.check_open_order_count(user_id, order.pair_id).await?;

        Ok(())
    }

    async fn check_balance(&self, user_id: u64, order: &Order) -> Result<(), RiskError> {
        let pair = self.get_trading_pair(order.pair_id).await?;

        match order.side {
            Side::Bid => {
                // Buying: need quote asset
                let required = order.price.unwrap_or(Decimal::ZERO) * order.qty;
                let balance = self.get_balance(user_id, pair.quote_asset_id).await?;

                if balance.available < required {
                    return Err(RiskError::InsufficientBalance {
                        asset: pair.quote_asset_id,
                        required,
                        available: balance.available,
                    });
                }
            }
            Side::Ask => {
                // Selling: need base asset
                let balance = self.get_balance(user_id, pair.base_asset_id).await?;

                if balance.available < order.qty {
                    return Err(RiskError::InsufficientBalance {
                        asset: pair.base_asset_id,
                        required: order.qty,
                        available: balance.available,
                    });
                }
            }
        }

        Ok(())
    }

    async fn check_price_bands(&self, order: &Order) -> Result<(), RiskError> {
        let price = order.price.ok_or(RiskError::MarketOrderNoPrice)?;

        // Get current market price (from recent trades)
        let market_price = self.get_market_price(order.pair_id).await?;

        // Check if price is within allowed deviation
        let deviation = ((price - market_price).abs() / market_price) * Decimal::from(100);

        if deviation > self.config.max_price_deviation_percent {
            return Err(RiskError::PriceOutsideBand {
                price,
                market_price,
                deviation,
            });
        }

        // Check against trading pair min/max
        let pair = self.get_trading_pair(order.pair_id).await?;
        if price < pair.min_price || price > pair.max_price {
            return Err(RiskError::PriceOutOfRange {
                price,
                min: pair.min_price,
                max: pair.max_price,
            });
        }

        Ok(())
    }
}
```

## Self-Trade Prevention

### Prevention Modes

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SelfTradePrevention {
    /// Cancel the newer order (taker)
    CancelNewest,
    /// Cancel the older order (maker)
    CancelOldest,
    /// Cancel both orders
    CancelBoth,
    /// Allow the trade (default, dangerous)
    Allow,
}

pub struct SelfTradeChecker {
    redis: RedisPool,
}

impl SelfTradeChecker {
    pub async fn check_and_handle(
        &self,
        maker_order: &Order,
        taker_order: &Order,
        mode: SelfTradePrevention,
    ) -> Result<TradeAction, Error> {
        // Check if same user
        if maker_order.user_id != taker_order.user_id {
            return Ok(TradeAction::Proceed);
        }

        match mode {
            SelfTradePrevention::CancelNewest => {
                return Ok(TradeAction::CancelOrder(taker_order.id));
            }
            SelfTradePrevention::CancelOldest => {
                return Ok(TradeAction::CancelOrder(maker_order.id));
            }
            SelfTradePrevention::CancelBoth => {
                return Ok(TradeAction::CancelBothOrders(maker_order.id, taker_order.id));
            }
            SelfTradePrevention::Allow => {
                return Ok(TradeAction::Proceed);
            }
        }
    }
}

pub enum TradeAction {
    Proceed,
    CancelOrder(u64),
    CancelBothOrders(u64, u64),
}
```

## Circuit Breakers

### Price Circuit Breaker

```rust
pub struct CircuitBreaker {
    threshold_percent: Decimal,
    window_duration: Duration,
    measurements: VecDeque<PriceMeasurement>,
}

#[derive(Debug, Clone)]
struct PriceMeasurement {
    price: Decimal,
    volume: Decimal,
    timestamp: Instant,
}

impl CircuitBreaker {
    pub fn new(threshold_percent: Decimal) -> Self {
        Self {
            threshold_percent,
            window_duration: Duration::from_secs(60),
            measurements: VecDeque::with_capacity(1000),
        }
    }

    pub fn check_trade(&mut self, price: Decimal, volume: Decimal) -> Result<(), CircuitBreakerError> {
        // Add measurement
        self.measurements.push_back(PriceMeasurement {
            price,
            volume,
            timestamp: Instant::now(),
        });

        // Remove old measurements
        let cutoff = Instant::now() - self.window_duration;
        while self
            .measurements
            .front()
            .map(|m| m.timestamp < cutoff)
            .unwrap_or(false)
        {
            self.measurements.pop_front();
        }

        // Check if we have enough data
        if self.measurements.len() < 10 {
            return Ok(());
        }

        // Calculate VWAP
        let total_value: Decimal = self
            .measurements
            .iter()
            .map(|m| m.price * m.volume)
            .sum();
        let total_volume: Decimal = self.measurements.iter().map(|m| m.volume).sum();
        let vwap = total_value / total_volume;

        // Check deviation
        let deviation = ((price - vwap).abs() / vwap) * Decimal::from(100);

        if deviation > self.threshold_percent {
            warn!(
                "Circuit breaker triggered: price {} deviates {}% from VWAP {}",
                price, deviation, vwap
            );
            return Err(CircuitBreakerError::PriceDeviationExceeded {
                price,
                vwap,
                deviation,
            });
        }

        Ok(())
    }
}
```

### Volume Circuit Breaker

```rust
pub struct VolumeCircuitBreaker {
    max_volume_per_second: Decimal,
    recent_volume: VecDeque<(Instant, Decimal)>,
}

impl VolumeCircuitBreaker {
    pub fn check_volume(&mut self, volume: Decimal) -> Result<(), CircuitBreakerError> {
        let now = Instant::now();

        // Add current volume
        self.recent_volume.push_back((now, volume));

        // Remove old measurements (older than 1 second)
        let cutoff = now - Duration::from_secs(1);
        while self
            .recent_volume
            .front()
            .map(|(t, _)| *t < cutoff)
            .unwrap_or(false)
        {
            self.recent_volume.pop_front();
        }

        // Sum volume in last second
        let total_volume: Decimal = self.recent_volume.iter().map(|(_, v)| *v).sum();

        if total_volume > self.max_volume_per_second {
            warn!(
                "Volume circuit breaker: {}/{} exceeded",
                total_volume, self.max_volume_per_second
            );
            return Err(CircuitBreakerError::VolumeExceeded {
                total_volume,
                max_volume: self.max_volume_per_second,
            });
        }

        Ok(())
    }
}
```

## Rate Limiting

### Token Bucket Algorithm

```rust
pub struct RateLimiter {
    redis: RedisPool,
}

impl RateLimiter {
    pub async fn check_rate_limit(
        &self,
        key: &str,
        limit: u32,
        window: Duration,
    ) -> Result<(), RateLimitError> {
        let now = Utc::now().timestamp();
        let window_start = now - window.as_secs() as i64;

        // Clean old entries
        self.redis
            .zrembyscore(key, 0, window_start)
            .await?;

        // Count current requests
        let count = self.redis.zcard(key).await? as u32;

        if count >= limit {
            return Err(RateLimitError::LimitExceeded {
                limit,
                window,
                retry_after: self.get_retry_after(key, window).await?,
            });
        }

        // Add current request
        self.redis.zadd(key, now, now.to_string()).await?;

        // Set expiration
        self.redis.expire(key, window.as_secs() as usize).await?;

        Ok(())
    }

    async fn get_retry_after(&self, key: &str, window: Duration) -> Result<Duration, Error> {
        let oldest = self
            .redis
            .zrange(key, 0, 0)
            .await?
            .into_iter()
            .next()
            .and_then(|s| s.parse::<i64>().ok());

        match oldest {
            Some(ts) => {
                let elapsed = Utc::now().timestamp() - ts;
                let remaining = window.as_secs() as i64 - elapsed;
                Ok(Duration::from_secs(remaining.max(0) as u64))
            }
            None => Ok(Duration::from_secs(0)),
        }
    }
}
```

### Rate Limit Tiers

```rust
#[derive(Debug, Clone)]
pub struct RateLimitTier {
    pub requests_per_second: u32,
    pub orders_per_minute: u32,
    pub websocket_connections: u32,
}

pub const RATE_LIMITS: &[(&str, RateLimitTier)] = &[
    ("free", RateLimitTier {
        requests_per_second: 10,
        orders_per_minute: 60,
        websocket_connections: 5,
    }),
    ("basic", RateLimitTier {
        requests_per_second: 50,
        orders_per_minute: 300,
        websocket_connections: 20,
    }),
    ("vip_1", RateLimitTier {
        requests_per_second: 100,
        orders_per_minute: 1000,
        websocket_connections: 50,
    }),
    ("vip_2", RateLimitTier {
        requests_per_second: 200,
        orders_per_minute: 2000,
        websocket_connections: 100,
    }),
    ("vip_3", RateLimitTier {
        requests_per_second: 500,
        orders_per_minute: 5000,
        websocket_connections: 200,
    }),
    ("institutional", RateLimitTier {
        requests_per_second: 1000,
        orders_per_minute: 10000,
        websocket_connections: 500,
    }),
];
```

## Input Sanitization

### Order Parameter Validation

```rust
use validator::Validate;

#[derive(Debug, Deserialize, Validate)]
pub struct PlaceOrderRequest {
    #[validate(range(min = 1, message = "Invalid pair ID"))]
    pub pair_id: u16,

    #[validate(custom(function = "validate_side"))]
    pub side: String,

    #[validate(custom(function = "validate_order_type"))]
    pub order_type: String,

    #[validate(custom(function = "validate_price"))]
    pub price: Option<String>,

    #[validate(custom(function = "validate_quantity"))]
    pub quantity: String,

    #[validate(custom(function = "validate_time_in_force"))]
    pub time_in_force: Option<String>,

    pub post_only: Option<bool>,
}

fn validate_side(side: &str) -> Result<(), validator::ValidationError> {
    match side {
        "buy" | "sell" | "bid" | "ask" => Ok(()),
        _ => Err(validator::ValidationError::new("invalid side")),
    }
}

fn validate_quantity(qty: &str) -> Result<(), validator::ValidationError> {
    let decimal = qty.parse::<Decimal>()
        .map_err(|_| validator::ValidationError::new("invalid quantity format"))?;

    if decimal <= Decimal::ZERO {
        return Err(validator::ValidationError::new("quantity must be positive"));
    }

    if decimal.scale() > 18 {
        return Err(validator::ValidationError::new("too many decimal places"));
    }

    Ok(())
}

pub fn sanitize_order_input(
    user_id: u64,
    req: PlaceOrderRequest,
) -> Result<Order, ValidationError> {
    req.validate()?;

    // Convert and validate
    let side = match req.side.as_str() {
        "buy" | "bid" => Side::Bid,
        "sell" | "ask" => Side::Ask,
        _ => return Err(ValidationError::InvalidSide),
    };

    let price = match req.order_type.as_str() {
        "market" => None,
        _ => Some(
            req.price
                .ok_or(ValidationError::MissingPrice)?
                .parse()
                .map_err(|_| ValidationError::InvalidPrice)?,
        ),
    };

    let qty = req
        .quantity
        .parse()
        .map_err(|_| ValidationError::InvalidQuantity)?;

    if qty <= Decimal::ZERO {
        return Err(ValidationError::InvalidQuantity);
    }

    Ok(Order {
        id: 0,
        user_id,
        side,
        order_type: parse_order_type(&req.order_type)?,
        price,
        qty,
        ..Default::default()
    })
}
```

## Audit Logging

```rust
#[derive(Debug, Serialize)]
pub struct AuditLog {
    pub id: Uuid,
    pub timestamp: i64,
    pub user_id: Option<u64>,
    pub api_key_id: Option<String>,
    pub ip_addr: IpAddr,
    pub user_agent: Option<String>,
    pub action: AuditAction,
    pub resource_type: String,
    pub resource_id: String,
    pub details: serde_json::Value,
    pub status: AuditStatus,
}

#[derive(Debug, Serialize)]
pub enum AuditAction {
    OrderPlaced,
    OrderCancelled,
    OrderFilled,
    BalanceLocked,
    BalanceUnlocked,
    WithdrawalRequested,
    WithdrawalApproved,
    WithdrawalRejected,
    DepositConfirmed,
    Login,
    Logout,
    ApiKeyCreated,
    ApiKeyDeleted,
}

#[derive(Debug, Serialize)]
pub enum AuditStatus {
    Success,
    Failed,
    Rejected,
}

pub struct AuditLogger {
    db: PgPool,
}

impl AuditLogger {
    pub async fn log(&self, entry: AuditLog) -> Result<(), Error> {
        query!(
            r#"
            INSERT INTO audit_logs (
                id, timestamp, user_id, api_key_id, ip_addr, user_agent,
                action, resource_type, resource_id, details, status
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
            "#,
            entry.id,
            entry.timestamp,
            entry.user_id,
            entry.api_key_id,
            entry.ip_addr.to_string(),
            entry.user_agent,
            serde_json::to_string(&entry.action)?,
            entry.resource_type,
            entry.resource_id,
            entry.details,
            serde_json::to_string(&entry.status)?
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }
}
```

## Database Security

### Constraints & Triggers

```sql
-- Prevent negative balances
CREATE OR REPLACE FUNCTION check_non_negative_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.available < 0 OR NEW.locked < 0 THEN
        RAISE EXCEPTION 'Balance cannot be negative';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_balance_non_negative
    BEFORE INSERT OR UPDATE ON balances
    FOR EACH ROW
    EXECUTE FUNCTION check_non_negative_balance();

-- Prevent double-spending on order placement
CREATE OR REPLACE FUNCTION check_balance_lock()
RETURNS TRIGGER AS $$
DECLARE
    avail NUMERIC;
BEGIN
    SELECT available INTO avail
    FROM balances
    WHERE wallet_id = (SELECT wallet_id FROM users WHERE id = NEW.user_id)
        AND asset_id = (
            CASE NEW.side
                WHEN 'bid' THEN (SELECT quote_asset_id FROM trading_pairs WHERE id = NEW.pair_id)
                ELSE (SELECT base_asset_id FROM trading_pairs WHERE id = NEW.pair_id)
            END
        );

    IF avail < (NEW.price * NEW.qty) THEN
        RAISE EXCEPTION 'Insufficient balance for order';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER check_order_balance
    BEFORE INSERT ON orders
    FOR EACH ROW
    EXECUTE FUNCTION check_balance_lock();
```

### Row-Level Security

```sql
-- Enable RLS on sensitive tables
ALTER TABLE balances ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE ledger_entries ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own data
CREATE POLICY user_isolation ON balances
    FOR ALL
    USING (wallet_id IN (SELECT id FROM wallets WHERE user_id = current_user_id()));

CREATE POLICY user_orders ON orders
    FOR ALL
    USING (user_id = current_user_id());
```

## Network Security

### IP Whitelisting

```rust
pub struct IpWhitelist {
    redis: RedisPool,
}

impl IpWhitelist {
    pub async fn is_allowed(&self, user_id: u64, ip: IpAddr) -> Result<bool, Error> {
        let key = format!("whitelist:{}", user_id);

        // Check if whitelist is enabled
        let enabled = self.redis.exists(&key).await?;

        if !enabled {
            return Ok(true); // No whitelist restriction
        }

        // Check if IP is in whitelist
        let is_whitelisted = self
            .redis
            .sismember(&key, ip.to_string())
            .await?;

        Ok(is_whitelisted)
    }

    pub async fn add_ip(&self, user_id: u64, ip: IpAddr) -> Result<(), Error> {
        let key = format!("whitelist:{}", user_id);
        self.redis.sadd(&key, ip.to_string()).await?;
        Ok(())
    }
}
```

### DDoS Protection

```rust
pub struct DdosProtection {
    redis: RedisPool,
    max_requests_per_ip: u32,
}

impl DdosProtection {
    pub async fn check_ip(&self, ip: IpAddr) -> Result<(), Error> {
        let key = format!("ddos:{}", ip);

        // Increment counter
        let count = self.redis.incr(&key).await?;

        // Set expiration on first request
        if count == 1 {
            self.redis.expire(&key, 60).await?;
        }

        if count > self.max_requests_per_ip {
            warn!("DDoS detected from IP {}: {} requests/minute", ip, count);
            return Err(Error::RateLimitExceeded);
        }

        Ok(())
    }
}
```

## Security Monitoring

### Anomaly Detection

```rust
pub struct AnomalyDetector {
    db: PgPool,
    config: AnomalyConfig,
}

#[derive(Debug, Clone)]
pub struct AnomalyConfig {
    pub unusual_login_location: bool,
    pub large_withdrawal_threshold: Decimal,
    pub rapid_order_placement_threshold: u32,
}

impl AnomalyDetector {
    pub async fn check_withdrawal(&self, user_id: u64, amount: Decimal) -> Result<Vec<Alert>, Error> {
        let mut alerts = vec![];

        // Check against threshold
        if amount > self.config.large_withdrawal_threshold {
            alerts.push(Alert {
                severity: AlertSeverity::High,
                message: format!("Large withdrawal: {}", amount),
                user_id: Some(user_id),
            });
        }

        // Check against typical pattern
        let typical_amount = self.get_typical_withdrawal_amount(user_id).await?;
        let ratio = amount / typical_amount;

        if ratio > Decimal::from(10) {
            alerts.push(Alert {
                severity: AlertSeverity::Medium,
                message: format!("Withdrawal {}x typical amount", ratio),
                user_id: Some(user_id),
            });
        }

        Ok(alerts)
    }

    pub async fn check_login(&self, user_id: u64, ip: IpAddr) -> Result<Vec<Alert>, Error> {
        let mut alerts = vec![];

        // Check if IP is in unusual location
        let typical_locations = self.get_typical_login_locations(user_id).await?;
        let current_location = geoip_lookup(ip).await?;

        if !typical_locations.contains(&current_location) {
            alerts.push(Alert {
                severity: AlertSeverity::Medium,
                message: format!("Login from unusual location: {:?}", current_location),
                user_id: Some(user_id),
            });
        }

        Ok(alerts)
    }
}
```

## Compliance

### KYC/AML Integration

```rust
pub struct KycService {
    client: reqwest::Client,
    provider_url: String,
}

impl KycService {
    pub async fn check_kyc_status(&self, user_id: u64) -> Result<KycStatus, Error> {
        let response = self
            .client
            .get(format!("{}/users/{}/kyc", self.provider_url, user_id))
            .send()
            .await?
            .json::<KycResponse>()
            .await?;

        Ok(response.status)
    }

    pub async fn require_kyc_for_withdrawal(&self, user_id: u64, amount: Decimal) -> Result<(), Error> {
        let status = self.check_kyc_status(user_id).await?;

        match status {
            KycStatus::Verified => Ok(()),
            KycStatus::Pending => Err(Error::KycPending),
            KycStatus::NotStarted => Err(Error::KycRequired),
            KycStatus::Rejected => Err(Error::KycRejected),
        }
    }
}
```

### Travel Rule (FATF)

```rust
#[derive(Debug, Serialize)]
pub struct TravelRuleInfo {
    pub originator: TravelRuleParty,
    pub beneficiary: TravelRuleParty,
    pub amount: Decimal,
    pub asset: String,
    pub transaction_id: String,
}

#[derive(Debug, Serialize)]
pub struct TravelRuleParty {
    pub name: String,
    pub account: String,
    pub address: String,
}

pub async fn send_travel_rule_notification(info: TravelRuleInfo) -> Result<(), Error> {
    // Send toVASP (Virtual Asset Service Provider) protocol
    let client = reqwest::Client::new();
    client
        .post("https://vasp.example.com/travel-rule")
        .json(&info)
        .send()
        .await?;

    Ok(())
}
```
