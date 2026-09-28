# Project G06: Health Check Service

> โมดูล: G — DevOps & Infrastructure | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Health Check Service** — ระบบตรวจสอบ uptime และ availability ของ services ต่าง ๆ แบบ production-grade ที่ทำงานได้จริงในสภาพแวดล้อม DevOps สมัยใหม่

Health checking เป็นส่วนสำคัญของ infrastructure ทุกขนาด — ตั้งแต่ startup ที่รัน service เพียงไม่กี่ตัว ไปจนถึงองค์กรขนาดใหญ่ที่มี microservices นับร้อยตัวทำงานพร้อมกัน โดยที่ทีม DevOps/SRE ต้องรู้ทันทีเมื่อ service ใดมีปัญหา ระบบที่เราจะสร้างครอบคลุมการตรวจสอบหลายรูปแบบ: HTTP endpoint, TCP port, DNS resolution, disk space, และ process status — รวมถึงระบบติดตาม SLO (Service Level Objective) ที่ช่วยตัดสินใจว่าเราผิด SLA กับลูกค้าหรือไม่

**Use cases จริงในโลก production:**
- **Kubernetes readiness/liveness probe** — `/health` endpoint ที่ k8s ใช้ตัดสินใจว่า pod พร้อมรับ traffic หรือต้องการ restart
- **SRE on-call automation** — ส่ง alert เมื่อมี consecutive failures ถึง threshold ก่อนที่ลูกค้าจะร้องเรียน
- **SLO monitoring** — วัด error budget ที่เหลืออยู่เพื่อตัดสินใจว่าปลอดภัยจะ deploy ใหม่หรือไม่
- **Status page** — แสดงสถานะ services ให้ทีมและผู้ใช้เห็นแบบ real-time เช่น `status.github.com`
- **Capacity planning** — เก็บ history latency เพื่อวิเคราะห์ degradation trend ก่อนจะกลายเป็นปัญหา

**ทำไมโปรเจคนี้ถึงน่าสร้างด้วย Rust?**

1. Health check ทำงาน concurrent หลายร้อยตัวพร้อมกัน — Rust + tokio ทำได้โดยไม่ cost CPU มาก
2. Ring buffer สำหรับ history ต้องการ memory efficiency — `VecDeque` ใน Rust ทำได้แบบ zero-cost
3. Atomic counters สำหรับ Prometheus metrics ไม่ต้องใช้ Mutex — ประสิทธิภาพสูงใน multi-threaded context
4. `async_trait` ช่วยให้ trait-based check interface สะอาดและ ergonomic
5. Type system ของ Rust ทำให้ severity ordering ของ `CheckStatus` รับประกันได้ที่ compile time

## สิ่งที่จะได้เรียนรู้

- **`async_trait` crate** — สร้าง trait ที่มี `async fn` ได้ก่อน Rust stable รองรับ native async fn in trait
- **Ring buffer pattern** ด้วย `VecDeque` — เก็บ history แบบ circular โดย evict รายการเก่าอัตโนมัติ
- **SLO/SLA mathematics** — คำนวณ uptime percentage, error budget, และ breach detection จาก first principles
- **`tokio::time::timeout`** — wrap async operation ด้วย timeout ป้องกัน hang
- **`tokio::task::spawn_blocking`** — รัน blocking I/O (เช่น disk check) บน dedicated thread pool ไม่บล็อก async runtime
- **Concurrent execution** ด้วย `tokio::spawn` + `JoinHandle` — รัน checks หลายตัวพร้อมกัน
- **Atomic counters** ด้วย `AtomicU64` สำหรับ Prometheus counters ที่ thread-safe
- **Prometheus text format** — histogram format พร้อม `_bucket`, `_sum`, `_count`, `+Inf` bucket
- **Trait objects** `Box<dyn HealthCheck>` — polymorphic check system ที่เพิ่ม check type ใหม่ได้โดยไม่แก้ core logic

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–30**: Rust basics — ownership, borrowing, structs, enums, traits, error handling
- **Part 31–40**: Trait objects, `Box<dyn Trait>`, `dyn` dispatch, `HashMap`, `VecDeque`
- **Part 41–50**: Concurrency — `Arc`, `Mutex`, `thread::spawn`
- **Part 51–60**: `std::sync::atomic` — `AtomicU64`, `Ordering`
- **Part 61–70**: tokio async runtime — `async/await`, `tokio::spawn`, `tokio::time`
- **Part 71–80**: External crates — `serde`, `serde_json`, `async-trait`
- **Part 96–100**: Production patterns — concurrent task management, graceful shutdown

## โครงสร้างโปรเจค (Project Layout)

```
health-checker/
├── Cargo.toml
└── src/
    ├── lib.rs            ← re-exports public API
    ├── main.rs           ← entry point + demo
    ├── types.rs          ← CheckStatus, CheckResult, CheckHistory, SloTracker, LatencyHistogram
    ├── aggregator.rs     ← ServiceHealth, StatusPage
    ├── scheduler.rs      ← HealthScheduler, ManagedCheck, CheckConfig
    ├── metrics.rs        ← CheckMetrics (Prometheus counters + histogram)
    └── checks/
        ├── mod.rs        ← HealthCheck trait (async_trait)
        ├── http.rs       ← HttpCheck (reqwest GET, expect status code)
        ├── tcp.rs        ← TcpCheck (tokio::net::TcpStream::connect)
        ├── dns.rs        ← DnsCheck (tokio::net::lookup_host)
        ├── disk.rs       ← DiskSpaceCheck (df command via spawn_blocking)
        ├── process.rs    ← ProcessCheck (scan /proc filesystem)
        └── mock.rs       ← MockCheck สำหรับ testing
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      HealthScheduler                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ ManagedCheck │  │ ManagedCheck │  │ ManagedCheck │  ...      │
│  │ HttpCheck    │  │ TcpCheck     │  │ DiskCheck    │          │
│  │ config       │  │ config       │  │ config       │          │
│  │ history ─┐  │  │ history ─┐  │  │ history ─┐  │          │
│  └──────────┼──┘  └──────────┼──┘  └──────────┼──┘          │
│             │                │                │                  │
│   tokio::spawn (concurrent)  │                │                  │
│             └──────┬─────────┘                │                  │
└────────────────────┼──────────────────────────┼──────────────────┘
                     │  Vec<(name, CheckResult)> │
                     ▼                           │
          ┌─────────────────────┐               │ CheckHistory
          │   ServiceHealth     │         (ring buffer, uptime%)
          │  worst_of(statuses) │               │
          └──────────┬──────────┘               ▼
                     │                  ┌──────────────────┐
          ┌──────────▼──────────┐       │   SloTracker     │
          │     StatusPage      │       │  is_breached()?  │
          │  overall worst_of   │       │  budget_remaining│
          │  to_json() / HTTP   │       └──────────────────┘
          └──────────┬──────────┘
                     │
          ┌──────────▼──────────┐
          │   CheckMetrics      │
          │  AtomicU64 counters │
          │  LatencyHistogram   │
          │  /metrics endpoint  │
          └─────────────────────┘
```

### ลำดับความรุนแรงของ CheckStatus

หัวใจสำคัญของระบบคือ `CheckStatus::worst_of()` ที่ aggregate หลาย checks เป็น overall status เดียว:

```
Healthy (0) < Degraded (1) < Timeout (2) < Unhealthy (3) < Unknown (4)
```

ลำดับนี้สะท้อน operational reality:
- **Healthy** — ทุกอย่างปกติ
- **Degraded** — ทำงานได้แต่มีปัญหา performance (เช่น latency สูง, disk เหลือน้อย)
- **Timeout** — ไม่รู้สถานะ แต่ check เองก็ไม่ตอบ — อาจแย่กว่า degraded
- **Unhealthy** — ยืนยันว่ามีปัญหาแน่ ๆ (เช่น HTTP 5xx, TCP refused)
- **Unknown** — check ไม่สามารถรันได้เลย (misconfiguration, etc.)

### SLO/Error Budget

SLO (Service Level Objective) ที่พบบ่อยในอุตสาหกรรมคือ **99.9% uptime** ซึ่งแปลว่า:

```
Downtime budget = (1 - 0.999) × window_minutes
               = 0.001 × 43,200 (30-day month)
               = 43.2 minutes per month
```

เมื่อ error budget หมด ทีมต้องหยุด deploy feature ใหม่และ focus ที่การเพิ่ม reliability แทน

| SLO Target | Monthly Budget | Weekly Budget | Daily Budget |
|-----------|---------------|--------------|-------------|
| 99.0%     | 432 นาที      | 100.8 นาที  | 14.4 นาที  |
| 99.5%     | 216 นาที      | 50.4 นาที   | 7.2 นาที   |
| 99.9%     | 43.2 นาที     | 10.1 นาที   | 1.44 นาที  |
| 99.99%    | 4.32 นาที     | 1.01 นาที   | 8.64 วินาที |

### Ring Buffer Strategy

`CheckHistory` ใช้ `VecDeque` เป็น ring buffer เพื่อเก็บ history ล่าสุด N รายการโดยไม่ allocate memory ใหม่:

```
max_size = 5
Push: [A] → [A, B] → [A, B, C] → [A, B, C, D] → [A, B, C, D, E]  (full)
Push: F → evict A → [B, C, D, E, F]
Push: G → evict B → [C, D, E, F, G]
```

`VecDeque::pop_front()` เป็น O(1) ไม่ต้อง shift elements เหมือน `Vec::remove(0)`

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: CheckStatus และ CheckResult

เริ่มจาก core types ที่ทุกส่วนของระบบใช้ร่วมกัน

สร้าง `Cargo.toml`:

```toml
[package]
name = "health-checker"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
async-trait = "0.1"
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls", "json"] }
```

สร้าง `src/types.rs` (ส่วนที่ 1 — CheckStatus + CheckResult):

```rust
use std::collections::{HashMap, VecDeque};
use serde::{Deserialize, Serialize};

/// สถานะของ health check แต่ละครั้ง
/// severity order: Healthy < Degraded < Timeout < Unhealthy < Unknown
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum CheckStatus {
    Healthy,
    Degraded,
    Timeout,
    Unhealthy,
    Unknown,
}

impl CheckStatus {
    /// ระดับความรุนแรง: ยิ่งสูงยิ่งแย่
    pub fn severity(&self) -> u8 {
        match self {
            CheckStatus::Healthy   => 0,
            CheckStatus::Degraded  => 1,
            CheckStatus::Timeout   => 2,
            CheckStatus::Unhealthy => 3,
            CheckStatus::Unknown   => 4,
        }
    }

    /// คืน status ที่แย่ที่สุดจาก slice ที่ให้มา
    pub fn worst_of(statuses: &[CheckStatus]) -> CheckStatus {
        if statuses.is_empty() {
            return CheckStatus::Unknown;
        }
        statuses
            .iter()
            .max_by_key(|s| s.severity())
            .cloned()
            .unwrap_or(CheckStatus::Unknown)
    }

    pub fn is_healthy(&self) -> bool {
        matches!(self, CheckStatus::Healthy)
    }

    pub fn as_str(&self) -> &'static str {
        match self {
            CheckStatus::Healthy   => "healthy",
            CheckStatus::Degraded  => "degraded",
            CheckStatus::Timeout   => "timeout",
            CheckStatus::Unhealthy => "unhealthy",
            CheckStatus::Unknown   => "unknown",
        }
    }
}

impl std::fmt::Display for CheckStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.as_str())
    }
}

/// ผลลัพธ์จากการ check ครั้งหนึ่ง
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CheckResult {
    pub status: CheckStatus,
    pub latency_ms: u64,
    pub message: String,
    pub metadata: HashMap<String, String>,
}

impl CheckResult {
    pub fn healthy(message: impl Into<String>, latency_ms: u64) -> Self {
        CheckResult {
            status: CheckStatus::Healthy,
            latency_ms,
            message: message.into(),
            metadata: HashMap::new(),
        }
    }

    pub fn degraded(message: impl Into<String>, latency_ms: u64) -> Self {
        CheckResult {
            status: CheckStatus::Degraded,
            latency_ms,
            message: message.into(),
            metadata: HashMap::new(),
        }
    }

    pub fn unhealthy(message: impl Into<String>, latency_ms: u64) -> Self {
        CheckResult {
            status: CheckStatus::Unhealthy,
            latency_ms,
            message: message.into(),
            metadata: HashMap::new(),
        }
    }

    pub fn timeout_result(latency_ms: u64) -> Self {
        CheckResult {
            status: CheckStatus::Timeout,
            latency_ms,
            message: "Check timed out".to_string(),
            metadata: HashMap::new(),
        }
    }

    /// Builder method เพิ่ม metadata key-value
    pub fn with_metadata(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.metadata.insert(key.into(), value.into());
        self
    }
}
```

**ทดสอบเบื้องต้น:**

```rust
fn main() {
    let statuses = vec![
        CheckStatus::Healthy,
        CheckStatus::Degraded,
        CheckStatus::Unhealthy,
    ];
    println!("Worst: {}", CheckStatus::worst_of(&statuses));
    // Output: unhealthy

    let result = CheckResult::healthy("HTTP 200 OK", 45)
        .with_metadata("status_code", "200");
    println!("{:?}", result.status);
    // Output: Healthy
}
```

### ขั้นที่ 2: CheckHistory — Ring Buffer

Ring buffer สำหรับเก็บประวัติ checks ล่าสุด N รายการ พร้อม uptime% และ consecutive failure tracking:

```rust
// ต่อจาก src/types.rs

/// Ring buffer สำหรับเก็บประวัติผล check ล่าสุด N ครั้ง
#[derive(Debug, Clone)]
pub struct CheckHistory {
    results: VecDeque<CheckResult>,
    max_size: usize,
}

impl CheckHistory {
    pub fn new(max_size: usize) -> Self {
        assert!(max_size > 0, "history max_size must be > 0");
        CheckHistory {
            results: VecDeque::with_capacity(max_size),
            max_size,
        }
    }

    /// เพิ่มผลลัพธ์ใหม่
    /// ถ้า buffer เต็มจะ evict รายการเก่าที่สุดออกก่อน (FIFO)
    pub fn push(&mut self, result: CheckResult) {
        if self.results.len() >= self.max_size {
            self.results.pop_front(); // O(1) — ไม่ต้อง shift elements
        }
        self.results.push_back(result);
    }

    pub fn len(&self) -> usize { self.results.len() }
    pub fn is_empty(&self) -> bool { self.results.is_empty() }
    pub fn capacity(&self) -> usize { self.max_size }

    /// คำนวณ uptime percentage จากประวัติที่มี
    /// นิยาม: uptime% = (จำนวน Healthy / ทั้งหมด) × 100
    pub fn uptime_percent(&self) -> f64 {
        if self.results.is_empty() {
            return 100.0; // ไม่มีข้อมูล → assume healthy
        }
        let healthy = self.results.iter()
            .filter(|r| r.status.is_healthy())
            .count();
        (healthy as f64 / self.results.len() as f64) * 100.0
    }

    /// นับจำนวน consecutive failures จากปลายบัฟเฟอร์ (ล่าสุดก่อน)
    /// ใช้ตรวจว่าควร alert หรือไม่
    pub fn consecutive_failures(&self) -> usize {
        let mut count = 0;
        for result in self.results.iter().rev() {
            if result.status.is_healthy() {
                break; // พบ Healthy → reset นับ
            }
            count += 1;
        }
        count
    }

    pub fn latest(&self) -> Option<&CheckResult> {
        self.results.back()
    }

    pub fn iter(&self) -> impl Iterator<Item = &CheckResult> {
        self.results.iter()
    }
}
```

**ทดสอบ ring buffer behavior:**

```rust
let mut h = CheckHistory::new(3);
h.push(CheckResult::unhealthy("fail", 100)); // [X]
h.push(CheckResult::healthy("ok", 10));      // [X, ✓]
h.push(CheckResult::healthy("ok", 10));      // [X, ✓, ✓]  ← full
h.push(CheckResult::healthy("new", 5));      // [✓, ✓, ✓]  ← X evicted!

assert_eq!(h.len(), 3);
assert!(h.iter().all(|r| r.status.is_healthy())); // X ถูกลบไปแล้ว
println!("Uptime: {}%", h.uptime_percent());       // 100%
```

### ขั้นที่ 3: HealthCheck Trait และ Check Implementations

#### HealthCheck Trait

สร้าง `src/checks/mod.rs`:

```rust
use async_trait::async_trait;
use crate::types::CheckResult;

/// Trait หลักสำหรับ health check ทุกประเภท
/// async_trait macro จัดการ boxing ของ future ให้อัตโนมัติ
#[async_trait]
pub trait HealthCheck: Send + Sync {
    async fn check(&self) -> CheckResult;
    fn name(&self) -> &str;
}

pub mod dns;
pub mod disk;
pub mod http;
pub mod mock;
pub mod process;
pub mod tcp;
```

**ทำไมต้องใช้ `async_trait`?**

Rust stable (ก่อน 1.75) ไม่รองรับ `async fn` ใน trait โดยตรง เพราะ return type ของ async fn คือ `impl Future<Output = T>` ซึ่งมีขนาดไม่แน่นอน (unsized) `async_trait` แก้ปัญหาด้วยการ transform เป็น:

```rust
// สิ่งที่ async_trait generate ให้เบื้องหลัง:
fn check(&self) -> Pin<Box<dyn Future<Output = CheckResult> + Send + '_>>
```

ตั้งแต่ Rust 1.75 เป็นต้นไป `async fn in trait` เป็น stable แล้ว แต่ `async_trait` crate ยังเป็นทางเลือกที่นิยมเพราะ ergonomic กว่า

#### HttpCheck — ตรวจสอบ HTTP Endpoint

สร้าง `src/checks/http.rs`:

```rust
use async_trait::async_trait;
use crate::types::{CheckResult, CheckStatus};
use super::HealthCheck;
use std::collections::HashMap;
use std::time::{Duration, Instant};

pub struct HttpCheck {
    pub name: String,
    pub url: String,
    pub expected_status: u16,
    pub timeout_secs: u64,
}

impl HttpCheck {
    pub fn new(name: impl Into<String>, url: impl Into<String>) -> Self {
        HttpCheck {
            name: name.into(),
            url: url.into(),
            expected_status: 200,
            timeout_secs: 10,
        }
    }

    pub fn with_expected_status(mut self, status: u16) -> Self {
        self.expected_status = status;
        self
    }

    pub fn with_timeout(mut self, secs: u64) -> Self {
        self.timeout_secs = secs;
        self
    }
}

#[async_trait]
impl HealthCheck for HttpCheck {
    async fn check(&self) -> CheckResult {
        let start = Instant::now();
        let timeout = Duration::from_secs(self.timeout_secs);

        let client = match reqwest::Client::builder()
            .timeout(timeout)
            .build()
        {
            Ok(c) => c,
            Err(e) => {
                return CheckResult {
                    status: CheckStatus::Unhealthy,
                    latency_ms: start.elapsed().as_millis() as u64,
                    message: format!("Failed to build HTTP client: {}", e),
                    metadata: HashMap::new(),
                };
            }
        };

        match client.get(&self.url).send().await {
            Ok(resp) => {
                let latency_ms = start.elapsed().as_millis() as u64;
                let status_code = resp.status().as_u16();
                let mut meta = HashMap::new();
                meta.insert("status_code".to_string(), status_code.to_string());
                meta.insert("url".to_string(), self.url.clone());

                if status_code == self.expected_status {
                    CheckResult {
                        status: CheckStatus::Healthy,
                        latency_ms,
                        message: format!("HTTP {} OK", status_code),
                        metadata: meta,
                    }
                } else if status_code >= 500 {
                    CheckResult {
                        status: CheckStatus::Unhealthy,
                        latency_ms,
                        message: format!("HTTP server error: {}", status_code),
                        metadata: meta,
                    }
                } else {
                    CheckResult {
                        status: CheckStatus::Degraded,
                        latency_ms,
                        message: format!("Unexpected HTTP status: {}", status_code),
                        metadata: meta,
                    }
                }
            }
            Err(e) => {
                let latency_ms = start.elapsed().as_millis() as u64;
                if e.is_timeout() {
                    CheckResult::timeout_result(latency_ms)
                } else {
                    CheckResult::unhealthy(
                        format!("HTTP request failed: {}", e),
                        latency_ms,
                    )
                }
            }
        }
    }

    fn name(&self) -> &str { &self.name }
}
```

#### TcpCheck — ตรวจสอบ TCP Connection

สร้าง `src/checks/tcp.rs`:

```rust
use async_trait::async_trait;
use crate::types::{CheckResult, CheckStatus};
use super::HealthCheck;
use std::collections::HashMap;
use std::time::{Duration, Instant};
use tokio::net::TcpStream;

pub struct TcpCheck {
    pub name: String,
    pub host: String,
    pub port: u16,
    pub timeout_secs: u64,
}

impl TcpCheck {
    pub fn new(name: impl Into<String>, host: impl Into<String>, port: u16) -> Self {
        TcpCheck { name: name.into(), host: host.into(), port, timeout_secs: 5 }
    }
}

#[async_trait]
impl HealthCheck for TcpCheck {
    async fn check(&self) -> CheckResult {
        let start = Instant::now();
        let addr = format!("{}:{}", self.host, self.port);
        let timeout = Duration::from_secs(self.timeout_secs);

        match tokio::time::timeout(timeout, TcpStream::connect(&addr)).await {
            Ok(Ok(_stream)) => {
                let latency_ms = start.elapsed().as_millis() as u64;
                let mut meta = HashMap::new();
                meta.insert("host".to_string(), self.host.clone());
                meta.insert("port".to_string(), self.port.to_string());
                CheckResult {
                    status: CheckStatus::Healthy,
                    latency_ms,
                    message: format!("TCP {}:{} connected", self.host, self.port),
                    metadata: meta,
                }
            }
            Ok(Err(e)) => CheckResult::unhealthy(
                format!("TCP {}:{} refused: {}", self.host, self.port, e),
                start.elapsed().as_millis() as u64,
            ),
            Err(_) => CheckResult::timeout_result(start.elapsed().as_millis() as u64),
        }
    }

    fn name(&self) -> &str { &self.name }
}
```

#### DnsCheck — ตรวจสอบ DNS Resolution

สร้าง `src/checks/dns.rs`:

```rust
use async_trait::async_trait;
use crate::types::{CheckResult, CheckStatus};
use super::HealthCheck;
use std::collections::HashMap;
use std::time::{Duration, Instant};

pub struct DnsCheck {
    pub name: String,
    pub hostname: String,
    pub timeout_secs: u64,
}

impl DnsCheck {
    pub fn new(name: impl Into<String>, hostname: impl Into<String>) -> Self {
        DnsCheck { name: name.into(), hostname: hostname.into(), timeout_secs: 5 }
    }
}

#[async_trait]
impl HealthCheck for DnsCheck {
    async fn check(&self) -> CheckResult {
        let start = Instant::now();
        // lookup_host ต้องการ "hostname:port" format
        let host_port: String = format!("{}:80", self.hostname);
        let timeout = Duration::from_secs(self.timeout_secs);

        match tokio::time::timeout(
            timeout,
            tokio::net::lookup_host(host_port), // ส่ง String (owned) แก้ lifetime issue
        ).await {
            Ok(Ok(addrs)) => {
                let latency_ms = start.elapsed().as_millis() as u64;
                let addr_list: Vec<String> = addrs.map(|a| a.to_string()).collect();
                if addr_list.is_empty() {
                    return CheckResult::unhealthy(
                        format!("DNS returned no addresses for {}", self.hostname),
                        latency_ms,
                    );
                }
                let mut meta = HashMap::new();
                meta.insert("addresses".to_string(), addr_list.join(", "));
                meta.insert("count".to_string(), addr_list.len().to_string());
                CheckResult {
                    status: CheckStatus::Healthy,
                    latency_ms,
                    message: format!(
                        "DNS resolved {} → {} address(es)",
                        self.hostname, addr_list.len()
                    ),
                    metadata: meta,
                }
            }
            Ok(Err(e)) => CheckResult::unhealthy(
                format!("DNS failed for {}: {}", self.hostname, e),
                start.elapsed().as_millis() as u64,
            ),
            Err(_) => CheckResult::timeout_result(start.elapsed().as_millis() as u64),
        }
    }

    fn name(&self) -> &str { &self.name }
}
```

**Pitfall: `&host_port` causes lifetime error กับ `async_trait`**

```rust
// ❌ Error: host_port does not live long enough
match tokio::time::timeout(timeout, tokio::net::lookup_host(&host_port)).await {

// ✅ แก้: ส่ง String (owned) แทน &str
let host_port: String = format!("{}:80", self.hostname);
match tokio::time::timeout(timeout, tokio::net::lookup_host(host_port)).await {
```

สาเหตุ: `async_trait` macro บังคับให้ future เป็น `'static` (Send + Sync) แต่ `&host_port` borrow local variable ที่อาจ drop ก่อน future resolve เราแก้โดยส่ง owned `String` ให้ `lookup_host` เลย

#### DiskSpaceCheck — ตรวจสอบพื้นที่ Disk

สร้าง `src/checks/disk.rs`:

```rust
use async_trait::async_trait;
use crate::types::{CheckResult, CheckStatus};
use super::HealthCheck;
use std::collections::HashMap;
use std::time::Instant;

pub struct DiskSpaceCheck {
    pub name: String,
    pub path: String,
    pub min_free_bytes: u64,
}

impl DiskSpaceCheck {
    pub fn new(name: impl Into<String>, path: impl Into<String>, min_free_bytes: u64) -> Self {
        DiskSpaceCheck { name: name.into(), path: path.into(), min_free_bytes }
    }

    pub fn with_min_free_gb(
        name: impl Into<String>,
        path: impl Into<String>,
        min_free_gb: u64,
    ) -> Self {
        DiskSpaceCheck::new(name, path, min_free_gb * 1_073_741_824)
    }
}

/// อ่าน free bytes ผ่าน `df -B1 <path>` (GNU coreutils)
fn get_free_bytes(path: &str) -> Result<u64, String> {
    let output = std::process::Command::new("df")
        .args(["-B1", path])
        .output()
        .map_err(|e| format!("Cannot run df: {}", e))?;

    if !output.status.success() {
        return Err(format!("df failed: {}", String::from_utf8_lossy(&output.stderr)));
    }
    // Output: Filesystem 1B-blocks Used Available Use% Mounted
    // เราต้องการ field index 3 = Available
    let stdout = String::from_utf8_lossy(&output.stdout);
    let data_line = stdout.lines().last().ok_or("Empty df output")?;
    let fields: Vec<&str> = data_line.split_whitespace().collect();
    if fields.len() < 4 {
        return Err(format!("Unexpected df output: '{}'", data_line));
    }
    fields[3].parse::<u64>().map_err(|e| format!("Parse error: {}", e))
}

#[async_trait]
impl HealthCheck for DiskSpaceCheck {
    async fn check(&self) -> CheckResult {
        let start = Instant::now();
        let path = self.path.clone();
        let min_free = self.min_free_bytes;

        // spawn_blocking: รัน blocking I/O บน dedicated thread pool
        // ป้องกันไม่ให้บล็อก tokio async executor
        let result = tokio::task::spawn_blocking(
            move || get_free_bytes(&path)
        ).await;

        let latency_ms = start.elapsed().as_millis() as u64;

        match result {
            Ok(Ok(free_bytes)) => {
                let mut meta = HashMap::new();
                meta.insert("free_bytes".to_string(), free_bytes.to_string());
                meta.insert("free_gb".to_string(),
                    format!("{:.2}", free_bytes as f64 / 1_073_741_824.0));

                if free_bytes >= min_free {
                    CheckResult {
                        status: CheckStatus::Healthy,
                        latency_ms,
                        message: format!("{} has {:.2} GB free",
                            self.path, free_bytes as f64 / 1_073_741_824.0),
                        metadata: meta,
                    }
                } else if free_bytes >= min_free / 2 {
                    CheckResult {
                        status: CheckStatus::Degraded,
                        latency_ms,
                        message: format!("{} low: {:.2} GB free",
                            self.path, free_bytes as f64 / 1_073_741_824.0),
                        metadata: meta,
                    }
                } else {
                    CheckResult {
                        status: CheckStatus::Unhealthy,
                        latency_ms,
                        message: format!("{} critically low: {:.2} GB free",
                            self.path, free_bytes as f64 / 1_073_741_824.0),
                        metadata: meta,
                    }
                }
            }
            Ok(Err(e)) => CheckResult::unhealthy(format!("Disk check failed: {}", e), latency_ms),
            Err(e) => CheckResult::unhealthy(format!("Task error: {}", e), latency_ms),
        }
    }

    fn name(&self) -> &str { &self.name }
}
```

#### ProcessCheck — ตรวจสอบว่า Process กำลังทำงาน

สร้าง `src/checks/process.rs`:

```rust
use async_trait::async_trait;
use crate::types::{CheckResult, CheckStatus};
use super::HealthCheck;
use std::collections::HashMap;
use std::time::Instant;

pub struct ProcessCheck {
    pub name: String,
    pub process_name: String,
}

impl ProcessCheck {
    pub fn new(name: impl Into<String>, process_name: impl Into<String>) -> Self {
        ProcessCheck { name: name.into(), process_name: process_name.into() }
    }
}

/// ค้นหา PIDs ผ่าน /proc filesystem (Linux-specific)
/// อ่าน /proc/<pid>/cmdline ของแต่ละ process
fn find_pids_in_proc(process_name: &str) -> Result<Vec<u32>, String> {
    let mut pids = Vec::new();
    let entries = std::fs::read_dir("/proc").map_err(|e| e.to_string())?;

    for entry in entries.flatten() {
        let fname = entry.file_name();
        let fname_str = fname.to_string_lossy();

        // เฉพาะ directory ชื่อตัวเลข = PID
        if let Ok(pid) = fname_str.parse::<u32>() {
            let cmdline_path = format!("/proc/{}/cmdline", pid);
            if let Ok(bytes) = std::fs::read(&cmdline_path) {
                // cmdline ใช้ null byte แบ่ง arguments
                let cmdline = String::from_utf8_lossy(&bytes);
                let argv0 = cmdline.split('\0').next().unwrap_or("");
                // ชื่อ executable อยู่ส่วนท้ายของ path
                let exe_name = argv0.rsplit('/').next().unwrap_or(argv0);
                if exe_name == process_name || argv0.contains(process_name) {
                    pids.push(pid);
                }
            }
        }
    }
    Ok(pids)
}

#[async_trait]
impl HealthCheck for ProcessCheck {
    async fn check(&self) -> CheckResult {
        let start = Instant::now();
        let process_name = self.process_name.clone();

        let result = tokio::task::spawn_blocking(
            move || find_pids_in_proc(&process_name)
        ).await;

        let latency_ms = start.elapsed().as_millis() as u64;

        match result {
            Ok(Ok(pids)) if !pids.is_empty() => {
                let mut meta = HashMap::new();
                let pid_strs: Vec<String> = pids.iter().map(|p| p.to_string()).collect();
                meta.insert("pids".to_string(), pid_strs.join(","));
                meta.insert("count".to_string(), pids.len().to_string());
                CheckResult {
                    status: CheckStatus::Healthy,
                    latency_ms,
                    message: format!("'{}' running ({} instance(s))",
                        self.process_name, pids.len()),
                    metadata: meta,
                }
            }
            Ok(Ok(_)) => CheckResult::unhealthy(
                format!("Process '{}' not found", self.process_name),
                latency_ms,
            ),
            Ok(Err(e)) => CheckResult::unhealthy(
                format!("Process check error: {}", e), latency_ms),
            Err(e) => CheckResult::unhealthy(
                format!("Task error: {}", e), latency_ms),
        }
    }

    fn name(&self) -> &str { &self.name }
}
```

#### MockCheck สำหรับ Testing

สร้าง `src/checks/mock.rs`:

```rust
use async_trait::async_trait;
use crate::types::CheckResult;
use super::HealthCheck;

/// MockCheck คืนผลลัพธ์ที่กำหนดไว้ล่วงหน้า — ใช้ใน unit tests
/// ทำให้ tests ไม่ต้องมี network หรือ filesystem จริง
pub struct MockCheck {
    check_name: String,
    result: CheckResult,
}

impl MockCheck {
    pub fn new(name: impl Into<String>, result: CheckResult) -> Self {
        MockCheck { check_name: name.into(), result }
    }
}

#[async_trait]
impl HealthCheck for MockCheck {
    async fn check(&self) -> CheckResult {
        self.result.clone() // คืน clone ของผลลัพธ์ที่กำหนดไว้
    }

    fn name(&self) -> &str { &self.check_name }
}
```

### ขั้นที่ 4: SLO Tracker และ Error Budget

เพิ่ม `SloTracker` ใน `src/types.rs`:

```rust
/// ติดตาม SLO (Service Level Objective)
/// target_percent = 99.9 → uptime ต้องไม่ต่ำกว่า 99.9%
/// window_minutes = 43200 → หน้าต่าง 30-day month (30 × 24 × 60)
#[derive(Debug)]
pub struct SloTracker {
    pub target_percent: f64,
    pub window_minutes: u64,
    history: CheckHistory,
}

impl SloTracker {
    /// สร้าง SloTracker ใหม่
    /// - target_percent: เช่น 99.9
    /// - window_minutes: เช่น 43200 (30 days), 10080 (7 days)
    /// - history_size: จำนวนสูงสุดของ check records ที่เก็บ
    pub fn new(target_percent: f64, window_minutes: u64, history_size: usize) -> Self {
        SloTracker {
            target_percent,
            window_minutes,
            history: CheckHistory::new(history_size),
        }
    }

    pub fn record(&mut self, result: CheckResult) {
        self.history.push(result);
    }

    pub fn uptime_percent(&self) -> f64 {
        self.history.uptime_percent()
    }

    /// SLO breach = uptime ต่ำกว่า target
    pub fn is_breached(&self) -> bool {
        if self.history.is_empty() { return false; }
        self.history.uptime_percent() < self.target_percent
    }

    /// downtime budget สูงสุดที่ยอมรับได้ในหน่วยนาที
    /// สำหรับ 99.9% / 30 days = 43.2 นาที
    pub fn downtime_budget_minutes(&self) -> f64 {
        let downtime_fraction = 1.0 - (self.target_percent / 100.0);
        downtime_fraction * self.window_minutes as f64
    }

    /// budget ที่เหลืออยู่ (ค่าติดลบ = breach แล้ว)
    pub fn budget_remaining_minutes(&self) -> f64 {
        if self.history.is_empty() {
            return self.downtime_budget_minutes();
        }
        let used_fraction = 1.0 - (self.history.uptime_percent() / 100.0);
        let used_minutes = used_fraction * self.window_minutes as f64;
        self.downtime_budget_minutes() - used_minutes
    }

    pub fn history(&self) -> &CheckHistory { &self.history }
}
```

**ตัวอย่าง SLO calculation:**

```rust
let mut slo = SloTracker::new(99.9, 43200, 1440);

// Simulate 30 days: 1440 checks (1 per minute)
for _ in 0..1430 { slo.record(CheckResult::healthy("OK", 5)); }
for _ in 0..10   { slo.record(CheckResult::unhealthy("Err", 100)); }

println!("Uptime: {:.3}%", slo.uptime_percent());
// → 99.306%

println!("Budget: {:.1} min / Remaining: {:.1} min",
    slo.downtime_budget_minutes(),      // → 43.2
    slo.budget_remaining_minutes());    // → -300.0 (breach!)

println!("Breached: {}", slo.is_breached()); // → true
```

### ขั้นที่ 5: HealthScheduler — Concurrent Check Runner

สร้าง `src/scheduler.rs`:

```rust
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::RwLock;
use crate::checks::HealthCheck;
use crate::types::{CheckHistory, CheckResult};

#[derive(Debug, Clone)]
pub struct CheckConfig {
    pub interval_secs: u64,       // รัน check ทุก N วินาที
    pub timeout_secs: u64,        // timeout ต่อ check
    pub max_retries: u32,         // จำนวน retry เมื่อ check fail
    pub history_size: usize,      // จำนวนสูงสุดของ history entries
    pub alert_consecutive_failures: usize, // alert threshold
}

impl Default for CheckConfig {
    fn default() -> Self {
        CheckConfig {
            interval_secs: 60,
            timeout_secs: 10,
            max_retries: 2,
            history_size: 1440,   // 24 ชั่วโมง ที่ 1 check/min
            alert_consecutive_failures: 3,
        }
    }
}

pub struct ManagedCheck {
    pub check: Box<dyn HealthCheck>,
    pub config: CheckConfig,
    pub history: Arc<RwLock<CheckHistory>>,
}

impl ManagedCheck {
    pub fn new(check: Box<dyn HealthCheck>, config: CheckConfig) -> Self {
        let history_size = config.history_size;
        ManagedCheck {
            check,
            config,
            history: Arc::new(RwLock::new(CheckHistory::new(history_size))),
        }
    }

    /// รัน check หนึ่งครั้ง พร้อม timeout และ retry
    pub async fn run_once(&self) -> CheckResult {
        let timeout = Duration::from_secs(self.config.timeout_secs);
        let mut last_result = CheckResult::timeout_result(timeout.as_millis() as u64);

        for attempt in 0..=self.config.max_retries {
            match tokio::time::timeout(timeout, self.check.check()).await {
                Ok(result) => {
                    last_result = result;
                    // หยุดทันทีเมื่อ healthy หรือถึง retry สุดท้าย
                    if last_result.status.is_healthy() || attempt == self.config.max_retries {
                        break;
                    }
                }
                Err(_) => {
                    last_result = CheckResult::timeout_result(timeout.as_millis() as u64);
                    if attempt == self.config.max_retries { break; }
                }
            }
        }

        self.history.write().await.push(last_result.clone());
        last_result
    }

    pub fn check_name(&self) -> &str { self.check.name() }

    pub async fn should_alert(&self) -> bool {
        let h = self.history.read().await;
        h.consecutive_failures() >= self.config.alert_consecutive_failures
    }
}

/// Scheduler รัน checks ทั้งหมด concurrent ด้วย tokio::spawn
pub struct HealthScheduler {
    checks: Vec<Arc<ManagedCheck>>,
}

impl HealthScheduler {
    pub fn new() -> Self {
        HealthScheduler { checks: Vec::new() }
    }

    pub fn add_check(
        &mut self,
        check: Box<dyn HealthCheck>,
        config: CheckConfig,
    ) -> Arc<ManagedCheck> {
        let managed = Arc::new(ManagedCheck::new(check, config));
        self.checks.push(Arc::clone(&managed));
        managed
    }

    /// รัน checks ทั้งหมดพร้อมกัน (concurrent) แล้วรอให้ทุกตัวเสร็จ
    pub async fn run_all_once(&self) -> Vec<(String, CheckResult)> {
        let mut handles = Vec::new();
        for managed in &self.checks {
            let mc = Arc::clone(managed);
            handles.push(tokio::spawn(async move {
                let name = mc.check_name().to_string();
                let result = mc.run_once().await;
                (name, result)
            }));
        }

        let mut results = Vec::new();
        for h in handles {
            if let Ok(pair) = h.await {
                results.push(pair);
            }
        }
        results
    }

    pub fn check_count(&self) -> usize { self.checks.len() }
}
```

**ตัวอย่างใช้ scheduler:**

```rust
let mut scheduler = HealthScheduler::new();

scheduler.add_check(
    Box::new(HttpCheck::new("api", "https://api.example.com/health")),
    CheckConfig { interval_secs: 30, timeout_secs: 5, ..Default::default() },
);
scheduler.add_check(
    Box::new(TcpCheck::new("postgres", "db.internal", 5432)),
    CheckConfig::default(),
);
scheduler.add_check(
    Box::new(DiskSpaceCheck::with_min_free_gb("disk-root", "/", 5)),
    CheckConfig { interval_secs: 300, ..Default::default() },
);

// รัน checks ทั้งหมดพร้อมกัน
let results = scheduler.run_all_once().await;
for (name, result) in &results {
    println!("{}: {} ({}ms)", name, result.status, result.latency_ms);
}
```

### ขั้นที่ 6: ServiceHealth และ StatusPage

สร้าง `src/aggregator.rs`:

```rust
use crate::types::{CheckResult, CheckStatus};
use serde::{Deserialize, Serialize};

/// Health status ของ service หนึ่ง
/// overall = worst_of ของ checks ทั้งหมดใน service นั้น
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ServiceHealth {
    pub name: String,
    pub checks: Vec<(String, CheckResult)>, // (check_name, result)
    pub overall: CheckStatus,
}

impl ServiceHealth {
    pub fn new(name: impl Into<String>, checks: Vec<(String, CheckResult)>) -> Self {
        let statuses: Vec<CheckStatus> = checks.iter()
            .map(|(_, r)| r.status.clone())
            .collect();
        let overall = CheckStatus::worst_of(&statuses);
        ServiceHealth { name: name.into(), checks, overall }
    }

    pub fn from_results(name: impl Into<String>, results: Vec<CheckResult>) -> Self {
        let named: Vec<(String, CheckResult)> = results
            .into_iter()
            .enumerate()
            .map(|(i, r)| (format!("check-{}", i), r))
            .collect();
        ServiceHealth::new(name, named)
    }

    pub fn is_healthy(&self) -> bool { self.overall.is_healthy() }
    pub fn check_count(&self) -> usize { self.checks.len() }
}

/// รวม health status ของ services ทั้งหมด
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StatusPage {
    pub services: Vec<ServiceHealth>,
    pub overall: CheckStatus,
    pub generated_at_unix: u64,
}

impl StatusPage {
    pub fn new(services: Vec<ServiceHealth>) -> Self {
        let statuses: Vec<CheckStatus> = services.iter()
            .map(|s| s.overall.clone())
            .collect();
        let overall = CheckStatus::worst_of(&statuses);
        let ts = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .map(|d| d.as_secs())
            .unwrap_or(0);
        StatusPage { services, overall, generated_at_unix: ts }
    }

    pub fn healthy_count(&self) -> usize {
        self.services.iter().filter(|s| s.is_healthy()).count()
    }

    pub fn to_json(&self) -> Result<String, serde_json::Error> {
        serde_json::to_string_pretty(self)
    }
}
```

### ขั้นที่ 7: LatencyHistogram และ Prometheus Metrics

เพิ่ม `LatencyHistogram` ใน `src/types.rs` และ `CheckMetrics` ใน `src/metrics.rs`:

**LatencyHistogram (src/types.rs):**

```rust
/// Histogram สำหรับวัด latency distribution
/// ใช้ cumulative counts: counts[i] = จำนวน observations ที่ ≤ buckets[i]
#[derive(Debug, Clone)]
pub struct LatencyHistogram {
    pub buckets: Vec<u64>,   // upper bounds ในหน่วย ms
    pub counts: Vec<u64>,    // cumulative counts ต่อ bucket
    pub total_count: u64,
    pub sum_ms: u64,
}

impl LatencyHistogram {
    pub fn new(buckets: Vec<u64>) -> Self {
        let n = buckets.len();
        LatencyHistogram { buckets, counts: vec![0u64; n], total_count: 0, sum_ms: 0 }
    }

    /// Default: 1, 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000 ms
    pub fn default_buckets() -> Vec<u64> {
        vec![1, 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000]
    }

    /// บันทึก observation หนึ่งค่า
    /// เพิ่ม 1 ให้ทุก bucket ที่ latency_ms ≤ upper bound (cumulative)
    pub fn observe(&mut self, latency_ms: u64) {
        self.total_count += 1;
        self.sum_ms += latency_ms;
        for (i, &upper) in self.buckets.iter().enumerate() {
            if latency_ms <= upper {
                self.counts[i] += 1;
            }
        }
    }

    /// คืน bucket ที่เล็กที่สุดที่ latency_ms ≤ upper
    /// None = ตกใน +Inf bucket (เกินทุก bucket)
    pub fn bucket_for(&self, latency_ms: u64) -> Option<u64> {
        self.buckets.iter().copied().find(|&upper| latency_ms <= upper)
    }

    pub fn mean_ms(&self) -> f64 {
        if self.total_count == 0 { return 0.0; }
        self.sum_ms as f64 / self.total_count as f64
    }

    /// สร้าง Prometheus text format lines
    pub fn to_prometheus_lines(&self, name: &str) -> String {
        let mut out = String::new();
        for (&upper, &count) in self.buckets.iter().zip(self.counts.iter()) {
            out.push_str(&format!("{}_bucket{{le=\"{}\"}} {}\n", name, upper, count));
        }
        out.push_str(&format!("{}_bucket{{le=\"+Inf\"}} {}\n", name, self.total_count));
        out.push_str(&format!("{}_sum {}\n", name, self.sum_ms));
        out.push_str(&format!("{}_count {}\n", name, self.total_count));
        out
    }
}
```

**CheckMetrics (src/metrics.rs):**

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::{Arc, Mutex};
use crate::types::LatencyHistogram;

/// Prometheus-compatible metrics สำหรับ health check service
pub struct CheckMetrics {
    pub check_total: Arc<AtomicU64>,
    pub check_failures: Arc<AtomicU64>,
    pub histogram: Mutex<LatencyHistogram>,
}

impl CheckMetrics {
    pub fn new() -> Self {
        CheckMetrics {
            check_total: Arc::new(AtomicU64::new(0)),
            check_failures: Arc::new(AtomicU64::new(0)),
            histogram: Mutex::new(LatencyHistogram::new(LatencyHistogram::default_buckets())),
        }
    }

    pub fn record(&self, latency_ms: u64, is_failure: bool) {
        self.check_total.fetch_add(1, Ordering::Relaxed);
        if is_failure {
            self.check_failures.fetch_add(1, Ordering::Relaxed);
        }
        if let Ok(mut hist) = self.histogram.lock() {
            hist.observe(latency_ms);
        }
    }

    pub fn total(&self) -> u64 { self.check_total.load(Ordering::Relaxed) }
    pub fn failures(&self) -> u64 { self.check_failures.load(Ordering::Relaxed) }

    pub fn success_rate(&self) -> f64 {
        let total = self.total();
        if total == 0 { return 100.0; }
        ((total - self.failures()) as f64 / total as f64) * 100.0
    }

    /// Export Prometheus text format สำหรับ /metrics endpoint
    pub fn to_prometheus(&self) -> String {
        let mut out = String::new();
        out.push_str("# HELP health_check_total Total health checks executed\n");
        out.push_str("# TYPE health_check_total counter\n");
        out.push_str(&format!("health_check_total {}\n\n", self.total()));
        out.push_str("# HELP health_check_failures_total Failed health checks\n");
        out.push_str("# TYPE health_check_failures_total counter\n");
        out.push_str(&format!("health_check_failures_total {}\n\n", self.failures()));
        out.push_str("# HELP health_check_latency_ms Latency histogram (ms)\n");
        out.push_str("# TYPE health_check_latency_ms histogram\n");
        if let Ok(hist) = self.histogram.lock() {
            out.push_str(&hist.to_prometheus_lines("health_check_latency_ms"));
        }
        out
    }
}
```

**ตัวอย่าง Prometheus output:**

```
# HELP health_check_total Total health checks executed
# TYPE health_check_total counter
health_check_total 150

# HELP health_check_failures_total Failed health checks
# TYPE health_check_failures_total counter
health_check_failures_total 3

# HELP health_check_latency_ms Latency histogram (ms)
# TYPE health_check_latency_ms histogram
health_check_latency_ms_bucket{le="1"} 0
health_check_latency_ms_bucket{le="5"} 10
health_check_latency_ms_bucket{le="10"} 45
health_check_latency_ms_bucket{le="25"} 89
health_check_latency_ms_bucket{le="50"} 120
health_check_latency_ms_bucket{le="100"} 143
health_check_latency_ms_bucket{le="250"} 148
health_check_latency_ms_bucket{le="500"} 149
health_check_latency_ms_bucket{le="1000"} 150
health_check_latency_ms_bucket{le="+Inf"} 150
health_check_latency_ms_sum 8450
health_check_latency_ms_count 150
```

### ขั้นที่ 8: HTTP API — /health, /status, /metrics

เพิ่ม HTTP API server ใน `src/main.rs` สำหรับ 3 endpoints:

```rust
use std::sync::Arc;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use health_checker::{CheckMetrics, ServiceHealth, StatusPage};
use health_checker::checks::mock::MockCheck;
use health_checker::checks::HealthCheck;
use health_checker::scheduler::{CheckConfig, HealthScheduler};
use health_checker::types::CheckResult;

/// จัดการ HTTP request แบบ minimal (ไม่ใช้ web framework)
async fn handle_request(
    mut stream: TcpStream,
    page: Arc<StatusPage>,
    metrics: Arc<CheckMetrics>,
) {
    let mut buf = [0u8; 4096];
    if stream.read(&mut buf).await.is_err() { return; }
    let request = String::from_utf8_lossy(&buf);
    let first_line = request.lines().next().unwrap_or("");

    let (status_line, body, content_type) = if first_line.starts_with("GET /metrics") {
        ("HTTP/1.1 200 OK", metrics.to_prometheus(), "text/plain; charset=utf-8")
    } else if first_line.starts_with("GET /status") {
        let json = page.to_json().unwrap_or_else(|_| "{}".to_string());
        ("HTTP/1.1 200 OK", json, "application/json")
    } else if first_line.starts_with("GET /health") {
        let body = if page.overall.is_healthy() {
            r#"{"status":"ok"}"#.to_string()
        } else {
            format!(r#"{{"status":"{}"}}"#, page.overall)
        };
        let status = if page.overall.is_healthy() { "HTTP/1.1 200 OK" } else { "HTTP/1.1 503 Service Unavailable" };
        (status, body, "application/json")
    } else {
        ("HTTP/1.1 404 Not Found", "Not found".to_string(), "text/plain")
    };

    let response = format!(
        "{}\r\nContent-Type: {}\r\nContent-Length: {}\r\n\r\n{}",
        status_line, content_type, body.len(), body
    );
    let _ = stream.write_all(response.as_bytes()).await;
}

#[tokio::main]
async fn main() {
    let mut scheduler = HealthScheduler::new();
    scheduler.add_check(
        Box::new(MockCheck::new("api", CheckResult::healthy("HTTP 200", 45))),
        CheckConfig::default(),
    );

    let results = scheduler.run_all_once().await;
    let service = ServiceHealth::from_results(
        "my-service",
        results.into_iter().map(|(_, r)| r).collect(),
    );
    let page = Arc::new(StatusPage::new(vec![service]));
    let metrics = Arc::new(CheckMetrics::new());
    metrics.record(45, false);

    let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();
    println!("Health Check API listening on http://127.0.0.1:8080");
    println!("  GET /health  → own service health");
    println!("  GET /status  → all services JSON");
    println!("  GET /metrics → Prometheus counters");

    loop {
        if let Ok((stream, _)) = listener.accept().await {
            let p = Arc::clone(&page);
            let m = Arc::clone(&metrics);
            tokio::spawn(handle_request(stream, p, m));
        }
    }
}
```

**ทดสอบด้วย curl:**

```bash
$ curl http://localhost:8080/health
{"status":"ok"}

$ curl http://localhost:8080/status | jq '.overall'
"Healthy"

$ curl http://localhost:8080/metrics
# HELP health_check_total Total health checks executed
# TYPE health_check_total counter
health_check_total 1
...
```

## การทดสอบ (Testing)

โปรเจคนี้มี unit tests ครอบคลุมทุก logic component ใน `src/types.rs`, `src/aggregator.rs`, `src/metrics.rs`, และ `src/checks/mock.rs`

### รัน cargo test จริง

```bash
cargo test
```

### ผล cargo test จริง (verbatim)

```
   Compiling health-checker v0.1.0 (...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.77s
     Running unittests src/lib.rs (target/debug/deps/health_checker-9e6e4e7b64f05b5a)

running 34 tests
test aggregator::tests::test_service_health_all_healthy ... ok
test aggregator::tests::test_status_page_all_healthy ... ok
test aggregator::tests::test_service_health_overall_worst_of_checks ... ok
test aggregator::tests::test_status_page_aggregates_services ... ok
test aggregator::tests::test_status_page_json_serialization ... ok
test metrics::tests::test_metrics_counts_total_and_failures ... ok
test checks::mock::tests::test_mock_check_returns_unhealthy ... ok
test checks::mock::tests::test_mock_check_returns_healthy ... ok
test checks::mock::tests::test_mock_check_with_metadata ... ok
test metrics::tests::test_metrics_prometheus_output_contains_counters ... ok
test types::tests::test_check_status_is_healthy_only_for_healthy ... ok
test metrics::tests::test_metrics_success_rate ... ok
test metrics::tests::test_metrics_success_rate_empty ... ok
test types::tests::test_check_status_severity_order ... ok
test types::tests::test_check_status_worst_of_empty_returns_unknown ... ok
test types::tests::test_check_status_worst_of_multiple_returns_unhealthy ... ok
test types::tests::test_check_status_worst_of_single ... ok
test types::tests::test_check_status_worst_of_timeout_beats_degraded ... ok
test types::tests::test_histogram_bucket_selection ... ok
test types::tests::test_histogram_cumulative_counts ... ok
test types::tests::test_histogram_mean_calculation ... ok
test types::tests::test_histogram_prometheus_format ... ok
test types::tests::test_history_basic_push_and_len ... ok
test types::tests::test_history_consecutive_failures_counted ... ok
test types::tests::test_history_consecutive_failures_none_when_latest_healthy ... ok
test types::tests::test_history_consecutive_failures_reset_by_healthy ... ok
test types::tests::test_history_ring_buffer_evicts_oldest ... ok
test types::tests::test_history_uptime_all_failing ... ok
test types::tests::test_history_uptime_all_healthy ... ok
test types::tests::test_history_uptime_partial_50_percent ... ok
test types::tests::test_slo_budget_calculation_99_9_percent ... ok
test types::tests::test_slo_not_breached_when_empty ... ok
test types::tests::test_slo_breached_when_too_many_failures ... ok
test types::tests::test_slo_not_breached_when_all_healthy ... ok

test result: ok. 34 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/health_checker-80616467d487f46b)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests health_checker

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### ผลการรัน demo (`cargo run`)

```
=== Health Check Service Demo ===

Worst of [Healthy, Degraded, Unhealthy]: unhealthy

History (10 entries):
  Uptime: 80.0%
  Consecutive failures: 2

SLO Tracking (99.9% target):
  Uptime: 99.00%
  Budget (30-day): 43.2 min
  SLO breached: true
  Budget remaining: -388.8 min

Latency Histogram:
  Mean: 424.4ms
  Bucket for 75ms: Some(100)
  Bucket for 6000ms: None (→ +Inf)

Mock check: healthy (42ms - HTTP 200)

Status Page:
  Overall: degraded
  api-service → healthy (2 checks)
  database → degraded (2 checks)

Prometheus Metrics:
  Total: 4, Failures: 1, Success rate: 75.0%
  Prometheus lines: 24 lines

Scheduler (3 checks):
  web → healthy (45ms)
  db → healthy (8ms)
  cache → degraded (15ms)

Demo complete!
```

### รายการ Tests แยกตาม Module

#### types.rs — CheckStatus (6 tests)

| Test | ตรวจสอบ |
|------|---------|
| `test_check_status_severity_order` | Healthy < Degraded < Timeout < Unhealthy < Unknown |
| `test_check_status_worst_of_empty_returns_unknown` | worst_of([]) = Unknown |
| `test_check_status_worst_of_single` | worst_of([X]) = X |
| `test_check_status_worst_of_multiple_returns_unhealthy` | worst_of([H,D,U]) = Unhealthy |
| `test_check_status_worst_of_timeout_beats_degraded` | worst_of([D,T]) = Timeout |
| `test_check_status_is_healthy_only_for_healthy` | Healthy → true, ทุกอื่น → false |

#### types.rs — CheckHistory (8 tests)

| Test | ตรวจสอบ |
|------|---------|
| `test_history_basic_push_and_len` | push/len/is_empty |
| `test_history_ring_buffer_evicts_oldest` | old items evicted when full |
| `test_history_uptime_all_healthy` | 100% uptime |
| `test_history_uptime_all_failing` | 0% uptime |
| `test_history_uptime_partial_50_percent` | 50% uptime |
| `test_history_consecutive_failures_none_when_latest_healthy` | reset when latest = healthy |
| `test_history_consecutive_failures_counted` | counts trailing failures |
| `test_history_consecutive_failures_reset_by_healthy` | healthy mid-stream resets |

#### types.rs — SloTracker (4 tests)

| Test | ตรวจสอบ |
|------|---------|
| `test_slo_not_breached_when_empty` | ไม่มี records → ไม่ breach |
| `test_slo_not_breached_when_all_healthy` | 100% uptime → ไม่ breach |
| `test_slo_breached_when_too_many_failures` | 99.8% < 99.9% → breach |
| `test_slo_budget_calculation_99_9_percent` | 0.1% × 43200 = 43.2 นาที |

#### types.rs — LatencyHistogram (4 tests)

| Test | ตรวจสอบ |
|------|---------|
| `test_histogram_bucket_selection` | bucket_for() หาถูก bucket |
| `test_histogram_cumulative_counts` | cumulative counting logic |
| `test_histogram_mean_calculation` | mean = sum / count |
| `test_histogram_prometheus_format` | Prometheus text format |

#### aggregator.rs (5 tests)

| Test | ตรวจสอบ |
|------|---------|
| `test_service_health_overall_worst_of_checks` | [H,U,D] → overall=Unhealthy |
| `test_service_health_all_healthy` | all H → overall=Healthy |
| `test_status_page_aggregates_services` | multi-service worst_of |
| `test_status_page_all_healthy` | healthy_count/unhealthy_count |
| `test_status_page_json_serialization` | to_json() |

#### metrics.rs (4 tests)

| Test | ตรวจสอบ |
|------|---------|
| `test_metrics_counts_total_and_failures` | atomic counter accuracy |
| `test_metrics_success_rate` | success_rate = (total-fail)/total |
| `test_metrics_prometheus_output_contains_counters` | Prometheus format |
| `test_metrics_success_rate_empty` | empty → 100% |

#### checks/mock.rs (3 tests)

| Test | ตรวจสอบ |
|------|---------|
| `test_mock_check_returns_healthy` | async check คืน healthy result |
| `test_mock_check_returns_unhealthy` | async check คืน unhealthy result |
| `test_mock_check_with_metadata` | metadata propagated correctly |

## ข้อผิดพลาดที่พบบ่อย (Pitfalls)

### Pitfall 1: Borrow ใน async_trait ทำให้ lifetime error

```rust
// ❌ Error: host_port does not live long enough
let host_port = format!("{}:80", self.hostname);
let _ = tokio::time::timeout(
    timeout,
    tokio::net::lookup_host(&host_port)  // borrow ที่อาจ outlive host_port
).await;

// ✅ แก้: ส่ง owned String ให้ function (String implements ToSocketAddrs)
let host_port: String = format!("{}:80", self.hostname);
let _ = tokio::time::timeout(
    timeout,
    tokio::net::lookup_host(host_port)  // moved, ไม่มี borrow
).await;
```

`async_trait` บังคับให้ future ที่สร้างจาก method เป็น `'static` เพราะ future อาจถูกส่งข้าม thread boundaries กฎคือ: ทุก reference ใน future ต้อง live ตลอดชีวิตของ future ซึ่งอาจนานกว่า scope ของ method call

### Pitfall 2: ใช้ `Vec::remove(0)` แทน `VecDeque::pop_front()`

```rust
// ❌ O(n) — ต้อง shift ทุก element หลัง remove
let mut results: Vec<CheckResult> = Vec::new();
if results.len() >= max_size {
    results.remove(0); // shift O(n) elements!
}

// ✅ O(1) — VecDeque ออกแบบมาสำหรับ front removal
let mut results: VecDeque<CheckResult> = VecDeque::new();
if results.len() >= max_size {
    results.pop_front(); // O(1)!
}
```

ที่ 1440 entries (24h history) ความแตกต่าง O(n) vs O(1) ไม่มีผลมาก แต่ที่ 1,000,000 entries (เช่น metric ring buffer) จะเห็นความต่างชัดเจน

### Pitfall 3: Blocking I/O บน tokio async thread

```rust
// ❌ DANGER: df command blocks async thread ทำให้ tokio stall
#[async_trait]
impl HealthCheck for DiskSpaceCheck {
    async fn check(&self) -> CheckResult {
        // std::process::Command::output() เป็น blocking!
        let output = std::process::Command::new("df").arg("/").output().unwrap();
        // ...
    }
}

// ✅ ใช้ spawn_blocking: รัน blocking code บน dedicated thread pool
async fn check(&self) -> CheckResult {
    let path = self.path.clone();
    tokio::task::spawn_blocking(move || {
        std::process::Command::new("df").arg(&path).output()
    }).await
    // ...
}
```

tokio runtime มี async threads จำนวนจำกัด (default = number of CPUs) ถ้าแต่ละ thread block รอ I/O อยู่ ระบบทั้งหมดจะ stall `spawn_blocking` ส่ง blocking work ไปที่ blocking thread pool ที่แยกต่างหาก

### Pitfall 4: ลืม Clone ก่อนส่งเข้า tokio::spawn

```rust
// ❌ Error: cannot move `managed` — ถูก borrow อยู่ใน loop
for managed in &self.checks {
    tokio::spawn(async move {
        managed.run_once().await  // Error: managed ไม่ได้ own ตรงนี้
    });
}

// ✅ Clone Arc ก่อนส่งเข้า spawn (clone Arc = เพิ่ม ref count, ไม่ deep copy)
for managed in &self.checks {
    let mc = Arc::clone(managed);  // ref count++
    tokio::spawn(async move {
        mc.run_once().await  // mc owns Arc copy นี้
    });
}
```

`tokio::spawn` ต้องการ `'static` future ซึ่งหมายความว่า future ต้องไม่ borrow อะไรที่อาจ drop ก่อนมัน `Arc::clone()` สร้าง owned copy ของ pointer (ไม่ใช่ deep copy ของ data) ที่ปลอดภัยส่งข้าม thread

### Pitfall 5: float comparison ใน SLO test

```rust
// ❌ อาจ fail เพราะ floating point precision
assert_eq!(slo.uptime_percent(), 99.9);

// ✅ ใช้ epsilon comparison
assert!((slo.uptime_percent() - 99.9).abs() < 0.001);

// หรือ round ก่อนเปรียบเทียบ
assert_eq!((slo.uptime_percent() * 1000.0).round() as u64, 999);
```

`f64` arithmetic มี precision error เล็กน้อย การหาร integer ด้วย integer อาจให้ผลต่างจากค่า ideal เช่น `999 / 1000 * 100` ใน f64 อาจเป็น `99.89999999999...` ไม่ใช่ `99.9` พอดี

## การ Package และ Deploy

### Build release binary

```bash
cargo build --release
# binary อยู่ที่ target/release/health-checker
ls -lh target/release/health-checker
# -rwxr-xr-x  1 user  staff  6.2M  health-checker
```

### Docker deployment

```dockerfile
# Multi-stage build: ลด image size
FROM rust:1.80-slim as builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/health-checker /usr/local/bin/
EXPOSE 8080
CMD ["health-checker"]
```

```bash
docker build -t health-checker:latest .
docker run -p 8080:8080 health-checker:latest
```

### Kubernetes ConfigMap สำหรับ check configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: health-checker-config
data:
  config.toml: |
    [[checks]]
    name = "api-gateway"
    type = "http"
    url = "http://api-gateway/health"
    interval_secs = 30

    [[checks]]
    name = "postgres"
    type = "tcp"
    host = "postgres.db.svc.cluster.local"
    port = 5432

    [slo]
    target_percent = 99.9
    window_days = 30
    alert_consecutive_failures = 3
```

### Systemd service unit

```ini
[Unit]
Description=Health Check Service
After=network.target

[Service]
Type=simple
User=health-checker
ExecStart=/usr/local/bin/health-checker --bind 0.0.0.0:8080
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม TOML Configuration File (ระดับกลาง)

ปัจจุบัน checks ถูก hard-code ใน main.rs อยากให้อ่าน config จากไฟล์:

```rust
// checks.toml
[[checks]]
name = "api"
type = "http"
url = "https://api.example.com/health"
timeout_secs = 5
interval_secs = 30

[[checks]]
name = "database"
type = "tcp"
host = "db.internal"
port = 5432
```

เพิ่ม dependency `toml = "0.8"` และ `serde` derive สำหรับ struct ที่สอดคล้องกัน จากนั้นสร้าง `fn load_config(path: &str) -> Vec<Box<dyn HealthCheck>>` ที่อ่านไฟล์และสร้าง check instances ให้อัตโนมัติ

### แบบฝึกหัดที่ 2: Alert Notification ผ่าน Webhook (ระดับกลาง)

เพิ่ม alerting system ที่ส่ง webhook เมื่อ consecutive failures ถึง threshold:

```rust
pub struct AlertManager {
    webhooks: Vec<String>,  // Slack/Teams webhook URLs
    cooldown_secs: u64,     // ไม่ alert ซ้ำใน cooldown period
    last_alert: std::collections::HashMap<String, std::time::Instant>,
}

impl AlertManager {
    pub async fn check_and_alert(
        &mut self,
        check_name: &str,
        consecutive_failures: usize,
        threshold: usize,
    ) {
        if consecutive_failures < threshold { return; }
        // ตรวจ cooldown
        // ส่ง POST ไปที่ webhook URLs ด้วย reqwest
    }
}
```

สร้าง Slack message format ที่สวยงาม พร้อม service name, failure count, และ last error message

### แบบฝึกหัดที่ 3: Persistent History ด้วย SQLite (ระดับสูง)

`CheckHistory` ปัจจุบันเก็บเฉพาะ in-memory ข้อมูลหายเมื่อ restart เพิ่ม persistence ด้วย `rusqlite`:

```rust
pub struct PersistentHistory {
    conn: rusqlite::Connection,
    check_name: String,
    max_size: usize,
}

impl PersistentHistory {
    pub fn new(db_path: &str, check_name: &str, max_size: usize) -> Result<Self, rusqlite::Error>;
    pub fn push(&self, result: &CheckResult) -> Result<(), rusqlite::Error>;
    pub fn uptime_percent_last_n(&self, n: usize) -> Result<f64, rusqlite::Error>;
    pub fn get_history(&self, limit: usize) -> Result<Vec<CheckResult>, rusqlite::Error>;
}
```

Schema: `CREATE TABLE check_history (id INTEGER PRIMARY KEY, check_name TEXT, status TEXT, latency_ms INTEGER, message TEXT, created_at INTEGER)`

### แบบฝึกหัดที่ 4: Multi-Region Check Aggregation (ระดับสูง)

Real-world health checking มักทำจากหลาย region พร้อมกัน เพิ่ม:

```rust
pub struct RegionalCheck {
    pub region: String,          // "us-east-1", "ap-southeast-1"
    pub check: Box<dyn HealthCheck>,
}

pub struct MultiRegionAggregator {
    checks: Vec<RegionalCheck>,
}

impl MultiRegionAggregator {
    /// ถ้า majority ของ region ผ่าน → Healthy
    /// ถ้า 1-2 region fail → Degraded (อาจเป็น network partition)
    /// ถ้าทุก region fail → Unhealthy (service จริงมีปัญหา)
    pub async fn check_with_quorum(&self) -> CheckResult;
}
```

การ aggregate แบบนี้ช่วยลด false positive จาก network issues ใน region เดียว

### แบบฝึกหัดที่ 5: Grafana Dashboard Export (ระดับกลาง)

เพิ่ม `GET /grafana-dashboard` endpoint ที่ generate Grafana dashboard JSON สำหรับแสดง:
- Uptime percentage time series
- Latency histogram heatmap
- SLO error budget gauge
- Consecutive failures alerting panel

```rust
pub fn generate_grafana_dashboard(service_names: &[&str]) -> serde_json::Value {
    // สร้าง JSON ที่ import ได้ใน Grafana โดยตรง
}
```

### แบบฝึกหัดที่ 6: gRPC Health Protocol (ระดับสูง)

Kubernetes และ gRPC ecosystem ใช้ [gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md) มาตรฐาน เพิ่ม crate `tonic` และ implement:

```protobuf
service Health {
    rpc Check (HealthCheckRequest) returns (HealthCheckResponse);
    rpc Watch (HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

`Watch` ส่ง stream ของ status changes แบบ real-time ซึ่งใช้ `tokio::sync::watch::channel` ได้

## สรุป

ในโปรเจคนี้เราได้สร้าง Health Check Service ที่ production-ready ครอบคลุม:

**Core types ที่สร้าง:**
- `CheckStatus` + `worst_of()` — severity-ordered aggregation ที่รับประกันด้วย type system
- `CheckResult` + builder methods — ergonomic API สำหรับสร้าง check results
- `CheckHistory` (ring buffer) — O(1) push/pop พร้อม uptime% และ consecutive failures
- `SloTracker` — คำนวณ error budget แบบ Google SRE standard
- `LatencyHistogram` — cumulative bucket counting แบบ Prometheus specification

**Check implementations ที่สร้าง:**
- `HttpCheck` — reqwest + timeout สำหรับ HTTP endpoint monitoring
- `TcpCheck` — tokio TCP connect สำหรับ port availability
- `DnsCheck` — tokio lookup_host สำหรับ DNS health
- `DiskSpaceCheck` — spawn_blocking + df command สำหรับ storage monitoring
- `ProcessCheck` — /proc scanning สำหรับ process liveness

**Infrastructure patterns ที่ได้เรียน:**
1. **`async_trait`** — สร้าง polymorphic async interface ที่สะอาดสำหรับ check plugins
2. **Ring buffer** ด้วย `VecDeque` — O(1) circular buffer ที่ memory-efficient
3. **`spawn_blocking`** — แยก blocking I/O จาก async runtime อย่างถูกต้อง
4. **Concurrent checks** ด้วย `tokio::spawn` + `Arc` — รัน checks หลายร้อยตัวพร้อมกัน
5. **SLO mathematics** — error budget calculation จาก first principles
6. **Prometheus histogram** — cumulative bucket format ที่ compatible กับ Grafana

โปรเจค G07 ถัดไปจะสร้าง **Secret Scanner** — ระบบ scan codebase หา secrets ที่รั่วไหล เช่น API keys, passwords, private keys ในไฟล์ต่าง ๆ ซึ่งจะใช้ pattern matching, regex engine, และ parallel file scanning ที่ได้จาก patterns ในโปรเจคนี้

---

**โปรเจคก่อนหน้า:** [Project G05: Config Manager](project-g05-config-manager.md) | **โปรเจคถัดไป:** [Project G07: Secret Scanner](project-g07-secret-scanner.md)
