# Project F05: Saga Pattern Orchestrator

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ใน microservices architecture การทำ **distributed transaction** เป็นปัญหาที่ซับซ้อนมาก เพราะเราไม่สามารถใช้ two-phase commit (2PC) แบบ database เดียวได้ เมื่อ transaction ข้าม service หลายตัว — ถ้า service หนึ่งล้มเหลว เราจะ "roll back" ข้าม network อย่างไร?

**Saga Pattern** คือคำตอบที่ world-class systems อย่าง Uber, Netflix, และ Amazon ใช้ในการจัดการปัญหานี้ แทนที่จะ lock resources ทั้งหมดด้วย 2PC, Saga แบ่ง transaction ขนาดใหญ่ออกเป็น **series of local transactions** โดยแต่ละ step มี:
- **action** — ทำงานหลัก (เช่น จองที่นั่ง, ตัดเงิน, สร้าง shipment)
- **compensating transaction** — undo งานที่ทำไปแล้วถ้า step ต่อมาล้มเหลว (เช่น คืนเงิน, ปล่อยที่นั่ง, ยกเลิก shipment)

โปรเจคนี้จะสร้าง **Saga Orchestration Engine** แบบ production-grade ที่มี:
- **SagaDefinition** — ประกาศ steps พร้อม action และ compensate function
- **SagaOrchestrator** — engine ที่รัน steps ตามลำดับ ติดตามสถานะ และจัดการ compensation
- **Retry Policy** — exponential backoff with jitter สำหรับ transient failure
- **Event Log** — บันทึกทุก event ที่เกิดขึ้นใน saga lifecycle เพื่อ audit และ debugging
- **Persistence** — serialize state เป็น JSON หลังทุก step เพื่อ resume จาก checkpoint ได้

**Use case จริงในโลก production:**
- E-commerce order flow: จองสินค้า → ตัดเงิน → สร้าง shipment → ส่ง notification
- Travel booking: จองเที่ยวบิน → จองโรงแรม → จองรถ → ยืนยัน package
- Bank transfer: debit ต้นทาง → credit ปลายทาง → อัปเดต ledger → ส่ง receipt
- Onboarding flow: สร้าง user → ตั้งค่า billing → provision resources → ส่ง welcome email

ความแตกต่างระหว่าง **Orchestration** กับ **Choreography**: ใน orchestration มี central coordinator (orchestrator) ที่รู้ทุก step และสั่งการ ส่วน choreography แต่ละ service ฟัง event แล้วตัดสินใจเอง — โปรเจคนี้ focus ที่ orchestration เพราะเข้าใจง่ายกว่าและ debug ได้ง่ายกว่า

## สิ่งที่จะได้เรียนรู้

- **Saga Pattern** — แนวคิด distributed transaction management ด้วย compensating transactions
- **Higher-kinded abstraction ใน Rust** — ใช้ generic type parameter `S` (saga state) เพื่อให้ engine ทำงานกับ state ทุกประเภท
- **Trait object boxing** — `Box<dyn Fn(&mut S) -> Result<(), String> + Send + Sync>` สำหรับ type-erased callbacks
- **Builder pattern** — `SagaDefinition::new().add_step(...).add_step(...)` fluent API
- **Exponential backoff with jitter** — algorithm ที่ใช้จริงใน production เพื่อหลีกเลี่ยง thundering herd
- **Event sourcing lite** — บันทึก ordered event log เพื่อ reconstruct สถานะและ audit trail
- **JSON serialization** — `serde`/`serde_json` สำหรับ serialize/deserialize complex structs
- **In-memory checkpoint store** — simulate persistence layer ด้วย `HashMap<String, String>` (JSON)

## ความรู้ที่ต้องมีมาก่อน

- **Part 13-17**: Generic types, traits, trait bounds — ใช้สำหรับ `SagaStep<S>` generic
- **Part 18-22**: Closures, `Fn`/`FnMut`/`FnOnce` traits — ใช้สำหรับ action/compensate callbacks
- **Part 23-27**: `Box<dyn Trait>`, dynamic dispatch — ใช้สำหรับ type-erased step functions
- **Part 35-40**: Error handling, `Result<T, E>` — ใช้ทั่วโปรเจค
- **Part 46-50**: Iterators, `collect`, `filter_map` — ใช้ใน event log queries
- **Part 55-60**: `serde`, `serde_json` — ใช้สำหรับ serialize/deserialize saga state
- **Part 61-65**: `uuid` crate, UUID generation — ใช้สำหรับ saga execution ID
- **Part 96-100**: `Arc`, `Mutex` — ใช้ใน retry test ที่ share state ระหว่าง closures

## โครงสร้างโปรเจค (Project Layout)

```
saga/
├── src/
│   ├── lib.rs          # module root — re-exports ทุก submodule
│   ├── main.rs         # demo binary — OrderSaga example
│   ├── saga.rs         # SagaStep, SagaDefinition, SagaExecution, SagaOrchestrator
│   ├── retry.rs        # RetryPolicy struct + delay_ms() calculation
│   ├── events.rs       # SagaEvent enum, EventRecord, EventLog
│   └── store.rs        # SagaStore — in-memory JSON persistence
│   └── tests.rs        # unit tests ทั้งหมด (17 tests)
└── Cargo.toml
```

**หมายเหตุ**: โปรเจคจริงในระบบ production มักแยก store ออกเป็น trait เพื่อ swap implementation ระหว่าง in-memory, PostgreSQL, หรือ Redis ได้ ในโปรเจคนี้เราใช้ in-memory เพื่อความเข้าใจ แต่ design ให้ extend ได้ง่าย

## การออกแบบ (Architecture & Design)

### Data Flow Overview

```
ผู้ใช้สร้าง SagaDefinition<S>
         │
         │ .add_step(SagaStep { action, compensate, retry_policy })
         │ .add_step(...)
         ▼
   SagaDefinition<S>
   [ step0, step1, step2, ... stepN ]
         │
         │ orchestrator.run(&definition, &mut state)
         ▼
   SagaOrchestrator
         │
         ├─ Forward Pass ──────────────────────────────────────────────────────
         │  for each step:
         │    1. push StepStarted to EventLog
         │    2. call step.action(&mut state)
         │       OK  → push StepCompleted, add to completed_steps, save checkpoint
         │       Err → retry up to max_retries times
         │             push StepFailed per attempt
         │             if all retries exhausted → enter Compensation Pass
         │
         └─ Compensation Pass (on failure) ────────────────────────────────────
            for each completed step in REVERSE order:
              1. push CompensationStarted
              2. call step.compensate(&mut state)
                 OK  → push CompensationCompleted
                 Err → push CompensationFailed, add to failed_steps
              3. save checkpoint
            final status:
              all compensations OK → Compensated
              some compensations failed → Failed (PartialFailure)
```

### การออกแบบ Generic State `S`

การที่ engine รับ `state: &mut S` แทนที่จะ hardcode type ทำให้:

```
SagaStep<OrderState>    — ใช้กับ e-commerce order
SagaStep<BookingState>  — ใช้กับ travel booking
SagaStep<TransferState> — ใช้กับ bank transfer
```

ทั้งหมดนี้ใช้ engine เดียวกันโดยไม่ต้องเขียน orchestration logic ซ้ำ

### Event Sourcing Mini-pattern

`EventLog` ใน `SagaExecution` เป็น append-only log ที่ทำหน้าที่เหมือน audit trail — เราสามารถ reconstruct สถานะของ saga ได้จาก event log เพียงอย่างเดียว นี่คือแนวคิดพื้นฐานของ Event Sourcing ที่จะเรียนใน Project F06

### Retry Strategy

```
attempt 1: ทันที
attempt 2: sleep(base * 2^0) = base ms
attempt 3: sleep(base * 2^1) = base*2 ms
attempt 4: sleep(base * 2^2) = base*4 ms
...
+ jitter: +0 ถึง +50% ของ delay แต่ละครั้ง (เพื่อหลีกเลี่ยง thundering herd)
```

ในโปรเจคนี้เราไม่ใส่ actual `sleep` (เพื่อให้ test เร็ว) แต่ `delay_ms()` method คำนวณค่าที่ถูกต้อง — production code จะนำค่านี้ไปใช้กับ `tokio::time::sleep`

### Compensation ≠ Rollback

สิ่งสำคัญที่ต้องเข้าใจ: **compensation ไม่ใช่ database rollback** มันคือ business operation ที่ทำงาน "ย้อนกลับ" เช่น:
- ถ้า action = "charge credit card $100" → compensation = "refund $100" (ไม่ใช่แค่ cancel charge)
- ถ้า action = "send email" → compensation = "send cancellation email" (ไม่สามารถ un-send ได้)
- Compensation อาจล้มเหลวได้ (เช่น payment gateway ล่ม) → ต้องติดตาม `PartialFailure`

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างพื้นฐาน — Event Types และ RetryPolicy

เริ่มจาก "building blocks" ที่เล็กที่สุดก่อน ได้แก่ event types สำหรับ audit log และ retry policy

**`Cargo.toml`**

```toml
[package]
name = "saga"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
rand = "0.8"
```

**`src/events.rs`** — Event types สำหรับ audit log

```rust
use serde::{Deserialize, Serialize};
use std::time::{SystemTime, UNIX_EPOCH};

/// แต่ละ event ที่เกิดขึ้นใน Saga lifecycle
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum SagaEvent {
    StepStarted { step_name: String, attempt: u32 },
    StepCompleted { step_name: String, attempt: u32 },
    StepFailed { step_name: String, attempt: u32, error: String },
    CompensationStarted { step_name: String },
    CompensationCompleted { step_name: String },
    CompensationFailed { step_name: String, error: String },
    SagaCompleted,
    SagaFailed { failed_step: String },
}

/// บันทึก event พร้อม timestamp (milliseconds since UNIX epoch)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventRecord {
    pub timestamp_ms: u64,
    pub event: SagaEvent,
}

impl EventRecord {
    pub fn new(event: SagaEvent) -> Self {
        let timestamp_ms = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_millis() as u64;
        Self { timestamp_ms, event }
    }
}

/// Event log สำหรับ saga หนึ่ง instance — append-only
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct EventLog {
    pub records: Vec<EventRecord>,
}

impl EventLog {
    pub fn push(&mut self, event: SagaEvent) {
        self.records.push(EventRecord::new(event));
    }

    pub fn events(&self) -> impl Iterator<Item = &SagaEvent> {
        self.records.iter().map(|r| &r.event)
    }

    pub fn count(&self) -> usize {
        self.records.len()
    }
}
```

**`src/retry.rs`** — Retry policy ด้วย exponential backoff

```rust
use rand::Rng;

/// นโยบายการ retry สำหรับแต่ละ step
#[derive(Debug, Clone)]
pub struct RetryPolicy {
    pub max_retries: u32,
    pub base_backoff_ms: u64,
    pub jitter: bool,
}

impl Default for RetryPolicy {
    fn default() -> Self {
        Self {
            max_retries: 0,      // ไม่ retry โดย default
            base_backoff_ms: 100,
            jitter: false,
        }
    }
}

impl RetryPolicy {
    pub fn new(max_retries: u32, base_backoff_ms: u64, jitter: bool) -> Self {
        Self { max_retries, base_backoff_ms, jitter }
    }

    /// คำนวณ delay ก่อน retry ครั้งที่ `attempt` (เริ่มที่ 1)
    /// สูตร: base * 2^(attempt-1) + optional random jitter (0 ถึง 50% ของ delay)
    pub fn delay_ms(&self, attempt: u32) -> u64 {
        let exp = self.base_backoff_ms
            .saturating_mul(2u64.saturating_pow(attempt.saturating_sub(1)));
        if self.jitter {
            let jitter_range = exp / 2;
            let j = if jitter_range > 0 {
                rand::thread_rng().gen_range(0..jitter_range)
            } else {
                0
            };
            exp.saturating_add(j)
        } else {
            exp
        }
    }
}
```

**ทดสอบเบื้องต้น:**

```rust
#[test]
fn test_retry_delay_exponential() {
    let policy = RetryPolicy::new(5, 100, false);
    assert_eq!(policy.delay_ms(1), 100);  // 100 * 2^0 = 100
    assert_eq!(policy.delay_ms(2), 200);  // 100 * 2^1 = 200
    assert_eq!(policy.delay_ms(3), 400);  // 100 * 2^2 = 400
    assert_eq!(policy.delay_ms(4), 800);  // 100 * 2^3 = 800
}
```

---

### ขั้นที่ 2: SagaStep และ SagaDefinition

ขั้นนี้สร้าง data structures หลักสำหรับ "ประกาศ" saga — ยังไม่มี execution logic

**`src/saga.rs`** — ส่วนแรก: type definitions

```rust
use crate::retry::RetryPolicy;

/// หนึ่งขั้นตอนใน Saga
/// S = type ของ saga state ที่ทุก step ใช้ร่วมกัน
pub struct SagaStep<S> {
    pub name: String,
    /// ฟังก์ชันหลักที่ทำงาน business logic
    pub action: Box<dyn Fn(&mut S) -> Result<(), String> + Send + Sync>,
    /// ฟังก์ชัน compensating transaction — ทำงานเมื่อ step หลังล้มเหลว
    pub compensate: Box<dyn Fn(&mut S) -> Result<(), String> + Send + Sync>,
    pub retry_policy: RetryPolicy,
}

impl<S> SagaStep<S> {
    pub fn new(
        name: impl Into<String>,
        action: impl Fn(&mut S) -> Result<(), String> + Send + Sync + 'static,
        compensate: impl Fn(&mut S) -> Result<(), String> + Send + Sync + 'static,
    ) -> Self {
        Self {
            name: name.into(),
            action: Box::new(action),
            compensate: Box::new(compensate),
            retry_policy: RetryPolicy::default(),
        }
    }

    /// ใส่ retry policy ให้ step นี้ (builder pattern)
    pub fn with_retry(mut self, policy: RetryPolicy) -> Self {
        self.retry_policy = policy;
        self
    }
}

/// รายการ steps ที่เรียงลำดับ — "สูตร" ของ saga
pub struct SagaDefinition<S> {
    pub name: String,
    pub steps: Vec<SagaStep<S>>,
}

impl<S> SagaDefinition<S> {
    pub fn new(name: impl Into<String>) -> Self {
        Self { name: name.into(), steps: Vec::new() }
    }

    /// เพิ่ม step เข้า definition — builder pattern ทำ chain ได้
    pub fn add_step(mut self, step: SagaStep<S>) -> Self {
        self.steps.push(step);
        self
    }
}
```

**ตัวอย่างการประกาศ OrderSaga:**

```rust
#[derive(Clone, Default)]
struct OrderState {
    payment_reserved: bool,
    inventory_reserved: bool,
    shipping_created: bool,
}

let definition = SagaDefinition::new("OrderSaga")
    .add_step(SagaStep::new(
        "ReservePayment",
        |s: &mut OrderState| {
            s.payment_reserved = true;
            println!("  ✓ Payment reserved");
            Ok(())
        },
        |s: &mut OrderState| {
            s.payment_reserved = false;
            println!("  ↩ Payment released");
            Ok(())
        },
    ))
    .add_step(SagaStep::new(
        "ReserveInventory",
        |s: &mut OrderState| {
            s.inventory_reserved = true;
            println!("  ✓ Inventory reserved");
            Ok(())
        },
        |s: &mut OrderState| {
            s.inventory_reserved = false;
            println!("  ↩ Inventory released");
            Ok(())
        },
    ));
```

จุดสำคัญที่ต้องเข้าใจ: `Box<dyn Fn(&mut S) -> Result<(), String> + Send + Sync>` บอกว่า:
- `dyn Fn(...)` — trait object, รองรับ closure ทุกประเภทที่ implement `Fn`
- `Send + Sync` — ปลอดภัยในการส่งข้าม thread (จำเป็นสำหรับ async context)
- `'static` lifetime ใน impl block — closure ต้องไม่ borrow local variables ที่หมดอายุ

---

### ขั้นที่ 3: SagaExecution และ Status Types

ขั้นนี้สร้าง struct สำหรับติดตามสถานะ runtime ของ saga

**`src/saga.rs`** — ส่วนที่สอง: execution state

```rust
use crate::events::EventLog;
use serde::{Deserialize, Serialize};

/// สถานะหลักของ Saga
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum SagaStatus {
    Running,      // กำลัง execute forward pass
    Completed,    // ทุก step สำเร็จ
    Compensating, // กำลัง execute compensation
    Compensated,  // compensation สำเร็จทั้งหมด
    Failed,       // compensation บางส่วนล้มเหลวด้วย
}

/// สถานะของ compensation process
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum CompensationStatus {
    NotStarted,
    InProgress,
    Completed,
    PartialFailure { failed_steps: Vec<String> },
}

/// สถานะ runtime ของ saga execution หนึ่ง instance
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SagaExecution {
    pub id: String,                          // UUID ที่ unique ต่อ run
    pub saga_name: String,                   // ชื่อ definition
    pub status: SagaStatus,
    pub completed_steps: Vec<String>,        // steps ที่ action สำเร็จ
    pub failed_step: Option<String>,         // step ที่ trigger compensation
    pub compensation_status: CompensationStatus,
    pub event_log: EventLog,                 // ordered audit trail
}

impl SagaExecution {
    pub fn new(id: String, saga_name: String) -> Self {
        Self {
            id,
            saga_name,
            status: SagaStatus::Running,
            completed_steps: Vec::new(),
            failed_step: None,
            compensation_status: CompensationStatus::NotStarted,
            event_log: EventLog::default(),
        }
    }
}
```

**`src/store.rs`** — In-memory persistence store

```rust
use crate::saga::SagaExecution;
use std::collections::HashMap;

/// In-memory saga state store (serialize state เป็น JSON string)
/// ใน production จะแทนด้วย PostgreSQL, Redis, หรือ DynamoDB
#[derive(Default)]
pub struct SagaStore {
    data: HashMap<String, String>,
}

impl SagaStore {
    pub fn new() -> Self {
        Self { data: HashMap::new() }
    }

    /// บันทึก execution ลง store (serialize เป็น JSON)
    pub fn save(&mut self, execution: &SagaExecution) {
        let json = serde_json::to_string(execution)
            .expect("serialization failed");
        self.data.insert(execution.id.clone(), json);
    }

    /// โหลด execution จาก store (deserialize จาก JSON)
    pub fn load(&self, id: &str) -> Option<SagaExecution> {
        self.data.get(id)
            .and_then(|json| serde_json::from_str(json).ok())
    }

    pub fn count(&self) -> usize {
        self.data.len()
    }
}
```

---

### ขั้นที่ 4: SagaOrchestrator — Forward Execution

นี่คือ "engine" หลัก ขั้นนี้ implement แค่ forward pass (happy path) ก่อน

**`src/saga.rs`** — ส่วน SagaOrchestrator (forward pass เท่านั้น)

```rust
use crate::store::SagaStore;

pub struct SagaOrchestrator {
    pub store: SagaStore,
}

impl SagaOrchestrator {
    pub fn new() -> Self {
        Self { store: SagaStore::new() }
    }

    pub fn run<S>(&mut self, definition: &SagaDefinition<S>, state: &mut S) -> SagaExecution {
        let id = uuid::Uuid::new_v4().to_string();
        let mut execution = SagaExecution::new(id, definition.name.clone());

        for step in &definition.steps {
            // 1. บันทึก event ว่าเริ่ม step
            execution.event_log.push(SagaEvent::StepStarted {
                step_name: step.name.clone(),
                attempt: 1,
            });

            // 2. รัน action
            match (step.action)(state) {
                Ok(()) => {
                    // สำเร็จ: บันทึก event, เพิ่มใน completed_steps, save checkpoint
                    execution.completed_steps.push(step.name.clone());
                    execution.event_log.push(SagaEvent::StepCompleted {
                        step_name: step.name.clone(),
                        attempt: 1,
                    });
                    self.store.save(&execution);
                }
                Err(e) => {
                    // ล้มเหลว (ยังไม่มี retry หรือ compensation ในขั้นนี้)
                    execution.event_log.push(SagaEvent::StepFailed {
                        step_name: step.name.clone(),
                        attempt: 1,
                        error: e,
                    });
                    execution.failed_step = Some(step.name.clone());
                    break;
                }
            }
        }

        // Happy path: ทุก step สำเร็จ
        if execution.failed_step.is_none() {
            execution.status = SagaStatus::Completed;
            execution.event_log.push(SagaEvent::SagaCompleted);
        }
        self.store.save(&execution);
        execution
    }
}
```

**ทดสอบ forward pass:**

```rust
#[test]
fn test_happy_path_all_steps_complete() {
    let def = SagaDefinition::new("T1")
        .add_step(ok_step("A"))
        .add_step(ok_step("B"))
        .add_step(ok_step("C"));

    let mut state = S::default();
    let mut orch = SagaOrchestrator::new();
    let exec = orch.run(&def, &mut state);

    assert_eq!(exec.status, SagaStatus::Completed);
    assert_eq!(exec.completed_steps, vec!["A", "B", "C"]);
    assert!(exec.failed_step.is_none());
    assert_eq!(state.log, vec!["do:A", "do:B", "do:C"]);
}
```

---

### ขั้นที่ 5: Compensation Logic

ขั้นนี้เพิ่ม compensation pass เมื่อมี step ล้มเหลว

```rust
// ใน run() ต่อจาก forward pass...

// ─── Compensation Pass ────────────────────────────────────────────────────
if let Some(failed_idx) = failed_at {
    execution.compensation_status = CompensationStatus::InProgress;
    let mut comp_failures: Vec<String> = Vec::new();

    // Compensate steps ที่ complete แล้ว ในลำดับ REVERSE
    // (steps ที่ index 0..failed_idx ถูก complete)
    for step in definition.steps[..failed_idx].iter().rev() {
        execution.event_log.push(SagaEvent::CompensationStarted {
            step_name: step.name.clone(),
        });

        match (step.compensate)(state) {
            Ok(()) => {
                execution.event_log.push(SagaEvent::CompensationCompleted {
                    step_name: step.name.clone(),
                });
            }
            Err(e) => {
                execution.event_log.push(SagaEvent::CompensationFailed {
                    step_name: step.name.clone(),
                    error: e,
                });
                comp_failures.push(step.name.clone());
            }
        }
        // Save checkpoint หลังทุก compensation step
        self.store.save(&execution);
    }

    // อัปเดต final status
    if comp_failures.is_empty() {
        execution.compensation_status = CompensationStatus::Completed;
        execution.status = SagaStatus::Compensated;
    } else {
        execution.compensation_status = CompensationStatus::PartialFailure {
            failed_steps: comp_failures,
        };
        execution.status = SagaStatus::Failed;
    }
    self.store.save(&execution);
}
```

**ทดสอบ compensation:**

```rust
#[test]
fn test_last_step_fails_compensates_all_previous() {
    let def = SagaDefinition::new("Test")
        .add_step(ok_step("A"))
        .add_step(ok_step("B"))
        .add_step(fail_step("C"));  // C ล้มเหลว

    let mut state = S::default();
    let mut orch = SagaOrchestrator::new();
    let exec = orch.run(&def, &mut state);

    assert_eq!(exec.status, SagaStatus::Compensated);
    assert_eq!(exec.completed_steps, vec!["A", "B"]);
    // compensation รันย้อนหลัง: B ก่อน แล้วค่อย A
    assert_eq!(state.log, vec!["do:A", "do:B", "undo:B", "undo:A"]);
}
```

---

### ขั้นที่ 6: Retry Logic

ขั้นนี้เพิ่ม retry loop ในก่อนที่จะ trigger compensation

```rust
// ใน forward pass loop ต่อจาก StepStarted event...

let max_attempts = step.retry_policy.max_retries + 1;
let mut last_err = String::new();

'retry: for attempt in 1..=max_attempts {
    // บันทึก StepStarted สำหรับทุก attempt (attempt > 1 = retry)
    if attempt > 1 {
        execution.event_log.push(SagaEvent::StepStarted {
            step_name: step.name.clone(),
            attempt,
        });
        // ใน production: tokio::time::sleep(Duration::from_millis(
        //     step.retry_policy.delay_ms(attempt - 1)
        // )).await;
    }

    match (step.action)(state) {
        Ok(()) => {
            execution.completed_steps.push(step.name.clone());
            execution.event_log.push(SagaEvent::StepCompleted {
                step_name: step.name.clone(),
                attempt,
            });
            self.store.save(&execution);
            continue 'steps;  // ไป step ถัดไป
        }
        Err(e) => {
            last_err = e.clone();
            execution.event_log.push(SagaEvent::StepFailed {
                step_name: step.name.clone(),
                attempt,
                error: e,
            });
        }
    }
}

// ทุก attempt หมดแล้ว → trigger compensation
execution.failed_step = Some(step.name.clone());
execution.status = SagaStatus::Compensating;
failed_at = Some(idx);
execution.event_log.push(SagaEvent::SagaFailed {
    failed_step: step.name.clone(),
});
self.store.save(&execution);
break 'steps;
```

**ทดสอบ retry exhaustion:**

```rust
#[test]
fn test_retry_exhaustion_triggers_compensation() {
    let step_b = SagaStep::new(
        "B",
        |_s: &mut S| Err("always fails".to_string()),
        |s: &mut S| { s.log.push("undo:B".into()); Ok(()) },
    )
    .with_retry(RetryPolicy::new(2, 0, false)); // max 2 retries = 3 attempts total

    let def = SagaDefinition::new("Test")
        .add_step(ok_step("A"))
        .add_step(step_b);

    let mut state = S::default();
    let mut orch = SagaOrchestrator::new();
    let exec = orch.run(&def, &mut state);

    assert_eq!(exec.status, SagaStatus::Compensated);

    // ต้องมี 3 StepFailed events สำหรับ B (1 initial + 2 retries)
    let fail_count = exec.event_log.events()
        .filter(|e| matches!(e,
            SagaEvent::StepFailed { step_name, .. } if step_name == "B"
        ))
        .count();
    assert_eq!(fail_count, 3);
}
```

**ทดสอบ retry สำเร็จใน attempt ที่ 2:**

```rust
#[test]
fn test_retry_succeeds_on_second_attempt() {
    use std::sync::{Arc, Mutex};
    let call_count = Arc::new(Mutex::new(0u32));
    let cc = call_count.clone();

    let step_b = SagaStep::new(
        "B",
        move |s: &mut S| {
            let mut c = cc.lock().unwrap();
            *c += 1;
            if *c < 2 {
                Err("not yet".to_string())
            } else {
                s.log.push("do:B".into());
                Ok(())
            }
        },
        |s: &mut S| { s.log.push("undo:B".into()); Ok(()) },
    )
    .with_retry(RetryPolicy::new(3, 0, false));

    let def = SagaDefinition::new("Test").add_step(ok_step("A")).add_step(step_b);

    let mut state = S::default();
    let mut orch = SagaOrchestrator::new();
    let exec = orch.run(&def, &mut state);

    assert_eq!(exec.status, SagaStatus::Completed);
    assert_eq!(*call_count.lock().unwrap(), 2); // เรียก 2 ครั้ง
}
```

---

### ขั้นที่ 7: Persistence และ Checkpoint Resume

ขั้นนี้ demo วิธี serialize state เป็น JSON และ load กลับมา

```rust
// เพิ่ม method load() ใน SagaOrchestrator
impl SagaOrchestrator {
    /// โหลด execution จาก store (ใช้สำหรับ resume หรือ inspection)
    pub fn load(&self, id: &str) -> Option<SagaExecution> {
        self.store.load(id)
    }
}
```

**ตัวอย่าง JSON ที่ถูก serialize:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "saga_name": "OrderSaga",
  "status": "Completed",
  "completed_steps": ["ReservePayment", "ReserveInventory", "CreateShipment"],
  "failed_step": null,
  "compensation_status": "NotStarted",
  "event_log": {
    "records": [
      {
        "timestamp_ms": 1735000000000,
        "event": { "StepStarted": { "step_name": "ReservePayment", "attempt": 1 } }
      },
      {
        "timestamp_ms": 1735000000001,
        "event": { "StepCompleted": { "step_name": "ReservePayment", "attempt": 1 } }
      }
    ]
  }
}
```

**ทดสอบ checkpoint resume:**

```rust
#[test]
fn test_checkpoint_resume_loads_correct_state() {
    let def = SagaDefinition::new("T8")
        .add_step(ok_step("X"))
        .add_step(ok_step("Y"))
        .add_step(ok_step("Z"));

    let mut state = S::default();
    let mut orch = SagaOrchestrator::new();
    let exec = orch.run(&def, &mut state);
    let exec_id = exec.id.clone();

    // Simulate loading from checkpoint (e.g., หลัง process restart)
    let resumed = orch.load(&exec_id).expect("checkpoint exists");
    assert_eq!(resumed.saga_name, "T8");
    assert_eq!(resumed.completed_steps.len(), 3);
    assert_eq!(resumed.status, SagaStatus::Completed);
}
```

**ทดสอบ JSON round-trip:**

```rust
#[test]
fn test_store_json_round_trip() {
    let def = SagaDefinition::new("T14")
        .add_step(ok_step("A"))
        .add_step(fail_step("B"));

    let mut state = S::default();
    let mut orch = SagaOrchestrator::new();
    let exec = orch.run(&def, &mut state);
    let id = exec.id.clone();

    // Serialize ด้วยตนเอง แล้ว deserialize กลับมา
    let json = serde_json::to_string(&exec).expect("serialize");
    let back: SagaExecution = serde_json::from_str(&json).expect("deserialize");

    assert_eq!(back.id, id);
    assert_eq!(back.compensation_status, CompensationStatus::Completed);
}
```

---

### ขั้นที่ 8: Demo Binary — OrderSaga

ขั้นสุดท้ายสร้าง demo program ที่แสดงให้เห็น saga แบบ end-to-end

**`src/main.rs`**

```rust
use saga::{SagaDefinition, SagaOrchestrator, SagaStep};

#[derive(Clone, Default, Debug)]
struct OrderState {
    order_id: u64,
    payment_reserved: bool,
    inventory_reserved: bool,
    shipping_created: bool,
    log: Vec<String>,
}

fn main() {
    let mut state = OrderState { order_id: 42, ..Default::default() };

    let definition = SagaDefinition::new("OrderSaga")
        .add_step(SagaStep::new(
            "ReservePayment",
            |s: &mut OrderState| {
                s.payment_reserved = true;
                s.log.push("payment reserved".into());
                Ok(())
            },
            |s: &mut OrderState| {
                s.payment_reserved = false;
                s.log.push("payment released".into());
                Ok(())
            },
        ))
        .add_step(SagaStep::new(
            "ReserveInventory",
            |s: &mut OrderState| {
                s.inventory_reserved = true;
                s.log.push("inventory reserved".into());
                Ok(())
            },
            |s: &mut OrderState| {
                s.inventory_reserved = false;
                s.log.push("inventory released".into());
                Ok(())
            },
        ))
        .add_step(SagaStep::new(
            "CreateShipment",
            |s: &mut OrderState| {
                s.shipping_created = true;
                s.log.push("shipment created".into());
                Ok(())
            },
            |s: &mut OrderState| {
                s.shipping_created = false;
                s.log.push("shipment cancelled".into());
                Ok(())
            },
        ));

    let mut orchestrator = SagaOrchestrator::new();
    let result = orchestrator.run(&definition, &mut state);

    println!("Saga status: {:?}", result.status);
    println!("Completed steps: {:?}", result.completed_steps);
    println!("State log: {:?}", state.log);
    println!("Events recorded: {}", result.event_log.count());
}
```

**Output จาก `cargo run`:**

```
Saga status: Completed
Completed steps: ["ReservePayment", "ReserveInventory", "CreateShipment"]
State log: ["payment reserved", "inventory reserved", "shipment created"]
Events recorded: 7
```

---

## การทดสอบ (Testing)

### โครงสร้าง Test Suite

โปรเจคนี้มี **17 unit tests** ครอบคลุม scenario สำคัญทุกอย่าง:

| Test | Scenario |
|------|----------|
| `test_happy_path_all_steps_complete` | ทุก step สำเร็จ → status Completed |
| `test_first_step_fails_no_compensation` | Step แรกล้มเหลว → ไม่มีอะไรต้อง compensate |
| `test_second_step_fails_compensates_first` | Step ที่ 2 ล้มเหลว → compensate step 1 |
| `test_last_step_fails_compensates_all_previous` | Step สุดท้ายล้มเหลว → compensate ทุก step ย้อนหลัง |
| `test_retry_exhaustion_triggers_compensation` | Retry หมด → trigger compensation |
| `test_partial_compensation_failure` | Compensation บาง step ล้มเหลว → status Failed |
| `test_execution_persisted_to_store` | State ถูก save ลง store |
| `test_checkpoint_resume_loads_correct_state` | โหลด state จาก checkpoint ได้ถูกต้อง |
| `test_event_log_order_happy_path` | Event log มีลำดับถูกต้อง (happy path) |
| `test_event_log_order_compensation` | CompensationStarted events เรียงย้อนหลัง |
| `test_event_log_contains_saga_failed` | มี SagaFailed event เมื่อ saga ล้มเหลว |
| `test_single_step_completes` | Saga ที่มี step เดียว |
| `test_retry_succeeds_on_second_attempt` | Retry สำเร็จใน attempt ที่ 2 |
| `test_store_json_round_trip` | JSON serialize/deserialize ถูกต้อง |
| `test_multiple_sagas_independent` | Saga หลาย instance ไม่ขัดแย้งกัน |
| `test_retry_delay_exponential` | คำนวณ exponential backoff ถูกต้อง |
| `test_compensation_reverse_order` | Compensation รันย้อนหลังเสมอ |

### Test Helpers

```rust
fn ok_step(name: &'static str) -> SagaStep<S> {
    SagaStep::new(
        name,
        move |s: &mut S| { s.log.push(format!("do:{}", name)); Ok(()) },
        move |s: &mut S| { s.log.push(format!("undo:{}", name)); Ok(()) },
    )
}

fn fail_step(name: &'static str) -> SagaStep<S> {
    SagaStep::new(
        name,
        move |_s: &mut S| Err(format!("{} failed", name)),
        move |s: &mut S| { s.log.push(format!("undo:{}", name)); Ok(()) },
    )
}

fn compensate_fail_step(name: &'static str) -> SagaStep<S> {
    SagaStep::new(
        name,
        move |s: &mut S| { s.log.push(format!("do:{}", name)); Ok(()) },
        move |_s: &mut S| Err(format!("compensate:{} failed", name)),
    )
}
```

### ผลการรัน `cargo test` จริง

```
running 17 tests
test tests::tests::test_checkpoint_resume_loads_correct_state ... ok
test tests::tests::test_event_log_order_compensation ... ok
test tests::tests::test_compensation_reverse_order ... ok
test tests::tests::test_event_log_contains_saga_failed ... ok
test tests::tests::test_event_log_order_happy_path ... ok
test tests::tests::test_execution_persisted_to_store ... ok
test tests::tests::test_first_step_fails_no_compensation ... ok
test tests::tests::test_happy_path_all_steps_complete ... ok
test tests::tests::test_last_step_fails_compensates_all_previous ... ok
test tests::tests::test_multiple_sagas_independent ... ok
test tests::tests::test_retry_delay_exponential ... ok
test tests::tests::test_retry_succeeds_on_second_attempt ... ok
test tests::tests::test_retry_exhaustion_triggers_compensation ... ok
test tests::tests::test_partial_compensation_failure ... ok
test tests::tests::test_single_step_completes ... ok
test tests::tests::test_second_step_fails_compensates_first ... ok
test tests::tests::test_store_json_round_trip ... ok

test result: ok. 17 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### Pitfall 1: ลืม `'static` bound บน closure

**ปัญหา:** เมื่อ borrow local variable ใน action closure

```rust
// ❌ ไม่ compile: closure borrows `config` ที่อยู่ใน stack
let config = Config::new();
SagaStep::new(
    "Step1",
    |s: &mut S| {
        s.apply(&config); // ERROR: `config` does not live long enough
        Ok(())
    },
    |_| Ok(()),
)
```

**วิธีแก้:** ใช้ `move` closure หรือ clone ค่าก่อน

```rust
// ✓ move ค่าเข้า closure
let config = Config::new();
SagaStep::new(
    "Step1",
    move |s: &mut S| {
        s.apply(&config); // OK: config ถูก move เข้า closure
        Ok(())
    },
    |_| Ok(()),
)
```

### Pitfall 2: Compensation ไม่ใช่ Idempotent

**ปัญหา:** ถ้า compensation ถูกรันสองครั้ง (เช่น หลัง process restart) อาจเกิดปัญหา

```rust
// ❌ ไม่ idempotent: charge สองครั้งถ้า compensation รันซ้ำ
compensate: |s| {
    charge_card(s.card_id, s.amount)?; // อาจ charge ซ้ำ!
    Ok(())
}
```

**วิธีแก้:** ใส่ idempotency key หรือ check ก่อน compensate

```rust
// ✓ idempotent: check ก่อน
compensate: |s| {
    if !s.already_refunded {
        refund_card(s.card_id, s.amount, s.refund_key)?;
        s.already_refunded = true;
    }
    Ok(())
}
```

### Pitfall 3: ใช้ `FnMut` แทน `Fn` ทำให้ call closure ซ้ำไม่ได้

**ปัญหา:** ถ้า action closure mutate ตัวเอง (capture by `&mut`) และต้องการ call หลายครั้ง (retry)

```rust
// ❌ ปัญหา: ถ้าใช้ FnMut แต่ Box เป็น Fn จะ compile error
// หรือถ้า closure capture variable ด้วย `&mut` ใน retry loop
```

**วิธีแก้:** ใช้ shared state ผ่าน `Arc<Mutex<T>>` สำหรับ retry counter

```rust
// ✓ ใช้ Arc<Mutex<u32>> สำหรับ call count ที่ต้อง mutate
use std::sync::{Arc, Mutex};
let count = Arc::new(Mutex::new(0u32));
let c = count.clone();

SagaStep::new(
    "RetryStep",
    move |_s| {
        *c.lock().unwrap() += 1;
        Ok(())
    },
    |_| Ok(()),
)
```

### Pitfall 4: Compensation รันตาม Index ผิด

**ปัญหา:** เมื่อ step ที่ index `i` ล้มเหลว ควร compensate steps ที่ index `0..i` เท่านั้น (ไม่รวม `i` เพราะ `i` ไม่เคย complete)

```rust
// ❌ ผิด: compensate ทุก step รวมถึง failed step
for step in definition.steps.iter().rev() { ... }

// ✓ ถูก: compensate เฉพาะ steps ที่ complete แล้ว (0..failed_idx)
for step in definition.steps[..failed_idx].iter().rev() { ... }
```

**ตัวอย่างที่ชัดเจน:**
```
steps:  [A, B, C, D]  (D ล้มเหลว)
completed: [A, B, C]
compensation index: [..3] = [A, B, C] → reverse → [C, B, A]
```

### Pitfall 5: ลืม Save Checkpoint หลังทุก Step

**ปัญหา:** ถ้า process crash ระหว่าง compensation เราต้องรู้ว่า compensate ไปถึงไหนแล้ว ถ้าไม่ save หลังทุก step จะ compensate ซ้ำ

```rust
// ❌ save แค่ตอนสุดท้าย — ถ้า crash ระหว่างกลาง ข้อมูลหาย
for step in steps.iter().rev() {
    (step.compensate)(state)?;
}
self.store.save(&execution); // สายเกินไป

// ✓ save หลังทุก compensation step
for step in steps.iter().rev() {
    (step.compensate)(state);
    self.store.save(&execution); // ทันที
}
```

### Pitfall 6: ไม่แยก `SagaStatus::Compensated` กับ `SagaStatus::Failed`

**ปัญหา:** หลาย implementation ไม่แยกระหว่าง "compensation สำเร็จ" กับ "compensation ล้มเหลวบางส่วน" ทำให้ monitoring ไม่ชัดเจน

```rust
// ❌ ไม่ชัดเจน: ไม่รู้ว่า compensation สำเร็จหรือเปล่า
enum SagaStatus { Running, Completed, Failed }

// ✓ ชัดเจน:
enum SagaStatus {
    Running,
    Completed,      // ทุก action สำเร็จ
    Compensating,   // กำลัง compensate
    Compensated,    // compensation สำเร็จทั้งหมด (clean rollback)
    Failed,         // compensation บางส่วนล้มเหลว (manual intervention required)
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
./target/release/saga
```

### Integration กับ Tokio (Async Version)

ในระบบ production จริง แต่ละ step action จะเป็น async function ที่ call remote service:

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust
use std::future::Future;
use std::pin::Pin;

// Async version ของ SagaStep
pub struct AsyncSagaStep<S> {
    pub name: String,
    pub action: Box<
        dyn Fn(&mut S) -> Pin<Box<dyn Future<Output = Result<(), String>> + Send>>
        + Send + Sync
    >,
    pub compensate: Box<
        dyn Fn(&mut S) -> Pin<Box<dyn Future<Output = Result<(), String>> + Send>>
        + Send + Sync
    >,
    pub retry_policy: RetryPolicy,
}

// Macro helper เพื่อให้เขียนง่ายขึ้น
macro_rules! async_step {
    ($action:expr, $compensate:expr) => {
        AsyncSagaStep {
            action: Box::new(|s| Box::pin($action(s))),
            compensate: Box::new(|s| Box::pin($compensate(s))),
            ..Default::default()
        }
    };
}
```

### Integration กับ Database Store

แทนที่ `SagaStore` ด้วย PostgreSQL store:

```rust
// store trait สำหรับ swap implementation
pub trait SagaStoreBackend: Send + Sync {
    fn save(&mut self, execution: &SagaExecution) -> Result<(), Box<dyn std::error::Error>>;
    fn load(&self, id: &str) -> Result<Option<SagaExecution>, Box<dyn std::error::Error>>;
}

// In-memory implementation (สำหรับ test)
pub struct InMemoryStore(HashMap<String, String>);

// PostgreSQL implementation (production)
// pub struct PgStore { pool: sqlx::PgPool }
```

### Observability

ใน production ควร expose metrics ผ่าน Prometheus:

```
saga_executions_total{status="completed"} 1523
saga_executions_total{status="compensated"} 47
saga_executions_total{status="failed"} 3
saga_step_duration_seconds{step="ReservePayment",quantile="0.99"} 0.045
saga_compensation_total{step="ReservePayment"} 12
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Async Saga Engine (ระดับกลาง)

ปรับปรุง `SagaOrchestrator` ให้ support async step functions โดยใช้ `tokio`:

```rust
pub struct AsyncSagaStep<S: Send> {
    pub name: String,
    pub action: Box<dyn Fn(&mut S) -> BoxFuture<'_, Result<(), String>> + Send + Sync>,
    pub compensate: Box<dyn Fn(&mut S) -> BoxFuture<'_, Result<(), String>> + Send + Sync>,
    pub retry_policy: RetryPolicy,
}
```

- เพิ่ม actual `tokio::time::sleep` ด้วย delay จาก `RetryPolicy::delay_ms()`
- เขียน test ที่ verify ว่า retry จริงๆ รอ delay ที่ถูกต้อง (ใช้ `tokio::time::pause()`)

### แบบฝึกหัดที่ 2: Persistent Store ด้วย SQLite (ระดับกลาง)

แทนที่ `HashMap` ใน `SagaStore` ด้วย SQLite database:

```sql
CREATE TABLE saga_executions (
    id TEXT PRIMARY KEY,
    saga_name TEXT NOT NULL,
    status TEXT NOT NULL,
    state_json TEXT NOT NULL,
    updated_at INTEGER NOT NULL
);
```

- ใช้ `rusqlite` crate สำหรับ synchronous SQLite access
- เพิ่ม `list_by_status(status: &SagaStatus) -> Vec<SagaExecution>` method
- เขียน integration test ที่สร้าง database จริงใน temp directory

### แบบฝึกหัดที่ 3: Saga Orchestration Server (ระดับสูง)

สร้าง HTTP API สำหรับ manage sagas ด้วย `axum`:

```
POST   /sagas/{definition_name}    — เริ่ม saga ใหม่ พร้อม initial state
GET    /sagas/{id}                 — ดูสถานะ saga
GET    /sagas/{id}/events          — ดู event log ทั้งหมด
POST   /sagas/{id}/compensate      — force trigger compensation (manual override)
GET    /sagas?status=running       — list sagas by status
```

- ใช้ `Arc<Mutex<SagaOrchestrator>>` สำหรับ shared state ใน handlers
- Register saga definitions ล่วงหน้าใน `HashMap<String, SagaDefinitionFn>`

### แบบฝึกหัดที่ 4: Distributed Saga ข้าม Services (ระดับสูง)

ปรับปรุงให้ action ของแต่ละ step ส่ง HTTP request ไปยัง service จริง:

```rust
SagaStep::new(
    "ReservePayment",
    |s: &mut OrderState| async move {
        let resp = reqwest::Client::new()
            .post("http://payment-service/reserve")
            .json(&ReserveRequest { order_id: s.order_id, amount: s.amount })
            .send()
            .await?;

        if resp.status().is_success() {
            s.payment_reservation_id = resp.json::<ReserveResponse>().await?.reservation_id;
            Ok(())
        } else {
            Err(format!("payment service error: {}", resp.status()))
        }
    },
    // compensate: ส่ง DELETE /reserve/{reservation_id}
)
```

- ใช้ `reqwest` สำหรับ HTTP client
- เพิ่ม timeout ต่อ request ใน retry policy
- สร้าง mock services ด้วย `wiremock` ใน integration tests

### แบบฝึกหัดที่ 5: Choreography-based Saga (ระดับขั้นสูง)

เปรียบเทียบ orchestration กับ choreography โดยสร้าง choreography version:

```rust
// แต่ละ service ฟัง event และตัดสินใจเอง
pub trait SagaParticipant {
    fn handle_event(&self, event: &DomainEvent) -> Option<DomainEvent>;
    fn handle_failure(&self, event: &FailureEvent) -> Option<CompensationEvent>;
}

// Event bus สำหรับส่ง events ระหว่าง participants
pub struct SagaEventBus {
    participants: Vec<Box<dyn SagaParticipant>>,
}
```

- วิเคราะห์ข้อดีข้อเสียของแต่ละ approach
- เขียน blog post หรือ README อธิบาย tradeoffs

### แบบฝึกหัดที่ 6: Dead Letter Queue สำหรับ Failed Sagas (ระดับกลาง)

เพิ่ม Dead Letter Queue (DLQ) สำหรับ saga ที่ `status == Failed`:

```rust
pub struct DeadLetterQueue {
    entries: Vec<DlqEntry>,
}

pub struct DlqEntry {
    pub execution_id: String,
    pub saga_name: String,
    pub failed_step: String,
    pub failed_compensations: Vec<String>,
    pub created_at: u64,
    pub manual_action_required: bool,
}

impl SagaOrchestrator {
    pub fn dlq_entries(&self) -> &[DlqEntry] { &self.dlq.entries }
    pub fn resolve_dlq_entry(&mut self, id: &str) -> Option<DlqEntry> { ... }
}
```

- เพิ่ม alert mechanism เมื่อ DLQ มี entry ใหม่
- สร้าง CLI tool สำหรับ admin ดู และ resolve DLQ entries

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **Saga Orchestration Engine** ที่ครบครันสำหรับ production ประกอบด้วย:

**สิ่งที่สร้าง:**
- `SagaStep<S>` — generic step พร้อม action, compensate, และ retry policy
- `SagaDefinition<S>` — builder pattern สำหรับประกาศ saga แบบ type-safe
- `SagaOrchestrator` — engine ที่ execute forward pass และ compensation pass
- `SagaExecution` — runtime state พร้อม `EventLog` สำหรับ audit trail
- `SagaStore` — in-memory persistence store (serialize เป็น JSON)
- `RetryPolicy` — exponential backoff with jitter

**Pattern สำคัญที่ได้เรียน:**
1. **Saga Pattern** — distributed transaction ด้วย compensating transactions แทน 2PC
2. **Generic trait objects** — `Box<dyn Fn(&mut S) -> Result<(), String>>` สำหรับ type-erased callbacks ที่รองรับ state ทุกประเภท
3. **Builder pattern** — fluent API `SagaDefinition::new().add_step(...)`
4. **Event sourcing lite** — append-only EventLog เป็น audit trail
5. **Checkpoint pattern** — serialize state หลังทุก step เพื่อ resume ได้
6. **Exponential backoff** — jitter ป้องกัน thundering herd

**ความแตกต่างจาก Project F04 (Circuit Breaker):**
- Circuit Breaker ป้องกัน failure cascade ใน service-to-service calls
- Saga Pattern จัดการ multi-step transaction ที่ข้าม service boundaries
- ทั้งสองเป็น complementary patterns: circuit breaker ใช้ใน action/compensate ของแต่ละ step

**โปรเจคถัดไป — F06 Event Sourcing** จะ go deeper ใน event log concept:
- แทนที่ mutable state ด้วย immutable event stream
- Rebuild state จาก event history ได้ทุกเวลา
- Temporal queries: "สถานะ ณ เวลา T คืออะไร?"
- CQRS (Command Query Responsibility Segregation) pattern

---

**โปรเจคก่อนหน้า:** [Project F04: Circuit Breaker](project-f04-circuit-breaker.md) | **โปรเจคถัดไป:** [Project F06: Event Sourcing](project-f06-event-sourcing.md)
