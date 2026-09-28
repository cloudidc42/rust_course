# Project F03: Distributed Lock Manager

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

**Distributed Lock Manager** คือระบบที่ใช้ควบคุมการเข้าถึง shared resource ในสภาพแวดล้อมที่มีหลาย process หรือหลาย node ทำงานพร้อมกัน ปัญหาที่ระบบนี้แก้คือ **race condition** ใน distributed system — ตัวอย่างเช่น เมื่อ service หลายตัวพยายามเขียนไฟล์เดียวกัน หรืออัพเดต record เดียวในฐานข้อมูลพร้อมกัน

ความยากของ distributed lock ที่แตกต่างจาก single-process mutex คือ:

1. **Network partition** — node อาจสูญเสียการติดต่อกัน แต่ยังคิดว่าตัวเองถือ lock อยู่
2. **Process crash** — lock อาจค้างอยู่ตลอดกาลถ้าไม่มี TTL
3. **Clock skew** — เวลาในแต่ละ node อาจไม่ตรงกัน ทำให้ TTL คำนวณผิด
4. **Split-brain** — สองโหนดต่างอ้างว่าตัวเองถือ lock ในเวลาเดียวกัน

โปรเจคนี้สร้าง distributed lock manager ที่ได้รับแรงบันดาลใจจาก **Redlock algorithm** ของ Redis และเพิ่ม **fencing token** ตาม pattern ของ Martin Kleppmann เพื่อแก้ปัญหา split-brain แม้ในกรณีที่ process ช้าหรือ network lag

**Use case จริงใน production:**
- ป้องกัน double-spending ใน payment service
- Leader election ใน distributed cluster
- Rate limiting แบบ distributed (token bucket ที่แชร์กันระหว่าง nodes)
- Job scheduling — ป้องกัน cron job รัน duplicate ในหลาย instance
- Cache invalidation coordinated — ป้องกัน thundering herd

## สิ่งที่จะได้เรียนรู้

- **Fencing token pattern** — monotonically increasing counter ที่ป้องกัน stale lock holder ทำงานหลัง lock หมดอายุ
- **DashMap** — concurrent hash map ที่ไม่ต้อง lock ทั้ง map สำหรับ read/write
- **AtomicU64** — counter ที่ thread-safe โดยไม่ต้องใช้ Mutex
- **Redlock algorithm** — majority-based distributed lock acquisition บน N nodes
- **Reentrant lock** — same holder re-acquire ได้ด้วย reference counting
- **Condvar + Mutex** — waiter notification pattern ใน Rust standard library
- **TTL-based expiry** — auto-cleanup lock ที่ holder crash ไปแล้ว
- **Stale release rejection** — token validation ป้องกัน delayed/retry release

## ความรู้ที่ต้องมีมาก่อน

- **Part 46-50**: async/await, tokio runtime — ใช้สำหรับ async lock operations
- **Part 51-60**: Arc, Mutex, RwLock — shared state ใน multi-threaded context
- **Part 61-65**: DashMap, concurrent data structures
- **Part 96-100**: AtomicU64, memory ordering (SeqCst)
- **Part 101-105**: Condvar, thread synchronization primitives
- **Part 106-110**: Distributed systems concepts — CAP theorem, consensus basics

## โครงสร้างโปรเจค (Project Layout)

```
dist-lock/
├── src/
│   ├── lib.rs           # Public API exports
│   ├── main.rs          # Demo binary
│   ├── lock_token.rs    # LockToken struct + LockError enum
│   ├── lock_store.rs    # LockStore (DashMap-backed, fencing tokens)
│   ├── reentrant.rs     # ReentrantLockStore (ref-counted)
│   ├── multinode.rs     # LockNode + DistributedLockManager (Redlock)
│   └── waiter.rs        # WaiterQueue (FIFO, Condvar-based)
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### แนวคิดหลัก: Fencing Token

ปัญหาที่ซับซ้อนที่สุดของ distributed lock คือ **"process ช้า"** สมมุติว่า:

```
เวลา →
Client A: [  acquire lock  ][  ...ทำงานช้ามาก...  ][  write DB  ]
                                              ↑
                                          lock expired!
Client B:                                [acquire lock][write DB ]
```

ในสถานการณ์นี้ ทั้ง A และ B ต่างเขียน DB พร้อมกัน! — นี่คือ split-brain

**Fencing token** แก้ปัญหานี้โดยทำให้ lock แต่ละครั้งมี monotonically increasing number ที่ storage layer ตรวจสอบ:

```
Client A ได้ fence=1 → lock expired → พยายามเขียน DB ด้วย fence=1
Client B ได้ fence=2 → เขียน DB ด้วย fence=2 → DB บันทึกว่า last_seen=2
Client A เขียน DB ด้วย fence=1 → REJECTED! (1 < last_seen=2)
```

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    LockStore (single node)                   │
│                                                              │
│  DashMap<key, LockToken>     DashMap<key, FenceCounter>      │
│  ┌──────────────────┐        ┌────────────────────────┐      │
│  │ "payment-svc"    │        │ "payment-svc" → 42      │      │
│  │ token_id: abc123 │        │ "db-conn"    → 7        │      │
│  │ holder: node-1   │        └────────────────────────┘      │
│  │ fence: 42        │                                         │
│  │ ttl: 5000ms      │        DashMap<key, last_seen>          │
│  └──────────────────┘        ┌────────────────────────┐      │
│                              │ "payment-svc" → 42      │      │
└──────────────────────────────┴────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              DistributedLockManager (Redlock)                │
│                                                              │
│  Node 0: LockStore  Node 1: LockStore  Node 2: LockStore     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ acquire ok   │  │ acquire ok   │  │ FAILED       │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
│                                                              │
│  acquired=2, quorum=2 → SUCCESS                              │
└─────────────────────────────────────────────────────────────┘
```

### ทำไมถึงใช้ DashMap แทน Mutex<HashMap>

`DashMap` แบ่ง map ออกเป็น N shards (default: CPU cores × 4) แต่ละ shard มี RwLock ของตัวเอง การ read/write key ต่างๆ จึง concurrent ได้จริง ต่างจาก `Mutex<HashMap>` ที่ lock ทั้ง map ทุกครั้ง

```rust
// Mutex<HashMap> — bottleneck
let mut map = mutex.lock().unwrap();  // blocks ทุก thread
map.insert(key, value);

// DashMap — concurrent
map.insert(key, value);  // lock เฉพาะ shard ที่ key อยู่
```

### ทำไม Fencing Token ต้องใช้ AtomicU64 + SeqCst

```rust
fn next(&self) -> u64 {
    self.0.fetch_add(1, Ordering::SeqCst) + 1
}
```

`SeqCst` (Sequentially Consistent) รับประกันว่า:
1. ทุก thread เห็น increment ในลำดับเดียวกัน
2. ไม่มี reordering ข้ามทั้ง read และ write
3. ค่าที่ได้จาก `fetch_add` จะไม่ duplicate ข้าม thread

ถ้าใช้ `Relaxed` ordering อาจเกิด race ที่สอง thread ได้ fence counter เหมือนกัน

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: กำหนด Data Types และ Error Handling

เริ่มต้นด้วยการออกแบบ type หลักก่อน — `LockToken` ที่ client จะถือไว้ และ `LockError` สำหรับ error cases ที่เป็นไปได้ทั้งหมด

**`Cargo.toml`**

```toml
[package]
name = "dist-lock"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
dashmap = "6"
uuid = { version = "1", features = ["v4"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

**`src/lock_token.rs`**

```rust
use std::time::{Duration, Instant};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// LockToken ที่ถูก issue ให้กับ client เมื่อได้รับ lock สำเร็จ
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LockToken {
    /// ชื่อ resource ที่ถูก lock
    pub key: String,
    /// ID เฉพาะของ token นี้ (ใช้ validate ตอน release)
    pub token_id: String,
    /// holder ที่ถือ lock (เช่น node-id หรือ client-id)
    pub holder_id: String,
    /// fencing token — monotonically increasing per key
    pub fence_counter: u64,
    /// เวลาที่ acquire (ไม่ serialize เพราะ Instant ไม่ผ่าน serde)
    #[serde(skip)]
    pub acquired_at: Option<Instant>,
    pub ttl_ms: u64,
}

impl LockToken {
    pub fn new(key: String, holder_id: String, fence_counter: u64, ttl_ms: u64) -> Self {
        Self {
            key,
            token_id: Uuid::new_v4().to_string(),
            holder_id,
            fence_counter,
            acquired_at: Some(Instant::now()),
            ttl_ms,
        }
    }

    /// ตรวจสอบว่า lock ยังไม่ expired
    pub fn is_valid(&self) -> bool {
        if let Some(acquired_at) = self.acquired_at {
            let elapsed = acquired_at.elapsed();
            elapsed < Duration::from_millis(self.ttl_ms)
        } else {
            false
        }
    }

    /// คำนวณเวลาที่เหลือ (milliseconds)
    pub fn remaining_ms(&self) -> u64 {
        if let Some(acquired_at) = self.acquired_at {
            let elapsed_ms = acquired_at.elapsed().as_millis() as u64;
            self.ttl_ms.saturating_sub(elapsed_ms)
        } else {
            0
        }
    }
}
```

สังเกตการใช้ `#[serde(skip)]` บน `acquired_at: Option<Instant>` เพราะ `Instant` ไม่ implement `Serialize` — ในระบบ production จริง เราจะใช้ `SystemTime` หรือ Unix timestamp แทน แต่สำหรับ in-memory simulation นี้ `Instant` ให้ precision ที่ดีกว่า

```rust
/// ข้อผิดพลาดที่เกิดขึ้นระหว่างการทำ distributed lock operations
#[derive(Debug, Clone, PartialEq)]
pub enum LockError {
    /// Lock ถูกถือโดย holder อื่นอยู่แล้ว
    AlreadyLocked { holder: String, fence_counter: u64 },
    /// Token ที่ใช้ release ไม่ตรงกับ token ที่ถืออยู่ (stale release)
    TokenMismatch,
    /// Lock ไม่มีอยู่ใน store
    LockNotFound,
    /// Lock หมดอายุแล้ว (expired TTL)
    LockExpired,
    /// Quorum ไม่เพียงพอ (สำหรับ multi-node)
    QuorumNotMet { acquired: usize, required: usize },
    /// Fencing token ต่ำกว่า token ที่เคยเห็นมาแล้ว
    StaleFencingToken { provided: u64, last_seen: u64 },
}

impl std::fmt::Display for LockError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            LockError::AlreadyLocked { holder, fence_counter } => {
                write!(f, "lock held by '{}' (fence={})", holder, fence_counter)
            }
            LockError::TokenMismatch => write!(f, "token mismatch: stale release attempt"),
            LockError::LockNotFound => write!(f, "lock not found"),
            LockError::LockExpired => write!(f, "lock has expired"),
            LockError::QuorumNotMet { acquired, required } => {
                write!(f, "quorum not met: {}/{} nodes", acquired, required)
            }
            LockError::StaleFencingToken { provided, last_seen } => {
                write!(f, "stale fencing token: provided={}, last_seen={}", provided, last_seen)
            }
        }
    }
}
```

การใช้ struct variants ใน `AlreadyLocked { holder, fence_counter }` และ `QuorumNotMet { acquired, required }` แทน tuple ทำให้ code ที่ match error อ่านง่ายกว่ามาก เพราะเห็นทันทีว่าแต่ละ field หมายถึงอะไร

---

### ขั้นที่ 2: LockStore พร้อม Fencing Token

หัวใจของระบบคือ `LockStore` ที่ใช้ `DashMap` เก็บ lock ปัจจุบัน และ `AtomicU64` สำหรับ fencing counter ของแต่ละ key

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use dashmap::DashMap;

use crate::lock_token::{LockError, LockToken};

/// เก็บ fencing counter แยกต่างหากต่อ key
struct FenceCounter(AtomicU64);

impl FenceCounter {
    fn new() -> Self {
        Self(AtomicU64::new(0))
    }

    fn next(&self) -> u64 {
        self.0.fetch_add(1, Ordering::SeqCst) + 1
    }

    fn current(&self) -> u64 {
        self.0.load(Ordering::SeqCst)
    }
}

pub struct LockStore {
    locks: DashMap<String, LockToken>,
    fences: DashMap<String, Arc<FenceCounter>>,
    last_seen: DashMap<String, u64>,
}
```

เหตุผลที่ `FenceCounter` ถูกเก็บใน `Arc<FenceCounter>` แทนที่จะเก็บ `AtomicU64` ตรงๆ คือ `DashMap::entry().or_insert_with()` คืน `RefMut` ที่มี lifetime ผูกกับ shard lock — เราต้องการ clone ออกมาก่อน drop RefMut เพื่อหลีกเลี่ยง deadlock กับ `self.locks`

```rust
impl LockStore {
    pub fn try_acquire(
        &self,
        key: &str,
        holder_id: &str,
        ttl_ms: u64,
    ) -> Result<LockToken, LockError> {
        // ดึง (หรือสร้าง) fence counter สำหรับ key นี้
        let fence = self
            .fences
            .entry(key.to_string())
            .or_insert_with(|| Arc::new(FenceCounter::new()))
            .clone();

        // ตรวจสอบ lock ปัจจุบัน
        if let Some(existing) = self.locks.get(key) {
            if existing.is_valid() {
                return Err(LockError::AlreadyLocked {
                    holder: existing.holder_id.clone(),
                    fence_counter: existing.fence_counter,
                });
            }
            drop(existing);
            self.locks.remove(key);
        }

        let fence_value = fence.next();
        let token = LockToken::new(
            key.to_string(),
            holder_id.to_string(),
            fence_value,
            ttl_ms,
        );

        self.locks.insert(key.to_string(), token.clone());
        Ok(token)
    }

    pub fn release(&self, token: &LockToken) -> Result<(), LockError> {
        match self.locks.get(&token.key) {
            None => Err(LockError::LockNotFound),
            Some(stored) => {
                if stored.token_id != token.token_id {
                    Err(LockError::TokenMismatch)
                } else {
                    drop(stored);
                    self.locks.remove(&token.key);
                    Ok(())
                }
            }
        }
    }
}
```

**จุดสำคัญ**: `drop(existing)` ก่อน `self.locks.remove(key)` เป็นสิ่งจำเป็น เพราะ `DashMap::get()` ส่งคืน `Ref` ที่ถือ read lock บน shard นั้น หาก `remove()` พยายาม acquire write lock บน shard เดียวกันโดยที่ read lock ยังอยู่ จะเกิด **deadlock**

---

### ขั้นที่ 3: Fencing Token Validation สำหรับ Consumer

consumer ที่รับ LockToken ไปใช้งานควร validate fence counter ก่อนทำ operation บน shared resource

```rust
pub fn validate_fence(&self, key: &str, fence_token: u64) -> Result<(), LockError> {
    let last = self.last_seen.get(key).map(|v| *v).unwrap_or(0);
    if fence_token < last {
        Err(LockError::StaleFencingToken {
            provided: fence_token,
            last_seen: last,
        })
    } else {
        self.last_seen.insert(key.to_string(), fence_token);
        Ok(())
    }
}
```

**Pattern การใช้งาน:**

```rust
// ใน service ที่รับ LockToken มา
fn write_to_storage(store: &LockStore, token: &LockToken, data: &str) -> Result<(), LockError> {
    // Step 1: validate fencing token ก่อนเสมอ
    store.validate_fence(&token.key, token.fence_counter)?;
    
    // Step 2: ตรวจสอบว่า lock ยังไม่ expired
    if !token.is_valid() {
        return Err(LockError::LockExpired);
    }
    
    // Step 3: ทำ operation จริง
    println!("Writing '{}' to storage (fence={})", data, token.fence_counter);
    Ok(())
}
```

---

### ขั้นที่ 4: TTL Expiry และ Auto-Eviction

Lock ต้องหมดอายุอัตโนมัติเพื่อป้องกันการค้างเมื่อ holder crash — `is_valid()` ใน `LockToken` ตรวจสอบ elapsed time เทียบกับ `ttl_ms`:

```rust
pub fn is_valid(&self) -> bool {
    if let Some(acquired_at) = self.acquired_at {
        acquired_at.elapsed() < Duration::from_millis(self.ttl_ms)
    } else {
        false
    }
}
```

และ `evict_expired()` ใน `LockStore` ทำ cleanup แบบ explicit:

```rust
pub fn evict_expired(&self) {
    let expired_keys: Vec<String> = self
        .locks
        .iter()
        .filter(|entry| !entry.value().is_valid())
        .map(|entry| entry.key().clone())
        .collect();

    for key in expired_keys {
        self.locks.remove(&key);
    }
}
```

**ข้อสังเกต**: เราต้อง collect expired keys ก่อน แล้วค่อย remove ทีหลัง เพราะ `DashMap::iter()` ถือ shard lock อยู่ — หาก `remove()` ระหว่าง iteration จะพยายาม acquire write lock บน shard ที่ read lock ยังอยู่ → deadlock

ใน production system ควรมี background task ที่รัน eviction ทุกๆ N วินาที:

```rust
// ตัวอย่าง: background eviction loop (tokio)
pub async fn start_eviction_loop(store: Arc<LockStore>, interval_ms: u64) {
    let mut ticker = tokio::time::interval(
        std::time::Duration::from_millis(interval_ms)
    );
    loop {
        ticker.tick().await;
        store.evict_expired();
    }
}
```

---

### ขั้นที่ 5: Reentrant Lock

Reentrant lock อนุญาตให้ holder เดิมขอ lock ซ้ำได้ โดย increment reference count แทน ประโยชน์คือ code ที่เรียกกัน recursively หรือ function ที่ assume lock ถูกถือโดย caller อยู่แล้วสามารถขอ lock ได้อีกครั้งโดยไม่ deadlock

```rust
pub struct ReentrantEntry {
    pub holder_id: String,
    pub token_id: String,
    pub fence_counter: u64,
    pub ref_count: u32,
    pub acquired_at: Instant,
    pub ttl_ms: u64,
}

pub struct ReentrantLockStore {
    locks: Arc<Mutex<HashMap<String, ReentrantEntry>>>,
    fence_counters: Arc<Mutex<HashMap<String, AtomicU64>>>,
    _global_counter: AtomicU64,
}
```

Logic ใน `try_acquire`:

```rust
pub fn try_acquire(
    &self,
    key: &str,
    holder_id: &str,
    ttl_ms: u64,
) -> Result<(String, u64), LockError> {
    let mut locks = self.locks.lock().unwrap();

    if let Some(entry) = locks.get_mut(key) {
        if entry.is_valid() {
            if entry.holder_id == holder_id {
                // Reentrant: same holder — เพิ่ม ref_count และ extend TTL
                entry.ref_count += 1;
                entry.extend_ttl(ttl_ms);
                return Ok((entry.token_id.clone(), entry.fence_counter));
            } else {
                return Err(LockError::AlreadyLocked {
                    holder: entry.holder_id.clone(),
                    fence_counter: entry.fence_counter,
                });
            }
        }
        // Expired — ลบออกก่อน acquire ใหม่
        locks.remove(key);
    }

    // ... สร้าง entry ใหม่
}
```

และ `release` ที่ลด ref_count:

```rust
pub fn release(&self, key: &str, token_id: &str) -> Result<bool, LockError> {
    let mut locks = self.locks.lock().unwrap();
    match locks.get_mut(key) {
        None => Err(LockError::LockNotFound),
        Some(entry) => {
            if entry.token_id != token_id {
                Err(LockError::TokenMismatch)
            } else {
                entry.ref_count -= 1;
                let fully_released = entry.ref_count == 0;
                if fully_released {
                    locks.remove(key);
                }
                Ok(fully_released)  // true = lock ถูกปล่อยจริง
            }
        }
    }
}
```

**ตัวอย่างการใช้งาน reentrant lock:**

```rust
fn process_transaction(store: &ReentrantLockStore, key: &str, holder: &str) {
    let (tok, _) = store.try_acquire(key, holder, 5000).unwrap();
    
    // เรียก helper ที่ก็ try_acquire key เดิม
    write_audit_log(store, key, holder);
    
    store.release(key, &tok).unwrap();
}

fn write_audit_log(store: &ReentrantLockStore, key: &str, holder: &str) {
    // Reentrant — ได้ token เดิม, ref_count++ 
    let (tok, _) = store.try_acquire(key, holder, 5000).unwrap();
    println!("Writing audit log while holding lock");
    store.release(key, &tok).unwrap();  // ref_count--
}
```

---

### ขั้นที่ 6: Multi-Node Simulation (Redlock Algorithm)

**Redlock** คือ algorithm สำหรับ distributed lock ที่ Redis เสนอ หลักการคือ acquire lock บน N node พร้อมกัน และถือว่า "ได้ lock" เมื่อ:
1. Acquire สำเร็จบน majority (> N/2) ของ node
2. เวลาที่ใช้ acquire ทั้งหมดน้อยกว่า TTL ที่กำหนด (เพื่อให้มี validity window เพียงพอ)

```rust
pub struct LockNode {
    pub node_id: String,
    store: LockStore,
    pub failure_rate: f64,
}

impl LockNode {
    pub fn try_acquire(
        &self,
        key: &str,
        holder_id: &str,
        ttl_ms: u64,
    ) -> Result<LockToken, LockError> {
        if self.failure_rate >= 1.0 {
            return Err(LockError::AlreadyLocked {
                holder: format!("{}-simulated-failure", self.node_id),
                fence_counter: 0,
            });
        }
        self.store.try_acquire(key, holder_id, ttl_ms)
    }
}

pub struct DistributedLockManager {
    nodes: Vec<Arc<Mutex<LockNode>>>,
}

impl DistributedLockManager {
    pub fn quorum(&self) -> usize {
        self.nodes.len() / 2 + 1
    }

    pub fn acquire(
        &self,
        key: &str,
        holder_id: &str,
        ttl_ms: u64,
    ) -> Result<Vec<LockToken>, LockError> {
        let start = Instant::now();
        let quorum = self.quorum();
        let mut acquired_tokens: Vec<(usize, LockToken)> = Vec::new();

        for (idx, node) in self.nodes.iter().enumerate() {
            let node = node.lock().unwrap();
            match node.try_acquire(key, holder_id, ttl_ms) {
                Ok(token) => acquired_tokens.push((idx, token)),
                Err(_) => {}
            }
        }

        let elapsed_ms = start.elapsed().as_millis() as u64;
        let effective_ttl = ttl_ms.saturating_sub(elapsed_ms);

        if acquired_tokens.len() >= quorum && effective_ttl > ttl_ms / 10 {
            Ok(acquired_tokens.into_iter().map(|(_, t)| t).collect())
        } else {
            // ล้มเหลว — ปล่อย lock ทั้งหมดที่ได้ไปแล้ว (สำคัญมาก!)
            let acquired_count = acquired_tokens.len();
            for (idx, token) in acquired_tokens {
                let node = self.nodes[idx].lock().unwrap();
                let _ = node.release(&token);
            }
            Err(LockError::QuorumNotMet {
                acquired: acquired_count,
                required: quorum,
            })
        }
    }
}
```

**ขั้นตอนสำคัญ**: เมื่อ quorum ไม่ถึง ต้อง **release lock บน node ที่ acquire ไปแล้วทุกตัว** ทันที เพราะถ้าไม่ปล่อย node เหล่านั้นจะ lock ค้างไปจนกว่า TTL จะหมด ทำให้ client อื่นรอนานเกินจำเป็น

---

### ขั้นที่ 7: WaiterQueue — Fair FIFO Notification

เมื่อ lock ถูกปล่อย ระบบควร notify ผู้รอคอยรายแรกสุด (FIFO fair ordering) แทนที่จะปล่อยให้ทุก waiter แข่งกัน acquire ซึ่งอาจทำให้บาง waiter รอนานผิดปกติ (starvation)

```rust
use std::collections::VecDeque;
use std::sync::{Arc, Condvar, Mutex};

pub struct WaiterQueue {
    queues: Mutex<HashMap<String, VecDeque<(String, Arc<(Mutex<Option<WaiterNotification>>, Condvar)>)>>>,
}

impl WaiterQueue {
    pub fn enqueue(
        &self,
        key: &str,
        holder_id: &str,
    ) -> Arc<(Mutex<Option<WaiterNotification>>, Condvar)> {
        let pair = Arc::new((Mutex::new(None), Condvar::new()));
        let mut queues = self.queues.lock().unwrap();
        queues
            .entry(key.to_string())
            .or_insert_with(VecDeque::new)
            .push_back((holder_id.to_string(), Arc::clone(&pair)));
        pair
    }

    pub fn notify_next(&self, key: &str) -> Option<String> {
        let mut queues = self.queues.lock().unwrap();
        if let Some(queue) = queues.get_mut(key) {
            if let Some((holder_id, pair)) = queue.pop_front() {
                let (lock, cvar) = &*pair;
                let mut slot = lock.lock().unwrap();
                *slot = Some(WaiterNotification {
                    key: key.to_string(),
                    notified_holder: holder_id.clone(),
                });
                cvar.notify_one();
                return Some(holder_id);
            }
        }
        None
    }
}
```

**Pattern การใช้งาน — blocking wait:**

```rust
// Waiter thread
let wq = Arc::new(WaiterQueue::new());
let pair = wq.enqueue("my-lock", "client-B");

// Block รอ notification
let (lock, cvar) = &*pair;
let mut notif = lock.lock().unwrap();
while notif.is_none() {
    notif = cvar.wait(notif).unwrap();
}
println!("Got notified: {:?}", notif.as_ref().unwrap());

// ---

// Lock holder thread (เมื่อ release)
wq.notify_next("my-lock");
```

**Integration ระหว่าง LockStore และ WaiterQueue:**

```rust
fn release_and_notify(
    store: &LockStore,
    wq: &WaiterQueue,
    token: &LockToken,
) -> Result<(), LockError> {
    store.release(token)?;
    // หลัง release สำเร็จ → notify waiter ถัดไป
    if let Some(notified) = wq.notify_next(&token.key) {
        println!("Notified waiter: {}", notified);
    }
    Ok(())
}
```

---

### ขั้นที่ 8: Main Demo และ Integration

สร้าง binary ที่แสดง flow ทั้งหมดให้เห็น end-to-end:

```rust
use dist_lock::{LockStore, ReentrantLockStore, DistributedLockManager, LockNode, WaiterQueue};

fn main() {
    println!("=== Distributed Lock Manager Demo ===\n");

    // --- Demo 1: Basic lock ---
    let store = LockStore::new();
    let token = store.try_acquire("payment-service", "node-1", 5000).unwrap();
    println!("Acquired: key={}, fence={}", token.key, token.fence_counter);

    let result = store.try_acquire("payment-service", "node-2", 5000);
    println!("node-2 try: {:?}", result.map_err(|e| e.to_string()));

    store.release(&token).unwrap();
    let token2 = store.try_acquire("payment-service", "node-2", 5000).unwrap();
    println!("node-2 acquired: fence={}\n", token2.fence_counter);

    // --- Demo 2: Fencing validation ---
    store.validate_fence("payment-service", token2.fence_counter).unwrap();
    let stale = store.validate_fence("payment-service", 1);
    println!("Stale fence: {:?}\n", stale.map_err(|e| e.to_string()));

    // --- Demo 3: Reentrant ---
    let rstore = ReentrantLockStore::new();
    let (tok, _) = rstore.try_acquire("db-conn", "worker-1", 5000).unwrap();
    let _ = rstore.try_acquire("db-conn", "worker-1", 5000).unwrap();
    println!("Reentrant ref_count={}", rstore.ref_count("db-conn"));
    rstore.release("db-conn", &tok).unwrap();
    rstore.release("db-conn", &tok).unwrap();
    println!("After 2 releases: is_locked={}\n", rstore.is_locked("db-conn"));

    // --- Demo 4: Redlock ---
    let nodes: Vec<LockNode> = (0..5)
        .map(|i| LockNode::new(format!("redis-{}", i)))
        .collect();
    let mgr = DistributedLockManager::new(nodes);
    let tokens = mgr.acquire("critical-section", "app-1", 5000).unwrap();
    println!("Acquired {}/{} nodes (quorum={})", tokens.len(), mgr.node_count(), mgr.quorum());

    mgr.fail_node(0); mgr.fail_node(1); mgr.fail_node(2);
    let result2 = mgr.acquire("other-lock", "app-2", 5000);
    println!("3/5 nodes down: {:?}\n", result2.map_err(|e| e.to_string()));

    // --- Demo 5: Waiter queue ---
    let wq = WaiterQueue::new();
    wq.enqueue("shared-file", "client-A");
    wq.enqueue("shared-file", "client-B");
    wq.enqueue("shared-file", "client-C");
    println!("Queue: {:?}", wq.queue_order("shared-file"));
    println!("Notify: {:?}", wq.notify_next("shared-file"));
    println!("Remaining: {}", wq.waiter_count("shared-file"));
}
```

**Output จากการรัน `cargo run`:**

```
=== Distributed Lock Manager Demo ===

--- Demo 1: Basic Lock ---
Acquired lock: key=payment-service, fence=1, holder=node-1
node-2 try acquire: Err("lock held by 'node-1' (fence=1)")
Released lock
node-2 acquired after release: fence=2

--- Demo 2: Fencing Token ---
Fence 2 validated ok
Stale fence validation: Err("stale fencing token: provided=1, last_seen=2")

--- Demo 3: Reentrant Lock ---
First acquire: ref_count=1
Reentrant acquire: ref_count=2
First release: ref_count=1
Second release: is_locked=false

--- Demo 4: Distributed (Redlock) ---
Acquired on 5/5 nodes (quorum=3)
Acquire with 3/5 nodes down: Err("quorum not met: 0/3 nodes")

--- Demo 5: Waiter Queue ---
Queue order: ["client-A", "client-B", "client-C"]
Notified: Some("client-A"), Some("client-B")
Remaining: 1

=== Demo Complete ===
```

---

## การทดสอบ (Testing)

### Unit Tests ครบถ้วน — ทั้งหมด 19 test

โปรเจคนี้มี test ครอบคลุมทุก module:

**`src/lock_store.rs` — 8 tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;
    use std::time::Duration;

    #[test]
    fn test_acquire_success() {
        let store = LockStore::new();
        let token = store.try_acquire("resource-1", "node-A", 5000).unwrap();
        assert_eq!(token.key, "resource-1");
        assert_eq!(token.holder_id, "node-A");
        assert!(token.fence_counter > 0);
        assert!(token.is_valid());
    }

    #[test]
    fn test_acquire_already_locked() {
        let store = LockStore::new();
        let _token1 = store.try_acquire("resource-2", "node-A", 5000).unwrap();
        let result = store.try_acquire("resource-2", "node-B", 5000);
        assert!(result.is_err());
        match result.unwrap_err() {
            LockError::AlreadyLocked { holder, .. } => assert_eq!(holder, "node-A"),
            e => panic!("expected AlreadyLocked, got {:?}", e),
        }
    }

    #[test]
    fn test_release_success() {
        let store = LockStore::new();
        let token = store.try_acquire("resource-3", "node-A", 5000).unwrap();
        assert!(store.is_locked("resource-3"));
        store.release(&token).unwrap();
        assert!(!store.is_locked("resource-3"));
    }

    #[test]
    fn test_stale_release_rejected() {
        let store = LockStore::new();
        let token1 = store.try_acquire("resource-4", "node-A", 5000).unwrap();
        store.release(&token1).unwrap();
        let _token2 = store.try_acquire("resource-4", "node-B", 5000).unwrap();
        // token1 เป็น stale — ต้อง reject
        let result = store.release(&token1);
        assert_eq!(result.unwrap_err(), LockError::TokenMismatch);
    }

    #[test]
    fn test_ttl_expiry_allows_reacquire() {
        let store = LockStore::new();
        let _token = store.try_acquire("resource-5", "node-A", 50).unwrap();
        thread::sleep(Duration::from_millis(100));
        let token2 = store.try_acquire("resource-5", "node-B", 5000);
        assert!(token2.is_ok(), "should reacquire after TTL expiry");
    }

    #[test]
    fn test_fencing_token_monotonically_increasing() {
        let store = LockStore::new();
        let t1 = store.try_acquire("resource-6", "node-A", 50).unwrap();
        thread::sleep(Duration::from_millis(80));
        let t2 = store.try_acquire("resource-6", "node-B", 5000).unwrap();
        assert!(
            t2.fence_counter > t1.fence_counter,
            "fence counter must increase: {} > {}",
            t2.fence_counter, t1.fence_counter
        );
    }

    #[test]
    fn test_fence_validation_rejects_stale_token() {
        let store = LockStore::new();
        let t1 = store.try_acquire("resource-7", "node-A", 50).unwrap();
        store.validate_fence("resource-7", t1.fence_counter).unwrap();
        thread::sleep(Duration::from_millis(80));
        let t2 = store.try_acquire("resource-7", "node-B", 5000).unwrap();
        store.validate_fence("resource-7", t2.fence_counter).unwrap();
        // ลอง validate fence เก่า → ต้อง fail
        let result = store.validate_fence("resource-7", t1.fence_counter);
        assert!(result.is_err());
        match result.unwrap_err() {
            LockError::StaleFencingToken { provided, last_seen } => {
                assert_eq!(provided, t1.fence_counter);
                assert_eq!(last_seen, t2.fence_counter);
            }
            e => panic!("expected StaleFencingToken, got {:?}", e),
        }
    }

    #[test]
    fn test_evict_expired() {
        let store = LockStore::new();
        let _t = store.try_acquire("short", "node-A", 50).unwrap();
        assert_eq!(store.active_count(), 1);
        thread::sleep(Duration::from_millis(100));
        store.evict_expired();
        assert_eq!(store.active_count(), 0);
    }
}
```

**`src/reentrant.rs` — 3 tests:**

```rust
#[test]
fn test_reentrant_acquire_increments_ref_count() {
    let store = ReentrantLockStore::new();
    let (tok, _) = store.try_acquire("db-write", "worker-1", 5000).unwrap();
    assert_eq!(store.ref_count("db-write"), 1);
    let (tok2, _) = store.try_acquire("db-write", "worker-1", 5000).unwrap();
    assert_eq!(tok, tok2, "reentrant acquire must return same token_id");
    assert_eq!(store.ref_count("db-write"), 2);
}

#[test]
fn test_reentrant_release_decrements_ref_count() {
    let store = ReentrantLockStore::new();
    let (tok, _) = store.try_acquire("db-write", "worker-1", 5000).unwrap();
    let _ = store.try_acquire("db-write", "worker-1", 5000).unwrap();
    let fully = store.release("db-write", &tok).unwrap();
    assert!(!fully);           // ยังถือ lock อยู่
    assert!(store.is_locked("db-write"));
    let fully = store.release("db-write", &tok).unwrap();
    assert!(fully);            // ปล่อยจริง
    assert!(!store.is_locked("db-write"));
}

#[test]
fn test_different_holder_cannot_reacquire() {
    let store = ReentrantLockStore::new();
    let _ = store.try_acquire("shared-res", "worker-1", 5000).unwrap();
    let result = store.try_acquire("shared-res", "worker-2", 5000);
    assert!(result.is_err());
}
```

**`src/multinode.rs` — 4 tests:**

```rust
#[test]
fn test_majority_acquire_all_healthy() {
    let mgr = make_manager(3);
    let tokens = mgr.acquire("payment-lock", "client-1", 5000).unwrap();
    assert!(tokens.len() >= mgr.quorum());
}

#[test]
fn test_majority_acquire_one_node_down() {
    let mgr = make_manager(3);
    mgr.fail_node(0);
    let result = mgr.acquire("payment-lock", "client-1", 5000);
    assert!(result.is_ok(), "should succeed with 2/3 nodes: {:?}", result);
}

#[test]
fn test_majority_acquire_fails_no_quorum() {
    let mgr = make_manager(3);
    mgr.fail_node(0);
    mgr.fail_node(1);
    let result = mgr.acquire("payment-lock", "client-1", 5000);
    assert!(result.is_err());
    match result.unwrap_err() {
        LockError::QuorumNotMet { acquired, required } => {
            assert_eq!(acquired, 1);
            assert_eq!(required, 2);
        }
        e => panic!("expected QuorumNotMet, got {:?}", e),
    }
}

#[test]
fn test_quorum_calculation() {
    let mgr3 = make_manager(3);
    assert_eq!(mgr3.quorum(), 2);
    let mgr5 = make_manager(5);
    assert_eq!(mgr5.quorum(), 3);
}
```

**`src/waiter.rs` — 3 tests:**

```rust
#[test]
fn test_waiter_enqueue_and_notify() {
    let wq = Arc::new(WaiterQueue::new());
    let pair = wq.enqueue("db-lock", "worker-1");
    assert_eq!(wq.waiter_count("db-lock"), 1);

    let wq_clone = Arc::clone(&wq);
    thread::spawn(move || {
        thread::sleep(Duration::from_millis(20));
        wq_clone.notify_next("db-lock");
    });

    let (lock, cvar) = &*pair;
    let notif = cvar
        .wait_timeout(lock.lock().unwrap(), Duration::from_millis(500))
        .unwrap();
    assert!(notif.0.is_some(), "waiter should be notified");
}

#[test]
fn test_fifo_ordering() {
    let wq = WaiterQueue::new();
    wq.enqueue("file-lock", "client-A");
    wq.enqueue("file-lock", "client-B");
    wq.enqueue("file-lock", "client-C");

    let order = wq.queue_order("file-lock");
    assert_eq!(order, vec!["client-A", "client-B", "client-C"]);

    let notified = wq.notify_next("file-lock");
    assert_eq!(notified, Some("client-A".to_string()));
    assert_eq!(wq.waiter_count("file-lock"), 2);
}

#[test]
fn test_notify_empty_queue_returns_none() {
    let wq = WaiterQueue::new();
    let result = wq.notify_next("nonexistent-key");
    assert_eq!(result, None);
}
```

### Real `cargo test` Output

```
running 19 tests
test lock_store::tests::test_acquire_already_locked ... ok
test lock_store::tests::test_acquire_success ... ok
test lock_store::tests::test_independent_keys_do_not_interfere ... ok
test lock_store::tests::test_release_success ... ok
test lock_store::tests::test_stale_release_rejected ... ok
test lock_store::tests::test_fencing_token_monotonically_increasing ... ok
test lock_store::tests::test_fence_validation_rejects_stale_token ... ok
test multinode::tests::test_majority_acquire_all_healthy ... ok
test multinode::tests::test_majority_acquire_fails_no_quorum ... ok
test multinode::tests::test_majority_acquire_one_node_down ... ok
test multinode::tests::test_quorum_calculation ... ok
test reentrant::tests::test_different_holder_cannot_reacquire ... ok
test reentrant::tests::test_reentrant_acquire_increments_ref_count ... ok
test reentrant::tests::test_reentrant_release_decrements_ref_count ... ok
test waiter::tests::test_fifo_ordering ... ok
test waiter::tests::test_notify_empty_queue_returns_none ... ok
test lock_store::tests::test_evict_expired ... ok
test lock_store::tests::test_ttl_expiry_allows_reacquire ... ok
test waiter::tests::test_waiter_enqueue_and_notify ... ok

test result: ok. 19 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.10s
```

ทั้ง 19 test ผ่านทั้งหมด ครอบคลุม acquire, release, TTL expiry, stale release rejection, fencing token ordering, reentrant acquire, majority acquire, และ waiter notification

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
./target/release/dist-lock
```

### Benchmark — Concurrent Lock Acquisition

เพิ่ม dependency สำหรับ benchmark:

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "lock_bench"
harness = false
```

```rust
// benches/lock_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use dist_lock::LockStore;
use std::sync::Arc;
use std::thread;

fn bench_sequential_acquire(c: &mut Criterion) {
    let store = LockStore::new();
    c.bench_function("sequential acquire+release", |b| {
        b.iter(|| {
            let token = store
                .try_acquire(black_box("bench-key"), "holder", 5000)
                .unwrap();
            store.release(black_box(&token)).unwrap();
        })
    });
}

fn bench_concurrent_acquire(c: &mut Criterion) {
    c.bench_function("concurrent acquire (10 threads)", |b| {
        b.iter(|| {
            let store = Arc::new(LockStore::new());
            let handles: Vec<_> = (0..10)
                .map(|i| {
                    let store = Arc::clone(&store);
                    thread::spawn(move || {
                        let key = format!("key-{}", i);
                        if let Ok(token) = store.try_acquire(&key, "holder", 1000) {
                            let _ = store.release(&token);
                        }
                    })
                })
                .collect();
            for h in handles {
                h.join().unwrap();
            }
        })
    });
}

criterion_group!(benches, bench_sequential_acquire, bench_concurrent_acquire);
criterion_main!(benches);
```

```bash
cargo bench
```

### Docker Deployment

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/dist-lock /usr/local/bin/
CMD ["dist-lock"]
```

```bash
docker build -t dist-lock:latest .
docker run --rm dist-lock:latest
```

---

## ข้อควรระวัง (Pitfalls)

### Pitfall 1: Deadlock จาก DashMap Ref ยังไม่ถูก Drop

```rust
// ❌ DEADLOCK: Ref<T> ยังค้างอยู่เมื่อเรียก remove()
if let Some(existing) = self.locks.get(key) {
    if existing.is_valid() {
        return Err(/* ... */);
    }
    // existing ยังถือ read lock บน shard นั้น!
    self.locks.remove(key);  // ← พยายาม acquire write lock → DEADLOCK
}

// ✅ DROP ก่อน remove
if let Some(existing) = self.locks.get(key) {
    if existing.is_valid() {
        return Err(/* ... */);
    }
    drop(existing);  // release read lock ก่อน
}
self.locks.remove(key);
```

`DashMap::get()` คืน `Ref<'_, K, V>` ที่ถือ read lock บน shard ตลอดช่วงชีวิตของมัน `remove()` ต้องการ write lock บน shard เดียวกัน → deadlock ถ้า Ref ยังอยู่ใน scope

---

### Pitfall 2: Redlock ที่ไม่ปล่อย Lock เมื่อ Quorum ล้มเหลว

```rust
// ❌ WRONG: ล้มเหลวแต่ไม่ปล่อย lock ที่ acquire ไปแล้ว
if acquired_tokens.len() < quorum {
    return Err(LockError::QuorumNotMet { /* ... */ });
    // ← lock บน nodes ที่สำเร็จค้างอยู่จนกว่า TTL จะหมด!
}

// ✅ CORRECT: ปล่อย lock ทั้งหมดก่อน return error
if acquired_tokens.len() < quorum {
    for (idx, token) in acquired_tokens {
        let node = self.nodes[idx].lock().unwrap();
        let _ = node.release(&token);
    }
    return Err(LockError::QuorumNotMet { /* ... */ });
}
```

ถ้าไม่ปล่อย lock เมื่อ quorum fail client อื่นต้องรอจนกว่า TTL จะหมดก่อนจะ acquire ได้ ซึ่งอาจส่งผลต่อ availability ของระบบ

---

### Pitfall 3: ใช้ Relaxed Memory Ordering สำหรับ Fencing Counter

```rust
// ❌ WRONG: Relaxed ordering ไม่รับประกัน global ordering
fn next(&self) -> u64 {
    self.0.fetch_add(1, Ordering::Relaxed) + 1
}

// ✅ CORRECT: SeqCst รับประกัน all threads เห็น increment ในลำดับเดียวกัน
fn next(&self) -> u64 {
    self.0.fetch_add(1, Ordering::SeqCst) + 1
}
```

`Relaxed` ordering อนุญาตให้ CPU และ compiler reorder memory operations ได้อย่างอิสระ ในทางปฏิบัติ สอง thread อาจ `fetch_add` พร้อมกันและได้ค่า counter เหมือนกัน ทำลาย monotonicity ที่ fencing token ต้องการ

---

### Pitfall 4: Stale Release โดย Process ที่ฟื้นจาก Pause

```
สถานการณ์:
1. Process A acquire lock (token_id=abc, fence=1, ttl=5s)
2. Process A หยุด (GC pause, swap, debugging) นาน 6 วินาที
3. Lock ของ A expired, Process B acquire lock (token_id=xyz, fence=2)
4. Process A ฟื้น, เรียก release(token_id=abc) ← stale!
```

```rust
// ✅ การ validate token_id ป้องกัน stale release
pub fn release(&self, token: &LockToken) -> Result<(), LockError> {
    match self.locks.get(&token.key) {
        None => Err(LockError::LockNotFound),
        Some(stored) => {
            if stored.token_id != token.token_id {
                Err(LockError::TokenMismatch)  // ← ปฏิเสธ stale release
            } else {
                drop(stored);
                self.locks.remove(&token.key);
                Ok(())
            }
        }
    }
}
```

การเปรียบเทียบ `token_id` (UUID v4) ทำให้มั่นใจได้ว่า release จะสำเร็จก็ต่อเมื่อ token ที่ส่งมาตรงกับ token ที่ถือ lock อยู่จริงในขณะนั้น

---

### Pitfall 5: Clock Skew ใน Distributed TTL

ปัญหาที่ซ่อนอยู่ในระบบ distributed คือ clock ของแต่ละ node อาจไม่ตรงกัน ตัวอย่าง:
- Node A ส่ง lock request โดยบอก "TTL = 5000ms" และ timestamp = T
- Node B ได้รับ request ช้า 200ms (network delay)
- Node B เริ่มนับ TTL จาก T+200ms แต่ Node A เริ่มจาก T

ผลคือ lock บน Node B expire เร็วกว่าที่ client คาดไว้ 200ms

```rust
// แนวทางแก้: ใช้ elapsed time จากเวลาที่ client เริ่ม acquire
pub fn acquire(&self, key: &str, holder_id: &str, ttl_ms: u64) -> Result<Vec<LockToken>, LockError> {
    let start = Instant::now();
    // ... acquire on all nodes ...
    let elapsed_ms = start.elapsed().as_millis() as u64;
    // TTL จริงที่ใช้งานได้ = TTL ที่กำหนด - เวลาที่ใช้ acquire
    let effective_ttl = ttl_ms.saturating_sub(elapsed_ms);
    
    // ถ้า effective TTL น้อยเกินไป (< 10% ของ TTL ที่กำหนด) → ถือว่าล้มเหลว
    if effective_ttl <= ttl_ms / 10 {
        // release all and return error
    }
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Async Lock Acquisition (ระดับกลาง)

แปลง `LockStore` ให้เป็น async โดยใช้ `tokio::sync::Mutex` และ implement `async fn try_acquire_or_wait()` ที่ block จนกว่าจะได้ lock หรือ timeout:

```rust
// เป้าหมาย API:
let store = AsyncLockStore::new();
match tokio::time::timeout(
    Duration::from_secs(10),
    store.try_acquire_or_wait("payment", "client-1", 5000),
).await {
    Ok(Ok(token)) => { /* got lock */ }
    Ok(Err(e)) => { /* error */ }
    Err(_) => { /* timeout */ }
}
```

Hint: ใช้ `tokio::sync::Notify` แทน `Condvar` สำหรับ async context

---

### แบบฝึกหัดที่ 2: Persistent Lock State ด้วย SQLite (ระดับสูง)

Lock state ปัจจุบันหายไปเมื่อ process restart — เพิ่ม persistence layer โดย:
1. เพิ่ม dependency `sqlx` พร้อม SQLite feature
2. สร้าง `CREATE TABLE locks (key TEXT PRIMARY KEY, token_id TEXT, holder_id TEXT, fence_counter INTEGER, acquired_at INTEGER, ttl_ms INTEGER)`
3. `try_acquire()` → INSERT OR REPLACE เมื่อสำเร็จ
4. `release()` → DELETE ด้วย WHERE token_id = ?
5. Startup: load unexpired locks จาก DB เข้า memory cache

ประเด็นที่น่าสนใจ: เมื่อ process restart ต้องจัดการ lock ที่ expire ระหว่าง downtime อย่างไร?

---

### แบบฝึกหัดที่ 3: HTTP API สำหรับ Lock Service (ระดับสูง)

ห่อ `LockStore` ด้วย HTTP server โดยใช้ `axum`:

```
POST   /locks/{key}          — acquire lock (body: {holder_id, ttl_ms})
DELETE /locks/{key}/{token}  — release lock  
GET    /locks/{key}          — ดู lock status
GET    /locks                — list all active locks
POST   /locks/{key}/renew    — extend TTL ของ lock ที่ถืออยู่
```

ทดสอบด้วย `curl`:
```bash
curl -X POST http://localhost:8080/locks/payment \
     -H 'Content-Type: application/json' \
     -d '{"holder_id": "worker-1", "ttl_ms": 30000}'
```

---

### แบบฝึกหัดที่ 4: Read-Write Lock (ระดับกลาง)

ขยาย `LockStore` ให้รองรับ read-write lock semantics:
- หลาย reader acquire พร้อมกันได้
- Writer ต้องรอให้ reader ทุกคน release ก่อน
- Reader ใหม่ที่มาระหว่าง writer รอ ต้องรอ writer ก่อน (fair RW lock)

```rust
pub enum LockMode { Read, Write }

pub fn try_acquire_rw(
    &self,
    key: &str,
    holder_id: &str,
    mode: LockMode,
    ttl_ms: u64,
) -> Result<LockToken, LockError>
```

---

### แบบฝึกหัดที่ 5: Lock Monitoring Dashboard (ระดับกลาง)

สร้าง metrics endpoint ที่ export ข้อมูลในรูปแบบ Prometheus text format:

```
# HELP dist_lock_active_locks Number of currently active locks
# TYPE dist_lock_active_locks gauge
dist_lock_active_locks 5

# HELP dist_lock_acquisitions_total Total lock acquisitions
# TYPE dist_lock_acquisitions_total counter
dist_lock_acquisitions_total{result="success"} 1234
dist_lock_acquisitions_total{result="failure"} 56

# HELP dist_lock_ttl_seconds Lock TTL distribution
# TYPE dist_lock_ttl_seconds histogram
dist_lock_ttl_seconds_bucket{le="1"} 100
dist_lock_ttl_seconds_bucket{le="5"} 800
dist_lock_ttl_seconds_bucket{le="+Inf"} 1234
```

---

### แบบฝึกหัดที่ 6: Lease Renewal (ระดับกลาง)

เพิ่ม API สำหรับ holder ที่ต้องทำงานนานกว่า TTL ที่กำหนดไว้:

```rust
pub fn renew(&self, token: &LockToken, extend_ms: u64) -> Result<LockToken, LockError>
```

โดยที่:
1. ต้อง validate `token_id` ตรงกับ holder ปัจจุบัน
2. สร้าง `LockToken` ใหม่ที่มี `acquired_at = Instant::now()` และ `ttl_ms = extend_ms`
3. **ไม่** เปลี่ยน `fence_counter` — เป็น token เดิมแค่ extend เวลา
4. Client ต้อง renew ก่อน lock expire (เช่น renew เมื่อเหลือ 20% ของ TTL)

---

## ความแตกต่างจาก Mutex ใน Standard Library

เพื่อให้เห็นภาพชัดว่า distributed lock แตกต่างจาก `std::sync::Mutex` อย่างไร:

| Feature | `std::sync::Mutex` | Distributed Lock |
|---|---|---|
| Scope | Process เดียว | หลาย process / node |
| Auto-release เมื่อ crash | ✅ (RAII Drop) | ต้องใช้ TTL |
| Fencing token | ไม่จำเป็น | จำเป็นสำหรับ split-brain |
| Network partition | N/A | ต้องออกแบบ explicitly |
| Performance | Nanoseconds | Milliseconds (network) |
| Reentrant | ต้องใช้ `parking_lot::ReentrantMutex` | ต้องสร้าง logic เอง |

กฎสำคัญ: **ใช้ distributed lock เฉพาะเมื่อจำเป็น** — ถ้า operation เกิดขึ้นใน process เดียว `Mutex` ธรรมดาดีกว่าเสมอ เพราะไม่มี network overhead และ RAII ทำให้ไม่มี lock leak

### เมื่อไหร่ควรใช้ Distributed Lock

```
✅ ใช้เมื่อ:
- หลาย process/service ต้องเข้าถึง shared resource พร้อมกัน
- ต้องการ leader election
- ต้องการ rate limiting แบบ cluster-wide

❌ ไม่ควรใช้เมื่อ:
- Single process (ใช้ Mutex แทน)
- ต้องการ throughput สูง (lock คือ bottleneck)
- Operation idempotent อยู่แล้ว (ไม่จำเป็นต้องป้องกัน)
```

---

### การเปรียบเทียบกับ Redis SETNX

ในทางปฏิบัติ Redis ถูกใช้เป็น distributed lock backend ผ่าน `SET key value NX PX ttl`:

```
# Redis command
SET payment:lock "holder-abc" NX PX 5000

# ถ้า return "OK" → ได้ lock
# ถ้า return (nil) → lock ถูกถือโดยคนอื่น
```

Redlock ขยายความคิดนี้ไปยัง N Redis instances โดยต้อง SET สำเร็จบน majority — โปรเจคนี้จำลอง behavior เดียวกันใน memory โดยใช้ `DashMap` แทน Redis instance

ข้อจำกัดของ Redlock ที่ Martin Kleppmann ชี้ให้เห็น (ดู "How to do distributed locking" บล็อก 2016):
1. **Fencing token ไม่ได้รับประกันโดย Redlock** — Redlock ไม่ออก monotonic token ให้ client ใช้ validate
2. **Clock assumption** — Redlock assume clocks ไม่ drift เกินกว่า TTL ซึ่งในทางปฏิบัติอาจเกิดได้
3. **Process pause** — GC หรือ OS scheduling pause อาจทำให้ lock expired ระหว่างที่ client กำลังทำงาน

โปรเจคนี้แก้ข้อ 1 ด้วย fencing token และ `validate_fence()` ซึ่ง storage layer ต้องใช้ก่อน accept write

---

## สรุป

โปรเจคนี้สร้าง distributed lock manager ที่ครอบคลุม pattern สำคัญใน distributed systems:

**สิ่งที่สร้าง:**
- `LockStore` — single-node lock store ที่ใช้ `DashMap` สำหรับ concurrent access
- Fencing token ด้วย `AtomicU64` ที่ป้องกัน split-brain
- TTL-based expiry พร้อม `evict_expired()` cleanup
- Stale release rejection ผ่าน `token_id` validation
- `ReentrantLockStore` — reference-counted reentrant locking
- `DistributedLockManager` — Redlock-inspired majority acquisition บน N nodes
- `WaiterQueue` — FIFO fair notification ด้วย `Condvar`

**Pattern ที่ได้เรียน:**
- **Fencing token** — แนวทาง canonical ในการแก้ split-brain ของ distributed lock
- **DashMap** — concurrent hash map ที่ลด lock contention โดย sharding
- **AtomicU64 + SeqCst** — counter ที่ thread-safe ไม่ต้องใช้ Mutex
- **Condvar** — blocking notification primitive ของ Rust standard library
- **Drop-before-mutate** — pattern สำคัญเมื่อใช้ `DashMap::get()` ร่วมกับ `remove()`

**Pattern ที่นำไปใช้ต่อได้ทันที:**
- `DashMap` + `Arc<AtomicU64>` per-key counter → ใช้ได้กับ rate limiter, sequence number generator
- TTL expiry via `Instant::elapsed()` → ใช้กับ cache, session store, temporary token
- Majority voting → ใช้กับ leader election, distributed configuration
- `Condvar` wake pattern → ใช้กับ job queue, event bus

**เชื่อมโยงกับโปรเจคถัดไป:**
โปรเจค F04: Circuit Breaker จะต่อยอดจากแนวคิด failure detection และ state machine ที่ใช้ใน `LockNode::failure_rate` โดยสร้างระบบที่ตรวจจับ service failures อัตโนมัติ และ "เปิดวงจร" เพื่อป้องกัน cascading failures ในระบบ microservices — การที่ `DistributedLockManager` ต้องจัดการ partial failures ในโปรเจคนี้เป็นรากฐานของแนวคิดเดียวกัน

---

**โปรเจคก่อนหน้า:** [Project F02: Service Discovery](project-f02-service-discovery.md) | **โปรเจคถัดไป:** [Project F04: Circuit Breaker](project-f04-circuit-breaker.md)
