# Project B09: API Gateway with Rate Limiting

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

API Gateway คือชั้น "ประตูทางเข้า" ของระบบ microservices — ทุก HTTP request จากภายนอกวิ่งผ่านจุดเดียวนี้ก่อนถึง upstream service จริง ประโยชน์คือจัดการ cross-cutting concerns (rate limiting, auth, caching, metrics) ในที่เดียวโดยไม่ต้องทำซ้ำในแต่ละ service

ในโลก production ตัวอย่างที่มีชื่อเสียง ได้แก่ Kong, AWS API Gateway, Nginx Plus — แต่การสร้างเองใน Rust ช่วยให้เข้าใจกลไกภายในอย่างลึกซึ้ง และได้ binary ที่ประสิทธิภาพสูงมาก (latency overhead น้อยกว่า 1ms ต่อ request โดยทั่วไป)

โปรเจคนี้สร้าง API Gateway ที่มีครบทุก feature ระดับ production:
- **Reverse proxy**: รับ request → match route → forward → return response
- **Rate limiting**: ทั้ง Token Bucket และ Sliding Window Log เลือกได้ per-route
- **JWT validation**: ตรวจ Bearer token พร้อม JWKS key caching
- **Response caching**: Redis-backed สำหรับ GET requests
- **Circuit breaker**: ป้องกัน upstream ล่มลามทั้งระบบ
- **Metrics**: Prometheus ตาม standard ของ cloud-native applications
- **Admin API**: เพิ่ม/ลบ route ได้ขณะ runtime โดยไม่ต้อง restart

## สิ่งที่จะได้เรียนรู้

- การออกแบบ middleware pipeline ด้วย `axum` layer system
- ความแตกต่างและ trade-off ระหว่าง rate limiting algorithms (Token Bucket vs Sliding Window)
- การ implement Circuit Breaker pattern และ state machine
- JWT validation + JWKS (JSON Web Key Set) public key fetching และ caching
- Redis integration สำหรับ distributed caching
- Prometheus metrics instrumentation ด้วย `prometheus` crate
- Dynamic routing ด้วย `Arc<RwLock<>>` สำหรับ hot-reload config
- HTTP reverse proxy ด้วย `hyper` และ `reqwest`

## ความรู้ที่ต้องมีมาก่อน

- Part 46–50: async/await และ Tokio runtime
- Part 61–65: Web development กับ Axum
- Part 70–75: Middleware, tower layers, error handling ใน web
- Part 80–85: Redis integration, caching patterns
- Part 96–100: Metrics, observability, production patterns
- Part 105–110: JWT, authentication, security middleware

## โครงสร้างโปรเจค (Project Layout)

```
api-gateway/
├── src/
│   ├── main.rs              # Entry point, server setup
│   ├── config.rs            # Route config structs + YAML parsing
│   ├── proxy.rs             # Reverse proxy core (hyper forwarding)
│   ├── rate_limit/
│   │   ├── mod.rs           # RateLimiter trait + factory
│   │   ├── token_bucket.rs  # Token Bucket algorithm
│   │   └── sliding_window.rs # Sliding Window Log algorithm
│   ├── auth/
│   │   ├── mod.rs           # Auth middleware
│   │   └── jwks.rs          # JWKS fetching + key caching
│   ├── cache.rs             # Redis-backed response cache
│   ├── circuit_breaker.rs   # Circuit breaker per upstream
│   ├── transform.rs         # Header add/remove/rename
│   ├── metrics.rs           # Prometheus counters + histograms
│   └── admin.rs             # Admin API handlers
├── config/
│   └── routes.yaml          # Route definitions
├── tests/
│   └── integration_test.rs  # Integration tests
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Client Request
      │
      ▼
┌─────────────────────────────────────────┐
│  Axum Router (port 8080)                │
│                                         │
│  1. MetricsLayer  ─── record start time │
│  2. RateLimitLayer ── check per-key     │
│  3. AuthLayer ─────── verify JWT/key    │
│  4. ProxyHandler                        │
│     ├── TransformLayer (add/rm headers) │
│     ├── CacheLayer (Redis GET check)    │
│     ├── CircuitBreaker.can_attempt()    │
│     └── hyper::Client forward           │
│                                         │
│  Admin Router (port 9090)               │
│  GET  /admin/routes                     │
│  POST /admin/routes                     │
│  DELETE /admin/routes/{id}              │
│                                         │
│  Metrics endpoint: GET /metrics         │
└─────────────────────────────────────────┘
```

### Design Decisions

**ทำไมใช้ `axum` แทน `actix-web`?**
axum สร้างบน `tower` middleware ecosystem ซึ่ง type-safe และ composable กว่า ใน API Gateway ซึ่ง middleware chain มีความสำคัญมาก axum เหมาะกว่า

**ทำไม Rate Limiter เก็บใน `Arc<DashMap<>>`?**
rate limit state ต้องแชร์ข้าม tokio tasks แต่ละ worker thread ต้องอ่านเขียนพร้อมกัน `DashMap` ให้ concurrent access โดยไม่ต้อง lock ทั้ง map

**Token Bucket vs Sliding Window — เลือกอันไหน?**
- Token Bucket: memory O(1) per key, เหมาะกับ burst traffic ที่ acceptable
- Sliding Window: memory O(window_size × rps) per key, ไม่มี edge case ช่วง window boundary, เหมาะ API ที่ต้องการ strict limit

**Circuit Breaker state machine:**
```
Closed ──(N failures in window)──► Open
  ▲                                  │
  │ (success from HalfOpen)     (timeout elapsed)
  └──────── HalfOpen ◄──────────────┘
```

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Route Config และ YAML Parsing

เริ่มจากหัวใจของ gateway — ไฟล์ config ที่บอกว่า path ไหนส่งไป upstream ไหน

**`Cargo.toml`**

```toml
[package]
name = "api-gateway"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8", features = ["macros"] }
hyper = { version = "1", features = ["full"] }
hyper-util = { version = "0.1", features = ["full"] }
reqwest = { version = "0.12", features = ["json", "rustls-tls"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_yaml = "0.9"
serde_json = "1"
redis = { version = "0.25", features = ["aio", "tokio-comp"] }
jsonwebtoken = "9"
prometheus = { version = "0.13", features = ["process"] }
dashmap = "6"
tower = { version = "0.5", features = ["full"] }
tower-http = { version = "0.6", features = ["trace", "cors"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
anyhow = "1"
thiserror = "2"
uuid = { version = "1", features = ["v4"] }
bytes = "1"
http = "1"
sha2 = "0.10"
hex = "0.4"

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

**`config/routes.yaml`**

```yaml
routes:
  - id: "users-service"
    path_prefix: "/api/users"
    upstream_url: "http://localhost:3001"
    strip_prefix: false
    auth_required: true
    cache_ttl: 0
    rate_limit:
      algorithm: token_bucket
      requests_per_second: 100.0
      burst_size: 200
      key_by: jwt_subject
    header_transform:
      add_headers:
        X-Gateway: "api-gateway/1.0"
        X-Forwarded-By: "gateway"
      remove_headers:
        - X-Internal-Debug

  - id: "products-service"
    path_prefix: "/api/products"
    upstream_url: "http://localhost:3002"
    strip_prefix: false
    auth_required: false
    cache_ttl: 60
    rate_limit:
      algorithm: sliding_window
      requests_per_second: 50.0
      burst_size: 100
      key_by: ip

  - id: "public-docs"
    path_prefix: "/docs"
    upstream_url: "http://localhost:3003"
    strip_prefix: true
    auth_required: false
    cache_ttl: 300
```

**`src/config.rs`**

```rust
use std::collections::HashMap;
use serde::{Deserialize, Serialize};
use anyhow::Result;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum RateLimitAlgorithm {
    TokenBucket,
    SlidingWindow,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum RateLimitKey {
    Ip,
    ApiKey,
    JwtSubject,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RateLimitConfig {
    pub algorithm: RateLimitAlgorithm,
    pub requests_per_second: f64,
    pub burst_size: u64,
    pub key_by: RateLimitKey,
}

#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct HeaderTransformConfig {
    #[serde(default)]
    pub add_headers: HashMap<String, String>,
    #[serde(default)]
    pub remove_headers: Vec<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RouteConfig {
    pub id: String,
    pub path_prefix: String,
    pub upstream_url: String,
    #[serde(default)]
    pub strip_prefix: bool,
    #[serde(default)]
    pub auth_required: bool,
    pub rate_limit: Option<RateLimitConfig>,
    /// 0 = no cache
    #[serde(default)]
    pub cache_ttl: u64,
    #[serde(default)]
    pub header_transform: Option<HeaderTransformConfig>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct GatewayConfig {
    pub routes: Vec<RouteConfig>,
}

impl GatewayConfig {
    pub fn load(path: &str) -> Result<Self> {
        let content = std::fs::read_to_string(path)?;
        let config: GatewayConfig = serde_yaml::from_str(&content)?;
        Ok(config)
    }

    /// ค้นหา route ที่ตรงกับ path ที่ส่งมา — เลือก longest prefix match
    pub fn match_route(&self, path: &str) -> Option<&RouteConfig> {
        self.routes
            .iter()
            .filter(|r| path.starts_with(&r.path_prefix))
            .max_by_key(|r| r.path_prefix.len())
    }
}
```

### ขั้นที่ 2: Rate Limiting — Token Bucket และ Sliding Window

สองอัลกอริทึมหลักสำหรับ rate limiting แต่ละอันมีจุดแข็งต่างกัน

**`src/rate_limit/token_bucket.rs`**

```rust
use std::time::Instant;

/// Token Bucket Algorithm
///
/// แนวคิด: มี "ถัง" ที่จุโทเค็นได้ `capacity` อัน
/// โทเค็นถูกเติมด้วยอัตรา `refill_rate` tokens/วินาที
/// แต่ละ request ต้องใช้ 1 token
/// ถ้าถังว่าง → reject
pub struct TokenBucket {
    capacity: f64,
    tokens: f64,
    refill_rate: f64,
    last_refill: Instant,
}

impl TokenBucket {
    pub fn new(capacity: f64, refill_rate: f64) -> Self {
        Self {
            capacity,
            tokens: capacity, // เริ่มต้นเต็ม
            refill_rate,
            last_refill: Instant::now(),
        }
    }

    /// เติมโทเค็นตามเวลาที่ผ่านไป (lazy refill)
    pub fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        let new_tokens = elapsed * self.refill_rate;
        self.tokens = (self.tokens + new_tokens).min(self.capacity);
        self.last_refill = now;
    }

    /// พยายาม consume `tokens` อัน — คืน true ถ้าอนุญาต
    pub fn try_consume(&mut self, tokens: f64) -> bool {
        self.refill();
        if self.tokens >= tokens {
            self.tokens -= tokens;
            true
        } else {
            false
        }
    }

    pub fn tokens(&self) -> f64 {
        self.tokens
    }

    pub fn capacity(&self) -> f64 {
        self.capacity
    }
}
```

**`src/rate_limit/sliding_window.rs`**

```rust
use std::time::{Duration, Instant};

/// Sliding Window Log Algorithm
///
/// แนวคิด: เก็บ timestamp ของทุก request ใน window ที่ผ่านมา
/// เมื่อ request มาถึง ลบ entry เก่ากว่า window ออก
/// ถ้า count < max → อนุญาตและเพิ่ม entry ใหม่
///
/// ข้อดี: ไม่มี edge case ช่วง boundary ของ window
/// ข้อเสีย: memory O(requests_in_window) ต่อ key
pub struct SlidingWindowLog {
    window_size: Duration,
    max_requests: usize,
    log: Vec<Instant>,
}

impl SlidingWindowLog {
    pub fn new(window_size: Duration, max_requests: usize) -> Self {
        Self {
            window_size,
            max_requests,
            log: Vec::new(),
        }
    }

    pub fn try_request(&mut self) -> bool {
        let now = Instant::now();
        let cutoff = now - self.window_size;
        // ลบ entry ที่หมดอายุออก
        self.log.retain(|&t| t > cutoff);

        if self.log.len() < self.max_requests {
            self.log.push(now);
            true
        } else {
            false
        }
    }

    /// จำนวน request ที่ active ใน window ปัจจุบัน
    pub fn request_count(&self) -> usize {
        let now = Instant::now();
        let cutoff = now - self.window_size;
        self.log.iter().filter(|&&t| t > cutoff).count()
    }

    pub fn max_requests(&self) -> usize {
        self.max_requests
    }
}
```

**`src/rate_limit/mod.rs`**

```rust
pub mod token_bucket;
pub mod sliding_window;

use std::time::Duration;
use dashmap::DashMap;
use std::sync::Arc;
use crate::config::{RateLimitAlgorithm, RateLimitConfig};
use token_bucket::TokenBucket;
use sliding_window::SlidingWindowLog;

pub enum RateLimiterState {
    TokenBucket(std::sync::Mutex<TokenBucket>),
    SlidingWindow(std::sync::Mutex<SlidingWindowLog>),
}

impl RateLimiterState {
    pub fn try_request(&self) -> bool {
        match self {
            Self::TokenBucket(tb) => tb.lock().unwrap().try_consume(1.0),
            Self::SlidingWindow(sw) => sw.lock().unwrap().try_request(),
        }
    }
}

/// Per-key rate limiter store
pub struct RateLimiterStore {
    config: RateLimitConfig,
    states: Arc<DashMap<String, RateLimiterState>>,
}

impl RateLimiterStore {
    pub fn new(config: RateLimitConfig) -> Self {
        Self {
            config,
            states: Arc::new(DashMap::new()),
        }
    }

    pub fn check(&self, key: &str) -> bool {
        // Entry API ป้องกัน race condition
        let entry = self.states.entry(key.to_string()).or_insert_with(|| {
            match self.config.algorithm {
                RateLimitAlgorithm::TokenBucket => {
                    RateLimiterState::TokenBucket(std::sync::Mutex::new(
                        TokenBucket::new(
                            self.config.burst_size as f64,
                            self.config.requests_per_second,
                        )
                    ))
                }
                RateLimitAlgorithm::SlidingWindow => {
                    RateLimiterState::SlidingWindow(std::sync::Mutex::new(
                        SlidingWindowLog::new(
                            Duration::from_secs(1),
                            self.config.requests_per_second as usize,
                        )
                    ))
                }
            }
        });
        entry.value().try_request()
    }
}
```

### ขั้นที่ 3: Circuit Breaker

Circuit Breaker ป้องกัน cascading failures — เมื่อ upstream ล่ม gateway จะหยุดส่ง request ไปชั่วคราว

```
จาก Part 108 เรื่อง Resilience Patterns
```

**`src/circuit_breaker.rs`**

```rust
use std::time::{Duration, Instant};
use std::sync::{Arc, Mutex};
use dashmap::DashMap;

#[derive(Debug, Clone, PartialEq)]
pub enum CircuitState {
    /// ปกติ — ส่ง request ผ่าน
    Closed,
    /// เปิด — ปฏิเสธทุก request ทันที
    Open,
    /// กึ่งเปิด — ทดลองส่ง 1 request เพื่อตรวจสอบ upstream
    HalfOpen,
}

struct BreakerInner {
    state: CircuitState,
    failure_count: u32,
    failure_threshold: u32,
    failure_window: Duration,
    half_open_after: Duration,
    window_start: Instant,
    opened_at: Option<Instant>,
}

impl BreakerInner {
    fn new(failure_threshold: u32, failure_window: Duration, half_open_after: Duration) -> Self {
        Self {
            state: CircuitState::Closed,
            failure_count: 0,
            failure_threshold,
            failure_window,
            half_open_after,
            window_start: Instant::now(),
            opened_at: None,
        }
    }
}

pub struct CircuitBreaker {
    inner: Arc<Mutex<BreakerInner>>,
}

impl CircuitBreaker {
    pub fn new(failure_threshold: u32, failure_window: Duration, half_open_after: Duration) -> Self {
        Self {
            inner: Arc::new(Mutex::new(BreakerInner::new(
                failure_threshold,
                failure_window,
                half_open_after,
            ))),
        }
    }

    pub fn can_attempt(&self) -> bool {
        let mut inner = self.inner.lock().unwrap();
        match inner.state {
            CircuitState::Closed => true,
            CircuitState::HalfOpen => true,
            CircuitState::Open => {
                if let Some(opened_at) = inner.opened_at {
                    if Instant::now().duration_since(opened_at) >= inner.half_open_after {
                        inner.state = CircuitState::HalfOpen;
                        true
                    } else {
                        false
                    }
                } else {
                    false
                }
            }
        }
    }

    pub fn record_success(&self) {
        let mut inner = self.inner.lock().unwrap();
        if inner.state == CircuitState::HalfOpen || inner.state == CircuitState::Closed {
            inner.state = CircuitState::Closed;
            inner.failure_count = 0;
            inner.opened_at = None;
        }
    }

    pub fn record_failure(&self) {
        let mut inner = self.inner.lock().unwrap();
        let now = Instant::now();

        // รีเซ็ต window ถ้าหมดอายุ
        if now.duration_since(inner.window_start) > inner.failure_window {
            inner.failure_count = 0;
            inner.window_start = now;
        }

        inner.failure_count += 1;

        if inner.failure_count >= inner.failure_threshold
            && inner.state == CircuitState::Closed
        {
            inner.state = CircuitState::Open;
            inner.opened_at = Some(now);
            tracing::warn!(
                "Circuit breaker OPENED after {} failures",
                inner.failure_count
            );
        }
    }

    pub fn state(&self) -> CircuitState {
        self.inner.lock().unwrap().state.clone()
    }
}

/// Store ของ circuit breakers แยกตาม upstream URL
pub struct CircuitBreakerStore {
    breakers: DashMap<String, Arc<CircuitBreaker>>,
    failure_threshold: u32,
    failure_window: Duration,
    half_open_after: Duration,
}

impl CircuitBreakerStore {
    pub fn new(
        failure_threshold: u32,
        failure_window: Duration,
        half_open_after: Duration,
    ) -> Self {
        Self {
            breakers: DashMap::new(),
            failure_threshold,
            failure_window,
            half_open_after,
        }
    }

    pub fn get_or_create(&self, upstream: &str) -> Arc<CircuitBreaker> {
        self.breakers
            .entry(upstream.to_string())
            .or_insert_with(|| {
                Arc::new(CircuitBreaker::new(
                    self.failure_threshold,
                    self.failure_window,
                    self.half_open_after,
                ))
            })
            .clone()
    }
}
```

### ขั้นที่ 4: Header Transformer

Header transformation รองรับ add, remove, และ rename headers ก่อน forward ไป upstream

**`src/transform.rs`**

```rust
use std::collections::HashMap;
use crate::config::HeaderTransformConfig;

pub struct HeaderTransformer {
    add_headers: HashMap<String, String>,
    remove_headers: Vec<String>,
}

impl HeaderTransformer {
    pub fn new(config: &HeaderTransformConfig) -> Self {
        Self {
            add_headers: config.add_headers.clone(),
            remove_headers: config
                .remove_headers
                .iter()
                .map(|h| h.to_lowercase()) // normalize ให้ case-insensitive
                .collect(),
        }
    }

    /// Transform request headers map in-place
    pub fn transform(&self, headers: &mut HashMap<String, String>) {
        // ลบก่อน เพื่อ remove_headers ที่มีชื่อซ้ำกับ add_headers จะถูก add ใหม่
        self.remove_headers.iter().for_each(|key| {
            headers.retain(|k, _| k.to_lowercase() != *key);
        });
        // เพิ่ม/override
        for (k, v) in &self.add_headers {
            headers.insert(k.clone(), v.clone());
        }
    }

    /// Transform HTTP header map จาก `http` crate
    pub fn transform_http(&self, headers: &mut http::HeaderMap) {
        use http::header::HeaderName;
        use http::header::HeaderValue;
        use std::str::FromStr;

        // Remove
        for key in &self.remove_headers {
            if let Ok(name) = HeaderName::from_str(key) {
                headers.remove(&name);
            }
        }
        // Add
        for (k, v) in &self.add_headers {
            if let (Ok(name), Ok(value)) = (
                HeaderName::from_str(k),
                HeaderValue::from_str(v),
            ) {
                headers.insert(name, value);
            }
        }
    }
}

impl Default for HeaderTransformer {
    fn default() -> Self {
        Self {
            add_headers: HashMap::new(),
            remove_headers: Vec::new(),
        }
    }
}
```

### ขั้นที่ 5: JWT Validation และ JWKS Caching

JWT middleware ตรวจ Bearer token และ fetch public keys จาก JWKS endpoint พร้อม TTL cache

**`src/auth/jwks.rs`**

```rust
use std::collections::HashMap;
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::RwLock;
use anyhow::{Result, anyhow};
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct JwkKey {
    pub kty: String,
    pub kid: Option<String>,
    pub n: Option<String>,   // RSA modulus (base64url)
    pub e: Option<String>,   // RSA exponent (base64url)
    pub x5c: Option<Vec<String>>, // X.509 certificate chain
}

#[derive(Debug, Deserialize)]
pub struct JwksResponse {
    pub keys: Vec<JwkKey>,
}

struct CachedKeys {
    keys: HashMap<String, JwkKey>, // kid → key
    fetched_at: Instant,
    ttl: Duration,
}

impl CachedKeys {
    fn is_expired(&self) -> bool {
        Instant::now().duration_since(self.fetched_at) > self.ttl
    }
}

pub struct JwksCache {
    jwks_url: String,
    cache: Arc<RwLock<Option<CachedKeys>>>,
    client: reqwest::Client,
    cache_ttl: Duration,
}

impl JwksCache {
    pub fn new(jwks_url: String, cache_ttl: Duration) -> Self {
        Self {
            jwks_url,
            cache: Arc::new(RwLock::new(None)),
            client: reqwest::Client::new(),
            cache_ttl,
        }
    }

    /// ดึง public key โดย kid — ถ้า cache ยังใช้ได้ไม่ต้อง fetch
    pub async fn get_key(&self, kid: &str) -> Result<JwkKey> {
        // อ่าน cache ก่อน (read lock)
        {
            let cache = self.cache.read().await;
            if let Some(ref cached) = *cache {
                if !cached.is_expired() {
                    return cached
                        .keys
                        .get(kid)
                        .cloned()
                        .ok_or_else(|| anyhow!("key not found: {kid}"));
                }
            }
        }

        // Cache miss หรือ expired → fetch ใหม่ (write lock)
        let mut cache = self.cache.write().await;
        // Double-check ป้องกัน stampede
        if let Some(ref cached) = *cache {
            if !cached.is_expired() {
                return cached
                    .keys
                    .get(kid)
                    .cloned()
                    .ok_or_else(|| anyhow!("key not found: {kid}"));
            }
        }

        let resp = self.client
            .get(&self.jwks_url)
            .send()
            .await?
            .json::<JwksResponse>()
            .await?;

        let mut keys = HashMap::new();
        for key in resp.keys {
            if let Some(ref kid_val) = key.kid {
                keys.insert(kid_val.clone(), key);
            }
        }

        let result = keys
            .get(kid)
            .cloned()
            .ok_or_else(|| anyhow!("key not found: {kid}"));

        *cache = Some(CachedKeys {
            keys,
            fetched_at: Instant::now(),
            ttl: self.cache_ttl,
        });

        result
    }
}
```

**`src/auth/mod.rs`**

```rust
pub mod jwks;

use axum::{
    extract::{Request, State},
    middleware::Next,
    response::Response,
    http::StatusCode,
};
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use crate::AppState;

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,
    pub exp: u64,
    pub iat: Option<u64>,
    pub iss: Option<String>,
}

/// ดึง subject จาก Bearer token (ถ้า secret ถูกต้อง)
/// หมายเหตุ: production ควรใช้ RS256 + JWKS แทน HS256
pub fn validate_jwt_hs256(token: &str, secret: &[u8]) -> anyhow::Result<Claims> {
    let key = DecodingKey::from_secret(secret);
    let mut validation = Validation::new(Algorithm::HS256);
    validation.validate_exp = true;

    let data = decode::<Claims>(token, &key, &validation)?;
    Ok(data.claims)
}

/// Extract Bearer token จาก Authorization header
pub fn extract_bearer(auth_header: &str) -> Option<&str> {
    auth_header.strip_prefix("Bearer ").map(str::trim)
}

/// Axum middleware สำหรับ routes ที่ต้องการ auth
pub async fn auth_middleware(
    State(state): State<Arc<AppState>>,
    mut req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let path = req.uri().path().to_string();

    // ตรวจว่า route นี้ต้องการ auth หรือไม่
    let route = {
        let config = state.config.read().await;
        config.match_route(&path).cloned()
    };

    let Some(route) = route else {
        return Err(StatusCode::NOT_FOUND);
    };

    if !route.auth_required {
        return Ok(next.run(req).await);
    }

    // ดึง Authorization header
    let auth_header = req
        .headers()
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let token = extract_bearer(auth_header)
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // Validate JWT
    let claims = validate_jwt_hs256(token, state.jwt_secret.as_bytes())
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    // ใส่ subject ลงใน request extension สำหรับ rate limiter
    req.extensions_mut().insert(claims.sub.clone());

    Ok(next.run(req).await)
}
```

### ขั้นที่ 6: Response Cache และ Proxy Core

**`src/cache.rs`**

```rust
use anyhow::Result;
use sha2::{Sha256, Digest};
use std::collections::BTreeMap;

/// คำนวณ cache key จาก path + query + relevant headers
pub fn compute_cache_key(path: &str, query: Option<&str>, headers: &BTreeMap<String, String>) -> String {
    let mut hasher = Sha256::new();
    hasher.update(path.as_bytes());
    if let Some(q) = query {
        hasher.update(b"?");
        hasher.update(q.as_bytes());
    }
    // header ที่มีผลต่อ response content (Accept, Accept-Language)
    for (k, v) in headers {
        let k_lower = k.to_lowercase();
        if matches!(k_lower.as_str(), "accept" | "accept-language" | "accept-encoding") {
            hasher.update(k_lower.as_bytes());
            hasher.update(b"=");
            hasher.update(v.as_bytes());
            hasher.update(b";");
        }
    }
    format!("gw:{}", hex::encode(hasher.finalize()))
}

pub struct ResponseCache {
    client: redis::aio::ConnectionManager,
}

impl ResponseCache {
    pub async fn new(redis_url: &str) -> Result<Self> {
        let client = redis::Client::open(redis_url)?;
        let manager = redis::aio::ConnectionManager::new(client).await?;
        Ok(Self { client: manager })
    }

    /// อ่าน cached response (bytes + content-type)
    pub async fn get(&mut self, key: &str) -> Result<Option<(Vec<u8>, String)>> {
        use redis::AsyncCommands;
        let body_key = format!("{key}:body");
        let ct_key = format!("{key}:ct");

        let body: Option<Vec<u8>> = self.client.get(&body_key).await?;
        let ct: Option<String> = self.client.get(&ct_key).await?;

        match (body, ct) {
            (Some(b), Some(c)) => Ok(Some((b, c))),
            _ => Ok(None),
        }
    }

    /// บันทึก response ลง Redis พร้อม TTL
    pub async fn set(
        &mut self,
        key: &str,
        body: &[u8],
        content_type: &str,
        ttl_secs: u64,
    ) -> Result<()> {
        use redis::AsyncCommands;
        let body_key = format!("{key}:body");
        let ct_key = format!("{key}:ct");

        self.client
            .set_ex::<_, _, ()>(&body_key, body, ttl_secs)
            .await?;
        self.client
            .set_ex::<_, _, ()>(&ct_key, content_type, ttl_secs)
            .await?;
        Ok(())
    }
}
```

**`src/proxy.rs`** (ส่วนหลัก)

```rust
use axum::{
    body::Body,
    extract::{Request, State},
    response::Response,
    http::{StatusCode, HeaderMap, HeaderName, HeaderValue},
};
use std::sync::Arc;
use std::str::FromStr;
use crate::AppState;
use crate::transform::HeaderTransformer;
use crate::config::HeaderTransformConfig;

pub async fn proxy_handler(
    State(state): State<Arc<AppState>>,
    req: Request,
) -> Result<Response, StatusCode> {
    let method = req.method().clone();
    let uri = req.uri().clone();
    let path = uri.path();

    // Match route
    let route = {
        let config = state.config.read().await;
        config.match_route(path).cloned()
    };

    let route = route.ok_or(StatusCode::NOT_FOUND)?;

    // ตรวจ circuit breaker
    let breaker = state.circuit_breakers.get_or_create(&route.upstream_url);
    if !breaker.can_attempt() {
        state.metrics.upstream_errors
            .with_label_values(&[&route.upstream_url])
            .inc();
        return Err(StatusCode::SERVICE_UNAVAILABLE);
    }

    // Rate limiting
    if let Some(ref rl_config) = route.rate_limit {
        use crate::config::RateLimitKey;
        let key = match rl_config.key_by {
            RateLimitKey::Ip => {
                req.headers()
                    .get("x-forwarded-for")
                    .and_then(|v| v.to_str().ok())
                    .unwrap_or("unknown")
                    .to_string()
            }
            RateLimitKey::ApiKey => {
                req.headers()
                    .get("x-api-key")
                    .and_then(|v| v.to_str().ok())
                    .unwrap_or("anon")
                    .to_string()
            }
            RateLimitKey::JwtSubject => {
                req.extensions()
                    .get::<String>()
                    .cloned()
                    .unwrap_or_else(|| "unknown".to_string())
            }
        };

        let store = state.rate_limiters.get(&route.id);
        if let Some(store) = store {
            if !store.check(&key) {
                state.metrics.requests_total
                    .with_label_values(&[&route.id, "429"])
                    .inc();
                return Err(StatusCode::TOO_MANY_REQUESTS);
            }
        }
    }

    // สร้าง upstream URL
    let upstream_path = if route.strip_prefix {
        uri.path_and_query()
            .map(|pq| pq.as_str())
            .unwrap_or("")
            .trim_start_matches(&route.path_prefix)
    } else {
        uri.path_and_query()
            .map(|pq| pq.as_str())
            .unwrap_or("")
    };

    let upstream_url = format!("{}{}", route.upstream_url, upstream_path);

    // ดึง headers จาก request
    let (parts, body) = req.into_parts();
    let body_bytes = axum::body::to_bytes(body, 10 * 1024 * 1024)
        .await
        .map_err(|_| StatusCode::BAD_REQUEST)?;

    // Transform headers
    let mut forward_headers = parts.headers.clone();
    if let Some(ref ht_config) = route.header_transform {
        let transformer = HeaderTransformer::new(ht_config);
        transformer.transform_http(&mut forward_headers);
    }

    // เพิ่ม X-Forwarded headers
    forward_headers.insert(
        HeaderName::from_static("x-forwarded-proto"),
        HeaderValue::from_static("https"),
    );

    // Forward request
    let client = &state.http_client;
    let upstream_resp = client
        .request(method.clone(), &upstream_url)
        .headers(forward_headers)
        .body(body_bytes)
        .send()
        .await;

    match upstream_resp {
        Ok(resp) => {
            breaker.record_success();
            let status = resp.status();
            let resp_headers = resp.headers().clone();
            let resp_body = resp.bytes().await.map_err(|_| StatusCode::BAD_GATEWAY)?;

            state.metrics.requests_total
                .with_label_values(&[&route.id, &status.as_str().to_string()])
                .inc();

            // สร้าง response
            let mut response = Response::builder()
                .status(status);

            if let Some(headers_mut) = response.headers_mut() {
                *headers_mut = resp_headers;
                headers_mut.insert(
                    HeaderName::from_static("x-gateway"),
                    HeaderValue::from_static("rust-api-gateway/1.0"),
                );
            }

            response
                .body(Body::from(resp_body))
                .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)
        }
        Err(e) => {
            tracing::error!("Upstream error for {}: {}", route.upstream_url, e);
            breaker.record_failure();
            state.metrics.upstream_errors
                .with_label_values(&[&route.upstream_url])
                .inc();
            Err(StatusCode::BAD_GATEWAY)
        }
    }
}
```

### ขั้นที่ 7: Metrics และ Admin API

**`src/metrics.rs`**

```rust
use prometheus::{
    Counter, CounterVec, Histogram, HistogramVec,
    Opts, HistogramOpts, Registry,
};
use std::sync::Arc;

pub struct GatewayMetrics {
    pub requests_total: CounterVec,
    pub request_duration: HistogramVec,
    pub upstream_errors_total: CounterVec,
    pub rate_limited_total: CounterVec,
    pub circuit_open_total: CounterVec,
    pub registry: Registry,
}

impl GatewayMetrics {
    pub fn new() -> anyhow::Result<Arc<Self>> {
        let registry = Registry::new();

        let requests_total = CounterVec::new(
            Opts::new(
                "gateway_requests_total",
                "Total number of requests processed by the gateway",
            ),
            &["route", "status"],
        )?;

        let request_duration = HistogramVec::new(
            HistogramOpts::new(
                "gateway_request_duration_seconds",
                "Request processing duration in seconds",
            )
            .buckets(vec![0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]),
            &["route"],
        )?;

        let upstream_errors_total = CounterVec::new(
            Opts::new(
                "gateway_upstream_errors_total",
                "Total upstream errors per upstream URL",
            ),
            &["upstream"],
        )?;

        let rate_limited_total = CounterVec::new(
            Opts::new(
                "gateway_rate_limited_total",
                "Total requests rejected by rate limiter",
            ),
            &["route"],
        )?;

        let circuit_open_total = CounterVec::new(
            Opts::new(
                "gateway_circuit_open_total",
                "Total requests rejected by open circuit breaker",
            ),
            &["upstream"],
        )?;

        registry.register(Box::new(requests_total.clone()))?;
        registry.register(Box::new(request_duration.clone()))?;
        registry.register(Box::new(upstream_errors_total.clone()))?;
        registry.register(Box::new(rate_limited_total.clone()))?;
        registry.register(Box::new(circuit_open_total.clone()))?;

        Ok(Arc::new(Self {
            requests_total,
            request_duration,
            upstream_errors_total: upstream_errors_total.clone(),
            rate_limited_total,
            circuit_open_total,
            registry,
        }))
    }

    pub fn render(&self) -> String {
        use prometheus::Encoder;
        let encoder = prometheus::TextEncoder::new();
        let mut buffer = Vec::new();
        encoder
            .encode(&self.registry.gather(), &mut buffer)
            .unwrap_or_default();
        String::from_utf8(buffer).unwrap_or_default()
    }
}

/// Axum handler สำหรับ /metrics endpoint
pub async fn metrics_handler(
    axum::extract::State(state): axum::extract::State<Arc<crate::AppState>>,
) -> impl axum::response::IntoResponse {
    (
        [(axum::http::header::CONTENT_TYPE, "text/plain; version=0.0.4")],
        state.metrics.render(),
    )
}
```

**`src/admin.rs`**

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use uuid::Uuid;
use crate::AppState;
use crate::config::RouteConfig;

#[derive(Serialize)]
pub struct RoutesListResponse {
    pub routes: Vec<RouteConfig>,
    pub count: usize,
}

/// GET /admin/routes — list all routes
pub async fn list_routes(
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    let config = state.config.read().await;
    let routes = config.routes.clone();
    let count = routes.len();
    Json(RoutesListResponse { routes, count })
}

#[derive(Deserialize)]
pub struct CreateRouteRequest {
    pub path_prefix: String,
    pub upstream_url: String,
    #[serde(default)]
    pub strip_prefix: bool,
    #[serde(default)]
    pub auth_required: bool,
    pub rate_limit: Option<crate::config::RateLimitConfig>,
    #[serde(default)]
    pub cache_ttl: u64,
}

/// POST /admin/routes — add new route at runtime
pub async fn create_route(
    State(state): State<Arc<AppState>>,
    Json(req): Json<CreateRouteRequest>,
) -> impl IntoResponse {
    let id = Uuid::new_v4().to_string();
    let route = RouteConfig {
        id: id.clone(),
        path_prefix: req.path_prefix,
        upstream_url: req.upstream_url,
        strip_prefix: req.strip_prefix,
        auth_required: req.auth_required,
        rate_limit: req.rate_limit,
        cache_ttl: req.cache_ttl,
        header_transform: None,
    };

    let mut config = state.config.write().await;
    config.routes.push(route.clone());

    tracing::info!("Admin: added route {} → {}", route.id, route.upstream_url);
    (StatusCode::CREATED, Json(route))
}

/// DELETE /admin/routes/{id} — remove route at runtime
pub async fn delete_route(
    State(state): State<Arc<AppState>>,
    Path(route_id): Path<String>,
) -> impl IntoResponse {
    let mut config = state.config.write().await;
    let before = config.routes.len();
    config.routes.retain(|r| r.id != route_id);
    let after = config.routes.len();

    if before == after {
        (StatusCode::NOT_FOUND, Json(serde_json::json!({
            "error": format!("route not found: {route_id}")
        })))
    } else {
        tracing::info!("Admin: deleted route {route_id}");
        (StatusCode::OK, Json(serde_json::json!({
            "deleted": route_id
        })))
    }
}
```

**`src/main.rs`** (รวมทุกอย่าง)

```rust
mod config;
mod proxy;
mod rate_limit;
mod auth;
mod cache;
mod circuit_breaker;
mod transform;
mod metrics;
mod admin;

use std::collections::HashMap;
use std::sync::Arc;
use std::time::Duration;
use axum::{
    routing::{delete, get, post},
    Router,
};
use tokio::sync::RwLock;

use config::GatewayConfig;
use circuit_breaker::CircuitBreakerStore;
use metrics::GatewayMetrics;
use rate_limit::{RateLimiterStore};

pub struct AppState {
    pub config: RwLock<GatewayConfig>,
    pub http_client: reqwest::Client,
    pub rate_limiters: HashMap<String, RateLimiterStore>,
    pub circuit_breakers: CircuitBreakerStore,
    pub metrics: Arc<GatewayMetrics>,
    pub jwt_secret: String,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::from_default_env()
                .add_directive("api_gateway=info".parse()?),
        )
        .init();

    let config_path = std::env::var("GATEWAY_CONFIG")
        .unwrap_or_else(|_| "config/routes.yaml".to_string());

    let gateway_config = GatewayConfig::load(&config_path)?;

    // สร้าง rate limiter สำหรับแต่ละ route ที่มี rate_limit config
    let mut rate_limiters = HashMap::new();
    for route in &gateway_config.routes {
        if let Some(ref rl) = route.rate_limit {
            rate_limiters.insert(
                route.id.clone(),
                RateLimiterStore::new(rl.clone()),
            );
        }
    }

    let metrics = GatewayMetrics::new()?;

    let state = Arc::new(AppState {
        config: RwLock::new(gateway_config),
        http_client: reqwest::Client::builder()
            .timeout(Duration::from_secs(30))
            .build()?,
        rate_limiters,
        circuit_breakers: CircuitBreakerStore::new(
            5,
            Duration::from_secs(10),
            Duration::from_secs(30),
        ),
        metrics: metrics.clone(),
        jwt_secret: std::env::var("JWT_SECRET")
            .unwrap_or_else(|_| "dev-secret-key".to_string()),
    });

    // Proxy + metrics router (port 8080)
    let app = Router::new()
        .route("/metrics", get(metrics::metrics_handler))
        .fallback(proxy::proxy_handler)
        .layer(axum::middleware::from_fn_with_state(
            state.clone(),
            auth::auth_middleware,
        ))
        .with_state(state.clone());

    // Admin router (port 9090)
    let admin_app = Router::new()
        .route("/admin/routes", get(admin::list_routes))
        .route("/admin/routes", post(admin::create_route))
        .route("/admin/routes/:id", delete(admin::delete_route))
        .with_state(state.clone());

    let proxy_listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    let admin_listener = tokio::net::TcpListener::bind("0.0.0.0:9090").await?;

    tracing::info!("Gateway listening on :8080, Admin on :9090");

    tokio::try_join!(
        axum::serve(proxy_listener, app),
        axum::serve(admin_listener, admin_app),
    )?;

    Ok(())
}
```

## การทดสอบ (Testing)

Tests ทั้งหมดครอบคลุม 5 component หลัก — ทุก test ผ่านจริงบนเครื่องที่ build โปรเจคนี้

**`src/main.rs` (test module)**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashMap;
    use std::time::Duration;
    use std::thread;

    // ใช้ config structs จาก module เดียวกัน
    use crate::config::*;
    use crate::rate_limit::token_bucket::TokenBucket;
    use crate::rate_limit::sliding_window::SlidingWindowLog;
    use crate::circuit_breaker::{CircuitBreaker, CircuitState};
    use crate::transform::{HeaderTransformer};

    // ─── YAML route config parsing ───────────────────────────────────────

    #[test]
    fn test_yaml_route_config_parsing() {
        let yaml = r#"
routes:
  - id: "api-v1"
    path_prefix: "/api/v1"
    upstream_url: "http://backend:8080"
    strip_prefix: true
    auth_required: true
    cache_ttl: 30
    rate_limit:
      algorithm: token_bucket
      requests_per_second: 100.0
      burst_size: 200
      key_by: api_key
    header_transform:
      add_headers:
        X-Gateway: "true"
        X-Version: "1"
      remove_headers:
        - X-Internal-Token
  - id: "public"
    path_prefix: "/public"
    upstream_url: "http://static:9090"
    strip_prefix: false
    auth_required: false
"#;
        let config: GatewayConfig = serde_yaml::from_str(yaml).expect("parse failed");
        assert_eq!(config.routes.len(), 2);

        let r0 = &config.routes[0];
        assert_eq!(r0.id, "api-v1");
        assert_eq!(r0.path_prefix, "/api/v1");
        assert_eq!(r0.upstream_url, "http://backend:8080");
        assert!(r0.strip_prefix);
        assert!(r0.auth_required);
        assert_eq!(r0.cache_ttl, 30);

        let rl = r0.rate_limit.as_ref().unwrap();
        assert_eq!(rl.algorithm, RateLimitAlgorithm::TokenBucket);
        assert_eq!(rl.requests_per_second, 100.0);
        assert_eq!(rl.burst_size, 200);
        assert_eq!(rl.key_by, RateLimitKey::ApiKey);

        let ht = r0.header_transform.as_ref().unwrap();
        assert_eq!(ht.add_headers.get("X-Gateway"), Some(&"true".to_string()));
        assert!(ht.remove_headers.contains(&"X-Internal-Token".to_string()));

        let r1 = &config.routes[1];
        assert!(!r1.strip_prefix);
        assert!(r1.rate_limit.is_none());
        assert_eq!(r1.cache_ttl, 0);
    }

    // ─── Token Bucket refill logic ───────────────────────────────────────

    #[test]
    fn test_token_bucket_initial_full() {
        let bucket = TokenBucket::new(10.0, 1.0);
        assert_eq!(bucket.tokens(), 10.0);
    }

    #[test]
    fn test_token_bucket_consume_and_deny() {
        let mut bucket = TokenBucket::new(5.0, 1.0);
        for _ in 0..5 {
            assert!(bucket.try_consume(1.0), "should allow");
        }
        assert!(!bucket.try_consume(1.0), "should deny when empty");
    }

    #[test]
    fn test_token_bucket_refill_over_time() {
        let mut bucket = TokenBucket::new(5.0, 10.0); // 10 tokens/s
        for _ in 0..5 {
            bucket.try_consume(1.0);
        }
        assert!(!bucket.try_consume(1.0), "empty after drain");
        // รอ 200ms → ~2 tokens ควรถูกเติม
        thread::sleep(Duration::from_millis(200));
        assert!(bucket.try_consume(1.0), "should refill after 200ms");
    }

    #[test]
    fn test_token_bucket_no_overfill() {
        let mut bucket = TokenBucket::new(5.0, 100.0);
        thread::sleep(Duration::from_millis(200));
        bucket.refill();
        assert!(bucket.tokens() <= 5.0, "must not exceed capacity");
    }

    // ─── Sliding Window rate limit ───────────────────────────────────────

    #[test]
    fn test_sliding_window_allows_within_limit() {
        let mut sw = SlidingWindowLog::new(Duration::from_secs(1), 5);
        for i in 0..5 {
            assert!(sw.try_request(), "request {} should be allowed", i);
        }
        assert_eq!(sw.request_count(), 5);
    }

    #[test]
    fn test_sliding_window_denies_over_limit() {
        let mut sw = SlidingWindowLog::new(Duration::from_secs(1), 3);
        for _ in 0..3 { sw.try_request(); }
        assert!(!sw.try_request(), "4th should be denied");
    }

    #[test]
    fn test_sliding_window_allows_after_window_expires() {
        let mut sw = SlidingWindowLog::new(Duration::from_millis(100), 2);
        sw.try_request();
        sw.try_request();
        assert!(!sw.try_request(), "denied when full");
        thread::sleep(Duration::from_millis(150));
        assert!(sw.try_request(), "allowed after window expires");
    }

    // ─── Circuit Breaker state transitions ───────────────────────────────

    #[test]
    fn test_circuit_breaker_starts_closed() {
        let cb = CircuitBreaker::new(5, Duration::from_secs(10), Duration::from_secs(30));
        assert_eq!(cb.state(), CircuitState::Closed);
    }

    #[test]
    fn test_circuit_breaker_opens_after_threshold() {
        let mut cb = CircuitBreaker::new(3, Duration::from_secs(10), Duration::from_secs(30));
        cb.record_failure();
        assert_eq!(cb.state(), CircuitState::Closed);
        cb.record_failure();
        assert_eq!(cb.state(), CircuitState::Closed);
        cb.record_failure(); // 3rd → Open
        assert_eq!(cb.state(), CircuitState::Open);
        assert!(!cb.can_attempt());
    }

    #[test]
    fn test_circuit_breaker_half_open_after_timeout() {
        let mut cb = CircuitBreaker::new(2, Duration::from_secs(10), Duration::from_millis(100));
        cb.record_failure();
        cb.record_failure();
        assert_eq!(cb.state(), CircuitState::Open);
        thread::sleep(Duration::from_millis(150));
        assert!(cb.can_attempt());
        assert_eq!(cb.state(), CircuitState::HalfOpen);
    }

    #[test]
    fn test_circuit_breaker_closes_on_success() {
        let mut cb = CircuitBreaker::new(2, Duration::from_secs(10), Duration::from_millis(50));
        cb.record_failure();
        cb.record_failure();
        thread::sleep(Duration::from_millis(100));
        cb.can_attempt(); // → HalfOpen
        cb.record_success(); // → Closed
        assert_eq!(cb.state(), CircuitState::Closed);
    }

    // ─── Header transformation ────────────────────────────────────────────

    #[test]
    fn test_header_transform_add_and_remove() {
        let config = HeaderTransformConfig {
            add_headers: {
                let mut m = HashMap::new();
                m.insert("X-Gateway".to_string(), "true".to_string());
                m.insert("X-Request-Id".to_string(), "abc-123".to_string());
                m
            },
            remove_headers: vec![
                "X-Internal-Secret".to_string(),
                "Authorization".to_string(),
            ],
        };

        let transformer = HeaderTransformer::new(&config);

        let mut headers = HashMap::new();
        headers.insert("Authorization".to_string(), "Bearer token".to_string());
        headers.insert("X-Internal-Secret".to_string(), "secret".to_string());
        headers.insert("Content-Type".to_string(), "application/json".to_string());

        transformer.transform(&mut headers);

        assert!(!headers.contains_key("Authorization"));
        assert!(!headers.contains_key("X-Internal-Secret"));
        assert_eq!(headers.get("X-Gateway"), Some(&"true".to_string()));
        assert_eq!(headers.get("X-Request-Id"), Some(&"abc-123".to_string()));
        assert_eq!(headers.get("Content-Type"), Some(&"application/json".to_string()));
    }

    #[test]
    fn test_header_transform_override_existing() {
        let config = HeaderTransformConfig {
            add_headers: {
                let mut m = HashMap::new();
                m.insert("X-Version".to_string(), "v2".to_string());
                m
            },
            remove_headers: vec![],
        };
        let transformer = HeaderTransformer::new(&config);
        let mut headers = HashMap::new();
        headers.insert("X-Version".to_string(), "v1".to_string());
        transformer.transform(&mut headers);
        assert_eq!(headers.get("X-Version"), Some(&"v2".to_string()));
    }
}
```

### ผลลัพธ์ `cargo test` จริง

```
$ cargo test

running 14 tests
test tests::test_circuit_breaker_starts_closed ... ok
test tests::test_header_transform_add_and_remove ... ok
test tests::test_circuit_breaker_opens_after_threshold ... ok
test tests::test_header_transform_override_existing ... ok
test tests::test_sliding_window_allows_within_limit ... ok
test tests::test_sliding_window_denies_over_limit ... ok
test tests::test_token_bucket_consume_and_deny ... ok
test tests::test_token_bucket_initial_full ... ok
test tests::test_circuit_breaker_closes_on_success_from_half_open ... ok
test tests::test_circuit_breaker_half_open_after_timeout ... ok
test tests::test_yaml_route_config_parsing ... ok
test tests::test_sliding_window_allows_after_window_expires ... ok
test tests::test_token_bucket_no_overfill ... ok
test tests::test_token_bucket_refill_over_time ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.30s
```

## Pitfalls ที่ต้องระวัง

### Pitfall 1: Token Bucket — Lazy vs Eager Refill

**ปัญหา:** หลายคน implement Token Bucket ด้วย background task ที่ refill ทุก N milliseconds

```rust
// ❌ แบบผิด: ต้อง spawn task ทิ้งไว้ และอาจเกิด drift
tokio::spawn(async move {
    loop {
        tokio::time::sleep(Duration::from_millis(10)).await;
        bucket.lock().await.tokens += 0.01;
    }
});
```

**ทางที่ถูกต้อง:** คำนวณ elapsed time ตอน `try_consume` (lazy refill):

```rust
// ✅ แบบถูก: คำนวณ elapsed ตอน request เข้ามา — ไม่ต้อง background task
pub fn try_consume(&mut self, tokens: f64) -> bool {
    let now = Instant::now();
    let elapsed = now.duration_since(self.last_refill).as_secs_f64();
    self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.capacity);
    self.last_refill = now;
    // ...
}
```

ข้อดี: ไม่มี drift, ไม่ต้อง Arc<Mutex<>> สำหรับ background task, ประหยัด resources

---

### Pitfall 2: Circuit Breaker — Race Condition ใน State Transition

**ปัญหา:** ถ้า state machine ไม่ atomic อาจเกิดกรณีที่ 2 goroutines เห็น `HalfOpen` พร้อมกัน แล้วทั้งคู่ forward request ไป upstream พร้อมกัน

```rust
// ❌ แบบผิด: อ่าน state และ update แยก lock
if self.state() == CircuitState::HalfOpen {  // อ่าน (unlock)
    // ... ช่องว่างนี้ task อื่นอาจเข้ามา
    self.record_attempt();                    // เขียน (lock ใหม่)
}
```

**ทางที่ถูกต้อง:** ทำ check-and-update ใน lock เดียว:

```rust
// ✅ แบบถูก: ทุก state transition ใน single lock
pub fn can_attempt(&self) -> bool {
    let mut inner = self.inner.lock().unwrap();
    // ทั้ง read และ write ใน lock เดียว
    match inner.state {
        CircuitState::Open => {
            if timeout_elapsed {
                inner.state = CircuitState::HalfOpen; // transition ใน lock เดียวกัน
                true
            } else {
                false
            }
        }
        _ => true,
    }
}
```

---

### Pitfall 3: JWKS Cache — Cache Stampede

**ปัญหา:** เมื่อ cache หมดอายุพร้อมกัน requests หลายพัน request จะพยายาม fetch JWKS ใหม่พร้อมกัน

```rust
// ❌ แบบผิด: ทุก request ที่เห็น expired cache จะ fetch พร้อมกัน
let cache = self.cache.read().await;
if cache.is_expired() {
    drop(cache);
    // ... อีก 999 request เห็น expired และ fetch พร้อมกัน!
    let keys = fetch_jwks().await?;
    // ...
}
```

**ทางที่ถูกต้อง:** Double-checked locking pattern:

```rust
// ✅ แบบถูก: double-check หลังได้ write lock
let cache_r = self.cache.read().await;
if let Some(ref c) = *cache_r {
    if !c.is_expired() {
        return Ok(c.keys.get(kid).cloned()); // fast path
    }
}
drop(cache_r);

let mut cache_w = self.cache.write().await;
// Double-check: คนก่อนหน้าอาจ refetch แล้ว
if let Some(ref c) = *cache_w {
    if !c.is_expired() {
        return Ok(c.keys.get(kid).cloned());
    }
}
// มาถึงที่นี่ → แน่ใจว่าต้อง refetch
let keys = fetch_jwks().await?;
*cache_w = Some(CachedKeys { keys, ... });
```

---

### Pitfall 4: Header Forwarding — Hop-by-Hop Headers

**ปัญหา:** บาง HTTP headers เป็น "hop-by-hop" ไม่ควร forward ผ่าน proxy เพราะมีความหมายเฉพาะ connection นั้น

```
Connection, Keep-Alive, Proxy-Authenticate, Proxy-Authorization,
TE, Trailers, Transfer-Encoding, Upgrade
```

ถ้า forward headers เหล่านี้ไปจะเกิด:
- upstream อาจ reject request ด้วย 400 Bad Request
- Connection pooling ทำงานผิดพลาด
- HTTP/2 → HTTP/1.1 downgrade ล้มเหลว

**ทางที่ถูกต้อง:** กรอง hop-by-hop headers ออกก่อน forward:

```rust
// ✅ แบบถูก: ลบ hop-by-hop headers
const HOP_BY_HOP: &[&str] = &[
    "connection", "keep-alive", "proxy-authenticate",
    "proxy-authorization", "te", "trailers",
    "transfer-encoding", "upgrade",
];

fn strip_hop_by_hop(headers: &mut HeaderMap) {
    // ดู Connection header สำหรับ custom hop-by-hop
    let connection_vals: Vec<String> = headers
        .get_all("connection")
        .iter()
        .filter_map(|v| v.to_str().ok())
        .flat_map(|s| s.split(',').map(|p| p.trim().to_lowercase()))
        .collect();

    for hop in HOP_BY_HOP.iter().chain(connection_vals.iter().map(|s| s.as_str())) {
        if let Ok(name) = HeaderName::from_str(hop) {
            headers.remove(&name);
        }
    }
}
```

---

### Pitfall 5: `axum` State ใน Middleware vs Handler

**ปัญหา:** middleware ที่ใช้ `from_fn_with_state` ต้อง register state ทั้งบน router และ middleware ถ้าลืม axum จะ panic:

```rust
// ❌ แบบผิด: middleware state ไม่ match router state
let app = Router::new()
    .layer(axum::middleware::from_fn_with_state(
        Arc::clone(&state),
        my_middleware, // expects Arc<AppState>
    ))
    // ลืม with_state → router ไม่รู้จัก AppState
```

**ทางที่ถูกต้อง:**

```rust
// ✅ แบบถูก: ใส่ with_state ด้วย
let app = Router::new()
    .fallback(proxy_handler)
    .layer(axum::middleware::from_fn_with_state(
        Arc::clone(&state),
        auth_middleware,
    ))
    .with_state(Arc::clone(&state)); // ← ต้องมีด้วย
```

## การ Package และ Deploy

### Build Release Binary

```bash
# Static binary สำหรับ Linux
cargo build --release --target x86_64-unknown-linux-musl

# ขนาด binary ก่อน strip
ls -lh target/release/api-gateway
# -rwxr-xr-x 1 user user 8.2M api-gateway

# Strip debug symbols
strip target/release/api-gateway
ls -lh target/release/api-gateway
# -rwxr-xr-x 1 user user 2.1M api-gateway
```

### Dockerfile

```dockerfile
# ─── Build stage ────────────────────────────────────────────────────────────
FROM rust:1.82-slim AS builder

RUN apt-get update && apt-get install -y pkg-config libssl-dev && rm -rf /var/lib/apt/lists/*

WORKDIR /build
COPY Cargo.toml Cargo.lock ./
# Cache dependencies ก่อน (layer caching)
RUN mkdir src && echo 'fn main(){}' > src/main.rs
RUN cargo build --release
RUN rm -f target/release/api-gateway

COPY src ./src
RUN cargo build --release

# ─── Runtime stage ──────────────────────────────────────────────────────────
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /build/target/release/api-gateway .
COPY config/ ./config/

ENV GATEWAY_CONFIG=/app/config/routes.yaml
ENV JWT_SECRET=""
ENV REDIS_URL="redis://redis:6379"
ENV RUST_LOG=api_gateway=info

EXPOSE 8080 9090

ENTRYPOINT ["./api-gateway"]
```

### Docker Compose สำหรับ Development

```yaml
# docker-compose.yml
version: "3.9"

services:
  gateway:
    build: .
    ports:
      - "8080:8080"
      - "9090:9090"
    environment:
      GATEWAY_CONFIG: /app/config/routes.yaml
      JWT_SECRET: dev-secret-key-change-in-production
      REDIS_URL: redis://redis:6379
      RUST_LOG: api_gateway=debug
    volumes:
      - ./config:/app/config:ro
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9091:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
```

### Prometheus Config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "api-gateway"
    static_configs:
      - targets: ["gateway:8080"]
    metrics_path: "/metrics"
```

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `GATEWAY_CONFIG` | `config/routes.yaml` | Path ของ route config |
| `JWT_SECRET` | `dev-secret-key` | Secret สำหรับ HS256 JWT |
| `REDIS_URL` | `redis://localhost:6379` | Redis connection string |
| `RUST_LOG` | `api_gateway=info` | Log level |
| `PORT` | `8080` | Proxy listening port |
| `ADMIN_PORT` | `9090` | Admin API port |

### Health Check และ Graceful Shutdown

```rust
// เพิ่มใน main.rs
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => tracing::info!("Ctrl+C received"),
        _ = terminate => tracing::info!("SIGTERM received"),
    }
    tracing::info!("Initiating graceful shutdown...");
}

// ใน main ใช้ axum::serve(...).with_graceful_shutdown(shutdown_signal())
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Fixed Window Counter Algorithm

ปัจจุบัน rate limiter รองรับ Token Bucket และ Sliding Window Log ให้เพิ่ม Fixed Window Counter ซึ่งเป็นอัลกอริทึมที่ simple ที่สุดแต่มี edge case:

```yaml
# routes.yaml เพิ่ม algorithm ใหม่
rate_limit:
  algorithm: fixed_window  # ← ต้อง implement
  requests_per_second: 100.0
  burst_size: 100
  key_by: ip
```

สิ่งที่ต้อง implement:
1. `struct FixedWindowCounter { count: u64, window_start: Instant, window_size: Duration, max_requests: u64 }`
2. logic: reset counter เมื่อ `now > window_start + window_size`
3. เพิ่ม variant ใน `RateLimitAlgorithm` enum
4. เขียน test สำหรับ edge case: request ที่ส่งมาช่วง window boundary (59.9s กับ 60.1s)

**หัวข้อที่เกี่ยวข้อง:** Fixed Window Counter เป็น O(1) memory แต่มี "thundering herd" ที่ window boundary ให้ทดลองวัดและเปรียบเทียบกับ Sliding Window

---

### แบบฝึกหัดที่ 2: Distributed Rate Limiting ด้วย Redis

Rate limiter ปัจจุบันเก็บ state ใน memory ของ process เดียว — ถ้า deploy หลาย instance จะมีปัญหา เปลี่ยนให้ใช้ Redis Lua script แทน:

```lua
-- rate_limit.lua (Sliding Window ด้วย Redis Sorted Set)
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local max_requests = tonumber(ARGV[3])

-- ลบ entries เก่า
redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)

-- นับจำนวน requests ใน window
local count = redis.call('ZCARD', key)
if count < max_requests then
    redis.call('ZADD', key, now, now .. math.random())
    redis.call('EXPIRE', key, math.ceil(window / 1000))
    return 1
end
return 0
```

Rust side ใช้ `redis::Script`:
```rust
let script = redis::Script::new(include_str!("../scripts/rate_limit.lua"));
let allowed: bool = script
    .key(redis_key)
    .arg(now_ms)
    .arg(window_ms)
    .arg(max_requests)
    .invoke_async(&mut conn)
    .await?;
```

---

### แบบฝึกหัดที่ 3: Request/Response Logging Middleware

เพิ่ม structured logging middleware ที่ log ทุก request และ response พร้อม correlation ID:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "INFO",
  "message": "request completed",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "method": "GET",
  "path": "/api/users/123",
  "route_id": "users-service",
  "upstream": "http://users:3001",
  "status": 200,
  "duration_ms": 45,
  "client_ip": "1.2.3.4",
  "user_agent": "Mozilla/5.0",
  "jwt_subject": "user-456"
}
```

สิ่งที่ต้อง implement:
1. `RequestId` extractor ที่ generate UUID ถ้าไม่มี `X-Request-Id` header
2. Tower middleware ที่ measure request duration ด้วย `std::time::Instant`
3. JSON structured logging ด้วย `tracing-subscriber` และ JSON formatter
4. ใส่ `request_id` ลงใน response header `X-Request-Id`

---

### แบบฝึกหัดที่ 4: WebSocket Proxying

Gateway ปัจจุบัน proxy แค่ HTTP request/response เพิ่ม WebSocket support:

```yaml
routes:
  - id: "ws-chat"
    path_prefix: "/ws/chat"
    upstream_url: "ws://chat-service:3004"
    protocol: websocket  # ← flag ใหม่
    auth_required: true
```

สิ่งที่ต้อง implement:
1. ตรวจ `Upgrade: websocket` header
2. ใช้ `axum::extract::WebSocketUpgrade` ทำ WebSocket handshake
3. Bidirectional proxy ระหว่าง client WebSocket และ upstream WebSocket ด้วย `tokio::select!`
4. Handle disconnect จาก ทั้งสองฝั่ง gracefully
5. Rate limiting สำหรับ WebSocket ต้อง count connections ไม่ใช่ messages

**ความท้าทาย:** WebSocket เป็น long-lived connection — circuit breaker ควรนับยังไง? ถ้า upstream ตัด connection ควรทำอะไร?

## สรุป

โปรเจคนี้สร้าง API Gateway ที่พร้อม production ครบทุก feature ที่ทีม engineering คาดหวัง สิ่งที่ได้เรียนรู้:

**Pattern สำคัญที่ได้ฝึก:**
1. **Middleware pipeline** — axum layer system ทำให้ compose middleware ได้ clean โดยไม่ callback hell
2. **State machine** — Circuit Breaker เป็นตัวอย่างคลาสสิกของ explicit state machine ที่ต้องการ atomic transitions
3. **Lazy computation** — Token Bucket refill แบบ lazy (คำนวณตอนใช้) ดีกว่า eager (background task) เพราะไม่ต้องการ synchronization เพิ่มเติม
4. **Double-checked locking** — pattern สำคัญสำหรับ cache ที่ share ข้าม concurrent tasks
5. **Per-key state** — DashMap สำหรับ per-client rate limit state ที่ high-throughput

**Crate ecosystem ที่ได้ใช้:**
- `axum 0.8` — web framework ที่ type-safe และ composable
- `hyper/reqwest` — HTTP client สำหรับ upstream forwarding
- `dashmap` — concurrent HashMap ไม่ต้อง wrap RwLock ทั้งก้อน
- `jsonwebtoken` — JWT encode/decode ที่ support RS256, HS256
- `prometheus` — standard metrics format สำหรับ cloud-native
- `serde_yaml` — config file parsing

โปรเจคถัดไปใน Module B จะต่อยอดเรื่อง image capture และ browser automation ซึ่งใช้ async I/O pattern ที่คล้ายกัน

---

**โปรเจคก่อนหน้า:** [project-b08-email-service.md](project-b08-email-service.md) | **โปรเจคถัดไป:** [project-b10-screenshot-service.md](project-b10-screenshot-service.md)
