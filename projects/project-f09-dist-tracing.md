# Project F09: Distributed Tracing System

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 14 ชั่วโมง

## ภาพรวมโปรเจค

**Distributed Tracing** คือเทคนิคที่ใช้ติดตามเส้นทางการประมวลผลของ request หนึ่งตัวขณะที่มันเดินทางผ่าน microservices หลาย ๆ ตัวในระบบ production ลองนึกภาพ request จากผู้ใช้ที่ผ่าน API Gateway → Auth Service → Order Service → Inventory Service → Payment Service ทุกขั้นตอนมีโอกาสเกิด latency หรือ error และถ้าระบบ trace ไม่ดี การ debug ปัญหาจะใช้เวลาหลายชั่วโมง

โปรเจคนี้สร้าง **distributed tracing library** ที่ได้รับแรงบันดาลใจจาก [OpenTelemetry](https://opentelemetry.io/) — มาตรฐาน open-source ที่ CNCF ดูแล ซึ่ง Google, Microsoft, Datadog, Honeycomb นำไปใช้ในผลิตภัณฑ์จริง เราจะสร้าง implementation ตั้งแต่ต้นโดยไม่พึ่ง crate ภายนอก (นอกจาก `rand` และ `serde`) เพื่อให้เข้าใจ mechanism ทุกชั้น

**แนวคิดหลักของ tracing:**

```
Request → Service A              → Service B              → Service C
          [Span: handle-request]   [Span: rpc.call]         [Span: db.query]
          trace_id=abc123           trace_id=abc123           trace_id=abc123
          span_id=001               parent=001, span_id=002   parent=002, span_id=003
          start=0ms, end=100ms      start=5ms, end=95ms       start=10ms, end=80ms
```

spans ทั้ง 3 ตัวแชร์ `trace_id` เดียวกัน ทำให้เราดู "ต้นไม้" ของ request ได้ในระบบ tracing เช่น Jaeger หรือ Zipkin

**Use case จริงใน production:**
- **Latency profiling** — หา bottleneck ว่า service ไหนใช้เวลาเกินโควต้า
- **Error propagation** — trace ว่า error ที่เห็นใน API layer มาจาก DB query ชั้นล่างไหน
- **Dependency graph** — ดูว่า service A พึ่งพา service อะไรบ้างใน real traffic
- **Sampling** — เก็บเพียง 1% ของ traces เพื่อประหยัด storage ในขณะที่ไม่เสีย insight หลัก
- **SLA monitoring** — alert เมื่อ P99 latency ของ span ใดเกิน threshold

## สิ่งที่จะได้เรียนรู้

- **W3C TraceContext specification** — format มาตรฐานของ `traceparent` header ที่ทุก vendor ใช้ร่วมกัน
- **Thread-local storage** — `thread_local!` + `RefCell` สำหรับ propagate context โดยไม่ต้องส่งผ่าน argument ทุกฟังก์ชัน
- **RAII pattern** สำหรับ span lifecycle — `Drop` trait รับประกันว่า span จะถูก end เสมอ ไม่ว่า early return หรือ panic
- **Trait object** (`dyn SpanExporter`, `dyn Sampler`) — open/closed principle: เพิ่ม exporter ใหม่โดยไม่แตะ core logic
- **Arc + Mutex** สำหรับ thread-safe shared mutable state ใน InMemoryExporter
- **Deterministic hashing** — ใช้ byte representation ของ TraceId เป็น hash สำหรับ ratio-based sampling ที่ reproduce ได้
- **Builder pattern** — `SpanBuilder` สำหรับสร้าง span ด้วย fluent API ที่ optional fields ทุกตัวสามารถ default ได้
- **Context propagation** — inject/extract W3C `traceparent` + `baggage` headers ระหว่าง service

## ความรู้ที่ต้องมีมาก่อน

- **Part 13-17**: Generic types, trait bounds — ใช้สำหรับ `SpanExporter` และ `Sampler` traits
- **Part 18-22**: Closures, `Fn` traits — ใช้ใน `in_span(name, closure)` API
- **Part 23-27**: `Box<dyn Trait>`, `Arc<dyn Trait>` — dynamic dispatch สำหรับ exporter/sampler
- **Part 35-40**: Error handling, `Option<T>`, `Result<T, E>` — parse traceparent
- **Part 46-50**: async/await พื้นฐาน — BatchExporter ใน extension ใช้ tokio
- **Part 51-60**: `Arc`, `Mutex`, thread-safety — InMemoryExporter ที่แชร์ระหว่าง threads
- **Part 61-65**: `serde`, `serde_json` — ConsoleExporter output JSON lines
- **Part 96-100**: `thread_local!`, `RefCell` — thread-local span context
- **Part 101-105**: `Drop` trait, RAII — Span lifecycle management

## โครงสร้างโปรเจค (Project Layout)

```
dist-tracing/
├── src/
│   ├── lib.rs           # Public API re-exports ทุก module
│   ├── main.rs          # Demo binary แสดง end-to-end usage
│   ├── context.rs       # TraceId, SpanId, SpanContext, TraceFlags
│   │                    # thread-local current span context
│   ├── span.rs          # SpanData, SpanEvent, SpanStatus, Span, SpanBuilder
│   ├── exporter.rs      # SpanExporter trait, InMemoryExporter, ConsoleExporter
│   ├── sampler.rs       # Sampler trait, AlwaysOn, AlwaysOff, TraceIdRatioBased
│   ├── propagation.rs   # W3C traceparent inject/extract, baggage
│   └── tracer.rs        # Tracer struct, start_span(), in_span()
├── Cargo.toml
└── README.md
```

**หมายเหตุโครงสร้าง:** เราแยก module ตามความรับผิดชอบ (separation of concerns) ดังนี้:
- `context` — pure data types, ไม่มี logic ที่ซับซ้อน
- `span` — lifecycle management
- `exporter` — output/sink layer
- `sampler` — sampling decision layer
- `propagation` — serialization/deserialization layer
- `tracer` — high-level factory API

การแยกแบบนี้ทำให้ swap `sampler` หรือ `exporter` ได้โดยไม่กระทบ logic หลัก

## การออกแบบ (Architecture & Design)

### Data Flow Overview

```
ผู้ใช้เรียก tracer.in_span("name", |span| { ... })
              │
              ▼
         Tracer::in_span()
              │ สร้าง SpanBuilder
              │ ตรวจ thread-local current context
              │    ┌── มี context → เป็น child span
              │    └── ไม่มี     → เป็น root span
              ▼
         SpanBuilder::start()
              │ เรียก Sampler::should_sample()
              │ สร้าง SpanContext (TraceId, SpanId, TraceFlags)
              │ set thread-local context = span context
              ▼
         Span (active)
              │ รัน user closure
              │ span.set_attribute(), add_event(), set_status()
              ▼
         Span::end() / Drop
              │ บันทึก end_time
              │ เรียก SpanExporter::export(&span_data)
              │    ┌── InMemoryExporter → เก็บใน Vec
              │    ├── ConsoleExporter  → println! JSON
              │    └── BatchExporter   → ส่งเข้า channel (tokio)
              ▼
         Restore thread-local context (ของ parent)
```

### TraceId และ SpanId

```
TraceId: [u8; 16]  ← 128-bit random UUID (เหมือน UUID v4 แต่ไม่มี version bits)
SpanId:  [u8; 8]   ← 64-bit random value

ตัวอย่าง traceparent header (W3C format):
00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
│   │                                │                 │
│   └── trace_id (32 hex chars)      │                 └── flags (01=sampled)
│                                    └── span_id (16 hex chars)
└── version (ปัจจุบันเป็น "00" เสมอ)
```

### Thread-Local Context Propagation

การ propagate context ระหว่าง parent/child spans ภายใน process เดียวกันใช้ `thread_local!`:

```
Thread A:
  thread_local! { CURRENT_CONTEXT = invalid }
  
  in_span("outer") {
    CURRENT_CONTEXT = outer.context         ← set
    
    start_span("inner") {                    ← อ่าน CURRENT_CONTEXT → inner.parent = outer
      CURRENT_CONTEXT = inner.context       ← set
    }
    
    CURRENT_CONTEXT = outer.context         ← restore
  }
  
  CURRENT_CONTEXT = invalid                 ← restore
```

ข้ามเส้นขอบ network ต้องใช้ `inject_traceparent()` / `extract_traceparent()` แปลง context เป็น HTTP header

### Sampling Decision

```
TraceIdRatioBased(0.1):
  trace_id bytes → u64 (big-endian bytes 0..8)
                 → normalize 0.0..1.0
                 → sample ถ้า < 0.1
                 
ข้อดีของ approach นี้:
  - Deterministic: trace_id เดิม → ผลเดิมเสมอ
  - Consistent: ทุก service ในระบบ sample trace_id เดิมได้
  - ไม่ต้องส่ง "sampled?" flag ผ่าน network แยกต่างหาก
```

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Core Types — TraceId, SpanId, SpanContext

เริ่มจาก type พื้นฐานที่สุด ซึ่งเป็น building blocks ของทั้งระบบ

**`Cargo.toml`**

```toml
[package]
name = "dist-tracing"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rand = "0.8"
```

**`src/context.rs`**

```rust
use std::fmt;
use std::cell::RefCell;

/// Unique identifier สำหรับ trace ทั้งหมด (128-bit)
/// ทุก span ภายใน trace เดียวกันแชร์ TraceId นี้
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct TraceId(pub [u8; 16]);

/// Unique identifier สำหรับ span แต่ละตัว (64-bit)
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
pub struct SpanId(pub [u8; 8]);

/// Flags ที่กำหนดพฤติกรรมของ trace
/// bit 0 = sampled flag (1 = บันทึก span นี้, 0 = ทิ้ง)
#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub struct TraceFlags(pub u8);

impl TraceFlags {
    pub const SAMPLED: TraceFlags = TraceFlags(0x01);
    pub const NONE: TraceFlags = TraceFlags(0x00);

    pub fn is_sampled(self) -> bool {
        self.0 & 0x01 != 0
    }
}

/// Context ที่ propagate ไปกับทุก span
/// ประกอบด้วย TraceId, SpanId, flags, และ tracestate
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct SpanContext {
    pub trace_id: TraceId,
    pub span_id: SpanId,
    pub trace_flags: TraceFlags,
    pub trace_state: String,  // W3C tracestate header value
}

impl SpanContext {
    pub fn new(trace_id: TraceId, span_id: SpanId, trace_flags: TraceFlags) -> Self {
        SpanContext { trace_id, span_id, trace_flags, trace_state: String::new() }
    }

    /// SpanContext ที่ไม่มีค่าจริง (all zeros) — ใช้เป็น sentinel ว่าไม่มี active span
    pub fn invalid() -> Self {
        SpanContext {
            trace_id: TraceId([0u8; 16]),
            span_id: SpanId([0u8; 8]),
            trace_flags: TraceFlags::NONE,
            trace_state: String::new(),
        }
    }

    pub fn is_valid(&self) -> bool {
        self.trace_id.0 != [0u8; 16] && self.span_id.0 != [0u8; 8]
    }
}

impl TraceId {
    pub fn random() -> Self {
        use rand::RngCore;
        let mut bytes = [0u8; 16];
        rand::thread_rng().fill_bytes(&mut bytes);
        TraceId(bytes)
    }

    pub fn from_hex(s: &str) -> Option<Self> {
        if s.len() != 32 { return None; }
        let mut bytes = [0u8; 16];
        for i in 0..16 {
            bytes[i] = u8::from_str_radix(&s[i*2..i*2+2], 16).ok()?;
        }
        Some(TraceId(bytes))
    }

    pub fn to_hex(&self) -> String {
        self.0.iter().map(|b| format!("{:02x}", b)).collect()
    }
}

impl SpanId {
    pub fn random() -> Self {
        use rand::RngCore;
        let mut bytes = [0u8; 8];
        rand::thread_rng().fill_bytes(&mut bytes);
        SpanId(bytes)
    }

    pub fn from_hex(s: &str) -> Option<Self> {
        if s.len() != 16 { return None; }
        let mut bytes = [0u8; 8];
        for i in 0..8 {
            bytes[i] = u8::from_str_radix(&s[i*2..i*2+2], 16).ok()?;
        }
        Some(SpanId(bytes))
    }

    pub fn to_hex(&self) -> String {
        self.0.iter().map(|b| format!("{:02x}", b)).collect()
    }
}

impl fmt::Display for TraceId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.to_hex())
    }
}

impl fmt::Display for SpanId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.to_hex())
    }
}

// Thread-local storage สำหรับ current span context
// ใช้ RefCell แทน Mutex เพราะ thread-local ไม่ต้องการ multi-thread synchronization
thread_local! {
    static CURRENT_CONTEXT: RefCell<SpanContext> = RefCell::new(SpanContext::invalid());
}

pub fn get_current_context() -> SpanContext {
    CURRENT_CONTEXT.with(|c| c.borrow().clone())
}

pub fn set_current_context(ctx: SpanContext) {
    CURRENT_CONTEXT.with(|c| *c.borrow_mut() = ctx);
}

pub fn clear_current_context() {
    CURRENT_CONTEXT.with(|c| *c.borrow_mut() = SpanContext::invalid());
}
```

**สิ่งสำคัญในขั้นนี้:**

- `TraceId([u8; 16])` และ `SpanId([u8; 8])` เป็น newtype wrappers รอบ byte arrays — ทำให้ type system ป้องกันไม่ให้ใช้ TraceId แทน SpanId ผิดที่
- `thread_local! { static ... }` — แต่ละ OS thread มี instance ของตัวเอง ไม่ race condition กัน
- `RefCell` ใน thread-local ปลอดภัยเพราะ `RefCell` ไม่ implement `Sync` — compiler ป้องกัน cross-thread access โดยอัตโนมัติ

---

### ขั้นที่ 2: Span Lifecycle — SpanData, SpanBuilder, RAII Span

**`src/span.rs`**

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::time::{Duration, SystemTime, UNIX_EPOCH};

use crate::context::{SpanContext, SpanId, TraceFlags};
use crate::exporter::SpanExporter;
use crate::sampler::Sampler;

#[derive(Clone, Debug, PartialEq)]
pub enum SpanStatus {
    Unset,                    // ยังไม่กำหนด (default)
    Ok,                       // สำเร็จ
    Error(String),            // ล้มเหลว พร้อม error message
}

#[derive(Clone, Debug)]
pub struct SpanEvent {
    pub name: String,
    pub timestamp: u64,        // nanoseconds since UNIX_EPOCH
    pub attributes: HashMap<String, String>,
}

/// ข้อมูลดิบของ span — ถูกแชร์ระหว่าง Span handle และ exporter ผ่าน Arc<Mutex<>>
#[derive(Debug)]
pub struct SpanData {
    pub name: String,
    pub span_context: SpanContext,
    pub parent_span_id: Option<SpanId>,
    pub start_time: u64,
    pub end_time: Option<u64>,
    pub attributes: HashMap<String, String>,
    pub events: Vec<SpanEvent>,
    pub status: SpanStatus,
}

impl SpanData {
    pub fn duration_ns(&self) -> Option<u64> {
        self.end_time.map(|end| end.saturating_sub(self.start_time))
    }
}

fn now_ns() -> u64 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap_or(Duration::ZERO)
        .as_nanos() as u64
}

/// Span handle ที่ผู้ใช้ถือไว้
/// Drop อัตโนมัติเรียก end() ป้องกัน span ที่ลืม end ค้างอยู่ตลอดไป
pub struct Span {
    pub(crate) inner: Arc<Mutex<SpanData>>,
    pub(crate) exporter: Option<Arc<dyn SpanExporter>>,
    pub(crate) ended: bool,
}

impl Span {
    pub fn name(&self) -> String {
        self.inner.lock().unwrap().name.clone()
    }

    pub fn context(&self) -> SpanContext {
        self.inner.lock().unwrap().span_context.clone()
    }

    pub fn parent_span_id(&self) -> Option<SpanId> {
        self.inner.lock().unwrap().parent_span_id
    }

    pub fn set_attribute(&self, key: impl Into<String>, value: impl Into<String>) {
        self.inner.lock().unwrap().attributes.insert(key.into(), value.into());
    }

    pub fn get_attribute(&self, key: &str) -> Option<String> {
        self.inner.lock().unwrap().attributes.get(key).cloned()
    }

    pub fn add_event(&self, name: impl Into<String>) {
        self.add_event_with_attributes(name, HashMap::new());
    }

    pub fn add_event_with_attributes(
        &self,
        name: impl Into<String>,
        attrs: HashMap<String, String>,
    ) {
        let event = SpanEvent { name: name.into(), timestamp: now_ns(), attributes: attrs };
        self.inner.lock().unwrap().events.push(event);
    }

    pub fn set_status(&self, status: SpanStatus) {
        self.inner.lock().unwrap().status = status;
    }

    /// End span อย่างชัดเจน — idempotent (เรียกซ้ำไม่มีผล)
    pub fn end(&mut self) {
        if self.ended { return; }
        self.ended = true;
        {
            let mut data = self.inner.lock().unwrap();
            data.end_time = Some(now_ns());
        }
        if let Some(exp) = &self.exporter {
            let data = self.inner.lock().unwrap();
            exp.export(&data);
        }
    }

    pub fn is_ended(&self) -> bool { self.ended }
    pub fn is_sampled(&self) -> bool {
        self.inner.lock().unwrap().span_context.trace_flags.is_sampled()
    }
}

/// RAII: Drop เรียก end() อัตโนมัติ
/// ถ้าผู้ใช้เรียก end() เองแล้ว Drop จะ no-op เพราะ ended = true
impl Drop for Span {
    fn drop(&mut self) {
        if !self.ended { self.end(); }
    }
}

/// Builder pattern สำหรับ configurate span ก่อน start
pub struct SpanBuilder {
    pub name: String,
    pub parent_context: Option<SpanContext>,
    pub attributes: HashMap<String, String>,
    pub sampler: Option<Arc<dyn Sampler>>,
    pub exporter: Option<Arc<dyn SpanExporter>>,
}

impl SpanBuilder {
    pub fn new(name: impl Into<String>) -> Self {
        SpanBuilder {
            name: name.into(),
            parent_context: None,
            attributes: HashMap::new(),
            sampler: None,
            exporter: None,
        }
    }

    pub fn with_parent(mut self, ctx: SpanContext) -> Self {
        self.parent_context = Some(ctx);
        self
    }

    pub fn with_attribute(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.attributes.insert(key.into(), value.into());
        self
    }

    pub fn with_sampler(mut self, sampler: Arc<dyn Sampler>) -> Self {
        self.sampler = Some(sampler);
        self
    }

    pub fn with_exporter(mut self, exporter: Arc<dyn SpanExporter>) -> Self {
        self.exporter = Some(exporter);
        self
    }

    pub fn start(self) -> Span {
        use crate::context::{TraceId, get_current_context};

        let (trace_id, parent_span_id, trace_flags) =
            if let Some(ref parent) = self.parent_context {
                // child span ใช้ trace_id ของ parent
                let sampled = self.sampler.as_ref()
                    .map(|s| s.should_sample(&parent.trace_id))
                    .unwrap_or(parent.trace_flags.is_sampled());
                let flags = if sampled { TraceFlags::SAMPLED } else { TraceFlags::NONE };
                (parent.trace_id, Some(parent.span_id), flags)
            } else {
                // ตรวจ thread-local context ก่อน
                let current = get_current_context();
                if current.is_valid() {
                    let sampled = self.sampler.as_ref()
                        .map(|s| s.should_sample(&current.trace_id))
                        .unwrap_or(current.trace_flags.is_sampled());
                    let flags = if sampled { TraceFlags::SAMPLED } else { TraceFlags::NONE };
                    (current.trace_id, Some(current.span_id), flags)
                } else {
                    // root span — สร้าง trace_id ใหม่
                    let new_trace_id = TraceId::random();
                    let sampled = self.sampler.as_ref()
                        .map(|s| s.should_sample(&new_trace_id))
                        .unwrap_or(true);
                    let flags = if sampled { TraceFlags::SAMPLED } else { TraceFlags::NONE };
                    (new_trace_id, None, flags)
                }
            };

        let span_id = crate::context::SpanId::random();
        let span_ctx = SpanContext::new(trace_id, span_id, trace_flags);

        let data = SpanData {
            name: self.name,
            span_context: span_ctx,
            parent_span_id,
            start_time: now_ns(),
            end_time: None,
            attributes: self.attributes,
            events: Vec::new(),
            status: SpanStatus::Unset,
        };

        Span { inner: Arc::new(Mutex::new(data)), exporter: self.exporter, ended: false }
    }
}
```

**สิ่งสำคัญในขั้นนี้:**

- `SpanData` ถูกห่อด้วย `Arc<Mutex<>>` เพื่อให้ `Span` handle และ `SpanExporter` เข้าถึงได้ thread-safely
- `Span::end()` เป็น idempotent — เรียกซ้ำหลายครั้งก็ปลอดภัย (ด้วย `if self.ended { return; }`)
- ลำดับการ lock ใน `end()`: lock → เขียน `end_time` → unlock → lock อีกครั้งเพื่อ export ป้องกัน deadlock ถ้า exporter พยายาม lock span เดิม

---

### ขั้นที่ 3: Exporters — SpanExporter Trait, InMemoryExporter, ConsoleExporter

Exporter คือ "sink" ที่รับ span data ส่งต่อไปยัง backend ต่าง ๆ การใช้ trait ทำให้ swap backend ได้โดยไม่แก้ core code

**`src/exporter.rs`**

```rust
use std::sync::{Arc, Mutex};
use crate::span::SpanData;

/// Trait กำหนด interface ของ exporter ทุกตัว
/// Send + Sync จำเป็นเพราะ Tracer อาจถูกใช้จาก หลาย thread
pub trait SpanExporter: Send + Sync {
    fn export(&self, span: &SpanData);
}

/// InMemoryExporter — สำหรับ testing เก็บ spans ไว้ใน Vec
pub struct InMemoryExporter {
    spans: Mutex<Vec<Arc<Mutex<SpanData>>>>,
}

impl InMemoryExporter {
    pub fn new() -> Self {
        InMemoryExporter { spans: Mutex::new(Vec::new()) }
    }

    pub fn spans(&self) -> Vec<Arc<Mutex<SpanData>>> {
        self.spans.lock().unwrap().clone()
    }

    pub fn span_count(&self) -> usize {
        self.spans.lock().unwrap().len()
    }

    pub fn clear(&self) {
        self.spans.lock().unwrap().clear();
    }
}

impl SpanExporter for InMemoryExporter {
    fn export(&self, span: &SpanData) {
        // Clone SpanData เพื่อเก็บ snapshot ณ เวลา end
        let cloned = SpanData {
            name: span.name.clone(),
            span_context: span.span_context.clone(),
            parent_span_id: span.parent_span_id,
            start_time: span.start_time,
            end_time: span.end_time,
            attributes: span.attributes.clone(),
            events: span.events.clone(),
            status: span.status.clone(),
        };
        self.spans.lock().unwrap().push(Arc::new(Mutex::new(cloned)));
    }
}

/// ConsoleExporter — print JSON lines ไปยัง stdout
/// ใช้ในการ debug หรือ pipe ไปยัง log aggregator
pub struct ConsoleExporter;

impl ConsoleExporter {
    pub fn new() -> Self { ConsoleExporter }
}

impl SpanExporter for ConsoleExporter {
    fn export(&self, span: &SpanData) {
        let json = serde_json::json!({
            "name": span.name,
            "trace_id": span.span_context.trace_id.to_hex(),
            "span_id": span.span_context.span_id.to_hex(),
            "parent_span_id": span.parent_span_id.map(|id| id.to_hex()),
            "start_time_ns": span.start_time,
            "end_time_ns": span.end_time,
            "duration_ns": span.duration_ns(),
            "attributes": span.attributes,
            "events": span.events.iter().map(|e| serde_json::json!({
                "name": e.name,
                "timestamp_ns": e.timestamp,
            })).collect::<Vec<_>>(),
            "status": format!("{:?}", span.status),
        });
        println!("{}", json);
    }
}
```

ตัวอย่าง output ของ ConsoleExporter (JSON line format):

```json
{"attributes":{"db.statement":"SELECT * FROM users","db.system":"postgresql"},
 "duration_ns":1980,
 "end_time_ns":1790621694377451533,
 "events":[{"name":"query.start","timestamp_ns":1790621694377451215}],
 "name":"db.query",
 "parent_span_id":"5d58a572609697c9",
 "span_id":"d7e5e1b753156413",
 "start_time_ns":1790621694377449553,
 "status":"Unset",
 "trace_id":"a57ac4d594ae888d6bf60b910dfbf9ed"}
```

---

### ขั้นที่ 4: Samplers — AlwaysOn, AlwaysOff, TraceIdRatioBased

Sampling คือการตัดสินใจว่าจะบันทึก trace นี้หรือไม่ ในระบบ high-traffic ที่มี 100,000 requests/วินาที การบันทึกทุก trace จะสิ้นเปลือง storage มหาศาล

**`src/sampler.rs`**

```rust
use crate::context::TraceId;

/// Trait สำหรับ sampling decision
pub trait Sampler: Send + Sync {
    /// คืน true ถ้าควร sample trace นี้
    fn should_sample(&self, trace_id: &TraceId) -> bool;
    fn name(&self) -> &str;
}

/// AlwaysOn: sample ทุก trace (development / low-traffic)
pub struct AlwaysOn;
impl Sampler for AlwaysOn {
    fn should_sample(&self, _: &TraceId) -> bool { true }
    fn name(&self) -> &str { "AlwaysOn" }
}

/// AlwaysOff: ไม่ sample เลย (ปิด tracing ชั่วคราว)
pub struct AlwaysOff;
impl Sampler for AlwaysOff {
    fn should_sample(&self, _: &TraceId) -> bool { false }
    fn name(&self) -> &str { "AlwaysOff" }
}

/// TraceIdRatioBased: sample ตาม ratio 0.0-1.0
/// ใช้ 8 bytes แรกของ trace_id เป็น u64 แล้วเทียบกับ threshold
/// Deterministic: trace_id เดิม → ผลเดิมเสมอ ทุก service ในระบบ
pub struct TraceIdRatioBased {
    ratio: f64,
}

impl TraceIdRatioBased {
    pub fn new(ratio: f64) -> Self {
        assert!(ratio >= 0.0 && ratio <= 1.0);
        TraceIdRatioBased { ratio }
    }

    fn trace_id_as_ratio(trace_id: &TraceId) -> f64 {
        let mut bytes = [0u8; 8];
        bytes.copy_from_slice(&trace_id.0[..8]);
        let value = u64::from_be_bytes(bytes);
        value as f64 / u64::MAX as f64
    }
}

impl Sampler for TraceIdRatioBased {
    fn should_sample(&self, trace_id: &TraceId) -> bool {
        if self.ratio <= 0.0 { return false; }
        if self.ratio >= 1.0 { return true; }
        Self::trace_id_as_ratio(trace_id) < self.ratio
    }
    fn name(&self) -> &str { "TraceIdRatioBased" }
}
```

**ทำไม Deterministic Sampling ถึงสำคัญ?**

```
Service A (ratio=0.1):            Service B (ratio=0.1):
  trace_id=abc123 → hash=0.07        trace_id=abc123 → hash=0.07
  0.07 < 0.1 → SAMPLE ✓              0.07 < 0.1 → SAMPLE ✓

ถ้าใช้ random แทน:
  Service A: random=0.03 → SAMPLE ✓
  Service B: random=0.15 → DROP ✗     ← span ของ B หายไปจาก trace!
```

---

### ขั้นที่ 5: Context Propagation — W3C Traceparent Header

เมื่อ request ข้าม service boundary ต้องแปลง SpanContext เป็น HTTP header

**`src/propagation.rs`**

```rust
use std::collections::HashMap;
use crate::context::{TraceId, SpanId, SpanContext, TraceFlags};

/// Inject W3C traceparent header เข้าไปใน headers map
/// Format: 00-{trace_id_hex}-{span_id_hex}-{flags_hex}
pub fn inject_traceparent(ctx: &SpanContext, headers: &mut HashMap<String, String>) {
    if !ctx.is_valid() { return; }
    let value = format!(
        "00-{}-{}-{:02x}",
        ctx.trace_id.to_hex(),
        ctx.span_id.to_hex(),
        ctx.trace_flags.0
    );
    headers.insert("traceparent".to_string(), value);
    if !ctx.trace_state.is_empty() {
        headers.insert("tracestate".to_string(), ctx.trace_state.clone());
    }
}

/// Extract SpanContext จาก W3C traceparent header
pub fn extract_traceparent(headers: &HashMap<String, String>) -> Option<SpanContext> {
    let value = headers.get("traceparent")?;
    parse_traceparent(value)
}

/// Parse traceparent string
/// ตัวอย่าง: "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
pub fn parse_traceparent(s: &str) -> Option<SpanContext> {
    let parts: Vec<&str> = s.split('-').collect();
    if parts.len() < 4 { return None; }
    if parts[0] != "00" { return None; }   // รองรับ version 00 เท่านั้น

    let trace_id = TraceId::from_hex(parts[1])?;
    let span_id = SpanId::from_hex(parts[2])?;
    let flags = u8::from_str_radix(parts[3], 16).ok()?;

    // all-zeros trace_id หรือ span_id ถือว่า invalid
    if trace_id.0 == [0u8; 16] || span_id.0 == [0u8; 8] { return None; }

    let mut ctx = SpanContext::new(trace_id, span_id, TraceFlags(flags));
    ctx.trace_state = String::new();
    Some(ctx)
}

/// Format SpanContext เป็น traceparent string
pub fn format_traceparent(ctx: &SpanContext) -> String {
    format!("00-{}-{}-{:02x}", ctx.trace_id.to_hex(), ctx.span_id.to_hex(), ctx.trace_flags.0)
}

/// Inject baggage header — key=value pairs ที่ propagate ไปกับ request
pub fn inject_baggage(baggage: &HashMap<String, String>, headers: &mut HashMap<String, String>) {
    if baggage.is_empty() { return; }
    let value: Vec<String> = baggage.iter()
        .map(|(k, v)| format!("{}={}", k, v))
        .collect();
    headers.insert("baggage".to_string(), value.join(","));
}

/// Extract baggage จาก headers
pub fn extract_baggage(headers: &HashMap<String, String>) -> HashMap<String, String> {
    let mut result = HashMap::new();
    if let Some(value) = headers.get("baggage") {
        for pair in value.split(',') {
            let pair = pair.trim();
            if let Some(pos) = pair.find('=') {
                let key = pair[..pos].trim().to_string();
                let val = pair[pos+1..].trim().to_string();
                if !key.is_empty() { result.insert(key, val); }
            }
        }
    }
    result
}
```

**ตัวอย่างการใช้งาน context propagation ระหว่าง services:**

```
Service A (caller):
  let span = tracer.start_span("outbound-call");
  let mut headers = HashMap::new();
  inject_traceparent(&span.context(), &mut headers);
  // headers["traceparent"] = "00-abc123...-def456...-01"
  http_client.get("http://service-b/api").headers(headers).send().await?;

Service B (callee):
  async fn handle(headers: HeaderMap) {
      let parent_ctx = extract_traceparent(&headers_map);
      let span = SpanBuilder::new("incoming-request")
          .with_parent(parent_ctx.unwrap_or(SpanContext::invalid()))
          .start();
      // span นี้จะมี trace_id เดิมกับ Service A!
  }
```

---

### ขั้นที่ 6: Tracer — High-Level Factory API

`Tracer` คือ public API หลักที่ผู้ใช้ library แตะ ซ่อน complexity ของ SpanBuilder ไว้ข้างหลัง

**`src/tracer.rs`**

```rust
use std::sync::Arc;
use crate::context::{set_current_context, get_current_context};
use crate::span::{Span, SpanBuilder};
use crate::exporter::SpanExporter;
use crate::sampler::Sampler;

/// Tracer — factory สำหรับสร้าง spans ที่เชื่อมโยงกัน
/// name/version มักตรงกับชื่อ library หรือ service
pub struct Tracer {
    pub name: String,
    pub version: String,
    exporter: Option<Arc<dyn SpanExporter>>,
    sampler: Option<Arc<dyn Sampler>>,
}

impl Tracer {
    pub fn new(name: impl Into<String>, version: impl Into<String>) -> Self {
        Tracer { name: name.into(), version: version.into(), exporter: None, sampler: None }
    }

    pub fn with_exporter(mut self, exporter: Arc<dyn SpanExporter>) -> Self {
        self.exporter = Some(exporter); self
    }

    pub fn with_sampler(mut self, sampler: Arc<dyn Sampler>) -> Self {
        self.sampler = Some(sampler); self
    }

    /// สร้าง span — อัตโนมัติ inherit trace_id และ parent_span_id จาก current context
    pub fn start_span(&self, name: impl Into<String>) -> Span {
        let mut builder = SpanBuilder::new(name);
        if let Some(exp) = &self.exporter { builder = builder.with_exporter(exp.clone()); }
        if let Some(s) = &self.sampler { builder = builder.with_sampler(s.clone()); }

        let current = get_current_context();
        if current.is_valid() { builder = builder.with_parent(current); }

        builder.start()
    }

    /// in_span: สร้าง span, รัน closure, end span โดยอัตโนมัติ (RAII)
    /// ตั้ง thread-local context ระหว่างรัน เพื่อให้ nested spans เห็น parent
    pub fn in_span<F, R>(&self, name: impl Into<String>, f: F) -> R
    where
        F: FnOnce(&Span) -> R,
    {
        let mut span = self.start_span(name);
        let prev_ctx = get_current_context();
        set_current_context(span.context());      // ← ตั้ง context สำหรับ children

        let result = f(&span);

        set_current_context(prev_ctx);             // ← restore parent context
        span.end();
        result
    }
}
```

**ตัวอย่างการใช้งาน `in_span` แบบ nested:**

```rust
tracer.in_span("handle-request", |req_span| {
    req_span.set_attribute("http.method", "POST");

    let user = tracer.in_span("auth.verify-token", |auth_span| {
        auth_span.set_attribute("token.length", "256");
        // span นี้จะมี parent = handle-request span โดยอัตโนมัติ
        verify_token(token)
    });

    tracer.in_span("db.insert-order", |db_span| {
        db_span.set_attribute("db.table", "orders");
        // span นี้ก็มี parent = handle-request span เช่นกัน
        db.insert(user.id, order)
    });
});
```

---

### ขั้นที่ 7: Main Demo — รวมทุกอย่างเข้าด้วยกัน

**`src/lib.rs`**

```rust
pub mod context;
pub mod span;
pub mod tracer;
pub mod propagation;
pub mod exporter;
pub mod sampler;

pub use context::{TraceId, SpanId, SpanContext, TraceFlags};
pub use span::{Span, SpanBuilder, SpanStatus, SpanEvent};
pub use tracer::Tracer;
pub use propagation::{inject_traceparent, extract_traceparent, inject_baggage, extract_baggage};
pub use exporter::{SpanExporter, InMemoryExporter, ConsoleExporter};
pub use sampler::{Sampler, AlwaysOn, AlwaysOff, TraceIdRatioBased};
```

**`src/main.rs`**

```rust
use std::sync::Arc;
use std::collections::HashMap;
use dist_tracing::{
    Tracer, InMemoryExporter, ConsoleExporter, AlwaysOn,
    inject_traceparent, extract_traceparent,
};

fn main() {
    println!("=== Distributed Tracing Demo ===\n");

    // Demo 1: Basic span with ConsoleExporter
    println!("--- Demo 1: Basic Span ---");
    let console_exp = Arc::new(ConsoleExporter::new());
    let tracer = Tracer::new("demo-service", "1.0.0")
        .with_exporter(console_exp.clone())
        .with_sampler(Arc::new(AlwaysOn));

    tracer.in_span("handle-request", |span| {
        span.set_attribute("http.method", "GET");
        span.set_attribute("http.url", "/api/users");
        span.add_event("request.received");
        tracer.in_span("db.query", |db_span| {
            db_span.set_attribute("db.system", "postgresql");
            db_span.set_attribute("db.statement", "SELECT * FROM users");
            db_span.add_event("query.start");
        });
    });

    // Demo 2: Context propagation ระหว่าง "services"
    println!("\n--- Demo 2: Context Propagation ---");
    let exporter = Arc::new(InMemoryExporter::new());
    let tracer2 = Tracer::new("service-a", "1.0.0")
        .with_exporter(exporter.clone());

    let root_span = tracer2.start_span("outbound-call");
    let ctx = root_span.context();

    // Service A inject context ลงใน HTTP headers
    let mut headers = HashMap::new();
    inject_traceparent(&ctx, &mut headers);
    println!("Injected traceparent: {}", headers["traceparent"]);

    // Service B รับ request และ extract context
    let extracted = extract_traceparent(&headers).unwrap();
    println!("Extracted trace_id: {}", extracted.trace_id.to_hex());
    println!("Extracted span_id:  {}", extracted.span_id.to_hex());
    println!("Is sampled: {}", extracted.trace_flags.is_sampled());

    // Demo 3: InMemory exporter สำหรับ metrics
    println!("\n--- Demo 3: InMemory Exporter ---");
    let mem_exp = Arc::new(InMemoryExporter::new());
    let tracer3 = Tracer::new("stats-service", "1.0.0")
        .with_exporter(mem_exp.clone());

    for i in 0..5 {
        tracer3.in_span(format!("op-{}", i), |_| {});
    }
    println!("Collected {} spans", mem_exp.span_count());
    for span in mem_exp.spans() {
        let data = span.lock().unwrap();
        println!("  span: {} ({}ns)", data.name, data.duration_ns().unwrap_or(0));
    }
}
```

ผลลัพธ์จากการรัน `cargo run`:

```
=== Distributed Tracing Demo ===

--- Demo 1: Basic Span ---
{"attributes":{"db.statement":"SELECT * FROM users","db.system":"postgresql"},
"duration_ns":1980,"end_time_ns":1790621694377451533,
"events":[{"name":"query.start","timestamp_ns":1790621694377451215}],
"name":"db.query","parent_span_id":"5d58a572609697c9",
"span_id":"d7e5e1b753156413","start_time_ns":1790621694377449553,
"status":"Unset","trace_id":"a57ac4d594ae888d6bf60b910dfbf9ed"}
{"attributes":{"http.method":"GET","http.url":"/api/users"},
"duration_ns":81631,"end_time_ns":1790621694377514527,
"events":[{"name":"request.received","timestamp_ns":1790621694377447934}],
"name":"handle-request","parent_span_id":null,
"span_id":"5d58a572609697c9","start_time_ns":1790621694377432896,
"status":"Unset","trace_id":"a57ac4d594ae888d6bf60b910dfbf9ed"}

--- Demo 2: Context Propagation ---
Injected traceparent: 00-9bb8ac55cc2b55d686ecb12b158fd1d3-fa0b0bda40ed2ad6-01
Extracted trace_id: 9bb8ac55cc2b55d686ecb12b158fd1d3
Extracted span_id:  fa0b0bda40ed2ad6
Is sampled: true

--- Demo 3: InMemory Exporter ---
Collected 5 spans
  span: op-0 (482ns)
  span: op-1 (328ns)
  span: op-2 (296ns)
  span: op-3 (301ns)
  span: op-4 (292ns)
```

สังเกตว่า `db.query` span ถูก export ก่อน `handle-request` เพราะ inner span จบก่อน ทั้งสองมี `trace_id` เดียวกัน และ `db.query` มี `parent_span_id` ตรงกับ `span_id` ของ `handle-request`

---

### ขั้นที่ 8: BatchExporter — Async Buffering ด้วย tokio

สำหรับ production workload การส่ง span ทุกครั้งที่ end จะเพิ่ม latency ให้กับ request หลัก `BatchExporter` แก้ปัญหานี้ด้วยการ buffer spans ไว้แล้วส่งเป็น batch

**เพิ่มใน `Cargo.toml`:**

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rand = "0.8"
```

**`src/batch_exporter.rs`** (extension):

```rust
use std::sync::{Arc, Mutex};
use std::time::Duration;
use crate::span::SpanData;
use crate::exporter::SpanExporter;

/// BatchExporter — รวม spans เป็น batch แล้วส่งพร้อมกัน
/// ลด I/O overhead เมื่อมี spans จำนวนมากต่อวินาที
pub struct BatchExporter {
    buffer: Mutex<Vec<SpanSnapshot>>,
    max_batch_size: usize,
    inner: Arc<dyn SpanExporter>,
}

/// Snapshot ของ span data (clone สำหรับ buffer)
struct SpanSnapshot {
    name: String,
    trace_id: String,
    span_id: String,
    duration_ns: Option<u64>,
}

impl BatchExporter {
    pub fn new(inner: Arc<dyn SpanExporter>, max_batch_size: usize) -> Self {
        BatchExporter {
            buffer: Mutex::new(Vec::new()),
            max_batch_size,
            inner,
        }
    }

    /// Flush spans ใน buffer ไปยัง inner exporter
    pub fn flush(&self) {
        let mut buf = self.buffer.lock().unwrap();
        if buf.is_empty() { return; }
        println!("[BatchExporter] flushing {} spans", buf.len());
        // ในระบบจริง: ส่งไปยัง OTLP endpoint หรือ backend
        buf.clear();
    }

    pub fn buffered_count(&self) -> usize {
        self.buffer.lock().unwrap().len()
    }
}

impl SpanExporter for BatchExporter {
    fn export(&self, span: &SpanData) {
        let snapshot = SpanSnapshot {
            name: span.name.clone(),
            trace_id: span.span_context.trace_id.to_hex(),
            span_id: span.span_context.span_id.to_hex(),
            duration_ns: span.duration_ns(),
        };

        let should_flush = {
            let mut buf = self.buffer.lock().unwrap();
            buf.push(snapshot);
            buf.len() >= self.max_batch_size
        };

        if should_flush {
            self.flush();
        }
    }
}

// ใน production จะใช้ tokio::spawn periodic flush:
// ```
// tokio::spawn(async move {
//     let mut interval = tokio::time::interval(Duration::from_secs(5));
//     loop {
//         interval.tick().await;
//         exporter.flush();
//     }
// });
// ```
```

---

## การทดสอบ (Testing)

โปรเจคมี 29 unit tests ครอบคลุมทุก component หลัก

### Tests ครบทุก module

```rust
// context::tests
#[test]
fn test_trace_id_hex_roundtrip() {
    let id = TraceId::random();
    let restored = TraceId::from_hex(&id.to_hex()).unwrap();
    assert_eq!(id, restored);
}

#[test]
fn test_thread_local_context() {
    let ctx = SpanContext::new(TraceId::random(), SpanId::random(), TraceFlags::SAMPLED);
    set_current_context(ctx.clone());
    let got = get_current_context();
    assert_eq!(got.trace_id, ctx.trace_id);
    clear_current_context();
    assert!(!get_current_context().is_valid());
}

// span::tests
#[test]
fn test_root_span_creation() {
    let span = SpanBuilder::new("root-op").start();
    assert!(span.context().is_valid());
    assert!(span.parent_span_id().is_none(), "root span must have no parent");
}

#[test]
fn test_child_span_parent_link() {
    let parent_ctx = SpanContext::new(TraceId::random(), SpanId::random(), TraceFlags::SAMPLED);
    let parent_trace_id = parent_ctx.trace_id;
    let parent_span_id = parent_ctx.span_id;

    let child = SpanBuilder::new("child").with_parent(parent_ctx).start();
    assert_eq!(child.context().trace_id, parent_trace_id);
    assert_eq!(child.parent_span_id(), Some(parent_span_id));
}

#[test]
fn test_raii_guard_ends_span() {
    let exporter = Arc::new(InMemoryExporter::new());
    {
        let _span = SpanBuilder::new("auto-end")
            .with_exporter(exporter.clone())
            .start();
        // _span drop อยู่ที่นี่ → end() ถูกเรียก
    }
    let spans = exporter.spans();
    assert_eq!(spans.len(), 1);
    assert!(spans[0].lock().unwrap().end_time.is_some());
}

// propagation::tests
#[test]
fn test_parse_traceparent_valid() {
    let tp = "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01";
    let ctx = parse_traceparent(tp).unwrap();
    assert_eq!(ctx.trace_id.to_hex(), "4bf92f3577b34da6a3ce929d0e0e4736");
    assert_eq!(ctx.span_id.to_hex(), "00f067aa0ba902b7");
    assert!(ctx.trace_flags.is_sampled());
}

// sampler::tests
#[test]
fn test_ratio_based_approximate_rate() {
    let sampler = TraceIdRatioBased::new(0.5);
    let total = 10000u32;
    let sampled = (0..total)
        .filter(|_| sampler.should_sample(&TraceId::random()))
        .count() as f64;
    let rate = sampled / total as f64;
    assert!(rate > 0.40 && rate < 0.60);
}
```

### ผลลัพธ์จาก `cargo test` (verbatim)

```
$ cargo test

   Compiling dist-tracing v0.1.0
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.96s
     Running unittests src/lib.rs (target/debug/deps/dist_tracing-bcaa99f93de31e55)

running 29 tests
test context::tests::test_span_context_invalid ... ok
test context::tests::test_thread_local_context ... ok
test context::tests::test_span_id_hex_roundtrip ... ok
test context::tests::test_span_context_valid ... ok
test context::tests::test_trace_id_hex_roundtrip ... ok
test exporter::tests::test_in_memory_exporter_collects_spans ... ok
test exporter::tests::test_in_memory_exporter_span_has_end_time ... ok
test propagation::tests::test_baggage_inject_extract ... ok
test propagation::tests::test_extract_traceparent ... ok
test propagation::tests::test_inject_extract_roundtrip ... ok
test propagation::tests::test_parse_traceparent_not_sampled ... ok
test propagation::tests::test_parse_traceparent_invalid_format ... ok
test propagation::tests::test_parse_traceparent_valid ... ok
test sampler::tests::test_always_off_samples_none ... ok
test sampler::tests::test_always_on_samples_all ... ok
test propagation::tests::test_inject_traceparent ... ok
test sampler::tests::test_ratio_based_zero_samples_none ... ok
test sampler::tests::test_ratio_based_one_samples_all ... ok
test sampler::tests::test_ratio_based_deterministic ... ok
test span::tests::test_child_span_parent_link ... ok
test span::tests::test_raii_guard_ends_span ... ok
test span::tests::test_root_span_creation ... ok
test span::tests::test_span_attributes ... ok
test span::tests::test_span_events ... ok
test span::tests::test_span_status ... ok
test tracer::tests::test_tracer_creates_root_span ... ok
test tracer::tests::test_tracer_in_span_raii ... ok
test tracer::tests::test_tracer_nested_spans ... ok
test sampler::tests::test_ratio_based_approximate_rate ... ok

test result: ok. 29 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s

     Running unittests src/main.rs (target/debug/deps/dist_tracing-b63495e19dec2bd3)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests dist_tracing

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ผ่านทั้ง 29 tests ใน 0.01 วินาที

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/dist-tracing
```

### ใช้เป็น Library

เพิ่มใน `Cargo.toml` ของโปรเจคอื่น:

```toml
[dependencies]
dist-tracing = { path = "../dist-tracing" }
# หรือ publish ไปยัง crates.io แล้วใช้ version
dist-tracing = "0.1.0"
```

### Export ไปยัง Jaeger (ผ่าน OTLP)

ในโปรเจคจริงสามารถเพิ่ม `OtlpExporter` ที่ส่ง spans ไปยัง Jaeger ผ่าน OTLP/gRPC:

```bash
# รัน Jaeger ด้วย Docker
docker run -d --name jaeger \
  -p 16686:16686 \   # Jaeger UI
  -p 4317:4317 \     # OTLP gRPC
  jaegertracing/all-in-one:latest

# เปิด http://localhost:16686 ดู traces
```

### Docker Image

```dockerfile
FROM rust:1.75 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/dist-tracing /usr/local/bin/
CMD ["dist-tracing"]
```

---

## ข้อควรระวัง (Pitfalls)

### Pitfall 1: ลืม end() span — Memory Leak และ Missing Spans

```rust
// ผิด: span ถูก end โดย Drop แต่ถ้า export มีผลข้างเคียงแปลก ๆ...
let span = tracer.start_span("operation");
// ลืมเรียก span.end()  ← span จะถูก end เมื่อออก scope
// ถ้าใช้ in_span() จะหลีกเลี่ยงปัญหานี้ได้โดยสมบูรณ์

// ถูก: ใช้ in_span() เสมอเมื่อทำได้
tracer.in_span("operation", |span| {
    // span จะ end เมื่อ closure จบเสมอ ไม่ว่า panic หรือ early return
    do_work()?;
    Ok(())
});
```

**ผลกระทบ:** ถ้า end() ล่าช้า `start_time` ถูกบันทึกตอน start แต่ `end_time` อาจล่าช้ากว่าที่ควร ทำให้ duration ดู "ผิดปกติ"

### Pitfall 2: Thread-Local Context ไม่ข้าม Thread Boundary

```rust
// ผิด: spawn thread ใหม่โดยไม่ carry context
let span = tracer.start_span("parent");
std::thread::spawn(|| {
    // Thread ใหม่มี CURRENT_CONTEXT = invalid!
    let child = tracer.start_span("child"); // จะเป็น root span ไม่ใช่ child!
});

// ถูก: ส่ง context ผ่าน argument
let parent_ctx = span.context();
std::thread::spawn(move || {
    let child = SpanBuilder::new("child")
        .with_parent(parent_ctx.clone())
        .start();
});
```

**หมายเหตุ:** OpenTelemetry แก้ปัญหานี้ด้วย `Context` object ที่ต้องส่ง explicitly ทั้ง across threads และ across async tasks

### Pitfall 3: Deadlock จาก Double-Lock SpanData

```rust
// อันตราย: lock SpanData ใน exporter ขณะที่ Span::end() กำลัง hold lock อยู่
impl SpanExporter for BuggyExporter {
    fn export(&self, span: &SpanData) {
        // span ที่ได้รับนี้ถูก lock มาจาก Span::end() แล้ว
        // ถ้า exporter พยายาม lock span เดิม → deadlock!
        // แต่ใน implementation ของเรา export(&SpanData) รับ reference โดยตรง ไม่ต้อง re-lock
    }
}

// ที่ถูกต้องใน Span::end():
pub fn end(&mut self) {
    self.ended = true;
    {
        let mut data = self.inner.lock().unwrap();
        data.end_time = Some(now_ns());
    }   // ← drop lock ที่นี่ก่อน
    if let Some(exp) = &self.exporter {
        let data = self.inner.lock().unwrap();  // lock ใหม่
        exp.export(&data);
    }   // ← drop lock อีกครั้ง
}
```

**กฎ:** ใน Rust ต้องระวัง lock order และ lock scope เสมอ ใช้ explicit block `{}` เพื่อบังคับ drop lock ก่อนทำงานถัดไป

### Pitfall 4: Sampling Inconsistency ระหว่าง Services

```rust
// ผิด: แต่ละ service ใช้ random sampling แยกกัน
// Service A: random → sample
// Service B: random → drop ← spans หาย!

// ถูก: ใช้ TraceIdRatioBased เหมือนกันทุก service
// เพราะ trace_id เดิม → hash เดิม → ผลเดิมเสมอ
let sampler = TraceIdRatioBased::new(0.1);  // 10% sampling

// ถูก: เมื่อ extract traceparent จาก header แล้วสร้าง child span
// ต้อง respect trace_flags ที่ได้รับมาด้วย (ถ้า parent บอก sampled → child ก็ sampled)
let parent_ctx = extract_traceparent(&headers)?;
if parent_ctx.trace_flags.is_sampled() {
    // parent trace ถูก sample อยู่แล้ว → child ควร sample ด้วย
}
```

### Pitfall 5: Clock Resolution ต่ำทำให้ duration = 0

```rust
// ใน environment บางอย่าง SystemTime resolution อาจต่ำ
// ทำให้ span สั้นมาก ๆ มี duration = 0 ns

// แก้: ใช้ std::time::Instant สำหรับ duration measurement
// (แต่ Instant ไม่ serialize ได้ง่าย จึงต้องเก็บทั้ง Instant และ SystemTime)
use std::time::Instant;
let start = Instant::now();
// ...
let duration = start.elapsed().as_nanos() as u64;
```

### Pitfall 6: Arc<dyn Trait> Clone ใน Hot Path

```rust
// แต่ละ start_span() clone Arc<dyn SpanExporter> และ Arc<dyn Sampler>
// Arc::clone() เพิ่ม reference count ด้วย atomic operation
// ใน hot path (>100k spans/sec) นี้อาจเป็น bottleneck

// แก้: เก็บ Arc ไว้ใน Tracer แล้ว clone เฉพาะตอน start_span
// หรือใช้ weak reference ถ้า Span lifetime สั้นกว่า Tracer
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: OTLP Exporter

เพิ่ม `OtlpExporter` ที่ส่ง spans ไปยัง OpenTelemetry Collector ผ่าน HTTP/JSON:

```rust
pub struct OtlpExporter {
    endpoint: String,    // "http://localhost:4318/v1/traces"
    client: reqwest::blocking::Client,
}

// Hint: OTLP HTTP/JSON format:
// POST /v1/traces
// Content-Type: application/json
// { "resourceSpans": [ { "scopeSpans": [ { "spans": [...] } ] } ] }
```

**ทักษะที่ฝึก:** HTTP client, JSON serialization, OTLP protocol

### แบบฝึกหัดที่ 2: Async BatchExporter ด้วย tokio

สร้าง `AsyncBatchExporter` ที่ใช้ `tokio::sync::mpsc::channel` รับ spans แล้ว flush เป็น batch ทุก N วินาที หรือเมื่อ buffer เต็ม:

```rust
pub struct AsyncBatchExporter {
    tx: tokio::sync::mpsc::Sender<SpanSnapshot>,
}

// Background task:
// loop {
//     tokio::select! {
//         Some(span) = rx.recv() => { buffer.push(span); }
//         _ = timer.tick() => { flush_buffer().await; }
//     }
// }
```

**ทักษะที่ฝึก:** tokio channels, select!, async background tasks

### แบบฝึกหัดที่ 3: Span Processor Pipeline

เพิ่ม `SpanProcessor` layer ระหว่าง Span กับ Exporter เพื่อ transform, filter, หรือ enrich spans:

```rust
pub trait SpanProcessor: Send + Sync {
    /// เรียกเมื่อ span เริ่ม — ใช้ inject default attributes
    fn on_start(&self, span: &mut SpanData);
    /// เรียกเมื่อ span จบ — ใช้ filter หรือ transform
    fn on_end(&self, span: &SpanData) -> Option<SpanData>;
}

// ตัวอย่าง: FilterProcessor ทิ้ง spans ที่สั้นกว่า threshold
pub struct MinDurationFilter { pub min_ns: u64 }
impl SpanProcessor for MinDurationFilter {
    fn on_end(&self, span: &SpanData) -> Option<SpanData> {
        if span.duration_ns().unwrap_or(0) < self.min_ns {
            None  // ทิ้ง span นี้
        } else {
            Some(span.clone())
        }
    }
}
```

**ทักษะที่ฝึก:** Pipeline/middleware pattern, trait composition

### แบบฝึกหัดที่ 4: ParentBased Sampler

ใน production ถ้า parent span ถูก sample แล้ว child ควรถูก sample ด้วยเสมอ (เพื่อให้ trace สมบูรณ์):

```rust
pub struct ParentBased {
    root_sampler: Box<dyn Sampler>,
}

impl Sampler for ParentBased {
    fn should_sample(&self, trace_id: &TraceId) -> bool {
        // ดึง current context — ถ้ามี parent ที่ sampled → sample เสมอ
        let ctx = get_current_context();
        if ctx.is_valid() {
            ctx.trace_flags.is_sampled()
        } else {
            // root span — ใช้ root_sampler
            self.root_sampler.should_sample(trace_id)
        }
    }
    fn name(&self) -> &str { "ParentBased" }
}
```

**ทักษะที่ฝึก:** Composite sampler pattern, context propagation

### แบบฝึกหัดที่ 5: Flame Graph Generator

เขียน `flame_graph()` function ที่รับ `Vec<Arc<Mutex<SpanData>>>` จาก `InMemoryExporter` แล้วสร้าง flame graph ใน format ที่ Speedscope หรือ Flamegraph.pl รับได้:

```
handle-request;db.query 1980
handle-request 79651
```

แต่ละบรรทัด: `stack_frames duration_ns`

**ทักษะที่ฝึก:** Tree traversal, span tree reconstruction จาก parent_span_id

### แบบฝึกหัดที่ 6: Metrics ที่สกัดจาก Spans

เพิ่ม `MetricsExporter` ที่ aggregate spans เป็น metrics โดยอัตโนมัติ:

```rust
pub struct MetricsExporter {
    // span_name → (count, total_duration_ns, max_duration_ns)
    metrics: Mutex<HashMap<String, (u64, u64, u64)>>,
}

impl SpanExporter for MetricsExporter {
    fn export(&self, span: &SpanData) {
        if let Some(d) = span.duration_ns() {
            let mut m = self.metrics.lock().unwrap();
            let entry = m.entry(span.name.clone()).or_insert((0, 0, 0));
            entry.0 += 1;            // count
            entry.1 += d;            // total duration
            entry.2 = entry.2.max(d); // max duration
        }
    }
}

// report():
// operation    count  avg_ms   p100_ms
// db.query     1234   2.1      45.3
// auth.verify  1234   0.8      12.1
```

**ทักษะที่ฝึก:** Aggregation, reporting, HashMap operations

---

## สรุป

โปรเจคนี้สร้าง **Distributed Tracing Library** ที่สมบูรณ์ตั้งแต่พื้นฐาน ครอบคลุม:

| Component | สิ่งที่สร้าง | Pattern ที่ใช้ |
|-----------|-------------|----------------|
| `context` | TraceId, SpanId, SpanContext, thread-local | Newtype wrapper, thread-local storage |
| `span` | SpanData, Span, SpanBuilder | Builder pattern, RAII via Drop |
| `exporter` | SpanExporter trait, InMemoryExporter, ConsoleExporter | Trait object, Strategy pattern |
| `sampler` | Sampler trait, AlwaysOn/Off, TraceIdRatioBased | Trait object, deterministic hashing |
| `propagation` | W3C traceparent inject/extract, baggage | Serialization, standard compliance |
| `tracer` | Tracer, start_span(), in_span() | Factory pattern, RAII closure |

**Pattern สำคัญที่ได้เรียน:**

1. **RAII via Drop** — span lifecycle ที่ไม่รั่วแม้เกิด panic หรือ early return
2. **Thread-local propagation** — carry context ใน single thread โดยไม่ต้องส่งผ่าน argument
3. **Deterministic sampling** — ใช้ trace_id เป็น hash key เพื่อ consistency ระหว่าง services
4. **Open/Closed principle** — เพิ่ม exporter หรือ sampler ใหม่โดยไม่แก้ core code
5. **W3C interoperability** — format มาตรฐานที่ทำงานร่วมกับ Jaeger, Zipkin, Datadog ได้ทันที

**เชื่อมโยงไปโปรเจคถัดไป:** Project F10 จะสร้าง **Message Queue** ซึ่งเป็นอีกหนึ่ง infrastructure ที่ distributed system ต้องการ — message queue ที่ดีควรรองรับ tracing propagation ด้วยการ inject `traceparent` ลงใน message headers เพื่อ link trace ระหว่าง producer และ consumer ได้

---

**โปรเจคก่อนหน้า:** [Project F08: Consistent Hashing](project-f08-consistent-hash.md) | **โปรเจคถัดไป:** [Project F10: Message Queue](project-f10-message-queue.md)
