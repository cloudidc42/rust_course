# Project F04: Circuit Breaker Library

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

**Circuit Breaker** เป็น pattern ที่ขาดไม่ได้ในระบบ distributed ทุกระบบที่ต้องเรียก external service เช่น database, HTTP API, หรือ message broker ปัญหาหลักของระบบแบบ microservices คือเมื่อ service ปลายทางเริ่ม slow หรือ down ระบบต้นทางจะกองคิว request ไว้จำนวนมาก ทำให้ใช้ memory สูงขึ้นเรื่อย ๆ และในที่สุดก็พังตามไปด้วย (cascading failure)

Circuit Breaker แก้ปัญหานี้โดยทำงานเหมือน "สะพานไฟ" ในบ้าน — ถ้าไฟฟ้าลัดวงจร สะพานไฟจะตัดวงจรอัตโนมัติเพื่อป้องกันความเสียหายมากกว่านี้ ในโลกซอฟต์แวร์ Circuit Breaker จะ:

- **ตรวจนับ failure** ใน sliding window — ถ้าเกิน threshold จะ **"เปิดวงจร"** (Open state) และ reject request ทันทีโดยไม่ต้องรอ timeout
- **รอระยะเวลาหนึ่ง** แล้วลองส่ง probe request เพื่อตรวจสอบว่า service ฟื้นหรือยัง (HalfOpen state)
- **ปิดวงจรกลับ** (Closed) เมื่อ probe สำเร็จตามเกณฑ์

ในโปรเจคนี้เราจะสร้าง Circuit Breaker library ระดับ production ด้วย Rust โดย feature หลัก ๆ ได้แก่:

- **State machine** ที่ thread-safe ด้วย `parking_lot::Mutex`
- **Sliding window** แบบ ring buffer สำหรับ failure tracking ที่ O(1)
- **กลยุทธ์ตรวจจับความล้มเหลว** สองแบบ: count-based และ rate-based ผ่าน `FailureDetector` trait
- **Async integration** ผ่าน `CircuitBreaker::call(async_fn)` พร้อม timeout wrapping
- **Metrics** แบบ Prometheus-compatible text format

**Use case จริงใน production:** Netflix Hystrix (Java), resilience4j (Java), polly (.NET) ล้วนใช้ pattern นี้ Rust crate ที่มี production usage จริงเช่น `failsafe-rs`, `circuit-breaker-rs` ล้วนมีโครงสร้างคล้ายกัน การสร้างเองทำให้เข้าใจ trade-off ของ lock granularity, sliding window design, และการใช้ async traits

---

## สิ่งที่จะได้เรียนรู้

- **State machine design** ใน Rust — encode states เป็น enum ที่มีข้อมูลใน variant
- **`parking_lot::Mutex`** ทำไมเร็วกว่า `std::sync::Mutex` และเหมาะกับ high-contention workloads
- **Ring buffer (circular buffer)** — implement sliding window ด้วย `Vec` + head pointer, O(1) insert + eviction
- **`trait` object ที่ Send + Sync** — `Arc<dyn FailureDetector>` เพื่อ swap strategy ตอน runtime
- **`tokio::time::timeout`** wrapping async functions เพื่อป้องกัน calls ที่ block นานเกินไป
- **Generic async closures** — `FnOnce() -> Fut` pattern สำหรับรับ async function เป็น argument
- **Prometheus text format** — เขียน metrics exporter แบบ hand-rolled โดยไม่ต้องพึ่ง crate หนัก
- **Property-based thinking** — test state transitions อย่างครบถ้วนด้วย tokio test runtime

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 41-50** — Async/await, tokio runtime, `Future` trait, `async fn`
- **Part 51-60** — Trait objects (`dyn Trait`), `Arc<T>`, interior mutability
- **Part 61-70** — `Mutex`, `RwLock`, lock poisoning, thread safety
- **Part 71-80** — Closures, `FnOnce`/`FnMut`/`Fn`, higher-order functions
- **Part 96-110** — Production patterns: error types, `thiserror`, metrics, structured logging
- โปรเจค F03 (Distributed Lock) — ความเข้าใจ concurrency primitives และ distributed state

---

## โครงสร้างโปรเจค (Project Layout)

```
circuit-breaker/
├── src/
│   ├── lib.rs           # Public API: re-exports ทุก module
│   ├── state.rs         # CircuitState enum (Closed/Open/HalfOpen)
│   ├── window.rs        # SlidingWindow แบบ ring buffer + Outcome enum
│   ├── detector.rs      # FailureDetector trait + Count/Rate implementations
│   ├── metrics.rs       # CircuitMetrics struct + Prometheus text output
│   ├── breaker.rs       # CircuitBreaker<S> — core logic, call(), state machine
│   └── main.rs          # Demo binary
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### State Machine Diagram

```
                  ┌─────────────────────────┐
                  │                         │
           failures >= threshold            │
                  │                         │
                  ▼                         │
            ┌──────────┐   timeout elapsed  │
  OK ──────▶│  Closed  │◀──────────────────┤
            └──────────┘                    │
                  │                    probe success
           failures >= threshold            │
                  │                    (N probes OK)
                  ▼                         │
             ┌────────┐                     │
     reject  │  Open  │──── timeout ───▶ ┌──────────┐
   ◀──────── │        │                  │ HalfOpen │
             └────────┘                  └──────────┘
                                              │
                                         probe fails
                                              │
                                              ▼
                                         back to Open
```

### Lock Strategy

CircuitBreaker ใช้ `parking_lot::Mutex<BreakerInner>` ห่อ state ทั้งหมดไว้ใน `Arc` เดียว ทำให้ clone CircuitBreaker ได้ฟรี (เพียง clone `Arc`) และทุก thread แชร์ state เดียวกัน

```
CircuitBreaker (cheap to clone)
  └── Arc<Mutex<BreakerInner>>
        ├── CircuitState  (current state + metadata)
        ├── SlidingWindow (ring buffer)
        └── CircuitMetrics (counters)
```

เหตุผลที่เลือก `parking_lot::Mutex` แทน `std::sync::Mutex`:
1. **ไม่มี lock poisoning** — `parking_lot::Mutex` ไม่ poison เมื่อ thread panic ใน critical section ทำให้ไม่ต้องเรียก `.unwrap()` ทุกครั้ง
2. **เร็วกว่าในกรณี uncontended** — ใช้ atomic operations ก่อนถึง OS futex
3. **API สะอาดกว่า** — `lock()` คืน `MutexGuard` โดยตรง ไม่ wrap ใน `Result`

### Sliding Window Design

```
capacity = 5, head = 3

index: [0]  [1]  [2]  [3]  [4]
data:   S    F    S   (new) F
                      ▲
                    head (จะเขียนที่นี่ต่อไป)

เมื่อ record(Failure):
1. อ่านค่าเดิมที่ index[3] = None → ไม่ต้อง evict
2. เขียน Failure ที่ index[3]
3. failures++
4. head = (3+1) % 5 = 4

เมื่อ record(Success) ครั้งที่ 6 (head=0 เนื่องจาก wrap):
1. อ่านค่าเดิมที่ index[0] = Success → evict → successes--
2. เขียน Success ที่ index[0]
3. successes++ (net = 0)
4. head = (0+1) % 5 = 1
```

การ evict ค่าเก่าแบบ O(1) คือ insight หลักของ ring buffer — ไม่ต้อง shift array ทั้งหมดเหมือน `VecDeque::pop_front()`

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Dependencies

สร้างโปรเจคใหม่ด้วย `cargo new circuit-breaker --lib`:

**`Cargo.toml`:**

```toml
[package]
name = "circuit-breaker"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
parking_lot = "0.12"

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

เหตุผลการเลือก crate:
- **`tokio`** — async runtime มาตรฐาน, ใช้ `tokio::time::timeout` และ `#[tokio::test]`
- **`parking_lot`** — Mutex ที่เร็วและ API สะอาดกว่า `std::sync::Mutex`
- **`serde`/`serde_json`** — serialize config และ metrics เป็น JSON ได้ในภายหลัง

**`src/lib.rs`** — จุดรวม public API:

```rust
pub mod state;
pub mod detector;
pub mod window;
pub mod metrics;
pub mod breaker;

pub use breaker::{CircuitBreaker, CircuitBreakerConfig, CircuitBreakerError};
pub use state::CircuitState;
pub use metrics::CircuitMetrics;
```

---

### ขั้นที่ 2: CircuitState — State Machine ด้วย Enum

State ของ Circuit Breaker มีข้อมูลเฉพาะแต่ละ state ซึ่ง Rust enum รองรับได้อย่างสมบูรณ์:

**`src/state.rs`:**

```rust
use std::time::Instant;

/// สถานะของ Circuit Breaker
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum CircuitState {
    /// วงจรปิด (ทำงานปกติ) — request ทุกอันผ่านได้
    Closed,
    /// วงจรเปิด (บล็อก) — reject request ทุกอัน ยกเว้น probe
    Open { opened_at: Instant },
    /// วงจรครึ่งเปิด — ยอมให้ probe requests ผ่านจำนวนจำกัด
    HalfOpen { probes_allowed: u32, probes_sent: u32 },
}

impl CircuitState {
    pub fn name(&self) -> &'static str {
        match self {
            CircuitState::Closed => "closed",
            CircuitState::Open { .. } => "open",
            CircuitState::HalfOpen { .. } => "half_open",
        }
    }

    pub fn is_closed(&self) -> bool {
        matches!(self, CircuitState::Closed)
    }

    pub fn is_open(&self) -> bool {
        matches!(self, CircuitState::Open { .. })
    }

    pub fn is_half_open(&self) -> bool {
        matches!(self, CircuitState::HalfOpen { .. })
    }
}
```

ข้อสังเกตสำคัญ:

1. `Open { opened_at: Instant }` — เก็บเวลาที่เปิดวงจรไว้ใน variant เอง ทำให้ตรวจสอบ timeout ได้โดยตรงโดยไม่ต้องพึ่ง field แยก
2. `HalfOpen { probes_allowed, probes_sent }` — นับ probe ที่ส่งแล้วเทียบกับ limit ใน variant เดียวกัน
3. `matches!` macro — เป็น Rust idiomatic สำหรับ boolean pattern matching โดยไม่ต้อง `if let`

---

### ขั้นที่ 3: Sliding Window ด้วย Ring Buffer

Sliding window เป็นโครงสร้างที่ต้องออกแบบดี ๆ เพราะถูกเรียกทุก request

**`src/window.rs`:**

```rust
/// ผลลัพธ์ของแต่ละ request
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Outcome {
    Success,
    Failure,
}

/// Sliding window แบบ ring buffer — O(1) insert และ O(1) eviction
#[derive(Debug)]
pub struct SlidingWindow {
    buffer: Vec<Option<Outcome>>,
    head: usize,
    size: usize,
    capacity: usize,
    failures: usize,
    successes: usize,
}

impl SlidingWindow {
    pub fn new(capacity: usize) -> Self {
        assert!(capacity > 0, "capacity must be > 0");
        SlidingWindow {
            buffer: vec![None; capacity],
            head: 0,
            size: 0,
            capacity,
            failures: 0,
            successes: 0,
        }
    }

    /// เพิ่ม outcome ใหม่ (O(1))
    pub fn record(&mut self, outcome: Outcome) {
        // evict ค่าเก่าออก (ถ้า buffer เต็ม)
        if let Some(old) = self.buffer[self.head] {
            match old {
                Outcome::Success => self.successes -= 1,
                Outcome::Failure => self.failures -= 1,
            }
        } else {
            self.size += 1;
        }

        // เพิ่มค่าใหม่
        self.buffer[self.head] = Some(outcome);
        match outcome {
            Outcome::Success => self.successes += 1,
            Outcome::Failure => self.failures += 1,
        }

        self.head = (self.head + 1) % self.capacity;
    }

    pub fn failures(&self) -> usize { self.failures }
    pub fn successes(&self) -> usize { self.successes }
    pub fn total(&self) -> usize { self.failures + self.successes }

    /// อัตราความล้มเหลว 0.0–1.0
    pub fn failure_rate(&self) -> f64 {
        let t = self.total();
        if t == 0 { 0.0 } else { self.failures as f64 / t as f64 }
    }

    pub fn reset(&mut self) {
        for slot in self.buffer.iter_mut() { *slot = None; }
        self.head = 0;
        self.size = 0;
        self.failures = 0;
        self.successes = 0;
    }
}
```

**ทำไมไม่ใช้ `VecDeque`?**

`VecDeque::push_back()` + `pop_front()` ก็ O(1) amortized เหมือนกัน แต่ ring buffer ของเราดีกว่าในแง่:
- ไม่ allocate memory ใหม่ (fixed capacity)
- Cache locality ดีกว่า — ข้อมูลอยู่ใน contiguous memory
- ไม่ต้องจัดการ `len()` แยกต่างหาก เพราะ `failures + successes` คือ total เสมอ

---

### ขั้นที่ 4: FailureDetector Trait และ Strategies

การออกแบบผ่าน trait ทำให้เปลี่ยน strategy ตอน runtime และ test แต่ละ strategy แยกกันได้:

**`src/detector.rs`:**

```rust
use crate::window::SlidingWindow;

/// Trait สำหรับกลยุทธ์การตรวจจับความล้มเหลว
pub trait FailureDetector: Send + Sync {
    /// ควร trip (เปิดวงจร) ไหม?
    fn should_trip(&self, window: &SlidingWindow) -> bool;
    /// ควร recover (ปิดวงจร) ไหม? ใช้ใน HalfOpen
    fn should_recover(&self, window: &SlidingWindow) -> bool;
    fn name(&self) -> &'static str;
}

/// Count-based detector: trip เมื่อมี failure >= threshold ใน window
#[derive(Debug, Clone)]
pub struct CountBasedDetector {
    pub failure_threshold: usize,
    pub success_threshold: usize,
}

impl CountBasedDetector {
    pub fn new(failure_threshold: usize, success_threshold: usize) -> Self {
        Self { failure_threshold, success_threshold }
    }
}

impl FailureDetector for CountBasedDetector {
    fn should_trip(&self, window: &SlidingWindow) -> bool {
        window.failures() >= self.failure_threshold
    }

    fn should_recover(&self, window: &SlidingWindow) -> bool {
        window.successes() >= self.success_threshold
    }

    fn name(&self) -> &'static str { "count_based" }
}

/// Rate-based detector: trip เมื่อ failure rate เกิน threshold
#[derive(Debug, Clone)]
pub struct RateBasedDetector {
    /// อัตราความล้มเหลวสูงสุดที่ยอมรับ (0.0–1.0)
    pub failure_rate_threshold: f64,
    /// จำนวน request ขั้นต่ำก่อนจะ trip (ป้องกัน false positive)
    pub minimum_calls: usize,
    /// อัตรา success ขั้นต่ำใน HalfOpen เพื่อ recover
    pub recovery_success_rate: f64,
}

impl RateBasedDetector {
    pub fn new(
        failure_rate_threshold: f64,
        minimum_calls: usize,
        recovery_success_rate: f64,
    ) -> Self {
        Self { failure_rate_threshold, minimum_calls, recovery_success_rate }
    }
}

impl FailureDetector for RateBasedDetector {
    fn should_trip(&self, window: &SlidingWindow) -> bool {
        window.total() >= self.minimum_calls
            && window.failure_rate() >= self.failure_rate_threshold
    }

    fn should_recover(&self, window: &SlidingWindow) -> bool {
        window.total() > 0
            && (1.0 - window.failure_rate()) >= self.recovery_success_rate
    }

    fn name(&self) -> &'static str { "rate_based" }
}
```

**ทำไม `minimum_calls` ถึงสำคัญ?**

สมมติ window size = 100 แต่เพิ่งเริ่ม service ขึ้นมา — request แรกเป็น failure → failure rate = 100% แต่ถ้าไม่มี `minimum_calls` circuit จะ trip ทันที ซึ่งเป็น false positive ที่ไม่ต้องการ `minimum_calls` บังคับให้รอให้มีข้อมูลพอก่อนตัดสินใจ

---

### ขั้นที่ 5: CircuitMetrics และ Prometheus Output

**`src/metrics.rs`:**

```rust
use std::time::{Instant, SystemTime, UNIX_EPOCH};

#[derive(Debug, Clone)]
pub struct CircuitMetrics {
    pub total_calls: u64,
    pub successes: u64,
    pub failures: u64,
    pub rejections: u64,
    pub timeouts: u64,
    pub state_changes: u64,
    pub last_state_change: Option<Instant>,
    pub current_state: String,
}

impl Default for CircuitMetrics {
    fn default() -> Self {
        Self {
            total_calls: 0,
            successes: 0,
            failures: 0,
            rejections: 0,
            timeouts: 0,
            state_changes: 0,
            last_state_change: None,
            current_state: "closed".to_string(),
        }
    }
}

impl CircuitMetrics {
    pub fn record_success(&mut self) {
        self.total_calls += 1;
        self.successes += 1;
    }

    pub fn record_failure(&mut self) {
        self.total_calls += 1;
        self.failures += 1;
    }

    pub fn record_rejection(&mut self) {
        self.rejections += 1;
    }

    pub fn record_timeout(&mut self) {
        self.total_calls += 1;
        self.timeouts += 1;
        self.failures += 1;
    }

    pub fn record_state_change(&mut self, new_state: &str) {
        self.state_changes += 1;
        self.last_state_change = Some(Instant::now());
        self.current_state = new_state.to_string();
    }

    /// สร้าง Prometheus-compatible text format
    pub fn to_prometheus_text(&self, name: &str) -> String {
        let now_ms = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_millis();

        let state_last_change_secs = self
            .last_state_change
            .map(|i| i.elapsed().as_secs())
            .unwrap_or(0);

        format!(
            "# HELP {name}_total_calls Total number of calls\n\
             # TYPE {name}_total_calls counter\n\
             {name}_total_calls {total} {ts}\n\
             # HELP {name}_successes_total Total successful calls\n\
             # TYPE {name}_successes_total counter\n\
             {name}_successes_total {successes} {ts}\n\
             # HELP {name}_failures_total Total failed calls\n\
             # TYPE {name}_failures_total counter\n\
             {name}_failures_total {failures} {ts}\n\
             # HELP {name}_rejections_total Total rejected calls (circuit open)\n\
             # TYPE {name}_rejections_total counter\n\
             {name}_rejections_total {rejections} {ts}\n\
             # HELP {name}_timeouts_total Total timed-out calls\n\
             # TYPE {name}_timeouts_total counter\n\
             {name}_timeouts_total {timeouts} {ts}\n\
             # HELP {name}_state_changes_total Total state transitions\n\
             # TYPE {name}_state_changes_total counter\n\
             {name}_state_changes_total {state_changes} {ts}\n\
             # HELP {name}_state_last_change_seconds Seconds since last state change\n\
             # TYPE {name}_state_last_change_seconds gauge\n\
             {name}_state_last_change_seconds {state_last_change_secs} {ts}\n",
            name = name,
            total = self.total_calls,
            successes = self.successes,
            failures = self.failures,
            rejections = self.rejections,
            timeouts = self.timeouts,
            state_changes = self.state_changes,
            state_last_change_secs = state_last_change_secs,
            ts = now_ms,
        )
    }
}
```

**Prometheus text format** มีกฎสำคัญสามข้อ:
1. แต่ละ metric ต้องมี `# HELP` line อธิบาย และ `# TYPE` line ระบุ type (`counter` หรือ `gauge`)
2. `counter` — เพิ่มขึ้นเสมอ (ห้ามลด), `gauge` — ขึ้นลงได้
3. Timestamp เป็น milliseconds Unix epoch (optional แต่ recommended)

---

### ขั้นที่ 6: CircuitBreaker — Core Logic

นี่คือหัวใจหลักของโปรเจค ซึ่งรวม state machine, window, detector, และ metrics เข้าด้วยกัน:

**`src/breaker.rs`:**

```rust
use std::future::Future;
use std::sync::Arc;
use std::time::Duration;
use parking_lot::Mutex;
use tokio::time::timeout;

use crate::state::CircuitState;
use crate::window::{SlidingWindow, Outcome};
use crate::detector::FailureDetector;
use crate::metrics::CircuitMetrics;

/// ข้อผิดพลาดที่ CircuitBreaker อาจส่งออกมา
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum CircuitBreakerError {
    /// วงจรเปิดอยู่ — request ถูก reject ทันที
    CircuitOpen,
    /// Timeout — ใช้เวลาเกิน call_timeout
    Timeout,
    /// ข้อผิดพลาดจาก upstream service
    ServiceError(String),
}

impl std::fmt::Display for CircuitBreakerError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            CircuitBreakerError::CircuitOpen => write!(f, "circuit breaker is open"),
            CircuitBreakerError::Timeout => write!(f, "call timed out"),
            CircuitBreakerError::ServiceError(msg) => write!(f, "service error: {}", msg),
        }
    }
}

impl std::error::Error for CircuitBreakerError {}

/// Configuration สำหรับ CircuitBreaker
#[derive(Debug, Clone)]
pub struct CircuitBreakerConfig {
    pub window_size: usize,
    pub open_duration: Duration,
    pub half_open_probes: u32,
    pub call_timeout: Option<Duration>,
    pub name: String,
}

impl Default for CircuitBreakerConfig {
    fn default() -> Self {
        Self {
            window_size: 10,
            open_duration: Duration::from_secs(30),
            half_open_probes: 3,
            call_timeout: Some(Duration::from_secs(5)),
            name: "default".to_string(),
        }
    }
}

/// สถานะภายในของ CircuitBreaker (ถูก guard ด้วย Mutex)
struct BreakerInner {
    state: CircuitState,
    window: SlidingWindow,
    metrics: CircuitMetrics,
}

/// Circuit Breaker ที่ใช้ FailureDetector strategy แบบ pluggable
pub struct CircuitBreaker {
    config: CircuitBreakerConfig,
    inner: Arc<Mutex<BreakerInner>>,
    detector: Arc<dyn FailureDetector>,
}

impl CircuitBreaker {
    pub fn new(config: CircuitBreakerConfig, detector: Arc<dyn FailureDetector>) -> Self {
        let window = SlidingWindow::new(config.window_size);
        let inner = BreakerInner {
            state: CircuitState::Closed,
            window,
            metrics: CircuitMetrics::default(),
        };
        CircuitBreaker {
            config,
            inner: Arc::new(Mutex::new(inner)),
            detector,
        }
    }

    /// ดึง snapshot ของ metrics (clone)
    pub fn metrics(&self) -> CircuitMetrics {
        self.inner.lock().metrics.clone()
    }

    /// ดึง state ปัจจุบัน
    pub fn state(&self) -> CircuitState {
        self.inner.lock().state.clone()
    }

    /// เรียกใช้ async function ผ่าน Circuit Breaker
    pub async fn call<F, Fut, T, E>(&self, f: F) -> Result<T, CircuitBreakerError>
    where
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<T, E>>,
        E: std::fmt::Display,
    {
        // ── Phase 1: ตรวจสอบสถานะก่อน execute ─────────────────────────
        {
            let mut inner = self.inner.lock();
            match &inner.state {
                CircuitState::Open { opened_at } => {
                    let elapsed = opened_at.elapsed();
                    if elapsed >= self.config.open_duration {
                        // เปลี่ยนเป็น HalfOpen
                        inner.state = CircuitState::HalfOpen {
                            probes_allowed: self.config.half_open_probes,
                            probes_sent: 0,
                        };
                        inner.window.reset();
                        inner.metrics.record_state_change("half_open");
                    } else {
                        inner.metrics.record_rejection();
                        return Err(CircuitBreakerError::CircuitOpen);
                    }
                }
                CircuitState::HalfOpen { probes_allowed, probes_sent } => {
                    if probes_sent >= probes_allowed {
                        inner.metrics.record_rejection();
                        return Err(CircuitBreakerError::CircuitOpen);
                    }
                    let allowed = *probes_allowed;
                    let sent = *probes_sent + 1;
                    inner.state = CircuitState::HalfOpen {
                        probes_allowed: allowed,
                        probes_sent: sent,
                    };
                }
                CircuitState::Closed => {}
            }
        } // ← lock หลุดก่อน execute (สำคัญมาก!)

        // ── Phase 2: Execute the function ──────────────────────────────
        let result = if let Some(timeout_dur) = self.config.call_timeout {
            match timeout(timeout_dur, f()).await {
                Ok(r) => r.map_err(|e| CircuitBreakerError::ServiceError(e.to_string())),
                Err(_) => Err(CircuitBreakerError::Timeout),
            }
        } else {
            f().await.map_err(|e| CircuitBreakerError::ServiceError(e.to_string()))
        };

        // ── Phase 3: อัปเดต window และ state ──────────────────────────
        let mut inner = self.inner.lock();
        match &result {
            Ok(_) => {
                inner.window.record(Outcome::Success);
                inner.metrics.record_success();
                if inner.state.is_half_open()
                    && self.detector.should_recover(&inner.window)
                {
                    inner.state = CircuitState::Closed;
                    inner.window.reset();
                    inner.metrics.record_state_change("closed");
                }
            }
            Err(CircuitBreakerError::Timeout) => {
                inner.window.record(Outcome::Failure);
                inner.metrics.record_timeout();
                self.check_and_trip(&mut inner);
            }
            Err(_) => {
                inner.window.record(Outcome::Failure);
                inner.metrics.record_failure();
                self.check_and_trip(&mut inner);
            }
        }

        result
    }

    fn check_and_trip(&self, inner: &mut BreakerInner) {
        if inner.state.is_closed() && self.detector.should_trip(&inner.window) {
            inner.state = CircuitState::Open {
                opened_at: std::time::Instant::now(),
            };
            inner.metrics.record_state_change("open");
        } else if inner.state.is_half_open() {
            // Probe failed — กลับไป Open
            inner.state = CircuitState::Open {
                opened_at: std::time::Instant::now(),
            };
            inner.window.reset();
            inner.metrics.record_state_change("open");
        }
    }

    pub fn reset(&self) {
        let mut inner = self.inner.lock();
        inner.state = CircuitState::Closed;
        inner.window.reset();
        inner.metrics.record_state_change("closed");
    }

    pub fn trip(&self) {
        let mut inner = self.inner.lock();
        inner.state = CircuitState::Open {
            opened_at: std::time::Instant::now(),
        };
        inner.metrics.record_state_change("open");
    }

    pub fn prometheus_metrics(&self) -> String {
        let inner = self.inner.lock();
        inner.metrics.to_prometheus_text(&self.config.name)
    }
}
```

**Lock-free execution** คือ design ที่สำคัญที่สุดในไฟล์นี้:

```
Lock acquired → ตรวจสอบ state → Lock RELEASED
                                      ↓
                              execute f() ← ไม่ hold lock ระหว่างนี้!
                                      ↓
Lock acquired → อัปเดต window/state → Lock released
```

ถ้า hold lock ระหว่าง `f()` ทุก thread จะต้องรอกัน ทำให้ throughput ตก และเกิด deadlock ได้ถ้า `f()` พยายาม call `CircuitBreaker` อีกครั้ง

---

### ขั้นที่ 7: Demo Binary

**`src/main.rs`:**

```rust
use circuit_breaker::{CircuitBreaker, CircuitBreakerConfig};
use circuit_breaker::detector::CountBasedDetector;
use std::sync::Arc;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let config = CircuitBreakerConfig {
        window_size: 5,
        open_duration: Duration::from_secs(2),
        half_open_probes: 2,
        call_timeout: Some(Duration::from_secs(1)),
        name: "demo_service".to_string(),
    };
    let detector = Arc::new(CountBasedDetector::new(3, 2));
    let cb = CircuitBreaker::new(config, detector);

    println!("=== Circuit Breaker Demo ===\n");
    println!("Initial state: {:?}", cb.state());

    // จำลอง 3 failures เพื่อ trip breaker
    for i in 1..=3 {
        let result = cb.call(|| async move {
            Err::<i32, String>(format!("service error #{}", i))
        }).await;
        println!("Call {}: {:?} | State: {}", i, result, cb.state().name());
    }

    println!("\n--- Circuit is now OPEN ---");
    let rejected = cb.call(|| async { Ok::<i32, String>(42) }).await;
    println!("Rejected call: {:?}", rejected);

    println!("\nMetrics:\n{}", cb.prometheus_metrics());
}
```

**Output จากการรัน `cargo run`:**

```
=== Circuit Breaker Demo ===

Initial state: Closed
Call 1: Err(ServiceError("service error #1")) | State: closed
Call 2: Err(ServiceError("service error #2")) | State: closed
Call 3: Err(ServiceError("service error #3")) | State: open

--- Circuit is now OPEN ---
Rejected call: Err(CircuitOpen)

Metrics:
# HELP demo_service_total_calls Total number of calls
# TYPE demo_service_total_calls counter
demo_service_total_calls 3 1790621030465
# HELP demo_service_successes_total Total successful calls
# TYPE demo_service_successes_total counter
demo_service_successes_total 0 1790621030465
# HELP demo_service_failures_total Total failed calls
# TYPE demo_service_failures_total counter
demo_service_failures_total 3 1790621030465
# HELP demo_service_rejections_total Total rejected calls (circuit open)
# TYPE demo_service_rejections_total counter
demo_service_rejections_total 1 1790621030465
# HELP demo_service_timeouts_total Total timed-out calls
# TYPE demo_service_timeouts_total counter
demo_service_timeouts_total 0 1790621030465
# HELP demo_service_state_changes_total Total state transitions
# TYPE demo_service_state_changes_total counter
demo_service_state_changes_total 1 1790621030465
# HELP demo_service_state_last_change_seconds Seconds since last state change
# TYPE demo_service_state_last_change_seconds gauge
demo_service_state_last_change_seconds 0 1790621030465
```

เราเห็นว่า:
- Call 1-2 ยัง `closed` เพราะ failure count ยังไม่ถึง threshold (3)
- Call 3 เป็น failure ครั้งที่ 3 → state เปลี่ยนเป็น `open` ทันทีหลัง call นั้น
- Rejected call ถูก reject เลยโดยไม่ต้อง execute function

---

## การทดสอบ (Testing)

โปรเจคนี้มี **23 unit tests** ครอบคลุม state transitions, detector logic, window eviction, และ metrics counting ทั้งหมด

### รัน Tests

```
cargo test
```

**Output จริงจากการรัน `cargo test`:**

```
   Compiling circuit-breaker v0.1.0 (...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.88s
     Running unittests src/lib.rs (target/debug/deps/circuit_breaker-e3328699abf52da3)

running 23 tests
test breaker::tests::test_closed_to_open_on_failure_threshold ... ok
test breaker::tests::test_metrics_rejection_count ... ok
test breaker::tests::test_open_rejects_immediately ... ok
test breaker::tests::test_prometheus_metrics_format ... ok
test breaker::tests::test_reset_clears_state ... ok
test breaker::tests::test_success_path_stays_closed ... ok
test breaker::tests::test_half_open_back_to_open_on_probe_failure ... ok
test detector::tests::test_count_detector_recover ... ok
test detector::tests::test_count_detector_trip ... ok
test detector::tests::test_rate_detector_minimum_calls ... ok
test detector::tests::test_rate_detector_recover ... ok
test detector::tests::test_rate_detector_trip ... ok
test metrics::tests::test_metrics_counting ... ok
test breaker::tests::test_half_open_to_closed_on_recovery ... ok
test metrics::tests::test_metrics_state_change ... ok
test window::tests::test_window_basic ... ok
test metrics::tests::test_prometheus_output_contains_metric_names ... ok
test window::tests::test_window_eviction ... ok
test window::tests::test_window_failure_rate ... ok
test window::tests::test_window_rate_after_eviction_rolls_over ... ok
test window::tests::test_window_reset ... ok
test breaker::tests::test_timeout_counts_as_failure ... ok
test breaker::tests::test_open_to_half_open_after_timeout ... ok

test result: ok. 23 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.06s

     Running unittests src/main.rs (target/debug/deps/circuit_breaker-1110f3b7f867b0bf)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests circuit_breaker

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### อธิบาย Tests ที่สำคัญ

**Test: Closed → Open on failure threshold**

```rust
#[tokio::test]
async fn test_closed_to_open_on_failure_threshold() {
    let cb = make_breaker(10, 3, Duration::from_secs(60));
    // จำลอง 3 failures
    for _ in 0..3 {
        let _ = cb.call(|| async { Err::<(), _>("err") }).await;
    }
    assert!(cb.state().is_open(), "should be open after 3 failures");
}
```

**Test: Open → HalfOpen → Closed recovery**

```rust
#[tokio::test]
async fn test_half_open_to_closed_on_recovery() {
    let cb = make_breaker(10, 1, Duration::from_millis(10));
    // Trip the breaker
    let _ = cb.call(|| async { Err::<(), _>("err") }).await;
    assert!(cb.state().is_open());

    tokio::time::sleep(Duration::from_millis(20)).await;

    // ส่ง 2 success probes (half_open_probes=2, success_threshold=2)
    let r1 = cb.call(|| async { Ok::<i32, &str>(1) }).await;
    assert!(r1.is_ok());
    let r2 = cb.call(|| async { Ok::<i32, &str>(2) }).await;
    assert!(r2.is_ok());

    assert!(cb.state().is_closed(), "should be closed after 2 successful probes");
}
```

**Test: Timeout counts as failure**

```rust
#[tokio::test]
async fn test_timeout_counts_as_failure() {
    let config = CircuitBreakerConfig {
        window_size: 10,
        open_duration: Duration::from_secs(60),
        half_open_probes: 2,
        call_timeout: Some(Duration::from_millis(30)),
        name: "timeout_test".to_string(),
    };
    let detector = Arc::new(CountBasedDetector::new(2, 2));
    let cb = CircuitBreaker::new(config, detector);

    // call ที่ใช้เวลา 100ms (นานกว่า timeout 30ms)
    let result = cb
        .call(|| async {
            tokio::time::sleep(Duration::from_millis(100)).await;
            Ok::<i32, &str>(1)
        })
        .await;
    assert_eq!(result, Err(CircuitBreakerError::Timeout));
    assert_eq!(cb.metrics().timeouts, 1);
}
```

**Test: Sliding window eviction**

```rust
#[test]
fn test_window_eviction() {
    let mut w = SlidingWindow::new(3);
    // ใส่ 3 failures จนเต็ม window
    w.record(Outcome::Failure);
    w.record(Outcome::Failure);
    w.record(Outcome::Failure);
    assert_eq!(w.failures(), 3);
    // ใส่ success — evict failure เก่าออก
    w.record(Outcome::Success);
    assert_eq!(w.failures(), 2);
    assert_eq!(w.successes(), 1);
    assert_eq!(w.total(), 3); // total คงที่ที่ capacity
}
```

**Test: Rate detector minimum_calls guard**

```rust
#[test]
fn test_rate_detector_minimum_calls() {
    let det = RateBasedDetector::new(0.5, 5, 0.8);
    let mut w = SlidingWindow::new(10);
    // failure rate 100% แต่ยังน้อยกว่า minimum_calls
    w.record(Outcome::Failure);
    w.record(Outcome::Failure);
    assert!(!det.should_trip(&w)); // ← ยังไม่ควร trip
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
```

Binary ถูก output ไปที่ `target/release/circuit-breaker`

### ใช้เป็น Library ใน Project อื่น

เพิ่มใน `Cargo.toml` ของ project ปลายทาง:

```toml
[dependencies]
circuit-breaker = { path = "../circuit-breaker" }
# หรือถ้า publish บน crates.io:
# circuit-breaker = "0.1"
```

แล้วใช้งาน:

```rust
use circuit_breaker::{CircuitBreaker, CircuitBreakerConfig, CircuitBreakerError};
use circuit_breaker::detector::RateBasedDetector;
use std::sync::Arc;
use std::time::Duration;

let config = CircuitBreakerConfig {
    window_size: 20,
    open_duration: Duration::from_secs(10),
    half_open_probes: 3,
    call_timeout: Some(Duration::from_secs(2)),
    name: "payment_service".to_string(),
};

let detector = Arc::new(RateBasedDetector::new(
    0.6,   // trip เมื่อ 60%+ failure rate
    10,    // ต้องมี request ≥ 10 ก่อน
    0.8,   // recover เมื่อ 80%+ success rate
));

let cb = Arc::new(CircuitBreaker::new(config, detector));

// share across threads/tasks:
let cb2 = Arc::clone(&cb);
tokio::spawn(async move {
    let result = cb2.call(|| async {
        reqwest::get("https://api.example.com/pay").await
            .map_err(|e| e.to_string())
    }).await;

    match result {
        Ok(resp) => println!("Success: {:?}", resp.status()),
        Err(CircuitBreakerError::CircuitOpen) => {
            println!("Service unavailable — circuit open, using fallback");
        }
        Err(CircuitBreakerError::Timeout) => {
            println!("Request timed out");
        }
        Err(CircuitBreakerError::ServiceError(e)) => {
            println!("Service error: {}", e);
        }
    }
});
```

### Expose Metrics Endpoint (axum)

เพิ่ม route `/metrics` ใน axum server เพื่อให้ Prometheus scrape ได้:

```rust
use axum::{Router, routing::get, Extension};
use std::sync::Arc;

async fn metrics_handler(
    Extension(cb): Extension<Arc<CircuitBreaker>>,
) -> String {
    cb.prometheus_metrics()
}

let app = Router::new()
    .route("/metrics", get(metrics_handler))
    .layer(Extension(Arc::new(cb)));
```

---

## หลุมพรางและข้อควรระวัง (Pitfalls)

### Pitfall 1: Hold Lock ระหว่าง Await

```rust
// ❌ อย่าทำแบบนี้!
pub async fn call_wrong(&self, f: impl Future<Output = ()>) {
    let _guard = self.inner.lock(); // hold lock...
    f.await;                        // ...ระหว่าง await ← deadlock หรือ poor performance
}
```

`parking_lot::MutexGuard` ไม่ implements `Send` ดังนั้นถ้าพยายาม hold ข้าม `.await` compiler จะ error ว่า "cannot be sent between threads safely" เป็น compile-time safety net ที่ Rust ให้มาฟรี วิธีแก้คือ drop lock ก่อน await เสมอ:

```rust
// ✅ ถูกต้อง
let should_proceed = {
    let inner = self.inner.lock();
    // ตรวจสอบ state...
    true
}; // lock dropped ที่นี่
if should_proceed {
    f.await; // ไม่ hold lock แล้ว
}
```

### Pitfall 2: Race Condition ใน HalfOpen — TOCTOU

สมมติ `half_open_probes = 1` มีสอง thread อ่านค่า `probes_sent = 0` พร้อมกัน ทั้งคู่ผ่านการตรวจสอบและส่ง probe ทั้งสองตัว ทำให้ส่ง probe 2 ตัวทั้งที่ตั้งใจให้มีแค่ 1

วิธีแก้คือในโปรเจคนี้เราอัปเดต `probes_sent` ทันทีที่ตัดสินใจให้ผ่าน (ภายใน lock เดียวกัน) ไม่ได้อ่านก่อนแล้วค่อยเขียนทีหลัง:

```rust
// ✅ read + write ใน critical section เดียวกัน
let allowed = *probes_allowed;
let sent = *probes_sent + 1;  // เพิ่มทันที
inner.state = CircuitState::HalfOpen {
    probes_allowed: allowed,
    probes_sent: sent,   // commit ก่อนปล่อย lock
};
```

### Pitfall 3: f64 Comparison และ Floating Point

```rust
// ❌ อย่าทำแบบนี้
assert_eq!(window.failure_rate(), 0.5); // อาจ fail เพราะ floating point precision
```

ใน test ควรใช้ epsilon comparison:

```rust
// ✅ ถูกต้อง
assert!((window.failure_rate() - 0.5).abs() < 1e-9);
```

เหตุผล: `1.0 / 2` ใน IEEE 754 อาจเป็น `0.4999999999999` หรือ `0.5000000000001` ขึ้นอยู่กับ FPU implementation

### Pitfall 4: Window Reset ใน State Transition

เมื่อเปลี่ยนจาก `Open → HalfOpen` หรือ `HalfOpen → Closed` ต้อง **reset window เสมอ** เพราะข้อมูลเก่าจาก Closed phase ไม่ควรนำมาคิดร่วมกับ probe outcomes:

```rust
// ❌ ถ้าลืม reset
// window ยังมี failures จาก phase เก่า → should_recover() เป็น false ตลอด
// → ไม่มีทางกลับ Closed ได้

// ✅ ถูกต้อง
inner.state = CircuitState::HalfOpen { ... };
inner.window.reset();  // ← สำคัญ!
```

### Pitfall 5: `Instant::now()` ใน Test

`Instant::now()` คืน monotonic clock ซึ่ง tokio สามารถ mock ได้ผ่าน `tokio::time::pause()` แต่ `std::time::Instant` ไม่ถูก mock ถ้า test ต้องการ control time ควรใช้ `tokio::time::Instant` แทน:

```rust
// สำหรับ production code ที่ต้อง test time:
use tokio::time::Instant; // ← mock-able
// แทนที่จะเป็น
use std::time::Instant;   // ← ไม่ mock-able
```

ในโปรเจคนี้เราเลี่ยงปัญหาโดยใช้ real sleep ใน test (`tokio::time::sleep`) แทนการ mock clock

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Time-based Sliding Window (ระดับกลาง)

ในโปรเจคนี้ window ขึ้นกับ **จำนวน request** (count-based) แต่ใน production บางกรณีต้องการ window ขึ้นกับ **เวลา** เช่น "failure ใน 60 วินาทีที่ผ่านมา"

สร้าง `TimeSlidingWindow` ที่เก็บ `Vec<(Instant, Outcome)>` แล้ว evict entries ที่เก่ากว่า window duration ทุกครั้งที่เรียก `record()`:

```rust
pub struct TimeSlidingWindow {
    entries: VecDeque<(Instant, Outcome)>,
    window_duration: Duration,
    failures: usize,
    successes: usize,
}

impl TimeSlidingWindow {
    pub fn record(&mut self, outcome: Outcome) {
        let now = Instant::now();
        // evict entries ที่เก่ากว่า window_duration
        while let Some(&(ts, old)) = self.entries.front() {
            if now.duration_since(ts) > self.window_duration {
                self.entries.pop_front();
                match old {
                    Outcome::Success => self.successes -= 1,
                    Outcome::Failure => self.failures -= 1,
                }
            } else {
                break;
            }
        }
        self.entries.push_back((now, outcome));
        // เพิ่ม counter...
    }
}
```

**สิ่งที่ได้ฝึก:** `VecDeque`, `Instant` arithmetic, เปรียบเทียบ trade-off ระหว่าง count และ time window

### แบบฝึกหัดที่ 2: Bulkhead Pattern (ระดับกลาง-สูง)

เพิ่ม **Bulkhead** (Semaphore-based concurrency limiter) เข้าไปใน CircuitBreaker เพื่อจำกัดจำนวน concurrent calls ที่ส่งไปยัง service พร้อมกัน:

```rust
pub struct CircuitBreakerConfig {
    // ... fields เดิม ...
    /// จำนวน concurrent calls สูงสุด (None = ไม่จำกัด)
    pub max_concurrent: Option<usize>,
}

// ใน CircuitBreaker
struct BreakerInner {
    // ... fields เดิม ...
    semaphore: Option<Arc<tokio::sync::Semaphore>>,
}

// ใน call():
let _permit = if let Some(sem) = &inner.semaphore {
    match sem.try_acquire() {
        Ok(permit) => Some(permit),
        Err(_) => {
            inner.metrics.record_rejection();
            return Err(CircuitBreakerError::BulkheadFull);
        }
    }
} else {
    None
};
```

**สิ่งที่ได้ฝึก:** `tokio::sync::Semaphore`, resource limiting, `TryAcquire`

### แบบฝึกหัดที่ 3: Retry-with-Backoff Integration (ระดับสูง)

สร้าง `RetryPolicy` ที่ทำงานร่วมกับ CircuitBreaker — ถ้า call fail และ circuit ยังไม่ open ให้ retry ด้วย exponential backoff:

```rust
pub struct RetryPolicy {
    pub max_attempts: u32,
    pub initial_delay: Duration,
    pub multiplier: f64,
    pub max_delay: Duration,
}

pub async fn call_with_retry<F, Fut, T, E>(
    cb: &CircuitBreaker,
    policy: &RetryPolicy,
    f: F,
) -> Result<T, CircuitBreakerError>
where
    F: Fn() -> Fut,
    Fut: Future<Output = Result<T, E>>,
    E: std::fmt::Display,
{
    let mut delay = policy.initial_delay;
    for attempt in 0..policy.max_attempts {
        match cb.call(|| f()).await {
            Ok(v) => return Ok(v),
            Err(CircuitBreakerError::CircuitOpen) => {
                return Err(CircuitBreakerError::CircuitOpen); // ไม่ retry ถ้า open
            }
            Err(e) if attempt + 1 == policy.max_attempts => return Err(e),
            Err(_) => {
                tokio::time::sleep(delay).await;
                delay = (delay.mul_f64(policy.multiplier)).min(policy.max_delay);
            }
        }
    }
    unreachable!()
}
```

**สิ่งที่ได้ฝึก:** exponential backoff, retry logic, ทำไมถึงไม่ควร retry เมื่อ circuit open

### แบบฝึกหัดที่ 4: Multi-Level Circuit Breaker (ระดับสูงมาก)

ใน microservices จริง service A เรียก B, B เรียก C ถ้า C down อยากให้ A รู้ตัวเร็ว (ไม่ต้องรอ B timeout ก่อน) สร้าง `CascadingCircuitBreaker` ที่สามารถ subscribe event จาก breaker อื่น:

```rust
pub trait CircuitBreakerObserver: Send + Sync {
    fn on_state_change(&self, name: &str, new_state: &str);
}

// เพิ่ม method ใน CircuitBreaker:
pub fn add_observer(&self, obs: Arc<dyn CircuitBreakerObserver>) {
    self.inner.lock().observers.push(obs);
}
```

แล้วสร้าง `DependencyAwareBreaker` ที่ trip ตัวเองทันทีเมื่อ dependency breaker เปิด:

```rust
struct DependencyAwareBreaker {
    own_breaker: Arc<CircuitBreaker>,
    dependency: Arc<CircuitBreaker>,
}
```

**สิ่งที่ได้ฝึก:** Observer pattern, event propagation, `Arc<dyn Trait>` graph

### แบบฝึกหัดที่ 5: Config Hot-Reload (ระดับกลาง)

เพิ่ม ability ให้ `CircuitBreakerConfig` สามารถอัปเดตได้ขณะ runtime โดยไม่ต้องสร้าง instance ใหม่ โดยใช้ `ArcSwap` จาก crate `arc-swap`:

```rust
use arc_swap::ArcSwap;

pub struct CircuitBreaker {
    config: ArcSwap<CircuitBreakerConfig>, // ← atomic swap
    inner: Arc<Mutex<BreakerInner>>,
    detector: ArcSwap<Box<dyn FailureDetector>>,
}

impl CircuitBreaker {
    pub fn update_config(&self, new_config: CircuitBreakerConfig) {
        self.config.store(Arc::new(new_config));
    }

    pub fn update_detector(&self, new_detector: Box<dyn FailureDetector>) {
        self.detector.store(Arc::new(new_detector));
    }
}
```

**สิ่งที่ได้ฝึก:** `arc-swap`, lock-free configuration updates, atomic pointer swap

### แบบฝึกหัดที่ 6: Distributed Circuit Breaker (ระดับ Expert)

ในระบบที่มีหลาย instance ของ service เดียวกัน (horizontal scaling) Circuit Breaker ของแต่ละ instance จะไม่รู้สถานะของ instance อื่น ทำให้บาง instance ยัง send traffic ไปยัง downstream ที่ down อยู่

สร้าง `RedisBackedCircuitBreaker` ที่ sync state กันผ่าน Redis Pub/Sub:

```rust
// เมื่อ state เปลี่ยน ให้ publish ไปที่ Redis channel
async fn publish_state_change(&self, new_state: &str) {
    self.redis.publish(
        format!("circuit:{}:state", self.config.name),
        new_state,
    ).await.ok();
}

// Subscribe และ sync state จาก instances อื่น
async fn start_sync_task(breaker: Arc<CircuitBreaker>, redis: RedisClient) {
    let mut sub = redis.subscribe(format!("circuit:{}:state", ...)).await?;
    while let Some(msg) = sub.next().await {
        match msg.as_str() {
            "open" => breaker.trip(),
            "closed" => breaker.reset(),
            _ => {}
        }
    }
}
```

**สิ่งที่ได้ฝึก:** Redis Pub/Sub, distributed state sync, network partition handling

---

## สรุป

ในโปรเจคนี้เราได้สร้าง Circuit Breaker library ที่มีคุณภาพระดับ production โดยครอบคลุม:

**Pattern และ Concepts สำคัญ:**

| สิ่งที่สร้าง | Rust Pattern |
|---|---|
| State machine | Enum with data in variants |
| Ring buffer | `Vec` + modular head pointer |
| Strategy pattern | `trait FailureDetector` + `Arc<dyn T>` |
| Thread-safe state | `Arc<parking_lot::Mutex<T>>` |
| Async wrapping | `FnOnce() -> Fut` + `tokio::time::timeout` |
| Lock-free execution | Drop lock before await |
| Metrics export | Prometheus text format |

**สิ่งที่ทำให้ implementation นี้ production-ready:**

1. **Lock-free execution** — ไม่ hold lock ระหว่าง async operation
2. **Configurable strategies** — swap detector ได้ตอน runtime
3. **Compile-time safety** — `MutexGuard` ไม่ `Send` ป้องกัน lock-across-await
4. **Zero false positive guard** — `minimum_calls` ใน rate detector
5. **Window reset on transition** — ป้องกัน stale data ข้าม phase

โปรเจคถัดไป **F05: Saga Pattern** จะนำ Circuit Breaker นี้ไปใช้เป็นส่วนหนึ่งของ distributed transaction — เมื่อ step ใดใน saga fail (และถูก circuit break) เราต้องรัน compensating transaction กลับไปยัง steps ก่อนหน้า ซึ่งต้องการ state machine ที่ซับซ้อนกว่านี้

---

**โปรเจคก่อนหน้า:** [Project F03: Distributed Lock](project-f03-dist-lock.md) | **โปรเจคถัดไป:** [Project F05: Saga Pattern](project-f05-saga.md)
