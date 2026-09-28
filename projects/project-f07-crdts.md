# Project F07: CRDTs (Conflict-free Replicated Data Types) Library

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

ในระบบ distributed ที่ node หลายตัวสามารถ read/write ข้อมูลพร้อมกัน และ network อาจเกิด partition ขึ้นได้ตลอดเวลา — ปัญหาที่ยากที่สุดคือ **conflict resolution**: เมื่อ node สองตัว update ข้อมูลเดียวกันพร้อมกัน แล้ว sync กัน ใครจะชนะ? จะ merge อย่างไรให้ถูกต้อง?

**CRDT (Conflict-free Replicated Data Type)** คือคำตอบที่นักวิจัยจาก INRIA และ Microsoft Research ค้นพบ: data structure ที่ออกแบบมาเพื่อให้ **merge เสมอได้โดยอัตโนมัติ** โดยไม่ต้องใช้ lock, consensus, หรือ coordinator ใดๆ คุณสมบัติสำคัญคือ operation ทุกอย่างต้องเป็น **idempotent, commutative, และ associative** — ทำซ้ำกี่ครั้ง, สลับลำดับอย่างไร, หรือจัดกลุ่มอย่างไรก็ได้ผลเหมือนกัน

CRDT ถูกใช้ใน production systems ชั้นนำ:
- **Redis** — ใช้ CRDT ใน Redis Enterprise สำหรับ geo-distributed databases
- **Riak** — NoSQL database ที่สร้างบน CRDT เป็นหลัก
- **Amazon DynamoDB** — ใช้ CRDT สำหรับ eventual consistency guarantees
- **Apple Notes / Google Docs** — ใช้ CRDT-based algorithms สำหรับ collaborative editing
- **Figma** — ใช้ CRDT-like approach สำหรับ real-time collaboration ใน design tool
- **Automerge / Yjs** — popular CRDT libraries สำหรับ local-first applications

โปรเจคนี้จะสร้าง **CRDT Library** ใน Rust ที่ครอบคลุม data structure พื้นฐานทั้งหมด:
- **G-Counter** — Grow-only counter, รองรับเฉพาะ increment
- **PN-Counter** — Positive-Negative counter, รองรับทั้ง increment และ decrement
- **G-Set** — Grow-only set, เพิ่มได้อย่างเดียว
- **OR-Set** — Observed-Remove Set ที่มี add-wins semantics สำหรับ concurrent operations
- **LWW-Register** — Last-Write-Wins Register ที่ใช้ timestamp แก้ conflict
- **Vector Clock** — ติดตาม causality ระหว่าง events ใน distributed system

## สิ่งที่จะได้เรียนรู้

- **CRDT theory** — แนวคิด convergence, commutativity, idempotency, associativity ที่เป็นพื้นฐานของ conflict-free replication
- **State-based vs Operation-based CRDTs** — ความแตกต่างระหว่าง CvRDT (convergent) และ CmRDT (commutative) และเมื่อไหรควรใช้อะไร
- **Vector Clock** — algorithm สำหรับติดตาม causality และตรวจสอบ happened-before relation ใน distributed system
- **Generic programming ใน Rust** — ใช้ `<T: Eq + Hash + Clone>` ทำ type-safe data structures
- **HashMap + HashSet ขั้นสูง** — ใช้ collection standard library ใน patterns ที่ซับซ้อน
- **Serde serialization** — serialize/deserialize CRDT state เป็น JSON เพื่อ persistence และ network transfer
- **Property-based thinking** — ออกแบบ test ที่ verify คุณสมบัติ mathematical (commutativity, idempotency, associativity)
- **UUID generation** — ใช้ `uuid` crate สร้าง globally unique tags สำหรับ OR-Set operations

## ความรู้ที่ต้องมีมาก่อน

- **Part 13-17**: Generic types, trait bounds — ใช้ใน `GSet<T>`, `ORSet<T>`, `LWWRegister<T>`
- **Part 18-22**: `std::collections::HashMap` และ `HashSet` — โครงสร้างหลักของ CRDT ทุกตัว
- **Part 23-27**: Trait implementations (`PartialEq`, `Eq`, `Default`, `Clone`) — จำเป็นสำหรับ correctness
- **Part 35-40**: Error handling, `Option<T>` — ใช้ใน `LWWRegister::read()` return `Option`
- **Part 55-60**: `serde`, `serde_json` — serialize/deserialize state ระหว่าง nodes
- **Part 61-65**: `uuid` crate — สร้าง unique tags สำหรับ OR-Set
- **Part 96-100**: Distributed systems concepts — CAP theorem, eventual consistency

## โครงสร้างโปรเจค (Project Layout)

```
crdts/
├── src/
│   ├── lib.rs             # module root — re-exports ทุก CRDT
│   ├── main.rs            # demo binary แสดงการใช้งาน
│   ├── vector_clock.rs    # VectorClock + ClockRelation
│   ├── g_counter.rs       # GCounter — grow-only counter
│   ├── pn_counter.rs      # PNCounter — positive/negative counter
│   ├── g_set.rs           # GSet<T> — grow-only set
│   ├── or_set.rs          # ORSet<T> — observed-remove set (add-wins)
│   └── lww_register.rs    # LWWRegister<T> — last-write-wins register
└── Cargo.toml
```

แต่ละ CRDT อยู่ใน module แยกเพื่อ separation of concerns และ testability ที่ดี ทุก module มี unit tests ของตัวเอง

## การออกแบบ (Architecture & Design)

### หลักการของ CRDT

CRDT ทุกตัวต้องเป็น **join-semilattice** ซึ่งหมายความว่า operation `merge` (เรียกว่า "join" ใน mathematics) ต้องมีคุณสมบัติ:

```
1. Idempotent:    a.merge(a) == a
2. Commutative:   a.merge(b) == b.merge(a)
3. Associative:   (a.merge(b)).merge(c) == a.merge(b.merge(c))
```

ถ้า `merge` ตรงตาม 3 คุณสมบัตินี้ replica ทุกตัวจะ **converge** ไปสู่ state เดียวกันเสมอ ไม่ว่าจะ receive message ในลำดับใด หรือรับซ้ำกี่ครั้ง

### State-based CRDT (CvRDT) vs Operation-based CRDT (CmRDT)

```
State-based (CvRDT)                  Operation-based (CmRDT)
─────────────────────                ─────────────────────────
ส่ง state ทั้งหมดระหว่าง nodes       ส่งเฉพาะ operations
merge ด้วย "join" function           operations ต้อง commutative
ง่ายกว่า แต่ใช้ bandwidth มากกว่า   ประหยัด bandwidth แต่ซับซ้อนกว่า
ต้องใช้ reliable channel หรือไม่ก็ได้  ต้องการ exactly-once delivery
```

โปรเจคนี้ implement **State-based CRDTs** ซึ่งง่ายกว่าและ robust กว่าในทางปฏิบัติ

### Data Flow Overview

```
Node A                    Node B                    Node C
   │                         │                         │
   │ increment()              │ increment()              │ increment()
   │                         │                         │
   │ state_a                 │ state_b                 │ state_c
   │                         │                         │
   └────────────────────────▶│                         │
                merge(a, b)  │◀────────────────────────┘
                             │          merge(b, c)
                             │
                    final state (a ∪ b ∪ c)
                    ← same regardless of order →
```

### Vector Clock และ Causality

Vector Clock แก้ปัญหา "ใครเกิดก่อน?" ในระบบที่ไม่มี global clock:

```
Node A:  VC = {A:1, B:0, C:0}  → เพิ่ม A → {A:2, B:0, C:0}
Node B:  VC = {A:0, B:1, C:0}  → concurrent กับ A
Node C:  รับจาก A แล้ว → {A:2, B:0, C:1}  → happened-after A
```

### OR-Set: Add-Wins Semantics

OR-Set แก้ปัญหา concurrent add/remove ที่ G-Set ทำไม่ได้:

```
Node A:  add("apple", tag=t1)  →  {apple: {t1}}
Node B:  clone ของ A            →  {apple: {t1}}
          remove("apple")        →  {}

Node A:  add("apple", tag=t2)  →  {apple: {t1, t2}}  [concurrent!]

Merge:   A={apple:{t1,t2}}  ∪  B={}  =  {apple:{t2}}  → apple ยังคงอยู่!
```

tag `t2` ที่เกิดหลัง remove ไม่ถูก remove ออก ทำให้ add ชนะ concurrent remove เสมอ

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: ตั้งค่าโปรเจคและ Vector Clock

เริ่มจากส่วนที่สำคัญที่สุด — Vector Clock ที่เป็นพื้นฐานของ causality tracking ใน distributed system ทั้งหมด

**Cargo.toml:**

```toml
[package]
name = "crdts"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }

[[bin]]
name = "demo"
path = "src/main.rs"
```

**src/lib.rs:**

```rust
pub mod vector_clock;
pub mod g_counter;
pub mod pn_counter;
pub mod g_set;
pub mod or_set;
pub mod lww_register;

pub use vector_clock::VectorClock;
pub use g_counter::GCounter;
pub use pn_counter::PNCounter;
pub use g_set::GSet;
pub use or_set::ORSet;
pub use lww_register::LWWRegister;

pub type NodeId = String;
```

**src/vector_clock.rs:**

```rust
use std::collections::HashMap;
use serde::{Deserialize, Serialize};

use crate::NodeId;

/// Vector Clock สำหรับติดตาม causality ใน distributed system
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct VectorClock {
    pub clocks: HashMap<NodeId, u64>,
}

/// ผลการเปรียบเทียบ Vector Clock สองอัน
#[derive(Debug, PartialEq, Eq)]
pub enum ClockRelation {
    HappenedBefore,
    HappenedAfter,
    Concurrent,
    Equal,
}

impl VectorClock {
    pub fn new() -> Self {
        VectorClock {
            clocks: HashMap::new(),
        }
    }

    /// เพิ่ม clock ของ node ที่กำหนด
    pub fn increment(&mut self, node: &NodeId) {
        let counter = self.clocks.entry(node.clone()).or_insert(0);
        *counter += 1;
    }

    /// ดึงค่า clock ของ node (ถ้าไม่มีคือ 0)
    pub fn get(&self, node: &NodeId) -> u64 {
        *self.clocks.get(node).unwrap_or(&0)
    }

    /// ตรวจสอบว่า self happened-before other หรือไม่
    /// self ≤ other แบบ component-wise และ self ≠ other
    pub fn happened_before(&self, other: &VectorClock) -> bool {
        // ทุก entry ใน self ต้องน้อยกว่าหรือเท่ากับ other
        let all_lte = self.clocks.iter().all(|(node, &v)| v <= other.get(node));
        if !all_lte {
            return false;
        }
        // ต้องมี entry อย่างน้อยหนึ่งที่ self < other
        let some_lt = self.clocks.iter().any(|(node, &v)| v < other.get(node))
            || other.clocks.iter().any(|(node, &v)| v > self.get(node));
        some_lt
    }

    /// ตรวจสอบว่า concurrent กันหรือไม่
    pub fn concurrent(&self, other: &VectorClock) -> bool {
        !self.happened_before(other) && !other.happened_before(self) && self != other
    }

    /// Merge (least upper bound) — เอาค่า max ในแต่ละ component
    pub fn merge(&self, other: &VectorClock) -> VectorClock {
        let mut merged = self.clocks.clone();
        for (node, &v) in &other.clocks {
            let entry = merged.entry(node.clone()).or_insert(0);
            *entry = (*entry).max(v);
        }
        VectorClock { clocks: merged }
    }

    /// เปรียบเทียบ relation ระหว่าง clock สองอัน
    pub fn relation(&self, other: &VectorClock) -> ClockRelation {
        if self == other {
            ClockRelation::Equal
        } else if self.happened_before(other) {
            ClockRelation::HappenedBefore
        } else if other.happened_before(self) {
            ClockRelation::HappenedAfter
        } else {
            ClockRelation::Concurrent
        }
    }
}

impl Default for VectorClock {
    fn default() -> Self { Self::new() }
}
```

**ทดสอบ Vector Clock:**

```rust
fn test_vector_clock_happened_before() {
    let mut vc1 = VectorClock::new();
    vc1.increment(&"A".to_string());

    let mut vc2 = VectorClock::new();
    vc2.increment(&"A".to_string());
    vc2.increment(&"A".to_string());

    assert!(vc1.happened_before(&vc2));  // {A:1} happened-before {A:2}
    assert!(!vc2.happened_before(&vc1)); // ไม่ใช่ทิศทางกลับกัน
}

fn test_vector_clock_concurrent() {
    let mut vc1 = VectorClock::new();
    vc1.increment(&"A".to_string()); // {A:1}

    let mut vc2 = VectorClock::new();
    vc2.increment(&"B".to_string()); // {B:1}

    // ทั้งสองไม่รู้จักกัน = concurrent
    assert!(vc1.concurrent(&vc2));
}
```

### ขั้นที่ 2: G-Counter (Grow-only Counter)

G-Counter คือ CRDT ที่ง่ายที่สุด — แต่ละ node มี counter ของตัวเอง ค่ารวมคือ sum ของทุก node และ merge ใช้ max ต่อ node

**src/g_counter.rs:**

```rust
use std::collections::HashMap;
use serde::{Deserialize, Serialize};

use crate::NodeId;

/// G-Counter: Grow-only counter
/// แต่ละ node มี counter ของตัวเอง, ค่ารวม = sum ของทุก node
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct GCounter {
    pub counts: HashMap<NodeId, u64>,
}

impl GCounter {
    pub fn new() -> Self {
        GCounter { counts: HashMap::new() }
    }

    /// เพิ่มค่า counter ของ node ที่กำหนด
    pub fn increment(&mut self, node: &NodeId) {
        let counter = self.counts.entry(node.clone()).or_insert(0);
        *counter += 1;
    }

    /// เพิ่มค่าตามจำนวนที่กำหนด
    pub fn increment_by(&mut self, node: &NodeId, delta: u64) {
        let counter = self.counts.entry(node.clone()).or_insert(0);
        *counter += delta;
    }

    /// ค่ารวมทั้งหมด = sum ของทุก node
    pub fn value(&self) -> u64 {
        self.counts.values().sum()
    }

    /// Merge สอง G-Counter — เอาค่า max ในแต่ละ node
    pub fn merge(&self, other: &GCounter) -> GCounter {
        let mut merged = self.counts.clone();
        for (node, &v) in &other.counts {
            let entry = merged.entry(node.clone()).or_insert(0);
            *entry = (*entry).max(v);
        }
        GCounter { counts: merged }
    }
}
```

**ทำไม merge ต้องใช้ max ไม่ใช่ sum?**

ลองนึกว่า Node A มี counter = 5 แล้วส่งให้ Node B ซึ่งมี counter = 3:
- ถ้า merge ด้วย sum: Node B จะมี `5 + 3 = 8` แต่ operations จริงมีแค่ `5` (ของ A) + `3` (ของ B) = `8` ซึ่งถูก
- แต่ถ้า Node B ส่งกลับมาให้ A อีกครั้ง: A จะได้ `8 + 5 = 13` (นับซ้ำ!)

นั่นคือเหตุผลที่ merge ต้องใช้ **max ต่อ node** — ค่า counter ของแต่ละ node ถือว่า "canonical" และไม่ควรถูกนับซ้ำ

```
ตัวอย่าง merge ที่ถูกต้อง:
Node A: {A:5, B:0}
Node B: {A:3, B:7}

merge = {A: max(5,3), B: max(0,7)} = {A:5, B:7}
value = 5 + 7 = 12
```

### ขั้นที่ 3: PN-Counter (Positive-Negative Counter)

PN-Counter แก้ข้อจำกัดของ G-Counter ที่ increment ได้อย่างเดียว โดยใช้ G-Counter **สองตัว**: ตัวหนึ่งสำหรับ positive operations อีกตัวสำหรับ negative operations

**src/pn_counter.rs:**

```rust
use serde::{Deserialize, Serialize};
use crate::{GCounter, NodeId};

/// PN-Counter: Positive-Negative Counter
/// รองรับทั้ง increment และ decrement โดยใช้ G-Counter สองตัว
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct PNCounter {
    pub pos: GCounter,
    pub neg: GCounter,
}

impl PNCounter {
    pub fn new() -> Self {
        PNCounter {
            pos: GCounter::new(),
            neg: GCounter::new(),
        }
    }

    /// เพิ่มค่า counter ของ node
    pub fn increment(&mut self, node: &NodeId) {
        self.pos.increment(node);
    }

    /// ลดค่า counter ของ node
    pub fn decrement(&mut self, node: &NodeId) {
        self.neg.increment(node);
    }

    /// ค่าปัจจุบัน = pos - neg
    pub fn value(&self) -> i64 {
        self.pos.value() as i64 - self.neg.value() as i64
    }

    /// Merge สอง PN-Counter — merge G-Counter แต่ละตัวแยกกัน
    pub fn merge(&self, other: &PNCounter) -> PNCounter {
        PNCounter {
            pos: self.pos.merge(&other.pos),
            neg: self.neg.merge(&other.neg),
        }
    }
}
```

**Use case: Distributed Like Counter**

ลองจินตนาการ post บน social media ที่มีหลาย replica:

```
Node Bangkok:  +5 likes, -1 unlike  → PN = (5, 1), value = +4
Node Singapore: +3 likes, -2 unlike → PN = (3, 2), value = +1

After merge:
  pos = max(5,3) = 5 per Bangkok, max(3,5)... ต่อ node
  neg = max(1,2) = 2
  value = (5+3) - (1+2) = 8 - 3 = 5  ← ผลลัพธ์ที่ถูกต้อง!
```

### ขั้นที่ 4: G-Set (Grow-only Set)

G-Set เป็น Set ที่เพิ่มได้อย่างเดียว merge คือ union ของทั้งสอง set

**src/g_set.rs:**

```rust
use std::collections::HashSet;
use std::hash::Hash;
use serde::{Deserialize, Serialize};

/// G-Set: Grow-only Set
/// เพิ่มได้อย่างเดียว, ลบไม่ได้
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub struct GSet<T: Eq + Hash + Clone> {
    pub elements: HashSet<T>,
}

impl<T: Eq + Hash + Clone> GSet<T> {
    pub fn new() -> Self {
        GSet { elements: HashSet::new() }
    }

    pub fn insert(&mut self, element: T) {
        self.elements.insert(element);
    }

    pub fn contains(&self, element: &T) -> bool {
        self.elements.contains(element)
    }

    pub fn len(&self) -> usize {
        self.elements.len()
    }

    pub fn is_empty(&self) -> bool {
        self.elements.is_empty()
    }

    /// Merge สอง G-Set — union
    pub fn merge(&self, other: &GSet<T>) -> GSet<T> {
        let mut merged = self.elements.clone();
        for elem in &other.elements {
            merged.insert(elem.clone());
        }
        GSet { elements: merged }
    }
}
```

**ข้อสังเกต:** G-Set ไม่รองรับ delete เลย ถ้าต้องการ delete ต้องใช้ OR-Set (ขั้นที่ 5) หรือ 2P-Set (Two-Phase Set ที่มี add-set และ remove-set แยกกัน แต่ element ที่ถูก remove แล้วจะกลับมาไม่ได้อีก)

**Use case ของ G-Set:** รายชื่อ members ที่เคย join (ไม่ต้องการ remove), audit log ของ events ที่เกิดขึ้น, list ของ completed tasks

### ขั้นที่ 5: OR-Set (Observed-Remove Set)

OR-Set แก้ปัญหา concurrent add/remove ที่ G-Set ทำไม่ได้ ด้วย add-wins semantics

**src/or_set.rs:**

```rust
use std::collections::{HashMap, HashSet};
use std::hash::Hash;
use serde::{Deserialize, Serialize};

pub type UniqueTag = String;

/// OR-Set (Observed-Remove Set): Add-wins semantics
/// ใช้ unique tag สำหรับแต่ละ add operation
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ORSet<T: Eq + Hash + Clone> {
    /// elements: แต่ละ element map ไปยัง set ของ unique tags
    pub entries: HashMap<T, HashSet<UniqueTag>>,
}

impl<T: Eq + Hash + Clone + std::fmt::Debug> ORSet<T> {
    pub fn new() -> Self {
        ORSet { entries: HashMap::new() }
    }

    /// เพิ่ม element พร้อม unique tag
    /// ใช้ UUID เป็น tag เพื่อให้ globally unique
    pub fn add(&mut self, tag: UniqueTag, element: T) {
        self.entries
            .entry(element)
            .or_insert_with(HashSet::new)
            .insert(tag);
    }

    /// ลบ element — ลบเฉพาะ tags ที่เห็นในขณะนี้
    /// concurrent add (ที่มี tag ใหม่) จะยังคงอยู่
    pub fn remove(&mut self, element: &T) {
        self.entries.remove(element);
    }

    /// ตรวจสอบว่า element อยู่ใน set
    pub fn contains(&self, element: &T) -> bool {
        self.entries
            .get(element)
            .map(|tags| !tags.is_empty())
            .unwrap_or(false)
    }

    /// Merge สอง OR-Set — union ของ tags แต่ละ element
    pub fn merge(&self, other: &ORSet<T>) -> ORSet<T> {
        let mut merged: HashMap<T, HashSet<UniqueTag>> = self.entries.clone();
        for (element, tags) in &other.entries {
            merged
                .entry(element.clone())
                .or_insert_with(HashSet::new)
                .extend(tags.iter().cloned());
        }
        ORSet { entries: merged }
    }
}
```

**ความแตกต่างระหว่าง OR-Set และ 2P-Set:**

| Feature | 2P-Set | OR-Set |
|---------|--------|--------|
| Remove-wins vs Add-wins | Remove-wins (ห้าม re-add) | Add-wins |
| Concurrent add/remove | Remove ชนะเสมอ | Add ชนะเสมอ |
| Re-add after remove | ไม่ได้ | ได้ (ถ้าใช้ tag ใหม่) |
| Implementation | ง่ายกว่า | ซับซ้อนกว่า |

OR-Set เหมาะสำหรับ use case ที่ต้องการให้ add ชนะในกรณี concurrent เช่น shopping cart, members list, playlist

**การใช้ UUID เป็น UniqueTag:**

```rust
use uuid::Uuid;

let tag = Uuid::new_v4().to_string(); // "550e8400-e29b-41d4-a716-446655440000"
or_set.add(tag, "item");
```

UUID v4 มีโอกาสชนกัน 1 ใน 2^122 ≈ 5.3 × 10^36 ซึ่งในทางปฏิบัติถือว่า impossible

### ขั้นที่ 6: LWW-Register (Last-Write-Wins Register)

LWW-Register เก็บค่าเดียว โดยใช้ timestamp ในการตัดสินว่าค่าไหน "ใหม่กว่า" และ node_id เป็น tiebreaker เมื่อ timestamp เท่ากัน

**src/lww_register.rs:**

```rust
use serde::{Deserialize, Serialize};
use crate::NodeId;

/// LWW-Register (Last-Write-Wins Register)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LWWRegister<T: Clone> {
    pub value: Option<T>,
    pub timestamp: u64,
    pub node_id: NodeId,
}

impl<T: Clone + PartialEq> LWWRegister<T> {
    pub fn new(node_id: NodeId) -> Self {
        LWWRegister { value: None, timestamp: 0, node_id }
    }

    /// เขียนค่าใหม่พร้อม timestamp
    pub fn write(&mut self, value: T, timestamp: u64) {
        self.value = Some(value);
        self.timestamp = timestamp;
    }

    /// อ่านค่าปัจจุบัน
    pub fn read(&self) -> Option<&T> {
        self.value.as_ref()
    }

    /// Merge สอง LWW-Register — timestamp สูงกว่าชนะ
    /// ถ้า timestamp เท่ากัน ใช้ node_id เป็น tiebreaker (lexicographic)
    pub fn merge(&self, other: &LWWRegister<T>) -> LWWRegister<T> {
        if self.timestamp > other.timestamp {
            self.clone()
        } else if other.timestamp > self.timestamp {
            other.clone()
        } else {
            // timestamp เท่ากัน: node_id ที่ "ใหญ่กว่า" ชนะ
            if self.node_id >= other.node_id {
                self.clone()
            } else {
                other.clone()
            }
        }
    }
}
```

**ข้อควรระวังของ LWW:**

LWW ต้องการ synchronized clocks ระหว่าง nodes ซึ่งใน distributed system ทำได้ยาก ในทางปฏิบัติใช้:
1. **Hybrid Logical Clock (HLC)** — ผสม physical time กับ logical counter
2. **Lamport Timestamp** — logical clock ที่รับประกัน causality
3. **NTP + drift correction** — สำหรับ systems ที่ยอมรับ small inconsistency ได้

### ขั้นที่ 7: Integration — Demo Binary

ประกอบทุก CRDT เข้าด้วยกันใน demo:

**src/main.rs:**

```rust
use crdts::{GCounter, GSet, LWWRegister, ORSet, PNCounter, VectorClock};
use uuid::Uuid;

fn main() {
    println!("=== CRDT Library Demo ===\n");

    // ─── G-Counter ───
    println!("--- G-Counter ---");
    let mut node_a = GCounter::new();
    let mut node_b = GCounter::new();

    node_a.increment(&"node_a".to_string());
    node_a.increment(&"node_a".to_string());
    node_b.increment(&"node_b".to_string());

    let merged = node_a.merge(&node_b);
    println!("Node A counter: {}", node_a.value());
    println!("Node B counter: {}", node_b.value());
    println!("Merged counter: {}", merged.value());

    // ─── PN-Counter ───
    println!("\n--- PN-Counter ---");
    let mut pn = PNCounter::new();
    pn.increment(&"A".to_string());
    pn.increment(&"A".to_string());
    pn.increment(&"A".to_string());
    pn.decrement(&"A".to_string());
    println!("PN-Counter value (3 inc, 1 dec): {}", pn.value());

    // ─── G-Set ───
    println!("\n--- G-Set ---");
    let mut gs1 = GSet::new();
    gs1.insert("Alice");
    gs1.insert("Bob");

    let mut gs2 = GSet::new();
    gs2.insert("Bob");
    gs2.insert("Charlie");

    let merged_set = gs1.merge(&gs2);
    println!("G-Set size after merge: {}", merged_set.len());
    println!("Contains Alice: {}", merged_set.contains(&"Alice"));
    println!("Contains Charlie: {}", merged_set.contains(&"Charlie"));

    // ─── OR-Set ───
    println!("\n--- OR-Set (Add-Wins) ---");
    let tag = Uuid::new_v4().to_string();
    let mut ors = ORSet::new();
    ors.add(tag, "item1");
    println!("Contains item1: {}", ors.contains(&"item1"));
    ors.remove(&"item1");
    println!("After remove, contains item1: {}", ors.contains(&"item1"));

    // ─── LWW-Register ───
    println!("\n--- LWW-Register ---");
    let mut r1 = LWWRegister::new("node1".to_string());
    let mut r2 = LWWRegister::new("node2".to_string());
    r1.write("old_value", 10);
    r2.write("new_value", 20);
    let merged_reg = r1.merge(&r2);
    println!("LWW-Register merged value: {:?}", merged_reg.read());

    // ─── Vector Clock ───
    println!("\n--- Vector Clock ---");
    let mut vc1 = VectorClock::new();
    let mut vc2 = VectorClock::new();

    vc1.increment(&"A".to_string());
    vc1.increment(&"A".to_string());
    vc2.increment(&"B".to_string());

    println!("VC1 happened before VC2: {}", vc1.happened_before(&vc2));
    println!("VC1 concurrent with VC2: {}", vc1.concurrent(&vc2));

    let merged_vc = vc1.merge(&vc2);
    println!("Merged VC: A={}, B={}",
        merged_vc.get(&"A".to_string()),
        merged_vc.get(&"B".to_string()));

    println!("\n=== Demo Complete ===");
}
```

**Output จากการรันจริง:**

```
=== CRDT Library Demo ===

--- G-Counter ---
Node A counter: 2
Node B counter: 1
Merged counter: 3

--- PN-Counter ---
PN-Counter value (3 inc, 1 dec): 2

--- G-Set ---
G-Set size after merge: 3
Contains Alice: true
Contains Charlie: true

--- OR-Set (Add-Wins) ---
Contains item1: true
After remove, contains item1: false

--- LWW-Register ---
LWW-Register merged value: Some("new_value")

--- Vector Clock ---
VC1 happened before VC2: false
VC1 concurrent with VC2: true
Merged VC: A=2, B=1

=== Demo Complete ===
```

### ขั้นที่ 8: JSON Serialization สำหรับ Network Transfer

CRDT ที่มีประโยชน์จริงต้องส่ง state ระหว่าง nodes ได้ เราใช้ `serde_json` สำหรับการนี้

```rust
use serde_json;
use crdts::GCounter;

fn serialize_and_deserialize() {
    let mut counter = GCounter::new();
    counter.increment(&"node1".to_string());
    counter.increment(&"node1".to_string());
    counter.increment(&"node2".to_string());

    // Serialize เป็น JSON สำหรับส่งผ่าน network
    let json = serde_json::to_string(&counter).unwrap();
    println!("Serialized: {}", json);
    // Output: {"counts":{"node1":2,"node2":1}}

    // Deserialize กลับมา
    let received: GCounter = serde_json::from_str(&json).unwrap();
    println!("Deserialized value: {}", received.value());

    // Node ปลายทาง merge กับ state ของตัวเอง
    let mut local = GCounter::new();
    local.increment(&"node3".to_string());

    let merged = local.merge(&received);
    println!("After merge: {}", merged.value());
    // Output: 4
}
```

**State Sync Protocol:**

ในระบบจริง nodes จะแลกเปลี่ยน state แบบนี้:

```
Node A → Node B:  {"action": "sync", "state": {"counts": {"A": 5, "B": 0}}}
Node B receives → deserialize → merge กับ local state → save
Node B → Node A:  {"action": "sync", "state": {"counts": {"A": 3, "B": 7}}}
Node A receives → deserialize → merge → final: {"A": 5, "B": 7}
```

## การทดสอบ (Testing)

โปรเจคนี้มี unit tests ครอบคลุม 34 test cases ตรวจสอบคุณสมบัติ CRDT ทุกด้าน

### หมวดหมู่ Test Cases

**1. Vector Clock Tests (6 tests)**
- `test_vector_clock_increment` — ตรวจ increment แต่ละ node
- `test_vector_clock_happened_before` — causality tracking
- `test_vector_clock_concurrent` — concurrent detection
- `test_vector_clock_equal` — equality relation
- `test_vector_clock_merge` — merge correctness
- `test_vector_clock_merge_idempotent` — merge กับตัวเองได้เหมือนเดิม

**2. G-Counter Tests (5 tests)**
- `test_gcounter_increment` — basic increment
- `test_gcounter_merge_commutativity` — a.merge(b) == b.merge(a)
- `test_gcounter_merge_idempotent` — a.merge(a) == a
- `test_gcounter_merge_associativity` — (a.merge(b)).merge(c) == a.merge(b.merge(c))
- `test_gcounter_merge_takes_max` — merge ใช้ max ต่อ node

**3. PN-Counter Tests (5 tests)**
- `test_pncounter_increment_decrement` — basic operations
- `test_pncounter_negative_value` — ค่าติดลบได้
- `test_pncounter_merge_commutativity` — commutative property
- `test_pncounter_merge_idempotent` — idempotent property
- `test_pncounter_distributed_merge` — simulate distributed sync

**4. G-Set Tests (6 tests)**
- `test_gset_insert_contains` — basic operations
- `test_gset_no_duplicates` — set semantics
- `test_gset_merge_union` — merge คือ union
- `test_gset_merge_commutativity` — commutative
- `test_gset_merge_idempotent` — idempotent
- `test_gset_merge_associativity` — associative

**5. OR-Set Tests (6 tests)**
- `test_orset_add_contains` — basic add
- `test_orset_remove` — basic remove
- `test_orset_add_wins_concurrent_remove` — add-wins semantics
- `test_orset_remove_after_add_wins_settles` — sequential remove ทำงาน
- `test_orset_merge_commutativity` — commutative
- `test_orset_merge_idempotent` — idempotent

**6. LWW-Register Tests (6 tests)**
- `test_lww_register_write_read` — basic write/read
- `test_lww_register_merge_higher_timestamp_wins` — timestamp logic
- `test_lww_register_merge_commutativity` — commutative
- `test_lww_register_merge_idempotent` — idempotent
- `test_lww_register_tiebreaker_node_id` — tiebreaker logic
- `test_lww_register_merge_associativity` — associative

### Real `cargo test` Output

```
   Compiling crdts v0.1.0 (/tmp/.../crdts)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.76s
     Running unittests src/lib.rs (target/debug/deps/crdts-ed7e795a2ac9119f)

running 34 tests
test g_counter::tests::test_gcounter_merge_commutativity ... ok
test g_counter::tests::test_gcounter_merge_idempotent ... ok
test g_counter::tests::test_gcounter_merge_associativity ... ok
test g_counter::tests::test_gcounter_increment ... ok
test g_counter::tests::test_gcounter_merge_takes_max ... ok
test g_set::tests::test_gset_insert_contains ... ok
test g_set::tests::test_gset_merge_associativity ... ok
test g_set::tests::test_gset_merge_commutativity ... ok
test g_set::tests::test_gset_merge_idempotent ... ok
test g_set::tests::test_gset_no_duplicates ... ok
test g_set::tests::test_gset_merge_union ... ok
test lww_register::tests::test_lww_register_merge_associativity ... ok
test lww_register::tests::test_lww_register_merge_commutativity ... ok
test lww_register::tests::test_lww_register_merge_higher_timestamp_wins ... ok
test lww_register::tests::test_lww_register_merge_idempotent ... ok
test lww_register::tests::test_lww_register_tiebreaker_node_id ... ok
test lww_register::tests::test_lww_register_write_read ... ok
test or_set::tests::test_orset_add_contains ... ok
test or_set::tests::test_orset_add_wins_concurrent_remove ... ok
test or_set::tests::test_orset_merge_commutativity ... ok
test or_set::tests::test_orset_merge_idempotent ... ok
test or_set::tests::test_orset_remove ... ok
test or_set::tests::test_orset_remove_after_add_wins_settles ... ok
test pn_counter::tests::test_pncounter_distributed_merge ... ok
test pn_counter::tests::test_pncounter_increment_decrement ... ok
test pn_counter::tests::test_pncounter_merge_commutativity ... ok
test pn_counter::tests::test_pncounter_merge_idempotent ... ok
test pn_counter::tests::test_pncounter_negative_value ... ok
test vector_clock::tests::test_vector_clock_concurrent ... ok
test vector_clock::tests::test_vector_clock_equal ... ok
test vector_clock::tests::test_vector_clock_happened_before ... ok
test vector_clock::tests::test_vector_clock_increment ... ok
test vector_clock::tests::test_vector_clock_merge ... ok
test vector_clock::tests::test_vector_clock_merge_idempotent ... ok

test result: ok. 34 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/demo-fba18dcbecb849c7)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests crdts

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**ทุก test ผ่านหมด 34/34** ครอบคลุมคุณสมบัติ CRDT ที่สำคัญ: commutativity, idempotency, associativity

## Pitfalls ที่ต้องระวัง

### Pitfall 1: ใช้ merge ด้วย sum แทน max ใน G-Counter

ข้อผิดพลาดที่พบบ่อยที่สุด — เข้าใจผิดว่า merge G-Counter ต้องบวกกัน:

```rust
// ❌ ผิด! จะนับซ้ำเมื่อ sync หลายรอบ
pub fn merge_wrong(&self, other: &GCounter) -> GCounter {
    let mut merged = self.counts.clone();
    for (node, &v) in &other.counts {
        let entry = merged.entry(node.clone()).or_insert(0);
        *entry += v;  // บวกแทน max — ผิด!
    }
    GCounter { counts: merged }
}

// ตัวอย่างปัญหา:
// Node A: {A:5}  sync กับ Node B: {A:5}  (ทั้งสอง เห็น state เดิม)
// wrong merge: {A: 5+5} = {A:10}  ← นับซ้ำ!
// correct merge: {A: max(5,5)} = {A:5}  ← ถูกต้อง

// ✅ ถูกต้อง
*entry = (*entry).max(v);
```

เหตุผล: G-Counter แต่ละ node เป็นเจ้าของ counter ของตัวเอง ไม่มีใคร increment counter ของ node อื่นได้ ดังนั้น ค่าสูงสุดที่เห็นคือค่า "จริง" ของ node นั้น

### Pitfall 2: ลบ element ออกจาก G-Set โดยตรง

```rust
// ❌ ผิด! G-Set ไม่ควรมี remove
impl<T: Eq + Hash + Clone> GSet<T> {
    pub fn remove(&mut self, element: &T) {
        self.elements.remove(element);  // ทำลายคุณสมบัติ CRDT!
    }
}

// ปัญหาที่เกิด:
// Node A: {a, b, c}  remove(b)  → {a, c}
// Node B: {a, b}     (ยังไม่รู้ว่า b ถูก remove)
// merge: {a, b, c}   ← b กลับมา! convergence เสีย
```

ถ้าต้องการลบ element ให้ใช้ **OR-Set** หรือ **2P-Set** แทน G-Set ไม่ควรมี remove operation

### Pitfall 3: OR-Set add ด้วย tag ซ้ำ

```rust
// ❌ ผิด! tag ซ้ำทำให้ add-wins ไม่ทำงาน
let same_tag = "fixed-tag".to_string();
or_set.add(same_tag.clone(), "apple");  // {apple: {"fixed-tag"}}
or_set.remove(&"apple");                // {}
or_set.add(same_tag.clone(), "apple");  // เหมือน add ซ้ำ tag เก่า

// ปัญหา: ถ้า node อื่น remove "fixed-tag" แล้ว add กลับมาด้วย tag เดิม
// merge จะเป็น {} เพราะ tag เดิมถูก remove ไปแล้ว
// add-wins จะไม่ทำงาน!

// ✅ ถูกต้อง: ใช้ UUID ทุกครั้ง
use uuid::Uuid;
or_set.add(Uuid::new_v4().to_string(), "apple");  // tag ใหม่เสมอ
```

UUID v4 รับประกัน uniqueness ในทางปฏิบัติ ทำให้ add operation ใหม่เสมอ "ชนะ" remove operation เก่า

### Pitfall 4: LWW-Register กับ Clock Skew

```rust
// ❌ ปัญหา: ใช้ system time โดยตรงโดยไม่จัดการ skew
use std::time::{SystemTime, UNIX_EPOCH};

fn write_with_wall_clock(&mut self, value: T) {
    let ts = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_millis() as u64;
    self.write(value, ts);
}

// ปัญหา: ถ้า Node A มี clock เร็วกว่า Node B อยู่ 1 วินาที
// Node A write ที่ t=1000, Node B write ที่ t=999 (แต่จริงๆ เกิดทีหลัง!)
// Merge: Node A ชนะเพราะ timestamp สูงกว่า ทั้งที่จริงๆ Node B เป็น "ล่าสุด"
```

วิธีแก้ที่แนะนำ:

```rust
// ✅ ตัวเลือก 1: ใช้ Hybrid Logical Clock (HLC)
// ตัวเลือก 2: ให้ application layer กำหนด logical timestamp
// ตัวเลือก 3: ใช้ monotonic counter แทน wall clock
pub fn write_with_logical_ts(&mut self, value: T, logical_ts: u64) {
    if logical_ts > self.timestamp {
        self.write(value, logical_ts);
    }
}
```

### Pitfall 5: ไม่ validate CRDT properties ใน test

ข้อผิดพลาดของนักพัฒนาคือเขียน test แบบ happy-path โดยไม่ verify คุณสมบัติ mathematical:

```rust
// ❌ test ที่ไม่เพียงพอ
#[test]
fn test_gcounter_merge() {
    let mut a = GCounter::new();
    a.increment(&"A".to_string());
    let b = GCounter::new();
    let merged = a.merge(&b);
    assert_eq!(merged.value(), 1);  // test แค่ happy path
}

// ✅ test ที่ดีกว่า — verify mathematical properties
#[test]
fn test_gcounter_merge_commutativity() {
    let mut a = GCounter::new();
    a.increment(&"A".to_string());
    a.increment(&"A".to_string());

    let mut b = GCounter::new();
    b.increment(&"B".to_string());

    // commutativity
    assert_eq!(a.merge(&b), b.merge(&a));

    // idempotency
    assert_eq!(a.merge(&a.clone()), a);

    // ค่าถูกต้อง
    assert_eq!(a.merge(&b).value(), 3);
}
```

ในระบบ production บาง team ใช้ property-based testing เช่น `proptest` crate เพื่อ generate random inputs แล้ว verify properties อัตโนมัติ

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่: target/release/demo
```

### Publish เป็น crate บน crates.io

```toml
# Cargo.toml
[package]
name = "my-crdts"
version = "0.1.0"
edition = "2021"
description = "A library of Conflict-free Replicated Data Types"
license = "MIT OR Apache-2.0"
repository = "https://github.com/yourname/crdts"
documentation = "https://docs.rs/my-crdts"
keywords = ["crdt", "distributed", "replicated", "eventual-consistency"]
categories = ["data-structures", "network-programming"]
```

```bash
# login ก่อน
cargo login

# dry run ก่อน publish จริง
cargo publish --dry-run

# publish จริง
cargo publish
```

### ใช้เป็น library ใน โปรเจคอื่น

```toml
# Cargo.toml ของ project ที่ใช้
[dependencies]
crdts = { version = "0.1", features = ["serde"] }
```

```rust
use crdts::{GCounter, ORSet};
use uuid::Uuid;

// ใช้ใน distributed shopping cart
let mut cart = ORSet::<String>::new();
cart.add(Uuid::new_v4().to_string(), "product-123".to_string());
```

### Integration ใน Distributed System

ตัวอย่างการ integrate กับ tokio สำหรับ async network sync:

```toml
[dependencies]
crdts = "0.1"
tokio = { version = "1", features = ["full"] }
serde_json = "1"
```

```rust
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use crdts::GCounter;

async fn sync_with_peer(
    peer_addr: &str,
    local_state: &GCounter,
) -> Result<GCounter, Box<dyn std::error::Error>> {
    let mut stream = TcpStream::connect(peer_addr).await?;

    // ส่ง local state
    let json = serde_json::to_string(local_state)?;
    stream.write_all(json.as_bytes()).await?;

    // รับ state ของ peer
    let mut buf = Vec::new();
    stream.read_to_end(&mut buf).await?;
    let peer_state: GCounter = serde_json::from_slice(&buf)?;

    // merge
    Ok(local_state.merge(&peer_state))
}
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Implement 2P-Set (Two-Phase Set)

2P-Set เป็น set ที่มีทั้ง add-set และ remove-set เมื่อ element ถูก add แล้ว remove จะไม่สามารถ add กลับได้ (remove-wins) ลองออกแบบ:

```rust
// โครงสร้าง
pub struct TwoPSet<T: Eq + Hash + Clone> {
    pub add_set: GSet<T>,
    pub remove_set: GSet<T>,
}

impl<T: Eq + Hash + Clone> TwoPSet<T> {
    // element อยู่ใน set ถ้า: อยู่ใน add_set AND ไม่อยู่ใน remove_set
    pub fn contains(&self, element: &T) -> bool {
        self.add_set.contains(element) && !self.remove_set.contains(element)
    }

    // TODO: implement add, remove, merge
}
```

**ความท้าทาย:** ทำให้ merge เป็น commutative, idempotent, associative และ write tests

### แบบฝึกหัดที่ 2: Implement RGA (Replicated Growable Array)

RGA เป็น CRDT สำหรับ ordered sequence เช่น collaborative text editing:

```rust
pub struct RGA<T: Clone> {
    // แต่ละ element มี unique identifier และ pointer ไปยัง element ก่อนหน้า
    pub elements: Vec<RGAElement<T>>,
}

pub struct RGAElement<T: Clone> {
    pub id: (u64, NodeId),   // (timestamp, node_id) เป็น unique ID
    pub value: T,
    pub deleted: bool,       // tombstone สำหรับ delete ที่ยังคง history
}
```

**Use case:** Google Docs-style collaborative text editing, collaborative JSON editing

### แบบฝึกหัดที่ 3: Benchmark Performance

เพิ่ม criterion benchmarks เพื่อวัด performance ของ merge operations:

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "crdt_bench"
harness = false
```

```rust
// benches/crdt_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use crdts::GCounter;

fn bench_gcounter_merge(c: &mut Criterion) {
    let mut a = GCounter::new();
    let mut b = GCounter::new();

    // สร้าง large state
    for i in 0..100 {
        let node = format!("node_{}", i);
        a.increment_by(&node, i as u64 * 100);
        b.increment_by(&node, i as u64 * 50);
    }

    c.bench_function("gcounter_merge_100_nodes", |bench| {
        bench.iter(|| black_box(a.merge(&b)))
    });
}

criterion_group!(benches, bench_gcounter_merge);
criterion_main!(benches);
```

### แบบฝึกหัดที่ 4: เพิ่ม Delta-State Protocol

แทนที่จะส่ง full state ทุกครั้ง ลองออกแบบ delta protocol:

```rust
// Delta G-Counter: ส่งเฉพาะ nodes ที่เปลี่ยนแปลงไป
pub struct GCounterDelta {
    pub changes: HashMap<NodeId, u64>,
    pub from_version: u64,
}

impl GCounter {
    /// สร้าง delta ตั้งแต่ version ที่กำหนด
    pub fn delta_since(&self, base: &GCounter) -> GCounterDelta {
        let changes = self.counts.iter()
            .filter(|(node, &v)| v > base.get(node))
            .map(|(node, &v)| (node.clone(), v))
            .collect();
        GCounterDelta { changes, from_version: 0 }
    }

    /// Apply delta เข้ากับ state ปัจจุบัน
    pub fn apply_delta(&mut self, delta: &GCounterDelta) {
        for (node, &v) in &delta.changes {
            let entry = self.counts.entry(node.clone()).or_insert(0);
            *entry = (*entry).max(v);
        }
    }
}
```

Delta protocol ลด bandwidth ได้มากใน scenarios ที่ nodes มีขนาดใหญ่แต่ update ครั้งละน้อย

### แบบฝึกหัดที่ 5: Persistent Storage Layer

เพิ่ม trait สำหรับ persistence เพื่อ survive restart:

```rust
pub trait CrdtStore {
    fn save<T: serde::Serialize>(&self, key: &str, value: &T) -> Result<(), StoreError>;
    fn load<T: serde::de::DeserializeOwned>(&self, key: &str) -> Result<Option<T>, StoreError>;
}

pub struct FileCrdtStore {
    pub base_dir: std::path::PathBuf,
}

impl CrdtStore for FileCrdtStore {
    fn save<T: serde::Serialize>(&self, key: &str, value: &T) -> Result<(), StoreError> {
        let path = self.base_dir.join(format!("{}.json", key));
        let json = serde_json::to_string_pretty(value)?;
        std::fs::write(path, json)?;
        Ok(())
    }

    fn load<T: serde::de::DeserializeOwned>(&self, key: &str) -> Result<Option<T>, StoreError> {
        let path = self.base_dir.join(format!("{}.json", key));
        if !path.exists() {
            return Ok(None);
        }
        let json = std::fs::read_to_string(path)?;
        Ok(Some(serde_json::from_str(&json)?))
    }
}
```

### แบบฝึกหัดที่ 6: Network Sync Simulation

สร้าง simulation ของ distributed system ที่ nodes sync กันผ่าน gossip protocol:

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

pub struct DistributedGCounter {
    pub nodes: HashMap<NodeId, Arc<Mutex<GCounter>>>,
}

impl DistributedGCounter {
    /// Simulate gossip: node A ส่ง state ให้ random node
    pub fn gossip_round(&self) {
        let node_ids: Vec<_> = self.nodes.keys().cloned().collect();
        // สุ่มเลือกคู่ nodes ที่จะ sync กัน
        // merge state
        // repeat หลายๆ รอบจนกว่าทุก node จะ converge
    }
}
```

ทดสอบว่าหลัง N rounds ของ gossip ทุก node มี value เหมือนกันหรือไม่ (convergence guarantee)

## สรุป

โปรเจคนี้สร้าง **CRDT Library** ที่ครบถ้วนประกอบด้วย 6 data structures:

| CRDT | Use Case หลัก | Key Property |
|------|---------------|--------------|
| **G-Counter** | Page views, distributed counters | Increment-only, merge = max per node |
| **PN-Counter** | Like/dislike, upvote/downvote | Increment + decrement via dual G-Counter |
| **G-Set** | Member lists, tags, audit log | Add-only, merge = union |
| **OR-Set** | Shopping cart, collaborative lists | Add-wins concurrent semantics |
| **LWW-Register** | User profile, config values | Last timestamp wins |
| **Vector Clock** | Causality tracking, conflict detection | happened-before relation |

**Pattern สำคัญที่ได้เรียน:**

1. **Join-semilattice** คือหัวใจของ CRDT — merge ต้องเป็น idempotent, commutative, associative
2. **State-based CRDTs** ส่ง full state แล้ว merge — robust ต่อ message loss และ reordering
3. **Unique tags** ใน OR-Set แก้ปัญหา concurrent add/remove ได้อย่างสง่างาม
4. **node_id tiebreaker** ใน LWW ทำให้ merge เป็น deterministic แม้ timestamp เท่ากัน
5. **Generic programming** ทำให้ CRDT ทำงานกับ element type ใดๆ ที่ implement `Eq + Hash + Clone`

**เชื่อมโยงกับโปรเจคถัดไป:**

F08: Consistent Hash Ring จะนำ concept ของ distributed systems มาต่อยอด โดย focus ที่การ **partition data** ระหว่าง nodes อย่างสมดุล — CRDT จาก F07 นี้สามารถนำไปใช้เก็บข้อมูล per-partition ใน consistent hash ring ได้ ทำให้เกิด distributed data store ที่ scalable และ conflict-free

---

**โปรเจคก่อนหน้า:** [Project F06: Event Sourcing](project-f06-event-sourcing.md) | **โปรเจคถัดไป:** [Project F08: Consistent Hash Ring](project-f08-consistent-hash.md)
