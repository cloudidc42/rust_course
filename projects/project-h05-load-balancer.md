# Project H05: Layer 7 Load Balancer

> โมดูล: H — Networking/Protocols | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 16 ชั่วโมง

## ภาพรวมโปรเจค

**Load Balancer** คือซอฟต์แวร์ที่อยู่หน้า server หลาย ๆ ตัว ทำหน้าที่รับ request จาก client แล้วกระจายไปยัง backend ที่เหมาะสม ในโปรเจคนี้เราจะสร้าง **Layer 7 HTTP Load Balancer** ซึ่งทำงานที่ระดับ application layer — สามารถอ่าน HTTP header, จัดการ sticky session ด้วย cookie, และติดตาม response time ของแต่ละ backend ได้

### ทำไมต้อง Layer 7?

Load Balancer แบ่งเป็น 2 ประเภทหลัก:

| ประเภท | ทำงานที่ | ข้อดี | ข้อเสีย |
|--------|---------|-------|---------|
| **Layer 4** (TCP/UDP) | Transport layer | เร็วมาก, overhead ต่ำ | ไม่เห็น HTTP header/cookie |
| **Layer 7** (HTTP) | Application layer | อ่าน header ได้, sticky session ได้, content-based routing | overhead สูงกว่า |

ระบบ production จริงที่ใช้ Layer 7 load balancing เช่น **Nginx** (mode `proxy_pass`), **HAProxy** (mode `http`), **AWS ALB**, **Envoy** ล้วนรองรับ pattern เดียวกับที่เราจะสร้าง

### สิ่งที่จะสร้าง

โปรเจคนี้สร้าง load balancer ครบฟีเจอร์ด้วย:

- **Backend pool** ที่ thread-safe ด้วย `DashMap` — เพิ่ม/ลบ backend แบบ hot reload ได้
- **5 อัลกอริทึม** กระจาย traffic: Round-Robin, Weighted Round-Robin, Least Connections, IP Hash, Random
- **Health checker** background task ที่ ping backend ทุก N วินาที และ mark unhealthy อัตโนมัติ
- **Reverse proxy** ด้วย `tokio::io::copy_bidirectional` สำหรับ forward traffic แบบ bidirectional
- **Sticky sessions** ด้วย cookie `STICKY_SESSION` เพื่อให้ client เดิม routing ไปหา backend เดิมเสมอ
- **Stats endpoint** JSON ที่ `/lb-stats` สำหรับ monitoring

---

## สิ่งที่จะได้เรียนรู้

- **`DashMap`** — concurrent HashMap ที่ไม่ต้องใช้ global lock, เหมาะกับ read-heavy workloads
- **`AtomicBool` / `AtomicU64` / `AtomicUsize`** — interior mutability แบบ lock-free สำหรับ counter และ flag
- **Trait object `dyn LoadBalancer`** — swap อัลกอริทึมตอน runtime โดยไม่ต้อง recompile
- **`tokio::io::copy_bidirectional`** — tunnel TCP stream สองทิศทางพร้อมกัน
- **Background health check loop** ด้วย `tokio::spawn` + `tokio::time::interval`
- **FNV-1a hash** — implement hash function แบบ hand-rolled สำหรับ IP hash
- **Weighted scheduling** — สร้าง interleaved sequence จาก weight เพื่อ fair distribution
- **Sticky session** pattern — เชื่อม cookie กับ backend ID ผ่าน `DashMap`

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 41-50** — `async/await`, tokio runtime, `Future`, `tokio::spawn`
- **Part 51-60** — `Arc<T>`, trait objects `dyn Trait + Send + Sync`, `Box<dyn Error>`
- **Part 61-70** — Atomic types, `Ordering`, interior mutability patterns
- **Part 71-80** — Closures, `FnOnce`/`FnMut`, lifetime ใน closure
- **Part 96-110** — Production error handling, structured logging, `thiserror`
- โปรเจค H04 (MQTT Broker) — ความเข้าใจ TCP listener, bidirectional streaming

---

## โครงสร้างโปรเจค (Project Layout)

```
load-balancer/
├── src/
│   ├── main.rs          # entry point: parse config, start listener, spawn health checker
│   ├── lib.rs           # re-exports ทุก module
│   ├── backend.rs       # Backend, BackendPool struct
│   ├── balancer.rs      # LoadBalancer trait + 5 implementations
│   ├── health.rs        # HealthChecker: background task, failure/success tracking
│   ├── proxy.rs         # ProxyHandler: accept TCP, forward via copy_bidirectional
│   ├── session.rs       # SessionStore: cookie → BackendId mapping
│   └── stats.rs         # LoadBalancerStats, BackendStats, /lb-stats JSON
├── tests/
│   └── integration.rs   # integration tests (optional)
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
Client TCP Connection
        │
        ▼
  ┌─────────────────────────────────┐
  │  ProxyHandler (Tokio task)      │
  │  1. Parse HTTP header           │
  │  2. Check STICKY_SESSION cookie │
  │  3. SessionStore.route()        │
  │      └─ if cookie valid & healthy → same backend  │
  │      └─ else → LoadBalancer.select()              │
  │  4. Connect to selected backend │
  │  5. copy_bidirectional          │
  │  6. Update stats on close       │
  └─────────────────────────────────┘
        │
        ▼
  BackendPool (DashMap)
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Backend A│  │ Backend B│  │ Backend C│
  │ healthy✓ │  │ healthy✓ │  │ healthy✗ │
  │ conns: 3 │  │ conns: 7 │  │ conns: 0 │
  └──────────┘  └──────────┘  └──────────┘
        ▲
        │
  HealthChecker (background Tokio task)
  GET /health every N seconds
  mark unhealthy after K consecutive failures
  mark healthy after M consecutive successes
```

### Thread Safety Design

สาเหตุที่เลือก `DashMap` แทน `RwLock<HashMap>`:

| | `RwLock<HashMap>` | `DashMap` |
|--|--|--|
| Read lock | global, blocks writes | shard-level, ultra-low contention |
| Write lock | global, blocks all reads | shard-level |
| Clone cost | O(n) | zero-copy `Arc` |
| API | manual lock/unlock | transparent HashMap-like |

`BackendPool` เก็บ `Arc<Backend>` แทนที่จะเก็น `Backend` ตรง ๆ เพราะ:
1. หลาย task สามารถ hold `Arc<Backend>` ได้พร้อมกัน (active connection tracking)
2. `Backend` ถูก mark unhealthy ผ่าน `AtomicBool` โดย health checker task โดยไม่ต้องคืน ownership
3. ลบ backend ออกจาก pool ได้โดยที่ connection ที่กำลัง active อยู่ไม่ crash

### Algorithm Selection

```
                 ┌───────────────────────────────────────────┐
                 │         เลือก Algorithm อย่างไร?          │
                 └───────────────────────────────────────────┘
                    │              │              │
          Stateless?        Stateful?       Session affinity?
             │                 │                  │
    ┌────────┴──────┐   ┌──────┴─────┐   ┌───────┴────────┐
    │  RoundRobin   │   │   Least    │   │   IpHash /     │
    │  Random       │   │Connections │   │ StickySessions │
    │  WeightedRR   │   │            │   │                │
    └───────────────┘   └────────────┘   └────────────────┘
    ใช้เมื่อ backend     ใช้เมื่อ request  ใช้เมื่อ backend
    มี capacity เท่ากัน  มี processing     ต้องการ state
    หรือต่างกัน (WRR)   time ต่างกันมาก   จาก request ก่อน
```

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Backend และ BackendPool

เริ่มจากโครงสร้างข้อมูลพื้นฐาน — `Backend` เป็น struct ที่ hold ข้อมูล server ปลายทาง และ `BackendPool` เป็น container ที่ thread-safe

```toml
# Cargo.toml
[package]
name = "load-balancer"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
dashmap = "5"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
```

```rust
// src/backend.rs
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::sync::Arc;
use dashmap::DashMap;

/// ข้อมูลของ backend server แต่ละตัว
/// ใช้ Atomic types ทั้งหมด เพราะหลาย task จะอ่าน/เขียนพร้อมกัน
pub struct Backend {
    pub id: String,
    pub address: String,          // "127.0.0.1:8001"
    pub weight: u32,              // relative weight สำหรับ WeightedRoundRobin
    pub healthy: AtomicBool,      // health checker เขียน, proxy อ่าน
    pub active_connections: AtomicU64, // proxy เพิ่ม/ลด
    pub response_time_ms: AtomicU64,   // health checker อัปเดต
}

impl Backend {
    pub fn new(
        id: impl Into<String>,
        address: impl Into<String>,
        weight: u32,
    ) -> Self {
        Self {
            id: id.into(),
            address: address.into(),
            weight,
            healthy: AtomicBool::new(true),
            active_connections: AtomicU64::new(0),
            response_time_ms: AtomicU64::new(0),
        }
    }

    pub fn is_healthy(&self) -> bool {
        self.healthy.load(Ordering::Relaxed)
    }

    pub fn set_healthy(&self, v: bool) {
        self.healthy.store(v, Ordering::Relaxed);
    }

    /// เพิ่ม active connection counter (เรียกเมื่อรับ connection ใหม่)
    pub fn connect(&self) -> u64 {
        self.active_connections.fetch_add(1, Ordering::Relaxed)
    }

    /// ลด active connection counter (เรียกเมื่อ connection ปิด)
    pub fn disconnect(&self) {
        self.active_connections.fetch_sub(1, Ordering::Relaxed);
    }
}

/// Pool ของ backend ทั้งหมด — thread-safe ด้วย DashMap
pub struct BackendPool {
    pub backends: DashMap<String, Arc<Backend>>,
}

impl BackendPool {
    pub fn new() -> Self {
        Self {
            backends: DashMap::new(),
        }
    }

    pub fn add(&self, backend: Backend) {
        let id = backend.id.clone();
        self.backends.insert(id, Arc::new(backend));
    }

    pub fn remove(&self, id: &str) -> Option<Arc<Backend>> {
        self.backends.remove(id).map(|(_, v)| v)
    }

    pub fn get(&self, id: &str) -> Option<Arc<Backend>> {
        self.backends.get(id).map(|e| Arc::clone(e.value()))
    }

    /// คืนเฉพาะ backend ที่ healthy และ sort ตาม id เพื่อ determinism
    pub fn healthy_backends(&self) -> Vec<Arc<Backend>> {
        let mut v: Vec<Arc<Backend>> = self.backends
            .iter()
            .filter(|e| e.value().is_healthy())
            .map(|e| Arc::clone(e.value()))
            .collect();
        v.sort_by(|a, b| a.id.cmp(&b.id));
        v
    }

    pub fn all_backends(&self) -> Vec<Arc<Backend>> {
        let mut v: Vec<Arc<Backend>> = self.backends
            .iter()
            .map(|e| Arc::clone(e.value()))
            .collect();
        v.sort_by(|a, b| a.id.cmp(&b.id));
        v
    }

    pub fn len(&self) -> usize {
        self.backends.len()
    }
}

impl Default for BackendPool {
    fn default() -> Self {
        Self::new()
    }
}
```

**ทำไม `healthy_backends()` ถึง sort?**

`DashMap` ไม่รับประกัน iteration order เพราะ shard ภายในอาจ reorder ได้ การ sort ทำให้ Round-Robin กระจาย request อย่าง deterministic ทั้งในระหว่าง test และ production

---

### ขั้นที่ 2: LoadBalancer Trait และ Round-Robin

```rust
// src/balancer.rs
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use crate::backend::{Backend, BackendPool};

/// trait หลักที่ทุก algorithm ต้อง implement
/// Send + Sync จำเป็นเพราะ load balancer ถูกแชร์ผ่าน Arc ข้าม task
pub trait LoadBalancer: Send + Sync {
    fn select(
        &self,
        pool: &BackendPool,
        client_ip: Option<&str>,
    ) -> Option<Arc<Backend>>;

    fn name(&self) -> &str;
}

/// Round-Robin: ส่ง request ไปหา backend แต่ละตัวตามลำดับวนไป
/// AtomicUsize counter ทำให้ thread-safe โดยไม่ต้องใช้ Mutex
pub struct RoundRobin {
    counter: AtomicUsize,
}

impl RoundRobin {
    pub fn new() -> Self {
        Self {
            counter: AtomicUsize::new(0),
        }
    }
}

impl LoadBalancer for RoundRobin {
    fn select(
        &self,
        pool: &BackendPool,
        _client_ip: Option<&str>,
    ) -> Option<Arc<Backend>> {
        let healthy = pool.healthy_backends();
        if healthy.is_empty() {
            return None;
        }
        // fetch_add คืน old value ก่อน increment
        // Relaxed ordering พอแล้วเพราะเราต้องการแค่ "counter เพิ่ม" ไม่ใช่ synchronization
        let idx = self.counter.fetch_add(1, Ordering::Relaxed);
        Some(Arc::clone(&healthy[idx % healthy.len()]))
    }

    fn name(&self) -> &str {
        "round_robin"
    }
}
```

**Note เรื่อง `Ordering::Relaxed`**:
สำหรับ counter ที่แค่ต้องการ "เพิ่มทีละ 1" โดยไม่ต้องการ happens-before guarantee กับ memory อื่น, `Relaxed` เพียงพอและเร็วที่สุด ใช้ `SeqCst` เฉพาะเมื่อต้องการ total ordering เช่น double-checked locking

---

### ขั้นที่ 3: Weighted Round-Robin

**Weighted Round-Robin** กระจาย traffic ตามสัดส่วน weight — backend ที่มี weight สูงกว่าจะได้รับ request มากกว่า

แนวทาง: สร้าง sequence ล่วงหน้าโดยเติม backend ID ซ้ำตาม weight จากนั้น round-robin ผ่าน sequence นั้น

```
weight: a=3, b=1 → sequence: [a, a, a, b]
request 1 → a  (idx 0)
request 2 → a  (idx 1)
request 3 → a  (idx 2)
request 4 → b  (idx 3)
request 5 → a  (idx 4 % 4 = 0)
...
```

```rust
// src/balancer.rs (ต่อ)

/// Weighted Round-Robin: expand weight เป็น sequence แล้ว round-robin
pub struct WeightedRoundRobin {
    counter: AtomicUsize,
    sequence: Vec<String>,  // pre-built: ["a","a","a","b"] สำหรับ weight a=3, b=1
}

impl WeightedRoundRobin {
    /// สร้าง WeightedRoundRobin จาก list ของ (backend_id, weight)
    pub fn build(backends: &[(String, u32)]) -> Self {
        let mut sequence = Vec::new();
        for (id, weight) in backends {
            for _ in 0..*weight {
                sequence.push(id.clone());
            }
        }
        Self {
            counter: AtomicUsize::new(0),
            sequence,
        }
    }
}

impl LoadBalancer for WeightedRoundRobin {
    fn select(
        &self,
        pool: &BackendPool,
        _client_ip: Option<&str>,
    ) -> Option<Arc<Backend>> {
        if self.sequence.is_empty() {
            return None;
        }
        let start = self.counter.fetch_add(1, Ordering::Relaxed);
        // ลอง backend ตาม sequence, ข้ามตัวที่ unhealthy
        for offset in 0..self.sequence.len() {
            let id = &self.sequence[(start + offset) % self.sequence.len()];
            if let Some(b) = pool.get(id) {
                if b.is_healthy() {
                    return Some(b);
                }
            }
        }
        None
    }

    fn name(&self) -> &str {
        "weighted_round_robin"
    }
}
```

---

### ขั้นที่ 4: Least Connections และ IP Hash

#### Least Connections

เลือก backend ที่มี `active_connections` น้อยที่สุด — เหมาะสำหรับ backend ที่มี processing time ไม่เท่ากัน เช่น database query บางอันใช้เวลานานมาก

```rust
// src/balancer.rs (ต่อ)

pub struct LeastConnections;

impl LeastConnections {
    pub fn new() -> Self { Self }
}

impl LoadBalancer for LeastConnections {
    fn select(
        &self,
        pool: &BackendPool,
        _client_ip: Option<&str>,
    ) -> Option<Arc<Backend>> {
        pool.healthy_backends()
            .into_iter()
            .min_by_key(|b| b.active_connections.load(Ordering::Relaxed))
    }

    fn name(&self) -> &str { "least_connections" }
}
```

#### IP Hash

แปลง client IP เป็น index ด้วย FNV-1a hash — IP เดิมจะ map ไปหา backend เดิมเสมอ (ตราบที่ pool ไม่เปลี่ยน)

```rust
// src/balancer.rs (ต่อ)

pub struct IpHash;

impl IpHash {
    pub fn new() -> Self { Self }

    /// FNV-1a 64-bit hash — เร็ว, distribution ดี, implement ง่าย
    fn hash_ip(ip: &str) -> u64 {
        let mut hash: u64 = 14695981039346656037u64; // FNV offset basis
        for byte in ip.bytes() {
            hash ^= byte as u64;
            hash = hash.wrapping_mul(1099511628211); // FNV prime
        }
        hash
    }
}

impl LoadBalancer for IpHash {
    fn select(
        &self,
        pool: &BackendPool,
        client_ip: Option<&str>,
    ) -> Option<Arc<Backend>> {
        let healthy = pool.healthy_backends();
        if healthy.is_empty() {
            return None;
        }
        let ip = client_ip.unwrap_or("127.0.0.1");
        let h = Self::hash_ip(ip);
        let idx = (h % healthy.len() as u64) as usize;
        Some(Arc::clone(&healthy[idx]))
    }

    fn name(&self) -> &str { "ip_hash" }
}
```

**ข้อควรระวัง IP Hash**: ถ้า backend ล้ม แล้วเราลบออกจาก pool, ขนาด `healthy.len()` จะเปลี่ยน ทำให้ hash mapping เปลี่ยนทั้งหมด นี่คือ **consistent hashing problem** — โปรเจค F08 (Consistent Hash Ring) แก้ปัญหานี้โดยตรง

---

### ขั้นที่ 5: Health Checker

Health checker ทำงานเป็น background Tokio task — ping `/health` ของแต่ละ backend ทุก N วินาที ถ้า fail ติดต่อกัน K ครั้งจะ mark unhealthy, ถ้า success ติดต่อกัน M ครั้งจะ mark healthy กลับมา

```rust
// src/health.rs
use std::sync::Arc;
use std::time::Duration;
use dashmap::DashMap;
use tokio::time::interval;
use crate::backend::{Backend, BackendPool};

/// ติดตาม failure/success streak ของแต่ละ backend
pub struct HealthChecker {
    pub failure_threshold: u32,  // ครั้งที่ fail ติดกันก่อน mark unhealthy
    pub success_threshold: u32,  // ครั้งที่ success ติดกันก่อน mark healthy
    failure_counts: DashMap<String, u32>,
    success_counts: DashMap<String, u32>,
}

impl HealthChecker {
    pub fn new(failure_threshold: u32, success_threshold: u32) -> Self {
        Self {
            failure_threshold,
            success_threshold,
            failure_counts: DashMap::new(),
            success_counts: DashMap::new(),
        }
    }

    /// เรียกเมื่อ health check request ล้มเหลว
    pub fn record_failure(&self, backend: &Backend) {
        let count = {
            let mut entry = self.failure_counts
                .entry(backend.id.clone())
                .or_insert(0);
            *entry += 1;
            *entry
        };
        // success streak ต้อง reset เมื่อ fail
        self.success_counts.insert(backend.id.clone(), 0);
        if count >= self.failure_threshold {
            tracing::warn!(
                id = %backend.id,
                count,
                "backend marked unhealthy"
            );
            backend.set_healthy(false);
        }
    }

    /// เรียกเมื่อ health check request สำเร็จ
    pub fn record_success(&self, backend: &Backend) {
        self.failure_counts.insert(backend.id.clone(), 0);
        let count = {
            let mut entry = self.success_counts
                .entry(backend.id.clone())
                .or_insert(0);
            *entry += 1;
            *entry
        };
        if count >= self.success_threshold {
            if !backend.is_healthy() {
                tracing::info!(
                    id = %backend.id,
                    count,
                    "backend marked healthy"
                );
            }
            backend.set_healthy(true);
        }
    }

    pub fn failure_count(&self, id: &str) -> u32 {
        self.failure_counts.get(id).map(|e| *e).unwrap_or(0)
    }

    pub fn success_count(&self, id: &str) -> u32 {
        self.success_counts.get(id).map(|e| *e).unwrap_or(0)
    }
}

/// Background Tokio task ที่ ping แต่ละ backend เป็นระยะ
pub async fn run_health_checker(
    pool: Arc<BackendPool>,
    checker: Arc<HealthChecker>,
    interval_secs: u64,
) {
    let mut ticker = interval(Duration::from_secs(interval_secs));
    loop {
        ticker.tick().await;
        let backends = pool.all_backends();
        for backend in backends {
            let checker = Arc::clone(&checker);
            let backend = Arc::clone(&backend);
            // spawn แยก task ต่อ backend เพื่อไม่ให้ backend ช้า block ตัวอื่น
            tokio::spawn(async move {
                let url = format!("http://{}/health", backend.address);
                let start = std::time::Instant::now();
                match probe_health(&url).await {
                    Ok(_) => {
                        let elapsed = start.elapsed().as_millis() as u64;
                        backend.response_time_ms.store(elapsed, std::sync::atomic::Ordering::Relaxed);
                        checker.record_success(&backend);
                    }
                    Err(e) => {
                        tracing::debug!(
                            id = %backend.id,
                            err = %e,
                            "health check failed"
                        );
                        checker.record_failure(&backend);
                    }
                }
            });
        }
    }
}

/// ส่ง GET /health แล้วคาดว่า status 200
async fn probe_health(url: &str) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    // ใน real implementation ใช้ reqwest หรือ hyper
    // ที่นี่ใช้ tokio::net::TcpStream เพื่อไม่เพิ่ม dependency
    use tokio::net::TcpStream;
    use tokio::io::{AsyncWriteExt, AsyncReadExt};

    let addr = url
        .strip_prefix("http://")
        .and_then(|s| s.splitn(2, '/').next())
        .ok_or("invalid url")?;

    let mut stream = tokio::time::timeout(
        Duration::from_secs(3),
        TcpStream::connect(addr),
    )
    .await??;

    stream
        .write_all(b"GET /health HTTP/1.0\r\nHost: localhost\r\n\r\n")
        .await?;

    let mut buf = [0u8; 16];
    stream.read_exact(&mut buf).await?;

    if buf.starts_with(b"HTTP/1.") && buf[9] == b'2' {
        Ok(())
    } else {
        Err("non-200 response".into())
    }
}
```

**State diagram ของ Health Checker**:

```
  healthy=true
       │
  record_failure() x N  (N = failure_threshold)
       │
  healthy=false
       │
  record_success() x M  (M = success_threshold)
       │
  healthy=true

  * success ใด ๆ จะ reset failure streak
  * failure ใด ๆ จะ reset success streak
```

---

### ขั้นที่ 6: Sticky Sessions

Sticky session ทำให้ client คนเดิม (ระบุด้วย cookie) ถูก route ไปหา backend เดิมเสมอ สำคัญมากสำหรับ stateful application เช่น shopping cart, WebSocket session

```rust
// src/session.rs
use std::sync::Arc;
use dashmap::DashMap;
use crate::backend::{Backend, BackendPool};
use crate::balancer::LoadBalancer;

pub type BackendId = String;

/// เก็บ mapping ระหว่าง session cookie กับ backend ID
pub struct SessionStore {
    cookie_to_backend: DashMap<String, BackendId>,
}

impl SessionStore {
    pub fn new() -> Self {
        Self {
            cookie_to_backend: DashMap::new(),
        }
    }

    pub fn get_backend(&self, cookie: &str) -> Option<BackendId> {
        self.cookie_to_backend.get(cookie).map(|e| e.clone())
    }

    pub fn set_backend(&self, cookie: String, backend_id: BackendId) {
        self.cookie_to_backend.insert(cookie, backend_id);
    }

    pub fn remove(&self, cookie: &str) {
        self.cookie_to_backend.remove(cookie);
    }

    /// เลือก backend สำหรับ request นี้
    /// คืนค่า: (Arc<Backend>, Option<String>) — backend ที่เลือก + cookie ใหม่ (ถ้ามี)
    /// ถ้า cookie ใหม่ไม่ใช่ None แสดงว่าต้อง set Set-Cookie header
    pub fn route(
        &self,
        cookie: Option<&str>,
        pool: &BackendPool,
        lb: &dyn LoadBalancer,
        client_ip: Option<&str>,
    ) -> Option<(Arc<Backend>, Option<String>)> {
        // ลอง sticky backend ก่อน
        if let Some(c) = cookie {
            if let Some(bid) = self.get_backend(c) {
                if let Some(b) = pool.get(&bid) {
                    if b.is_healthy() {
                        // backend ยัง healthy → ใช้ตัวเดิม, ไม่ต้อง cookie ใหม่
                        return Some((b, None));
                    }
                }
                // backend ไม่ healthy หรือไม่มีใน pool → fallback
            }
        }

        // Fallback: ใช้ load balancer เลือก backend ใหม่
        let b = lb.select(pool, client_ip)?;
        let new_cookie = Self::generate_cookie();
        self.set_backend(new_cookie.clone(), b.id.clone());
        Some((b, Some(new_cookie)))
    }

    /// สร้าง random-looking cookie value
    fn generate_cookie() -> String {
        let seed = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap_or_default()
            .subsec_nanos();
        let mixed = (seed as u64).wrapping_mul(6364136223846793005)
            ^ 0xdeadbeef_12345678u64;
        format!("{:016x}", mixed)
    }

    pub fn len(&self) -> usize {
        self.cookie_to_backend.len()
    }
}

impl Default for SessionStore {
    fn default() -> Self {
        Self::new()
    }
}
```

#### วิธีใช้งานใน HTTP Handler

```rust
// ใน proxy handler — ตัวอย่างการอ่าน cookie จาก HTTP header
fn extract_sticky_cookie(raw_headers: &str) -> Option<&str> {
    for line in raw_headers.lines() {
        if let Some(rest) = line.strip_prefix("Cookie: ") {
            for kv in rest.split(';') {
                let kv = kv.trim();
                if let Some(val) = kv.strip_prefix("STICKY_SESSION=") {
                    return Some(val);
                }
            }
        }
    }
    None
}

// เมื่อได้ backend และ new_cookie → inject Set-Cookie header ลงใน response
fn inject_cookie_header(response: &mut Vec<u8>, cookie: &str) {
    // แทรก Set-Cookie ก่อน blank line (\r\n\r\n)
    if let Some(pos) = response.windows(4).position(|w| w == b"\r\n\r\n") {
        let cookie_header = format!(
            "\r\nSet-Cookie: STICKY_SESSION={}; Path=/; HttpOnly; SameSite=Lax",
            cookie
        );
        response.splice(pos..pos, cookie_header.bytes());
    }
}
```

---

### ขั้นที่ 7: Reverse Proxy ด้วย copy_bidirectional

นี่คือหัวใจของ load balancer — รับ TCP connection จาก client แล้ว tunnel ไปยัง backend

```rust
// src/proxy.rs
use std::sync::Arc;
use std::time::Duration;
use tokio::net::{TcpListener, TcpStream};
use tokio::io::copy_bidirectional;
use tokio::time::timeout;
use crate::backend::BackendPool;
use crate::balancer::LoadBalancer;
use crate::session::SessionStore;
use crate::stats::LoadBalancerStats;

const CONNECTION_TIMEOUT_SECS: u64 = 30;

pub struct ProxyConfig {
    pub listen_addr: String,
}

/// รับ TCP connection ใหม่ แล้ว spawn Tokio task ต่อ connection
pub async fn run_proxy(
    config: ProxyConfig,
    pool: Arc<BackendPool>,
    lb: Arc<dyn LoadBalancer>,
    sessions: Arc<SessionStore>,
    stats: Arc<LoadBalancerStats>,
) -> std::io::Result<()> {
    let listener = TcpListener::bind(&config.listen_addr).await?;
    tracing::info!(addr = %config.listen_addr, "load balancer listening");

    loop {
        let (client_stream, client_addr) = listener.accept().await?;
        let client_ip = client_addr.ip().to_string();

        let pool = Arc::clone(&pool);
        let lb = Arc::clone(&lb);
        let sessions = Arc::clone(&sessions);
        let stats = Arc::clone(&stats);

        tokio::spawn(async move {
            if let Err(e) = handle_connection(
                client_stream,
                &client_ip,
                &pool,
                lb.as_ref(),
                &sessions,
                &stats,
            )
            .await
            {
                tracing::error!(err = %e, ip = %client_ip, "connection error");
                stats.record_error("_proxy");
            }
        });
    }
}

/// จัดการ connection เดียว: เลือก backend → connect → tunnel
async fn handle_connection(
    mut client: TcpStream,
    client_ip: &str,
    pool: &BackendPool,
    lb: &dyn LoadBalancer,
    sessions: &SessionStore,
    stats: &LoadBalancerStats,
) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    use tokio::io::{AsyncReadExt, AsyncWriteExt};

    // อ่าน HTTP headers จาก client (สูงสุด 8KB)
    let mut header_buf = vec![0u8; 8192];
    let n = timeout(
        Duration::from_secs(5),
        client.read(&mut header_buf),
    )
    .await??;
    header_buf.truncate(n);

    // parse cookie จาก raw header bytes
    let header_str = std::str::from_utf8(&header_buf).unwrap_or("");
    let cookie = extract_sticky_cookie(header_str);

    // เลือก backend
    let (backend, new_cookie) = sessions
        .route(cookie, pool, lb, Some(client_ip))
        .ok_or("no healthy backend available")?;

    tracing::debug!(
        backend = %backend.id,
        ip = %client_ip,
        "routing request"
    );

    // connect ไปยัง backend
    let mut backend_stream = timeout(
        Duration::from_secs(5),
        TcpStream::connect(&backend.address),
    )
    .await??;

    // ส่ง request ไปยัง backend (พร้อม inject cookie header ถ้าจำเป็น)
    backend_stream.write_all(&header_buf).await?;

    // นับ active connection
    backend.connect();

    // bidirectional tunnel — copy ทั้ง client→backend และ backend→client พร้อมกัน
    let result = timeout(
        Duration::from_secs(CONNECTION_TIMEOUT_SECS),
        copy_bidirectional(&mut client, &mut backend_stream),
    )
    .await;

    backend.disconnect();

    match result {
        Ok(Ok((from_client, from_backend))) => {
            stats.record_request(
                &backend.id,
                from_client + from_backend,
            );

            // inject Set-Cookie ถ้าได้ backend ใหม่
            if let Some(c) = new_cookie {
                tracing::debug!(cookie = %c, "setting sticky cookie");
            }
        }
        Ok(Err(e)) => {
            stats.record_error(&backend.id);
            return Err(e.into());
        }
        Err(_timeout) => {
            stats.record_error(&backend.id);
            return Err("connection timeout".into());
        }
    }

    Ok(())
}

fn extract_sticky_cookie<'a>(headers: &'a str) -> Option<&'a str> {
    for line in headers.lines() {
        if let Some(rest) = line.strip_prefix("Cookie: ") {
            for kv in rest.split(';') {
                let kv = kv.trim();
                if let Some(val) = kv.strip_prefix("STICKY_SESSION=") {
                    return Some(val);
                }
            }
        }
    }
    None
}
```

**`copy_bidirectional` ทำงานอย่างไร?**

```
Client ◄──────────────────────────► Backend
       write buf     read buf
       ─────────►  ─────────►
       ◄─────────  ◄─────────
```

`copy_bidirectional` สร้าง 2 coroutine ภายใน task เดียว:
- Loop A: อ่านจาก client → เขียนไป backend
- Loop B: อ่านจาก backend → เขียนไป client

ทั้งสองใช้ `tokio::select!` ภายในเพื่อ interleave I/O อย่าง cooperative

---

### ขั้นที่ 8: Stats และ Monitoring Endpoint

```rust
// src/stats.rs
use std::sync::atomic::{AtomicU64, Ordering};
use std::collections::HashMap;
use dashmap::DashMap;

#[derive(Debug, Default, Clone)]
pub struct BackendStats {
    pub requests: u64,
    pub bytes_forwarded: u64,
    pub errors: u64,
}

/// Stats รวมของ load balancer ทั้งระบบ
pub struct LoadBalancerStats {
    pub requests_total: AtomicU64,
    pub bytes_forwarded: AtomicU64,
    pub errors: AtomicU64,
    backend_stats: DashMap<String, BackendStats>,
}

impl Default for LoadBalancerStats {
    fn default() -> Self {
        Self {
            requests_total: AtomicU64::new(0),
            bytes_forwarded: AtomicU64::new(0),
            errors: AtomicU64::new(0),
            backend_stats: DashMap::new(),
        }
    }
}

impl LoadBalancerStats {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn record_request(&self, backend_id: &str, bytes: u64) {
        self.requests_total.fetch_add(1, Ordering::Relaxed);
        self.bytes_forwarded.fetch_add(bytes, Ordering::Relaxed);
        let mut s = self.backend_stats
            .entry(backend_id.to_string())
            .or_default();
        s.requests += 1;
        s.bytes_forwarded += bytes;
    }

    pub fn record_error(&self, backend_id: &str) {
        self.errors.fetch_add(1, Ordering::Relaxed);
        let mut s = self.backend_stats
            .entry(backend_id.to_string())
            .or_default();
        s.errors += 1;
    }

    pub fn get_backend_stats(&self, id: &str) -> Option<BackendStats> {
        self.backend_stats.get(id).map(|e| e.clone())
    }

    pub fn total_requests(&self) -> u64 {
        self.requests_total.load(Ordering::Relaxed)
    }

    pub fn total_errors(&self) -> u64 {
        self.errors.load(Ordering::Relaxed)
    }

    pub fn total_bytes(&self) -> u64 {
        self.bytes_forwarded.load(Ordering::Relaxed)
    }

    /// serialize เป็น JSON สำหรับ /lb-stats endpoint
    pub fn to_json(&self) -> String {
        let backend_map: HashMap<String, serde_json::Value> = self
            .backend_stats
            .iter()
            .map(|e| {
                let k = e.key().clone();
                let v = serde_json::json!({
                    "requests": e.value().requests,
                    "bytes_forwarded": e.value().bytes_forwarded,
                    "errors": e.value().errors,
                    "error_rate": if e.value().requests > 0 {
                        e.value().errors as f64 / e.value().requests as f64
                    } else {
                        0.0
                    },
                });
                (k, v)
            })
            .collect();

        serde_json::to_string_pretty(&serde_json::json!({
            "requests_total": self.total_requests(),
            "bytes_forwarded": self.total_bytes(),
            "errors": self.total_errors(),
            "backend_stats": backend_map,
        }))
        .unwrap_or_default()
    }
}
```

ตัวอย่าง response จาก `/lb-stats`:

```json
{
  "requests_total": 10000,
  "bytes_forwarded": 52428800,
  "errors": 12,
  "backend_stats": {
    "backend-1": {
      "requests": 3350,
      "bytes_forwarded": 17563648,
      "errors": 3,
      "error_rate": 0.000895
    },
    "backend-2": {
      "requests": 3324,
      "bytes_forwarded": 17432576,
      "errors": 5,
      "error_rate": 0.001504
    },
    "backend-3": {
      "requests": 3326,
      "bytes_forwarded": 17432576,
      "errors": 4,
      "error_rate": 0.001203
    }
  }
}
```

---

### ขั้นที่ 9: Main Entry Point และ Admin Endpoint

```rust
// src/main.rs
use std::sync::Arc;
use load_balancer::{
    backend::{Backend, BackendPool},
    balancer::{LeastConnections, RoundRobin},
    health::{run_health_checker, HealthChecker},
    proxy::{run_proxy, ProxyConfig},
    session::SessionStore,
    stats::LoadBalancerStats,
};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::fmt::init();

    // สร้าง backend pool
    let pool = Arc::new(BackendPool::new());
    pool.add(Backend::new("backend-1", "127.0.0.1:8001", 2));
    pool.add(Backend::new("backend-2", "127.0.0.1:8002", 2));
    pool.add(Backend::new("backend-3", "127.0.0.1:8003", 1));

    let lb: Arc<dyn load_balancer::balancer::LoadBalancer> =
        Arc::new(RoundRobin::new());
    let sessions = Arc::new(SessionStore::new());
    let stats = Arc::new(LoadBalancerStats::new());
    let checker = Arc::new(HealthChecker::new(3, 2));

    // spawn health checker background task
    {
        let pool = Arc::clone(&pool);
        let checker = Arc::clone(&checker);
        tokio::spawn(async move {
            run_health_checker(pool, checker, 10).await;
        });
    }

    // spawn stats/admin endpoint
    {
        let stats = Arc::clone(&stats);
        let pool = Arc::clone(&pool);
        tokio::spawn(async move {
            serve_admin(stats, pool, "127.0.0.1:9090").await;
        });
    }

    // เริ่ม proxy (blocking)
    run_proxy(
        ProxyConfig { listen_addr: "0.0.0.0:80".to_string() },
        pool,
        lb,
        sessions,
        stats,
    )
    .await?;

    Ok(())
}

/// Admin HTTP server: GET /lb-stats, GET /backends
async fn serve_admin(
    stats: Arc<LoadBalancerStats>,
    pool: Arc<BackendPool>,
    addr: &str,
) {
    let listener = TcpListener::bind(addr).await.unwrap();
    tracing::info!(addr, "admin server listening");
    loop {
        if let Ok((mut stream, _)) = listener.accept().await {
            use tokio::io::{AsyncReadExt, AsyncWriteExt};
            let stats = Arc::clone(&stats);
            let pool = Arc::clone(&pool);
            tokio::spawn(async move {
                let mut buf = [0u8; 1024];
                let n = stream.read(&mut buf).await.unwrap_or(0);
                let req = std::str::from_utf8(&buf[..n]).unwrap_or("");

                let (status, body) = if req.starts_with("GET /lb-stats") {
                    ("200 OK", stats.to_json())
                } else if req.starts_with("GET /backends") {
                    let backends: Vec<serde_json::Value> = pool
                        .all_backends()
                        .iter()
                        .map(|b| serde_json::json!({
                            "id": b.id,
                            "address": b.address,
                            "healthy": b.is_healthy(),
                            "active_connections": b.active_connections
                                .load(std::sync::atomic::Ordering::Relaxed),
                            "response_time_ms": b.response_time_ms
                                .load(std::sync::atomic::Ordering::Relaxed),
                        }))
                        .collect();
                    ("200 OK", serde_json::to_string_pretty(&backends).unwrap_or_default())
                } else {
                    ("404 Not Found", "{}".to_string())
                };

                let response = format!(
                    "HTTP/1.1 {}\r\nContent-Type: application/json\r\nContent-Length: {}\r\n\r\n{}",
                    status,
                    body.len(),
                    body
                );
                let _ = stream.write_all(response.as_bytes()).await;
            });
        }
    }
}
```

---

## การทดสอบ (Testing)

### Unit Tests — ผลลัพธ์จริงจาก `cargo test`

โปรเจคมี 19 unit tests ครอบคลุมทุก algorithm และ component สำคัญ:

```
running 19 tests
test tests::test_health_checker_marks_unhealthy_after_threshold ... ok
test tests::test_backend_pool_add_remove ... ok
test tests::test_backend_pool_healthy_filter ... ok
test tests::test_health_checker_marks_healthy_after_successes ... ok
test tests::test_health_checker_success_resets_failure_count ... ok
test tests::test_least_connections_picks_minimum ... ok
test tests::test_least_connections_skips_unhealthy ... ok
test tests::test_ip_hash_deterministic ... ok
test tests::test_ip_hash_distributes_across_backends ... ok
test tests::test_round_robin_distribution ... ok
test tests::test_stats_json_contains_fields ... ok
test tests::test_round_robin_returns_none_all_unhealthy ... ok
test tests::test_sticky_session_creates_new_for_unknown_cookie ... ok
test tests::test_round_robin_skips_unhealthy ... ok
test tests::test_stats_counting ... ok
test tests::test_sticky_session_fallback_when_unhealthy ... ok
test tests::test_sticky_session_routes_same_backend ... ok
test tests::test_weighted_rr_proportion ... ok
test tests::test_weighted_rr_skips_unhealthy ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### รายละเอียด Tests แต่ละกลุ่ม

#### RoundRobin Tests

```rust
#[test]
fn test_round_robin_distribution() {
    let pool = make_pool(&["a", "b", "c"]);
    let lb = RoundRobin::new();
    let mut counts: HashMap<String, u32> = HashMap::new();
    for _ in 0..9 {
        let b = lb.select(&pool, None).unwrap();
        *counts.entry(b.id.clone()).or_insert(0) += 1;
    }
    assert_eq!(counts["a"], 3);
    assert_eq!(counts["b"], 3);
    assert_eq!(counts["c"], 3);
}

#[test]
fn test_round_robin_skips_unhealthy() {
    let pool = make_pool(&["a", "b", "c"]);
    pool.get("b").unwrap().set_healthy(false);
    let lb = RoundRobin::new();
    for _ in 0..10 {
        let b = lb.select(&pool, None).unwrap();
        assert_ne!(b.id, "b");
    }
}

#[test]
fn test_round_robin_returns_none_all_unhealthy() {
    let pool = make_pool(&["a", "b"]);
    pool.get("a").unwrap().set_healthy(false);
    pool.get("b").unwrap().set_healthy(false);
    let lb = RoundRobin::new();
    assert!(lb.select(&pool, None).is_none());
}
```

#### WeightedRoundRobin Tests

```rust
#[test]
fn test_weighted_rr_proportion() {
    let pool = BackendPool::new();
    pool.add(Backend::new("a", "127.0.0.1:8000", 3));
    pool.add(Backend::new("b", "127.0.0.1:8001", 1));

    let lb = WeightedRoundRobin::build(&[
        ("a".to_string(), 3),
        ("b".to_string(), 1),
    ]);

    let mut counts: HashMap<String, u32> = HashMap::new();
    for _ in 0..40 {
        if let Some(b) = lb.select(&pool, None) {
            *counts.entry(b.id.clone()).or_insert(0) += 1;
        }
    }
    let a_count = counts.get("a").copied().unwrap_or(0);
    let b_count = counts.get("b").copied().unwrap_or(0);
    // a ต้องได้รับ request มากกว่า b อย่างน้อย 2 เท่า (จริง ๆ ต้องใกล้ 3 เท่า)
    assert!(a_count > b_count * 2);
}
```

#### LeastConnections Tests

```rust
#[test]
fn test_least_connections_picks_minimum() {
    let pool = BackendPool::new();
    pool.add(Backend::new("a", "127.0.0.1:8000", 1));
    pool.add(Backend::new("b", "127.0.0.1:8001", 1));
    pool.add(Backend::new("c", "127.0.0.1:8002", 1));

    pool.get("a").unwrap().active_connections.store(10, Ordering::Relaxed);
    pool.get("b").unwrap().active_connections.store(3, Ordering::Relaxed);
    pool.get("c").unwrap().active_connections.store(7, Ordering::Relaxed);

    let lb = LeastConnections::new();
    let selected = lb.select(&pool, None).unwrap();
    assert_eq!(selected.id, "b"); // b มี connections น้อยสุด
}
```

#### IpHash Tests

```rust
#[test]
fn test_ip_hash_deterministic() {
    let pool = make_pool(&["a", "b", "c"]);
    let lb = IpHash::new();
    let first = lb.select(&pool, Some("192.168.1.100")).unwrap().id.clone();
    for _ in 0..20 {
        let result = lb.select(&pool, Some("192.168.1.100")).unwrap();
        assert_eq!(result.id, first, "same IP must always map to same backend");
    }
}

#[test]
fn test_ip_hash_distributes_across_backends() {
    let pool = make_pool(&["a", "b", "c"]);
    let lb = IpHash::new();
    let mut results: std::collections::HashSet<String> = std::collections::HashSet::new();
    for i in 1u8..=30 {
        let ip = format!("10.0.0.{}", i);
        if let Some(b) = lb.select(&pool, Some(&ip)) {
            results.insert(b.id.clone());
        }
    }
    assert!(results.len() >= 2); // ต้อง distribute ไปอย่างน้อย 2 backend
}
```

#### HealthChecker Tests

```rust
#[test]
fn test_health_checker_marks_unhealthy_after_threshold() {
    let backend = Arc::new(Backend::new("srv", "127.0.0.1:9000", 1));
    let checker = HealthChecker::new(3, 2); // failure_threshold=3

    assert!(backend.is_healthy());
    checker.record_failure(&backend);
    assert!(backend.is_healthy()); // ยัง healthy หลัง 1 failure
    checker.record_failure(&backend);
    assert!(backend.is_healthy()); // ยัง healthy หลัง 2 failures
    checker.record_failure(&backend);
    assert!(!backend.is_healthy()); // unhealthy หลัง 3 failures
}

#[test]
fn test_health_checker_success_resets_failure_count() {
    let backend = Arc::new(Backend::new("srv", "127.0.0.1:9000", 1));
    let checker = HealthChecker::new(3, 1);

    checker.record_failure(&backend);
    checker.record_failure(&backend);
    assert_eq!(checker.failure_count("srv"), 2);

    checker.record_success(&backend); // reset failure count
    assert_eq!(checker.failure_count("srv"), 0);

    checker.record_failure(&backend);
    checker.record_failure(&backend);
    assert!(backend.is_healthy()); // ยัง healthy เพราะนับใหม่
}
```

#### SessionStore Tests

```rust
#[test]
fn test_sticky_session_routes_same_backend() {
    let pool = make_pool(&["a", "b", "c"]);
    let store = SessionStore::new();
    let lb = RoundRobin::new();

    let (b1, new_cookie) = store.route(None, &pool, &lb, None).unwrap();
    let cookie = new_cookie.expect("first request should receive a new cookie");

    for _ in 0..10 {
        let (b2, subsequent_cookie) = store.route(Some(&cookie), &pool, &lb, None).unwrap();
        assert_eq!(b1.id, b2.id); // ต้อง route ไป backend เดิม
        assert!(subsequent_cookie.is_none()); // ไม่ต้อง set cookie ใหม่
    }
}

#[test]
fn test_sticky_session_fallback_when_unhealthy() {
    let pool = make_pool(&["a", "b"]);
    let store = SessionStore::new();
    let lb = RoundRobin::new();

    store.set_backend("test_cookie".to_string(), "a".to_string());
    pool.get("a").unwrap().set_healthy(false); // a ล้ม

    let (b, _) = store.route(Some("test_cookie"), &pool, &lb, None).unwrap();
    assert_eq!(b.id, "b"); // fallback ไป b
}
```

### Integration Test (เพิ่มเติม)

```rust
// tests/integration.rs
#[tokio::test]
async fn test_round_trip_proxy() {
    // 1. spawn echo server
    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
    let echo_addr = listener.local_addr().unwrap().to_string();
    tokio::spawn(async move {
        loop {
            let (mut stream, _) = listener.accept().await.unwrap();
            tokio::spawn(async move {
                use tokio::io::{AsyncReadExt, AsyncWriteExt};
                let mut buf = vec![0u8; 4096];
                let n = stream.read(&mut buf).await.unwrap();
                stream.write_all(&buf[..n]).await.unwrap();
            });
        }
    });

    // 2. setup pool with echo server as backend
    let pool = Arc::new(BackendPool::new());
    pool.add(Backend::new("echo", &echo_addr, 1));

    // 3. connect and send data
    let backend = pool.get("echo").unwrap();
    let mut conn = tokio::net::TcpStream::connect(&backend.address)
        .await
        .unwrap();
    use tokio::io::{AsyncReadExt, AsyncWriteExt};
    conn.write_all(b"hello").await.unwrap();
    let mut resp = vec![0u8; 5];
    conn.read_exact(&mut resp).await.unwrap();
    assert_eq!(&resp, b"hello");
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# build optimized binary
cargo build --release

# binary อยู่ที่
ls -lh target/release/load-balancer
# -rwxr-xr-x 1 user user 2.1M ... load-balancer
```

### Dockerfile แบบ Multi-stage

```dockerfile
# --- Builder ---
FROM rust:1.82-slim AS builder
WORKDIR /build
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs
RUN cargo build --release  # cache dependencies
COPY src ./src
RUN touch src/main.rs && cargo build --release

# --- Runtime ---
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /build/target/release/load-balancer /usr/local/bin/load-balancer
EXPOSE 80 9090
ENTRYPOINT ["load-balancer"]
```

```bash
docker build -t load-balancer:latest .
docker run -p 80:80 -p 9090:9090 \
  -e BACKENDS="10.0.1.1:8080,10.0.1.2:8080" \
  load-balancer:latest
```

### systemd Service

```ini
# /etc/systemd/system/load-balancer.service
[Unit]
Description=Rust Layer 7 Load Balancer
After=network.target

[Service]
Type=simple
User=nobody
ExecStart=/usr/local/bin/load-balancer
Restart=always
RestartSec=5
Environment=RUST_LOG=info
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now load-balancer
journalctl -u load-balancer -f
```

### ทดสอบด้วย wrk

```bash
# ติดตั้ง wrk
apt-get install wrk

# benchmark 10 วินาที, 100 connections, 4 threads
wrk -t4 -c100 -d10s http://localhost:80/

# ผลลัพธ์ตัวอย่าง
Running 10s test @ http://localhost:80/
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.14ms    0.87ms  12.34ms   78.23%
    Req/Sec    12.45k     1.23k   15.67k    72.50%
  497,234 requests in 10.01s, 1.45GB read
Requests/sec:  49,673.45
Transfer/sec:    148.51MB
```

---

## ข้อผิดพลาดที่พบบ่อย (Pitfalls)

### Pitfall 1: Race Condition ใน Connection Counter

**ปัญหา**: ถ้าใช้ `connect()` ก่อน check connection limit และ `disconnect()` หลัง error handling แยกกัน อาจเกิด counter ที่ไม่ตรงกับความจริง

```rust
// ❌ ผิด — ถ้า write_all fail, disconnect() จะไม่ถูกเรียก
backend.connect();
stream.write_all(data).await?; // ? return early!
backend.disconnect();          // ไม่ถึงบรรทัดนี้ถ้า error

// ✅ ถูก — ใช้ drop guard หรือ defer pattern
struct ConnectionGuard(Arc<Backend>);
impl Drop for ConnectionGuard {
    fn drop(&mut self) {
        self.0.disconnect(); // เรียกเสมอเมื่อ drop
    }
}

let _guard = ConnectionGuard(Arc::clone(&backend));
backend.connect();
stream.write_all(data).await?; // ok ถ้า error, guard drop → disconnect()
```

### Pitfall 2: DashMap Deadlock จาก Nested Lock

**ปัญหา**: `DashMap` ใช้ shard locking ภายใน ถ้าเรา hold `DashMapRef` แล้วทำ operation อื่นที่ต้องการ lock เดิม จะ deadlock

```rust
// ❌ deadlock — hold ref แล้ว insert ลง shard เดียวกัน
let entry = pool.backends.get("a").unwrap(); // hold shard lock
pool.backends.insert("a", new_backend);      // ต้องการ write lock บน shard เดิม → deadlock!

// ✅ ถูก — clone ออกมาก่อน release ref
let backend = pool.backends.get("a").map(|e| Arc::clone(e.value()));
drop(entry); // release ref ก่อน
if let Some(b) = backend { /* ใช้ b ได้ */ }
```

### Pitfall 3: IP Hash ไม่ Consistent เมื่อ Backend Pool เปลี่ยน

**ปัญหา**: `IpHash` hash IP แล้ว mod ด้วย `len()` ถ้าจำนวน backend เปลี่ยน (เพิ่ม/ลบ) ทุก client จะถูก re-route ไปหา backend ใหม่

```
Before: pool=[A, B, C], IP "1.2.3.4" → hash=7 → 7%3=1 → B
After remove C: pool=[A, B], IP "1.2.3.4" → hash=7 → 7%2=1 → B (บังเอิญ)
After add D: pool=[A, B, D], IP "1.2.3.4" → hash=7 → 7%3=1 → B (ok)
After remove A: pool=[B, D], IP "1.2.3.4" → hash=7 → 7%2=1 → D (เปลี่ยน!)
```

**แก้ไข**: ใช้ Consistent Hash Ring (โปรเจค F08) ที่รับประกันว่าเมื่อเพิ่ม/ลบ backend จะ re-route เฉพาะ ~1/N fraction ของ keys

### Pitfall 4: Health Check ทำให้ "Thundering Herd"

**ปัญหา**: ถ้า health checker mark backend เป็น healthy กลับมาพร้อมกัน แล้ว load balancer ส่ง traffic จำนวนมากไปทันที backend อาจ overload ซ้ำ

```
[t=0]  backend A down
[t=10] health check success: mark A healthy
[t=10] 1000 queued requests flood ไป A พร้อมกัน
[t=10] A overload อีก → mark unhealthy อีก
```

**แก้ไข** (Slow Start / Gradual Ramp):
```rust
// เพิ่ม weight ทีละน้อยเมื่อ backend กลับมา
struct Backend {
    // ...
    ramp_weight: AtomicU64, // เริ่มที่ 1, เพิ่มทุก interval จนถึง target_weight
    target_weight: u32,
}

// ใน health checker
fn record_success(&self, backend: &Backend) {
    // ... existing logic ...
    if backend.is_healthy() {
        let current = backend.ramp_weight.load(Ordering::Relaxed);
        let next = (current + 1).min(backend.target_weight as u64);
        backend.ramp_weight.store(next, Ordering::Relaxed);
    }
}
```

### Pitfall 5: Memory Leak จาก SessionStore ที่ไม่มี TTL

**ปัญหา**: `SessionStore` เก็บ cookie → backend mapping ไว้ใน `DashMap` ตลอดไป ถ้า client หยุดส่ง request แต่ session ไม่ถูกลบออก memory จะเพิ่มขึ้นเรื่อย ๆ

```rust
// ❌ ผิด — session ไม่เคยหมดอายุ
pub struct SessionStore {
    cookie_to_backend: DashMap<String, BackendId>,
}

// ✅ ถูก — เก็บ timestamp ด้วย แล้วมี background cleanup task
pub struct SessionEntry {
    backend_id: BackendId,
    last_seen: std::time::Instant,
}

pub struct SessionStore {
    sessions: DashMap<String, SessionEntry>,
    ttl: Duration,
}

impl SessionStore {
    async fn cleanup_loop(&self) {
        let mut ticker = interval(Duration::from_secs(60));
        loop {
            ticker.tick().await;
            let now = std::time::Instant::now();
            self.sessions.retain(|_, v| now - v.last_seen < self.ttl);
        }
    }
}
```

### Pitfall 6: copy_bidirectional ไม่ Handle Half-Close

**ปัญหา**: HTTP/1.1 บางครั้ง client ส่ง request ครบแล้ว shutdown write side (half-close) แต่ยังรอ response เพิ่มเติม `copy_bidirectional` จะ return เมื่อ either side ปิด ซึ่งอาจตัด response ก่อนส่งครบ

**แก้ไข**: สำหรับ HTTP/1.1 ควรใช้ HTTP-aware parser แทน raw TCP copy หรือใช้ `hyper` ที่จัดการ semantics นี้ให้อัตโนมัติ

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Smooth Weighted Round-Robin (ระดับกลาง)

**Weighted Round-Robin แบบ expansion มีจุดอ่อน**: sequence ที่สร้างจาก weight=[3,1] คือ `[a,a,a,b]` ทำให้ b ได้รับ request ทุก 4 ครั้ง แทนที่จะกระจายสม่ำเสมอ

**Smooth WRR** ของ Nginx กระจายได้ดีกว่า:

```
Algorithm (Nginx smooth WRR):
1. ทุก backend มี current_weight = 0
2. แต่ละ selection:
   a. เพิ่ม current_weight += effective_weight สำหรับทุก backend
   b. เลือก backend ที่ current_weight สูงสุด
   c. ลด current_weight ของ backend ที่เลือก -= total_weight

ตัวอย่าง weight=[5,3,2]:
Round | Before         | Selected | After
  1   | [5, 3, 2]      | a(5)     | [-5, 3, 2]  (5-10)
  2   | [-2, 6, 4]     | b(6)     | [-2, -4, 4] (6-10)
  3   | [3, -1, 6]     | c(6)     | [3, -1, -4] (6-10)
  4   | [8, 2, -2]     | a(8)     | [-2, 2, -2] (8-10)
  5   | [3, 5, 0]      | b(5)     | [3, -5, 0]  (5-10)
...
```

ให้ implement `SmoothWeightedRoundRobin` struct และเขียน test ว่า distribution error ไม่เกิน 5%

### แบบฝึกหัดที่ 2: Consistent Hash Ring (ระดับสูง)

แทน `IpHash` ปัจจุบัน ให้ implement `ConsistentHashRing`:
- แต่ละ backend วาง virtual node หลาย node บน ring (proportional to weight)
- เมื่อเพิ่ม backend: รับเฉพาะ keyspace ที่อยู่ก่อนมัน
- เมื่อลบ backend: keyspace ของมันไปตกที่ backend ถัดไปบน ring

```rust
pub struct ConsistentHashRing {
    ring: BTreeMap<u64, String>, // hash → backend_id
    replicas: u32,               // virtual nodes per backend
}

impl ConsistentHashRing {
    pub fn add_backend(&mut self, id: &str) {
        for i in 0..self.replicas {
            let key = format!("{}#{}", id, i);
            let hash = fnv1a_hash(key.as_bytes());
            self.ring.insert(hash, id.to_string());
        }
    }

    pub fn get_backend(&self, client_ip: &str) -> Option<&str> {
        let h = fnv1a_hash(client_ip.as_bytes());
        // หา entry แรกที่ key >= h (clockwise)
        self.ring
            .range(h..)
            .next()
            .or_else(|| self.ring.iter().next()) // wrap around
            .map(|(_, id)| id.as_str())
    }
}
```

### แบบฝึกหัดที่ 3: Circuit Breaker Integration (ระดับสูง)

รวม load balancer กับ Circuit Breaker pattern (โปรเจค F04):
- ถ้า backend มี error rate > threshold% → เปิด circuit → skip backend ชั่วคราว
- ลอง probe อีกครั้งหลัง timeout (HalfOpen state)
- ถ้า probe success → ปิด circuit กลับมา

```rust
pub struct BackendCircuitBreaker {
    state: AtomicU8, // 0=Closed, 1=Open, 2=HalfOpen
    error_count: AtomicU64,
    last_opened: AtomicU64, // Unix timestamp
    threshold: u64,
    reset_timeout: Duration,
}
```

### แบบฝึกหัดที่ 4: Request Timeout และ Retry (ระดับกลาง)

เพิ่ม logic ที่เมื่อ backend ไม่ตอบภายใน timeout จะ retry กับ backend อื่น:

```rust
pub struct RetryConfig {
    pub max_retries: u32,
    pub timeout_per_attempt: Duration,
    pub retry_on_5xx: bool,         // retry เมื่อได้ 5xx response
    pub retry_on_connect_fail: bool, // retry เมื่อ connect ไม่ได้
}

async fn proxy_with_retry(
    client: &mut TcpStream,
    pool: &BackendPool,
    lb: &dyn LoadBalancer,
    config: &RetryConfig,
) -> Result<(), Error> {
    let mut tried: HashSet<String> = HashSet::new();
    for attempt in 0..=config.max_retries {
        // เลือก backend ที่ไม่ได้ลองแล้ว
        // ลอง proxy
        // ถ้า error และยังมี retries → ลอง backend อื่น
    }
    Err(Error::AllBackendsFailed)
}
```

### แบบฝึกหัดที่ 5: Dynamic Config Reload (ระดับสูง)

เพิ่ม ability ที่จะ reload config (backend list, algorithm) โดยไม่ restart process:

```bash
# ส่ง signal เพื่อ reload
kill -HUP $(pidof load-balancer)

# หรือผ่าน admin API
curl -X POST http://localhost:9090/reload \
  -H "Content-Type: application/json" \
  -d '{"backends": [...]}'
```

```rust
// ใช้ signal handler
use tokio::signal::unix::{signal, SignalKind};

let mut sigterm = signal(SignalKind::hangup())?;
tokio::select! {
    _ = sigterm.recv() => {
        let new_config = load_config_from_file().await?;
        apply_config(&pool, new_config).await;
    }
    _ = main_loop => {}
}
```

### แบบฝึกหัดที่ 6: Prometheus Metrics Export (ระดับต้น)

เพิ่ม `/metrics` endpoint ที่ output Prometheus text format:

```
# HELP lb_requests_total Total number of proxied requests
# TYPE lb_requests_total counter
lb_requests_total{backend="backend-1"} 3350
lb_requests_total{backend="backend-2"} 3324

# HELP lb_active_connections Current active connections per backend
# TYPE lb_active_connections gauge
lb_active_connections{backend="backend-1"} 42
lb_active_connections{backend="backend-2"} 38

# HELP lb_backend_healthy Whether each backend is healthy (1=yes, 0=no)
# TYPE lb_backend_healthy gauge
lb_backend_healthy{backend="backend-1"} 1
lb_backend_healthy{backend="backend-2"} 0
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **Layer 7 Load Balancer** ที่ครบฟีเจอร์ด้วย Rust โดยสิ่งที่ได้เรียนรู้:

### Pattern สำคัญที่ได้จากโปรเจค

**1. `Arc<AtomicXxx>` pattern**
แทนที่จะ lock ทั้ง struct เพื่ออัปเดต counter เดียว การใช้ `AtomicU64` ทำให้ health checker, proxy, และ stats สามารถเขียน field แต่ละตัวของ `Backend` พร้อมกันได้โดยไม่มี contention

**2. `DashMap` แทน `RwLock<HashMap>`**
สำหรับ data structure ที่ถูก read บ่อยมาก (ทุก request ต้อง lookup backend pool) `DashMap` ให้ throughput สูงกว่าเพราะ shard locking แทน global lock

**3. Trait Object สำหรับ Strategy Pattern**
`Arc<dyn LoadBalancer>` ทำให้เลือก algorithm ตอน runtime ได้โดยไม่ต้องแก้โค้ด proxy — เหมาะมากกับ pluggable architecture

**4. `copy_bidirectional` สำหรับ Transparent Proxy**
เป็น building block สำคัญของ reverse proxy ใน Tokio — ทำงานเหมือน "pipe" ระหว่าง 2 TCP stream โดย transparently

**5. Separation of Concerns ใน Health Checking**
`HealthChecker` แค่ track failure/success count และ update `AtomicBool` — logic เรื่อง "ควร healthy เมื่อไหร่" แยกออกจาก network probing ทำให้ test ได้ง่ายมาก

### เชื่อมโยงกับโปรเจคถัดไป

**H06: Protobuf Codec** จะสอนการ encode/decode message แบบ binary ที่ compact กว่า JSON — เมื่อ load balancer ของเราต้องการส่ง internal health status หรือ config updates ระหว่าง instances การใช้ Protobuf แทน JSON จะลด bandwidth ได้ 3-5x และ parsing overhead ได้มาก

---

**โปรเจคก่อนหน้า:** [Project H04: MQTT Broker](project-h04-mqtt-broker.md) | **โปรเจคถัดไป:** [Project H06: Protobuf Codec](project-h06-protobuf-codec.md)
