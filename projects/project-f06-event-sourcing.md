# Project F06: Event Sourcing Framework

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

ระบบ software ที่ซับซ้อนมักเผชิญกับปัญหาพื้นฐานข้อหนึ่ง: **state ของข้อมูลในปัจจุบันไม่เพียงพอ** เราต้องการรู้ว่า "เกิดอะไรขึ้นบ้าง" ไม่ใช่แค่ "ตอนนี้ค่าเป็นเท่าไหร่" — ธนาคารต้องการ audit trail ของทุก transaction, e-commerce ต้องการรู้ว่า order ผ่าน state ไหนบ้าง, inventory system ต้องการ replay ประวัติการเบิก-จ่ายสินค้า

**Event Sourcing** คือ architectural pattern ที่ตอบโจทย์นี้: แทนที่จะเก็บแค่ "ค่าปัจจุบัน", เราเก็บ **ทุก event ที่เกิดขึ้น** เรียงตามลำดับเวลา และ state ปัจจุบันคือผลของการ replay events ทั้งหมดตั้งแต่ต้น

โปรเจคนี้จะสร้าง **Event Sourcing Framework** ที่สมบูรณ์ประกอบด้วย:

- **EventStore** — interface และ in-memory implementation สำหรับ append/load events พร้อม optimistic locking
- **Aggregate** — domain object ที่ handle commands, emit events, และ rebuild state จาก event stream
- **CommandHandler** — orchestrates การโหลด aggregate, รัน command, และ append events ด้วย concurrency control
- **Projections** — read model ที่ rebuild ได้จาก event stream สำหรับ query ที่ต่างจาก write model (CQRS)
- **Snapshots** — checkpoint mechanism เพื่อเร่ง rebuild performance ไม่ต้อง replay events ทุกตัว
- **EventBus** — in-memory pub/sub ด้วย `tokio::sync::broadcast` สำหรับ real-time event notifications

**Use case จริงในโลก production:**
- **Banking & Fintech**: ทุก debit/credit เป็น event ที่ไม่สามารถลบได้ — audit trail สมบูรณ์
- **E-commerce**: order lifecycle ตั้งแต่ created → paid → shipped → delivered แต่ละ step เป็น event
- **Healthcare**: patient record ทุกการเปลี่ยนแปลงเก็บเป็น event — HIPAA compliance
- **Gaming**: game state reconstruction — replay match เพื่อ debug หรือสร้าง replay feature
- **Supply chain**: ติดตาม inventory movement ทุก event จาก PO → receiving → picking → shipping

Event Sourcing มักใช้คู่กับ **CQRS (Command Query Responsibility Segregation)**: write model ประมวลผล commands และ emit events, ส่วน read model (Projection) subscribe events และสร้าง view ที่ optimize สำหรับ query

## สิ่งที่จะได้เรียนรู้

- **Event Sourcing pattern** — เก็บ state เป็น sequence ของ events แทนค่าปัจจุบัน
- **CQRS** — แยก write model (aggregate) จาก read model (projection) อย่างชัดเจน
- **Aggregate pattern** — encapsulate domain logic, enforce invariants, emit events
- **Optimistic locking** — concurrency control ด้วย expected_version check ไม่ต้อง lock
- **Trait-based design** — `EventStore<T>`, `Aggregate`, `Projection<T>`, `SnapshotStore<S>` generics
- **`serde::de::DeserializeOwned`** — การใช้ `DeserializeOwned` (= `for<'de> Deserialize<'de>`) เพื่อ avoid lifetime conflict
- **tokio broadcast channel** — pub/sub pattern สำหรับ fan-out event notifications
- **Snapshot pattern** — checkpoint เพื่อเร่ง aggregate reconstruction ลด replay time

## ความรู้ที่ต้องมีมาก่อน

- **Part 13-17**: Generic types, traits, trait bounds — ใช้กับ `EventStore<T>`, `Aggregate` trait
- **Part 18-22**: Closures, `Fn` trait — ใช้สำหรับ iterator operations ใน projections
- **Part 23-27**: `Box<dyn Trait>`, dynamic dispatch, `Arc<Mutex<dyn Projection>>` — ใช้ใน `CommandHandler` และ `ProjectionManager`
- **Part 35-40**: Error handling, `Result<T, E>`, custom error types — ใช้ทั่วโปรเจค
- **Part 46-50**: Iterators, `fold`, `filter`, `map` — ใช้ใน aggregate rebuild
- **Part 55-60**: `serde`, `serde_json`, `#[derive(Serialize, Deserialize)]` — serialization ของ events
- **Part 61-65**: `uuid` crate, `chrono` crate — metadata ของ events
- **Part 71-80**: `tokio` async runtime, `broadcast::channel` — EventBus implementation
- **Part 96-100**: `Arc<Mutex<T>>`, thread-safe shared state — InMemoryEventStore

## โครงสร้างโปรเจค (Project Layout)

```
event-sourcing/
├── src/
│   ├── lib.rs          # module root — re-exports ทุก submodule
│   ├── main.rs         # demo binary — BankAccount example
│   ├── events.rs       # Event<T> struct — ข้อมูลหลักของทุก event
│   ├── store.rs        # EventStore trait + InMemoryEventStore
│   ├── aggregate.rs    # Aggregate trait + BankAccount implementation
│   ├── command.rs      # CommandHandler<A> — orchestrates command execution
│   ├── projection.rs   # Projection trait + AccountSummaryProjection + ProjectionManager
│   ├── snapshot.rs     # Snapshot<S>, SnapshotStore, SnapshotPolicy
│   ├── bus.rs          # EventBus<T> — tokio broadcast pub/sub
│   └── tests.rs        # unit tests ทั้งหมด (19 tests)
└── Cargo.toml
```

**Design principle**: แต่ละ module มีความรับผิดชอบชัดเจน ไม่มี circular dependency — `events.rs` เป็น foundational type ที่ทุก module ใช้, `store.rs` ขึ้นอยู่กับ `events.rs` เท่านั้น, `aggregate.rs` ขึ้นอยู่กับ `events.rs` เท่านั้น

## การออกแบบ (Architecture & Design)

### Data Flow — Write Path (Command Side)

```
ผู้ใช้ส่ง Command
        │
        ▼
CommandHandler<A>
   1. store.load(aggregate_id, from_seq=1)
        │ Vec<Event<A::Event>>
        ▼
   2. A::rebuild(id, &events) → Aggregate
        │ state ปัจจุบัน
        ▼
   3. aggregate.handle_command(cmd) → Vec<Event>
        │ new events
        ▼
   4. store.append(id, events, expected_version)
        │ optimistic lock check
        ▼
   5. bus.publish(event) ← notify subscribers
        │
        ▼
   Return: Vec<Event> หรือ CommandError
```

### Data Flow — Read Path (Query Side)

```
Event stream ใน EventStore
        │
        ▼
ProjectionManager::rebuild_from_store(store, aggregate_ids)
        │ load events → dispatch แต่ละ event
        ▼
Projection::handle_event(&event)
        │ update in-memory read model
        ▼
Query read model → คืนข้อมูลสำหรับ UI/API
```

### Snapshot-assisted Rebuild

```
store.load_latest_snapshot(id)  →  Some(Snapshot { version: 50, state })
        │                               │
        │ found snapshot                 │ no snapshot
        ▼                               ▼
store.load(id, from_seq=51)      store.load(id, from_seq=1)
        │                               │
        ▼                               ▼
A::rebuild บน events 51-60      A::rebuild บน events 1-60
(10 events แทน 60 events)
```

### Optimistic Locking — ป้องกัน Concurrent Write

```
Client A                        Client B
   │                               │
   ├─ load(id, 1) → version=5      │
   │                               ├─ load(id, 1) → version=5
   │                               ├─ handle_command → events[seq=6]
   │                               ├─ append(id, events, expected=5) → OK ✓
   │                               │  (version becomes 6)
   ├─ handle_command → events[seq=6]
   ├─ append(id, events, expected=5) → ERROR ✗
   │  OptimisticLockConflict { expected: 5, actual: 6 }
   │
   └─ retry: load again → version=6, re-handle command
```

### Design Decisions

**ทำไมใช้ `serde::de::DeserializeOwned` แทน `for<'de> Deserialize<'de>`?**

ใน Rust เมื่อเราเขียน `struct Event<T: for<'de> Deserialize<'de>>` แล้วพยายาม `#[derive(Deserialize)]` บน struct นั้น compiler จะ complain ว่า lifetime `'de` ถูกประกาศซ้ำ เพราะ derive macro ก็สร้าง `'de` lifetime ของตัวเองด้วย วิธีแก้ที่ถูกต้องคือใช้ `DeserializeOwned` ซึ่งเป็น alias สำหรับ `for<'de> Deserialize<'de>` และไม่ conflict กับ derive macro

**ทำไม aggregate `apply()` return `Self` แทน mutate in-place?**

Pattern นี้เรียกว่า "immutable apply" — แต่ละ event สร้าง state ใหม่แทนที่จะ mutate ตัวเดิม ทำให้ `rebuild()` ทำได้ด้วย `fold()` แบบ functional: `events.iter().fold(initial_state, |state, event| state.apply(event))` ซึ่ง easier to reason about และ testable กว่า

**ทำไม Projection ใช้ `Arc<Mutex<dyn Projection>>`?**

`ProjectionManager` ต้องเก็บ projections หลาย type พร้อมกัน (dynamic dispatch) และ share ระหว่าง threads ได้ `Arc<Mutex<dyn Projection<T>>>` เป็น pattern มาตรฐานสำหรับ `Send + Sync` shared mutable state ใน Rust

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Event struct และ EventStore Trait

เริ่มจาก building block พื้นฐานที่สุด: `Event<T>` struct และ `EventStore<T>` trait ที่เป็น interface หลักของระบบ

**`Cargo.toml`**:
```toml
[package]
name = "event-sourcing"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
uuid = { version = "1.0", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
tokio = { version = "1.0", features = ["full"] }
```

**`src/events.rs`**:
```rust
use chrono::{DateTime, Utc};
use serde::{de::DeserializeOwned, Deserialize, Serialize};
use std::collections::HashMap;
use uuid::Uuid;

/// Event ที่บันทึกใน EventStore — แต่ละ event แทน state change ที่เกิดขึ้น
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(bound = "T: Serialize + DeserializeOwned")]
pub struct Event<T: Clone + Serialize + DeserializeOwned> {
    pub id: Uuid,
    pub aggregate_id: String,
    pub sequence: u64,
    pub event_type: String,
    pub payload: T,
    pub timestamp: DateTime<Utc>,
    pub metadata: HashMap<String, String>,
}

impl<T: Clone + Serialize + DeserializeOwned> Event<T> {
    pub fn new(
        aggregate_id: impl Into<String>,
        sequence: u64,
        event_type: impl Into<String>,
        payload: T,
    ) -> Self {
        Event {
            id: Uuid::new_v4(),
            aggregate_id: aggregate_id.into(),
            sequence,
            event_type: event_type.into(),
            payload,
            timestamp: Utc::now(),
            metadata: HashMap::new(),
        }
    }

    pub fn with_metadata(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.metadata.insert(key.into(), value.into());
        self
    }
}
```

**ประเด็นสำคัญ**: `#[serde(bound = "T: Serialize + DeserializeOwned")]` บอก serde derive ว่าให้ใช้ bound นี้แทน bound ที่ derive สร้างเอง ซึ่งจะ conflict กับ lifetime `'de` ในนิยาม struct

**`src/store.rs`**:
```rust
use crate::events::Event;
use serde::{de::DeserializeOwned, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Debug, Clone, PartialEq)]
pub enum StoreError {
    OptimisticLockConflict { expected: u64, actual: u64 },
    AggregateNotFound(String),
    SequenceError(String),
}

impl std::fmt::Display for StoreError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            StoreError::OptimisticLockConflict { expected, actual } => {
                write!(f, "Optimistic lock conflict: expected version {}, got {}", expected, actual)
            }
            StoreError::AggregateNotFound(id) => write!(f, "Aggregate not found: {}", id),
            StoreError::SequenceError(msg) => write!(f, "Sequence error: {}", msg),
        }
    }
}

/// Trait หลักของ EventStore
pub trait EventStore<T: Clone + Serialize + DeserializeOwned + Send + Sync>: Send + Sync {
    fn append(
        &self,
        aggregate_id: &str,
        events: Vec<Event<T>>,
        expected_version: Option<u64>,
    ) -> Result<(), StoreError>;

    fn load(&self, aggregate_id: &str, from_sequence: u64) -> Vec<Event<T>>;

    fn current_version(&self, aggregate_id: &str) -> Option<u64>;
}
```

เหตุผลที่ `EventStore` trait bound ต้องมี `T: Send + Sync` คือ store ควรใช้ข้ามthread ได้ใน concurrent environment

### ขั้นที่ 2: InMemoryEventStore พร้อม Optimistic Locking

In-memory implementation ใช้ `Arc<Mutex<HashMap<String, Vec<Event<T>>>>>` เป็น backing storage

```rust
pub struct InMemoryEventStore<T: Clone + Serialize + DeserializeOwned> {
    events: Arc<Mutex<HashMap<String, Vec<Event<T>>>>>,
}

impl<T: Clone + Serialize + DeserializeOwned + Send + Sync> EventStore<T>
    for InMemoryEventStore<T>
{
    fn append(
        &self,
        aggregate_id: &str,
        events: Vec<Event<T>>,
        expected_version: Option<u64>,
    ) -> Result<(), StoreError> {
        if events.is_empty() {
            return Ok(());
        }

        let mut store = self.events.lock().unwrap();
        let existing = store
            .entry(aggregate_id.to_string())
            .or_insert_with(Vec::new);

        // Optimistic locking check
        let current_version = existing.last().map(|e| e.sequence);
        if let Some(expected) = expected_version {
            match current_version {
                None if expected == 0 => {}            // ยังไม่มี events, OK
                Some(actual) if actual == expected => {} // version ตรงกัน, OK
                Some(actual) => {
                    return Err(StoreError::OptimisticLockConflict { expected, actual })
                }
                None => {
                    return Err(StoreError::OptimisticLockConflict { expected, actual: 0 })
                }
            }
        }

        // ตรวจสอบ sequence continuity
        let next_seq = current_version.map(|v| v + 1).unwrap_or(1);
        let first_incoming = events[0].sequence;
        if first_incoming != next_seq {
            return Err(StoreError::SequenceError(format!(
                "Expected next sequence {}, got {}",
                next_seq, first_incoming
            )));
        }

        existing.extend(events);
        Ok(())
    }

    fn load(&self, aggregate_id: &str, from_sequence: u64) -> Vec<Event<T>> {
        let store = self.events.lock().unwrap();
        match store.get(aggregate_id) {
            None => vec![],
            Some(events) => events
                .iter()
                .filter(|e| e.sequence >= from_sequence)
                .cloned()
                .collect(),
        }
    }

    fn current_version(&self, aggregate_id: &str) -> Option<u64> {
        let store = self.events.lock().unwrap();
        store
            .get(aggregate_id)
            .and_then(|events| events.last().map(|e| e.sequence))
    }
}
```

**Optimistic locking logic**:
- ถ้า `expected_version = None` → ไม่ตรวจสอบ (ใช้ตอน first append หรือ unguarded write)
- ถ้า `expected_version = Some(0)` และ aggregate ยังไม่มี events → OK
- ถ้า `expected_version = Some(v)` และ current version = `v` → OK
- กรณีอื่น → return `OptimisticLockConflict` error

### ขั้นที่ 3: Aggregate Trait และ BankAccount

`Aggregate` trait กำหนด interface ที่ domain object ทุกตัวต้อง implement

```rust
use crate::events::Event;
use serde::{de::DeserializeOwned, Deserialize, Serialize};

pub trait Aggregate: Sized + Default + Clone + Send + Sync {
    type Event: Clone + Serialize + DeserializeOwned + Send + Sync;
    type Command: Send + Sync;
    type Error: std::fmt::Debug + std::fmt::Display + PartialEq;

    fn id(&self) -> &str;
    fn version(&self) -> u64;

    /// Apply event และ return state ใหม่ (immutable style)
    fn apply(self, event: &Event<Self::Event>) -> Self;

    /// Handle command → คืน events ที่ต้องการ append
    fn handle_command(&self, cmd: Self::Command) -> Result<Vec<Event<Self::Event>>, Self::Error>;

    /// Rebuild aggregate จาก sequence of events
    fn rebuild(id: &str, events: &[Event<Self::Event>]) -> Self {
        events
            .iter()
            .fold(Self::default(), |agg, event| agg.apply(event))
    }
}
```

**BankAccount Aggregate** — domain model สำหรับ bank account ที่มี open/deposit/withdraw/close operations:

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum BankAccountEvent {
    AccountOpened { owner: String, initial_balance: f64 },
    MoneyDeposited { amount: f64 },
    MoneyWithdrawn { amount: f64 },
    AccountClosed,
}

#[derive(Debug, Clone)]
pub enum BankAccountCommand {
    OpenAccount { owner: String, initial_balance: f64 },
    Deposit { amount: f64 },
    Withdraw { amount: f64 },
    CloseAccount,
}

#[derive(Debug, Clone, PartialEq)]
pub enum BankAccountError {
    AlreadyOpen,
    NotOpen,
    InsufficientFunds { balance: f64, requested: f64 },
    InvalidAmount,
}

#[derive(Debug, Default, Clone, Serialize, Deserialize)]
pub struct BankAccount {
    pub id: String,
    pub owner: String,
    pub balance: f64,
    pub is_open: bool,
    pub version: u64,
}

impl Aggregate for BankAccount {
    type Event = BankAccountEvent;
    type Command = BankAccountCommand;
    type Error = BankAccountError;

    fn id(&self) -> &str { &self.id }
    fn version(&self) -> u64 { self.version }

    fn apply(mut self, event: &Event<Self::Event>) -> Self {
        match &event.payload {
            BankAccountEvent::AccountOpened { owner, initial_balance } => {
                self.owner = owner.clone();
                self.balance = *initial_balance;
                self.is_open = true;
            }
            BankAccountEvent::MoneyDeposited { amount } => {
                self.balance += amount;
            }
            BankAccountEvent::MoneyWithdrawn { amount } => {
                self.balance -= amount;
            }
            BankAccountEvent::AccountClosed => {
                self.is_open = false;
            }
        }
        self.id = event.aggregate_id.clone();
        self.version = event.sequence;
        self
    }

    fn handle_command(&self, cmd: Self::Command)
        -> Result<Vec<Event<Self::Event>>, Self::Error>
    {
        let next_seq = self.version + 1;
        match cmd {
            BankAccountCommand::OpenAccount { owner, initial_balance } => {
                if self.is_open {
                    return Err(BankAccountError::AlreadyOpen);
                }
                if initial_balance < 0.0 {
                    return Err(BankAccountError::InvalidAmount);
                }
                Ok(vec![Event::new(
                    &self.id, next_seq, "AccountOpened",
                    BankAccountEvent::AccountOpened { owner, initial_balance },
                )])
            }
            BankAccountCommand::Deposit { amount } => {
                if !self.is_open { return Err(BankAccountError::NotOpen); }
                if amount <= 0.0 { return Err(BankAccountError::InvalidAmount); }
                Ok(vec![Event::new(
                    &self.id, next_seq, "MoneyDeposited",
                    BankAccountEvent::MoneyDeposited { amount },
                )])
            }
            BankAccountCommand::Withdraw { amount } => {
                if !self.is_open { return Err(BankAccountError::NotOpen); }
                if amount <= 0.0 { return Err(BankAccountError::InvalidAmount); }
                if self.balance < amount {
                    return Err(BankAccountError::InsufficientFunds {
                        balance: self.balance,
                        requested: amount,
                    });
                }
                Ok(vec![Event::new(
                    &self.id, next_seq, "MoneyWithdrawn",
                    BankAccountEvent::MoneyWithdrawn { amount },
                )])
            }
            BankAccountCommand::CloseAccount => {
                if !self.is_open { return Err(BankAccountError::NotOpen); }
                Ok(vec![Event::new(
                    &self.id, next_seq, "AccountClosed",
                    BankAccountEvent::AccountClosed,
                )])
            }
        }
    }

    fn rebuild(id: &str, events: &[Event<Self::Event>]) -> Self {
        let mut account = BankAccount { id: id.to_string(), ..Default::default() };
        for event in events {
            account = account.apply(event);
        }
        account
    }
}
```

**Invariants ที่ enforce ใน `handle_command`**:
- ไม่สามารถ open account ที่ open แล้ว
- ไม่สามารถ deposit/withdraw บัญชีที่ปิดแล้ว
- ไม่สามารถ withdraw เกิน balance
- amount ต้องเป็นบวก

### ขั้นที่ 4: CommandHandler พร้อม Concurrency Control

`CommandHandler<A>` orchestrates flow ทั้งหมด: load → rebuild → command → append

```rust
use crate::aggregate::Aggregate;
use crate::events::Event;
use crate::store::{EventStore, StoreError};
use serde::{de::DeserializeOwned, Serialize};

#[derive(Debug)]
pub enum CommandError<E: std::fmt::Debug + std::fmt::Display> {
    DomainError(E),
    StoreError(StoreError),
}

impl<E: std::fmt::Debug + std::fmt::Display> From<StoreError> for CommandError<E> {
    fn from(e: StoreError) -> Self {
        CommandError::StoreError(e)
    }
}

pub struct CommandHandler<A: Aggregate> {
    store: Box<dyn EventStore<A::Event>>,
}

impl<A: Aggregate + 'static> CommandHandler<A>
where
    A::Event: Clone + Serialize + DeserializeOwned + Send + Sync + 'static,
{
    pub fn new(store: impl EventStore<A::Event> + 'static) -> Self {
        CommandHandler { store: Box::new(store) }
    }

    pub fn execute(
        &self,
        aggregate_id: &str,
        command: A::Command,
    ) -> Result<Vec<Event<A::Event>>, CommandError<A::Error>> {
        // โหลด event history ทั้งหมด
        let events = self.store.load(aggregate_id, 1);
        let aggregate = A::rebuild(aggregate_id, &events);

        // กำหนด expected_version สำหรับ optimistic lock
        let expected_version = if events.is_empty() {
            None
        } else {
            Some(aggregate.version())
        };

        // รัน command → ได้ new events
        let new_events = aggregate
            .handle_command(command)
            .map_err(CommandError::DomainError)?;

        if new_events.is_empty() {
            return Ok(vec![]);
        }

        // Append พร้อม optimistic lock check
        self.store
            .append(aggregate_id, new_events.clone(), expected_version)
            .map_err(CommandError::StoreError)?;

        Ok(new_events)
    }

    /// Execute พร้อม explicit expected_version สำหรับ strict concurrency control
    pub fn execute_with_version(
        &self,
        aggregate_id: &str,
        command: A::Command,
        expected_version: Option<u64>,
    ) -> Result<Vec<Event<A::Event>>, CommandError<A::Error>> {
        let events = self.store.load(aggregate_id, 1);
        let aggregate = A::rebuild(aggregate_id, &events);

        let new_events = aggregate
            .handle_command(command)
            .map_err(CommandError::DomainError)?;

        if new_events.is_empty() {
            return Ok(vec![]);
        }

        self.store
            .append(aggregate_id, new_events.clone(), expected_version)
            .map_err(CommandError::StoreError)?;

        Ok(new_events)
    }
}
```

**ตัวอย่างการใช้งาน `CommandHandler`**:

```rust
use event_sourcing::aggregate::{BankAccount, BankAccountCommand};
use event_sourcing::command::CommandHandler;
use event_sourcing::store::InMemoryEventStore;

let store = InMemoryEventStore::new();
let handler = CommandHandler::<BankAccount>::new(store);

// Open account
handler.execute("ACC-001", BankAccountCommand::OpenAccount {
    owner: "Alice".to_string(),
    initial_balance: 1000.0,
}).unwrap();

// Deposit
handler.execute("ACC-001", BankAccountCommand::Deposit { amount: 500.0 }).unwrap();

// Withdraw
handler.execute("ACC-001", BankAccountCommand::Withdraw { amount: 200.0 }).unwrap();
```

### ขั้นที่ 5: Projections และ CQRS Read Model

Projection คือ read model ที่ subscribe events และ maintain view ที่ optimize สำหรับ query — แยกจาก aggregate (write model) อย่างสมบูรณ์

```rust
use crate::events::Event;
use crate::store::EventStore;
use serde::{de::DeserializeOwned, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

/// Trait สำหรับ Projection
pub trait Projection<T: Clone + Serialize + DeserializeOwned>: Send + Sync {
    fn name(&self) -> &str;
    fn handle_event(&mut self, event: &Event<T>);
    fn reset(&mut self);
}

/// Read model สำหรับ BankAccount
#[derive(Debug, Default, Clone)]
pub struct AccountSummaryProjection {
    pub accounts: HashMap<String, AccountSummary>,
}

#[derive(Debug, Clone)]
pub struct AccountSummary {
    pub id: String,
    pub owner: String,
    pub balance: f64,
    pub is_open: bool,
    pub transaction_count: u64,
}

impl Projection<BankAccountEvent> for AccountSummaryProjection {
    fn name(&self) -> &str { "AccountSummary" }

    fn handle_event(&mut self, event: &Event<BankAccountEvent>) {
        let id = event.aggregate_id.clone();
        match &event.payload {
            BankAccountEvent::AccountOpened { owner, initial_balance } => {
                self.accounts.insert(id.clone(), AccountSummary {
                    id, owner: owner.clone(), balance: *initial_balance,
                    is_open: true, transaction_count: 0,
                });
            }
            BankAccountEvent::MoneyDeposited { amount } => {
                if let Some(a) = self.accounts.get_mut(&id) {
                    a.balance += amount;
                    a.transaction_count += 1;
                }
            }
            BankAccountEvent::MoneyWithdrawn { amount } => {
                if let Some(a) = self.accounts.get_mut(&id) {
                    a.balance -= amount;
                    a.transaction_count += 1;
                }
            }
            BankAccountEvent::AccountClosed => {
                if let Some(a) = self.accounts.get_mut(&id) {
                    a.is_open = false;
                }
            }
        }
    }

    fn reset(&mut self) { self.accounts.clear(); }
}
```

**`ProjectionManager`** รองรับหลาย projections พร้อมกันและ rebuild ทั้งหมดจาก store:

```rust
pub struct ProjectionManager<T: Clone + Serialize + DeserializeOwned + Send + Sync> {
    projections: Vec<Arc<Mutex<dyn Projection<T>>>>,
}

impl<T: Clone + Serialize + DeserializeOwned + Send + Sync + 'static>
    ProjectionManager<T>
{
    pub fn new() -> Self {
        ProjectionManager { projections: vec![] }
    }

    pub fn register(&mut self, projection: impl Projection<T> + 'static) {
        self.projections.push(Arc::new(Mutex::new(projection)));
    }

    /// Rebuild ทุก projection จาก store (ใช้ตอน startup หรือ replay)
    pub fn rebuild_from_store(
        &self,
        store: &dyn EventStore<T>,
        aggregate_ids: &[&str],
    ) {
        for proj in &self.projections {
            proj.lock().unwrap().reset();
        }
        for id in aggregate_ids {
            let events = store.load(id, 1);
            for event in &events {
                for proj in &self.projections {
                    proj.lock().unwrap().handle_event(event);
                }
            }
        }
    }

    /// Dispatch event ใหม่ไปยังทุก projection (live update)
    pub fn dispatch(&self, event: &Event<T>) {
        for proj in &self.projections {
            proj.lock().unwrap().handle_event(event);
        }
    }
}
```

### ขั้นที่ 6: Snapshot System

Snapshot แก้ปัญหา aggregate ที่มี event history ยาวมาก — แทนที่จะ replay 10,000 events ทุกครั้ง เราเก็บ checkpoint ที่ version 1000, 2000, ... แล้ว replay เฉพาะ events หลัง checkpoint ล่าสุด

```rust
use chrono::{DateTime, Utc};
use serde::{de::DeserializeOwned, Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(bound = "S: Serialize + DeserializeOwned")]
pub struct Snapshot<S: Clone + Serialize + DeserializeOwned> {
    pub aggregate_id: String,
    pub version: u64,
    pub state: S,
    pub created_at: DateTime<Utc>,
}

impl<S: Clone + Serialize + DeserializeOwned> Snapshot<S> {
    pub fn new(aggregate_id: impl Into<String>, version: u64, state: S) -> Self {
        Snapshot {
            aggregate_id: aggregate_id.into(),
            version,
            state,
            created_at: Utc::now(),
        }
    }
}

pub trait SnapshotStore<S: Clone + Serialize + DeserializeOwned>: Send + Sync {
    fn save(&self, snapshot: Snapshot<S>);
    fn load_latest(&self, aggregate_id: &str) -> Option<Snapshot<S>>;
}

pub struct InMemorySnapshotStore<S: Clone + Serialize + DeserializeOwned> {
    snapshots: Arc<Mutex<HashMap<String, Snapshot<S>>>>,
}

impl<S: Clone + Serialize + DeserializeOwned + Send + Sync> SnapshotStore<S>
    for InMemorySnapshotStore<S>
{
    fn save(&self, snapshot: Snapshot<S>) {
        let mut store = self.snapshots.lock().unwrap();
        let entry = store.get(&snapshot.aggregate_id).map(|s| s.version);
        // บันทึกเฉพาะถ้า version ใหม่กว่า snapshot ที่มีอยู่
        if entry.map(|v| snapshot.version > v).unwrap_or(true) {
            store.insert(snapshot.aggregate_id.clone(), snapshot);
        }
    }

    fn load_latest(&self, aggregate_id: &str) -> Option<Snapshot<S>> {
        let store = self.snapshots.lock().unwrap();
        store.get(aggregate_id).cloned()
    }
}

/// กำหนด policy ว่าควร snapshot ที่ version ไหน
pub struct SnapshotPolicy {
    pub every_n_events: u64,
}

impl SnapshotPolicy {
    pub fn new(every_n_events: u64) -> Self {
        SnapshotPolicy { every_n_events }
    }

    pub fn should_snapshot(&self, version: u64) -> bool {
        version > 0 && version % self.every_n_events == 0
    }
}
```

**Pattern การใช้งาน Snapshot**:

```rust
// โหลด aggregate ด้วย snapshot-assisted rebuild
fn load_with_snapshot(
    id: &str,
    store: &InMemoryEventStore<BankAccountEvent>,
    snap_store: &InMemorySnapshotStore<BankAccount>,
) -> BankAccount {
    if let Some(snapshot) = snap_store.load_latest(id) {
        // มี snapshot → load events หลัง snapshot เท่านั้น
        let events = store.load(id, snapshot.version + 1);
        events.iter().fold(snapshot.state, |agg, e| agg.apply(e))
    } else {
        // ไม่มี snapshot → rebuild จาก events ทั้งหมด
        let events = store.load(id, 1);
        BankAccount::rebuild(id, &events)
    }
}
```

### ขั้นที่ 7: EventBus ด้วย tokio Broadcast Channel

EventBus ทำให้ components อื่น ๆ subscribe events แบบ real-time โดยไม่ต้อง poll store

```rust
use crate::events::Event;
use serde::{de::DeserializeOwned, Serialize};
use std::sync::Arc;
use tokio::sync::broadcast;

/// EventBus — in-memory pub/sub
pub struct EventBus<T: Clone + Serialize + DeserializeOwned + Send + Sync + 'static> {
    sender: Arc<broadcast::Sender<Event<T>>>,
}

impl<T: Clone + Serialize + DeserializeOwned + Send + Sync + 'static> EventBus<T> {
    pub fn new(capacity: usize) -> Self {
        let (sender, _) = broadcast::channel(capacity);
        EventBus { sender: Arc::new(sender) }
    }

    pub fn subscribe(&self) -> broadcast::Receiver<Event<T>> {
        self.sender.subscribe()
    }

    pub fn publish(&self, event: Event<T>)
        -> Result<usize, broadcast::error::SendError<Event<T>>>
    {
        self.sender.send(event)
    }

    pub fn receiver_count(&self) -> usize {
        self.sender.receiver_count()
    }
}

impl<T: Clone + Serialize + DeserializeOwned + Send + Sync + 'static> Clone for EventBus<T> {
    fn clone(&self) -> Self {
        EventBus { sender: self.sender.clone() }
    }
}
```

`tokio::sync::broadcast` เป็น MPMC (Multi-Producer Multi-Consumer) channel:
- **Sender**: share ได้ผ่าน `Arc` — หลาย producers publish ได้
- **Receiver**: แต่ละ subscriber ได้รับ copy ของทุก message
- **Capacity**: ถ้า receiver ไม่ดึง message ทัน และ buffer เต็ม → `RecvError::Lagged`

**ตัวอย่าง async subscriber**:

```rust
#[tokio::main]
async fn main() {
    let bus = EventBus::<BankAccountEvent>::new(128);

    // Spawn projection updater task
    let bus2 = bus.clone();
    tokio::spawn(async move {
        let mut rx = bus2.subscribe();
        loop {
            match rx.recv().await {
                Ok(event) => {
                    println!("Got event: {} on {}", event.event_type, event.aggregate_id);
                    // update projection...
                }
                Err(broadcast::error::RecvError::Lagged(n)) => {
                    eprintln!("Missed {} events, re-syncing...", n);
                    // re-rebuild projection from store...
                }
                Err(broadcast::error::RecvError::Closed) => break,
            }
        }
    });

    // Publish events
    let event = Event::new("ACC-001", 3, "MoneyWithdrawn",
        BankAccountEvent::MoneyWithdrawn { amount: 200.0 });
    bus.publish(event).unwrap();
}
```

### ขั้นที่ 8: Demo Binary — Bank Account Workflow ครบวงจร

**`src/main.rs`** รวมทุก component เข้าด้วยกัน:

```rust
use event_sourcing::aggregate::{Aggregate, BankAccount, BankAccountCommand, BankAccountEvent};
use event_sourcing::bus::EventBus;
use event_sourcing::projection::{AccountSummaryProjection, Projection};
use event_sourcing::snapshot::{InMemorySnapshotStore, Snapshot, SnapshotPolicy, SnapshotStore};
use event_sourcing::store::{EventStore, InMemoryEventStore};

#[tokio::main]
async fn main() {
    println!("=== Event Sourcing Framework Demo ===\n");

    let store = InMemoryEventStore::<BankAccountEvent>::new();
    let account_id = "ACC-001";
    let mut account = BankAccount { id: account_id.to_string(), ..Default::default() };

    // Open account
    let e1 = account.handle_command(BankAccountCommand::OpenAccount {
        owner: "Alice".to_string(),
        initial_balance: 1000.0,
    }).unwrap();
    store.append(account_id, e1.clone(), None).unwrap();
    account = BankAccount::rebuild(account_id, &store.load(account_id, 1));

    // Deposit
    let e2 = account.handle_command(BankAccountCommand::Deposit { amount: 500.0 }).unwrap();
    store.append(account_id, e2.clone(), Some(account.version())).unwrap();
    account = BankAccount::rebuild(account_id, &store.load(account_id, 1));

    // Withdraw
    let e3 = account.handle_command(BankAccountCommand::Withdraw { amount: 200.0 }).unwrap();
    store.append(account_id, e3.clone(), Some(account.version())).unwrap();
    account = BankAccount::rebuild(account_id, &store.load(account_id, 1));

    println!("Balance after transactions: {:.2}", account.balance);
    println!("Account version: {}", account.version());

    // Rebuild จาก events ทั้งหมด
    let all_events = store.load(account_id, 1);
    let rebuilt = BankAccount::rebuild(account_id, &all_events);
    println!("Rebuilt balance: {:.2}", rebuilt.balance);

    // Projection
    let mut projection = AccountSummaryProjection::default();
    for event in &all_events {
        projection.handle_event(event);
    }
    let summary = projection.accounts.get(account_id).unwrap();
    println!("Projection — owner: {}, balance: {:.2}, txns: {}",
        summary.owner, summary.balance, summary.transaction_count);

    // Snapshot
    let snap_store = InMemorySnapshotStore::<BankAccount>::new();
    let policy = SnapshotPolicy::new(3);
    if policy.should_snapshot(rebuilt.version()) {
        snap_store.save(Snapshot::new(account_id, rebuilt.version(), rebuilt.clone()));
        println!("Snapshot saved at version {}", rebuilt.version());
    }

    // EventBus
    let bus = EventBus::<BankAccountEvent>::new(16);
    let mut rx = bus.subscribe();
    let final_event = all_events.last().unwrap().clone();
    bus.publish(final_event).ok();
    if let Ok(received) = rx.try_recv() {
        println!("EventBus received: {}", received.event_type);
    }

    println!("\nDemo complete.");
}
```

**Output จากการรัน `cargo run`**:

```
=== Event Sourcing Framework Demo ===

Balance after transactions: 1300.00
Account version: 3
Rebuilt balance: 1300.00
Projection — owner: Alice, balance: 1300.00, txns: 2
Snapshot saved at version 3
EventBus received: MoneyWithdrawn

Demo complete.
```

## การทดสอบ (Testing)

โปรเจคมี 19 unit tests ครอบคลุมทุก component ใน `src/tests.rs` ทดสอบทั้ง happy path, error cases, และ integration flow

### รายการ Tests ทั้งหมด

**EventStore Tests (5 tests)**:

| Test | สิ่งที่ทดสอบ |
|------|------------|
| `test_append_and_load_events` | append event แล้ว load กลับมาได้ถูกต้อง |
| `test_load_empty_aggregate` | load aggregate ที่ไม่มี events คืน empty vec |
| `test_sequence_ordering` | events เรียงตาม sequence ถูกต้อง |
| `test_load_from_sequence` | load จาก sequence ที่กำหนดได้ |
| `test_optimistic_lock_conflict` | append ด้วย expected_version ผิด → error |
| `test_current_version` | current_version คืนค่าถูกต้อง |

**Aggregate Tests (4 tests)**:

| Test | สิ่งที่ทดสอบ |
|------|------------|
| `test_aggregate_apply_open` | apply AccountOpened event ได้ state ถูกต้อง |
| `test_aggregate_apply_sequence` | apply events ต่อเนื่อง balance คำนวณถูก |
| `test_insufficient_funds_error` | withdraw เกิน balance → InsufficientFunds |
| `test_command_on_closed_account` | command บัญชีปิด → NotOpen |
| `test_aggregate_rebuild` | rebuild จาก events ได้ state เหมือนเดิม |

**Projection Tests (2 tests)**:

| Test | สิ่งที่ทดสอบ |
|------|------------|
| `test_projection_rebuild` | rebuild projection จาก event stream |
| `test_projection_reset` | reset projection ล้างข้อมูลทั้งหมด |

**Snapshot Tests (3 tests)**:

| Test | สิ่งที่ทดสอบ |
|------|------------|
| `test_snapshot_save_and_load` | save snapshot แล้ว load กลับมาถูกต้อง |
| `test_snapshot_policy` | SnapshotPolicy ระบุว่าควร snapshot ที่ version ไหน |
| `test_snapshot_only_keeps_latest` | บันทึก snapshot ใหม่กว่าทับของเก่า |

**EventBus Tests (2 tests, async)**:

| Test | สิ่งที่ทดสอบ |
|------|------------|
| `test_event_bus_publish_subscribe` | publish event → subscriber ได้รับ |
| `test_event_bus_multiple_subscribers` | หลาย subscribers รับ event พร้อมกัน |

**Integration Test (1 test)**:

| Test | สิ่งที่ทดสอบ |
|------|------------|
| `test_full_flow_command_event_projection` | command → store → projection ครบวงจร |

### Real `cargo test` Output

```
running 19 tests
test tests::tests::test_aggregate_apply_open ... ok
test tests::tests::test_aggregate_apply_sequence ... ok
test tests::tests::test_command_on_closed_account ... ok
test tests::tests::test_aggregate_rebuild ... ok
test tests::tests::test_append_and_load_events ... ok
test tests::tests::test_current_version ... ok
test tests::tests::test_full_flow_command_event_projection ... ok
test tests::tests::test_insufficient_funds_error ... ok
test tests::tests::test_event_bus_multiple_subscribers ... ok
test tests::tests::test_load_empty_aggregate ... ok
test tests::tests::test_event_bus_publish_subscribe ... ok
test tests::tests::test_load_from_sequence ... ok
test tests::tests::test_projection_reset ... ok
test tests::tests::test_optimistic_lock_conflict ... ok
test tests::tests::test_sequence_ordering ... ok
test tests::tests::test_snapshot_only_keeps_latest ... ok
test tests::tests::test_projection_rebuild ... ok
test tests::tests::test_snapshot_policy ... ok
test tests::tests::test_snapshot_save_and_load ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Pitfalls และปัญหาที่พบบ่อย

### Pitfall 1: `for<'de> Deserialize<'de>` ใน Struct Definition

**ปัญหา**: เขียน `struct Event<T: for<'de> Deserialize<'de>>` แล้วใส่ `#[derive(Deserialize)]` บน struct จะได้ error:

```
error[E0496]: lifetime name `'de` shadows a lifetime name that is already in scope
```

เพราะ `#[derive(Deserialize)]` สร้าง impl block ที่มี `'de` lifetime ของตัวเอง ซึ่ง conflict กับ `'de` ใน struct definition

**วิธีแก้**: ใช้ `serde::de::DeserializeOwned` แทน ซึ่งเป็น alias ของ `for<'de> Deserialize<'de>` และไม่ conflict:

```rust
// ผิด
struct Event<T: for<'de> Deserialize<'de>> { ... }

// ถูก
use serde::de::DeserializeOwned;
#[derive(Deserialize)]
#[serde(bound = "T: Serialize + DeserializeOwned")]
struct Event<T: Clone + Serialize + DeserializeOwned> { ... }
```

`#[serde(bound = "...")]` บอก serde ว่าให้ใช้ bound ที่กำหนดแทน bound ที่ derive สร้างเอง (ซึ่งอาจ overly restrictive หรือ conflict)

---

### Pitfall 2: UUID ต้องการ Feature Flag `serde`

**ปัญหา**: ใส่ `uuid` ใน dependencies แต่ `Uuid` ไม่ implement `Serialize`/`Deserialize` ทำให้ struct ที่มี `Uuid` field ไม่สามารถ derive Serialize/Deserialize ได้:

```
error[E0277]: the trait bound `Uuid: serde::Deserialize<'de>` is not satisfied
```

**วิธีแก้**: เปิด feature flag `serde` ใน Cargo.toml:

```toml
# ผิด
uuid = { version = "1.0", features = ["v4"] }

# ถูก
uuid = { version = "1.0", features = ["v4", "serde"] }
```

Pattern นี้พบบ่อยมากใน Rust ecosystem — crates หลายตัว (chrono, uuid, url, bytes) ต้องเปิด feature flag สำหรับ serde support ก่อน

---

### Pitfall 3: Sequence Gap ทำให้ append ล้มเหลว

**ปัญหา**: สร้าง events ด้วย sequence เริ่มต้นที่ 1 แต่ aggregate ที่ load มามี version = 2 แล้ว ทำให้ events ใหม่มี sequence = 1 ซึ่ง gap กับ current version

```rust
// BUG: สร้าง event ด้วย next_seq = version + 1
// แต่ถ้า version ยังไม่ได้ update หลัง apply events ก่อนหน้า
let next_seq = self.version + 1;  // self.version อาจยังเป็น 0 ถ้าไม่ได้ rebuild
```

**วิธีแก้**: ทุกครั้งที่ต้องการ handle command ต้อง rebuild aggregate จาก store ก่อนเสมอ ไม่ใช้ aggregate ที่ค้างอยู่ในหน่วยความจำโดยไม่ตรวจสอบ version:

```rust
// ถูก — ทุกครั้ง rebuild จาก store
let events = store.load(id, 1);
let current_account = BankAccount::rebuild(id, &events);
let new_events = current_account.handle_command(cmd)?;
store.append(id, new_events, Some(current_account.version()))?;
```

---

### Pitfall 4: Broadcast Channel Lagged Error

**ปัญหา**: ถ้า subscriber ดึง events ไม่ทัน และ channel buffer เต็ม (capacity ที่กำหนดตอน `EventBus::new(capacity)`) subscriber จะได้รับ `RecvError::Lagged(n)` แทนที่จะเป็น event จริง:

```rust
// ถ้าสร้าง bus ด้วย capacity = 16 แล้วมี 20 events publish ก่อน subscriber ดึง
let bus = EventBus::new(16);  // buffer เล็กเกินไป
// subscriber จะ miss 4 events แรกและได้รับ Lagged(4) error
```

**วิธีแก้**:
1. กำหนด capacity ให้ใหญ่พอสำหรับ expected throughput
2. Handle `RecvError::Lagged` โดย re-sync projection จาก store:

```rust
loop {
    match rx.recv().await {
        Ok(event) => projection.handle_event(&event),
        Err(RecvError::Lagged(missed)) => {
            eprintln!("Missed {} events, rebuilding projection...", missed);
            // rebuild from store ใหม่ทั้งหมด
            projection.reset();
            for event in store.load(id, 1) {
                projection.handle_event(&event);
            }
        }
        Err(RecvError::Closed) => break,
    }
}
```

---

### Pitfall 5: Aggregate Snapshot State ต้อง Serialize ได้

**ปัญหา**: ต้องการใช้ `InMemorySnapshotStore<BankAccount>` แต่ `BankAccount` ไม่ implement `Serialize`/`Deserialize` จะได้ error ตอน compile

**วิธีแก้**: Aggregate struct ต้อง derive Serialize และ Deserialize ด้วย — ไม่ใช่แค่ Event type:

```rust
// ต้องมี Serialize + Deserialize ด้วย
#[derive(Debug, Default, Clone, Serialize, Deserialize)]
pub struct BankAccount {
    pub id: String,
    pub owner: String,
    pub balance: f64,
    pub is_open: bool,
    pub version: u64,
}
```

---

### Pitfall 6: Event Ordering กับ Multi-Aggregate Projection

**ปัญหา**: เมื่อ `ProjectionManager::rebuild_from_store` รับ events จาก aggregates หลายตัว events จะเรียงตาม aggregate ID ไม่ใช่ตาม global timestamp — ถ้า projection ต้องการ global ordering จะได้ผลผิด

**วิธีแก้**: ถ้า projection ต้องการ global ordering ควร:
1. เพิ่ม global sequence number ให้ EventStore (ต่างจาก aggregate-local sequence)
2. หรือ store events ทั้งหมดใน single ordered list แทน per-aggregate map
3. หรือ sort events ด้วย timestamp ก่อน dispatch (แต่ timestamp precision อาจไม่เพียงพอ)

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/event-sourcing
```

### ใช้เป็น Library Crate

เพิ่มใน Cargo.toml ของโปรเจคอื่น:

```toml
[dependencies]
event-sourcing = { path = "../event-sourcing" }
```

แล้ว expose public API ผ่าน `lib.rs`:

```rust
pub use aggregate::{Aggregate, BankAccount, BankAccountCommand, BankAccountEvent};
pub use bus::EventBus;
pub use command::{CommandError, CommandHandler};
pub use events::Event;
pub use projection::{AccountSummaryProjection, Projection, ProjectionManager};
pub use snapshot::{InMemorySnapshotStore, Snapshot, SnapshotPolicy, SnapshotStore};
pub use store::{EventStore, InMemoryEventStore, StoreError};
```

### Migration ไป PostgreSQL EventStore

โปรเจคนี้ใช้ in-memory store แต่ design ให้ swap ได้ง่าย เพราะ `EventStore<T>` เป็น trait สร้าง `PostgresEventStore` ที่ implement trait เดียวกัน:

```rust
pub struct PostgresEventStore {
    pool: sqlx::PgPool,
}

impl<T: Clone + Serialize + DeserializeOwned + Send + Sync> EventStore<T>
    for PostgresEventStore
{
    fn append(&self, id: &str, events: Vec<Event<T>>, expected: Option<u64>)
        -> Result<(), StoreError>
    {
        // INSERT INTO events (aggregate_id, sequence, event_type, payload, ...)
        // WITH optimistic lock check in transaction
        todo!()
    }

    fn load(&self, id: &str, from_sequence: u64) -> Vec<Event<T>> {
        // SELECT * FROM events WHERE aggregate_id = $1 AND sequence >= $2
        // ORDER BY sequence ASC
        todo!()
    }

    fn current_version(&self, id: &str) -> Option<u64> {
        // SELECT MAX(sequence) FROM events WHERE aggregate_id = $1
        todo!()
    }
}
```

SQL schema สำหรับ events table:

```sql
CREATE TABLE events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id TEXT NOT NULL,
    sequence    BIGINT NOT NULL,
    event_type  TEXT NOT NULL,
    payload     JSONB NOT NULL,
    timestamp   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata    JSONB NOT NULL DEFAULT '{}',
    UNIQUE (aggregate_id, sequence)
);

CREATE INDEX idx_events_aggregate_seq ON events (aggregate_id, sequence);
```

`UNIQUE (aggregate_id, sequence)` ทำหน้าที่ optimistic locking ระดับ database — ถ้า 2 transactions พยายาม insert sequence เดียวกัน database จะ reject อีกอันด้วย unique constraint violation

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม `PostgresEventStore` (ปานกลาง)

Implement `EventStore<T>` สำหรับ PostgreSQL โดยใช้ `sqlx`:

```toml
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio", "json", "uuid", "chrono"] }
```

สิ่งที่ต้องทำ:
- สร้าง `events` table ตาม schema ข้างต้น
- Implement `append` ด้วย transaction + `ON CONFLICT DO NOTHING` + check rows affected
- Implement `load` ด้วย SELECT + deserialize payload JSONB
- เขียน integration tests ที่ใช้ real PostgreSQL (ใช้ testcontainers หรือ Docker)

---

### แบบฝึกหัดที่ 2: Event Upcasting และ Schema Evolution (ยาก)

ในระบบ production events ถูกเก็บตลอดไป แต่ domain model เปลี่ยนได้ เช่น เพิ่ม field ใหม่ใน `MoneyDeposited`:

```rust
// Version 1
MoneyDeposited { amount: f64 }

// Version 2 — เพิ่ม description
MoneyDeposited { amount: f64, description: String }
```

สร้าง **Upcaster** system:
- Define `EventUpcaster` trait ที่แปลง old event format → new format
- `UpcasterRegistry` ที่เก็บ upcasters ตาม event_type และ version
- Apply upcasters ตอน load events จาก store ก่อน dispatch ไปยัง aggregate

---

### แบบฝึกหัดที่ 3: Process Manager / Saga Integration (ยาก)

สร้าง `ProcessManager` ที่ฟัง events และ emit commands ไปยัง aggregates อื่น — เชื่อม Event Sourcing กับ Saga Pattern จาก Project F05:

```rust
pub trait ProcessManager: Send + Sync {
    type Event;
    type Command;
    fn handle_event(&mut self, event: &Event<Self::Event>) -> Vec<Self::Command>;
}
```

ตัวอย่าง: `TransferProcessManager` ที่ฟัง `MoneyWithdrawn` จาก source account แล้ว emit `Deposit` command ไปยัง destination account — และถ้า destination fails ให้ emit `Refund` command กลับ

---

### แบบฝึกหัดที่ 4: Snapshot-assisted CommandHandler (ปานกลาง)

ปรับ `CommandHandler` ให้ใช้ snapshot store ช่วย load aggregate เร็วขึ้น:

```rust
pub struct CommandHandlerWithSnapshot<A: Aggregate> {
    store: Box<dyn EventStore<A::Event>>,
    snap_store: Box<dyn SnapshotStore<A>>,
    snap_policy: SnapshotPolicy,
}

impl<A: Aggregate + 'static> CommandHandlerWithSnapshot<A> {
    pub fn execute(&self, id: &str, cmd: A::Command)
        -> Result<Vec<Event<A::Event>>, CommandError<A::Error>>
    {
        // 1. load snapshot ถ้ามี
        // 2. load events หลัง snapshot.version
        // 3. rebuild aggregate จาก snapshot + events
        // 4. handle command
        // 5. append events
        // 6. ถ้า snap_policy.should_snapshot(new_version) → save snapshot
        todo!()
    }
}
```

เขียน benchmark เปรียบเทียบ rebuild time ระหว่าง:
- Aggregate ที่มี 1000 events (ไม่มี snapshot)
- Aggregate ที่มี 1000 events แต่มี snapshot ที่ version 990 (replay แค่ 10 events)

---

### แบบฝึกหัดที่ 5: Event Replay Tool (ปานกลาง)

สร้าง CLI tool สำหรับ replay events จาก production store ไปยัง environment ใหม่:

```
Usage: event-replay [OPTIONS]

Options:
  --from-store <TYPE>      ประเภท source store (postgres, file)
  --to-store <TYPE>        ประเภท destination store
  --aggregate-ids <IDS>    เฉพาะ aggregate IDs ที่กำหนด (comma-separated)
  --from-date <DATE>       replay events ตั้งแต่วันที่กำหนด
  --dry-run                แสดงผลโดยไม่ write จริง
```

ประโยชน์: migrate production data ไป staging, debug ด้วย replay state ณ วันที่เฉพาะ, ทดสอบ new projection กับ historical data

---

### แบบฝึกหัดที่ 6: Dead Letter Queue สำหรับ EventBus (ยาก)

ปรับ `EventBus` ให้มี Dead Letter Queue (DLQ) สำหรับ events ที่ subscriber handle ล้มเหลว:

```rust
pub struct EventBusWithDLQ<T> {
    bus: EventBus<T>,
    dlq: Arc<Mutex<Vec<DeadLetter<T>>>>,
}

pub struct DeadLetter<T> {
    pub event: Event<T>,
    pub error: String,
    pub retry_count: u32,
    pub failed_at: DateTime<Utc>,
}
```

Implement:
- `process_with_retry(event, handler, max_retries)` — retry N ครั้งก่อนส่งไป DLQ
- `drain_dlq()` — process events ใน DLQ อีกครั้ง
- CLI command สำหรับ inspect DLQ contents

## สรุป

โปรเจคนี้สร้าง Event Sourcing Framework ที่ครบสมบูรณ์ประกอบด้วยส่วนหลัก 6 ส่วน:

1. **`Event<T>` + `EventStore<T>`** — foundation ของระบบ — immutable event log พร้อม optimistic locking
2. **`Aggregate` trait** — domain logic encapsulation ด้วย command validation และ immutable state transitions
3. **`CommandHandler<A>`** — orchestrates write path: load → rebuild → command → append
4. **`Projection<T>` + `ProjectionManager`** — CQRS read side ที่ rebuild ได้จาก event stream
5. **`Snapshot<S>` + `SnapshotPolicy`** — performance optimization สำหรับ aggregates ที่มี event history ยาว
6. **`EventBus<T>`** — real-time pub/sub สำหรับ live event notifications ด้วย `tokio::broadcast`

**Pattern สำคัญที่ได้เรียน:**
- `DeserializeOwned` แก้ปัญหา lifetime conflict กับ `#[derive(Deserialize)]`
- Immutable `apply()` + `fold()` ทำให้ `rebuild()` เขียนง่ายและ test ได้
- `Box<dyn EventStore<T>>` ใน `CommandHandler` ทำให้ swap implementation ระหว่าง in-memory และ database ได้โดยไม่เปลี่ยน command logic
- `Arc<Mutex<dyn Projection<T>>>` สำหรับ type-erased, thread-safe projections

**โปรเจคถัดไป** (F07) จะสำรวจ **CRDTs (Conflict-free Replicated Data Types)** — data structures ที่ออกแบบสำหรับ distributed systems ที่ต้องการ merge state จาก multiple nodes โดยไม่มี conflict ซึ่งเป็น alternative approach ในการแก้ปัญหา consistency ที่ต่างออกไปจาก Event Sourcing

---

**โปรเจคก่อนหน้า:** [Project F05: Saga Pattern Orchestrator](project-f05-saga.md) | **โปรเจคถัดไป:** [Project F07: CRDTs](project-f07-crdts.md)
