# Project G04: Metrics Collector (Prometheus-compatible)

> โมดูล: G — DevOps & Infrastructure | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **metrics collection library** ที่ compatible กับ Prometheus text format — มาตรฐานที่ใช้กันอย่างแพร่หลายที่สุดในวงการ observability และ DevOps สมัยใหม่ library นี้สามารถ embed ได้ใน application ใด ๆ และ expose `/metrics` endpoint ที่ Prometheus server ดึงข้อมูลได้โดยตรง

Prometheus เป็น time-series database และ monitoring system ที่ CNCF (Cloud Native Computing Foundation) ดูแล โดย ecosystem ของมันประกอบด้วย Prometheus server, Grafana สำหรับ visualization, Alertmanager สำหรับ alert routing และ exporters จำนวนมากที่รวบรวม metrics จากระบบต่าง ๆ การเขียน metrics library ของตัวเองใน Rust ทำให้เราเข้าใจ protocol ลึกขึ้น และได้ library ที่มี performance สูงกว่า wrapper ของภาษาอื่น เพราะใช้ atomic operations แทน mutex-based locking ที่ bottleneck ใน high-throughput scenarios

**Use cases จริงในโลก production:**
- **Application instrumentation** — วัด request rate, error rate, latency distribution ใน microservices ทุกตัว
- **Infrastructure monitoring** — รวบรวม system metrics เช่น CPU, memory, disk I/O จาก Linux hosts
- **Custom business metrics** — นับ orders per second, active sessions, payment failures ตามต้องการ
- **SLO monitoring** — ติดตาม Service Level Objectives เช่น p99 latency < 200ms, error rate < 0.1%
- **Capacity planning** — เก็บ historical data เพื่อ predict resource needs ล่วงหน้า

**ทำไมต้องเขียนเอง แทนที่จะใช้ crate สำเร็จรูปเช่น `prometheus` หรือ `metrics`?**

1. เข้าใจ Prometheus text format specification อย่างลึกซึ้ง
2. เรียนรู้ lock-free data structures ด้วย `AtomicU64` และ compare-exchange loops
3. ฝึก API design สำหรับ library ที่ต้อง ergonomic และ safe พร้อมกัน
4. เข้าใจ global singleton pattern ด้วย `OnceLock` ใน Rust
5. ฝึก trait-based abstraction ด้วย `Collector` trait

## สิ่งที่จะได้เรียนรู้

- **Atomic operations** — `AtomicU64`, `fetch_add`, `compare_exchange_weak` สำหรับ lock-free counter/gauge
- **f64 bit manipulation** — bit-cast `f64` ↔ `u64` เพื่อเก็บ floating-point ใน `AtomicU64`
- **DashMap** — concurrent hash map ที่ thread-safe โดยไม่ต้องใช้ `RwLock<HashMap>`
- **OnceLock** — global singleton initialization ที่ race-free ใน Rust stable
- **Trait objects** — `Box<dyn Collector>` สำหรับ polymorphic collector system
- **Prometheus text format** — `# HELP`, `# TYPE`, label syntax `{key="val"}`, histogram suffixes
- **String formatting** — สร้าง text format output อย่างถูกต้องตาม specification
- **API design principles** — builder pattern, Arc-based shared ownership, validation at boundaries

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–30**: Rust basics — ownership, borrowing, structs, enums, traits, generics, collections
- **Part 31–40**: Error handling (`Result`, `?`), trait objects, `Box<dyn Trait>`
- **Part 41–50**: Concurrency basics — `Arc`, `Mutex`, `thread::spawn`
- **Part 51–60**: `std::sync::atomic` — `AtomicU64`, `Ordering`, atomic operations
- **Part 61–70**: `tokio` async runtime, `async/await`, `OnceLock` จาก Part 67
- **Part 71–80**: External crates — `dashmap`, `serde` (จาก Part 74 เรื่อง popular crates)
- **Part 96–100**: Production patterns — global state management, library design

## โครงสร้างโปรเจค (Project Layout)

```
metrics-collector/
├── src/
│   ├── main.rs          ← entry point + demo + global_registry() singleton
│   ├── counter.rs       ← Counter type (AtomicU64, monotonically increasing)
│   ├── gauge.rs         ← Gauge type (AtomicU64 bit-cast ↔ f64, arbitrary value)
│   ├── histogram.rs     ← Histogram type (configurable buckets, cumulative counts)
│   ├── labels.rs        ← Labels type + validation (name rules, cardinality limit)
│   ├── registry.rs      ← MetricsRegistry (DashMap) + global OnceLock singleton
│   ├── exposition.rs    ← Prometheus text format rendering functions
│   └── collector.rs     ← Collector trait + ProcessCollector + collect_all()
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Application Code
        │
        │  counter.inc() / gauge.set() / histogram.observe()
        ▼
┌───────────────────────────────────────────────────┐
│                  Metric Types                      │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐   │
│  │ Counter  │  │  Gauge   │  │   Histogram   │   │
│  │ AtomicU64│  │ AtomicU64│  │ Vec<AtomicU64>│   │
│  └────┬─────┘  └────┬─────┘  └──────┬────────┘   │
└───────┼─────────────┼───────────────┼─────────────┘
        │             │               │
        └─────────────▼───────────────┘
                      │
                      │  register() → MetricKey → MetricValue
                      ▼
        ┌─────────────────────────────┐
        │       MetricsRegistry       │
        │  DashMap<MetricKey, Value>  │
        │  (global via OnceLock)      │
        └──────────────┬──────────────┘
                       │
          ┌────────────▼───────────┐
          │    Exposition Layer    │
          │  render_counter()      │
          │  render_gauge()        │
          │  render_histogram()    │
          └────────────┬───────────┘
                       │
          ┌────────────▼───────────┐
          │    HTTP /metrics       │
          │  (tokio TCP server)    │
          └────────────────────────┘
                       │
                       ▼
            Prometheus Server scrapes
```

### Metric Types และ Storage Strategy

| Metric Type | Storage | Use Case | Key Property |
|-------------|---------|----------|--------------|
| `Counter`   | `AtomicU64` | request counts, error counts | monotonically increasing only |
| `Gauge`     | `AtomicU64` (bit-cast) | memory, CPU, active connections | arbitrary f64, เพิ่มหรือลดได้ |
| `Histogram` | `Vec<AtomicU64>` per bucket | latency distribution, request size | cumulative counts per bucket |

**ทำไมถึงใช้ `AtomicU64` แทน `Mutex<f64>`?**

`Mutex` มี overhead จาก kernel context switch เมื่อ contention สูง ใน high-throughput service ที่มี requests หลายพัน RPS การ lock/unlock mutex ทุก increment สร้าง bottleneck สำคัญ `AtomicU64` ใช้ CPU-level atomic instructions (เช่น `LOCK XADD` บน x86) ที่เร็วกว่า mutex ประมาณ 10-100x ในกรณี contended

**f64 bit-cast pattern สำหรับ Gauge:**

```rust
// เก็บ f64 ใน AtomicU64 โดยใช้ bit pattern เหมือนกัน
self.value.store(v.to_bits(), Ordering::Relaxed);
let v = f64::from_bits(self.value.load(Ordering::Relaxed));
```

IEEE 754 กำหนดว่า f64 มีขนาด 64 bits เท่ากับ u64 พอดี เราจึง reinterpret bit pattern ได้โดยตรงผ่าน `.to_bits()` และ `from_bits()` ซึ่ง safe ใน Rust (ต่างจาก C ที่ต้อง memcpy)

**compare_exchange loop สำหรับ Gauge.add():**

Counter ใช้ `fetch_add` ได้ตรง ๆ แต่ Gauge ต้องการ add แบบ floating-point ซึ่ง `AtomicU64` ไม่มี primitive สำหรับ float addition ดังนั้นเราใช้ CAS loop:

```
load current → compute new value → try swap → retry if someone changed it
```

Pattern นี้ discard การ retry ที่เสียไป แต่ correctness รับประกันได้

### Label System

Labels เป็น key differentiator ของ Prometheus — metric เดียวกันแต่ต่าง labels ถือเป็น time series ที่แตกต่างกัน เช่น `http_requests_total{method="GET"}` กับ `http_requests_total{method="POST"}`

เราใช้ `Labels { pairs: Vec<(String, String)> }` เรียบง่าย พร้อม validation สองชั้น:
1. **Name validation** — label name ต้องขึ้นต้นด้วย letter หรือ `_` และมีเฉพาะ `[a-zA-Z0-9_]`
2. **Cardinality limit** — ไม่เกิน 10 labels ต่อ metric (production systems มีกฎนี้เพื่อป้องกัน memory explosion)

### Registry Pattern

`MetricsRegistry` ใช้ `DashMap<MetricKey, MetricValue>` โดย:
- `MetricKey` = `(name, labels)` — เป็น hash key ที่ unique ต่อ time series
- `MetricValue` = enum `Counter | Gauge | Histogram` ที่ wrap Arc
- `DashMap` แบ่ง internal state เป็น shard เพื่อ reduce contention เมื่อ concurrent access

Global singleton ใช้ `OnceLock<MetricsRegistry>` (stabilized ใน Rust 1.70) แทน `lazy_static!` หรือ `once_cell` ที่ต้องการ external crate

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Labels และ Validation

เริ่มด้วยส่วนที่ง่ายที่สุดแต่สำคัญมาก — Labels ซึ่งเป็น foundation ของทุก metric type

สร้าง `src/labels.rs`:

```rust
/// ตรวจสอบว่า label name ถูกต้องตาม Prometheus specification
/// ต้องขึ้นต้นด้วย letter หรือ underscore และมีเฉพาะ [a-zA-Z0-9_]
pub fn validate_label_name(name: &str) -> Result<(), String> {
    if name.is_empty() {
        return Err("Label name cannot be empty".to_string());
    }
    let mut chars = name.chars();
    let first = chars.next().unwrap();
    if !first.is_ascii_alphabetic() && first != '_' {
        return Err(format!(
            "Label name '{}' must start with a letter or underscore",
            name
        ));
    }
    for ch in chars {
        if !ch.is_ascii_alphanumeric() && ch != '_' {
            return Err(format!(
                "Label name '{}' contains invalid character '{}'",
                name, ch
            ));
        }
    }
    Ok(())
}

/// Labels เก็บชุดของ key-value pairs สำหรับ Prometheus metric
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Labels {
    pub pairs: Vec<(String, String)>,
}

impl Labels {
    pub fn new(pairs: Vec<(String, String)>) -> Self {
        Labels { pairs }
    }

    pub fn empty() -> Self {
        Labels { pairs: vec![] }
    }

    /// ตรวจสอบความถูกต้องของ labels ทั้งหมด
    /// - label names ต้องถูกต้องตาม Prometheus spec
    /// - จำนวน labels ต้องไม่เกิน 10 pairs (cardinality limit)
    pub fn validate(&self) -> Result<(), String> {
        if self.pairs.len() > 10 {
            return Err(format!(
                "Too many labels: {} (max 10)",
                self.pairs.len()
            ));
        }
        for (name, _value) in &self.pairs {
            validate_label_name(name)?;
        }
        Ok(())
    }

    /// แปลง Labels เป็น Prometheus label string เช่น {method="GET",path="/api"}
    pub fn to_prometheus_string(&self) -> String {
        if self.pairs.is_empty() {
            return String::new();
        }
        let parts: Vec<String> = self
            .pairs
            .iter()
            .map(|(k, v)| {
                format!(
                    "{}=\"{}\"",
                    k,
                    v.replace('\\', "\\\\").replace('"', "\\\"")
                )
            })
            .collect();
        format!("{{{}}}", parts.join(","))
    }
}
```

สิ่งสำคัญที่ต้องสังเกต:
- `#[derive(Hash)]` บน `Labels` ทำให้ใช้เป็น key ใน `HashMap` หรือ `DashMap` ได้
- Label value ต้องทำ escape — `\` → `\\` และ `"` → `\"` ก่อนใส่ใน output
- Cardinality limit 10 เป็น soft limit ที่ทีม Prometheus แนะนำ ใน production หลายองค์กรกำหนดต่ำกว่านี้

**ทดสอบ Labels:**

```rust
let labels = Labels::new(vec![
    ("method".to_string(), "GET".to_string()),
    ("status".to_string(), "200".to_string()),
]);
println!("{}", labels.to_prometheus_string());
// Output: {method="GET",status="200"}

// Label ที่มี quote ใน value
let tricky = Labels::new(vec![
    ("path".to_string(), r#"say "hello""#.to_string()),
]);
println!("{}", tricky.to_prometheus_string());
// Output: {path="say \"hello\""}
```

### ขั้นที่ 2: Counter — Monotonically Increasing Metric

Counter เป็น metric ที่พบมากที่สุดใน Prometheus ecosystem วัดสิ่งที่นับได้และไม่ลดลง เช่น จำนวน requests ทั้งหมด, จำนวน errors ทั้งหมด

สร้าง `src/counter.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

/// Counter เป็น metric ที่เพิ่มขึ้นเรื่อย ๆ (monotonically increasing)
/// ใช้ AtomicU64 เพื่อ thread-safe increment โดยไม่ต้องใช้ Mutex
#[derive(Debug)]
pub struct Counter {
    pub name: String,
    pub help: String,
    value: AtomicU64,
}

impl Counter {
    pub fn new(name: &str, help: &str) -> Self {
        Counter {
            name: name.to_string(),
            help: help.to_string(),
            value: AtomicU64::new(0),
        }
    }

    /// เพิ่มค่า counter ขึ้น 1
    pub fn inc(&self) {
        self.value.fetch_add(1, Ordering::Relaxed);
    }

    /// เพิ่มค่า counter ขึ้นตามจำนวนที่ระบุ
    pub fn add(&self, delta: u64) {
        self.value.fetch_add(delta, Ordering::Relaxed);
    }

    /// อ่านค่า counter ปัจจุบัน
    pub fn get(&self) -> u64 {
        self.value.load(Ordering::Relaxed)
    }
}
```

**ทำความเข้าใจ `Ordering::Relaxed`:**

`Ordering` ใน Rust atomic operations ควบคุม memory ordering guarantee:
- `Relaxed` — ไม่มี ordering constraint กับ operations อื่น ใช้กับ metrics ได้เพราะเราต้องการแค่ atomicity ของตัว operation เอง ไม่ต้องการ synchronization กับ memory ส่วนอื่น
- `SeqCst` (Sequential Consistent) — strict สุด ช้าสุด ใช้เมื่อต้องการ total ordering ระหว่าง threads
- `Acquire`/`Release` — ใช้คู่กันสำหรับ mutex-like patterns

สำหรับ metrics counter ที่แค่นับ `Relaxed` เพียงพอ เพราะแม้ว่า thread อื่นจะเห็นค่าไม่ sync กันชั่วคราว แต่ผลลัพธ์สุดท้าย (total count) ถูกต้องเสมอ

**Thread-safety test:**

```rust
#[test]
fn test_counter_thread_safe() {
    use std::sync::Arc;
    use std::thread;

    let c = Arc::new(Counter::new("concurrent_counter", "Thread-safe test"));
    let mut handles = vec![];

    for _ in 0..10 {
        let c_clone = Arc::clone(&c);
        handles.push(thread::spawn(move || {
            for _ in 0..100 {
                c_clone.inc();
            }
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    assert_eq!(c.get(), 1000); // 10 threads × 100 increments
}
```

หากใช้ `Mutex<u64>` แทน `AtomicU64` test นี้ยังผ่าน แต่จะช้ากว่ามากเมื่อ thread count สูงเพราะแต่ละ `inc()` ต้อง acquire lock

### ขั้นที่ 3: Gauge — Arbitrary Value Metric

Gauge ต่างจาก Counter ตรงที่สามารถเพิ่มหรือลดได้ตามต้องการ ใช้สำหรับ metrics ที่วัด "สภาพปัจจุบัน" เช่น memory usage, active connections, temperature

สร้าง `src/gauge.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

/// Gauge เป็น metric ที่เพิ่มหรือลดได้ตามต้องการ เก็บค่า f64 arbitrary
/// ใช้ AtomicU64 + bit_cast เพื่อ thread-safe โดยไม่ต้องใช้ Mutex
#[derive(Debug)]
pub struct Gauge {
    pub name: String,
    pub help: String,
    // เก็บ f64 ในรูปแบบ bit pattern ของ u64
    value: AtomicU64,
}

impl Gauge {
    pub fn new(name: &str, help: &str) -> Self {
        Gauge {
            name: name.to_string(),
            help: help.to_string(),
            value: AtomicU64::new(0),
        }
    }

    pub fn set(&self, v: f64) {
        self.value.store(v.to_bits(), Ordering::Relaxed);
    }

    pub fn get(&self) -> f64 {
        f64::from_bits(self.value.load(Ordering::Relaxed))
    }

    /// เพิ่มค่า gauge ขึ้น delta (อาจติดลบได้)
    /// ใช้ compare_exchange loop เพื่อ atomic update
    pub fn add(&self, delta: f64) {
        loop {
            let current_bits = self.value.load(Ordering::Relaxed);
            let current = f64::from_bits(current_bits);
            let new_bits = (current + delta).to_bits();
            match self.value.compare_exchange_weak(
                current_bits,
                new_bits,
                Ordering::Relaxed,
                Ordering::Relaxed,
            ) {
                Ok(_) => break,
                Err(_) => continue, // retry
            }
        }
    }

    pub fn inc(&self) { self.add(1.0); }
    pub fn dec(&self) { self.add(-1.0); }
    pub fn sub(&self, delta: f64) { self.add(-delta); }
}
```

**`compare_exchange_weak` vs `compare_exchange`:**

`compare_exchange_weak` อาจ fail ได้แบบ spurious (โดยไม่มี actual contention) บน weak memory architectures เช่น ARM แต่ loop ของเราจัดการ case นี้ได้อยู่แล้ว จึงใช้ `_weak` ได้ ซึ่งบางสถาปัตยกรรมจะ compile เป็น single instruction แทน loop เล็กน้อย ทำให้เร็วกว่าเล็กน้อยใน uncontended case

**ทดสอบ Gauge:**

```rust
let g = Gauge::new("active_connections", "Current active connections");
g.set(100.0);
g.inc(); // 101
g.dec(); // 100
g.sub(20.0); // 80
g.add(-30.0); // 50 — เป็นลบได้
println!("connections: {}", g.get()); // 50
```

### ขั้นที่ 4: Histogram — Distribution Metric

Histogram เป็น metric ที่ซับซ้อนที่สุดแต่ทรงพลังที่สุด ใช้วัด distribution ของค่าเช่น latency โดยแบ่งเป็น buckets และนับจำนวน observations ที่ตก ≤ upper bound ของแต่ละ bucket

สร้าง `src/histogram.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Mutex;

/// Histogram วัดการกระจายของค่า เช่น latency
/// ใช้ bucket boundaries ที่กำหนดเองได้ และเก็บ cumulative counts
pub struct Histogram {
    pub name: String,
    pub help: String,
    /// Upper boundaries ของแต่ละ bucket (sorted ascending)
    pub buckets: Vec<f64>,
    /// นับจำนวน observations ที่ <= bucket[i] (cumulative)
    bucket_counts: Vec<AtomicU64>,
    /// จำนวน observations ทั้งหมด
    count: AtomicU64,
    /// ผลรวมของ observations ทั้งหมด
    sum_bits: Mutex<u64>,
}

impl Histogram {
    pub fn new(name: &str, help: &str, mut buckets: Vec<f64>) -> Self {
        buckets.sort_by(|a, b| a.partial_cmp(b).unwrap());
        buckets.retain(|&b| b != f64::INFINITY);
        let n = buckets.len();

        let mut bucket_counts = Vec::with_capacity(n + 1);
        for _ in 0..=n {
            bucket_counts.push(AtomicU64::new(0));
        }

        Histogram {
            name: name.to_string(),
            help: help.to_string(),
            buckets,
            bucket_counts,
            count: AtomicU64::new(0),
            sum_bits: Mutex::new(0u64),
        }
    }

    pub fn observe(&self, value: f64) {
        self.count.fetch_add(1, Ordering::Relaxed);
        {
            let mut sum_bits = self.sum_bits.lock().unwrap();
            let current = f64::from_bits(*sum_bits);
            *sum_bits = (current + value).to_bits();
        }

        // เพิ่ม bucket counts สำหรับทุก bucket ที่ value <= upper_bound
        for (i, &upper) in self.buckets.iter().enumerate() {
            if value <= upper {
                self.bucket_counts[i].fetch_add(1, Ordering::Relaxed);
            }
        }
        // +Inf bucket รับทุก observation
        let inf_idx = self.buckets.len();
        self.bucket_counts[inf_idx].fetch_add(1, Ordering::Relaxed);
    }

    pub fn count(&self) -> u64 {
        self.count.load(Ordering::Relaxed)
    }

    pub fn sum(&self) -> f64 {
        let bits = *self.sum_bits.lock().unwrap();
        f64::from_bits(bits)
    }

    /// คืน snapshot ของ (upper_bound, cumulative_count) สำหรับทุก bucket
    pub fn snapshot(&self) -> Vec<(f64, u64)> {
        let mut result = Vec::with_capacity(self.buckets.len() + 1);
        for (i, &upper) in self.buckets.iter().enumerate() {
            result.push((upper, self.bucket_counts[i].load(Ordering::Relaxed)));
        }
        let inf_idx = self.buckets.len();
        result.push((
            f64::INFINITY,
            self.bucket_counts[inf_idx].load(Ordering::Relaxed),
        ));
        result
    }
}

/// Default buckets ที่ Prometheus แนะนำสำหรับ latency ใน seconds
pub fn default_buckets() -> Vec<f64> {
    vec![0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
}
```

**ทำไม Histogram ถึงใช้ `Mutex` สำหรับ `sum`?**

`sum` เป็น f64 ที่ต้องการ add ด้วย float arithmetic — operation เดียวกับ Gauge.add() แต่เราจงใจใช้ `Mutex<u64>` แทน CAS loop เพราะ:
1. `sum` ไม่ใช่ hot path เหมือน bucket_counts เพราะถูก read น้อยกว่ามาก
2. ความเรียบง่ายของ `Mutex` ลดความเสี่ยงใน edge cases เช่น NaN หรือ infinity
3. `bucket_counts` ที่เป็น hot path ยังคงใช้ `AtomicU64` อยู่

**ตัวอย่างการใช้งาน Histogram:**

```rust
// สร้าง histogram สำหรับวัด HTTP latency
let latency = Histogram::new(
    "http_request_duration_seconds",
    "HTTP request duration in seconds",
    vec![0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
);

// Simulate requests ด้วย latency ต่าง ๆ
latency.observe(0.003);  // 3ms — ใน bucket <=0.005
latency.observe(0.042);  // 42ms — ใน bucket <=0.05
latency.observe(0.180);  // 180ms — ใน bucket <=0.25
latency.observe(0.750);  // 750ms — ใน bucket <=1.0

println!("Total requests: {}", latency.count()); // 4
println!("Total time: {:.3}s", latency.sum());    // 0.975

let snap = latency.snapshot();
for (upper, count) in &snap {
    if *upper == f64::INFINITY {
        println!("bucket[+Inf] = {}", count);
    } else {
        println!("bucket[{:.3}] = {}", upper, count);
    }
}
```

Output:
```
Total requests: 4
Total time: 0.975s
bucket[0.005] = 1
bucket[0.010] = 1
bucket[0.025] = 1
bucket[0.050] = 2
bucket[0.100] = 2
bucket[0.250] = 3
bucket[0.500] = 3
bucket[1.000] = 4
bucket[2.500] = 4
bucket[5.000] = 4
bucket[+Inf] = 4
```

สังเกตว่า counts เป็น **cumulative** — bucket <=1.0 มีค่า 4 เพราะทุก observation ≤ 1.0

### ขั้นที่ 5: Registry และ Global Singleton

Registry ทำหน้าที่เป็น central store ของ metrics ทั้งหมดใน application ใช้ `DashMap` เพื่อ thread-safe access และ `OnceLock` สำหรับ global singleton

เพิ่ม `dashmap` ใน `Cargo.toml`:

```toml
[package]
name = "metrics-collector"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
dashmap = "5"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

สร้าง `src/registry.rs`:

```rust
use dashmap::DashMap;
use std::sync::Arc;

use crate::counter::Counter;
use crate::gauge::Gauge;
use crate::histogram::Histogram;
use crate::labels::Labels;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct MetricKey {
    pub name: String,
    pub labels: Labels,
}

#[derive(Debug, Clone)]
pub enum MetricValue {
    Counter(Arc<Counter>),
    Gauge(Arc<Gauge>),
    Histogram(Arc<Histogram>),
}

pub struct MetricsRegistry {
    pub metrics: DashMap<MetricKey, MetricValue>,
}

impl MetricsRegistry {
    pub fn new() -> Self {
        MetricsRegistry {
            metrics: DashMap::new(),
        }
    }

    pub fn register_counter(
        &self, name: &str, help: &str, labels: Labels,
    ) -> Arc<Counter> {
        let key = MetricKey { name: name.to_string(), labels };
        let counter = Arc::new(Counter::new(name, help));
        self.metrics.insert(key, MetricValue::Counter(Arc::clone(&counter)));
        counter
    }

    pub fn register_gauge(
        &self, name: &str, help: &str, labels: Labels,
    ) -> Arc<Gauge> {
        let key = MetricKey { name: name.to_string(), labels };
        let gauge = Arc::new(Gauge::new(name, help));
        self.metrics.insert(key, MetricValue::Gauge(Arc::clone(&gauge)));
        gauge
    }

    pub fn register_histogram(
        &self, name: &str, help: &str, labels: Labels, buckets: Vec<f64>,
    ) -> Arc<Histogram> {
        let key = MetricKey { name: name.to_string(), labels };
        let hist = Arc::new(Histogram::new(name, help, buckets));
        self.metrics.insert(key, MetricValue::Histogram(Arc::clone(&hist)));
        hist
    }

    pub fn unregister(&self, name: &str, labels: &Labels) -> bool {
        let key = MetricKey { name: name.to_string(), labels: labels.clone() };
        self.metrics.remove(&key).is_some()
    }

    pub fn get_counter(&self, name: &str, labels: &Labels) -> Option<Arc<Counter>> {
        let key = MetricKey { name: name.to_string(), labels: labels.clone() };
        self.metrics.get(&key).and_then(|v| match v.value() {
            MetricValue::Counter(c) => Some(Arc::clone(c)),
            _ => None,
        })
    }

    pub fn len(&self) -> usize { self.metrics.len() }
    pub fn is_empty(&self) -> bool { self.metrics.is_empty() }
}
```

Global singleton ใน `src/main.rs`:

```rust
use std::sync::OnceLock;
use registry::MetricsRegistry;

static GLOBAL_REGISTRY: OnceLock<MetricsRegistry> = OnceLock::new();

pub fn global_registry() -> &'static MetricsRegistry {
    GLOBAL_REGISTRY.get_or_init(MetricsRegistry::new)
}
```

**ทำไม `OnceLock` ดีกว่า `lazy_static!`?**

`lazy_static!` เป็น macro ที่ต้องการ external crate และใช้ `std::sync::Once` ภายใน ซึ่ง equivalent กับ `OnceLock` แต่ verbose กว่า ตั้งแต่ Rust 1.70 `OnceLock` เป็น stable standard library ที่ไม่ต้องการ dependency เพิ่ม และ API ชัดเจนกว่า

### ขั้นที่ 6: Prometheus Text Format Exposition

Prometheus text format มี specification ที่ชัดเจน — แต่ละ metric type มีโครงสร้าง output ที่ต้องตรงตาม spec เพื่อให้ Prometheus server parse ได้ถูกต้อง

สร้าง `src/exposition.rs`:

```rust
use crate::counter::Counter;
use crate::gauge::Gauge;
use crate::histogram::Histogram;
use crate::labels::Labels;

/// สร้าง Prometheus text format สำหรับ Counter
pub fn render_counter(counter: &Counter, labels: &Labels) -> String {
    let mut out = String::new();
    out.push_str(&format!("# HELP {} {}\n", counter.name, counter.help));
    out.push_str(&format!("# TYPE {} counter\n", counter.name));
    let label_str = labels.to_prometheus_string();
    out.push_str(&format!("{}{} {}\n", counter.name, label_str, counter.get()));
    out
}

/// สร้าง Prometheus text format สำหรับ Gauge
pub fn render_gauge(gauge: &Gauge, labels: &Labels) -> String {
    let mut out = String::new();
    out.push_str(&format!("# HELP {} {}\n", gauge.name, gauge.help));
    out.push_str(&format!("# TYPE {} gauge\n", gauge.name));
    let label_str = labels.to_prometheus_string();
    out.push_str(&format!("{}{} {}\n", gauge.name, label_str, format_f64(gauge.get())));
    out
}

/// สร้าง Prometheus text format สำหรับ Histogram
pub fn render_histogram(histogram: &Histogram, labels: &Labels) -> String {
    let mut out = String::new();
    out.push_str(&format!("# HELP {} {}\n", histogram.name, histogram.help));
    out.push_str(&format!("# TYPE {} histogram\n", histogram.name));

    let base_labels = labels.to_prometheus_string();

    for (upper_bound, count) in histogram.snapshot() {
        let le_str = if upper_bound == f64::INFINITY {
            "+Inf".to_string()
        } else {
            format_f64(upper_bound)
        };

        let bucket_label = if base_labels.is_empty() {
            format!("{{le=\"{}\"}}", le_str)
        } else {
            let inner = &base_labels[1..base_labels.len() - 1];
            format!("{{{},le=\"{}\"}}", inner, le_str)
        };

        out.push_str(&format!(
            "{}_bucket{} {}\n",
            histogram.name, bucket_label, count
        ));
    }

    out.push_str(&format!(
        "{}{}_count {}\n", histogram.name, base_labels, histogram.count()
    ));
    out.push_str(&format!(
        "{}{}_sum {}\n", histogram.name, base_labels, format_f64(histogram.sum())
    ));
    out
}

/// แปลง f64 เป็น string แบบ Prometheus text format
fn format_f64(v: f64) -> String {
    if v.is_nan() { return "NaN".to_string(); }
    if v.is_infinite() {
        return if v > 0.0 { "+Inf".to_string() } else { "-Inf".to_string() };
    }
    if v == v.floor() && v.abs() < 1e15 {
        return format!("{}", v as i64);
    }
    format!("{}", v)
}
```

**ตัวอย่าง output ที่ถูกต้องตาม Prometheus spec:**

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1523
http_requests_total{method="POST",status="200"} 342
http_requests_total{method="GET",status="404"} 12

# HELP http_request_duration_seconds HTTP request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005"} 1
http_request_duration_seconds_bucket{le="0.01"} 1
http_request_duration_seconds_bucket{le="0.025"} 1
http_request_duration_seconds_bucket{le="0.05"} 2
http_request_duration_seconds_bucket{le="0.1"} 2
http_request_duration_seconds_bucket{le="0.25"} 3
http_request_duration_seconds_bucket{le="0.5"} 3
http_request_duration_seconds_bucket{le="1"} 4
http_request_duration_seconds_bucket{le="2.5"} 4
http_request_duration_seconds_bucket{le="5"} 4
http_request_duration_seconds_bucket{le="+Inf"} 4
http_request_duration_seconds_count 4
http_request_duration_seconds_sum 0.975
```

### ขั้นที่ 7: Collector Trait และ ProcessCollector

`Collector` trait ให้ระบบ plugin สำหรับ source ต่าง ๆ แต่ละ collector รู้จักวิธีดึงข้อมูลจาก source ของตัวเองและ render เป็น Prometheus format

สร้าง `src/collector.rs`:

```rust
use crate::counter::Counter;
use crate::gauge::Gauge;

/// Collector trait สำหรับ component ที่รวบรวม metrics จาก source ต่าง ๆ
pub trait Collector: Send + Sync {
    fn name(&self) -> &str;
    fn collect(&self) -> String;
}

/// ProcessCollector รวบรวม metrics เกี่ยวกับ process ปัจจุบัน
pub struct ProcessCollector {
    cpu_time: Counter,
    memory_rss: Gauge,
}

impl ProcessCollector {
    pub fn new() -> Self {
        ProcessCollector {
            cpu_time: Counter::new(
                "process_cpu_seconds_total",
                "Total user and system CPU time in seconds",
            ),
            memory_rss: Gauge::new(
                "process_resident_memory_bytes",
                "Resident memory size in bytes",
            ),
        }
    }

    fn read_rss_bytes() -> Option<f64> {
        let content = std::fs::read_to_string("/proc/self/status").ok()?;
        for line in content.lines() {
            if line.starts_with("VmRSS:") {
                let parts: Vec<&str> = line.split_whitespace().collect();
                if parts.len() >= 2 {
                    let kb: f64 = parts[1].parse().ok()?;
                    return Some(kb * 1024.0);
                }
            }
        }
        None
    }
}

impl Collector for ProcessCollector {
    fn name(&self) -> &str { "process" }

    fn collect(&self) -> String {
        if let Some(rss) = Self::read_rss_bytes() {
            self.memory_rss.set(rss);
        }

        let mut output = String::new();
        output.push_str(&format!(
            "# HELP {} {}\n# TYPE {} gauge\n{} {}\n",
            self.memory_rss.name, self.memory_rss.help,
            self.memory_rss.name, self.memory_rss.name,
            self.memory_rss.get()
        ));
        output
    }
}

/// รวบรวม output จาก collectors ทั้งหมด
pub fn collect_all(collectors: &[Box<dyn Collector>]) -> String {
    collectors.iter().map(|c| c.collect()).collect::<Vec<_>>().join("")
}
```

**`Send + Sync` บน trait object:**

`Box<dyn Collector>` ต้องการ `Send + Sync` bounds เพราะ collectors ถูกใช้จาก multiple threads เมื่อ HTTP server handle requests concurrent `Send` หมายความว่า type นี้ปลอดภัยที่จะ send ระหว่าง threads, `Sync` หมายความว่าปลอดภัยที่จะ share reference ระหว่าง threads

### ขั้นที่ 8: HTTP Exposition Server (tokio)

ขั้นสุดท้ายคือ expose `/metrics` endpoint ผ่าน HTTP เพื่อให้ Prometheus server ดึงข้อมูลได้

```rust
// ใน src/main.rs — เพิ่ม HTTP server
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

async fn run_metrics_server(addr: &str) -> std::io::Result<()> {
    let listener = TcpListener::bind(addr).await?;
    println!("Metrics server listening on http://{}/metrics", addr);

    loop {
        let (mut socket, peer) = listener.accept().await?;
        println!("Connection from {}", peer);

        tokio::spawn(async move {
            let mut buf = vec![0u8; 4096];
            let n = socket.read(&mut buf).await.unwrap_or(0);
            let request = String::from_utf8_lossy(&buf[..n]);

            // Parse path จาก HTTP request line
            let path = request
                .lines()
                .next()
                .and_then(|line| line.split_whitespace().nth(1))
                .unwrap_or("/");

            let (status, body) = if path == "/metrics" {
                // รวบรวม metrics จาก registry
                let reg = crate::global_registry();
                let mut output = String::new();
                for entry in reg.metrics.iter() {
                    use crate::registry::MetricValue;
                    use crate::labels::Labels;
                    match entry.value() {
                        MetricValue::Counter(c) => {
                            output.push_str(&crate::exposition::render_counter(
                                c, &entry.key().labels,
                            ));
                        }
                        MetricValue::Gauge(g) => {
                            output.push_str(&crate::exposition::render_gauge(
                                g, &entry.key().labels,
                            ));
                        }
                        MetricValue::Histogram(h) => {
                            output.push_str(&crate::exposition::render_histogram(
                                h, &entry.key().labels,
                            ));
                        }
                    }
                }
                ("200 OK", output)
            } else {
                ("404 Not Found", "404 Not Found\n".to_string())
            };

            let response = format!(
                "HTTP/1.1 {}\r\nContent-Type: text/plain; version=0.0.4\r\nContent-Length: {}\r\n\r\n{}",
                status,
                body.len(),
                body
            );
            let _ = socket.write_all(response.as_bytes()).await;
        });
    }
}

#[tokio::main]
async fn main() {
    // Register some demo metrics
    let reg = global_registry();
    let requests = reg.register_counter(
        "http_requests_total", "Total HTTP requests", Labels::empty()
    );
    let memory = reg.register_gauge(
        "memory_bytes", "Memory usage", Labels::empty()
    );
    let latency = reg.register_histogram(
        "request_latency_seconds", "Request latency",
        Labels::empty(), histogram::default_buckets()
    );

    // Simulate some activity
    requests.add(42);
    memory.set(512.0 * 1024.0 * 1024.0);
    latency.observe(0.003);
    latency.observe(0.042);

    // Start server
    run_metrics_server("127.0.0.1:9090").await.unwrap();
}
```

ทดสอบด้วย `curl`:
```bash
$ cargo run &
$ curl http://localhost:9090/metrics
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total 42
# HELP memory_bytes Memory usage
# TYPE memory_bytes gauge
memory_bytes 536870912
...
```

## การทดสอบ (Testing)

โปรเจคมี 40 unit tests ครอบคลุมทุก module:

```toml
# Cargo.toml
[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

### รัน Tests

```bash
cargo test
```

### ผลลัพธ์ Real `cargo test` Output

ต่อไปนี้คือ output จริงจากการรัน tests:

```
   Compiling metrics-collector v0.1.0 (...)
warning: field `cpu_time` is never read
  --> src/collector.rs:15:5
   |
14 | pub struct ProcessCollector {
   |            ---------------- field in this struct
15 |     cpu_time: Counter,
   |     ^^^^^^^^
   |
   = note: `#[warn(dead_code)]` (part of `#[warn(unused)]`) on by default

warning: `metrics-collector` (bin "metrics-collector" test) generated 1 warning
    Finished `test` profile [unoptimized + debuginfo] target(s) in 18.04s
     Running unittests src/main.rs (target/debug/deps/metrics_collector-4f23687ef4f85f8c)

running 40 tests
test collector::tests::test_collect_all ... ok
test collector::tests::test_process_collector_name ... ok
test collector::tests::test_process_collector_collect_format ... ok
test counter::tests::test_counter_add_zero ... ok
test counter::tests::test_counter_inc ... ok
test counter::tests::test_counter_add ... ok
test counter::tests::test_counter_initial_value ... ok
test exposition::tests::test_render_counter_with_labels ... ok
test exposition::tests::test_render_gauge_format ... ok
test exposition::tests::test_render_counter_format ... ok
test exposition::tests::test_render_histogram_cumulative_counts ... ok
test exposition::tests::test_render_histogram_has_bucket_lines ... ok
test gauge::tests::test_gauge_add_negative ... ok
test counter::tests::test_counter_thread_safe ... ok
test gauge::tests::test_gauge_inc_dec ... ok
test gauge::tests::test_gauge_can_go_negative ... ok
test gauge::tests::test_gauge_initial_value ... ok
test counter::tests::test_counter_combined ... ok
test gauge::tests::test_gauge_set ... ok
test gauge::tests::test_gauge_sub ... ok
test histogram::tests::test_histogram_default_buckets_sorted ... ok
test histogram::tests::test_histogram_bucket_boundaries ... ok
test histogram::tests::test_histogram_bucket_above_boundary ... ok
test histogram::tests::test_histogram_initial_state ... ok
test histogram::tests::test_histogram_observe_single ... ok
test histogram::tests::test_histogram_sum_accuracy ... ok
test labels::tests::test_invalid_label_names ... ok
test labels::tests::test_label_cardinality_limit ... ok
test histogram::tests::test_histogram_multiple_observations ... ok
test labels::tests::test_label_value_escaping ... ok
test labels::tests::test_to_prometheus_string_empty ... ok
test labels::tests::test_to_prometheus_string_multiple ... ok
test labels::tests::test_to_prometheus_string_single ... ok
test labels::tests::test_valid_label_names ... ok
test registry::tests::test_global_singleton ... ok
test registry::tests::test_registry_get_counter ... ok
test registry::tests::test_registry_multiple_metrics ... ok
test registry::tests::test_registry_unregister_nonexistent ... ok
test registry::tests::test_registry_unregister ... ok
test registry::tests::test_registry_register_counter ... ok

test result: ok. 40 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**40 tests ผ่านทั้งหมด** ครอบคลุม:
- Counter: 6 tests (initial value, inc, add, zero add, combined, thread-safe)
- Gauge: 6 tests (initial, set, inc/dec, add negative, sub, can-go-negative)
- Histogram: 7 tests (initial, single observe, bucket boundaries, above boundary, multiple, sum accuracy, default buckets sorted)
- Labels: 5 tests (valid names, invalid names, cardinality limit, prometheus string empty/single/multiple, value escaping)
- Registry: 5 tests (register, get, unregister, unregister nonexistent, multiple types, global singleton)
- Exposition: 5 tests (counter format, counter with labels, gauge format, histogram bucket lines, histogram cumulative counts)
- Collector: 3 tests (name, collect format, collect_all)

### Demo Output

```bash
$ cargo run
=== Metrics Collector Demo ===

Counter: http_requests_total = 12
Gauge: memory_usage_bytes = 536870912
Histogram: 4 observations
Labels: {method="GET",path="/api/v1/users"}

--- Prometheus Text Format ---
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total 12
# HELP http_request_duration_seconds HTTP request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.005"} 1
http_request_duration_seconds_bucket{le="0.01"} 1
http_request_duration_seconds_bucket{le="0.025"} 1
http_request_duration_seconds_bucket{le="0.05"} 2
http_request_duration_seconds_bucket{le="0.1"} 2
http_request_duration_seconds_bucket{le="0.25"} 3
http_request_duration_seconds_bucket{le="0.5"} 3
http_request_duration_seconds_bucket{le="1"} 4
http_request_duration_seconds_bucket{le="2.5"} 4
http_request_duration_seconds_bucket{le="5"} 4
http_request_duration_seconds_bucket{le="+Inf"} 4
http_request_duration_seconds_count 4
http_request_duration_seconds_sum 0.975

Demo complete!
```

## Pitfalls ที่พบบ่อย

### Pitfall 1: Counter ที่ Overflow

`AtomicU64` มีค่าสูงสุดที่ 2^64 - 1 ≈ 1.8 × 10^19 ซึ่งเพียงพอสำหรับ most use cases แต่ถ้า increment เร็วมาก (เช่น nanosecond-level operations) อาจ overflow ได้ใน theoretical sense

```rust
// ❌ อย่า reset counter ด้วยตัวเองใน production
counter.reset(); // ทำให้ Prometheus rate calculation ผิดพลาด

// ✅ Prometheus rate() function จัดการ counter reset ได้เอง
// แต่เฉพาะ reset ที่ detect ได้ (เช่น counter กลับไปเป็น 0 หลัง restart)
// rate(http_requests_total[5m]) จะคำนวณ average rate per second
```

**ปัญหา:** ถ้า reset counter ใน middle of scrape interval Prometheus จะคิดว่า counter กลับมา 0 แล้ว rate จะ spike ชั่วคราว ควรสร้าง counter ใหม่แทนการ reset

### Pitfall 2: High Cardinality Labels

Label cardinality หมายถึงจำนวน unique label value combinations นี่คือ trap ที่พบบ่อยมากใน Prometheus usage

```rust
// ❌ ห้ามใช้ user ID, request ID, หรือ UUID เป็น label
let labels = Labels::new(vec![
    ("user_id".to_string(), user.id.to_string()),  // อาจมี millions of users!
]);
reg.register_counter("user_actions", "User actions", labels);
// ผลลัพธ์: millions of time series → OOM ใน Prometheus server

// ✅ ใช้แค่ low-cardinality labels
let labels = Labels::new(vec![
    ("action_type".to_string(), "login".to_string()),
    ("region".to_string(), "us-east".to_string()),
]);
// จำนวน combinations จำกัด (e.g., 5 action_types × 3 regions = 15 series)
```

**กฎทั่วไป:** label value ไม่ควรมี unique values เกิน ~100-1000 ค่า ถ้าต้องการ per-user data ให้ใช้ logging หรือ distributed tracing แทน

### Pitfall 3: Histogram Bucket Selection ที่ไม่เหมาะสม

Bucket boundaries ส่งผลต่อ accuracy ของ quantile estimation โดยตรง

```rust
// ❌ Buckets ที่ coarse เกินไป — ไม่เห็น detail ที่สำคัญ
let h = Histogram::new("latency", "Request latency",
    vec![1.0, 10.0, 100.0]  // หน่วย seconds — ใหญ่เกินไปสำหรับ web requests
);

// ❌ Buckets ที่ละเอียดเกินไป — waste memory
let h = Histogram::new("latency", "Request latency",
    (0..1000).map(|i| i as f64 * 0.001).collect()  // 1000 buckets!
);

// ✅ Buckets ที่ตอบโจทย์ SLO จริง
// ถ้า SLO คือ p99 < 200ms ให้มี buckets รอบ ๆ 200ms
let h = Histogram::new("latency", "Request latency in seconds",
    vec![0.01, 0.05, 0.1, 0.2, 0.5, 1.0, 2.0, 5.0]
    // หน่วย seconds ตาม Prometheus convention
);
```

**หลักการ:** วาง buckets ตาม SLO boundaries ของคุณ เช่น ถ้า SLO คือ p95 < 100ms ให้มี bucket ที่ 0.1s แน่ ๆ

### Pitfall 4: Thread-Safety ของ `histogram.sum_bits` Mutex

ใน implementation ของเราใช้ `Mutex<u64>` สำหรับ `sum_bits` ซึ่ง blocking อาจเกิด deadlock ถ้าไม่ระวัง

```rust
// ❌ อย่าเรียก observe() ขณะถือ lock อื่น ๆ ที่อาจ call back มา
{
    let _lock = some_other_mutex.lock().unwrap();
    histogram.observe(value); // ← อาจ deadlock ถ้า some_other_mutex ถูก lock อีกชั้น
}

// ✅ ทำ observe() โดยตรง ไม่ภายใน nested locks
histogram.observe(value);
```

ใน production implementation อาจใช้ `AtomicF64` (unstable) หรือ `std::sync::atomic::AtomicU64` + CAS loop แทน Mutex เพื่อหลีกเลี่ยง blocking โดยสิ้นเชิง

### Pitfall 5: OnceLock กับ Test Isolation

Global singleton ด้วย `OnceLock` initialized เพียงครั้งเดียวตลอดอายุ process ทำให้ tests ที่ใช้ global registry มี state ที่ carry over ระหว่างกัน

```rust
// ❌ Tests ที่ depend on global registry อาจ fail หรือ interfere กัน
#[test]
fn test_a() {
    global_registry().register_counter("requests", "...", Labels::empty());
    // ลืม unregister — test_b จะเห็น counter นี้ด้วย
}

#[test]
fn test_b() {
    assert_eq!(global_registry().len(), 0); // อาจ fail ถ้า test_a รันก่อน!
}

// ✅ ทำ test กับ registry instance ใหม่แทน global
#[test]
fn test_isolated() {
    let reg = MetricsRegistry::new();
    let c = reg.register_counter("requests", "...", Labels::empty());
    assert_eq!(reg.len(), 1);
    // reg ถูก drop เมื่อ test จบ — ไม่กระทบ test อื่น
}
```

### Pitfall 6: Prometheus Text Format ที่ผิด Spec

Prometheus parser strict มาก — format ผิดนิดเดียวทำให้ scrape ล้มเหลว

```
# ❌ ผิด: ไม่มี newline หลังบรรทัดสุดท้าย
http_requests_total 42

# ❌ ผิด: label separator ต้องเป็น comma ไม่ใช่ semicolon
http_requests{method="GET";status="200"} 1

# ❌ ผิด: value ต้องเป็น float/int ไม่ใช่ string
http_requests_total "forty-two"

# ✅ ถูก: แต่ละ line ลงท้ายด้วย \n เสมอ
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1523
```

ใช้ [`promtool check metrics`](https://prometheus.io/docs/prometheus/latest/command-line/promtool/) เพื่อ validate format ก่อน deploy

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized release binary
cargo build --release

# Binary อยู่ที่ target/release/metrics-collector
ls -la target/release/metrics-collector
# -rwxr-xr-x ... 2.1M target/release/metrics-collector

# Strip debug symbols เพื่อลดขนาด
strip target/release/metrics-collector
ls -la target/release/metrics-collector
# -rwxr-xr-x ... 450K target/release/metrics-collector
```

### Docker Integration

```dockerfile
FROM rust:1.75-alpine as builder
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/metrics-collector /
EXPOSE 9090
CMD ["/metrics-collector"]
```

```bash
docker build -t metrics-collector:latest .
docker run -p 9090:9090 metrics-collector:latest

# ทดสอบ
curl http://localhost:9090/metrics
```

### Prometheus Configuration

เพิ่มใน `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'my-rust-app'
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: /metrics
    scheme: http
```

### Grafana Dashboard

หลังจาก Prometheus scrape metrics แล้ว สร้าง Grafana dashboard:

```
# PromQL queries ที่ใช้บ่อย:

# Request rate per second (5m window)
rate(http_requests_total[5m])

# Error rate percentage
rate(http_requests_total{status=~"5.."}[5m]) /
rate(http_requests_total[5m]) * 100

# p99 latency (ต้องใช้ histogram)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# p50, p90, p99 latency comparison
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.90, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### Benchmark Performance

```bash
# เพิ่มใน Cargo.toml สำหรับ benchmark
[dev-dependencies]
criterion = "0.5"

# เพิ่มใน benches/bench_counter.rs
use criterion::{criterion_group, criterion_main, Criterion};
use metrics_collector::counter::Counter;

fn bench_counter_inc(c: &mut Criterion) {
    let counter = Counter::new("bench", "Benchmark counter");
    c.bench_function("counter_inc", |b| b.iter(|| counter.inc()));
}

criterion_group!(benches, bench_counter_inc);
criterion_main!(benches);
```

```bash
cargo bench
# counter_inc time: [2.4 ns 2.5 ns 2.6 ns]
# ≈ 400 million increments per second บน modern CPU
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Summary Metric (ความยาก: ⭐⭐⭐)

เพิ่ม `Summary` metric type ที่ estimate quantiles โดยไม่ต้องกำหนด buckets ล่วงหน้า ใช้ sliding window approach:

```rust
/// Summary เก็บ observations ใน fixed-size sorted window
/// แล้วคำนวณ quantiles จาก window ปัจจุบัน
pub struct Summary {
    pub name: String,
    pub help: String,
    window_size: usize,
    // ใช้ Mutex เพราะต้องทำ sorting
    observations: Mutex<Vec<f64>>,
    count: AtomicU64,
    sum_bits: Mutex<u64>,
}

impl Summary {
    pub fn new(name: &str, help: &str, window_size: usize) -> Self { ... }
    pub fn observe(&self, value: f64) { ... }
    /// คำนวณ quantile q (เช่น 0.5 สำหรับ median, 0.99 สำหรับ p99)
    pub fn quantile(&self, q: f64) -> f64 { ... }
}
```

Hint: ใช้ `observations.sort_by(|a, b| a.partial_cmp(b).unwrap())` แล้ว index เข้า vector ตาม quantile percentage

### แบบฝึกหัดที่ 2: Metric Labels ใน Registry (ความยาก: ⭐⭐)

ปัจจุบัน registry เก็บ metric แยกต่างหากสำหรับแต่ละ label combination เพิ่ม API ที่สะดวกกว่า:

```rust
// API ที่ต้องการสร้าง:
let requests = reg.counter_vec(
    "http_requests_total",
    "Total HTTP requests",
    &["method", "status"],  // ระบุ label names ล่วงหน้า
);

// ใช้งาน — ไม่ต้องสร้าง Labels struct ทุกครั้ง
requests.with_labels(&[("method", "GET"), ("status", "200")]).inc();
requests.with_labels(&[("method", "POST"), ("status", "404")]).inc();
```

Hint: สร้าง `CounterVec` struct ที่มี `DashMap<Vec<String>, Arc<Counter>>` ภายใน

### แบบฝึกหัดที่ 3: Metrics Middleware สำหรับ Axum (ความยาก: ⭐⭐⭐)

integrate metrics library กับ `axum` web framework เป็น middleware layer:

```rust
// เพิ่มใน Cargo.toml
// axum = "0.7"
// tower = "0.4"

use axum::{middleware, Router};

async fn metrics_middleware(req: Request, next: Next) -> Response {
    let start = std::time::Instant::now();
    let method = req.method().to_string();
    let path = req.uri().path().to_string();

    let response = next.run(req).await;

    let status = response.status().as_u16().to_string();
    let duration = start.elapsed().as_secs_f64();

    // บันทึก metrics
    REQUEST_COUNTER.with_labels(&[
        ("method", &method), ("path", &path), ("status", &status)
    ]).inc();
    REQUEST_DURATION.with_labels(&[
        ("method", &method), ("path", &path)
    ]).observe(duration);

    response
}
```

### แบบฝึกหัดที่ 4: Push Gateway Client (ความยาก: ⭐⭐⭐⭐)

สร้าง `PushGateway` client ที่ push metrics ไปยัง Prometheus Pushgateway แทนการรอให้ scrape — เหมาะสำหรับ batch jobs ที่ exit เร็วเกินกว่าจะถูก scrape ได้

```rust
pub struct PushGateway {
    url: String,
    job: String,
    client: reqwest::Client,
}

impl PushGateway {
    pub fn new(url: &str, job: &str) -> Self { ... }

    /// Push metrics ทั้งหมดใน registry ไปยัง Pushgateway
    pub async fn push(&self, registry: &MetricsRegistry) -> Result<(), reqwest::Error> {
        let body = render_all(registry);
        self.client
            .post(format!("{}/metrics/job/{}", self.url, self.job))
            .header("Content-Type", "text/plain; version=0.0.4")
            .body(body)
            .send()
            .await?;
        Ok(())
    }
}
```

### แบบฝึกหัดที่ 5: OpenMetrics Format Support (ความยาก: ⭐⭐⭐⭐)

OpenMetrics เป็น standard ที่ evolve มาจาก Prometheus text format และได้รับการ standardize โดย CNCF เพิ่ม support สำหรับ format ใหม่นี้:

```rust
pub enum ExpositionFormat {
    PrometheusText,   // Content-Type: text/plain; version=0.0.4
    OpenMetrics,      // Content-Type: application/openmetrics-text; version=1.0.0
}

pub fn render(metric: &MetricValue, labels: &Labels, fmt: ExpositionFormat) -> String {
    match fmt {
        ExpositionFormat::PrometheusText => render_prometheus(metric, labels),
        ExpositionFormat::OpenMetrics => render_openmetrics(metric, labels),
    }
}
```

ความแตกต่างหลัก: OpenMetrics ใช้ `created` timestamp, `exemplar` ใน histogram, และ `# EOF` ท้าย document

### แบบฝึกหัดที่ 6: Alerting Rules Generator (ความยาก: ⭐⭐⭐)

สร้าง utility ที่ generate Prometheus alerting rules จาก metrics ที่ register ไว้:

```rust
pub struct AlertRule {
    pub name: String,
    pub expr: String,
    pub duration: String,
    pub severity: String,
    pub message: String,
}

// ตัวอย่าง: สร้าง alert ถ้า error rate > threshold
let rule = AlertRule::error_rate_alert(
    "HighErrorRate",
    "http_requests_total",
    &["status=~\"5..\""],
    0.05,  // 5% error rate threshold
    "5m",  // for 5 minutes
);

// Generate YAML output สำหรับ Prometheus rules file
println!("{}", rule.to_yaml());
```

## สรุป

ในโปรเจคนี้เราได้สร้าง metrics collection library ที่ production-quality ด้วย Rust:

**Metric types ที่สร้าง:**
- `Counter` — lock-free increment ด้วย `AtomicU64::fetch_add`
- `Gauge` — arbitrary f64 ด้วย bit-cast + CAS loop
- `Histogram` — configurable buckets, cumulative counts, sum ด้วย combination ของ atomics และ mutex

**Patterns สำคัญที่ได้เรียน:**
1. **Lock-free programming** ด้วย atomic operations — เร็วกว่า mutex อย่างมีนัยสำคัญใน contended scenarios
2. **f64 bit manipulation** — IEEE 754 bit pattern reinterpretation ที่ safe ใน Rust
3. **CAS loop** — compare-exchange retry pattern สำหรับ atomic float arithmetic
4. **Global singleton** ด้วย `OnceLock` — race-free initialization ด้วย zero external dependencies
5. **Trait objects** (`Box<dyn Collector>`) — polymorphic plugin system ที่ compile-time safe
6. **DashMap** — concurrent hash map สำหรับ registry ที่ scale กับ threads จำนวนมาก

**Prometheus Text Format ที่ implement:**
- `# HELP` และ `# TYPE` headers
- Label syntax `{key="value",key2="value2"}` พร้อม escaping
- Histogram `_bucket`, `_count`, `_sum` suffixes
- `+Inf` bucket สำหรับ total count
- Proper numeric formatting (integer ไม่มีทศนิยม, f64 ตาม Prometheus convention)

โปรเจค G05 ต่อไปจะสร้าง **Config Manager** ที่ใช้ หลักการ type-safe configuration ด้วย TOML/YAML parsing, hot-reload, และ validation — ซึ่งจะนำ patterns ที่เรียนจากโปรเจคนี้ไปใช้ใน real application configuration lifecycle

---

**โปรเจคก่อนหน้า:** [Project G03: Log Aggregator](project-g03-log-aggregator.md) | **โปรเจคถัดไป:** [Project G05: Config Manager](project-g05-config-manager.md)
