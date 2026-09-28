# Project G10: Blue-Green Deployment Controller

> โมดูล: G — DevOps & Infrastructure | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 14 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Blue-Green Deployment Controller** — ระบบ orchestration สำหรับจัดการ deployment แบบ zero-downtime ที่ใช้กันในทีม DevOps ระดับ production ทั่วโลก แนวคิดคือการรักษา environment สองชุดพร้อมกันคือ **Blue** (version ปัจจุบัน) กับ **Green** (version ใหม่) เมื่อ deploy version ใหม่ระบบจะ deploy ลง Green ก่อน ตรวจสุขภาพ (health check) แล้วค่อยสลับ traffic ไปยัง Green ทั้งหมด — ถ้ามีปัญหาก็ roll back กลับ Blue ได้ทันที

แนวคิดนี้แตกต่างจาก rolling deployment ตรงที่ไม่มี mixed-version traffic เลย — ในทุกขณะ user จะโดนส่งไปที่ Blue ทั้งหมดหรือ Green ทั้งหมด ทำให้ rollback ทำได้ใน milliseconds แทนที่จะต้อง re-deploy ใหม่ทั้งหมด

**Use cases จริงในโลก production:**
- **Financial systems** — ระบบธนาคาร ร้านค้า e-commerce ที่ downtime มีราคาสูง
- **Mobile backends** — API ที่ต้อง serve app หลายล้าน users พร้อมกัน
- **SaaS platforms** — ระบบ multi-tenant ที่ต้องการ strict SLA
- **CI/CD pipelines** — ขั้นตอนสุดท้ายใน pipeline หลัง unit test + integration test ผ่าน
- **Kubernetes environments** — เป็น building block ของ ArgoCD, Flux ที่ทีม DevOps ใช้

**ทำไมต้องเขียนเองใน Rust แทนที่จะใช้เครื่องมือสำเร็จรูป?**

1. เข้าใจ Finite State Machine (FSM) design pattern อย่างลึกซึ้งในบริบท deployment
2. เรียนรู้ `std::hash` สำหรับ deterministic routing (sticky sessions ต้องใช้ hash ไม่ใช่ random)
3. ฝึก Rust enum ที่ซับซ้อน — `DeployStatus`, `DeployEventKind`, `ControllerState`
4. เข้าใจ audit log แบบ append-only ที่ immutable พร้อม JSON export ด้วย `serde_json`
5. ฝึก `std::time::Instant` สำหรับ cooldown enforcement ที่ monotonic

## สิ่งที่จะได้เรียนรู้

- **Finite State Machine (FSM)** — ออกแบบ state transitions ที่ valid/invalid อย่างชัดเจนด้วย Rust enum
- **Deterministic hashing** — `std::hash::DefaultHasher` สำหรับ sticky session routing ที่ reproducible
- **Weighted routing** — แปลง `(blue_pct, green_pct)` เป็น routing decisions ด้วย bucket arithmetic
- **Health gate pattern** — ตรวจ % instances ที่ healthy ก่อน promote — safety check ที่สำคัญมาก
- **Append-only data structures** — สร้าง audit log ที่ไม่มี `remove()` API เพื่อ immutability
- **`serde` + `serde_json`** — derive macros, `#[serde(skip)]`, `to_string_pretty()` สำหรับ JSON export
- **`std::time::Instant` + `Duration`** — cooldown tracking ด้วย monotonic clock ที่ไม่ drift
- **`uuid` crate** — generate UUIDs สำหรับ deployment IDs
- **`std::mem::discriminant`** — เปรียบเทียบ enum variant โดยไม่สนใจ payload

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–30**: Rust basics — ownership, borrowing, structs, enums, traits, generics, `Option`, `Result`
- **Part 31–40**: Error handling (`?`, `Box<dyn Error>`), trait objects, pattern matching ขั้นสูง
- **Part 41–50**: `std::collections`, iterators, closures, `HashMap`
- **Part 51–60**: Concurrency concepts, `std::sync`, `Arc`, `Mutex` (เข้าใจ concept แม้โปรเจคนี้ไม่ใช้ mutex หนัก)
- **Part 61–70**: `std::time`, `tokio` async runtime (เตรียมพร้อมสำหรับ Extension async health checker)
- **Part 71–80**: `serde`, `serde_json`, external crates, `Cargo.toml` dependencies
- **Part 96–110**: Production patterns — FSM design, audit log, deployment automation

## โครงสร้างโปรเจค (Project Layout)

```
blue-green/
├── src/
│   ├── lib.rs          ← core library: Slot, Deployment, DeploymentState,
│   │                     TrafficRouter, HealthGate, DeploymentLog,
│   │                     DeploymentController (FSM), RollbackEvent
│   └── main.rs         ← demo binary ใช้ controller จริง
├── Cargo.toml
└── README.md
```

โปรเจคนี้เลือก library crate (`lib.rs`) เป็น core เพราะต้องการให้ test เข้าถึง internal types ได้โดยตรง และ `main.rs` เป็น thin wrapper ที่ demo การใช้งาน pattern นี้ตรงกับ production crate ที่ real teams ทำ — เช่น `cargo-deploy` หรือ ArgoCD plugin ที่ embed library logic แยกจาก CLI

## การออกแบบ (Architecture & Design)

### Data Flow ของ Deployment

```
[Engineer triggers deploy "v2.0.0"]
          │
          ▼
DeploymentController::deploy(version, health_url, instances, health_results)
          │
          ├─ Guard: is_in_cooldown()? → ❌ Err("in cooldown")
          │
          ▼
     ┌─────────────────────────────────────────────────────┐
     │              Finite State Machine                    │
     │                                                     │
     │  Idle ──→ Deploying ──→ HealthChecking              │
     │                               │                     │
     │               pass gate ◄─────┤────► fail gate      │
     │                   │                       │         │
     │                   ▼                       ▼         │
     │              Promoting               RollingBack    │
     │                   │                       │         │
     │                   ▼                       ▼         │
     │               Active ◄────────────── Active        │
     │         (new slot promoted)     (old slot restored) │
     └─────────────────────────────────────────────────────┘
          │                          │
          ▼                          ▼
    audit log append           rollback_history append
    (Deployed,                 (RollbackEvent {
     TrafficShifted,            from_slot, to_slot,
     Promoted)                  reason, version })
```

### Component Responsibilities

| Component | หน้าที่ |
|-----------|---------|
| `Slot` | enum ง่าย ๆ `Blue`/`Green` พร้อม `opposite()` helper |
| `Deployment` | ข้อมูลของ deployment 1 ชุด: id, version, status, health_url, instances |
| `DeploymentState` | ถือ state รวม: active_slot, blue slot, green slot |
| `TrafficRouter` | routing decisions ด้วย hash-based bucket — deterministic, no randomness |
| `HealthGate` | evaluate health percentage, enforce min_healthy_pct threshold |
| `DeploymentLog` | append-only audit log พร้อม JSON export |
| `DeploymentController` | FSM orchestrator รวมทุกอย่างเข้าด้วยกัน |

### Design Decision: Deterministic Routing

`TrafficRouter::route_request(req_id)` ใช้ `DefaultHasher` แทน `rand::random()` โดยเจตนา เพราะ:

1. **Sticky sessions** — request จาก user เดิม (session ID เดิม) ต้องไปที่ slot เดิมเสมอ
2. **Reproducibility** — ใน test เราสามารถ assert ได้แน่นอนว่า input X → slot Y
3. **No side effects** — hash function ไม่มี global state ไม่ต้องใช้ mutex

ข้อเสียคือ `DefaultHasher` ไม่ cryptographically secure และ hash output อาจต่างกันระหว่าง Rust versions แต่สำหรับ routing purposes ความ consistency ภายใน process เดียวก็เพียงพอ

### Design Decision: FSM ใน `deploy()` method เดียว

แทนที่จะ expose `transition(next_state)` แบบ step-by-step เราซ่อน FSM transitions ไว้ใน `deploy()` method เดียว เหตุผลคือ:

- **Testability** — test ทดสอบ behavior ไม่ใช่ implementation details
- **Atomicity** — caller ไม่สามารถทิ้ง controller ไว้ใน intermediate state ได้
- **Type safety** — ถ้า health_results ผ่านแล้วต้อง Active เสมอ ไม่มี "forgot to promote" bug

ใน production จริงอาจต้อง async step-by-step FSM เพื่อ poll health endpoint จริง — นั่นคือหัวข้อ extension ข้อที่ 1

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างข้อมูลพื้นฐาน — `Slot`, `DeployStatus`, `Deployment`

เริ่มจาก building blocks ที่ง่ายที่สุด: enum `Slot` สำหรับ Blue/Green, `DeployStatus` สำหรับ lifecycle ของ deployment, และ `Deployment` struct ที่เก็บข้อมูลทุกอย่างของ deployment 1 ชุด

**`Cargo.toml`**

```toml
[package]
name = "blue-green"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
```

**`src/lib.rs` — ส่วนที่ 1: types พื้นฐาน**

```rust
use serde::{Deserialize, Serialize};
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};
use std::time::{Duration, Instant};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub enum Slot {
    Blue,
    Green,
}

impl Slot {
    pub fn opposite(&self) -> Slot {
        match self {
            Slot::Blue => Slot::Green,
            Slot::Green => Slot::Blue,
        }
    }
}

impl std::fmt::Display for Slot {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Slot::Blue => write!(f, "blue"),
            Slot::Green => write!(f, "green"),
        }
    }
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum DeployStatus {
    Idle,
    Deploying,
    HealthChecking,
    Promoting,
    Active,
    RollingBack,
    Failed,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Deployment {
    pub id: String,
    pub slot: Slot,
    pub version: String,
    pub status: DeployStatus,
    pub health_url: String,
    pub instances: u32,
}

impl Deployment {
    pub fn new(slot: Slot, version: &str, health_url: &str, instances: u32) -> Self {
        Deployment {
            id: uuid::Uuid::new_v4().to_string(),
            slot,
            version: version.to_string(),
            status: DeployStatus::Deploying,
            health_url: health_url.to_string(),
            instances,
        }
    }
}
```

สังเกตว่า `Slot` derive `Hash` เพราะเราจะใช้มันใน `HashMap` ในขั้นตอนถัดไป และ `Copy` เพราะ 2-variant enum ไม่มี heap allocation ทำให้ copy ถูกมากกว่าการ borrow

**Output จาก test เบื้องต้น:**

```
$ cargo test test_slot
test tests::test_slot_opposite ... ok
test tests::test_slot_display ... ok
```

### ขั้นที่ 2: `DeploymentState` — ถือสถานะรวมของ Cluster

`DeploymentState` เป็น "single source of truth" ของ cluster เก็บว่า slot ไหน active และข้อมูล deployment ของแต่ละ slot เป็นอะไร

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DeploymentState {
    pub active_slot: Slot,
    pub blue: Option<Deployment>,
    pub green: Option<Deployment>,
}

impl DeploymentState {
    pub fn new(initial_slot: Slot) -> Self {
        DeploymentState {
            active_slot: initial_slot,
            blue: None,
            green: None,
        }
    }

    pub fn active_deployment(&self) -> Option<&Deployment> {
        match self.active_slot {
            Slot::Blue => self.blue.as_ref(),
            Slot::Green => self.green.as_ref(),
        }
    }

    pub fn inactive_deployment(&self) -> Option<&Deployment> {
        match self.active_slot.opposite() {
            Slot::Blue => self.blue.as_ref(),
            Slot::Green => self.green.as_ref(),
        }
    }

    pub fn set_slot(&mut self, slot: Slot, dep: Deployment) {
        match slot {
            Slot::Blue => self.blue = Some(dep),
            Slot::Green => self.green = Some(dep),
        }
    }

    pub fn get_slot_mut(&mut self, slot: Slot) -> &mut Option<Deployment> {
        match slot {
            Slot::Blue => &mut self.blue,
            Slot::Green => &mut self.green,
        }
    }
}
```

`get_slot_mut()` คืน `&mut Option<Deployment>` ไม่ใช่ `Option<&mut Deployment>` เพราะเราต้องการความสามารถในการ set ค่าใหม่ทั้งหมด (ทั้ง `None → Some(...)`) ได้ด้วย

ความแตกต่างสำคัญ:
- `&mut Option<T>` — เราสามารถ set field เป็น `None` หรือ `Some(value)` ได้
- `Option<&mut T>` — เราแก้ไขข้างในได้เท่านั้น ถ้าปัจจุบันเป็น `None` ทำอะไรไม่ได้

### ขั้นที่ 3: `TrafficRouter` — Hash-Based Weighted Routing

`TrafficRouter` ตัดสินใจว่า request นี้ควรไปที่ Blue หรือ Green โดยใช้ hash ของ request ID ทำให้ผลลัพธ์ deterministic — request ID เดิมไปที่ slot เดิมเสมอ (sticky sessions)

```rust
#[derive(Debug, Clone)]
pub struct TrafficRouter {
    /// (blue_pct, green_pct) — ต้องรวมกันเป็น 100
    pub weights: (u8, u8),
    pub canary_mode: bool,
}

impl TrafficRouter {
    pub fn new(blue_pct: u8, green_pct: u8) -> Result<Self, String> {
        if blue_pct as u16 + green_pct as u16 != 100 {
            return Err(format!(
                "weights must sum to 100, got {}+{}={}",
                blue_pct, green_pct,
                blue_pct as u16 + green_pct as u16
            ));
        }
        Ok(TrafficRouter {
            weights: (blue_pct, green_pct),
            canary_mode: false,
        })
    }

    /// Canary mode: 95% Blue, 5% Green
    pub fn canary() -> Self {
        TrafficRouter {
            weights: (95, 5),
            canary_mode: true,
        }
    }

    /// Deterministic routing: hash(req_id) % 100 → slot
    /// req_id เดิมจะได้ slot เดิมเสมอ (sticky session guarantee)
    pub fn route_request(&self, req_id: &str) -> Slot {
        let mut hasher = DefaultHasher::new();
        req_id.hash(&mut hasher);
        let hash = hasher.finish();
        let bucket = (hash % 100) as u8;
        if bucket < self.weights.0 {
            Slot::Blue
        } else {
            Slot::Green
        }
    }

    /// คำนวณ fraction ของ requests ที่ route ไป Green
    /// ใช้ตรวจสอบว่า canary routing ทำงานใกล้เคียง target percentage
    pub fn canary_green_fraction(&self, sample_size: usize) -> f64 {
        let green_count = (0..sample_size)
            .filter(|i| self.route_request(&i.to_string()) == Slot::Green)
            .count();
        green_count as f64 / sample_size as f64
    }

    pub fn set_weights(&mut self, blue_pct: u8, green_pct: u8) -> Result<(), String> {
        if blue_pct as u16 + green_pct as u16 != 100 {
            return Err("weights must sum to 100".to_string());
        }
        self.weights = (blue_pct, green_pct);
        Ok(())
    }
}
```

**หลักการของ bucket routing:**

```
hash("req-001") = 0x7f3a...   → % 100 = 63
weights = (70, 30)            → bucket 63 < 70 → Blue

hash("req-002") = 0x2b1c...   → % 100 = 78
weights = (70, 30)            → bucket 78 >= 70 → Green

hash("req-001") อีกครั้ง      → bucket 63 → Blue (เหมือนเดิมเสมอ)
```

ข้อควรระวัง: `DefaultHasher` ไม่ได้รับประกัน output เหมือนกันระหว่าง Rust version ต่าง ๆ ใน production ควรใช้ hash ที่ stable เช่น `fnv`, `ahash` พร้อม seed ที่กำหนดไว้

### ขั้นที่ 4: `HealthGate` — Safety Gate ก่อน Promote

`HealthGate` เป็น guardian ที่บล็อกการ promote ถ้า deployment ยังไม่ healthy พอ โดยกำหนด threshold เป็น % ของ instances ที่ต้องตอบ health check ผ่าน

```rust
#[derive(Debug, Clone)]
pub struct HealthGate {
    pub min_healthy_pct: f64,        // เช่น 80.0 = ต้องการ 80% healthy
    pub check_interval: Duration,    // poll health endpoint ทุก N วินาที
    pub warmup_period: Duration,     // รอ warmup ก่อน health check แรก
}

impl HealthGate {
    pub fn new(min_healthy_pct: f64, check_interval_secs: u64, warmup_secs: u64) -> Self {
        HealthGate {
            min_healthy_pct,
            check_interval: Duration::from_secs(check_interval_secs),
            warmup_period: Duration::from_secs(warmup_secs),
        }
    }

    /// ประเมิน health results: คืน Ok(()) ถ้าผ่าน gate, Err(reason) ถ้าไม่ผ่าน
    pub fn evaluate(&self, healthy: u32, total: u32) -> Result<(), String> {
        if total == 0 {
            return Err("no instances to check".to_string());
        }
        let pct = healthy as f64 / total as f64 * 100.0;
        if pct >= self.min_healthy_pct {
            Ok(())
        } else {
            Err(format!(
                "health gate failed: {:.1}% healthy, need {:.1}%",
                pct, self.min_healthy_pct
            ))
        }
    }
}
```

ตัวอย่างการใช้งาน `HealthGate` ในชีวิตจริง:
- `min_healthy_pct = 100.0` — strict mode ต้อง 100% ก่อน promote (เหมาะกับ stateful services)
- `min_healthy_pct = 80.0` — standard mode ยอมให้ 2/10 instances ยัง warm up อยู่
- `min_healthy_pct = 50.0` — lenient mode สำหรับ dev/staging environments

### ขั้นที่ 5: Audit Log — Append-Only Event Store

`DeploymentLog` เก็บ history ของทุก event ที่เกิดขึ้นในระบบ deployment เป็น append-only โดยไม่มี `remove()`, `pop()`, หรือ `clear()` API — ทำให้ audit trail ไม่สามารถถูกลบทิ้งได้

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum DeployEventKind {
    Deployed,
    Promoted,
    RolledBack,
    HealthFailed,
    TrafficShifted,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DeployEvent {
    pub kind: DeployEventKind,
    pub slot: Slot,
    pub version: String,
    pub detail: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DeploymentLog {
    pub events: Vec<DeployEvent>,
}

impl DeploymentLog {
    pub fn new() -> Self {
        DeploymentLog { events: Vec::new() }
    }

    /// Append event — เพิ่มได้อย่างเดียว ลบไม่ได้ (append-only guarantee)
    pub fn append(&mut self, event: DeployEvent) {
        self.events.push(event);
    }

    /// Export ทุก event เป็น JSON string
    pub fn export_json(&self) -> Result<String, serde_json::Error> {
        serde_json::to_string_pretty(&self.events)
    }

    /// นับ event ตาม kind โดยใช้ std::mem::discriminant
    /// เปรียบเทียบ variant เท่านั้น ไม่สนใจ payload
    pub fn count_by_kind(&self, kind: &DeployEventKind) -> usize {
        self.events
            .iter()
            .filter(|e| {
                std::mem::discriminant(&e.kind) == std::mem::discriminant(kind)
            })
            .count()
    }
}

impl Default for DeploymentLog {
    fn default() -> Self {
        Self::new()
    }
}
```

`std::mem::discriminant` เป็น trick สำคัญ — ใช้เปรียบเทียบว่า enum variant เหมือนกันหรือไม่ โดยไม่ต้อง `match` กับ payload:

```rust
// ถ้าใช้ pattern matching ธรรมดา:
matches!(&e.kind, DeployEventKind::HealthFailed)

// หรือใช้ discriminant สำหรับ generic function:
std::mem::discriminant(&e.kind) == std::mem::discriminant(&DeployEventKind::HealthFailed)
```

**ตัวอย่าง JSON output จาก `export_json()`:**

```json
[
  {
    "kind": "Deployed",
    "slot": "Green",
    "version": "v1.0.0",
    "detail": "Deploying 3 instances to green"
  },
  {
    "kind": "TrafficShifted",
    "slot": "Green",
    "version": "v1.0.0",
    "detail": "Shifting traffic to green"
  },
  {
    "kind": "Promoted",
    "slot": "Green",
    "version": "v1.0.0",
    "detail": "Promoted green to active"
  }
]
```

### ขั้นที่ 6: `RollbackEvent` และ Rollback History

เมื่อ deployment ล้มเหลว ระบบต้อง track ว่า rollback เกิดขึ้นเพราะอะไร เพื่อใช้ debug และ improve deployment process

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RollbackEvent {
    pub from_slot: Slot,       // slot ที่พยายาม promote
    pub to_slot: Slot,         // slot ที่ roll back ไป
    pub reason: String,        // ข้อความอธิบายสาเหตุ
    pub version: String,       // version ที่ล้มเหลว
    #[serde(skip)]
    pub timestamp: Option<Instant>,  // เวลาที่เกิด (ไม่ serialize เพราะ Instant ไม่ impl Serialize)
}
```

สังเกต `#[serde(skip)]` บน field `timestamp` เพราะ `std::time::Instant` ไม่ implement `Serialize`/`Deserialize` (มันขึ้นอยู่กับ OS-specific monotonic clock) ถ้าต้องการ serialize เวลาจริง ๆ ให้ใช้ `chrono::DateTime<Utc>` หรือ `std::time::SystemTime` แทน

### ขั้นที่ 7: `DeploymentController` — FSM Orchestrator

นี่คือหัวใจของระบบ `DeploymentController` รวม FSM, health gate, routing, และ audit log เข้าด้วยกัน method `deploy()` ขับเคลื่อน state transitions ทั้งหมด

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ControllerState {
    Idle,
    Deploying,
    HealthChecking,
    Promoting,
    Active,
    RollingBack,
}

pub struct DeploymentController {
    pub state: ControllerState,
    pub deployment_state: DeploymentState,
    pub router: TrafficRouter,
    pub health_gate: HealthGate,
    pub log: DeploymentLog,
    pub rollback_history: Vec<RollbackEvent>,
    last_deploy_time: Option<Instant>,   // private: ใช้เฉพาะ cooldown check
    pub cooldown: Duration,
}
```

**`deploy()` method — FSM driver:**

```rust
impl DeploymentController {
    pub fn new(initial_slot: Slot, cooldown_secs: u64) -> Self {
        DeploymentController {
            state: ControllerState::Idle,
            deployment_state: DeploymentState::new(initial_slot),
            router: TrafficRouter::new(100, 0).unwrap(),
            health_gate: HealthGate::new(80.0, 10, 30),
            log: DeploymentLog::new(),
            rollback_history: Vec::new(),
            last_deploy_time: None,
            cooldown: Duration::from_secs(cooldown_secs),
        }
    }

    pub fn cooldown_remaining(&self) -> Option<Duration> {
        self.last_deploy_time.map(|t| {
            let elapsed = t.elapsed();
            if elapsed < self.cooldown {
                self.cooldown - elapsed
            } else {
                Duration::ZERO
            }
        })
    }

    pub fn is_in_cooldown(&self) -> bool {
        self.cooldown_remaining()
            .map(|d| d > Duration::ZERO)
            .unwrap_or(false)
    }

    /// Drive FSM: Idle/Active → Deploying → HealthChecking → Promoting/RollingBack → Active
    /// health_results: (healthy_instances, total_instances)
    pub fn deploy(
        &mut self,
        version: &str,
        health_url: &str,
        instances: u32,
        health_results: (u32, u32),
    ) -> Result<Slot, String> {
        // Guard: cooldown period
        if self.is_in_cooldown() {
            return Err(format!(
                "in cooldown period, {:?} remaining",
                self.cooldown_remaining().unwrap()
            ));
        }

        let target_slot = self.deployment_state.active_slot.opposite();

        // Guard: valid start state
        if self.state != ControllerState::Idle && self.state != ControllerState::Active {
            return Err(format!("cannot deploy from state {:?}", self.state));
        }

        // ── Idle/Active → Deploying ──────────────────────────────────────────
        self.state = ControllerState::Deploying;
        let dep = Deployment::new(target_slot, version, health_url, instances);
        self.deployment_state.set_slot(target_slot, dep);
        self.log.append(DeployEvent {
            kind: DeployEventKind::Deployed,
            slot: target_slot,
            version: version.to_string(),
            detail: format!("Deploying {} instances to {}", instances, target_slot),
        });

        // ── Deploying → HealthChecking ────────────────────────────────────────
        self.state = ControllerState::HealthChecking;
        if let Some(d) = self.deployment_state.get_slot_mut(target_slot).as_mut() {
            d.status = DeployStatus::HealthChecking;
        }

        // ── HealthChecking → Promoting (pass) หรือ RollingBack (fail) ─────────
        match self.health_gate.evaluate(health_results.0, health_results.1) {
            Ok(()) => {
                // Health pass: Promoting → Active
                self.state = ControllerState::Promoting;
                self.log.append(DeployEvent {
                    kind: DeployEventKind::TrafficShifted,
                    slot: target_slot,
                    version: version.to_string(),
                    detail: format!("Shifting traffic to {}", target_slot),
                });
                self.deployment_state.active_slot = target_slot;
                self.state = ControllerState::Active;
                if let Some(d) = self.deployment_state.get_slot_mut(target_slot).as_mut() {
                    d.status = DeployStatus::Active;
                }
                self.log.append(DeployEvent {
                    kind: DeployEventKind::Promoted,
                    slot: target_slot,
                    version: version.to_string(),
                    detail: format!("Promoted {} to active", target_slot),
                });
                self.last_deploy_time = Some(Instant::now());
                Ok(target_slot)
            }
            Err(reason) => {
                // Health fail: RollingBack → Active (บน old slot)
                self.state = ControllerState::RollingBack;
                self.log.append(DeployEvent {
                    kind: DeployEventKind::HealthFailed,
                    slot: target_slot,
                    version: version.to_string(),
                    detail: reason.clone(),
                });
                let rollback_target = target_slot.opposite();
                self.rollback_history.push(RollbackEvent {
                    from_slot: target_slot,
                    to_slot: rollback_target,
                    reason: reason.clone(),
                    version: version.to_string(),
                    timestamp: Some(Instant::now()),
                });
                self.deployment_state.active_slot = rollback_target;
                self.log.append(DeployEvent {
                    kind: DeployEventKind::RolledBack,
                    slot: rollback_target,
                    version: version.to_string(),
                    detail: format!(
                        "Rolled back from {} to {}",
                        target_slot, rollback_target
                    ),
                });
                self.state = ControllerState::Active;
                self.last_deploy_time = Some(Instant::now());
                Err(format!("health check failed, rolled back: {}", reason))
            }
        }
    }

    /// Manual rollback: Active → RollingBack → Active (บน opposite slot)
    pub fn manual_rollback(&mut self, reason: &str) -> Result<Slot, String> {
        if self.state != ControllerState::Active {
            return Err(format!("cannot rollback from state {:?}", self.state));
        }
        let current = self.deployment_state.active_slot;
        let target = current.opposite();
        self.state = ControllerState::RollingBack;
        self.rollback_history.push(RollbackEvent {
            from_slot: current,
            to_slot: target,
            reason: reason.to_string(),
            version: self
                .deployment_state
                .active_deployment()
                .map(|d| d.version.clone())
                .unwrap_or_default(),
            timestamp: Some(Instant::now()),
        });
        self.deployment_state.active_slot = target;
        self.log.append(DeployEvent {
            kind: DeployEventKind::RolledBack,
            slot: target,
            version: String::from("manual"),
            detail: format!("Manual rollback: {}", reason),
        });
        self.state = ControllerState::Active;
        Ok(target)
    }
}
```

### ขั้นที่ 8: `main.rs` — Demo Binary

```rust
use blue_green::{DeploymentController, Slot, TrafficRouter};

fn main() {
    println!("=== Blue-Green Deployment Controller Demo ===\n");

    // สร้าง controller เริ่มต้นด้วย Blue เป็น active
    let mut ctrl = DeploymentController::new(Slot::Blue, 0);
    println!("Initial active slot: {}", ctrl.deployment_state.active_slot);

    // Deploy v1.0.0 ไปที่ Green (health: 10/10 = 100%)
    match ctrl.deploy("v1.0.0", "http://green:8080/health", 3, (10, 10)) {
        Ok(slot) => println!("Deploy succeeded: active slot is now {}", slot),
        Err(e) => println!("Deploy failed: {}", e),
    }

    // Demo canary routing
    let router = TrafficRouter::canary();
    println!("\nCanary routing (5% green):");
    let fraction = router.canary_green_fraction(1000);
    println!("  Green fraction over 1000 requests: {:.1}%", fraction * 100.0);

    // แสดง audit log
    println!("\nAudit log ({} events):", ctrl.log.events.len());
    for event in &ctrl.log.events {
        println!("  [{:?}] slot={} ver={}", event.kind, event.slot, event.version);
    }

    println!("\nDone.");
}
```

**Output จากการรัน `cargo run`:**

```
=== Blue-Green Deployment Controller Demo ===

Initial active slot: blue
Deploy succeeded: active slot is now green

Canary routing (5% green):
  Green fraction over 1000 requests: 5.1%

Audit log (3 events):
  [Deployed] slot=green ver=v1.0.0
  [TrafficShifted] slot=green ver=v1.0.0
  [Promoted] slot=green ver=v1.0.0

Done.
```

### ขั้นที่ 9: Scenario ที่ซับซ้อน — Health Failure และ Consecutive Deploys

ทดสอบ scenario จริงที่เกิดขึ้นใน production:

**Scenario 1: Health failure → automatic rollback**

```rust
fn demo_health_failure() {
    let mut ctrl = DeploymentController::new(Slot::Blue, 0);

    // Deploy v2.0.0 (buggy) ไป Green — health ล้มเหลว (5/10 = 50%)
    println!("Deploying buggy v2.0.0...");
    match ctrl.deploy("v2.0.0", "http://green/health", 10, (5, 10)) {
        Ok(_) => println!("Deployed (unexpected)"),
        Err(e) => println!("Deploy failed (expected): {}", e),
    }

    println!(
        "Active slot after failure: {}",
        ctrl.deployment_state.active_slot
    );
    println!("Rollback history: {} events", ctrl.rollback_history.len());
    println!(
        "Rollback reason: {}",
        ctrl.rollback_history[0].reason
    );
}
```

```
Deploying buggy v2.0.0...
Deploy failed (expected): health check failed, rolled back: health gate failed: 50.0% healthy, need 80.0%
Active slot after failure: blue
Rollback history: 1 events
Rollback reason: health gate failed: 50.0% healthy, need 80.0%
```

**Scenario 2: Consecutive deploys สลับ slots**

```rust
fn demo_consecutive() {
    let mut ctrl = DeploymentController::new(Slot::Blue, 0);

    // Deploy v1 → Green
    ctrl.deploy("v1.0.0", "http://green/health", 3, (10, 10)).unwrap();
    println!("After v1: active = {}", ctrl.deployment_state.active_slot);

    // Deploy v2 → Blue (เพราะตอนนี้ Green เป็น active, ดังนั้น inactive = Blue)
    ctrl.deploy("v2.0.0", "http://blue/health", 3, (10, 10)).unwrap();
    println!("After v2: active = {}", ctrl.deployment_state.active_slot);

    // Deploy v3 → Green
    ctrl.deploy("v3.0.0", "http://green/health", 3, (10, 10)).unwrap();
    println!("After v3: active = {}", ctrl.deployment_state.active_slot);
}
```

```
After v1: active = green
After v2: active = blue
After v3: active = green
```

Slots สลับกันทุก deploy เหมือน ping-pong — นี่คือ property ที่ถูกต้องของ blue-green deployment

### ขั้นที่ 10: Cooldown Enforcement และ Manual Rollback

**Cooldown** ป้องกันการ deploy ถี่เกินไปหลังจาก deploy ครั้งล่าสุด ไม่ว่าจะสำเร็จหรือล้มเหลว

```rust
fn demo_cooldown() {
    // cooldown = 3600 วินาที (1 ชั่วโมง)
    let mut ctrl = DeploymentController::new(Slot::Blue, 3600);

    // deploy แรกสำเร็จ
    ctrl.deploy("v1.0.0", "http://green/health", 2, (10, 10)).unwrap();

    // deploy ที่สองล้มเหลวเพราะ cooldown
    let result = ctrl.deploy("v1.1.0", "http://blue/health", 2, (10, 10));
    println!("Second deploy result: {:?}", result.is_err()); // true

    let remaining = ctrl.cooldown_remaining().unwrap();
    println!("Cooldown remaining: ~{} secs", remaining.as_secs());
}
```

**Manual rollback** ใช้เมื่อ engineer ต้องการ rollback ทันทีโดยไม่รอ health gate:

```rust
fn demo_manual_rollback() {
    let mut ctrl = DeploymentController::new(Slot::Blue, 0);
    ctrl.deploy("v1.0.0", "http://green/health", 3, (10, 10)).unwrap();

    println!("Before rollback: {}", ctrl.deployment_state.active_slot); // green

    // Engineer สังเกต bug ใน production → manual rollback
    let result = ctrl.manual_rollback("critical regression in payment processing");
    println!("Rolled back to: {:?}", result.unwrap()); // Blue
    println!("After rollback: {}", ctrl.deployment_state.active_slot); // blue
    println!("Rollback reason: {}", ctrl.rollback_history[0].reason);
}
```

```
Before rollback: green
Rolled back to: Blue
After rollback: blue
Rollback reason: critical regression in payment processing
```

## การทดสอบ (Testing)

### Test Suite ครบถ้วน

โปรเจคมี 24 unit tests ครอบคลุม:

1. **Slot tests** — `opposite()`, `Display`
2. **TrafficRouter tests** — weight validation, all-Blue, all-Green, sticky sessions, distribution, canary percentage
3. **HealthGate tests** — pass at threshold, fail below threshold, zero instances
4. **FSM tests** — successful deploy, health failure rollback, state sequence, consecutive deploys alternating slots
5. **Cooldown test** — second deploy rejected during cooldown period
6. **Manual rollback test** — rollback history populated with correct reason
7. **Audit log tests** — append, immutable order, JSON export, event sequence from deploy, HealthFailed/RolledBack on failure

### ไฟล์ทดสอบ (ใน `src/lib.rs`)

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_slot_opposite() {
        assert_eq!(Slot::Blue.opposite(), Slot::Green);
        assert_eq!(Slot::Green.opposite(), Slot::Blue);
    }

    #[test]
    fn test_router_weights_must_sum_to_100() {
        assert!(TrafficRouter::new(50, 60).is_err());
        assert!(TrafficRouter::new(100, 0).is_ok());
        assert!(TrafficRouter::new(0, 100).is_ok());
        assert!(TrafficRouter::new(50, 50).is_ok());
    }

    #[test]
    fn test_sticky_session_deterministic() {
        let router = TrafficRouter::new(50, 50).unwrap();
        for _ in 0..10 {
            let slot1 = router.route_request("session-abc-123");
            let slot2 = router.route_request("session-abc-123");
            assert_eq!(slot1, slot2);
        }
    }

    #[test]
    fn test_canary_router_approximately_5_percent_green() {
        let router = TrafficRouter::canary();
        assert_eq!(router.weights, (95, 5));
        let frac = router.canary_green_fraction(10000);
        assert!(
            frac >= 0.03 && frac <= 0.08,
            "canary green fraction {:.3} out of range 3-8%",
            frac
        );
    }

    #[test]
    fn test_fsm_successful_deploy() {
        let mut ctrl = DeploymentController::new(Slot::Blue, 0);
        let result = ctrl.deploy("v1.2.0", "http://green:8080/health", 3, (9, 10));
        assert!(result.is_ok());
        assert_eq!(result.unwrap(), Slot::Green);
        assert_eq!(ctrl.deployment_state.active_slot, Slot::Green);
        assert_eq!(ctrl.state, ControllerState::Active);
    }

    #[test]
    fn test_fsm_health_failure_triggers_rollback() {
        let mut ctrl = DeploymentController::new(Slot::Blue, 0);
        let result = ctrl.deploy("v1.2.0", "http://green:8080/health", 3, (5, 10));
        assert!(result.is_err());
        assert_eq!(ctrl.deployment_state.active_slot, Slot::Blue);
        assert_eq!(ctrl.rollback_history.len(), 1);
        assert_eq!(ctrl.state, ControllerState::Active);
    }

    #[test]
    fn test_cooldown_prevents_deploy() {
        let mut ctrl = DeploymentController::new(Slot::Blue, 3600);
        ctrl.deploy("v1.0.0", "http://green/health", 2, (10, 10)).unwrap();
        let result = ctrl.deploy("v1.1.0", "http://blue/health", 2, (10, 10));
        assert!(result.is_err());
        assert!(result.unwrap_err().contains("cooldown"));
    }

    // ... (tests ทั้งหมดอยู่ใน src/lib.rs)
}
```

### Real `cargo test` Output

ต่อไปนี้คือ output จริงจากการรัน `cargo test` บน scratchpad project:

```
$ cargo test
   Compiling blue-green v0.1.0 (/tmp/blue-green)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.76s
     Running unittests src/lib.rs (target/debug/deps/blue_green-bbfa72d594e616f3)

running 24 tests
test tests::test_audit_log_appends_events ... ok
test tests::test_audit_log_immutable_append_only ... ok
test tests::test_audit_log_json_export ... ok
test tests::test_consecutive_deploy_alternates_slots ... ok
test tests::test_cooldown_prevents_deploy ... ok
test tests::test_deploy_logs_correct_event_sequence ... ok
test tests::test_deployment_state_active_inactive ... ok
test tests::test_different_req_ids_different_buckets ... ok
test tests::test_fsm_health_failure_triggers_rollback ... ok
test tests::test_fsm_state_transitions_in_order ... ok
test tests::test_canary_router_approximately_5_percent_green ... ok
test tests::test_fsm_successful_deploy ... ok
test tests::test_health_gate_passes ... ok
test tests::test_health_failure_logs_health_failed_and_rolled_back ... ok
test tests::test_health_gate_fails ... ok
test tests::test_health_gate_zero_instances ... ok
test tests::test_manual_rollback ... ok
test tests::test_router_weight_update ... ok
test tests::test_router_all_blue ... ok
test tests::test_router_all_green ... ok
test tests::test_slot_display ... ok
test tests::test_sticky_session_deterministic ... ok
test tests::test_router_weights_must_sum_to_100 ... ok
test tests::test_slot_opposite ... ok

test result: ok. 24 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/blue_green-335ce23d0173d7bf)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests blue_green

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**ทุก test ผ่านทั้งหมด 24/24** ครอบคลุมทุก component ในระบบ

### Test Coverage Analysis

| กลุ่ม test | จำนวน | สิ่งที่ตรวจสอบ |
|------------|-------|---------------|
| Slot | 2 | `opposite()`, `Display` trait |
| TrafficRouter | 6 | weight validation, all-Blue, all-Green, sticky session, distribution, canary % |
| HealthGate | 3 | pass/fail/zero-instances |
| FSM / Controller | 6 | successful deploy, health failure rollback, state transitions, consecutive slots, cooldown, manual rollback |
| DeploymentLog | 5 | append, immutable order, JSON export, event sequence, health failure events |
| DeploymentState | 1 | active/inactive accessors |
| TrafficRouter update | 1 | set_weights validation |

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary จะอยู่ที่
./target/release/blue-green

# ขนาด binary โดยประมาณ (พร้อม strip)
strip target/release/blue-green
ls -lh target/release/blue-green
# -rwxr-xr-x 1 user user 1.2M blue-green
```

### Docker Image

```dockerfile
# ── Builder stage ─────────────────────────────────────────────────────────────
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release && strip target/release/blue-green

# ── Runtime stage ─────────────────────────────────────────────────────────────
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/blue-green /usr/local/bin/
EXPOSE 8080
CMD ["blue-green"]
```

```bash
docker build -t blue-green:latest .
docker run --rm blue-green:latest
```

### Integration กับ CI/CD Pipeline

ใน production ระบบนี้จะถูก invoke จาก CI/CD pipeline เช่น GitHub Actions:

```yaml
# .github/workflows/deploy.yml
name: Blue-Green Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build controller
        run: cargo build --release
      - name: Run tests
        run: cargo test
      - name: Deploy new version
        run: |
          ./target/release/blue-green deploy \
            --version ${{ github.sha }} \
            --instances 3 \
            --health-url http://green.internal/health
```

### Environment Variables สำหรับ Configuration

```bash
export BG_COOLDOWN_SECS=300        # cooldown 5 นาที
export BG_MIN_HEALTHY_PCT=80.0     # ต้องการ 80% healthy
export BG_WARMUP_SECS=30           # รอ warmup 30 วินาที
export BG_CHECK_INTERVAL_SECS=10   # poll health ทุก 10 วินาที
```

## จุดอันตราย (Pitfalls)

### Pitfall 1: DefaultHasher ไม่ Stable ข้าม Process Restarts

`std::collections::hash_map::DefaultHasher` ใช้ random seed ตั้งแต่ Rust 1.36 เพื่อป้องกัน HashDoS attacks ดังนั้น hash output ของ string เดิมจะ**ต่างกัน** ระหว่าง process restarts

```rust
// ❌ ปัญหา: bucket ของ "user-123" เปลี่ยนทุก restart
// ถ้า user เดิม connect ใหม่หลัง restart อาจไปคนละ slot
let bucket = hash("user-123") % 100;  // process 1: 42 / process 2: 67
```

**วิธีแก้:** ใช้ hash function ที่ stable เช่น `fnv`, `sha256` หรือ consistent hash ring:

```rust
// ✅ ใช้ FNV hash ที่ deterministic ทุก run
use fnv::FnvHasher;
let mut hasher = FnvHasher::default();
req_id.hash(&mut hasher);
```

หรือถ้า sticky session ข้าม restart สำคัญมาก ให้ store routing decision ใน Redis/database

### Pitfall 2: `Option<Instant>` ใน Serialized Struct

`std::time::Instant` ไม่ implement `Serialize` เพราะมันขึ้นอยู่กับ OS monotonic clock และไม่มีนิยามที่ตายตัวว่าจะ serialize อย่างไร ถ้าใส่ใน struct ที่ derive `Serialize` โดยไม่มี `#[serde(skip)]` จะ compile error

```rust
// ❌ Compile error!
#[derive(Serialize)]
struct Event {
    timestamp: Instant,  // error: Instant doesn't implement Serialize
}

// ✅ Skip field
#[derive(Serialize)]
struct Event {
    #[serde(skip)]
    timestamp: Option<Instant>,
}

// ✅ หรือใช้ SystemTime (serialize ได้กับ serde feature)
#[derive(Serialize)]
struct Event {
    timestamp: std::time::SystemTime,
}
```

### Pitfall 3: `u8` Overflow ใน Weight Validation

เมื่อตรวจสอบว่า `blue_pct + green_pct == 100` ถ้าใช้ `u8` โดยตรง:

```rust
// ❌ Overflow! u8::MAX = 255, 200 + 100 = 44 (overflow, ไม่ใช่ 300)
if blue_pct + green_pct != 100 { ... }  // blue=200, green=100: 200u8+100u8 = 44 (wraps)
```

**วิธีแก้:** cast เป็น `u16` ก่อนบวก:

```rust
// ✅ ใช้ u16 ป้องกัน overflow
if blue_pct as u16 + green_pct as u16 != 100 {
    return Err("weights must sum to 100".to_string());
}
```

หรือใช้ `checked_add()`:

```rust
if blue_pct.checked_add(green_pct) != Some(100) { ... }
```

### Pitfall 4: Cooldown หลัง Rollback ด้วย

ในโปรเจคนี้ `last_deploy_time` ถูก set ทั้งใน success path และ rollback path หมายความว่า **หลัง health failure + rollback ก็ยังมี cooldown** — นี่เป็น intentional design decision เพราะ:

1. Rollback ก็เป็น "deployment event" ที่ต้องให้ระบบ stabilize
2. ป้องกัน retry loop ถ้า underlying problem ยังไม่ได้แก้

แต่ถ้า business requirement ต้องการให้ rollback ไม่มี cooldown ต้องเปลี่ยน:

```rust
// ✅ Set cooldown เฉพาะ success path
Err(reason) => {
    // ... rollback logic ...
    // ลบ self.last_deploy_time = Some(Instant::now()); ออกจาก path นี้
    Err(...)
}
```

### Pitfall 5: `get_slot_mut` คืน `&mut Option<T>` ไม่ใช่ `Option<&mut T>`

```rust
// ❌ Type mismatch: get_slot_mut คืน &mut Option<Deployment>
// ไม่ใช่ Option<&mut Deployment>
if let Some(dep) = self.deployment_state.get_slot_mut(target_slot) {
    // error: expected Deployment, found Option<_>
    dep.status = DeployStatus::Active;
}

// ✅ ใช้ .as_mut() เพื่อแปลง &mut Option<T> → Option<&mut T>
if let Some(d) = self.deployment_state.get_slot_mut(target_slot).as_mut() {
    d.status = DeployStatus::Active;
}
```

นี่คือ idiom สำคัญ: `(&mut Option<T>).as_mut()` → `Option<&mut T>` ทำให้ pattern match ได้

### Pitfall 6: `std::mem::discriminant` vs Pattern Matching

เมื่อต้องการ filter events ตาม `DeployEventKind` สิ่งที่ผิดพลาดบ่อยคือพยายาม `==` กับ enum variant ที่ไม่ implement `PartialEq` บน payload

```rust
// ❌ ถ้า DeployEventKind มี payload เช่น Deployed { instances: u32 }
// จะไม่สามารถ == กันได้ถ้า payload ต่างกัน
e.kind == DeployEventKind::Deployed  // compile error ถ้าไม่ derive PartialEq

// ✅ ใช้ discriminant เพื่อ compare variant เท่านั้น ไม่สนใจ payload
std::mem::discriminant(&e.kind) == std::mem::discriminant(&DeployEventKind::Deployed)

// ✅ หรือใช้ matches! macro (ง่ายกว่า)
matches!(&e.kind, DeployEventKind::Deployed)
```

## การต่อยอด (Extensions & Exercises)

### Exercise 1: Async Health Checker (ระดับ Medium)

ในระบบจริง `HealthGate::evaluate()` ต้องส่ง HTTP request ไปยัง health endpoint จริง ๆ ไม่ใช่รับ `(healthy, total)` จากภายนอก ให้เพิ่ม method async:

```rust
use tokio::time::timeout;

impl HealthGate {
    pub async fn check_real(
        &self,
        health_url: &str,
        instances: u32,
    ) -> Result<(), String> {
        let client = reqwest::Client::new();
        let mut healthy = 0u32;

        for i in 0..instances {
            let url = format!("{}/instance/{}", health_url, i);
            let result = timeout(
                self.check_interval,
                client.get(&url).send()
            ).await;

            match result {
                Ok(Ok(resp)) if resp.status().is_success() => healthy += 1,
                _ => {} // unhealthy or timeout
            }
        }

        self.evaluate(healthy, instances)
    }
}
```

ต้องเพิ่ม `reqwest = { version = "0.11", features = ["json"] }` ใน `Cargo.toml` และเปลี่ยน `deploy()` เป็น `async fn`

### Exercise 2: Traffic Weight Progression — Canary → Full Rollout (ระดับ Medium)

แทนที่จะ switch 0% → 100% ทันที ให้สร้าง gradient rollout ที่ค่อย ๆ เพิ่ม Green traffic:

```rust
pub struct GradualRollout {
    controller: DeploymentController,
    steps: Vec<u8>,     // เช่น [5, 20, 50, 100] = canary → 20% → 50% → full
    current_step: usize,
}

impl GradualRollout {
    pub fn advance_step(&mut self) -> Result<(), String> {
        if self.current_step >= self.steps.len() {
            return Err("already at final step".to_string());
        }
        let green_pct = self.steps[self.current_step];
        let blue_pct = 100 - green_pct;
        self.controller.router.set_weights(blue_pct, green_pct)?;
        self.current_step += 1;
        Ok(())
    }
}
```

เพิ่ม test ที่ verify ว่า หลัง `advance_step()` 4 ครั้ง traffic ถึง 100% Green

### Exercise 3: Persistent Audit Log ด้วย JSON File (ระดับ Easy)

เพิ่ม method ให้ `DeploymentLog` บันทึกและโหลด log จากไฟล์ JSON:

```rust
use std::path::Path;

impl DeploymentLog {
    pub fn save_to_file(&self, path: &Path) -> std::io::Result<()> {
        let json = self.export_json().map_err(|e| {
            std::io::Error::new(std::io::ErrorKind::Other, e)
        })?;
        std::fs::write(path, json)
    }

    pub fn load_from_file(path: &Path) -> std::io::Result<Self> {
        let content = std::fs::read_to_string(path)?;
        let events: Vec<DeployEvent> = serde_json::from_str(&content)
            .map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidData, e))?;
        Ok(DeploymentLog { events })
    }
}
```

เขียน test ที่ save log ลงไฟล์ใน temp directory แล้ว load กลับขึ้นมาและตรวจสอบว่า events เหมือนเดิม

### Exercise 4: Multi-Region Deployment State (ระดับ Hard)

ขยาย `DeploymentState` ให้รองรับ multiple regions โดยแต่ละ region มี active_slot ของตัวเอง:

```rust
use std::collections::HashMap;

pub struct MultiRegionState {
    pub regions: HashMap<String, DeploymentState>,
}

impl MultiRegionState {
    pub fn new(regions: &[&str], initial_slot: Slot) -> Self {
        let mut map = HashMap::new();
        for &region in regions {
            map.insert(region.to_string(), DeploymentState::new(initial_slot));
        }
        MultiRegionState { regions: map }
    }

    /// Deploy ในทุก region พร้อมกัน
    pub fn deploy_all_regions(
        &mut self,
        version: &str,
        health_results: HashMap<String, (u32, u32)>,
    ) -> HashMap<String, Result<Slot, String>> {
        // implement this
        todo!()
    }

    /// Roll back ทุก region ที่ active_slot เป็น target_slot กลับ
    pub fn rollback_unhealthy_regions(&mut self) {
        todo!()
    }
}
```

### Exercise 5: Deployment Metrics Export (ระดับ Medium)

เชื่อมต่อกับ Prometheus metrics จาก Project G04 เพื่อ export deployment metrics:

```rust
pub struct DeploymentMetrics {
    deployments_total: Counter,
    rollbacks_total: Counter,
    active_slot: Gauge,             // 0 = Blue, 1 = Green
    deploy_duration_seconds: Histogram,
}

impl DeploymentMetrics {
    pub fn record_deploy(&self, slot: Slot, duration: Duration, success: bool) {
        self.deploy_duration_seconds.observe(duration.as_secs_f64());
        if success {
            self.deployments_total.inc();
            self.active_slot.set(match slot {
                Slot::Blue => 0.0,
                Slot::Green => 1.0,
            });
        } else {
            self.rollbacks_total.inc();
        }
    }
}
```

### Exercise 6: Webhook Notifications (ระดับ Medium)

เพิ่ม observer pattern สำหรับ notify ระบบภายนอก (Slack, PagerDuty) เมื่อ deployment events เกิดขึ้น:

```rust
pub trait DeploymentObserver: Send + Sync {
    fn on_event(&self, event: &DeployEvent);
}

pub struct SlackNotifier {
    webhook_url: String,
}

impl DeploymentObserver for SlackNotifier {
    fn on_event(&self, event: &DeployEvent) {
        let emoji = match event.kind {
            DeployEventKind::Promoted => "✅",
            DeployEventKind::RolledBack => "🔴",
            DeployEventKind::HealthFailed => "⚠️",
            _ => "ℹ️",
        };
        println!(
            "[Slack] {} Deployment event: {:?} on {} ({})",
            emoji, event.kind, event.slot, event.version
        );
        // ใน production: ส่ง HTTP POST ไปที่ webhook_url
    }
}
```

เพิ่ม `observers: Vec<Box<dyn DeploymentObserver>>` ใน `DeploymentController` และ call `observer.on_event()` ทุกครั้งที่ `log.append()` ถูกเรียก

## สรุป

ในโปรเจคนี้เราได้สร้าง **Blue-Green Deployment Controller** ที่ครบถ้วนตั้งแต่ data model ไปถึง FSM orchestration พร้อมด้วย:

**สิ่งที่สร้างขึ้น:**
- `Slot` enum พร้อม `opposite()` helper ที่ type-safe
- `Deployment` + `DeploymentState` สำหรับจัดการ dual-slot state
- `TrafficRouter` ที่ใช้ hash-based deterministic routing สำหรับ sticky sessions และ canary mode
- `HealthGate` ที่ enforce minimum healthy percentage ก่อน promote
- `DeploymentLog` ที่ append-only พร้อม JSON export
- `DeploymentController` ที่ implement FSM: Idle → Deploying → HealthChecking → Promoting/RollingBack → Active
- Cooldown enforcement ด้วย `std::time::Instant`
- Manual rollback พร้อม rollback history

**Pattern สำคัญที่ได้เรียน:**
1. **FSM ใน single method** — ซ่อน state transitions ไว้ใน `deploy()` เพื่อ atomicity
2. **Deterministic hash routing** — `DefaultHasher` สำหรับ sticky sessions
3. **`#[serde(skip)]`** — handle non-serializable types ใน struct
4. **`(&mut Option<T>).as_mut()`** — convert `&mut Option<T>` → `Option<&mut T>`
5. **`std::mem::discriminant`** — compare enum variants โดยไม่สนใจ payload
6. **Append-only log** — ไม่มี `remove()` API เพื่อ immutability guarantee

**การเชื่อมโยงไปโปรเจคถัดไป:**

โปรเจค [H01: HTTP Server](project-h01-http-server.md) จะนำความรู้เรื่อง async networking มาต่อยอด — เราจะเพิ่ม HTTP API endpoint ให้ `DeploymentController` รองรับ `POST /deploy`, `GET /status`, `POST /rollback` เพื่อให้ CI/CD pipeline เรียกใช้ได้ผ่าน HTTP แทนที่จะต้อง embed library โดยตรง

---

**โปรเจคก่อนหน้า:** [Project G09: Rate Limiter](project-g09-rate-limiter.md) | **โปรเจคถัดไป:** [Project H01: HTTP Server](project-h01-http-server.md)
