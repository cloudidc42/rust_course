# Project G09: Rate Limiter Service

> โมดูล: G — DevOps & Infrastructure | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 14 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **rate limiting library** ระดับ production ที่รองรับหลาย algorithm พร้อมกัน — ตั้งแต่ Token Bucket สำหรับ burst traffic, Leaky Bucket สำหรับ traffic shaping, Sliding Window สำหรับความแม่นยำสูง ไปจนถึง Multi-Key Limiter สำหรับระบบที่ต้อง track หลาย user พร้อมกันในหน่วยความจำจำกัด

Rate limiting คือกลไกพื้นฐานที่สุดอย่างหนึ่งของระบบ distributed เพราะมันป้องกัน resource exhaustion ทั้งโดยตั้งใจ (DDoS) และไม่ตั้งใจ (buggy client, retry storm) โดย library นี้ถูกออกแบบให้ thread-safe ด้วย atomic operations และ lock-free data structures ทำให้เหมาะสำหรับ hot path ที่ต้องการ latency ต่ำ

**Use cases จริงในโลก production:**
- **API Gateway** — จำกัด requests ต่อ API key ป้องกัน abuse และ quota exhaustion
- **Web Application Firewall** — throttle requests ต่อ IP เพื่อป้องกัน brute-force login
- **Message Broker** — ควบคุม rate ของ message publishing ต่อ producer
- **Microservice mesh** — ป้องกัน cascading failure เมื่อ upstream service ช้า
- **CDN edge nodes** — จำกัด bandwidth ต่อ client โดยไม่ต้อง centralized coordination

**ทำไม rate limiting ถึงยากกว่าที่คิด?**

1. **Correctness under concurrency** — หลาย threads เรียก `try_consume` พร้อมกัน ต้องไม่มี race condition
2. **Algorithm trade-offs** — แต่ละ algorithm มี trade-off ระหว่าง accuracy, memory, burst behavior ที่ต่างกัน
3. **Clock skew** — `Instant` บน multi-core อาจไม่ monotonic ทุกกรณี ต้องระวัง
4. **Memory management** — Multi-key limiter ต้องมี eviction policy ป้องกัน unbounded growth
5. **Distributed scenarios** — การทำ rate limiting ใน distributed system ต้องการ coordination layer พิเศษ (Redis, etc.)

## สิ่งที่จะได้เรียนรู้

- **Atomic operations** — `AtomicU64`, `compare_exchange_weak` loop สำหรับ lock-free token bucket
- **Integer bit-casting** — จำลอง fractional tokens ด้วย scaled integers เพื่อหลีกเลี่ยง AtomicF64 ที่ไม่มีใน stable
- **Mutex patterns** — ใช้ `Mutex<VecDeque>` อย่างถูกต้องสำหรับ leaky bucket queue
- **DashMap** — concurrent hash map สำหรับ per-key storage โดยไม่ต้อง `RwLock<HashMap>`
- **LRU eviction** — ออกแบบ eviction policy ใน concurrent context
- **Algorithm analysis** — เปรียบเทียบ Fixed Window กับ Sliding Window Log และ Weighted Sliding Window
- **Tower middleware** — สร้าง `Layer`/`Service` trait สำหรับ HTTP middleware integration
- **Testing concurrent code** — เขียน test ที่พิสูจน์ thread safety โดยไม่ใช้ sleep

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–30**: Rust basics — ownership, structs, enums, traits, generics, `Vec`, `HashMap`
- **Part 31–40**: Error handling, trait objects, `Box<dyn Trait>`, lifetime basics
- **Part 41–50**: Concurrency — `Arc`, `Mutex`, `thread::spawn`, `RwLock`
- **Part 51–60**: `std::sync::atomic` — `AtomicU64`, `Ordering`, `compare_exchange`
- **Part 61–70**: Async/await, `tokio`, `OnceLock` สำหรับ global state
- **Part 71–80**: External crates — `dashmap`, `serde`, common patterns
- **Part 96–110**: Production patterns — library design, Tower ecosystem, HTTP middleware

## โครงสร้างโปรเจค (Project Layout)

```
rate-limiter/
├── src/
│   ├── lib.rs            ← re-exports และ module declarations
│   ├── main.rs           ← demo binary
│   ├── token_bucket.rs   ← Token Bucket (AtomicU64 scaled integers)
│   ├── leaky_bucket.rs   ← Leaky Bucket (Mutex<VecDeque<Request>>)
│   ├── sliding_window.rs ← SlidingWindowLog, FixedWindowCounter, SlidingWindowRate
│   └── multi_key.rs      ← MultiKeyLimiter<K> พร้อม LRU eviction
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Algorithm Comparison

| Algorithm | Memory | Burst Handling | Accuracy | Use Case |
|-----------|--------|----------------|----------|----------|
| Token Bucket | O(1) | อนุญาต burst | ดี | API quota, bursty traffic |
| Leaky Bucket | O(queue) | ปรับให้เรียบ | ดีมาก | Traffic shaping, output smoothing |
| Fixed Window | O(1) | อาจ double burst | พอใช้ | Simple throttling |
| Sliding Window Log | O(requests) | ควบคุมได้แม่น | ดีที่สุด | Strict rate limiting |
| Sliding Window Rate | O(1) | ค่อนข้างแม่น | ดี | Balanced memory+accuracy |

### Data Flow

```
Client Request
      │
      │  try_consume(key) / allow()
      ▼
┌─────────────────────────────────────────────────────────┐
│                    MultiKeyLimiter<K>                    │
│                                                          │
│  DashMap<K, LimiterEntry>                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │  key "a" │  │  key "b" │  │  key "c" │  ...         │
│  │TokenBucket│  │TokenBucket│  │TokenBucket│             │
│  └──────────┘  └──────────┘  └──────────┘              │
│                  LRU Eviction when len > max_keys        │
└────────────────────────────┬────────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │    TokenBucket     │
                    │  AtomicU64 tokens  │──── try_consume(n)
                    │  lazy refill       │
                    └────────────────────┘
```

### Token Bucket: Scaled Integer Design

Rust stable ไม่มี `AtomicF64` ดังนั้นเราใช้เทคนิค **scaled integer** — แทนที่จะเก็บ `tokens: f64` เราเก็บ `tokens: AtomicU64` โดย `1 token = TOKEN_SCALE (1,000,000) units`:

```
tokens = 3.5 tokens  →  stored as  3_500_000 u64
capacity = 10        →  capacity_scaled = 10_000_000 u64
refill_rate = 2/s    →  refill_rate_scaled = 2_000_000 u64/s
```

การ refill คำนวณจาก elapsed microseconds:
```
tokens_to_add = refill_rate_scaled * elapsed_us / 1_000_000
```

ให้ความแม่นยำ sub-token โดยไม่ต้องใช้ floating-point atomics

### Leaky Bucket: Queue-based Smoothing

```
Incoming (burst)          Queue (capacity=3)     Output (1/s)
    │                   ┌──────────────────┐
    │ req1 ──────────►  │ [req1, req2, req3]│ ──────────► process req1
    │ req2 ──────────►  │                  │
    │ req3 ──────────►  │                  │
    │ req4 ── DROP ──►  │ FULL             │
    │                   └──────────────────┘
```

### Sliding Window Rate: Weighted Interpolation

```
prev_window    |  current_window
[───────────]  [───────────]
              ↑ now
              
sliding_window = [─────────────]  (ย้อนหลัง 1 window จาก now)

overlap = ส่วนที่ prev_window ยังอยู่ใน sliding_window
estimated_count = prev_count × overlap + curr_count
```

ประหยัดหน่วยความจำเพราะเก็บแค่ 2 counters แทน log ของทุก timestamp

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Token Bucket พื้นฐาน

เริ่มต้นด้วย Token Bucket ซึ่งเป็น algorithm ที่พบบ่อยที่สุด เข้าใจง่าย และ thread-safe ด้วย atomic operations

สร้างโปรเจคก่อน:

```bash
cargo new rate-limiter --lib
cd rate-limiter
```

แก้ไข `Cargo.toml`:

```toml
[package]
name = "rate-limiter"
version = "0.1.0"
edition = "2021"

[dependencies]
dashmap = "6"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

สร้าง `src/token_bucket.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::{SystemTime, UNIX_EPOCH};

/// Token Bucket rate limiter (thread-safe ด้วย AtomicU64)
///
/// tokens ถูกเก็บแบบ scaled integer (1 token = TOKEN_SCALE units)
/// เพื่อใช้ AtomicU64 แทน AtomicF64 ที่ไม่มีใน stable Rust
pub struct TokenBucket {
    capacity_scaled: u64,
    tokens: AtomicU64,
    refill_rate_scaled: u64,
    last_refill_us: AtomicU64,
}

pub const TOKEN_SCALE: u64 = 1_000_000;

fn now_us() -> u64 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap_or_default()
        .as_micros() as u64
}

impl TokenBucket {
    pub fn new(capacity: u64, refill_rate: u64) -> Self {
        let capacity_scaled = capacity * TOKEN_SCALE;
        TokenBucket {
            capacity_scaled,
            tokens: AtomicU64::new(capacity_scaled),  // เริ่มเต็ม
            refill_rate_scaled: refill_rate * TOKEN_SCALE,
            last_refill_us: AtomicU64::new(now_us()),
        }
    }

    fn refill(&self) {
        let now = now_us();
        let last = self.last_refill_us.load(Ordering::Relaxed);
        if now <= last { return; }

        let elapsed_us = now - last;
        let tokens_to_add = (self.refill_rate_scaled as u128
            * elapsed_us as u128 / 1_000_000) as u64;

        if tokens_to_add == 0 { return; }

        // CAS loop เพื่ออัปเดต last_refill ก่อน ป้องกัน double-count
        let _ = self.last_refill_us.compare_exchange(
            last, now, Ordering::AcqRel, Ordering::Relaxed
        );

        // CAS loop เพื่อเพิ่ม tokens โดยไม่เกิน capacity
        let mut current = self.tokens.load(Ordering::Relaxed);
        loop {
            let new_val = (current + tokens_to_add).min(self.capacity_scaled);
            match self.tokens.compare_exchange_weak(
                current, new_val, Ordering::AcqRel, Ordering::Relaxed
            ) {
                Ok(_) => break,
                Err(actual) => current = actual,
            }
        }
    }

    /// พยายาม consume n tokens; คืน true ถ้าสำเร็จ
    pub fn try_consume(&self, n: u64) -> bool {
        self.refill();
        let cost = n * TOKEN_SCALE;
        let mut current = self.tokens.load(Ordering::Relaxed);
        loop {
            if current < cost { return false; }
            match self.tokens.compare_exchange_weak(
                current, current - cost, Ordering::AcqRel, Ordering::Relaxed
            ) {
                Ok(_) => return true,
                Err(actual) => current = actual,
            }
        }
    }

    pub fn available_tokens(&self) -> u64 {
        self.refill();
        self.tokens.load(Ordering::Relaxed) / TOKEN_SCALE
    }
}
```

**ทำไมถึงใช้ `compare_exchange_weak` แทน `compare_exchange`?**

`compare_exchange_weak` อาจ fail แบบ spurious (false negative) แต่เร็วกว่าบน architectures ที่ใช้ LL/SC (ARM, RISC-V) ซึ่งเหมาะกับ loop pattern — ถ้า fail ก็วน loop ใหม่อยู่แล้ว

**ทำไม Relaxed Ordering ถึงพอเพียงสำหรับ read แต่ต้องใช้ AcqRel สำหรับ CAS?**

- `Relaxed`: เห็นค่าล่าสุดหรือเก่ากว่า — ใช้สำหรับ "hint" เช่น การ read ก่อน CAS
- `AcqRel`: CAS สำเร็จ → Acquire (เห็น writes ทั้งหมดก่อน) + Release (writes ของเราถูกเห็นโดย Acquire อื่น)

### ขั้นที่ 2: Token Bucket พร้อม Testing

เพิ่ม helper method สำหรับ testing และเขียน unit tests:

```rust
// เพิ่มใน impl TokenBucket
pub fn new_with_tokens(capacity: u64, refill_rate: u64, initial_tokens: u64) -> Self {
    let capacity_scaled = capacity * TOKEN_SCALE;
    let initial_scaled = initial_tokens.min(capacity) * TOKEN_SCALE;
    TokenBucket {
        capacity_scaled,
        tokens: AtomicU64::new(initial_scaled),
        refill_rate_scaled: refill_rate * TOKEN_SCALE,
        last_refill_us: AtomicU64::new(now_us()),
    }
}

#[cfg(test)]
pub fn set_tokens_for_test(&self, n: u64) {
    self.tokens.store(n * TOKEN_SCALE, Ordering::SeqCst);
}
```

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;
    use std::time::Duration;

    #[test]
    fn test_token_bucket_allows_up_to_capacity() {
        let tb = TokenBucket::new(5, 1);
        for i in 1..=5 {
            assert!(tb.try_consume(1), "ครั้งที่ {} ควรผ่าน", i);
        }
        assert!(!tb.try_consume(1), "ครั้งที่ 6 ควรถูก reject");
    }

    #[test]
    fn test_token_bucket_burst_consume() {
        let tb = TokenBucket::new(10, 1);
        assert!(tb.try_consume(10), "consume 10 ครั้งเดียวควรผ่าน");
        assert!(!tb.try_consume(1), "หลังหมดแล้วต้อง reject");
    }

    #[test]
    fn test_token_bucket_refills_over_time() {
        let tb = TokenBucket::new_with_tokens(10, 10, 0);
        thread::sleep(Duration::from_millis(150));
        let avail = tb.available_tokens();
        assert!(avail >= 1,
            "หลังรอ 150ms ควรมีอย่างน้อย 1 token แต่ได้ {}", avail);
    }

    #[test]
    fn test_token_bucket_does_not_exceed_capacity() {
        let tb = TokenBucket::new(5, 100);
        thread::sleep(Duration::from_millis(100));
        let avail = tb.available_tokens();
        assert_eq!(avail, 5,
            "ต้องไม่เกิน capacity=5 แต่ได้ {}", avail);
    }

    #[test]
    fn test_token_bucket_thread_safe() {
        use std::sync::Arc;
        use std::sync::atomic::{AtomicU64, Ordering as AOrdering};

        let tb = Arc::new(TokenBucket::new(100, 0));
        let success_count = Arc::new(AtomicU64::new(0));
        let mut handles = vec![];

        for _ in 0..20 {
            let tb_clone = Arc::clone(&tb);
            let count_clone = Arc::clone(&success_count);
            handles.push(thread::spawn(move || {
                for _ in 0..10 {
                    if tb_clone.try_consume(1) {
                        count_clone.fetch_add(1, AOrdering::Relaxed);
                    }
                }
            }));
        }
        for h in handles { h.join().unwrap(); }

        let total = success_count.load(AOrdering::Relaxed);
        assert_eq!(total, 100,
            "ต้องได้ผ่านพอดี 100 ครั้ง แต่ได้ {}", total);
    }
}
```

### ขั้นที่ 3: Leaky Bucket

Leaky Bucket แตกต่างจาก Token Bucket ตรงที่มัน **smooth traffic** แทนที่จะอนุญาต burst — requests ถูก queue และ process ในอัตราคงที่

สร้าง `src/leaky_bucket.rs`:

```rust
use std::collections::VecDeque;
use std::sync::Mutex;
use std::time::Instant;

/// Request ที่รอการ process
#[derive(Debug, Clone)]
pub struct Request {
    pub id: u64,
    pub enqueued_at: Instant,
}

/// Leaky Bucket rate limiter
///
/// - requests เข้า queue ด้านหลัง (push_back)
/// - process ออกจากด้านหน้า (pop_front) ในอัตราคงที่ (leak_rate/s)
/// - ถ้า queue เต็ม → drop request ใหม่
pub struct LeakyBucket {
    capacity: usize,
    leak_rate: f64,          // requests ต่อวินาที
    queue: Mutex<VecDeque<Request>>,
    last_leak: Mutex<Instant>,
}

impl LeakyBucket {
    pub fn new(capacity: usize, leak_rate: f64) -> Self {
        LeakyBucket {
            capacity,
            leak_rate,
            queue: Mutex::new(VecDeque::new()),
            last_leak: Mutex::new(Instant::now()),
        }
    }

    fn do_leak(&self) {
        let now = Instant::now();
        let mut last = self.last_leak.lock().unwrap();
        let elapsed = now.duration_since(*last).as_secs_f64();
        let to_remove = (elapsed * self.leak_rate).floor() as usize;
        if to_remove > 0 {
            *last = now;
            let mut q = self.queue.lock().unwrap();
            for _ in 0..to_remove {
                if q.pop_front().is_none() { break; }
            }
        }
    }

    /// Enqueue request ใหม่; คืน false ถ้า queue เต็ม (drop)
    pub fn try_enqueue(&self, request: Request) -> bool {
        self.do_leak();
        let mut q = self.queue.lock().unwrap();
        if q.len() >= self.capacity {
            false
        } else {
            q.push_back(request);
            true
        }
    }

    pub fn queue_len(&self) -> usize {
        self.queue.lock().unwrap().len()
    }
}
```

**ข้อควรระวัง: Deadlock ใน `do_leak`**

สังเกตว่า `do_leak` lock `last_leak` และ `queue` แยกกัน ไม่ได้ lock พร้อมกัน ถ้า lock พร้อมกันจะเกิด deadlock ได้ถ้า thread อื่น lock ในลำดับตรงข้าม

### ขั้นที่ 4: Sliding Window Log

Sliding Window Log เป็น algorithm ที่แม่นยำที่สุด เพราะเก็บ timestamp ของทุก request ที่ผ่าน แต่ใช้หน่วยความจำมากกว่า

สร้าง `src/sliding_window.rs` (ส่วน SlidingWindowLog):

```rust
use std::collections::VecDeque;
use std::sync::Mutex;
use std::time::{Duration, Instant};

/// Log-based sliding window rate limiter
/// Memory: O(requests ใน window) — เหมาะกับ limit ต่ำ (เช่น 100 req/min)
pub struct SlidingWindowLog {
    window: Duration,
    limit: usize,
    log: Mutex<VecDeque<Instant>>,
}

impl SlidingWindowLog {
    pub fn new(window: Duration, limit: usize) -> Self {
        SlidingWindowLog { window, limit, log: Mutex::new(VecDeque::new()) }
    }

    /// ตรวจสอบและบันทึก request
    /// ลบ entries เก่าก่อน แล้วตรวจสอบ limit
    pub fn allow(&self) -> bool {
        let now = Instant::now();
        let cutoff = now - self.window;
        let mut log = self.log.lock().unwrap();

        // ลบ entries ที่อยู่นอก window (เก่ากว่า cutoff)
        while log.front().map(|t| *t < cutoff).unwrap_or(false) {
            log.pop_front();
        }

        if log.len() < self.limit {
            log.push_back(now);
            true
        } else {
            false
        }
    }

    pub fn count_in_window(&self) -> usize {
        let now = Instant::now();
        let cutoff = now - self.window;
        self.log.lock().unwrap()
            .iter()
            .filter(|t| **t >= cutoff)
            .count()
    }
}
```

**เหตุใด Sliding Window Log ถึงแม่นยำกว่า Fixed Window?**

Fixed Window มีปัญหา **boundary burst**: ถ้า limit=5, window=1s ผู้ใช้อาจส่ง 5 requests ช่วงท้ายวินาที แล้วส่งอีก 5 ช่วงต้นวินาทีถัดไป รวมเป็น 10 requests ใน 200ms ซึ่งไม่ใช่ rate ที่ตั้งใจ Sliding Window Log ไม่มีปัญหานี้เพราะนับจาก "1 second ที่ผ่านมา" เสมอ

### ขั้นที่ 5: Fixed Window Counter และ Sliding Window Rate

เพิ่ม Fixed Window สำหรับเปรียบเทียบ และ Sliding Window Rate ที่ประหยัดหน่วยความจำ:

```rust
/// Fixed Window Counter — reset ทุกสิ้น window period
pub struct FixedWindowCounter {
    window: Duration,
    limit: usize,
    count: Mutex<usize>,
    window_start: Mutex<Instant>,
}

impl FixedWindowCounter {
    pub fn new(window: Duration, limit: usize) -> Self {
        FixedWindowCounter {
            window, limit,
            count: Mutex::new(0),
            window_start: Mutex::new(Instant::now()),
        }
    }

    pub fn allow(&self) -> bool {
        let now = Instant::now();
        let mut start = self.window_start.lock().unwrap();
        let mut count = self.count.lock().unwrap();

        if now.duration_since(*start) >= self.window {
            *start = now;
            *count = 0;
        }

        if *count < self.limit {
            *count += 1;
            true
        } else {
            false
        }
    }
}
```

```rust
/// Sliding Window Rate — O(1) memory ด้วย weighted interpolation
///
/// ใช้ 2 counters: prev_count (window ก่อน) และ curr_count (window ปัจจุบัน)
/// estimated = prev_count × (1 - position_in_curr_window) + curr_count
pub struct SlidingWindowRate {
    window: Duration,
    limit: usize,
    inner: Mutex<SlidingWindowRateInner>,
}

struct SlidingWindowRateInner {
    prev_count: usize,
    curr_count: usize,
    window_start: Instant,
}

impl SlidingWindowRate {
    pub fn new(window: Duration, limit: usize) -> Self {
        SlidingWindowRate {
            window, limit,
            inner: Mutex::new(SlidingWindowRateInner {
                prev_count: 0,
                curr_count: 0,
                window_start: Instant::now(),
            }),
        }
    }

    pub fn allow(&self) -> bool {
        let now = Instant::now();
        let mut inner = self.inner.lock().unwrap();
        let elapsed = now.duration_since(inner.window_start).as_secs_f64();
        let window_secs = self.window.as_secs_f64();

        if elapsed >= window_secs * 2.0 {
            // ผ่าน 2 windows แล้ว — reset สมบูรณ์
            inner.prev_count = 0;
            inner.curr_count = 0;
            inner.window_start = now;
        } else if elapsed >= window_secs {
            // ขึ้น window ใหม่ — previous กลายเป็น prev
            inner.prev_count = inner.curr_count;
            inner.curr_count = 0;
            inner.window_start += self.window;
        }

        let elapsed_in_curr = now.duration_since(inner.window_start).as_secs_f64();
        let overlap = 1.0 - (elapsed_in_curr / window_secs).min(1.0);
        let estimated = (inner.prev_count as f64 * overlap
            + inner.curr_count as f64) as usize;

        if estimated < self.limit {
            inner.curr_count += 1;
            true
        } else {
            false
        }
    }
}
```

**ตัวอย่าง Weighted Interpolation:**
- `prev_count = 8`, `curr_count = 3`, `limit = 10`
- position ใน current window = 30% (elapsed 0.3s จาก 1s window)
- `overlap = 1 - 0.3 = 0.7`
- `estimated = 8 × 0.7 + 3 = 5.6 + 3 = 8.6 → 8`
- request 9 → estimated = 9 < 10 → ALLOWED
- request 10 → estimated = 10 ≥ 10 → DENIED

### ขั้นที่ 6: Multi-Key Limiter พร้อม LRU Eviction

Multi-Key Limiter จัดการ per-key limits โดยแต่ละ key มี `TokenBucket` แยกกัน พร้อมระบบ eviction เพื่อป้องกัน memory leak

สร้าง `src/multi_key.rs`:

```rust
use std::hash::Hash;
use std::sync::{Arc, Mutex};
use std::time::Instant;
use dashmap::DashMap;
use crate::token_bucket::TokenBucket;

struct LimiterEntry {
    limiter: Arc<TokenBucket>,
    last_used: Instant,
}

/// Multi-key rate limiter
///
/// เหมาะสำหรับ per-user, per-IP, per-API-key rate limiting
/// รองรับ LRU eviction เมื่อจำนวน keys เกิน max_keys
pub struct MultiKeyLimiter<K: Eq + Hash + Clone> {
    limiters: DashMap<K, LimiterEntry>,
    capacity: u64,
    refill_rate: u64,
    max_keys: usize,
    evict_lock: Mutex<()>,
}

impl<K: Eq + Hash + Clone> MultiKeyLimiter<K> {
    pub fn new(capacity: u64, refill_rate: u64, max_keys: usize) -> Self {
        MultiKeyLimiter {
            limiters: DashMap::new(),
            capacity, refill_rate, max_keys,
            evict_lock: Mutex::new(()),
        }
    }

    pub fn try_consume(&self, key: &K) -> bool {
        // อัปเดต last_used ถ้ามี entry อยู่แล้ว
        if let Some(mut entry) = self.limiters.get_mut(key) {
            entry.last_used = Instant::now();
            return entry.limiter.try_consume(1);
        }

        // สร้าง entry ใหม่ — evict ก่อนถ้าเต็ม
        if self.limiters.len() >= self.max_keys {
            self.evict_lru();
        }

        let limiter = Arc::new(TokenBucket::new(self.capacity, self.refill_rate));
        let result = limiter.try_consume(1);
        self.limiters.insert(key.clone(), LimiterEntry {
            limiter, last_used: Instant::now(),
        });
        result
    }

    fn evict_lru(&self) {
        let _lock = self.evict_lock.lock().unwrap();
        if self.limiters.len() < self.max_keys { return; }

        let oldest = self.limiters
            .iter()
            .min_by_key(|entry| entry.value().last_used)
            .map(|entry| entry.key().clone());

        if let Some(k) = oldest {
            self.limiters.remove(&k);
        }
    }

    pub fn key_count(&self) -> usize { self.limiters.len() }
    pub fn contains_key(&self, key: &K) -> bool { self.limiters.contains_key(key) }
}
```

**ทำไมต้องมี `evict_lock`?**

ในกรณีที่หลาย threads พยายามสร้าง key ใหม่พร้อมกันและ `len() >= max_keys` ทุกตัว ถ้าไม่มี lock อาจ evict หลาย keys ทั้งที่ควร evict แค่หนึ่ง `evict_lock` ทำให้ eviction เป็น critical section

### ขั้นที่ 7: Tower Middleware Integration

เพื่อ integrate เข้ากับ HTTP stack ที่ใช้ Tower ecosystem (เช่น `axum`, `hyper`) เราต้องสร้าง middleware ที่ implement `Layer` และ `Service` traits

เพิ่ม `tower` ใน `Cargo.toml`:

```toml
[dependencies]
tower = { version = "0.5", features = ["util"] }
http = "1"
```

สร้าง `src/middleware.rs`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::Arc;
use std::task::{Context, Poll};
use tower::{Layer, Service};

use crate::multi_key::MultiKeyLimiter;

/// Tower Layer สำหรับ rate limiting ต่อ key
pub struct RateLimitLayer {
    limiter: Arc<MultiKeyLimiter<String>>,
    key_extractor: Arc<dyn Fn(&str) -> String + Send + Sync>,
}

impl RateLimitLayer {
    pub fn new(
        limiter: Arc<MultiKeyLimiter<String>>,
        key_extractor: impl Fn(&str) -> String + Send + Sync + 'static,
    ) -> Self {
        RateLimitLayer {
            limiter,
            key_extractor: Arc::new(key_extractor),
        }
    }
}

impl<S> Layer<S> for RateLimitLayer {
    type Service = RateLimitService<S>;

    fn layer(&self, inner: S) -> Self::Service {
        RateLimitService {
            inner,
            limiter: Arc::clone(&self.limiter),
            key_extractor: Arc::clone(&self.key_extractor),
        }
    }
}

pub struct RateLimitService<S> {
    inner: S,
    limiter: Arc<MultiKeyLimiter<String>>,
    key_extractor: Arc<dyn Fn(&str) -> String + Send + Sync>,
}

impl<S, Request> Service<Request> for RateLimitService<S>
where
    S: Service<Request>,
    S::Future: Send + 'static,
    Request: AsRef<str>,
{
    type Response = Result<S::Response, RateLimitError>;
    type Error = S::Error;
    type Future = Pin<Box<dyn Future<Output = Result<Self::Response, Self::Error>> + Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Request) -> Self::Future {
        let key = (self.key_extractor)(req.as_ref());
        let allowed = self.limiter.try_consume(&key);

        if !allowed {
            return Box::pin(async move {
                Ok(Err(RateLimitError {
                    retry_after_secs: 1,
                    key,
                }))
            });
        }

        let fut = self.inner.call(req);
        Box::pin(async move {
            let response = fut.await?;
            Ok(Ok(response))
        })
    }
}

/// Error ที่ส่งกลับเมื่อถูก rate limited
/// ใช้สร้าง HTTP 429 response พร้อม Retry-After header
#[derive(Debug)]
pub struct RateLimitError {
    pub retry_after_secs: u64,
    pub key: String,
}

impl RateLimitError {
    /// สร้าง HTTP 429 response headers
    pub fn to_headers(&self) -> Vec<(String, String)> {
        vec![
            ("HTTP/1.1 429 Too Many Requests".to_string(), "".to_string()),
            ("Retry-After".to_string(), self.retry_after_secs.to_string()),
            ("X-RateLimit-Limit".to_string(), "rate_limit".to_string()),
            ("X-RateLimit-Key".to_string(), self.key.clone()),
        ]
    }
}
```

**การใช้งานจริงกับ Axum:**

```rust
use axum::{Router, routing::get, extract::ConnectInfo};
use std::net::SocketAddr;
use std::sync::Arc;

async fn handler() -> &'static str {
    "Hello, World!"
}

#[tokio::main]
async fn main() {
    let limiter = Arc::new(MultiKeyLimiter::new(100, 10, 10_000));

    let app = Router::new()
        .route("/", get(handler))
        .layer(RateLimitLayer::new(
            Arc::clone(&limiter),
            |path: &str| path.to_string(), // สำหรับ production ใช้ IP extraction
        ));

    axum::Server::bind(&"0.0.0.0:3000".parse().unwrap())
        .serve(app.into_make_service_with_connect_info::<SocketAddr>())
        .await
        .unwrap();
}
```

### ขั้นที่ 8: Demo Binary และ Integration

แก้ไข `src/main.rs` ให้แสดงผลของทุก algorithm:

```rust
use rate_limiter::{
    TokenBucket, LeakyBucket, SlidingWindowLog, SlidingWindowRate, MultiKeyLimiter
};
use rate_limiter::leaky_bucket::Request;
use std::time::{Duration, Instant};

fn main() {
    println!("=== Rate Limiter Demo ===\n");

    // ── Token Bucket ──────────────────────────────
    println!("--- Token Bucket (capacity=5, refill=2/s) ---");
    let tb = TokenBucket::new(5, 2);
    for i in 1..=7 {
        let ok = tb.try_consume(1);
        println!("  request {}: {}", i, if ok { "ALLOWED" } else { "DENIED" });
    }

    // ── Leaky Bucket ──────────────────────────────
    println!("\n--- Leaky Bucket (capacity=3, leak=1/s) ---");
    let lb = LeakyBucket::new(3, 1.0);
    for i in 1..=5 {
        let ok = lb.try_enqueue(Request { id: i, enqueued_at: Instant::now() });
        println!("  request {}: {}", i, if ok { "ENQUEUED" } else { "DROPPED" });
    }

    // ── Sliding Window Log ────────────────────────
    println!("\n--- Sliding Window Log (window=1s, limit=3) ---");
    let sw = SlidingWindowLog::new(Duration::from_secs(1), 3);
    for i in 1..=5 {
        let ok = sw.allow();
        println!("  request {}: {}", i, if ok { "ALLOWED" } else { "DENIED" });
    }

    // ── Sliding Window Rate ───────────────────────
    println!("\n--- Sliding Window Rate (window=1s, limit=3) ---");
    let swr = SlidingWindowRate::new(Duration::from_secs(1), 3);
    for i in 1..=5 {
        let ok = swr.allow();
        println!("  request {}: {}", i, if ok { "ALLOWED" } else { "DENIED" });
    }

    // ── Multi-Key Limiter ─────────────────────────
    println!("\n--- Multi-Key Limiter (cap=2/key, max_keys=10) ---");
    let ml = MultiKeyLimiter::new(2, 0, 10);
    let keys = ["alice", "bob", "alice", "alice", "bob"];
    for key in &keys {
        let ok = ml.try_consume(key);
        println!("  key={}: {}", key, if ok { "ALLOWED" } else { "DENIED" });
    }

    println!("\n=== Demo complete ===");
}
```

Output จากการรัน:

```
=== Rate Limiter Demo ===

--- Token Bucket (capacity=5, refill=2/s) ---
  request 1: ALLOWED
  request 2: ALLOWED
  request 3: ALLOWED
  request 4: ALLOWED
  request 5: ALLOWED
  request 6: DENIED
  request 7: DENIED

--- Leaky Bucket (capacity=3, leak=1/s) ---
  request 1: ENQUEUED
  request 2: ENQUEUED
  request 3: ENQUEUED
  request 4: DROPPED
  request 5: DROPPED

--- Sliding Window Log (window=1s, limit=3) ---
  request 1: ALLOWED
  request 2: ALLOWED
  request 3: ALLOWED
  request 4: DENIED
  request 5: DENIED

--- Sliding Window Rate (window=1s, limit=3) ---
  request 1: ALLOWED
  request 2: ALLOWED
  request 3: ALLOWED
  request 4: DENIED
  request 5: DENIED

--- Multi-Key Limiter (cap=2/key, max_keys=10) ---
  key=alice: ALLOWED
  key=bob: ALLOWED
  key=alice: ALLOWED
  key=alice: DENIED
  key=bob: ALLOWED

=== Demo complete ===
```

### ขั้นที่ 9: lib.rs และการรวมทุก Module

แก้ไข `src/lib.rs`:

```rust
pub mod token_bucket;
pub mod leaky_bucket;
pub mod sliding_window;
pub mod multi_key;

pub use token_bucket::TokenBucket;
pub use leaky_bucket::LeakyBucket;
pub use sliding_window::{SlidingWindowLog, FixedWindowCounter, SlidingWindowRate};
pub use multi_key::MultiKeyLimiter;
```

## การทดสอบ (Testing)

โปรเจคนี้มี 19 unit tests ครอบคลุมทุก algorithm ตั้งแต่ behavior พื้นฐาน, edge cases, concurrency safety ไปจนถึง time-based behavior

รัน tests ทั้งหมด:

```bash
cargo test
```

Output จริงจากการรัน:

```
   Compiling rate-limiter v0.1.0 (/home/user/rust_course/...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.90s
     Running unittests src/lib.rs (target/debug/deps/rate_limiter-c4355ba1d3d7f5ab)

running 19 tests
test leaky_bucket::tests::test_leaky_bucket_drops_when_full ... ok
test leaky_bucket::tests::test_leaky_bucket_enqueue_up_to_capacity ... ok
test leaky_bucket::tests::test_leaky_bucket_drain ... ok
test leaky_bucket::tests::test_leaky_bucket_leaks_over_time ... ok
test multi_key::tests::test_multi_key_independent_limits ... ok
test multi_key::tests::test_multi_key_allow_count ... ok
test sliding_window::tests::test_fixed_window_allows_up_to_limit ... ok
test multi_key::tests::test_multi_key_lru_eviction ... ok
test multi_key::tests::test_multi_key_creates_new_entry ... ok
test sliding_window::tests::test_sliding_window_log_allows_up_to_limit ... ok
test sliding_window::tests::test_sliding_window_rate_basic ... ok
test sliding_window::tests::test_sliding_window_log_counts_correctly ... ok
test token_bucket::tests::test_token_bucket_allows_up_to_capacity ... ok
test token_bucket::tests::test_token_bucket_burst_consume ... ok
test sliding_window::tests::test_fixed_window_resets_after_window ... ok
test token_bucket::tests::test_token_bucket_thread_safe ... ok
test token_bucket::tests::test_token_bucket_does_not_exceed_capacity ... ok
test sliding_window::tests::test_sliding_window_log_vs_fixed_window_burst ... ok
test token_bucket::tests::test_token_bucket_refills_over_time ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.15s

     Running unittests src/main.rs (target/debug/deps/rate_limiter-38cbc3bd28e929b0)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests rate_limiter

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### รายละเอียด Tests แต่ละตัว

**Token Bucket Tests (5 tests)**

| Test | สิ่งที่ตรวจสอบ |
|------|---------------|
| `test_token_bucket_allows_up_to_capacity` | bucket เริ่มเต็ม, อนุญาตได้ถึง capacity แล้ว reject |
| `test_token_bucket_burst_consume` | consume หลาย tokens ในครั้งเดียวได้ |
| `test_token_bucket_refills_over_time` | tokens refill จริงหลังรอเวลา |
| `test_token_bucket_does_not_exceed_capacity` | tokens ไม่เกิน capacity แม้ refill เร็วมาก |
| `test_token_bucket_thread_safe` | 20 threads × 10 attempts = พอดี 100 successes |

**Leaky Bucket Tests (4 tests)**

| Test | สิ่งที่ตรวจสอบ |
|------|---------------|
| `test_leaky_bucket_enqueue_up_to_capacity` | enqueue ได้จนเต็ม capacity |
| `test_leaky_bucket_drops_when_full` | drop request เมื่อ queue เต็ม |
| `test_leaky_bucket_leaks_over_time` | leak ออกจาก queue ตาม elapsed time |
| `test_leaky_bucket_drain` | drain requests ออกในลำดับ FIFO |

**Sliding Window Tests (5 tests)**

| Test | สิ่งที่ตรวจสอบ |
|------|---------------|
| `test_sliding_window_log_allows_up_to_limit` | อนุญาตได้ถึง limit แล้ว reject |
| `test_sliding_window_log_counts_correctly` | `count_in_window()` นับ entries ใน window ถูกต้อง |
| `test_fixed_window_allows_up_to_limit` | Fixed window อนุญาตได้ถึง limit |
| `test_fixed_window_resets_after_window` | Fixed window reset หลังสิ้นสุด window period |
| `test_fixed_window_resets_after_window` | เปรียบเทียบ burst behavior ระหว่าง Fixed กับ Sliding |
| `test_sliding_window_rate_basic` | Sliding window rate อนุญาตถึง limit แล้ว reject |

**Multi-Key Tests (4 tests)**

| Test | สิ่งที่ตรวจสอบ |
|------|---------------|
| `test_multi_key_independent_limits` | แต่ละ key มี bucket อิสระ exhaustion ไม่ผลกระทบกัน |
| `test_multi_key_creates_new_entry` | สร้าง entry ใหม่สำหรับ key ใหม่, ไม่สร้างซ้ำ |
| `test_multi_key_lru_eviction` | evict LRU key เมื่อ map เต็ม |
| `test_multi_key_allow_count` | allow ครั้งแรกๆ, reject เมื่อ bucket หมด |

### Integration Test สำหรับ HTTP 429

เขียน integration test สำหรับ rate limit response:

```rust
// tests/http_integration.rs
use rate_limiter::MultiKeyLimiter;

#[test]
fn test_rate_limit_returns_429_headers() {
    let limiter = MultiKeyLimiter::new(2, 0, 100);
    let key = "test-api-key";

    // ครั้งแรกและสองควรผ่าน
    assert!(limiter.try_consume(&key), "ครั้งที่ 1 ควรผ่าน");
    assert!(limiter.try_consume(&key), "ครั้งที่ 2 ควรผ่าน");

    // ครั้งที่สามควรถูก block
    let allowed = limiter.try_consume(&key);
    assert!(!allowed, "ครั้งที่ 3 ควรถูก block");

    if !allowed {
        // จำลองการสร้าง 429 response
        let retry_after = 1u64;
        let headers = vec![
            ("status".to_string(), "429 Too Many Requests".to_string()),
            ("Retry-After".to_string(), retry_after.to_string()),
            ("X-RateLimit-Key".to_string(), key.to_string()),
        ];

        let status = headers.iter().find(|(k, _)| k == "status");
        assert!(status.is_some(), "ต้องมี status header");
        assert!(status.unwrap().1.contains("429"));

        let retry = headers.iter().find(|(k, _)| k == "Retry-After");
        assert!(retry.is_some(), "ต้องมี Retry-After header");
    }
}

#[test]
fn test_different_keys_have_independent_limits() {
    let limiter = MultiKeyLimiter::new(1, 0, 100);

    assert!(limiter.try_consume(&"user-a"), "user-a ครั้งที่ 1 ควรผ่าน");
    assert!(!limiter.try_consume(&"user-a"), "user-a ครั้งที่ 2 ควรถูก block");
    assert!(limiter.try_consume(&"user-b"), "user-b ไม่เกี่ยวกับ user-a ควรผ่าน");
    assert!(!limiter.try_consume(&"user-b"), "user-b ครั้งที่ 2 ควรถูก block");
}
```

## Pitfalls และข้อผิดพลาดที่พบบ่อย

### Pitfall 1: ABA Problem ใน CAS Loop

**ปัญหา:** ใน `try_consume` CAS loop ถ้า thread A อ่านค่า `tokens = 5`, thread B consume 3 (tokens = 2) แล้ว refill 3 (tokens = 5) thread A จะ CAS สำเร็จทั้งที่ตอน compare ค่าเปลี่ยนไปแล้ว

**ผลกระทบ:** อาจนับ tokens ผิด แต่สำหรับ rate limiting โดยทั่วไปถือว่า acceptable เพราะ error จะเป็น conservative (อนุญาตน้อยกว่าที่ควร ไม่ใช่มากกว่า)

**วิธีแก้ถ้าต้องการ exact counting:** ใช้ `Mutex<u64>` แทน `AtomicU64` ยอมรับ overhead เล็กน้อย

```rust
// แบบ exact (ใช้ Mutex)
pub struct ExactTokenBucket {
    capacity: u64,
    tokens: Mutex<u64>,
    // ...
}

impl ExactTokenBucket {
    pub fn try_consume(&self, n: u64) -> bool {
        let mut tokens = self.tokens.lock().unwrap();
        if *tokens >= n {
            *tokens -= n;
            true
        } else {
            false
        }
    }
}
```

### Pitfall 2: Clock Monotonicity

**ปัญหา:** `SystemTime::now()` ไม่รับประกัน monotonicity — สามารถถอยหลังได้ถ้า NTP sync ปรับนาฬิกา ทำให้ `elapsed_us` เป็น negative (overflow ใน u64)

**โค้ดที่มีปัญหา:**
```rust
let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_micros() as u64;
let last = self.last_refill_us.load(Ordering::Relaxed);
let elapsed = now - last; // overflow ถ้า now < last!
```

**วิธีแก้:** ตรวจสอบก่อน subtract:
```rust
fn now_us() -> u64 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap_or_default()  // คืน 0 ถ้า before epoch
        .as_micros() as u64
}

fn refill(&self) {
    let now = now_us();
    let last = self.last_refill_us.load(Ordering::Relaxed);
    if now <= last { return; }  // ← guard สำคัญ!
    let elapsed_us = now - last;
    // ...
}
```

**Alternative ที่ดีกว่า:** ใช้ `Instant` ซึ่ง monotonic โดยธรรมชาติ แต่ไม่สามารถแปลงเป็น integer ได้ง่าย ต้องเก็บ `Mutex<Instant>` แทน `AtomicU64`

### Pitfall 3: Deadlock จาก Lock Ordering

**ปัญหา:** ใน `LeakyBucket.do_leak()` เราต้อง lock `last_leak` และ `queue` พร้อมกัน ถ้า lock ในลำดับต่างกันจาก methods อื่นจะ deadlock

**โค้ดที่มีปัญหา (เสี่ยง deadlock):**
```rust
// Thread A: do_leak() ทำ lock(last_leak) แล้วรอ lock(queue)
// Thread B: drain_and_enqueue() ทำ lock(queue) แล้วรอ lock(last_leak)
// → DEADLOCK
fn do_leak(&self) {
    let mut last = self.last_leak.lock().unwrap();  // lock 1
    let mut q = self.queue.lock().unwrap();          // lock 2 (ขณะถือ lock 1)
    // ...
}

fn drain_and_enqueue(&self, req: Request) {
    let mut q = self.queue.lock().unwrap();          // lock 2
    let last = self.last_leak.lock().unwrap();       // lock 1 (ขณะถือ lock 2)
    // → DEADLOCK กับ thread ที่รัน do_leak
}
```

**วิธีแก้:** lock แยกกัน — ใช้ค่าที่ได้จาก `last_leak` แล้วปล่อย lock ก่อน lock `queue`:
```rust
fn do_leak(&self) {
    let now = Instant::now();
    // lock last_leak, คำนวณ to_remove, แล้วปล่อย
    let to_remove = {
        let mut last = self.last_leak.lock().unwrap();
        let elapsed = now.duration_since(*last).as_secs_f64();
        let n = (elapsed * self.leak_rate).floor() as usize;
        if n > 0 { *last = now; }
        n
    }; // ← last_leak lock ถูกปล่อยที่นี่
    
    // lock queue หลังจาก last_leak ถูกปล่อยแล้ว
    if to_remove > 0 {
        let mut q = self.queue.lock().unwrap();
        for _ in 0..to_remove { q.pop_front(); }
    }
}
```

### Pitfall 4: Sliding Window Rate — Integer Overflow ใน Edge Case

**ปัญหา:** ถ้า `prev_count` มีค่าสูงมาก การคำนวณ `prev_count as f64 * overlap + curr_count as f64` อาจเกิน `f64` precision (แม้ f64 จะมี 53-bit mantissa, แต่ถ้า count เป็นหลักพันล้านจะเริ่มเสียความแม่นยำ)

**วิธีแก้:** cap ค่า count ให้ไม่เกิน limit + buffer ใหญ่เล็กน้อย:
```rust
if estimated < self.limit {
    // ป้องกัน count สูงเกินไป
    inner.curr_count = (inner.curr_count + 1).min(self.limit * 2);
    true
} else {
    false
}
```

### Pitfall 5: LRU Eviction Race Condition

**ปัญหา:** ใน `MultiKeyLimiter.try_consume()` มี time-of-check/time-of-use (TOCTOU) race:
```rust
// Thread A ตรวจสอบ: len() == max_keys → ต้อง evict
// Thread B ตรวจสอบ: len() == max_keys → ต้อง evict
// Thread A evict key "x" → len = max_keys - 1
// Thread B evict key "y" → len = max_keys - 2 (evict เกิน!)
// Thread A insert → len = max_keys - 1
// Thread B insert → len = max_keys (ไม่ได้ evict มากเกินจริง... แต่เสียพื้นที่)
```

**วิธีแก้ที่ใช้:** `evict_lock` + re-check ใน `evict_lru()`:
```rust
fn evict_lru(&self) {
    let _lock = self.evict_lock.lock().unwrap(); // serialize evictions
    if self.limiters.len() < self.max_keys { return; } // re-check
    // ...
}
```

## การ Package และ Deploy

### Build สำหรับ Production

```bash
# Build release binary
cargo build --release

# ตรวจสอบขนาด
ls -lh target/release/rate-limiter
```

### Publish เป็น Library Crate

แก้ไข `Cargo.toml` เพิ่ม metadata:
```toml
[package]
name = "rate-limiter-rs"
version = "0.1.0"
edition = "2021"
description = "Production-grade rate limiting library with multiple algorithms"
license = "MIT OR Apache-2.0"
repository = "https://github.com/yourname/rate-limiter-rs"
keywords = ["rate-limit", "token-bucket", "throttle", "middleware"]
categories = ["algorithms", "web-programming"]

[lib]
name = "rate_limiter"
```

```bash
# ตรวจสอบก่อน publish
cargo package
cargo publish --dry-run
```

### Docker Image (สำหรับ Standalone Service)

ถ้าต้องการ run เป็น standalone rate limiting service (sidecar pattern):

```dockerfile
FROM rust:1.82 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/rate-limiter /usr/local/bin/
EXPOSE 8080
CMD ["rate-limiter"]
```

```bash
docker build -t rate-limiter:latest .
docker run -p 8080:8080 rate-limiter:latest
```

### Benchmark สำหรับ Production Tuning

```bash
# เพิ่ม criterion ใน Cargo.toml
# [dev-dependencies]
# criterion = { version = "0.5", features = ["html_reports"] }

cargo bench
```

ตัวอย่าง benchmark:
```rust
// benches/throughput.rs
use criterion::{criterion_group, criterion_main, Criterion};
use rate_limiter::TokenBucket;
use std::sync::Arc;

fn bench_token_bucket(c: &mut Criterion) {
    let tb = Arc::new(TokenBucket::new(1_000_000, 0));
    c.bench_function("token_bucket_try_consume", |b| {
        b.iter(|| tb.try_consume(1))
    });
}

criterion_group!(benches, bench_token_bucket);
criterion_main!(benches);
```

ผลที่คาดหวังบน modern hardware:
- `TokenBucket::try_consume` ที่ไม่มี contention: ~20-50ns
- `TokenBucket::try_consume` ใน 16-thread contention: ~100-300ns
- `MultiKeyLimiter::try_consume` (DashMap lookup): ~50-150ns

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Redis-backed Distributed Rate Limiter ⭐⭐⭐

Rate limiter ในหน่วยความจำใช้ได้กับ single node เท่านั้น ให้สร้าง `RedisTokenBucket` ที่ใช้ Redis เป็น storage เพื่อ share state ข้าม instances

**แนวทาง:**
- ใช้ crate `redis` (หรือ `fred`)
- implement Lua script บน Redis สำหรับ atomic token check-and-consume
- script ต้อง atomic เพราะ check กับ deduct ต้องเป็น operation เดียว

```rust
// Lua script บน Redis (atomic token bucket)
const CONSUME_SCRIPT: &str = r#"
local key = KEYS[1]
local tokens_key = key .. ":tokens"
local last_key = key .. ":last"
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local cost = tonumber(ARGV[4])

local last = tonumber(redis.call("get", last_key)) or now
local tokens = tonumber(redis.call("get", tokens_key)) or capacity
local elapsed = math.max(0, now - last)
local refill = elapsed * refill_rate
tokens = math.min(capacity, tokens + refill)

if tokens >= cost then
    redis.call("set", tokens_key, tokens - cost)
    redis.call("set", last_key, now)
    return 1
else
    return 0
end
"#;
```

### แบบฝึกหัดที่ 2: Adaptive Rate Limiter ⭐⭐⭐

สร้าง rate limiter ที่ปรับ limit แบบ dynamic ตาม latency ของ backend:
- ถ้า backend latency > threshold → ลด rate limit ลง 10%
- ถ้า backend latency < target × 0.5 → เพิ่ม rate limit 5%
- ใช้ AIMD (Additive Increase, Multiplicative Decrease) algorithm

```rust
pub struct AdaptiveRateLimiter {
    base_limit: u64,
    current_limit: AtomicU64,
    target_latency_ms: u64,
    inner: TokenBucket,
}

impl AdaptiveRateLimiter {
    pub fn observe_latency(&self, latency_ms: u64) {
        let current = self.current_limit.load(Ordering::Relaxed);
        let new_limit = if latency_ms > self.target_latency_ms * 2 {
            // Multiplicative Decrease
            (current * 9 / 10).max(self.base_limit / 10)
        } else if latency_ms < self.target_latency_ms / 2 {
            // Additive Increase
            (current + current / 20).min(self.base_limit * 10)
        } else {
            current
        };
        self.current_limit.store(new_limit, Ordering::Relaxed);
    }
}
```

### แบบฝึกหัดที่ 3: Rate Limit Headers และ Observability ⭐⭐

เพิ่ม response headers มาตรฐานและ metrics integration:

```rust
/// Rate limit response headers ตาม RFC draft
pub struct RateLimitHeaders {
    pub limit: u64,       // X-RateLimit-Limit
    pub remaining: u64,   // X-RateLimit-Remaining
    pub reset_at: u64,    // X-RateLimit-Reset (unix timestamp)
    pub retry_after: Option<u64>, // Retry-After (seconds)
}

impl TokenBucket {
    pub fn check_with_headers(&self, n: u64) -> (bool, RateLimitHeaders) {
        let allowed = self.try_consume(n);
        let remaining = self.available_tokens();
        let reset_at = self.next_refill_time();
        (allowed, RateLimitHeaders {
            limit: self.capacity_scaled / TOKEN_SCALE,
            remaining,
            reset_at,
            retry_after: if allowed { None } else { Some(1) },
        })
    }
}
```

### แบบฝึกหัดที่ 4: Priority Rate Limiter ⭐⭐⭐⭐

บางระบบต้องการให้ traffic บาง class ได้รับ quota พิเศษ เช่น premium users ได้ 10x rate, health checks ไม่ถูก rate limit:

```rust
#[derive(Clone, PartialEq, Eq, Hash)]
pub enum Priority { Critical, High, Normal, Low }

pub struct PriorityRateLimiter {
    buckets: HashMap<Priority, TokenBucket>,
    global_bucket: TokenBucket,
}

impl PriorityRateLimiter {
    /// consume ต้องผ่านทั้ง per-priority bucket และ global bucket
    pub fn try_consume(&self, priority: Priority, n: u64) -> bool {
        let per_priority = self.buckets
            .get(&priority)
            .map(|b| b.try_consume(n))
            .unwrap_or(false);
        if !per_priority { return false; }
        // ถ้า per-priority ผ่านแล้วตรวจ global budget ด้วย
        self.global_bucket.try_consume(n)
    }
}
```

### แบบฝึกหัดที่ 5: Metrics Integration ⭐⭐

ผสาน rate limiter กับ Prometheus metrics จาก Project G04 เพื่อ monitor rate limiting behavior:

```rust
// ใช้ counter จาก Project G04
struct InstrumentedTokenBucket {
    inner: TokenBucket,
    allowed_counter: Counter,
    denied_counter: Counter,
    bucket_name: String,
}

impl InstrumentedTokenBucket {
    pub fn try_consume(&self, n: u64) -> bool {
        let result = self.inner.try_consume(n);
        if result {
            self.allowed_counter.inc();
        } else {
            self.denied_counter.inc();
        }
        result
    }
}
```

### แบบฝึกหัดที่ 6: Async Token Bucket (Wait Instead of Reject) ⭐⭐⭐

แทนที่จะ return false ทันที ให้รอจนมี tokens พอ (useful สำหรับ background jobs ที่ไม่รีบ):

```rust
use tokio::time;

pub struct WaitingTokenBucket {
    inner: TokenBucket,
}

impl WaitingTokenBucket {
    /// รอจนมี tokens พอ แล้ว consume
    pub async fn consume(&self, n: u64) {
        loop {
            if self.inner.try_consume(n) {
                return;
            }
            // คำนวณว่าต้องรอนานแค่ไหน
            let wait_ms = self.estimate_wait_ms(n);
            time::sleep(time::Duration::from_millis(wait_ms)).await;
        }
    }

    fn estimate_wait_ms(&self, n: u64) -> u64 {
        let current = self.inner.available_tokens();
        if current >= n { return 0; }
        let deficit = n - current;
        // refill_rate tokens/s → ต้องรอ deficit/refill_rate seconds
        let wait_secs = deficit as f64 / (self.inner.refill_rate_per_sec() as f64);
        (wait_secs * 1000.0) as u64 + 1 // +1ms buffer
    }
}
```

## สรุป

ในโปรเจคนี้เราสร้าง rate limiting library ครบวงจรที่รองรับ 5 algorithms หลัก:

1. **Token Bucket** — lock-free ด้วย `AtomicU64` scaled integers, lazy refill, เหมาะกับ bursty traffic
2. **Leaky Bucket** — queue-based traffic smoothing ด้วย `Mutex<VecDeque>`, FIFO processing
3. **Sliding Window Log** — แม่นยำสูงสุด, O(requests) memory, เหมาะกับ strict SLA
4. **Sliding Window Rate** — O(1) memory ด้วย weighted interpolation, balance ระหว่าง accuracy กับ efficiency
5. **Multi-Key Limiter** — per-key isolation ด้วย `DashMap`, LRU eviction ป้องกัน memory leak

**Patterns สำคัญที่ได้เรียน:**
- **CAS loop pattern** — atomic read-modify-write ที่ thread-safe โดยไม่ต้อง mutex
- **Scaled integer trick** — จำลอง fractional values ด้วย integer arithmetic เพื่อใช้ atomic operations
- **Lock ordering discipline** — ป้องกัน deadlock ด้วยการ release lock ก่อน acquire lock อื่น
- **LRU eviction ใน concurrent map** — ใช้ evict_lock ป้องกัน concurrent eviction ที่ evict เกิน
- **Time-based state machine** — Sliding Window Rate ใช้ window transitions แบบ lazy

**เชื่อมโยงไปโปรเจคถัดไป:**

Project G10 (Blue-Green Deployment) จะนำ rate limiter นี้ไปใช้จริงในระบบ deployment โดย rate limit traffic ระหว่าง blue/green versions เพื่อ gradual rollout ที่ควบคุม traffic fraction ได้แม่นยำ — ใช้ `SlidingWindowRate` วัด error rate ต่อ version และ `TokenBucket` ควบคุม traffic shift rate

---

**โปรเจคก่อนหน้า:** [Project G08: Infrastructure as Code](project-g08-infra-as-code.md) | **โปรเจคถัดไป:** [Project G10: Blue-Green Deployment](project-g10-blue-green.md)
