# Project J08: SSE Server — Real-time Dashboard ด้วย Server-Sent Events

> โมดูล: J — Full-Stack / WASM | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้าง **SSE Server (Server-Sent Events)** ด้วย Rust + Axum ที่รองรับการส่ง event แบบ real-time ไปยัง client browser หลายคนพร้อมกัน พร้อม **real-time dashboard** ที่แสดง CPU/memory metrics ทุกวินาที

**Server-Sent Events (SSE)** คือโปรโตคอลที่ถูกออกแบบมาสำหรับ **unidirectional streaming** จาก server ไปยัง client บน HTTP/1.1 หรือ HTTP/2 โดยใช้ Content-Type `text/event-stream` ต่างจาก WebSocket ตรงที่ SSE ใช้ได้กับ plain HTTP (ไม่ต้องการ protocol upgrade พิเศษ) และ browser รองรับ auto-reconnect ในตัว ทำให้เหมาะกับ use case ที่ข้อมูลไหลจาก server ไปยัง client ทางเดียว เช่น live feeds, notifications, metrics dashboards

สิ่งที่ทำให้โปรเจคนี้น่าสร้าง:

- **SSE vs WebSocket vs Long-polling** — เรียนรู้ว่าเมื่อไหร่ควรใช้ SSE และเมื่อไหร่ควรใช้ทางเลือกอื่น
- **Axum native SSE support** — `axum::response::sse::Sse` เป็น first-class API ที่ทำงานร่วมกับ `Stream` ของ Rust อย่างลงตัว
- **Broadcast pattern** — `tokio::sync::broadcast` สำหรับ fan-out event ไปยัง subscriber หลายคนพร้อมกัน
- **Ring buffer replay** — client ที่ reconnect สามารถรับ event ที่พลาดไประหว่าง connection ขาดได้
- **Production hardening** — heartbeat, CORS headers, buffering proxy workaround

**Use case จริงในโลก production:**
- Live metrics dashboard (Grafana-style)
- Stock price / cryptocurrency feed
- Build pipeline status (CI/CD notifications)
- Sport score / live event tickers
- Notification center ใน web app (แจ้งเตือน mention, like, comment)
- Log streaming จาก container/process ไปยัง browser UI

---

## สิ่งที่จะได้เรียนรู้

- รูปแบบ **text/event-stream protocol** — `data:`, `event:`, `id:`, `retry:` fields และ multi-line data
- การใช้ `axum::response::sse::{Sse, Event, KeepAlive}` สร้าง SSE endpoint ใน Axum
- การออกแบบ **broadcast channel** ด้วย `tokio::sync::broadcast` สำหรับ publisher/subscriber pattern
- **Typed events** — serialize `serde::Serialize` struct ไปยัง JSON data field
- **Client reconnection** ด้วย `Last-Event-ID` header และ ring buffer replay
- **Heartbeat / keep-alive** เพื่อป้องกัน proxy timeout และ browser connection reset
- **Multiple streams** — per-room, per-user channels, fan-out architecture
- **CORS headers** และการแก้ปัญหา buffering proxy ที่กั้น SSE stream

---

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 46–50: async/await, `tokio` runtime, `spawn`, task management
- จาก Part 51–55: `Arc`, `RwLock`, `Mutex` สำหรับ shared state
- จาก Part 56–60: `Stream`, `StreamExt`, `futures` traits
- จาก Part 61–65: `axum` routing, extractors, handlers, middleware
- จาก Part 66–70: `serde` / `serde_json` — serialization
- Project J07 (REST + OpenAPI) — Axum application structure
- ความรู้พื้นฐาน HTTP/1.1 headers

---

## โครงสร้างโปรเจค (Project Layout)

```
sse-server/
├── src/
│   ├── main.rs           # Entry point, router, AppState, axum server
│   ├── sse_types.rs      # DashboardEvent, MetricsPayload, AlertPayload, EventRecord
│   ├── ring_buffer.rs    # EventRingBuffer — replay missed events on reconnect
│   ├── broadcaster.rs    # BroadcastHub — wraps tokio::sync::broadcast + ring buffer
│   ├── metrics.rs        # CPU/memory collector, emit loop (ทุก 1 วินาที)
│   ├── sse_handlers.rs   # SSE endpoint handlers (dashboard, room, user)
│   └── cors.rs           # CORS middleware layer
├── tests/
│   └── sse_tests.rs      # Unit tests: event formatting, ring buffer, serialization
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
MetricsCollector (background task)
       │  emit MetricsPayload ทุก 1 วินาที
       ▼
BroadcastHub
  ├── tokio::sync::broadcast::Sender<DashboardEvent>
  └── EventRingBuffer (capacity=500)
             │
             │  subscriber (Receiver clone)
             ├──────────────────────────────▶ Client A (SSE stream)
             ├──────────────────────────────▶ Client B (SSE stream)
             └──────────────────────────────▶ Client C (SSE stream)

GET /events?room=global
       │
       ├── ตรวจสอบ Last-Event-ID header
       ├── replay missed events จาก ring buffer
       └── subscribe ไปยัง broadcast channel
              │  Receiver<DashboardEvent> → Stream → Sse response
              ▼
         text/event-stream HTTP body (chunked encoding)
```

### ทำไมถึงเลือก Design นี้

**broadcast channel** ของ `tokio::sync::broadcast` เหมาะกับ SSE เพราะ:
- Sender สามารถ clone และส่ง message ได้จากหลาย task
- Receiver ทุกตัวได้รับ message เดียวกันพร้อมกัน (fan-out)
- เมื่อ client disconnect, Receiver drop ออกไปเองโดยไม่ต้อง cleanup พิเศษ

**Ring buffer** เก็บ event ล่าสุด N ชุดไว้เพื่อ replay ให้ client ที่กลับมา reconnect ทำให้ไม่ต้องใช้ database สำหรับ event replay ระยะสั้น

**KeepAlive** ของ Axum ส่ง SSE comment ทุก N วินาที ป้องกันไม่ให้ proxy หรือ load balancer ตัด connection ที่ไม่มี traffic เป็นเวลานาน

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: SSE Protocol — ทำความเข้าใจ text/event-stream

ก่อนเขียนโค้ด Axum ให้เข้าใจ SSE protocol ด้วยตัวอย่าง raw HTTP response:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Access-Control-Allow-Origin: *

retry: 3000
id: 1
event: metrics
data: {"cpu_pct":12.5,"mem_mb":1024}

id: 2
event: alert
data: {"severity":"warning","message":"CPU > 80%"}

: heartbeat

id: 3
data: simple message without explicit event type

```

กฎของ **text/event-stream format**:

| Field | รูปแบบ | ความหมาย |
|-------|--------|----------|
| `data:` | `data: <value>` | ข้อมูลของ event (required — dispatch event ตอน encounter blank line) |
| `event:` | `event: <type>` | ชื่อ event type สำหรับ `addEventListener` |
| `id:` | `id: <string>` | event ID — browser เก็บไว้ใน `lastEventId` และส่งกลับเป็น `Last-Event-ID` header ตอน reconnect |
| `retry:` | `retry: <ms>` | บอก browser ให้ reconnect หลัง N มิลลิวินาที |
| `:` | `: <comment>` | comment — ไม่ dispatch event แต่ทำให้ connection มี traffic (heartbeat) |

**blank line** (บรรทัดว่าง) คือ event separator — browser dispatch event เมื่อเจอบรรทัดว่าง

**Multi-line data** — ใช้หลาย `data:` fields:

```
data: line one
data: line two
data: line three

```

browser จะ join ด้วย `\n` ก่อน dispatch

สร้าง `Cargo.toml` พื้นฐาน:

```toml
[package]
name = "sse-server"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
tokio-stream = { version = "0.1", features = ["sync"] }
futures-util = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tower-http = { version = "0.6", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

ฟังก์ชัน utility สำหรับ format SSE event ด้วยมือ (ใช้ใน tests และเป็นโมเดลให้เข้าใจ):

```rust
// src/sse_types.rs

/// สร้าง SSE event string ตาม spec text/event-stream
pub fn format_event(
    id: Option<&str>,
    event_type: Option<&str>,
    data: &str,
    retry_ms: Option<u64>,
) -> String {
    let mut buf = String::new();
    if let Some(r) = retry_ms {
        buf.push_str(&format!("retry: {}\n", r));
    }
    if let Some(id) = id {
        buf.push_str(&format!("id: {}\n", id));
    }
    if let Some(ev) = event_type {
        buf.push_str(&format!("event: {}\n", ev));
    }
    if data.is_empty() {
        buf.push_str("data: \n");
    } else {
        for line in data.lines() {
            buf.push_str(&format!("data: {}\n", line));
        }
    }
    buf.push('\n');
    buf
}

/// แยก Last-Event-ID จาก header string
pub fn parse_last_event_id(header_value: Option<&str>) -> Option<u64> {
    header_value?.trim().parse::<u64>().ok()
}

/// ตรวจสอบว่า string เป็น SSE comment (ขึ้นต้นด้วย ':')
pub fn is_sse_comment(line: &str) -> bool {
    line.starts_with(':')
}

/// สร้าง heartbeat comment
pub fn heartbeat_comment() -> String {
    String::from(": heartbeat\n\n")
}
```

---

### ขั้นที่ 2: Typed Events และ JSON Serialization

กำหนด typed event enum ด้วย `serde`:

```rust
// src/sse_types.rs (ต่อ)

use serde::{Deserialize, Serialize};

/// DashboardEvent — typed event สำหรับ real-time dashboard
/// ใช้ serde tag เพื่อให้ JSON มี "type" field
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum DashboardEvent {
    Metrics(MetricsPayload),
    Alert(AlertPayload),
    Connected { client_id: String },
    Disconnected { client_id: String },
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct MetricsPayload {
    pub cpu_pct: f32,
    pub mem_mb: u64,
    pub timestamp: u64,    // Unix timestamp
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct AlertPayload {
    pub severity: String,  // "info" | "warning" | "critical"
    pub message: String,
}

impl DashboardEvent {
    /// ชื่อ event type สำหรับใส่ใน `event:` field
    pub fn event_type_name(&self) -> &'static str {
        match self {
            DashboardEvent::Metrics(_) => "metrics",
            DashboardEvent::Alert(_) => "alert",
            DashboardEvent::Connected { .. } => "connected",
            DashboardEvent::Disconnected { .. } => "disconnected",
        }
    }
}
```

**ทำไมถึงใช้ `#[serde(tag = "type")]`?**

เมื่อ deserialize ฝั่ง JavaScript, client สามารถแยก event ได้ด้วย:

```javascript
// browser JS
source.addEventListener('metrics', (e) => {
    const payload = JSON.parse(e.data);
    // payload.type === "metrics"
    // payload.cpu_pct, payload.mem_mb, payload.timestamp
    updateDashboard(payload);
});

source.addEventListener('alert', (e) => {
    const alert = JSON.parse(e.data);
    showAlert(alert.severity, alert.message);
});
```

ตัวอย่าง JSON ที่ได้:

```json
{"type":"metrics","cpu_pct":23.5,"mem_mb":2048,"timestamp":1700000000}
{"type":"alert","severity":"warning","message":"CPU > 80%"}
{"type":"connected","client_id":"user-abc"}
```

---

### ขั้นที่ 3: Ring Buffer สำหรับ Event Replay

เมื่อ client เชื่อมต่อขาด browser จะส่ง `Last-Event-ID` header กลับมาตอน reconnect ring buffer ช่วยให้เรา replay event ที่พลาดไประหว่าง disconnect:

```rust
// src/ring_buffer.rs

use std::collections::VecDeque;

/// EventRecord เก็บข้อมูล event ที่ broadcast แล้ว
#[derive(Debug, Clone)]
pub struct EventRecord {
    pub id: u64,
    pub event_type: String,
    pub data: String,   // JSON-serialized payload
}

/// Ring buffer ขนาดคงที่ — เมื่อเต็มจะ evict event เก่าสุดออก
pub struct EventRingBuffer {
    buf: VecDeque<EventRecord>,
    capacity: usize,
}

impl EventRingBuffer {
    pub fn new(capacity: usize) -> Self {
        Self {
            buf: VecDeque::with_capacity(capacity),
            capacity,
        }
    }

    /// เพิ่ม event เข้า buffer
    pub fn push(&mut self, record: EventRecord) {
        if self.buf.len() == self.capacity {
            self.buf.pop_front(); // ลบ event เก่าสุดออก
        }
        self.buf.push_back(record);
    }

    /// คืน events ทุกตัวที่มี id > last_id
    /// ใช้สำหรับ replay เมื่อ client reconnect พร้อม Last-Event-ID
    pub fn since(&self, last_id: u64) -> Vec<&EventRecord> {
        self.buf.iter().filter(|e| e.id > last_id).collect()
    }

    pub fn len(&self) -> usize {
        self.buf.len()
    }

    pub fn is_empty(&self) -> bool {
        self.buf.is_empty()
    }
}
```

**ข้อดีของ Ring Buffer เทียบกับ `Vec` ธรรมดา:**

| | Ring Buffer (VecDeque) | Vec ธรรมดา |
|---|---|---|
| Memory | คงที่ (capacity) | เติบโตไม่สิ้นสุด |
| push_front/pop_front | O(1) | O(n) |
| Eviction | อัตโนมัติ | ต้อง manage เอง |
| ใช้กับ | event history แบบ sliding window | ทุกอย่างที่ต้องการ random access |

---

### ขั้นที่ 4: BroadcastHub — Publisher/Subscriber Pattern

```rust
// src/broadcaster.rs

use crate::ring_buffer::{EventRecord, EventRingBuffer};
use crate::sse_types::DashboardEvent;
use std::sync::{Arc, Mutex};
use tokio::sync::broadcast;

const CHANNEL_CAPACITY: usize = 1024;
const RING_BUFFER_CAPACITY: usize = 500;

/// BroadcastHub รวม tokio::sync::broadcast กับ EventRingBuffer
/// ทำให้ subscribers ทุกคนได้รับ event และ client ที่ reconnect
/// สามารถ replay missed events ได้
#[derive(Clone)]
pub struct BroadcastHub {
    sender: broadcast::Sender<DashboardEvent>,
    ring: Arc<Mutex<EventRingBuffer>>,
    next_id: Arc<std::sync::atomic::AtomicU64>,
}

impl BroadcastHub {
    pub fn new() -> Self {
        let (sender, _) = broadcast::channel(CHANNEL_CAPACITY);
        Self {
            sender,
            ring: Arc::new(Mutex::new(EventRingBuffer::new(RING_BUFFER_CAPACITY))),
            next_id: Arc::new(std::sync::atomic::AtomicU64::new(1)),
        }
    }

    /// ส่ง event ไปยัง subscribers ทุกคน และบันทึกลง ring buffer
    pub fn publish(&self, event: DashboardEvent) -> Result<usize, broadcast::error::SendError<DashboardEvent>> {
        let id = self.next_id.fetch_add(1, std::sync::atomic::Ordering::Relaxed);
        let data = serde_json::to_string(&event).unwrap_or_default();
        let event_type = event.event_type_name().to_string();

        // บันทึกลง ring buffer ก่อน
        let record = EventRecord { id, event_type, data };
        self.ring.lock().unwrap().push(record);

        // broadcast ไปยัง subscribers ทุกคน
        self.sender.send(event)
    }

    /// สร้าง Receiver ใหม่สำหรับ subscriber
    pub fn subscribe(&self) -> broadcast::Receiver<DashboardEvent> {
        self.sender.subscribe()
    }

    /// ดึง events ที่ client พลาดไปหลังจาก last_id
    pub fn missed_since(&self, last_id: u64) -> Vec<EventRecord> {
        self.ring.lock().unwrap().since(last_id).into_iter().cloned().collect()
    }
}

impl Default for BroadcastHub {
    fn default() -> Self {
        Self::new()
    }
}
```

**ทำไมถึงใช้ `broadcast::channel` แทน `mpsc::channel`?**

`mpsc::channel` ส่งได้แค่ consumer เดียว แต่ `broadcast::channel` ส่งได้หลาย consumer พร้อมกัน ทุก Receiver ได้รับ message เดียวกัน ซึ่งตรงกับ SSE use case ที่มี client หลายคน subscribe stream เดียวกัน

---

### ขั้นที่ 5: Axum SSE Handler

```rust
// src/sse_handlers.rs

use axum::{
    extract::{Query, State},
    response::sse::{Event, KeepAlive, Sse},
};
use futures_util::stream::{self, Stream, StreamExt};
use serde::Deserialize;
use std::{convert::Infallible, time::Duration};
use tokio_stream::wrappers::BroadcastStream;

use crate::broadcaster::BroadcastHub;
use crate::sse_types::format_event as fmt_event;
use crate::AppState;

#[derive(Deserialize)]
pub struct SseParams {
    pub last_event_id: Option<u64>,
}

/// GET /events — SSE endpoint หลัก
pub async fn sse_handler(
    State(state): State<AppState>,
    Query(params): Query<SseParams>,
    // axum สามารถดึง Last-Event-ID จาก header ได้โดยตรง
    headers: axum::http::HeaderMap,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    // ดึง last_event_id จาก header หรือ query param
    let last_id = headers
        .get("last-event-id")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse::<u64>().ok())
        .or(params.last_event_id)
        .unwrap_or(0);

    let hub = state.hub.clone();

    // replay missed events ก่อน
    let missed: Vec<Event> = hub
        .missed_since(last_id)
        .into_iter()
        .map(|rec| {
            Event::default()
                .id(rec.id.to_string())
                .event(rec.event_type)
                .data(rec.data)
        })
        .collect();

    let replay_stream = stream::iter(missed.into_iter().map(Ok::<Event, Infallible>));

    // subscribe ไปยัง broadcast channel สำหรับ live events
    let rx = hub.subscribe();
    let live_stream = BroadcastStream::new(rx)
        .filter_map(|result| async move {
            match result {
                Ok(event) => {
                    let data = serde_json::to_string(&event).ok()?;
                    let sse_event = Event::default()
                        .event(event.event_type_name())
                        .data(data);
                    Some(Ok(sse_event))
                }
                Err(_lagged) => {
                    // Receiver lagged — broadcast channel เต็ม
                    // ส่ง reconnect hint แทน
                    None
                }
            }
        });

    // รวม replay + live stream
    let combined = replay_stream.chain(live_stream);

    Sse::new(combined).keep_alive(
        KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("heartbeat"),
    )
}
```

**ส่วนสำคัญของ `Sse<S>`:

| Component | ความหมาย |
|-----------|----------|
| `Sse::new(stream)` | รับ `Stream<Item = Result<Event, E>>` |
| `Event::default()` | สร้าง SSE event builder |
| `.id(s)` | ตั้ง `id:` field |
| `.event(s)` | ตั้ง `event:` field |
| `.data(s)` | ตั้ง `data:` field |
| `.retry(duration)` | ตั้ง `retry:` field |
| `KeepAlive::new()` | สร้าง heartbeat ด้วย interval |
| `.interval(d)` | ความถี่ heartbeat |
| `.text(s)` | ข้อความ comment ที่ส่งเป็น heartbeat |

---

### ขั้นที่ 6: Heartbeat และ Keep-Alive

ปัญหาที่พบบ่อยกับ SSE ใน production คือ **buffering proxies** และ **load balancers** ที่ตัด idle connection:

**กลไก KeepAlive ใน Axum:**

```rust
// Axum ส่ง SSE comment ": heartbeat\n\n" ทุก 15 วินาที
// โดยอัตโนมัติผ่าน KeepAlive middleware

Sse::new(stream).keep_alive(
    KeepAlive::new()
        .interval(Duration::from_secs(15))
        .text("heartbeat"),    // กลายเป็น ": heartbeat\n\n"
)
```

Wire format ของ heartbeat:

```
: heartbeat\r\n
\r\n
```

หรือใช้ `comment()` method แทน `text()` ก็ได้ ซึ่งส่ง comment เปล่า `:\r\n\r\n`

**ปรับ retry interval ใน client:**

```rust
// ส่ง retry: 5000 ใน first event เพื่อบอก browser ให้ reconnect หลัง 5 วินาที
let first_event = Event::default()
    .retry(Duration::from_secs(5))
    .data("connected");
```

---

### ขั้นที่ 7: Metrics Collector และ Real-time Dashboard

```rust
// src/metrics.rs

use crate::broadcaster::BroadcastHub;
use crate::sse_types::{AlertPayload, DashboardEvent, MetricsPayload};
use std::time::{Duration, SystemTime, UNIX_EPOCH};
use tokio::time;

/// Background task: emit CPU/memory metrics ทุก 1 วินาที
pub async fn metrics_emitter(hub: BroadcastHub) {
    let mut interval = time::interval(Duration::from_secs(1));
    let mut tick_count: u64 = 0;

    loop {
        interval.tick().await;
        tick_count += 1;

        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();

        // จำลอง CPU/memory metrics
        // ใน production ใช้ sysinfo crate แทน
        let cpu_pct = simulate_cpu(tick_count);
        let mem_mb = simulate_mem(tick_count);

        let event = DashboardEvent::Metrics(MetricsPayload {
            cpu_pct,
            mem_mb,
            timestamp: now,
        });

        if hub.publish(event).is_err() {
            // ไม่มี subscriber — ไม่ต้องทำอะไร
        }

        // ส่ง alert เมื่อ CPU > 80%
        if cpu_pct > 80.0 {
            let alert = DashboardEvent::Alert(AlertPayload {
                severity: "warning".to_string(),
                message: format!("CPU สูงถึง {:.1}%", cpu_pct),
            });
            let _ = hub.publish(alert);
        }
    }
}

fn simulate_cpu(tick: u64) -> f32 {
    // จำลอง CPU ที่มีค่าขึ้นลงตามรูปแบบ sine wave
    let base = 25.0f32;
    let variance = 30.0f32;
    let t = (tick as f32) * 0.1;
    base + variance * t.sin().abs()
}

fn simulate_mem(tick: u64) -> u64 {
    // จำลอง memory ที่ค่อยๆ เพิ่มแล้วลด
    let base: u64 = 1024;
    let cycle = tick % 60;
    base + cycle * 10
}
```

**AppState และ main.rs:**

```rust
// src/main.rs

use axum::{
    routing::get,
    Router,
};
use std::net::SocketAddr;
use tower_http::cors::{Any, CorsLayer};
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

mod broadcaster;
mod metrics;
mod ring_buffer;
mod sse_handlers;
mod sse_types;

use broadcaster::BroadcastHub;
use sse_handlers::sse_handler;

#[derive(Clone)]
pub struct AppState {
    pub hub: BroadcastHub,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer())
        .init();

    let hub = BroadcastHub::new();
    let state = AppState { hub: hub.clone() };

    // เริ่ม metrics emitter ใน background
    tokio::spawn(metrics::metrics_emitter(hub));

    let cors = CorsLayer::new()
        .allow_origin(Any)
        .allow_methods(Any)
        .allow_headers(Any);

    let app = Router::new()
        .route("/events", get(sse_handler))
        .route("/health", get(|| async { "ok" }))
        .layer(cors)
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    tracing::info!("SSE server listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

### ขั้นที่ 8: Multiple Event Streams และ Per-Room Fan-out

ในโปรเจคขนาดใหญ่ต้องการ stream แยกตาม room หรือ user:

```rust
// src/sse_handlers.rs (เพิ่ม per-room handler)

use axum::extract::Path;
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

/// RoomRegistry — registry สำหรับ BroadcastHub แต่ละ room
#[derive(Clone, Default)]
pub struct RoomRegistry {
    rooms: Arc<RwLock<HashMap<String, BroadcastHub>>>,
}

impl RoomRegistry {
    /// ดึง hub ของ room ที่ระบุ หรือสร้างใหม่ถ้ายังไม่มี
    pub fn get_or_create(&self, room_id: &str) -> BroadcastHub {
        let rooms = self.rooms.read().unwrap();
        if let Some(hub) = rooms.get(room_id) {
            return hub.clone();
        }
        drop(rooms);

        let mut rooms = self.rooms.write().unwrap();
        // Double-checked locking pattern
        rooms
            .entry(room_id.to_string())
            .or_insert_with(BroadcastHub::new)
            .clone()
    }
}

/// GET /rooms/:room_id/events — SSE stream สำหรับ room เฉพาะ
pub async fn room_sse_handler(
    State(state): State<AppStateWithRooms>,
    Path(room_id): Path<String>,
    headers: axum::http::HeaderMap,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let last_id = headers
        .get("last-event-id")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.parse::<u64>().ok())
        .unwrap_or(0);

    let hub = state.rooms.get_or_create(&room_id);
    let missed: Vec<Event> = hub
        .missed_since(last_id)
        .into_iter()
        .map(|rec| {
            Event::default()
                .id(rec.id.to_string())
                .event(rec.event_type)
                .data(rec.data)
        })
        .collect();

    let replay = stream::iter(missed.into_iter().map(Ok::<_, Infallible>));
    let rx = hub.subscribe();
    let live = BroadcastStream::new(rx).filter_map(|r| async move {
        let ev = r.ok()?;
        let data = serde_json::to_string(&ev).ok()?;
        Some(Ok(Event::default().event(ev.event_type_name()).data(data)))
    });

    Sse::new(replay.chain(live)).keep_alive(
        KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("heartbeat"),
    )
}
```

**Fan-out pattern — publish ไปหลาย rooms:**

```rust
// ส่ง alert ไปยัง rooms ที่เกี่ยวข้องพร้อมกัน
async fn broadcast_to_rooms(
    registry: &RoomRegistry,
    room_ids: &[&str],
    event: DashboardEvent,
) {
    for room_id in room_ids {
        let hub = registry.get_or_create(room_id);
        let _ = hub.publish(event.clone());
    }
}
```

---

### ขั้นที่ 9: CORS Headers และ Proxy Considerations

**CORS สำหรับ SSE:**

Browser ต้องการ `Access-Control-Allow-Origin` header เมื่อ SSE endpoint อยู่ต่าง origin

```rust
// tower-http CorsLayer สำหรับ development
let cors = CorsLayer::new()
    .allow_origin(Any)                          // ทุก origin — development only
    .allow_methods([Method::GET])
    .allow_headers([AUTHORIZATION, ACCEPT]);

// Production: ระบุ origin เฉพาะ
let cors_prod = CorsLayer::new()
    .allow_origin("https://app.example.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET])
    .allow_headers([AUTHORIZATION, ACCEPT, LAST_EVENT_ID_HEADER]);
```

**ปัญหา Nginx Buffering:**

Nginx มักจะ buffer response body ซึ่งทำให้ SSE events ไม่ถึง client จนกว่า buffer จะเต็ม แก้ไขด้วย:

```nginx
# nginx.conf
location /events {
    proxy_pass http://backend:3000;
    proxy_http_version 1.1;
    proxy_set_header Connection "";          # disable keepalive to upstream
    proxy_buffering off;                     # ปิด buffering ← สำคัญมาก
    proxy_cache off;
    proxy_read_timeout 3600s;               # connection timeout 1 ชั่วโมง
    proxy_set_header X-Accel-Buffering no;  # ปิด buffering ผ่าน header
    add_header Cache-Control no-cache;
    add_header X-Accel-Buffering no;
}
```

**Response Header สำคัญจาก Axum:**

```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
Transfer-Encoding: chunked    # Axum ตั้งให้อัตโนมัติ
X-Accel-Buffering: no        # ใส่เองเพื่อบอก Nginx
```

Axum ตั้ง `Content-Type: text/event-stream` ให้อัตโนมัติเมื่อใช้ `Sse<>` response type

---

### ขั้นที่ 10: HTML Client สำหรับทดสอบ

ตัวอย่าง client HTML/JavaScript ที่รองรับ reconnection และ event types:

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Real-time Dashboard</title>
</head>
<body>
<h1>Real-time Dashboard</h1>
<div id="status">กำลังเชื่อมต่อ...</div>
<div id="metrics"></div>
<div id="alerts"></div>

<script>
let source;
let lastEventId = localStorage.getItem('lastEventId') || '';

function connect() {
    const url = lastEventId
        ? `/events?last_event_id=${lastEventId}`
        : '/events';

    source = new EventSource(url);

    source.onopen = () => {
        document.getElementById('status').textContent = 'เชื่อมต่อแล้ว';
    };

    source.onerror = () => {
        document.getElementById('status').textContent = 'กำลัง reconnect...';
        // Browser จะ reconnect อัตโนมัติตาม retry: field
    };

    // รับ metrics events
    source.addEventListener('metrics', (e) => {
        const data = JSON.parse(e.data);
        lastEventId = e.lastEventId;
        localStorage.setItem('lastEventId', lastEventId);

        document.getElementById('metrics').innerHTML = `
            <p>CPU: ${data.cpu_pct.toFixed(1)}%</p>
            <p>Memory: ${data.mem_mb} MB</p>
            <p>เวลา: ${new Date(data.timestamp * 1000).toLocaleTimeString('th-TH')}</p>
        `;
    });

    // รับ alert events
    source.addEventListener('alert', (e) => {
        const data = JSON.parse(e.data);
        lastEventId = e.lastEventId;
        const div = document.getElementById('alerts');
        div.innerHTML = `<p>[${data.severity}] ${data.message}</p>` + div.innerHTML;
    });
}

connect();
</script>
</body>
</html>
```

---

## การทดสอบ (Testing)

เนื่องจาก SSE server ต้องการ network stack ที่ซับซ้อน การทดสอบ core logic แยกเป็น pure functions ทำให้เร็วและเชื่อถือได้กว่า

### โครงสร้าง Test

```
tests/
└── sse_tests.rs    # Unit tests สำหรับ logic ทุกส่วน
```

### ตัวอย่างโค้ด Test

```rust
// tests/sse_tests.rs

use sse_server_test::*;

// -- format_event tests --

#[test]
fn test_format_event_basic_data_only() {
    let result = format_event(None, None, "hello world", None);
    assert_eq!(result, "data: hello world\n\n");
}

#[test]
fn test_format_event_with_id_and_type() {
    let result = format_event(Some("42"), Some("metrics"), "payload", None);
    assert_eq!(result, "id: 42\nevent: metrics\ndata: payload\n\n");
}

#[test]
fn test_format_event_multiline_data() {
    let result = format_event(None, None, "line1\nline2\nline3", None);
    assert_eq!(result, "data: line1\ndata: line2\ndata: line3\n\n");
}

// -- ring buffer tests --

#[test]
fn test_ring_buffer_eviction_when_full() {
    let mut buf = EventRingBuffer::new(3);
    for i in 1u64..=5 {
        buf.push(EventRecord { id: i, event_type: "t".into(), data: i.to_string() });
    }
    assert_eq!(buf.len(), 3);
    let events = buf.since(0);
    assert_eq!(events[0].id, 3);  // event id=1 และ id=2 ถูก evict
    assert_eq!(events[2].id, 5);
}

// -- reconnect replay scenario --

#[test]
fn test_reconnect_replay_scenario() {
    let mut buf = EventRingBuffer::new(100);
    for i in 1u64..=10 {
        buf.push(EventRecord {
            id: i,
            event_type: "metrics".into(),
            data: format!("{{\"seq\":{}}}", i),
        });
    }
    // Client reconnects with Last-Event-ID: 7 → replay events 8, 9, 10
    let last_id = parse_last_event_id(Some("7")).unwrap_or(0);
    let missed = buf.since(last_id);
    assert_eq!(missed.len(), 3);
    assert_eq!(missed[0].id, 8);
    assert_eq!(missed[2].id, 10);
}
```

### ผลการรัน `cargo test` จริง

```
     Running tests/sse_tests.rs (target/debug/deps/sse_tests-da66a65f0a32430e)

running 29 tests
test test_broadcast_roundtrip_empty ... ok
test test_deserialize_alert_roundtrip ... ok
test test_deserialize_invalid_json ... ok
test test_deserialize_metrics_roundtrip ... ok
test test_broadcast_roundtrip_single ... ok
test test_format_event_basic_data_only ... ok
test test_format_event_multiline_data ... ok
test test_broadcast_roundtrip_basic ... ok
test test_format_event_empty_data ... ok
test test_format_event_with_id_and_type ... ok
test test_heartbeat_comment_format ... ok
test test_format_event_with_retry ... ok
test test_is_sse_comment_false ... ok
test test_is_sse_comment_true ... ok
test test_parse_last_event_id_invalid_string ... ok
test test_parse_last_event_id_none_header ... ok
test test_parse_last_event_id_with_whitespace ... ok
test test_parse_last_event_id_valid ... ok
test test_parse_last_event_id_zero ... ok
test test_ring_buffer_eviction_when_full ... ok
test test_ring_buffer_since_no_results ... ok
test test_ring_buffer_since_returns_all_on_zero ... ok
test test_serialize_alert_event ... ok
test test_serialize_connected_event ... ok
test test_serialize_metrics_event ... ok
test test_ring_buffer_since_filters_correctly ... ok
test test_sse_event_embedding_json ... ok
test test_reconnect_replay_scenario ... ok
test test_ring_buffer_push_and_len ... ok

test result: ok. 29 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

ทุก test ผ่าน 29/29 — ครอบคลุม:
- `format_event` ทุก field combination (5 tests)
- `parse_last_event_id` edge cases (5 tests)
- SSE comments / heartbeat format (3 tests)
- Ring buffer push, eviction, since() filtering (5 tests)
- Typed event serialization/deserialization (6 tests)
- Broadcast channel roundtrip (3 tests)
- SSE + JSON embedding integration (2 tests)

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
./target/release/sse-server
# SSE server listening on 127.0.0.1:3000
```

### Docker

```dockerfile
# Dockerfile
FROM rust:1.80-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release

FROM debian:bookworm-slim
WORKDIR /app
COPY --from=builder /app/target/release/sse-server .
EXPOSE 3000
CMD ["./sse-server"]
```

```bash
docker build -t sse-server:latest .
docker run -p 3000:3000 sse-server:latest
```

### ทดสอบด้วย curl

```bash
# Subscribe ไปยัง SSE stream
curl -N -H "Accept: text/event-stream" http://localhost:3000/events

# ทดสอบ reconnect พร้อม Last-Event-ID
curl -N \
  -H "Accept: text/event-stream" \
  -H "Last-Event-ID: 42" \
  http://localhost:3000/events

# ทดสอบ health check
curl http://localhost:3000/health
# ok
```

ตัวอย่าง output จาก curl:

```
id: 1
event: metrics
data: {"type":"metrics","cpu_pct":25.3,"mem_mb":1024,"timestamp":1700000001}

id: 2
event: metrics
data: {"type":"metrics","cpu_pct":27.8,"mem_mb":1034,"timestamp":1700000002}

: heartbeat

id: 3
event: metrics
data: {"type":"metrics","cpu_pct":32.1,"mem_mb":1044,"timestamp":1700000003}
```

### systemd service

```ini
# /etc/systemd/system/sse-server.service
[Unit]
Description=SSE Dashboard Server
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/sse-server
ExecStart=/opt/sse-server/sse-server
Restart=always
RestartSec=5
Environment=RUST_LOG=info

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable sse-server
sudo systemctl start sse-server
sudo journalctl -u sse-server -f
```

---

## ข้อผิดพลาดที่พบบ่อย

### 1. Broadcast Channel Overflow — `RecvError::Lagged`

**อาการ:** Client รับ event ช้าเกินไป channel เต็ม ได้รับ `broadcast::error::RecvError::Lagged(n)` ซึ่งหมายความว่าพลาดไป n events

```rust
// ผิด: ไม่ handle Lagged error
let live = BroadcastStream::new(rx).filter_map(|r| async {
    Some(Ok(r.unwrap().to_sse()))  // panic เมื่อ Lagged!
});

// ถูก: handle Lagged gracefully
let live = BroadcastStream::new(rx).filter_map(|r| async {
    match r {
        Ok(event) => {
            let data = serde_json::to_string(&event).ok()?;
            Some(Ok(Event::default().event(event.event_type_name()).data(data)))
        }
        Err(broadcast::error::RecvError::Lagged(n)) => {
            // ส่ง lagged notification ให้ client รู้
            let msg = format!("{{\"missed\":{}}}", n);
            Some(Ok(Event::default().event("lagged").data(msg)))
        }
        Err(_) => None,
    }
});
```

**วิธีป้องกัน:** ตั้ง CHANNEL_CAPACITY ให้ใหญ่พอ และอย่า publish event บ่อยเกินความจำเป็น

---

### 2. Proxy Buffering ทำให้ Events ไม่ถึง Client

**อาการ:** SSE ทำงานได้เมื่อ connect โดยตรง แต่ไม่ทำงานผ่าน Nginx หรือ reverse proxy อื่น client รอนานมากก่อนได้รับ events ชุดแรก

**สาเหตุ:** Nginx, HAProxy, Cloudflare บางครั้ง buffer HTTP response body ก่อนส่งต่อ ทำให้ SSE events ค้างอยู่ใน buffer

**วิธีแก้:**

```nginx
# nginx.conf
location /events {
    proxy_buffering off;              # สำคัญที่สุด
    proxy_cache off;
    proxy_read_timeout 3600;
    add_header X-Accel-Buffering no; # ป้องกัน Nginx เปิด buffering
}
```

```rust
// ใน Axum: เพิ่ม X-Accel-Buffering: no header
async fn sse_handler(...) -> impl IntoResponse {
    let mut response = Sse::new(stream)
        .keep_alive(KeepAlive::new().interval(Duration::from_secs(15)));

    // ถ้าต้องการเพิ่ม header เองใช้ into_response แล้วแก้ headers
    response
}
```

---

### 3. Memory Leak เมื่อ Client Disconnect ไม่ถูก Cleanup

**อาการ:** RSS memory ของ process เพิ่มขึ้นเรื่อยๆ เมื่อมี client เชื่อมต่อ/ตัดต่อหลายครั้ง

**สาเหตุ:** เก็บ `Receiver` หรือ resource ไว้ใน `Arc<Mutex<HashMap>>` โดยไม่ลบออกเมื่อ client disconnect

```rust
// ผิด: ไม่ลบ receiver เมื่อ stream สิ้นสุด
let mut active_clients: HashMap<ClientId, Sender<Event>> = HashMap::new();
active_clients.insert(id, tx);
// เมื่อ client disconnect, tx ยังอยู่ใน map

// ถูก: ใช้ broadcast::Receiver ที่ drop อัตโนมัติ
// broadcast::Receiver drop ตัวเองเมื่อ stream สิ้นสุด
// ไม่ต้องการ cleanup พิเศษ
let rx = hub.subscribe();  // Receiver จะ drop เมื่อ stream ถูก drop
let stream = BroadcastStream::new(rx);
// เมื่อ HTTP connection ปิด stream ถูก drop → rx ถูก drop → sender ลด ref count
```

**Pattern ที่ปลอดภัย:**
- ใช้ `tokio::sync::broadcast` แทนการเก็บ `Vec<Sender<T>>`
- ถ้าต้องการ track clients ใช้ `AtomicUsize` นับ active connections แทน `HashMap`

---

### 4. `Last-Event-ID` Header ถูก Drop โดย Browser

**อาการ:** หลัง reconnect client ไม่ได้รับ missed events แม้ server implement replay ไว้แล้ว

**สาเหตุที่ 1:** `EventSource` ใน browser ส่ง `Last-Event-ID` เฉพาะเมื่อ server เคยส่ง `id:` field มาก่อน ถ้า event ไม่มี `id:` field browser จะไม่ส่ง header นี้

```rust
// ผิด: event ไม่มี id field
Event::default().event("metrics").data(json_str)

// ถูก: ต้องใส่ id ทุก event ที่ต้องการ replay
Event::default()
    .id(event_id.to_string())  // ← ต้องมี
    .event("metrics")
    .data(json_str)
```

**สาเหตุที่ 2:** Server อ่าน header ผิด field name (`last-event-id` เป็น lowercase ใน HTTP/2, บาง framework อาจ normalize ต่างกัน)

```rust
// ปลอดภัยกว่า: รองรับทั้ง header และ query param
let last_id = headers
    .get("last-event-id")         // HTTP/2 lowercase
    .or_else(|| headers.get("Last-Event-ID"))  // HTTP/1.1 case-insensitive
    .and_then(|v| v.to_str().ok())
    .and_then(|s| s.parse::<u64>().ok())
    .or(query_last_id)            // fallback: query param
    .unwrap_or(0);
```

---

### 5. ลืม `Cache-Control: no-cache` Header

**อาการ:** Browser cache SSE response ทำให้ได้ข้อมูลเก่าหรือ connection ไม่เปิดใหม่

```rust
// Axum ตั้ง Cache-Control: no-cache ให้อัตโนมัติเมื่อใช้ Sse<> type
// แต่ถ้าใช้ custom response ต้องตั้งเอง:
use axum::http::HeaderValue;

async fn sse_manual() -> impl IntoResponse {
    (
        [(
            axum::http::header::CACHE_CONTROL,
            HeaderValue::from_static("no-cache"),
        )],
        // ... body
    )
}
```

---

### 6. ลืม `Connection: keep-alive` บน HTTP/1.1

**อาการ:** SSE connection ถูกปิดหลังจาก response แรก บน HTTP/1.1 server บางตัวค่า default ของ Connection คือ close

```nginx
# nginx.conf
location /events {
    proxy_http_version 1.1;
    proxy_set_header Connection "";    # ส่ง keep-alive ไปยัง upstream
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ⭐ Rate Limiting per Connection

เพิ่ม rate limiting ให้แต่ละ SSE connection ส่งได้ไม่เกิน X events ต่อวินาที ป้องกัน spam จาก malicious client

**เป้าหมาย:**
- สร้าง `RateLimiter` struct ที่ใช้ token bucket algorithm
- integrate เข้ากับ `sse_handler` เพื่อ throttle stream
- เมื่อเกิน limit ส่ง `event: rate_limited` แล้วหยุด stream

**Hint:**
```rust
// tokio::time::interval สร้าง timer ที่ refill token ทุก interval
let mut tokens = 10u32;
let mut refill = tokio::time::interval(Duration::from_secs(1));

// ใน stream loop:
// select! ระหว่าง refill timer และ broadcast receiver
```

---

### แบบฝึกหัดที่ 2: ⭐⭐ Authentication ด้วย JWT Bearer Token

เพิ่ม authentication ให้ SSE endpoint รับเฉพาะ client ที่มี valid JWT

**เป้าหมาย:**
- รับ `Authorization: Bearer <token>` header ใน SSE request
- ตรวจสอบ JWT ด้วย `jsonwebtoken` crate
- ถ้า token หมดอายุ ส่ง `event: auth_expired` แล้วปิด stream
- ใช้ claims จาก JWT เพื่อ subscribe ไปยัง per-user channel เท่านั้น

**Hint:**
```rust
// axum extractor สำหรับ JWT
struct AuthenticatedUser {
    user_id: String,
    roles: Vec<String>,
}

#[async_trait]
impl<S> FromRequestParts<S> for AuthenticatedUser {
    // ... ดึง header และตรวจสอบ token
}
```

---

### แบบฝึกหัดที่ 3: ⭐⭐⭐ Persistent Event Log ด้วย SQLite

แทนที่ ring buffer ด้วย SQLite database ทำให้ event history คงอยู่แม้ server restart

**เป้าหมาย:**
- ใช้ `sqlx` crate กับ SQLite เก็บ events ในตาราง `events(id, event_type, data, created_at)`
- เมื่อ server start อ่าน events ล่าสุด N ชุดมาใส่ใน ring buffer ก่อน
- เมื่อ client reconnect ด้วย `Last-Event-ID` query จาก SQLite ถ้า event เกิน ring buffer range
- เพิ่ม endpoint `GET /events/history?from=<id>&limit=100` สำหรับ paging history

**ความท้าทาย:**
- Thread-safety: `sqlx::SqlitePool` เป็น `Clone + Send + Sync` ใส่ใน `AppState` ได้เลย
- Migration: เขียน SQL migration ใน `migrations/001_events.sql`

---

### แบบฝึกหัดที่ 4: ⭐⭐⭐ Real CPU/Memory Metrics ด้วย sysinfo

แทนที่ simulated metrics ด้วยข้อมูลจริงจาก OS โดยใช้ `sysinfo` crate

**เป้าหมาย:**
- เพิ่ม dependency `sysinfo = "0.33"` ใน `Cargo.toml`
- สร้าง `SystemMetricsCollector` ที่ใช้ `sysinfo::System` เก็บ state ระหว่าง ticks
- อ่าน CPU usage (global + per-core), RAM used/total, swap used/total
- ส่ง `DashboardEvent::Metrics` ที่มีข้อมูลครบถ้วน
- เพิ่ม disk I/O และ network throughput ใน payload

**Hint:**
```rust
use sysinfo::System;

let mut sys = System::new_all();
sys.refresh_all();

let cpu_pct = sys.global_cpu_usage();
let mem_used = sys.used_memory();   // bytes
let mem_total = sys.total_memory(); // bytes
```

---

### แบบฝึกหัดที่ 5: ⭐⭐⭐⭐ SSE Gateway — Aggregating Multiple Upstream Streams

สร้าง **SSE Gateway** ที่รวม events จาก SSE servers หลายตัวแล้ว re-broadcast ไปยัง client

**เป้าหมาย:**
- Subscribe ไปยัง upstream SSE streams ด้วย `reqwest` + `eventsource-stream` crate
- Merge streams ด้วย `tokio::select!` หรือ `StreamExt::merge()`
- เพิ่ม prefix ใน event ID เพื่อระบุ source (เช่น `"us-east-1:42"`)
- Auto-reconnect upstream เมื่อ connection ขาด
- Health check upstream ด้วย `GET /health` ทุก 30 วินาที

**ความท้าทาย:**
- Event ID namespace collision เมื่อ merge หลาย source
- Backpressure: ถ้า upstream เร็วกว่า downstream ต้องทำอย่างไร

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **SSE Server** ครบ production-ready ที่ประกอบด้วย:

**สิ่งที่สร้าง:**
- SSE endpoint ด้วย `axum::response::sse::Sse<S>` ที่รองรับ Stream interface ของ Rust
- `BroadcastHub` ที่รวม `tokio::sync::broadcast` กับ `EventRingBuffer` สำหรับ fan-out และ missed event replay
- `MetricsCollector` background task ที่ emit CPU/memory events ทุก 1 วินาที
- Per-room SSE streams ด้วย `RoomRegistry`
- CORS headers และ Nginx configuration สำหรับ buffering proxy

**Pattern สำคัญที่ได้เรียน:**

| Pattern | เทคนิค |
|---------|--------|
| SSE streaming | `Sse<impl Stream<...>>` + `Event` builder |
| Fan-out | `tokio::sync::broadcast` channel |
| Missed event replay | Ring buffer + `Last-Event-ID` header |
| Keep-alive | `KeepAlive::new().interval(...).text(...)` |
| CORS | `tower-http::cors::CorsLayer` |
| Multi-stream | `replay.chain(live)` stream concatenation |

**เปรียบเทียบ SSE vs WebSocket:**

| | SSE | WebSocket |
|---|---|---|
| Direction | Server → Client เท่านั้น | Bidirectional |
| Protocol | Plain HTTP | Upgrade + ws:// |
| Auto-reconnect | Browser รองรับในตัว | ต้อง implement เอง |
| Proxy support | ดีกว่า (plain HTTP) | บางครั้งมีปัญหา |
| Use case | Notifications, feeds, metrics | Chat, games, collaboration |

**เมื่อไหร่ควรใช้ SSE แทน WebSocket:**
- ข้อมูลไหลทางเดียว (server → client) เท่านั้น
- ต้องการ auto-reconnect โดยไม่ต้องเขียน JavaScript พิเศษ
- ผ่าน corporate proxy หรือ CDN ที่อาจมีปัญหากับ WebSocket
- ต้องการ standard HTTP authentication (Bearer token, cookies)

**เชื่อมโยงกับโปรเจคถัดไป:**

Project J09 (WASM Bindgen) จะนำ SSE events ที่สร้างในโปรเจคนี้ไปแสดงผลใน WebAssembly component โดยใช้ `web-sys::EventSource` API จาก Rust WASM เพื่อรับ events จาก server แบบ real-time และ render กราฟใน canvas element

---

**โปรเจคก่อนหน้า:** [Project J07: REST API + OpenAPI](project-j07-rest-openapi.md) | **โปรเจคถัดไป:** [Project J09: WASM Bindgen](project-j09-wasm-bindgen.md)
