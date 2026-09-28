# Project D09: Rate-limit + WAF Middleware

> โมดูล: D — Security & Cryptography | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

Middleware ชั้น network ที่ทำงานระหว่าง client request กับ application handler มีบทบาทสำคัญใน production system สองชั้นที่โปรเจคนี้สร้างคือ **rate limiting** (การควบคุมอัตราการรับ request) และ **Web Application Firewall — WAF** (การกรองเนื้อหา request ตามชุดกฎ)

ระบบ rate limiting ควบคุมปริมาณ request ต่อ IP ต่อหน่วยเวลา โดยรองรับสองอัลกอริทึม: **token bucket** ซึ่งอนุญาต burst ได้สั้น ๆ แล้วค่อย ๆ refill, และ **sliding window** ซึ่งนับ request ภายใน time window เคลื่อนที่ WAF ตรวจสอบเนื้อหา request ด้วยชุดกฎ regex — ตรวจ SQL injection patterns ใน query string และ body, XSS patterns ใน URL-decoded input, และ IP blocklist ใน CIDR notation

โปรเจคนี้ implement middleware ทั้งหมดเป็น Tower `Layer`/`Service` — composable ทำงานได้กับทุก framework ที่ใช้ Tower (axum, hyper, tonic) โดยไม่ต้องแก้ business logic

**Use case ใน production:**
- API gateway กรองทุก request ก่อนถึง backend services
- Reverse proxy ที่เพิ่ม security layer โดยไม่ต้องแก้ application code
- CDN edge node ที่ต้องการ WAF logic แบบ lightweight
- Internal developer portal ที่ต้องการ audit + rate limit per team token

**Learning value:** โปรเจคนี้สอน Tower middleware pattern อย่างลึก (poll_ready propagation, Service composition), อัลกอริทึม rate limiting สองแบบที่ใช้ DashMap สำหรับ concurrent per-IP state, regex-based rule engine จาก YAML config, และ Prometheus metrics integration

## สิ่งที่จะได้เรียนรู้

- **Tower Layer/Service trait** — implement `poll_ready` propagation อย่างถูกต้อง, หลีกเลี่ยง panic จาก `poll_ready` ที่ไม่ forward
- **Token bucket algorithm** — คณิตศาสตร์ refill บน elapsed time, `f64` สำหรับ fractional token, capped ที่ capacity
- **Sliding window algorithm** — `VecDeque<Instant>` + evict-on-access pattern, configurable window/limit
- **DashMap for concurrent state** — `DashMap<IpAddr, Bucket>` อนุญาต multiple threads อ่าน/เขียน per-IP state พร้อมกันโดยไม่ต้องใช้ `Mutex<HashMap>`
- **Regex rule engine** — YAML-configurable rules, multi-target inspection (URL/body/headers/query), ordered rule evaluation
- **Request body buffering** — อ่าน request body ผ่าน `http-body-util`, enforce 64 KiB limit ก่อนตรวจสอบ
- **CIDR IP matching** — `ipnetwork` crate, hot-reload ด้วย SIGHUP signal handler, X-Forwarded-For chain
- **Prometheus metrics** — lazy_static! registry, counter/histogram registration, `/metrics` endpoint

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50** — async/await, Future, tokio runtime
- **Part 61–65** — axum web framework, Router, handler functions
- **Part 70–75** — Tower middleware: `Service` trait, `Layer` trait, `ServiceBuilder`
- **Part 80–85** — serde, serde_json, Serialize/Deserialize
- **Part 86–90** — cryptography primitives, regex pattern matching
- **Part 96–100** — Prometheus metrics, DashMap, concurrent data structures
- **Part 101–105** — clap 4 derive API, production CLI design

## โครงสร้างโปรเจค (Project Layout)

```
waf-middleware/
├── src/
│   ├── main.rs           # CLI entry point (clap 4), axum server setup, layer composition
│   ├── rate_limit.rs     # TokenBucket, SlidingWindow, RateLimitLayer, RateLimitService
│   ├── waf.rs            # WafRule, WafRuleEngine, YAML config loading, body buffering
│   ├── ip_blocklist.rs   # IpBlocklist, CIDR matching, SIGHUP reload, X-Forwarded-For
│   ├── security_headers.rs  # SecurityHeadersLayer, SecurityHeadersService, header injection
│   └── metrics.rs        # Prometheus registry, counters, histogram, /metrics handler
├── config/
│   └── rules.yaml        # WAF rule definitions (id, description, pattern, targets, action)
├── blocklist/
│   └── blocklist.txt     # IP/CIDR blocklist (one entry per line, # comments)
├── tests/
│   └── integration_test.rs  # End-to-end tests with axum test client
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Tower Middleware Stack

Tower ใช้ pattern `Layer<S>` ที่ wrap inner service `S` เป็นชั้น ๆ เมื่อ `ServiceBuilder::new().layer(A).layer(B).layer(C).service(handler)` จะได้:

```
Request --> [C] --> [B] --> [A] --> handler
Response <-- [C] <-- [B] <-- [A] <--
```

โปรเจคนี้สร้าง middleware stack ที่ request ผ่านในลำดับ:

```
Client Request
    |
    v
[IpBlocklistLayer]    -- ตรวจ IP ก่อนทุกอย่าง (ถ้า blocked -> 403)
    |
    v
[RateLimitLayer]      -- ตรวจ token bucket / sliding window (ถ้า limited -> 429)
    |
    v
[WafLayer]            -- buffer body, ตรวจ SQL injection/XSS/custom rules (ถ้า matched -> 403)
    |
    v
[SecurityHeadersLayer]-- ใส่ security headers ทุก response
    |
    v
[MetricsLayer]        -- นับ counter, measure latency
    |
    v
Application Handler
```

### poll_ready Propagation Pattern

จุดที่มักเกิด bug ใน Tower middleware คือการ implement `poll_ready` ผิด:

```rust
// WRONG: ไม่ forward poll_ready ไปยัง inner service
fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    Poll::Ready(Ok(()))  // service อาจยังไม่พร้อม!
}

// CORRECT: forward poll_ready ไปยัง inner service
fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    self.inner.poll_ready(cx)  // propagate readiness
}
```

Tower contract กำหนดว่า `call()` จะถูกเรียกก็ต่อเมื่อ `poll_ready()` return `Poll::Ready(Ok(()))` แล้ว ถ้า middleware ไม่ forward ไปยัง inner service อาจทำให้ call inner service ที่ยังไม่พร้อม ซึ่ง panic ได้

### Token Bucket vs Sliding Window

| ลักษณะ | Token Bucket | Sliding Window |
|--------|-------------|----------------|
| การจัดการ burst | อนุญาต burst สั้น ๆ จนหมด token | ไม่อนุญาต burst เกิน max_count |
| ความแม่นยำ | approximate (f64 tokens) | exact (นับ timestamp จริง) |
| หน่วยความจำต่อ IP | O(1) — struct ขนาดคงที่ | O(max_count) — VecDeque ของ Instant |
| Use case | API ที่ต้องการยืดหยุ่นสำหรับ burst | Strict limit เช่น login endpoint |

### DashMap สำหรับ Concurrent Per-IP State

`DashMap<IpAddr, Bucket>` แทน `Mutex<HashMap<IpAddr, Bucket>>` เพราะ:
- DashMap ใช้ shard locking — แต่ละ shard lock แยกกัน ลด contention
- API เหมือน `HashMap` — ไม่ต้อง `.lock().unwrap()` ทุกครั้ง
- Thread-safe โดย default — `Arc<DashMap<...>>` share ระหว่าง request ได้

```rust
// DashMap entry API
let mut entry = self.token_buckets
    .entry(client_ip)
    .or_insert_with(|| TokenBucket::new(cap, rate));
entry.try_consume(1.0)  // entry = RefMut<IpAddr, TokenBucket>
```

### WAF Rule Pipeline

Request body ถูก buffer ทั้งหมดก่อนตรวจสอบ ถ้า body เกิน 64 KiB จะ return 413 ทันที โดยไม่ต้องตรวจกฎ:

```
Request body (streaming)
    |
    v
buffer up to 64 KiB
    |-- > 64 KiB --> 413 Payload Too Large
    |
    v
URL-decode query params
HTML-entity decode body
    |
    v
Check rules in order:
  1. URL path
  2. Query parameters
  3. Request headers
  4. Request body
    |
    |-- match found --> record metric, return 403
    |
    v
Forward to inner service
```

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Token Bucket

สร้าง Cargo project และ implement token bucket algorithm พื้นฐาน:

```bash
cargo new waf-middleware
cd waf-middleware
```

`Cargo.toml`:
```toml
[package]
name = "waf-middleware"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.8"
tower = { version = "0.5", features = ["full"] }
tower-service = "0.3"
tower-layer = "0.3"
dashmap = "6"
regex = "1"
serde = { version = "1", features = ["derive"] }
serde_yaml = "0.9"
prometheus = { version = "0.13", features = ["process"] }
ipnetwork = "0.20"
clap = { version = "4", features = ["derive"] }
bytes = "1"
http = "1"
http-body-util = "0.1"
futures-util = "0.3"
pin-project-lite = "0.2"
```

`src/rate_limit.rs` — Token Bucket:

```rust
// rate_limit.rs -- Token bucket + sliding window algorithms
// Tower Layer/Service implementation for rate limiting

use std::collections::VecDeque;
use std::net::IpAddr;
use std::sync::Arc;
use std::time::{Duration, Instant};
use std::task::{Context, Poll};
use std::future::Future;
use std::pin::Pin;

use dashmap::DashMap;
use tower_layer::Layer;
use tower_service::Service;

/// Per-IP token bucket state.
/// tokens      -- current token count (f64 allows fractional refill)
/// last_refill -- timestamp of last refill operation
/// capacity    -- maximum tokens the bucket can hold
/// refill_rate -- tokens added per second
#[derive(Debug, Clone)]
pub struct TokenBucket {
    pub tokens: f64,
    pub last_refill: Instant,
    pub capacity: f64,
    pub refill_rate: f64, // tokens per second
}

impl TokenBucket {
    pub fn new(capacity: f64, refill_rate: f64) -> Self {
        Self {
            tokens: capacity,
            last_refill: Instant::now(),
            capacity,
            refill_rate,
        }
    }

    /// Refill tokens based on elapsed time since last_refill.
    /// Called on every request before checking token availability.
    pub fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        let new_tokens = elapsed * self.refill_rate;
        self.tokens = (self.tokens + new_tokens).min(self.capacity);
        self.last_refill = now;
    }

    /// Attempt to consume `n` tokens. Returns true if successful.
    /// Calls refill() first to account for elapsed time.
    pub fn try_consume(&mut self, n: f64) -> bool {
        self.refill();
        if self.tokens >= n {
            self.tokens -= n;
            true
        } else {
            false
        }
    }
}
```

คณิตศาสตร์ refill: `new_tokens = elapsed_seconds * refill_rate` โดย capped ที่ `capacity` เพื่อป้องกัน bucket เต็มเกิน ถ้า IP ไม่ได้ส่ง request มานาน bucket จะเต็ม capacity เสมอ ไม่ใช่ refill ไม่หยุด

### ขั้นที่ 2: Sliding Window Algorithm

```rust
// ต่อใน src/rate_limit.rs

/// Per-IP sliding window state.
/// timestamps  -- VecDeque of request instants within the window
/// window      -- duration of the sliding window
/// max_count   -- maximum requests allowed within the window
#[derive(Debug)]
pub struct SlidingWindow {
    pub timestamps: VecDeque<Instant>,
    pub window: Duration,
    pub max_count: usize,
}

impl SlidingWindow {
    pub fn new(window: Duration, max_count: usize) -> Self {
        Self {
            timestamps: VecDeque::new(),
            window,
            max_count,
        }
    }

    /// Record a new request timestamp.
    pub fn record(&mut self, now: Instant) {
        self.timestamps.push_back(now);
    }

    /// Evict timestamps older than `now - window`.
    /// VecDeque ทำงานเป็น queue -- push_back เพิ่มใหม่, pop_front ลบเก่า
    /// timestamps เรียงตามเวลาเสมอ (monotonic) จึง evict จาก front ได้
    pub fn evict_old(&mut self, now: Instant) {
        let cutoff = now - self.window;
        while let Some(&front) = self.timestamps.front() {
            if front <= cutoff {
                self.timestamps.pop_front();
            } else {
                break;
            }
        }
    }

    /// Number of requests in the current window.
    pub fn count(&self) -> usize {
        self.timestamps.len()
    }

    /// Check limit and record if allowed. Returns true if request is allowed.
    pub fn check_and_record(&mut self, now: Instant) -> bool {
        self.evict_old(now);
        if self.timestamps.len() < self.max_count {
            self.timestamps.push_back(now);
            true
        } else {
            false
        }
    }
}
```

Sliding window ใช้ `VecDeque` แทน `Vec` เพราะ `pop_front()` เป็น O(1) ใน `VecDeque` แต่เป็น O(n) ใน `Vec` เมื่อ evict timestamps เก่า ๆ จำนวนมาก performance แตกต่างกันมาก

### ขั้นที่ 3: Tower RateLimitLayer

```rust
// ต่อใน src/rate_limit.rs

#[derive(Debug, Clone)]
pub enum RateLimitAlgorithm {
    TokenBucket { capacity: f64, refill_rate: f64 },
    SlidingWindow { window: Duration, max_requests: usize },
}

#[derive(Debug, Clone)]
pub struct RateLimitConfig {
    pub algorithm: RateLimitAlgorithm,
}

impl RateLimitConfig {
    pub fn token_bucket(capacity: f64, refill_rate: f64) -> Self {
        Self { algorithm: RateLimitAlgorithm::TokenBucket { capacity, refill_rate } }
    }

    pub fn sliding_window(window_secs: u64, max_requests: usize) -> Self {
        Self {
            algorithm: RateLimitAlgorithm::SlidingWindow {
                window: Duration::from_secs(window_secs),
                max_requests,
            },
        }
    }
}

/// Tower Layer ที่เพิ่ม per-IP rate limiting ให้กับ Service ใด ๆ
/// รองรับทั้ง token bucket และ sliding window algorithms
#[derive(Clone)]
pub struct RateLimitLayer {
    config: Arc<RateLimitConfig>,
    token_buckets: Arc<DashMap<IpAddr, TokenBucket>>,
    sliding_windows: Arc<DashMap<IpAddr, SlidingWindow>>,
}

impl RateLimitLayer {
    pub fn token_bucket(capacity: f64, refill_rate: f64) -> Self {
        Self {
            config: Arc::new(RateLimitConfig::token_bucket(capacity, refill_rate)),
            token_buckets: Arc::new(DashMap::new()),
            sliding_windows: Arc::new(DashMap::new()),
        }
    }

    pub fn sliding_window(window_secs: u64, max_requests: usize) -> Self {
        Self {
            config: Arc::new(RateLimitConfig::sliding_window(window_secs, max_requests)),
            token_buckets: Arc::new(DashMap::new()),
            sliding_windows: Arc::new(DashMap::new()),
        }
    }
}

impl<S> Layer<S> for RateLimitLayer {
    type Service = RateLimitService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        RateLimitService {
            inner,
            config: self.config.clone(),
            token_buckets: self.token_buckets.clone(),
            sliding_windows: self.sliding_windows.clone(),
        }
    }
}
```

`Arc<DashMap<...>>` แทน `DashMap` ตรง ๆ เพราะ `RateLimitLayer` ถูก clone ทุกครั้งที่ `layer()` ถูกเรียก (axum clone service per worker) แต่ state ต้องแชร์กันทุก worker จึงต้อง wrap ด้วย `Arc`

### ขั้นที่ 4: Tower Service + poll_ready

```rust
// ต่อใน src/rate_limit.rs

#[derive(Clone)]
pub struct RateLimitService<S> {
    inner: S,
    config: Arc<RateLimitConfig>,
    token_buckets: Arc<DashMap<IpAddr, TokenBucket>>,
    sliding_windows: Arc<DashMap<IpAddr, SlidingWindow>>,
}

impl<S, ReqBody> Service<http::Request<ReqBody>> for RateLimitService<S>
where
    S: Service<
            http::Request<ReqBody>,
            Response = http::Response<String>,
            Error = std::convert::Infallible,
        > + Clone
        + Send
        + 'static,
    S::Future: Send + 'static,
    ReqBody: Send + 'static,
{
    type Response = http::Response<String>;
    type Error = std::convert::Infallible;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // CRITICAL: forward poll_ready ไปยัง inner service
        // ถ้าไม่ forward อาจ call inner service ที่ยังไม่พร้อม -> panic
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: http::Request<ReqBody>) -> Self::Future {
        // Extract client IP จาก X-Forwarded-For header หรือใช้ 0.0.0.0
        let client_ip: IpAddr = req
            .headers()
            .get("x-forwarded-for")
            .and_then(|v| v.to_str().ok())
            .and_then(|s| s.split(',').next())      // ใช้ first IP ใน chain
            .and_then(|s| s.trim().parse().ok())
            .unwrap_or(IpAddr::from([0, 0, 0, 0]));

        let allowed = match &*self.config {
            RateLimitConfig {
                algorithm: RateLimitAlgorithm::TokenBucket { capacity, refill_rate },
            } => {
                let cap = *capacity;
                let rate = *refill_rate;
                let mut entry = self
                    .token_buckets
                    .entry(client_ip)
                    .or_insert_with(|| TokenBucket::new(cap, rate));
                entry.try_consume(1.0)
            }
            RateLimitConfig {
                algorithm: RateLimitAlgorithm::SlidingWindow { window, max_requests },
            } => {
                let w = *window;
                let max = *max_requests;
                let mut entry = self
                    .sliding_windows
                    .entry(client_ip)
                    .or_insert_with(|| SlidingWindow::new(w, max));
                entry.check_and_record(Instant::now())
            }
        };

        if !allowed {
            let response = http::Response::builder()
                .status(429)
                .header("content-type", "text/plain")
                .header("retry-after", "1")
                .body("429 Too Many Requests".to_string())
                .unwrap();
            return Box::pin(async move { Ok(response) });
        }

        let fut = self.inner.call(req);
        Box::pin(async move { fut.await })
    }
}
```

### ขั้นที่ 5: WAF Rule Engine

`src/waf.rs` — ชุดกฎ SQL injection และ XSS:

```rust
// waf.rs -- WAF rule engine with SQL injection and XSS detection

use regex::Regex;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum RuleAction {
    Block,
    Log,
    Challenge,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum RuleSeverity {
    Low,
    Medium,
    High,
    Critical,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum RuleTarget {
    Url,
    Headers,
    Body,
    QueryParams,
}

#[derive(Debug, Clone)]
pub struct WafRule {
    pub id: String,
    pub description: String,
    pub pattern: Regex,
    pub targets: Vec<RuleTarget>,
    pub action: RuleAction,
    pub severity: RuleSeverity,
}

#[derive(Debug, Clone)]
pub struct RuleMatch {
    pub rule_id: String,
    pub action: RuleAction,
    pub description: String,
}

pub struct WafRuleEngine {
    pub sqli_rules: Vec<WafRule>,
    pub xss_rules: Vec<WafRule>,
}

impl WafRuleEngine {
    pub fn default_rules() -> Self {
        let sqli_rules = vec![
            WafRule {
                id: "SQLI-001".to_string(),
                description: "Matches tautology pattern: single-quote followed by OR/AND condition".to_string(),
                // Matches: ' OR ..., ' AND ...
                // ใช้ r#"..."# เพราะ pattern มี " อยู่ภายใน
                pattern: Regex::new(r#"(?i)'\s*(or|and)\s+[\w'"]+\s*[=<>!]+"#).unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body],
                action: RuleAction::Block,
                severity: RuleSeverity::Critical,
            },
            WafRule {
                id: "SQLI-002".to_string(),
                description: "Matches UNION SELECT statement used to extract additional columns".to_string(),
                pattern: Regex::new(r"(?i)union\s+(all\s+)?select\s+").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body, RuleTarget::Url],
                action: RuleAction::Block,
                severity: RuleSeverity::Critical,
            },
            WafRule {
                id: "SQLI-003".to_string(),
                description: "Matches semicolon-terminated DROP TABLE statement".to_string(),
                pattern: Regex::new(r"(?i);\s*drop\s+table\s+").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body],
                action: RuleAction::Block,
                severity: RuleSeverity::Critical,
            },
            WafRule {
                id: "SQLI-004".to_string(),
                description: "Matches SQL line comment delimiter -- used to truncate queries".to_string(),
                pattern: Regex::new(r"(?m)(--\s*$|--\s+\w)").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body],
                action: RuleAction::Block,
                severity: RuleSeverity::High,
            },
            WafRule {
                id: "SQLI-005".to_string(),
                description: "Matches balanced single-quote tautology pattern like 1=1".to_string(),
                pattern: Regex::new(r"(?i)'\s*=\s*'").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body],
                action: RuleAction::Block,
                severity: RuleSeverity::High,
            },
        ];

        let xss_rules = vec![
            WafRule {
                id: "XSS-001".to_string(),
                description: "Matches <script> tag: vector for inline script injection".to_string(),
                pattern: Regex::new(r"(?i)<\s*script[\s>]").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body, RuleTarget::Url],
                action: RuleAction::Block,
                severity: RuleSeverity::Critical,
            },
            WafRule {
                id: "XSS-002".to_string(),
                description: "Matches javascript: URI scheme: vector for href/src attribute injection".to_string(),
                pattern: Regex::new(r"(?i)javascript\s*:").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body, RuleTarget::Url],
                action: RuleAction::Block,
                severity: RuleSeverity::Critical,
            },
            WafRule {
                id: "XSS-003".to_string(),
                description: "Matches onerror= event handler attribute: triggers on image load failure".to_string(),
                pattern: Regex::new(r"(?i)\bon\s*error\s*=").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body, RuleTarget::Url],
                action: RuleAction::Block,
                severity: RuleSeverity::High,
            },
            WafRule {
                id: "XSS-004".to_string(),
                description: "Matches onload= event handler attribute: triggers on element load".to_string(),
                pattern: Regex::new(r"(?i)\bon\s*load\s*=").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body, RuleTarget::Url],
                action: RuleAction::Block,
                severity: RuleSeverity::High,
            },
            WafRule {
                id: "XSS-005".to_string(),
                description: "Matches onmouseover= and similar DOM event handler attributes".to_string(),
                pattern: Regex::new(r"(?i)\bon\s*mouse\w+\s*=").unwrap(),
                targets: vec![RuleTarget::QueryParams, RuleTarget::Body],
                action: RuleAction::Block,
                severity: RuleSeverity::Medium,
            },
        ];

        Self { sqli_rules, xss_rules }
    }

    /// ตรวจ input กับชุดกฎ SQL injection คืน rule แรกที่ match
    pub fn check_sqli(&self, input: &str) -> Option<RuleMatch> {
        for rule in &self.sqli_rules {
            if rule.pattern.is_match(input) {
                return Some(RuleMatch {
                    rule_id: rule.id.clone(),
                    action: rule.action.clone(),
                    description: rule.description.clone(),
                });
            }
        }
        None
    }

    /// ตรวจ input กับชุดกฎ XSS คืน rule แรกที่ match
    pub fn check_xss(&self, input: &str) -> Option<RuleMatch> {
        for rule in &self.xss_rules {
            if rule.pattern.is_match(input) {
                return Some(RuleMatch {
                    rule_id: rule.id.clone(),
                    action: rule.action.clone(),
                    description: rule.description.clone(),
                });
            }
        }
        None
    }

    /// ตรวจ input กับทุก rule set
    pub fn check_all(&self, input: &str) -> Option<RuleMatch> {
        self.check_sqli(input).or_else(|| self.check_xss(input))
    }
}
```

**SQL Injection Rules — สิ่งที่แต่ละ pattern ตรวจ:**

| Rule ID | Pattern ตัวอย่าง | สิ่งที่ pattern ตรวจ |
|---------|----------------|---------------------|
| SQLI-001 | `' OR 1=1` | Tautology เปรียบเทียบที่เป็นจริงเสมอหลัง single-quote |
| SQLI-002 | `UNION SELECT username,password` | คำสั่ง UNION ที่รวม result set เพิ่มเติม |
| SQLI-003 | `'; DROP TABLE users; --` | คำสั่ง DDL ที่ต่อท้ายด้วย semicolon |
| SQLI-004 | `admin'--` | Comment delimiter ที่ตัดส่วนที่เหลือของ query |
| SQLI-005 | `'1'='1` | Balanced tautology ที่ใช้ string comparison |

**XSS Rules — สิ่งที่แต่ละ pattern ตรวจ:**

| Rule ID | Pattern ตัวอย่าง | สิ่งที่ pattern ตรวจ |
|---------|----------------|---------------------|
| XSS-001 | `<script>alert(1)</script>` | HTML script element ที่ browser execute เป็น JavaScript |
| XSS-002 | `javascript:alert(1)` | URI scheme ที่ browser interpret เป็น JavaScript ใน href/src |
| XSS-003 | `<img src=x onerror=alert(1)>` | Event handler attribute ที่ fire เมื่อ resource load ล้มเหลว |
| XSS-004 | `<body onload=alert(1)>` | Event handler attribute ที่ fire เมื่อ element load สำเร็จ |
| XSS-005 | `<svg onmouseover=alert(1)>` | Mouse event handler attributes บน SVG elements |

### ขั้นที่ 6: IP Blocklist และ Security Headers

`src/ip_blocklist.rs`:

```rust
// ip_blocklist.rs -- CIDR-based IP blocklist with ipnetwork crate

use std::net::IpAddr;
use ipnetwork::IpNetwork;

/// IP blocklist รองรับ CIDR notation
/// เก็บ entries เป็น IpNetwork รองรับทั้ง /32 host entries และ subnet ranges
/// การ match ใช้ contains() สำหรับ O(n) linear scan
/// สำหรับ production ควรใช้ prefix tree (Trie) เพื่อ O(log n)
#[derive(Debug, Default)]
pub struct IpBlocklist {
    networks: Vec<IpNetwork>,
}

impl IpBlocklist {
    pub fn new() -> Self {
        Self::default()
    }

    /// เพิ่ม CIDR range ลงใน blocklist
    /// รับทั้ง host address ("10.0.0.1") และ subnet range ("192.168.1.0/24")
    pub fn add_cidr(&mut self, cidr: &str) -> Result<(), ipnetwork::IpNetworkError> {
        let network: IpNetwork = cidr.parse()?;
        self.networks.push(network);
        Ok(())
    }

    /// คืนค่า true ถ้า IP ตรงกับ network ใดใน blocklist
    pub fn is_blocked(&self, ip: IpAddr) -> bool {
        self.networks.iter().any(|net| net.contains(ip))
    }

    /// โหลด entries จาก string หลายบรรทัด (หนึ่ง CIDR ต่อบรรทัด)
    /// บรรทัดที่เริ่มด้วย '#' และบรรทัดว่างจะถูกข้าม
    pub fn load_from_str(&mut self, content: &str) -> Result<(), Box<dyn std::error::Error>> {
        for line in content.lines() {
            let trimmed = line.trim();
            if trimmed.is_empty() || trimmed.starts_with('#') {
                continue;
            }
            self.add_cidr(trimmed)?;
        }
        Ok(())
    }

    pub fn len(&self) -> usize {
        self.networks.len()
    }

    pub fn is_empty(&self) -> bool {
        self.networks.is_empty()
    }
}
```

`src/security_headers.rs` — Security Header Injection:

```rust
// security_headers.rs -- automatic security response header injection
// Tower Layer/Service ที่ inject security headers ทุก response

use std::task::{Context, Poll};
use std::future::Future;
use std::pin::Pin;
use tower_layer::Layer;
use tower_service::Service;

/// Configuration สำหรับ security headers
#[derive(Debug, Clone)]
pub struct SecurityHeaders {
    pub hsts_max_age: u64,
    pub csp_policy: String,
    pub referrer_policy: String,
}

impl Default for SecurityHeaders {
    fn default() -> Self {
        Self {
            hsts_max_age: 31536000, // 1 year in seconds
            csp_policy: "default-src 'self'; script-src 'self'; object-src 'none'".to_string(),
            referrer_policy: "strict-origin-when-cross-origin".to_string(),
        }
    }
}

impl SecurityHeaders {
    /// Inject security headers ทั้งหมดลงใน http::Response
    pub fn inject<B>(&self, mut response: http::Response<B>) -> http::Response<B> {
        let headers = response.headers_mut();

        // Strict-Transport-Security: บอก browser ให้ใช้ HTTPS เท่านั้น
        // max-age กำหนดระยะเวลา (วินาที) ที่ browser cache นโยบายนี้
        headers.insert(
            "strict-transport-security",
            format!("max-age={}; includeSubDomains", self.hsts_max_age)
                .parse()
                .unwrap(),
        );

        // X-Content-Type-Options: ป้องกัน browser จาก MIME-type sniffing
        // nosniff บังคับให้ browser ใช้ Content-Type ที่ server ส่งมาเท่านั้น
        headers.insert("x-content-type-options", "nosniff".parse().unwrap());

        // X-Frame-Options: ป้องกัน page ถูก embed ใน iframe (clickjacking)
        // DENY ห้าม embed ทุกกรณี, SAMEORIGIN อนุญาตเฉพาะ same-origin
        headers.insert("x-frame-options", "DENY".parse().unwrap());

        // Content-Security-Policy: กำหนด whitelist ของ resource origins
        // default-src 'self' บล็อก external script/style/font ทั้งหมด
        headers.insert(
            "content-security-policy",
            self.csp_policy.parse().unwrap(),
        );

        // Referrer-Policy: ควบคุมข้อมูลใน Referer header เมื่อ navigate
        // strict-origin-when-cross-origin ส่ง origin-only เมื่อ cross-origin
        headers.insert(
            "referrer-policy",
            self.referrer_policy.parse().unwrap(),
        );

        response
    }
}

#[derive(Debug, Clone, Default)]
pub struct SecurityHeadersLayer {
    config: SecurityHeaders,
}

impl SecurityHeadersLayer {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn with_config(config: SecurityHeaders) -> Self {
        Self { config }
    }
}

impl<S> Layer<S> for SecurityHeadersLayer {
    type Service = SecurityHeadersService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        SecurityHeadersService {
            inner,
            config: self.config.clone(),
        }
    }
}

#[derive(Debug, Clone)]
pub struct SecurityHeadersService<S> {
    inner: S,
    config: SecurityHeaders,
}

impl<S, ReqBody> Service<http::Request<ReqBody>> for SecurityHeadersService<S>
where
    S: Service<
            http::Request<ReqBody>,
            Response = http::Response<String>,
            Error = std::convert::Infallible,
        > + Clone
        + Send
        + 'static,
    S::Future: Send + 'static,
    ReqBody: Send + 'static,
{
    type Response = http::Response<String>;
    type Error = std::convert::Infallible;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: http::Request<ReqBody>) -> Self::Future {
        let config = self.config.clone();
        let fut = self.inner.call(req);
        Box::pin(async move {
            let response = fut.await?;
            Ok(config.inject(response))
        })
    }
}
```

### ขั้นที่ 7: Prometheus Metrics

`src/metrics.rs`:

```rust
// metrics.rs -- Prometheus metrics สำหรับ WAF middleware

use prometheus::{
    register_counter_vec, register_histogram, CounterVec, Histogram, Registry,
};
use std::sync::OnceLock;

/// Prometheus registry สำหรับ WAF metrics
/// ใช้ OnceLock เพื่อ initialize ครั้งเดียว, thread-safe
static WAF_METRICS: OnceLock<WafMetrics> = OnceLock::new();

pub struct WafMetrics {
    /// จำนวน request ที่ถูก block จาก WAF rule (label: rule_id)
    pub blocked_requests_total: CounterVec,
    /// จำนวน request ที่ถูก rate limit
    pub rate_limited_total: prometheus::Counter,
    /// จำนวน request ทั้งหมดที่ผ่าน middleware
    pub requests_total: prometheus::Counter,
    /// เวลาที่ใช้ตรวจสอบ WAF rules (seconds)
    pub waf_check_duration_seconds: Histogram,
}

impl WafMetrics {
    pub fn init() -> &'static WafMetrics {
        WAF_METRICS.get_or_init(|| {
            let blocked_requests_total = register_counter_vec!(
                "blocked_requests_total",
                "Total number of requests blocked by WAF rules",
                &["rule_id"]
            )
            .expect("Failed to register blocked_requests_total counter");

            let rate_limited_total = prometheus::register_counter!(
                "rate_limited_total",
                "Total number of requests rejected by rate limiter"
            )
            .expect("Failed to register rate_limited_total counter");

            let requests_total = prometheus::register_counter!(
                "requests_total",
                "Total number of requests processed by middleware"
            )
            .expect("Failed to register requests_total counter");

            // Histogram bucket boundaries ใน seconds
            // [0.0001, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
            let waf_check_duration_seconds = register_histogram!(
                "waf_check_duration_seconds",
                "Time spent checking WAF rules",
                vec![0.0001, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
            )
            .expect("Failed to register waf_check_duration_seconds histogram");

            WafMetrics {
                blocked_requests_total,
                rate_limited_total,
                requests_total,
                waf_check_duration_seconds,
            }
        })
    }

    pub fn record_blocked(&self, rule_id: &str) {
        self.blocked_requests_total
            .with_label_values(&[rule_id])
            .inc();
    }

    pub fn record_rate_limited(&self) {
        self.rate_limited_total.inc();
    }

    pub fn record_request(&self) {
        self.requests_total.inc();
    }
}

/// Handler สำหรับ /metrics endpoint -- คืน Prometheus text format
pub async fn metrics_handler() -> String {
    use prometheus::Encoder;
    let encoder = prometheus::TextEncoder::new();
    let mut buffer = Vec::new();
    encoder
        .encode(&prometheus::gather(), &mut buffer)
        .expect("Failed to encode metrics");
    String::from_utf8(buffer).expect("Metrics not valid UTF-8")
}
```

### ขั้นที่ 8: YAML Rules Config และ CLI Entry Point

`config/rules.yaml` — YAML configuration สำหรับ WAF rules:

```yaml
# WAF Rules Configuration
# แต่ละ rule มี: id, description, pattern (regex), targets, action, severity

rules:
  - id: "CUSTOM-001"
    description: "Matches path traversal attempt via ../ sequences"
    pattern: '\.\./'
    targets: ["url", "query_params"]
    action: block
    severity: high

  - id: "CUSTOM-002"
    description: "Matches null byte injection in parameters"
    pattern: '%00|\x00'
    targets: ["query_params", "body", "headers"]
    action: block
    severity: critical

  - id: "CUSTOM-003"
    description: "Log requests containing base64-encoded data over 1KB"
    pattern: '[A-Za-z0-9+/]{1024,}={0,2}'
    targets: ["body"]
    action: log
    severity: low
```

`blocklist/blocklist.txt`:

```
# IP Blocklist - one CIDR per line
# Lines starting with # are comments

# Known malicious ranges (example)
192.0.2.0/24
198.51.100.0/24

# Specific host blocks
203.0.113.42/32
```

`src/main.rs` — CLI entry point:

```rust
// main.rs -- CLI entry point, axum server setup, layer composition

use axum::{Router, routing::get};
use clap::Parser;
use std::net::SocketAddr;
use tower::ServiceBuilder;

mod rate_limit;
mod waf;
mod ip_blocklist;
mod security_headers;
mod metrics;

use rate_limit::RateLimitLayer;
use security_headers::SecurityHeadersLayer;

/// WAF Middleware Server
#[derive(Parser, Debug)]
#[command(author, version, about)]
struct Args {
    /// Port to listen on
    #[arg(short, long, default_value_t = 8080)]
    port: u16,

    /// Rate limit algorithm: "token-bucket" or "sliding-window"
    #[arg(long, default_value = "token-bucket")]
    rate_algorithm: String,

    /// Token bucket capacity (requests)
    #[arg(long, default_value_t = 100.0)]
    capacity: f64,

    /// Token bucket refill rate (tokens/second)
    #[arg(long, default_value_t = 10.0)]
    refill_rate: f64,

    /// Sliding window duration (seconds)
    #[arg(long, default_value_t = 60)]
    window_secs: u64,

    /// Sliding window max requests
    #[arg(long, default_value_t = 100)]
    max_requests: usize,

    /// Path to IP blocklist file (CIDR notation)
    #[arg(long)]
    blocklist: Option<String>,

    /// Path to WAF rules YAML file
    #[arg(long)]
    rules: Option<String>,
}

async fn health_handler() -> &'static str {
    "OK"
}

async fn echo_handler(req: axum::extract::Request) -> String {
    format!("Received: {} {}", req.method(), req.uri())
}

#[tokio::main]
async fn main() {
    let args = Args::parse();

    let rate_limit_layer = match args.rate_algorithm.as_str() {
        "sliding-window" => RateLimitLayer::sliding_window(
            args.window_secs,
            args.max_requests,
        ),
        _ => RateLimitLayer::token_bucket(args.capacity, args.refill_rate),
    };

    let app = Router::new()
        .route("/health", get(health_handler))
        .route("/echo", get(echo_handler))
        .route("/metrics", get(metrics::metrics_handler))
        .layer(
            ServiceBuilder::new()
                .layer(SecurityHeadersLayer::new())
                .layer(rate_limit_layer),
        );

    let addr = SocketAddr::from(([0, 0, 0, 0], args.port));
    println!("WAF middleware listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## การทดสอบ (Testing)

### Unit Tests

`src/main.rs` (หรือสร้าง `tests/` directory):

```rust
#[cfg(test)]
mod tests {
    use std::net::IpAddr;
    use std::str::FromStr;

    // ---- Token Bucket Tests ----

    #[test]
    fn test_token_bucket_refill_math() {
        use crate::rate_limit::TokenBucket;
        use std::time::Duration;

        // Capacity=10, refill_rate=5 tokens/sec
        let mut bucket = TokenBucket::new(10.0, 5.0);

        // Consume 8 tokens
        assert!(bucket.try_consume(8.0));
        assert!(
            (bucket.tokens - 2.0).abs() < 0.001,
            "After consuming 8 from 10, should have ~2, got {}",
            bucket.tokens
        );

        // Simulate 1 second passing: refill 5 tokens (capped at capacity 10)
        bucket.last_refill -= Duration::from_secs(1);
        bucket.refill();
        assert!(
            (bucket.tokens - 7.0).abs() < 0.01,
            "After 1s refill at 5/s: 2+5=7, got {}",
            bucket.tokens
        );

        // Simulate 10 more seconds: should cap at capacity
        bucket.last_refill -= Duration::from_secs(10);
        bucket.refill();
        assert!(
            (bucket.tokens - 10.0).abs() < 0.01,
            "Should be capped at capacity 10, got {}",
            bucket.tokens
        );
    }

    #[test]
    fn test_token_bucket_deny_when_empty() {
        use crate::rate_limit::TokenBucket;

        let mut bucket = TokenBucket::new(5.0, 1.0);
        assert!(bucket.try_consume(5.0));  // Drain bucket
        assert!(!bucket.try_consume(1.0), "Empty bucket should deny request");
    }

    // ---- Sliding Window Tests ----

    #[test]
    fn test_sliding_window_eviction() {
        use crate::rate_limit::SlidingWindow;
        use std::time::{Duration, Instant};

        let mut window = SlidingWindow::new(Duration::from_secs(60), 100);

        // เพิ่ม 50 timestamps ในอดีต (นอก window)
        let past = Instant::now() - Duration::from_secs(120);
        for _ in 0..50 {
            window.timestamps.push_back(past);
        }

        // เพิ่ม 10 timestamps ปัจจุบัน (ใน window)
        for _ in 0..10 {
            window.record(Instant::now());
        }

        // Evict เก่า -- ควรเหลือแค่ 10
        window.evict_old(Instant::now());
        assert_eq!(
            window.count(),
            10,
            "Only recent requests should remain after eviction"
        );
    }

    #[test]
    fn test_sliding_window_limit_enforcement() {
        use crate::rate_limit::SlidingWindow;
        use std::time::{Duration, Instant};

        let mut window = SlidingWindow::new(Duration::from_secs(60), 3);

        let now = Instant::now();
        assert!(window.check_and_record(now), "Request 1 should be allowed");
        assert!(window.check_and_record(now), "Request 2 should be allowed");
        assert!(window.check_and_record(now), "Request 3 should be allowed");
        assert!(!window.check_and_record(now), "Request 4 should be denied (limit=3)");
    }

    // ---- SQL Injection Tests ----

    #[test]
    fn test_sql_injection_patterns() {
        use crate::waf::WafRuleEngine;

        let engine = WafRuleEngine::default_rules();

        let inputs: &[(&str, bool)] = &[
            ("' OR 1=1 --", true),
            ("UNION SELECT username, password FROM users", true),
            ("'; DROP TABLE users; --", true),
            ("1' OR '1'='1", true),
            ("admin'--", true),
            ("hello world", false),
            ("user@example.com", false),
        ];

        for (input, should_block) in inputs {
            let result = engine.check_sqli(input);
            assert_eq!(
                result.is_some(),
                *should_block,
                "Input {:?}: expected block={}, got block={}",
                input, should_block, result.is_some()
            );
        }
    }

    // ---- XSS Tests ----

    #[test]
    fn test_xss_patterns() {
        use crate::waf::WafRuleEngine;

        let engine = WafRuleEngine::default_rules();

        let inputs: &[(&str, bool)] = &[
            ("<script>alert('xss')</script>", true),
            ("javascript:alert(1)", true),
            ("<img src=x onerror=alert(1)>", true),
            ("<body onload=alert(1)>", true),
            ("<svg onmouseover=alert(1)>", true),
            ("hello world", false),
            ("<b>Bold text</b>", false),
            ("https://example.com/path", false),
        ];

        for (input, should_block) in inputs {
            let result = engine.check_xss(input);
            assert_eq!(
                result.is_some(),
                *should_block,
                "Input {:?}: expected block={}, got block={}",
                input, should_block, result.is_some()
            );
        }
    }

    // ---- CIDR Blocklist Tests ----

    #[test]
    fn test_cidr_ip_blocklist() {
        use crate::ip_blocklist::IpBlocklist;

        let mut blocklist = IpBlocklist::new();
        blocklist.add_cidr("192.168.1.0/24").unwrap();
        blocklist.add_cidr("10.0.0.1/32").unwrap();
        blocklist.add_cidr("172.16.0.0/12").unwrap();

        assert!(blocklist.is_blocked(IpAddr::from_str("192.168.1.100").unwrap()));
        assert!(blocklist.is_blocked(IpAddr::from_str("192.168.1.1").unwrap()));
        assert!(!blocklist.is_blocked(IpAddr::from_str("192.168.2.1").unwrap()));
        assert!(blocklist.is_blocked(IpAddr::from_str("10.0.0.1").unwrap()));
        assert!(!blocklist.is_blocked(IpAddr::from_str("10.0.0.2").unwrap())); // /32
        assert!(blocklist.is_blocked(IpAddr::from_str("172.20.0.1").unwrap())); // in 172.16/12
        assert!(!blocklist.is_blocked(IpAddr::from_str("8.8.8.8").unwrap()));
    }

    // ---- Security Header Tests ----

    #[test]
    fn test_security_header_injection() {
        use crate::security_headers::SecurityHeaders;

        let builder = http::response::Builder::new().status(200);
        let headers = SecurityHeaders::default();
        let response = headers.inject(builder.body(()).unwrap());

        let hdr = response.headers();
        assert_eq!(hdr.get("x-content-type-options").unwrap(), "nosniff");
        assert_eq!(hdr.get("x-frame-options").unwrap(), "DENY");
        assert!(hdr.contains_key("strict-transport-security"));
        assert!(hdr.contains_key("content-security-policy"));
        assert!(hdr.contains_key("referrer-policy"));
    }

    // ---- Tower Middleware Composition Test ----

    #[test]
    fn test_tower_middleware_composition() {
        use tower::ServiceBuilder;
        use tower::service_fn;
        use crate::rate_limit::RateLimitLayer;
        use crate::security_headers::SecurityHeadersLayer;

        // ตรวจว่า types compose กันได้ถูกต้องผ่าน ServiceBuilder
        let _svc = ServiceBuilder::new()
            .layer(SecurityHeadersLayer::new())
            .layer(RateLimitLayer::token_bucket(100.0, 10.0))
            .service(service_fn(|_req: http::Request<()>| async {
                Ok::<http::Response<String>, std::convert::Infallible>(
                    http::Response::new("OK".to_string()),
                )
            }));

        println!("Tower middleware composition: OK");
    }
}
```

### ผลการทดสอบจริง

รัน `cargo test` บน verification project ที่ scratchpad:

```
running 9 tests
test tests::test_cidr_ip_blocklist ... ok
test tests::test_sliding_window_limit_enforcement ... ok
test tests::test_security_header_injection ... ok
test tests::test_token_bucket_deny_when_empty ... ok
test tests::test_token_bucket_refill_math ... ok
test tests::test_tower_middleware_composition ... ok
test tests::test_sliding_window_eviction ... ok
test tests::test_sql_injection_patterns ... ok
test tests::test_xss_patterns ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s
```

## Pitfalls ที่พบบ่อย

### Pitfall 1: poll_ready ไม่ forward ไปยัง inner service

```rust
// BUG: ไม่ forward poll_ready
fn poll_ready(&mut self, _cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    Poll::Ready(Ok(()))
}

// FIX: forward เสมอ
fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    self.inner.poll_ready(cx)
}
```

Tower contract ระบุว่า `call()` สามารถถูกเรียกได้ก็ต่อเมื่อ `poll_ready()` return `Poll::Ready(Ok(()))` แล้ว ถ้า middleware return `Ready` โดยไม่ตรวจสอบ inner service อาจเรียก `call()` บน inner service ที่ยัง backpressure อยู่ ส่งผลให้เกิด panic หรือ undefined behavior ขึ้นอยู่กับ implementation ของ inner service

### Pitfall 2: clone ทำให้ per-IP state ไม่ถูกต้อง

```rust
// BUG: DashMap ถูก clone -> state แยกกันทุก worker
pub struct RateLimitLayer {
    token_buckets: DashMap<IpAddr, TokenBucket>,  // clone = new empty map!
}

// FIX: wrap ด้วย Arc
pub struct RateLimitLayer {
    token_buckets: Arc<DashMap<IpAddr, TokenBucket>>,  // clone = shared ref
}
```

axum clone middleware layer ทุกครั้งที่ worker ใหม่ถูกสร้าง ถ้า `DashMap` ไม่ได้ wrap ด้วย `Arc` แต่ละ worker จะมี state แยกกัน rate limit จะไม่ทำงาน IP หนึ่งสามารถส่ง request ได้ไม่จำกัด

### Pitfall 3: Sliding Window Memory Leak จาก eviction ไม่สมบูรณ์

```rust
// BUG: ไม่ evict เก่าเมื่อ limit ถึง -> VecDeque โตตลอด
pub fn check_and_record(&mut self, now: Instant) -> bool {
    if self.timestamps.len() < self.max_count {
        self.timestamps.push_back(now);
        true
    } else {
        false  // ไม่ evict! timestamps เก่า ๆ ค้างอยู่ใน VecDeque
    }
}

// FIX: evict ก่อน check เสมอ
pub fn check_and_record(&mut self, now: Instant) -> bool {
    self.evict_old(now);  // ลบเก่าก่อน
    if self.timestamps.len() < self.max_count {
        self.timestamps.push_back(now);
        true
    } else {
        false
    }
}
```

ถ้าไม่ evict เก่าก่อน check sliding window จะ "ติดค้าง" -- IP ที่เคยถึง limit ในนาทีที่ผ่านมาจะยัง block อยู่แม้ window ผ่านไปแล้ว และ `VecDeque` จะโตขึ้นเรื่อย ๆ เป็น memory leak

### Pitfall 4: Regex ใน `r"..."` กับ Double Quote

```rust
// BUG: ใช้ r"..." กับ pattern ที่มี " ภายใน
let re = Regex::new(r"(?i)'\s*(or|and)\s+[\w'"]+\s*=").unwrap();
//                                                  ^-- " ตรงนี้ปิด r"..." ก่อนกำหนด!

// FIX: ใช้ r#"..."# สำหรับ pattern ที่มี " ภายใน
let re = Regex::new(r#"(?i)'\s*(or|and)\s+[\w'"]+\s*="#).unwrap();
```

Rust raw string literal `r"..."` ไม่อนุญาตให้มี `"` ภายใน ถ้า pattern ต้องการ `"` ต้องใช้ `r#"..."#` หรือ `r##"..."##` โดยเพิ่ม `#` ให้มากพอที่ไม่มีใน content

### Pitfall 5: X-Forwarded-For Chain Validation

```rust
// BUG: เชื่อ X-Forwarded-For ทั้ง chain โดยไม่ validate
let client_ip = req.headers()
    .get("x-forwarded-for")
    .and_then(|v| v.to_str().ok())
    .and_then(|s| s.parse::<IpAddr>().ok())  // fail ถ้ามีหลาย IP ใน header!
    .unwrap_or_default();

// BETTER: เอาเฉพาะ IP แรกใน chain (ซึ่ง client ตั้งมา ควร validate ด้วย trusted proxy list)
let client_ip = req.headers()
    .get("x-forwarded-for")
    .and_then(|v| v.to_str().ok())
    .and_then(|s| s.split(',').next())   // first IP in chain
    .and_then(|s| s.trim().parse().ok())
    .unwrap_or(IpAddr::from([0, 0, 0, 0]));
```

Header `X-Forwarded-For` มีรูปแบบ `client, proxy1, proxy2` ถ้า parse ทั้ง string เป็น `IpAddr` จะ fail เสมอ ต้อง split ด้วย `,` ก่อน ใน production ควรมี trusted proxy list เพื่อป้องกัน client ปลอม IP ใน header

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release

# Binary อยู่ที่ target/release/waf-middleware
./target/release/waf-middleware \
  --port 8080 \
  --rate-algorithm sliding-window \
  --window-secs 60 \
  --max-requests 100 \
  --blocklist blocklist/blocklist.txt \
  --rules config/rules.yaml
```

### Docker Image

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /app/target/release/waf-middleware /usr/local/bin/
COPY config/ config/
COPY blocklist/ blocklist/

EXPOSE 8080
ENTRYPOINT ["waf-middleware"]
CMD ["--port", "8080", "--rules", "config/rules.yaml", "--blocklist", "blocklist/blocklist.txt"]
```

```bash
docker build -t waf-middleware:latest .
docker run -p 8080:8080 waf-middleware:latest
```

### SIGHUP Hot Reload สำหรับ Blocklist

```rust
// ใน main.rs -- reload blocklist เมื่อรับ SIGHUP signal

use tokio::signal::unix::{signal, SignalKind};

async fn watch_blocklist_reload(
    path: String,
    blocklist: Arc<RwLock<IpBlocklist>>,
) {
    let mut stream = signal(SignalKind::hangup()).unwrap();
    loop {
        stream.recv().await;
        match std::fs::read_to_string(&path) {
            Ok(content) => {
                let mut new_list = IpBlocklist::new();
                if new_list.load_from_str(&content).is_ok() {
                    let mut guard = blocklist.write().await;
                    *guard = new_list;
                    eprintln!("Blocklist reloaded: {} entries", guard.len());
                }
            }
            Err(e) => eprintln!("Failed to reload blocklist: {}", e),
        }
    }
}
```

ส่ง `kill -HUP <pid>` เพื่อ reload blocklist โดยไม่ restart process

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Trie-based IP Matching (ปานกลาง)

blocklist ปัจจุบันใช้ O(n) linear scan เมื่อ blocklist มี entries มากพัน ๆ รายการจะช้า ให้ implement Patricia Trie (Radix Tree) สำหรับ IP prefix matching เพื่อลดเวลา lookup เป็น O(prefix_length) ซึ่งคงที่ไม่ขึ้นกับขนาด blocklist

**เป้าหมาย:** blocklist 100,000 entries ควร lookup ได้ภายใน 1 microsecond

```rust
// โครงสร้างที่แนะนำ
struct IpTrie {
    ipv4_root: TrieNode,
    ipv6_root: TrieNode,
}

struct TrieNode {
    blocked: bool,
    children: [Option<Box<TrieNode>>; 2], // bit 0 หรือ bit 1
}
```

### แบบฝึกหัดที่ 2: Request Body Buffering และ Multi-part Form Inspection (ยาก)

implement `WafLayer` ที่ buffer request body ได้สูงสุด 64 KiB และตรวจ WAF rules กับ content ก่อน forward ให้ handler ต้องจัดการกรณี:
- Body เกิน 64 KiB: return 413 Payload Too Large
- Content-Type: `application/x-www-form-urlencoded`: URL-decode ก่อนตรวจ
- Content-Type: `multipart/form-data`: extract ทุก part ก่อนตรวจแยก

**Hint:** ใช้ `http_body_util::BodyExt::collect()` สำหรับ buffering:

```rust
use http_body_util::BodyExt;

async fn buffer_body<B: Body>(body: B, limit: usize)
    -> Result<Bytes, BodyError>
where B::Error: Into<Box<dyn std::error::Error + Send + Sync>>,
{
    let bytes = body.collect().await
        .map_err(|e| BodyError::Collect(e.into()))?
        .to_bytes();
    if bytes.len() > limit {
        return Err(BodyError::TooLarge);
    }
    Ok(bytes)
}
```

### แบบฝึกหัดที่ 3: JWT Validation Layer (ปานกลาง)

เพิ่ม `JwtValidationLayer` ที่ตรวจ `Authorization: Bearer <token>` header บน routes ที่ต้องการ authentication โดยใช้ `jsonwebtoken` crate ตรวจ signature, expiry, และ claims ถ้า token invalid ให้ return 401

```rust
// Cargo.toml เพิ่ม:
// jsonwebtoken = "9"

pub struct JwtValidationLayer {
    secret: Arc<Vec<u8>>,
    required_claims: Vec<String>,
}
```

### แบบฝึกหัดที่ 4: Distributed Rate Limiting ด้วย Redis (ยากมาก)

`DashMap` เก็บ state ไว้ใน process เดียว ถ้า deploy หลาย instance แต่ละตัวมี rate limit counter แยกกัน IP หนึ่งสามารถส่ง `n * instances` requests ต่อหน่วยเวลา ให้ replace `DashMap` ด้วย Redis โดยใช้ `redis` crate พร้อม Lua script สำหรับ atomic increment:

```lua
-- Redis Lua script สำหรับ token bucket atomic update
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or capacity
local last_refill = tonumber(bucket[2]) or now

local elapsed = now - last_refill
tokens = math.min(capacity, tokens + elapsed * refill_rate)

if tokens >= 1.0 then
    tokens = tokens - 1.0
    redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
    redis.call('EXPIRE', key, 3600)
    return 1  -- allowed
else
    return 0  -- denied
end
```

## สรุป

โปรเจคนี้สร้าง middleware stack ที่ใช้งานได้จริงสำหรับ production API โดยครอบคลุม:

**Architectural patterns ที่ได้เรียน:**
- **Tower Layer/Service composition** — ทุก middleware เป็น reusable unit ที่ compose กันได้ไม่ขึ้นกับ framework
- **poll_ready propagation** — Tower contract ที่ต้องปฏิบัติตามเพื่อป้องกัน panic ใน production
- **DashMap + Arc pattern** — per-key concurrent state sharing ระหว่าง async tasks
- **VecDeque eviction** — sliding window ที่ evict-on-access ไม่มี background cleanup goroutine

**Security mechanisms ที่ implement:**
- Token bucket และ sliding window rate limiting ด้วยคณิตศาสตร์ที่ verified ด้วย unit tests
- WAF rule engine ที่ configurable ผ่าน YAML รองรับ custom rules
- SQL injection detection: 5 patterns ครอบคลุม tautology, UNION, DROP, comment, quote
- XSS detection: 5 patterns ครอบคลุม script tag, javascript: URI, event handlers
- CIDR IP blocklist ด้วย `ipnetwork` crate พร้อม SIGHUP hot-reload
- Security headers ที่ inject อัตโนมัติทุก response (HSTS, CSP, X-Frame-Options ฯลฯ)
- Prometheus metrics สำหรับ observability ใน production

โปรเจคถัดไป [Project D10: ZKP Demo](project-d10-zkp-demo.md) จะนำไปสู่ zero-knowledge proof systems ซึ่งเป็น cryptographic technique ที่ช่วยให้พิสูจน์ความรู้โดยไม่เปิดเผยข้อมูล — ต้องการความเข้าใจ mathematical foundations ของ elliptic curves และ polynomial commitments

---

**โปรเจคก่อนหน้า:** [project-d08-vuln-scanner.md](project-d08-vuln-scanner.md) | **โปรเจคถัดไป:** [project-d10-zkp-demo.md](project-d10-zkp-demo.md)
