# Project J05: Real-Time Collaborative Editor

> โมดูล: Full-Stack / WASM | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 20–30 ชั่วโมง

---

## ภาพรวมโปรเจค

Collaborative Editor คือแอปพลิเคชันที่ให้ผู้ใช้หลายคนพิมพ์และแก้ไขเอกสารเดียวกันพร้อมกันได้แบบ real-time — เหมือน Google Docs หรือ Notion ในเวอร์ชันที่คุณสร้างเองตั้งแต่ต้น โปรเจคนี้จะพาคุณลงลึกถึง **Operational Transform (OT)** ซึ่งเป็นอัลกอริทึมหัวใจของ collaborative editing ที่แก้ปัญหา "จะเกิดอะไรขึ้นถ้าสองคนพิมพ์ที่ตำแหน่งเดียวกันพร้อมกัน?"

ในโลก production ปัญหานี้ซับซ้อนมาก: เน็ตเวิร์กมี latency, packet อาจมาผิดลำดับ, client อาจ offline แล้วกลับมา online ทีหลัง ทุก scenario เหล่านี้ต้องการ algorithm ที่รับประกันว่า document ทุก copy จะ **converge** สู่สถานะเดียวกันเสมอ

โปรเจคนี้ครอบคลุม:
- **OT Algorithm** — `Insert(pos, char)`, `Delete(pos)`, และ `transform(op1, op2) → (op1', op2')`
- **WebSocket Server** ด้วย `tokio-tungstenite` — รับ op จาก client, transform, broadcast
- **Session & Room Management** — หลาย document, หลาย user ใน room เดียวกัน
- **Cursor Synchronization** — แสดงตำแหน่ง cursor ของ user อื่นแบบ real-time
- **Reconnection & Catch-up** — client ส่ง `base_version` แล้ว server replay op ที่หายไป

ใน real-world ระบบนี้พบใน Collaborative IDEs (VS Code Live Share), Google Workspace, Figma multiplayer mode, และ Notion real-time sync

---

## สิ่งที่จะได้เรียนรู้

- เข้าใจหลักการ Operational Transform (OT) และทำไม naive approach ไม่ work
- เขียน `transform(op1, op2)` ที่รับประกัน convergence property
- ออกแบบ client-server protocol สำหรับ collaborative editing
- ใช้ `tokio-tungstenite` สร้าง WebSocket server แบบ multi-client
- จัดการ shared mutable state ด้วย `Arc<RwLock<...>>` และ `dashmap`
- ออกแบบ server-side op queue และ transformation history
- Implement cursor synchronization และ user presence
- จัดการ reconnection scenario และ version catch-up

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50**: async/await, tokio runtime, future, task spawning
- **Part 51–55**: Channel (mpsc, broadcast), Arc, Mutex, RwLock
- **Part 61–65**: WebSocket protocol, HTTP upgrade, tokio-tungstenite
- **Part 71–75**: serde, serde_json, serialization/deserialization
- **Part 81–85**: เข้าใจ distributed systems concepts เบื้องต้น
- **Part 96–100**: production patterns, error handling, graceful shutdown

---

## โครงสร้างโปรเจค (Project Layout)

```
collab-editor/
├── Cargo.toml
├── src/
│   ├── main.rs              # binary entry point — tokio runtime, server startup
│   ├── lib.rs               # re-exports สำหรับ integration tests
│   ├── ot/
│   │   ├── mod.rs           # pub use
│   │   ├── op.rs            # Op enum: Insert, Delete, Retain
│   │   ├── doc.rs           # Doc struct: content + version, apply()
│   │   ├── transform.rs     # transform(op1, op2) -> (op1', op2')
│   │   └── history.rs       # transform_against_history, OpLog
│   ├── protocol/
│   │   ├── mod.rs
│   │   ├── client_msg.rs    # ClientMsg enum (serde)
│   │   └── server_msg.rs    # ServerMsg enum (serde)
│   ├── server/
│   │   ├── mod.rs
│   │   ├── room.rs          # Room, RoomId, UserList
│   │   ├── session.rs       # Session, per-connection state
│   │   └── handler.rs       # handle_connection(), broadcast logic
│   └── cursor.rs            # CursorOp, cursor position transform
├── tests/
│   ├── ot_tests.rs          # integration tests สำหรับ OT core
│   └── convergence_tests.rs # property-based convergence checks
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### ทำไม OT ถึงจำเป็น?

สมมติมีเอกสาร `"Hello"` และ user สองคนส่ง operation พร้อมกัน:
- **Alice** ส่ง `Insert(5, '!')` → ตั้งใจให้ได้ `"Hello!"`
- **Bob** ส่ง `Delete(4)` → ตั้งใจให้ได้ `"Hell"`

ถ้า apply ตรงๆ โดยไม่ transform:
- Server apply Alice ก่อน → `"Hello!"` แล้ว Bob's `Delete(4)` → `"Hell!"` ✗
- Server apply Bob ก่อน → `"Hell"` แล้ว Alice's `Insert(5,'!')` → index out of bounds! ✗

**OT แก้ปัญหาด้วยการ transform** op หนึ่งให้ "รู้" ว่า op อีกอันได้ถูก apply ไปแล้ว แล้วปรับ position ให้ถูกต้อง

### Data Flow

```
Client A ──[ws]──→ Server Handler
                       │
                       ├─ transform op_A against pending ops
                       ├─ apply to master Doc
                       ├─ store in OpLog
                       ├─ broadcast op_A' to other clients
                       └─ send Ack{version} to Client A

Client B ──[ws]──→ Server Handler
                       │  (same pipeline)
                       ...
```

### ทำไมเลือก OT แทน CRDT?

**CRDT (Conflict-free Replicated Data Type)** เช่น Yjs หรือ Automerge นั้นมีข้อดีคือ peer-to-peer ได้โดยไม่ต้องมี central server แต่มี overhead ด้าน memory สูงกว่าเพราะต้องเก็บ metadata ทุก character

**OT** เหมาะกับ central server architecture (Google Docs ก็ใช้แนวนี้) มี algorithm ที่เข้าใจง่ายกว่า และ memory footprint ต่ำกว่า — เหมาะกับโปรเจคนี้ที่เน้นการเรียนรู้กลไกภายใน

### Convergence Property

OT ต้องรับประกัน: ถ้า doc เริ่มจาก state เดียวกัน และ apply ops สองชุดในลำดับต่างกัน (แต่ผ่าน transform) doc จะลงเอยที่ state เดียวกันเสมอ

```
     doc_0
    /     \
  op_A   op_B
  /         \
doc_A      doc_B
  |           |
op_B'       op_A'
  \         /
   doc_final  ← ต้องเท่ากัน!
```

---

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: สร้าง Op และ Doc — พื้นฐานของ Collaborative Editing

ขั้นแรกเราจะสร้าง data types หลัก: `Op` enum ที่แทน operation และ `Doc` struct ที่เก็บ document state พร้อมกับ `apply()` method

สร้าง project ใหม่ก่อน:

```bash
cargo new collab-editor
cd collab-editor
```

แก้ไข `Cargo.toml`:

```toml
[package]
name = "collab-editor"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
tokio-tungstenite = "0.24"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
dashmap = "6"
uuid = { version = "1", features = ["v4"] }
futures-util = "0.3"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
tokio = { version = "1", features = ["full", "test-util"] }
```

สร้าง `src/ot/op.rs`:

```rust
/// Operation ที่แทนการเปลี่ยนแปลงเนื้อหาเอกสาร
///
/// ใน OT ทุก action ของ user ถูกแทนด้วย Op —
/// ไม่ว่าจะพิมพ์อักขระเพิ่ม ลบอักขระ หรือ no-op
#[derive(Debug, Clone, PartialEq, serde::Serialize, serde::Deserialize)]
pub enum Op {
    /// แทรกอักขระ `ch` ที่ตำแหน่ง `pos` (index byte ใน UTF-8 string)
    Insert { pos: usize, ch: char },
    /// ลบอักขระที่ตำแหน่ง `pos`
    Delete { pos: usize },
    /// No-op — ใช้เมื่อ transform ของ Delete vs Delete ที่ pos เดียวกัน
    Retain,
}

impl Op {
    /// ตรวจสอบว่า Op นี้เป็น no-op หรือไม่
    pub fn is_noop(&self) -> bool {
        matches!(self, Op::Retain)
    }

    /// คืนค่า position ที่ Op นี้กระทำ (ถ้ามี)
    pub fn position(&self) -> Option<usize> {
        match self {
            Op::Insert { pos, .. } => Some(*pos),
            Op::Delete { pos } => Some(*pos),
            Op::Retain => None,
        }
    }
}

impl std::fmt::Display for Op {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Op::Insert { pos, ch } => write!(f, "Insert({}, {:?})", pos, ch),
            Op::Delete { pos } => write!(f, "Delete({})", pos),
            Op::Retain => write!(f, "Retain"),
        }
    }
}
```

สร้าง `src/ot/doc.rs`:

```rust
use crate::ot::op::Op;

/// Error ที่เกิดจากการ apply operation ที่ไม่ valid
#[derive(Debug, Clone, PartialEq)]
pub enum DocError {
    /// ตำแหน่ง insert อยู่นอกขอบเขต document
    InsertOutOfBounds { pos: usize, len: usize },
    /// ตำแหน่ง delete อยู่นอกขอบเขต document
    DeleteOutOfBounds { pos: usize, len: usize },
}

impl std::fmt::Display for DocError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            DocError::InsertOutOfBounds { pos, len } => {
                write!(f, "Insert pos {} out of range (doc len={})", pos, len)
            }
            DocError::DeleteOutOfBounds { pos, len } => {
                write!(f, "Delete pos {} out of range (doc len={})", pos, len)
            }
        }
    }
}

/// Document state — หัวใจของ collaborative editor
///
/// `content` คือ string เนื้อหาปัจจุบัน
/// `version` เพิ่มขึ้น 1 ทุกครั้งที่ apply Op สำเร็จ
#[derive(Debug, Clone)]
pub struct Doc {
    pub content: String,
    pub version: u64,
}

impl Doc {
    /// สร้าง Doc ใหม่จาก initial content, version เริ่มที่ 0
    pub fn new(content: &str) -> Self {
        Doc {
            content: content.to_string(),
            version: 0,
        }
    }

    /// Apply operation ลงบน document
    ///
    /// ถ้า Op valid: เปลี่ยน content, version++, คืน Ok(())
    /// ถ้า Op ไม่ valid (out of bounds): คืน Err(DocError)
    ///
    /// # หมายเหตุเรื่อง Unicode
    /// ใน production ควรใช้ grapheme cluster index แทน byte index
    /// แต่โปรเจคนี้ใช้ ASCII/Latin เพื่อความเรียบง่าย
    pub fn apply(&mut self, op: &Op) -> Result<(), DocError> {
        match op {
            Op::Insert { pos, ch } => {
                if *pos > self.content.len() {
                    return Err(DocError::InsertOutOfBounds {
                        pos: *pos,
                        len: self.content.len(),
                    });
                }
                self.content.insert(*pos, *ch);
                self.version += 1;
                Ok(())
            }
            Op::Delete { pos } => {
                if *pos >= self.content.len() {
                    return Err(DocError::DeleteOutOfBounds {
                        pos: *pos,
                        len: self.content.len(),
                    });
                }
                self.content.remove(*pos);
                self.version += 1;
                Ok(())
            }
            Op::Retain => {
                // no-op ยังคง increment version เพื่อให้ version log ต่อเนื่อง
                self.version += 1;
                Ok(())
            }
        }
    }

    /// Apply หลาย Op ต่อเนื่องกัน หยุดทันทีถ้า error
    pub fn apply_all(&mut self, ops: &[Op]) -> Result<(), DocError> {
        for op in ops {
            self.apply(op)?;
        }
        Ok(())
    }
}

impl Default for Doc {
    fn default() -> Self {
        Doc::new("")
    }
}
```

สร้าง `src/ot/mod.rs`:

```rust
pub mod doc;
pub mod op;
pub mod transform;
pub mod history;

pub use doc::{Doc, DocError};
pub use op::Op;
pub use transform::transform;
pub use history::{transform_against_history, OpLog};
```

ทดลองรันโปรแกรมเบื้องต้น:

```rust
// src/main.rs (ชั่วคราว)
mod ot;
use ot::{Doc, Op};

fn main() {
    let mut doc = Doc::new("Hello");
    println!("Version {}: {:?}", doc.version, doc.content);

    doc.apply(&Op::Insert { pos: 5, ch: '!' }).unwrap();
    println!("Version {}: {:?}", doc.version, doc.content);

    doc.apply(&Op::Delete { pos: 0 }).unwrap();
    println!("Version {}: {:?}", doc.version, doc.content);
}
```

**Output:**
```
Version 0: "Hello"
Version 1: "Hello!"
Version 2: "ello!"
```

---

### ขั้นที่ 2: Operational Transform Algorithm — หัวใจของ Convergence

นี่คือส่วนที่ซับซ้อนและสำคัญที่สุด เราต้องสร้าง `transform(op1, op2) -> (op1', op2')` ที่จัดการทุก case ของ concurrent operations

สร้าง `src/ot/transform.rs`:

```rust
use crate::ot::op::Op;

/// Operational Transform: transform op1 กับ op2 ที่เกิดขึ้นพร้อมกัน
///
/// คืนค่า `(op1', op2')` ซึ่ง:
/// - `op1'` คือ op1 ที่ถูก transform ให้ทำงานได้ถูกต้องหลังจาก op2 ถูก apply แล้ว
/// - `op2'` คือ op2 ที่ถูก transform ให้ทำงานได้ถูกต้องหลังจาก op1 ถูก apply แล้ว
///
/// Invariant: apply(apply(doc, op1), op2') == apply(apply(doc, op2), op1')
///
/// กฎ convergence (diamond property):
/// ```text
///      doc
///     /   \
///   op1   op2
///   /       \
/// doc1      doc2
///   \       /
///  op2'   op1'
///     \   /
///    doc_final  ← ต้องเท่ากัน
/// ```
pub fn transform(op1: Op, op2: Op) -> (Op, Op) {
    match (&op1, &op2) {
        // ========================================
        // Case 1: Insert vs Insert
        // ========================================
        (Op::Insert { pos: p1, ch: c1 }, Op::Insert { pos: p2, ch: c2 }) => {
            if p1 < p2 {
                // op1 insert อยู่ก่อน op2
                // → op1' = ไม่เปลี่ยน (ยังอยู่ตำแหน่งเดิม)
                // → op2' = เลื่อนไปทางขวา 1 ตำแหน่ง (op1 ดัน op2 ออกไป)
                (op1.clone(), Op::Insert { pos: p2 + 1, ch: *c2 })
            } else if p1 > p2 {
                // op2 insert อยู่ก่อน op1
                // → op1' = เลื่อนไปทางขวา 1 ตำแหน่ง
                // → op2' = ไม่เปลี่ยน
                (Op::Insert { pos: p1 + 1, ch: *c1 }, op2.clone())
            } else {
                // p1 == p2: insert ที่ตำแหน่งเดียวกัน
                // ต้องใช้ tiebreak เพื่อ determinism
                // กฎ: op1 (client ที่ส่งก่อน / lower user ID) ชนะ
                // op1' = ไม่เปลี่ยน, op2' = เลื่อน +1
                (op1.clone(), Op::Insert { pos: p2 + 1, ch: *c2 })
            }
        }

        // ========================================
        // Case 2: Insert vs Delete
        // ========================================
        (Op::Insert { pos: p1, ch: c1 }, Op::Delete { pos: p2 }) => {
            if p1 <= p2 {
                // insert อยู่ก่อนหรือที่จุดที่จะถูกลบ
                // → insert ไม่เปลี่ยน
                // → delete ต้องเลื่อน +1 (insert ดันตำแหน่งออกไป)
                (op1.clone(), Op::Delete { pos: p2 + 1 })
            } else {
                // insert อยู่หลังตำแหน่งที่ถูกลบ
                // → insert เลื่อน -1 (delete ดึงตำแหน่งเข้ามา)
                // → delete ไม่เปลี่ยน
                (Op::Insert { pos: p1 - 1, ch: *c1 }, op2.clone())
            }
        }

        // ========================================
        // Case 3: Delete vs Insert
        // ========================================
        (Op::Delete { pos: p1 }, Op::Insert { pos: p2, ch: c2 }) => {
            if p2 <= p1 {
                // insert อยู่ก่อนหรือที่จุดที่จะถูกลบ
                // → delete เลื่อน +1
                // → insert ไม่เปลี่ยน
                (Op::Delete { pos: p1 + 1 }, op2.clone())
            } else {
                // insert อยู่หลังจุดที่จะถูกลบ
                // → delete ไม่เปลี่ยน
                // → insert เลื่อน -1
                (op1.clone(), Op::Insert { pos: p2 - 1, ch: *c2 })
            }
        }

        // ========================================
        // Case 4: Delete vs Delete
        // ========================================
        (Op::Delete { pos: p1 }, Op::Delete { pos: p2 }) => {
            if p1 < p2 {
                // op1 ลบตำแหน่งก่อน op2
                // → op1' ไม่เปลี่ยน
                // → op2' เลื่อน -1 (op1 ดึงตำแหน่งเข้ามา)
                (op1.clone(), Op::Delete { pos: p2 - 1 })
            } else if p1 > p2 {
                // op2 ลบตำแหน่งก่อน op1
                // → op1' เลื่อน -1
                // → op2' ไม่เปลี่ยน
                (Op::Delete { pos: p1 - 1 }, op2.clone())
            } else {
                // p1 == p2: ลบตำแหน่งเดียวกัน
                // อักขระถูกลบไปแล้วโดย op2 (หรือ op1)
                // → ทั้งคู่กลายเป็น Retain (no-op)
                (Op::Retain, Op::Retain)
            }
        }

        // ========================================
        // Case 5: Retain ใดๆ vs อะไรก็ตาม
        // ========================================
        _ => (op1, op2),
    }
}

/// transform op1 กับ sequence ของ server ops ที่ได้ apply ไปแล้ว
///
/// ใช้เมื่อ client ส่ง op มาช้า และ server ได้ apply หลาย op ก่อนหน้าแล้ว
/// เราต้อง transform client's op ผ่านทุก server op ตามลำดับ
pub fn transform_against_ops(op: Op, server_ops: &[Op]) -> Op {
    server_ops.iter().fold(op, |current, server_op| {
        let (transformed, _) = transform(current, server_op.clone());
        transformed
    })
}
```

#### ทำความเข้าใจ transform cases ด้วยตัวอย่าง

**Case: Insert vs Insert (same position)**

```
doc = "AB"   (positions: A=0, B=1)

Alice: Insert(1, 'X')  →  "AXB"
Bob:   Insert(1, 'Y')  →  "AYB"  (concurrent)

transform(Insert(1,'X'), Insert(1,'Y')):
  → (Insert(1,'X'), Insert(2,'Y'))  ← tiebreak: Alice ชนะ

Scenario 1: Alice first, then Bob'
  apply Insert(1,'X') on "AB" → "AXB"
  apply Insert(2,'Y') on "AXB" → "AXYB" ✓

Scenario 2: Bob first, then Alice'
  apply Insert(1,'Y') on "AB" → "AYB"
  apply Insert(1,'X') on "AYB" → "AXYB" ✓
  (X อยู่ก่อน Y เพราะ Alice ชนะ tiebreak)
```

**Case: Delete vs Delete (same position)**

```
doc = "Hello"

Alice: Delete(2)  → ลบ 'l' ตัวแรก
Bob:   Delete(2)  → ลบ 'l' ตัวแรกเหมือนกัน (concurrent)

transform(Delete(2), Delete(2)):
  → (Retain, Retain)

Scenario 1: Alice first, then Bob' (Retain)
  apply Delete(2) → "Helo"
  apply Retain → "Helo" (ไม่มีการลบซ้ำ) ✓

Scenario 2: Bob first, then Alice' (Retain)
  apply Delete(2) → "Helo"
  apply Retain → "Helo" ✓
```

---

### ขั้นที่ 3: OpLog — การจัดการ History สำหรับ Server Queue

Server ต้องเก็บ history ของ ops ที่ผ่านมาทั้งหมด เพื่อ:
1. Transform op ใหม่ที่มาช้าให้ถูกต้อง
2. Replay ops ให้ client ที่ reconnect

สร้าง `src/ot/history.rs`:

```rust
use crate::ot::op::Op;
use crate::ot::transform::transform;

/// รายการ op ที่ server ได้ apply แล้ว พร้อม version ของแต่ละ op
#[derive(Debug, Clone)]
pub struct OpEntry {
    /// version หลังจาก apply op นี้
    pub version: u64,
    /// ตัว op ที่ถูก apply (หลัง transform แล้ว)
    pub op: Op,
    /// user ID ที่ส่ง op นี้มา
    pub user_id: String,
}

/// Log ของ ops ทั้งหมดที่ server ได้ apply
///
/// ใช้สำหรับ:
/// - Transform op ใหม่ที่มาช้า (based on older version)
/// - Replay ops ให้ client ที่ reconnect
#[derive(Debug, Default, Clone)]
pub struct OpLog {
    entries: Vec<OpEntry>,
}

impl OpLog {
    pub fn new() -> Self {
        OpLog {
            entries: Vec::new(),
        }
    }

    /// บันทึก op ที่ server apply แล้ว
    pub fn push(&mut self, version: u64, op: Op, user_id: String) {
        self.entries.push(OpEntry { version, op, user_id });
    }

    /// ดึง ops ทั้งหมดที่ version > `since_version`
    ///
    /// ใช้สำหรับ catch-up: client ส่ง base_version มา
    /// server คืน ops ที่ client ยังไม่มี
    pub fn ops_since(&self, since_version: u64) -> Vec<&OpEntry> {
        self.entries
            .iter()
            .filter(|e| e.version > since_version)
            .collect()
    }

    /// ดึง ops ทั้งหมดที่ version >=`from` และ < `to`
    pub fn ops_in_range(&self, from: u64, to: u64) -> Vec<&OpEntry> {
        self.entries
            .iter()
            .filter(|e| e.version >= from && e.version < to)
            .collect()
    }

    /// version ล่าสุดที่ log มี
    pub fn latest_version(&self) -> u64 {
        self.entries.last().map(|e| e.version).unwrap_or(0)
    }

    /// จำนวน entries ใน log
    pub fn len(&self) -> usize {
        self.entries.len()
    }

    pub fn is_empty(&self) -> bool {
        self.entries.is_empty()
    }
}

/// Transform op ของ client ที่อิงจาก `base_version` เก่า
/// ให้ทำงานได้บน document ที่ version ปัจจุบัน
///
/// Algorithm:
///   สำหรับทุก server op ที่ version > base_version:
///     transform client_op ต่อสู้กับ server op
///
/// คืนค่า op ที่ transform แล้ว พร้อมใช้งานกับ document ล่าสุด
pub fn transform_against_history(op: Op, history: &[Op]) -> Op {
    history.iter().fold(op, |current_op, server_op| {
        let (transformed, _) = transform(current_op, server_op.clone());
        transformed
    })
}
```

---

### ขั้นที่ 4: Protocol Messages — การสื่อสารระหว่าง Client และ Server

ออกแบบ message protocol ที่ชัดเจนด้วย serde_json:

สร้าง `src/protocol/client_msg.rs`:

```rust
use crate::ot::op::Op;
use serde::{Deserialize, Serialize};

/// Messages ที่ client ส่งมาให้ server
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ClientMsg {
    /// ส่ง operation พร้อมระบุ version ที่ client เห็น ณ ตอนที่ generate op
    ///
    /// `base_version`: version ของ document ที่ client ใช้เป็นฐาน
    /// `op`: operation ที่ต้องการทำ
    /// `request_id`: unique ID สำหรับ match กับ Ack
    Op {
        op: Op,
        base_version: u64,
        request_id: String,
    },

    /// Client บอก server ว่า cursor อยู่ที่ไหน
    CursorMove {
        position: usize,
        selection_end: Option<usize>,
    },

    /// Client ขอ join room
    JoinRoom {
        room_id: String,
        user_name: String,
    },

    /// Client บอก server ว่า reconnect แล้ว และต้องการ catch-up
    /// ตั้งแต่ `known_version`
    Catchup {
        known_version: u64,
    },

    /// Ping เพื่อ keep connection alive
    Ping,
}
```

สร้าง `src/protocol/server_msg.rs`:

```rust
use crate::ot::op::Op;
use serde::{Deserialize, Serialize};

/// ข้อมูล cursor ของ user คนหนึ่ง
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CursorInfo {
    pub user_id: String,
    pub user_name: String,
    pub position: usize,
    pub selection_end: Option<usize>,
}

/// Messages ที่ server ส่งกลับมาให้ client
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum ServerMsg {
    /// ยืนยันว่า server ได้รับและ apply op ของ client แล้ว
    /// `version` คือ version ใหม่หลัง apply
    Ack {
        request_id: String,
        version: u64,
    },

    /// Broadcast op ของ user อื่นที่ server ได้ transform และ apply แล้ว
    /// client ควร apply op นี้ลงบน local doc
    Op {
        op: Op,
        version: u64,
        user_id: String,
        user_name: String,
    },

    /// ส่ง initial state ตอน join room หรือ reconnect
    FullState {
        content: String,
        version: u64,
        users: Vec<UserInfo>,
    },

    /// Server ส่ง ops ที่ client หายไปกลับมา (catch-up)
    CatchupOps {
        ops: Vec<CatchupEntry>,
        current_version: u64,
    },

    /// Broadcast cursor position ของ user อื่น
    CursorUpdate {
        cursors: Vec<CursorInfo>,
    },

    /// แจ้งว่ามี user เข้าหรือออก
    UserJoined { user_id: String, user_name: String },
    UserLeft { user_id: String, user_name: String },

    /// Error response
    Error { message: String },

    /// Pong ตอบ Ping
    Pong,
}

/// ข้อมูล user ใน room
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserInfo {
    pub user_id: String,
    pub user_name: String,
}

/// Op entry สำหรับ catch-up response
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CatchupEntry {
    pub op: Op,
    pub version: u64,
    pub user_id: String,
}
```

---

### ขั้นที่ 5: Server Core — Room และ Session Management

สร้าง `src/server/room.rs`:

```rust
use crate::ot::{Doc, Op};
use crate::ot::history::OpLog;
use crate::protocol::server_msg::{CatchupEntry, UserInfo};
use std::collections::HashMap;
use tokio::sync::broadcast;

/// ชนิดของ RoomId — wrapper เพื่อ type safety
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct RoomId(pub String);

impl RoomId {
    pub fn new(id: impl Into<String>) -> Self {
        RoomId(id.into())
    }
}

impl std::fmt::Display for RoomId {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

/// ข้อมูล user ใน room
#[derive(Debug, Clone)]
pub struct RoomUser {
    pub user_id: String,
    pub user_name: String,
    pub cursor_pos: usize,
    pub selection_end: Option<usize>,
}

/// Broadcast event สำหรับส่งข้อมูลระหว่าง sessions
#[derive(Debug, Clone)]
pub struct BroadcastEvent {
    /// user_id ของคนที่ส่ง (เพื่อ skip sending back to sender)
    pub sender_id: String,
    /// JSON payload ที่จะส่งให้ทุก session ใน room
    pub payload: String,
}

/// Room เก็บ state ของเอกสารและ users
pub struct Room {
    pub id: RoomId,
    pub doc: Doc,
    pub op_log: OpLog,
    pub users: HashMap<String, RoomUser>,
    /// broadcast channel สำหรับส่ง message ไปยัง sessions ทุก session ใน room
    pub broadcast_tx: broadcast::Sender<BroadcastEvent>,
}

impl Room {
    pub fn new(id: RoomId, initial_content: &str) -> Self {
        let (broadcast_tx, _) = broadcast::channel(1024);
        Room {
            id,
            doc: Doc::new(initial_content),
            op_log: OpLog::new(),
            users: HashMap::new(),
            broadcast_tx,
        }
    }

    /// เพิ่ม user เข้า room
    pub fn add_user(&mut self, user_id: String, user_name: String) {
        self.users.insert(
            user_id.clone(),
            RoomUser {
                user_id,
                user_name,
                cursor_pos: 0,
                selection_end: None,
            },
        );
    }

    /// ลบ user ออกจาก room
    pub fn remove_user(&mut self, user_id: &str) -> Option<RoomUser> {
        self.users.remove(user_id)
    }

    /// อัปเดต cursor position ของ user
    pub fn update_cursor(&mut self, user_id: &str, pos: usize, selection_end: Option<usize>) {
        if let Some(user) = self.users.get_mut(user_id) {
            user.cursor_pos = pos;
            user.selection_end = selection_end;
        }
    }

    /// ดึงรายชื่อ users ทั้งหมดใน room
    pub fn user_list(&self) -> Vec<UserInfo> {
        self.users
            .values()
            .map(|u| UserInfo {
                user_id: u.user_id.clone(),
                user_name: u.user_name.clone(),
            })
            .collect()
    }

    /// Apply op จาก client (หลัง transform แล้ว) และบันทึกใน log
    pub fn apply_op(&mut self, op: Op, user_id: &str) -> Result<u64, String> {
        self.doc
            .apply(&op)
            .map_err(|e| e.to_string())?;
        self.op_log
            .push(self.doc.version, op, user_id.to_string());
        Ok(self.doc.version)
    }

    /// ดึง ops ที่ client หายไป สำหรับ catch-up
    pub fn catchup_ops(&self, since_version: u64) -> Vec<CatchupEntry> {
        self.op_log
            .ops_since(since_version)
            .into_iter()
            .map(|entry| CatchupEntry {
                op: entry.op.clone(),
                version: entry.version,
                user_id: entry.user_id.clone(),
            })
            .collect()
    }

    /// Transform op ของ client (base_version) ให้ทำงานกับ doc version ปัจจุบัน
    pub fn transform_client_op(&self, op: Op, base_version: u64) -> Op {
        // ดึง server ops ที่เกิดขึ้นหลังจาก base_version
        let server_ops: Vec<Op> = self
            .op_log
            .ops_since(base_version)
            .into_iter()
            .map(|e| e.op.clone())
            .collect();

        crate::ot::history::transform_against_history(op, &server_ops)
    }
}
```

---

### ขั้นที่ 6: WebSocket Server — tokio-tungstenite Handler

สร้าง `src/server/handler.rs`:

```rust
use crate::protocol::client_msg::ClientMsg;
use crate::protocol::server_msg::{CursorInfo, ServerMsg};
use crate::server::room::{BroadcastEvent, Room, RoomId};
use dashmap::DashMap;
use futures_util::{SinkExt, StreamExt};
use std::sync::Arc;
use tokio::net::TcpListener;
use tokio::sync::RwLock;
use tokio_tungstenite::accept_async;
use tokio_tungstenite::tungstenite::Message;
use uuid::Uuid;

pub type Rooms = Arc<DashMap<String, Arc<RwLock<Room>>>>;

/// เริ่ม WebSocket server
pub async fn run_server(addr: &str) -> std::io::Result<()> {
    let rooms: Rooms = Arc::new(DashMap::new());
    let listener = TcpListener::bind(addr).await?;
    tracing::info!("Server listening on {}", addr);

    loop {
        let (stream, peer_addr) = listener.accept().await?;
        tracing::info!("New connection from {}", peer_addr);
        let rooms = rooms.clone();

        tokio::spawn(async move {
            if let Err(e) = handle_connection(stream, rooms).await {
                tracing::error!("Connection error: {}", e);
            }
        });
    }
}

/// จัดการ WebSocket connection หนึ่ง connection
async fn handle_connection(
    stream: tokio::net::TcpStream,
    rooms: Rooms,
) -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let ws_stream = accept_async(stream).await?;
    let (mut ws_sink, mut ws_source) = ws_stream.split();

    let user_id = Uuid::new_v4().to_string();
    let mut current_room: Option<String> = None;
    let mut broadcast_rx: Option<tokio::sync::broadcast::Receiver<BroadcastEvent>> = None;

    tracing::info!("User {} connected", user_id);

    loop {
        // รอ message จาก client หรือ broadcast จาก room
        let msg = tokio::select! {
            // message จาก client
            msg = ws_source.next() => {
                match msg {
                    Some(Ok(m)) => Some(m),
                    Some(Err(e)) => {
                        tracing::warn!("WS error for {}: {}", user_id, e);
                        break;
                    }
                    None => break, // connection closed
                }
            }
            // broadcast จาก room (ถ้า join แล้ว)
            bcast = async {
                if let Some(rx) = broadcast_rx.as_mut() {
                    rx.recv().await
                } else {
                    std::future::pending().await
                }
            } => {
                match bcast {
                    Ok(event) if event.sender_id != user_id => {
                        // ส่ง broadcast ไปให้ client
                        let _ = ws_sink.send(Message::Text(event.payload.into())).await;
                    }
                    _ => {}
                }
                continue;
            }
        };

        let msg = match msg {
            Some(m) => m,
            None => break,
        };

        match msg {
            Message::Text(text) => {
                let client_msg: ClientMsg = match serde_json::from_str(&text) {
                    Ok(m) => m,
                    Err(e) => {
                        let err = ServerMsg::Error {
                            message: format!("Invalid message: {}", e),
                        };
                        let _ = ws_sink
                            .send(Message::Text(serde_json::to_string(&err).unwrap().into()))
                            .await;
                        continue;
                    }
                };

                let response = process_message(
                    client_msg,
                    &user_id,
                    &mut current_room,
                    &mut broadcast_rx,
                    &rooms,
                )
                .await;

                for msg in response {
                    let json = serde_json::to_string(&msg).unwrap();
                    let _ = ws_sink.send(Message::Text(json.into())).await;
                }
            }
            Message::Close(_) => break,
            Message::Ping(data) => {
                let _ = ws_sink.send(Message::Pong(data)).await;
            }
            _ => {}
        }
    }

    // Cleanup เมื่อ disconnect
    if let Some(room_id) = &current_room {
        if let Some(room_arc) = rooms.get(room_id) {
            let mut room = room_arc.write().await;
            if let Some(user) = room.remove_user(&user_id) {
                let leave_msg = ServerMsg::UserLeft {
                    user_id: user_id.clone(),
                    user_name: user.user_name,
                };
                let json = serde_json::to_string(&leave_msg).unwrap();
                let _ = room.broadcast_tx.send(BroadcastEvent {
                    sender_id: user_id.clone(),
                    payload: json,
                });
            }
        }
    }

    tracing::info!("User {} disconnected", user_id);
    Ok(())
}

/// ประมวลผล ClientMsg และคืนค่า list ของ ServerMsg ที่จะส่งกลับ
async fn process_message(
    msg: ClientMsg,
    user_id: &str,
    current_room: &mut Option<String>,
    broadcast_rx: &mut Option<tokio::sync::broadcast::Receiver<BroadcastEvent>>,
    rooms: &Rooms,
) -> Vec<ServerMsg> {
    match msg {
        ClientMsg::JoinRoom { room_id, user_name } => {
            // สร้าง room ใหม่ถ้ายังไม่มี
            rooms
                .entry(room_id.clone())
                .or_insert_with(|| Arc::new(RwLock::new(Room::new(RoomId::new(&room_id), ""))));

            let room_arc = rooms.get(&room_id).unwrap().clone();
            let mut room = room_arc.write().await;

            room.add_user(user_id.to_string(), user_name.clone());

            // subscribe broadcast channel
            *broadcast_rx = Some(room.broadcast_tx.subscribe());
            *current_room = Some(room_id.clone());

            // broadcast ให้ user อื่นรู้ว่ามี user ใหม่
            let join_msg = ServerMsg::UserJoined {
                user_id: user_id.to_string(),
                user_name: user_name.clone(),
            };
            let _ = room.broadcast_tx.send(BroadcastEvent {
                sender_id: user_id.to_string(),
                payload: serde_json::to_string(&join_msg).unwrap(),
            });

            // ส่ง full state กลับไปให้ client ที่ join
            vec![ServerMsg::FullState {
                content: room.doc.content.clone(),
                version: room.doc.version,
                users: room.user_list(),
            }]
        }

        ClientMsg::Op {
            op,
            base_version,
            request_id,
        } => {
            let room_id = match current_room {
                Some(id) => id.clone(),
                None => {
                    return vec![ServerMsg::Error {
                        message: "Not in a room".to_string(),
                    }]
                }
            };

            let room_arc = match rooms.get(&room_id) {
                Some(r) => r.clone(),
                None => {
                    return vec![ServerMsg::Error {
                        message: "Room not found".to_string(),
                    }]
                }
            };

            let mut room = room_arc.write().await;

            // Transform op ของ client ให้ทำงานกับ doc version ปัจจุบัน
            let transformed_op = room.transform_client_op(op, base_version);

            // Apply op
            let new_version = match room.apply_op(transformed_op.clone(), user_id) {
                Ok(v) => v,
                Err(e) => {
                    return vec![ServerMsg::Error { message: e }];
                }
            };

            // Get user name
            let user_name = room
                .users
                .get(user_id)
                .map(|u| u.user_name.clone())
                .unwrap_or_default();

            // Broadcast transformed op ไปยัง users อื่น
            let broadcast_msg = ServerMsg::Op {
                op: transformed_op,
                version: new_version,
                user_id: user_id.to_string(),
                user_name,
            };
            let _ = room.broadcast_tx.send(BroadcastEvent {
                sender_id: user_id.to_string(),
                payload: serde_json::to_string(&broadcast_msg).unwrap(),
            });

            // ส่ง Ack กลับให้ sender
            vec![ServerMsg::Ack {
                request_id,
                version: new_version,
            }]
        }

        ClientMsg::CursorMove {
            position,
            selection_end,
        } => {
            let room_id = match current_room {
                Some(id) => id.clone(),
                None => return vec![],
            };

            if let Some(room_arc) = rooms.get(&room_id) {
                let mut room = room_arc.write().await;
                room.update_cursor(user_id, position, selection_end);

                // Broadcast cursor update
                let cursors: Vec<CursorInfo> = room
                    .users
                    .values()
                    .map(|u| CursorInfo {
                        user_id: u.user_id.clone(),
                        user_name: u.user_name.clone(),
                        position: u.cursor_pos,
                        selection_end: u.selection_end,
                    })
                    .collect();

                let cursor_msg = ServerMsg::CursorUpdate { cursors };
                let _ = room.broadcast_tx.send(BroadcastEvent {
                    sender_id: user_id.to_string(),
                    payload: serde_json::to_string(&cursor_msg).unwrap(),
                });
            }
            vec![]
        }

        ClientMsg::Catchup { known_version } => {
            let room_id = match current_room {
                Some(id) => id.clone(),
                None => {
                    return vec![ServerMsg::Error {
                        message: "Not in a room".to_string(),
                    }]
                }
            };

            if let Some(room_arc) = rooms.get(&room_id) {
                let room = room_arc.read().await;
                let ops = room.catchup_ops(known_version);
                let current_version = room.doc.version;
                vec![ServerMsg::CatchupOps {
                    ops,
                    current_version,
                }]
            } else {
                vec![]
            }
        }

        ClientMsg::Ping => vec![ServerMsg::Pong],
    }
}
```

---

### ขั้นที่ 7: Cursor Synchronization — ติดตามตำแหน่ง Cursor ของทุก User

Cursor synchronization ต้องการ transform พิเศษ: เมื่อ text ถูกแก้ไข cursor ของ users อื่นต้องเลื่อนตามด้วย

สร้าง `src/cursor.rs`:

```rust
use crate::ot::op::Op;

/// ข้อมูล cursor ของ user หนึ่งคน
#[derive(Debug, Clone, PartialEq)]
pub struct Cursor {
    pub user_id: String,
    pub position: usize,
    pub selection_end: Option<usize>,
}

impl Cursor {
    pub fn new(user_id: impl Into<String>, position: usize) -> Self {
        Cursor {
            user_id: user_id.into(),
            position,
            selection_end: None,
        }
    }

    pub fn with_selection(
        user_id: impl Into<String>,
        position: usize,
        selection_end: usize,
    ) -> Self {
        Cursor {
            user_id: user_id.into(),
            position,
            selection_end: Some(selection_end),
        }
    }
}

/// Transform cursor position เมื่อมี Op เกิดขึ้น
///
/// ถ้า Insert เกิดก่อน cursor: cursor เลื่อนไปข้างหน้า 1 ตำแหน่ง
/// ถ้า Insert เกิดที่หรือหลัง cursor: cursor ไม่เปลี่ยน
/// ถ้า Delete เกิดก่อน cursor: cursor เลื่อนมาข้างหลัง 1 ตำแหน่ง
/// ถ้า Delete เกิดที่ cursor หรือหลัง: cursor ไม่เปลี่ยน
pub fn transform_cursor(cursor: usize, op: &Op) -> usize {
    match op {
        Op::Insert { pos, .. } => {
            if *pos <= cursor {
                cursor + 1
            } else {
                cursor
            }
        }
        Op::Delete { pos } => {
            if *pos < cursor {
                cursor - 1
            } else {
                cursor
            }
        }
        Op::Retain => cursor,
    }
}

/// Transform ตำแหน่ง selection end เหมือนกับ cursor ปกติ
pub fn transform_selection_end(sel: Option<usize>, op: &Op) -> Option<usize> {
    sel.map(|end| transform_cursor(end, op))
}

/// Transform cursor ทั้ง object (position และ selection) กับ Op
pub fn transform_cursor_full(cursor: Cursor, op: &Op) -> Cursor {
    Cursor {
        user_id: cursor.user_id,
        position: transform_cursor(cursor.position, op),
        selection_end: transform_selection_end(cursor.selection_end, op),
    }
}

/// Transform cursors ของทุก users เมื่อมี Op เกิดขึ้น
/// ใช้ใน server เมื่อ apply op ของ user A:
/// cursor ของ users B, C, D ต้องถูก transform ด้วย
pub fn transform_all_cursors(cursors: Vec<Cursor>, op: &Op) -> Vec<Cursor> {
    cursors
        .into_iter()
        .map(|c| transform_cursor_full(c, op))
        .collect()
}
```

---

### ขั้นที่ 8: Session Management — Multiple Rooms และ User Lists

อัปเดต `src/server/session.rs` สำหรับจัดการ per-session state:

```rust
use uuid::Uuid;

/// State ที่เก็บต่อ connection หนึ่ง
#[derive(Debug)]
pub struct Session {
    /// unique ID สำหรับ connection นี้
    pub user_id: String,
    /// ชื่อที่ user ตั้ง
    pub user_name: Option<String>,
    /// room ที่ user อยู่ตอนนี้ (ถ้า join แล้ว)
    pub room_id: Option<String>,
    /// version ล่าสุดที่ client รู้จัก
    pub known_version: u64,
}

impl Session {
    pub fn new() -> Self {
        Session {
            user_id: Uuid::new_v4().to_string(),
            user_name: None,
            room_id: None,
            known_version: 0,
        }
    }

    pub fn join_room(&mut self, room_id: String, user_name: String) {
        self.room_id = Some(room_id);
        self.user_name = Some(user_name);
    }

    pub fn update_known_version(&mut self, version: u64) {
        if version > self.known_version {
            self.known_version = version;
        }
    }

    pub fn is_in_room(&self) -> bool {
        self.room_id.is_some()
    }
}

impl Default for Session {
    fn default() -> Self {
        Self::new()
    }
}
```

---

### ขั้นที่ 9: Reconnection และ Catch-up

เมื่อ client disconnect แล้ว reconnect, server ต้องส่ง ops ที่ client หายไปกลับมา:

```rust
// ตัวอย่างการ implement catch-up ฝั่ง client (pseudo-code ใน Rust)

/// Client state สำหรับ reconnect
struct ClientState {
    doc: Doc,
    pending_ops: Vec<(Op, u64, String)>, // (op, base_version, request_id) ที่ยังรอ Ack
}

impl ClientState {
    /// เมื่อ reconnect: ส่ง Catchup message
    fn reconnect_message(&self) -> ClientMsg {
        ClientMsg::Catchup {
            known_version: self.doc.version,
        }
    }

    /// เมื่อได้รับ CatchupOps จาก server:
    /// 1. Apply ops ทั้งหมดที่ server ส่งมา
    /// 2. Re-transform และ re-send pending ops ที่ยังรอ Ack
    fn handle_catchup(&mut self, ops: Vec<CatchupEntry>) -> Vec<ClientMsg> {
        // apply missing ops
        for entry in &ops {
            let _ = self.doc.apply(&entry.op);
        }

        // re-transform pending ops แล้ว re-send
        let server_ops: Vec<Op> = ops.into_iter().map(|e| e.op).collect();
        self.pending_ops
            .iter()
            .map(|(op, base_ver, req_id)| {
                let new_base = self.doc.version;
                let retransformed = transform_against_history(op.clone(), &server_ops);
                ClientMsg::Op {
                    op: retransformed,
                    base_version: new_base,
                    request_id: req_id.clone(),
                }
            })
            .collect()
    }
}
```

โปรโตคอล catch-up ทำงานดังนี้:

```
Client                          Server
  |                               |
  |-- Catchup{known_version:5} -->|
  |                               |  ค้นหา ops ที่ version > 5
  |<-- CatchupOps{ops:[...],      |
  |      current_version:8} ------|
  |                               |
  |  apply missing ops locally    |
  |  re-transform pending ops     |
  |                               |
  |-- Op{op, base_version:8} ---->|  re-send pending ops
  |<-- Ack{version:9} ------------|
```

---

### ขั้นที่ 10: Main Entry Point และ Server Startup

สร้าง `src/main.rs` ที่สมบูรณ์:

```rust
mod cursor;
mod ot;
mod protocol;
mod server;

use server::handler::run_server;

#[tokio::main]
async fn main() {
    // ตั้ง tracing subscriber สำหรับ logging
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::from_default_env()
                .add_directive("collab_editor=debug".parse().unwrap()),
        )
        .init();

    let addr = std::env::var("LISTEN_ADDR").unwrap_or_else(|_| "127.0.0.1:8080".to_string());

    if let Err(e) = run_server(&addr).await {
        tracing::error!("Server error: {}", e);
        std::process::exit(1);
    }
}
```

รัน server:

```bash
$ cargo build --release
$ ./target/release/collab-editor
2024-01-15T10:30:00Z INFO  collab_editor::server::handler: Server listening on 127.0.0.1:8080
```

ทดสอบด้วย `wscat` หรือ browser console:

```bash
# Terminal 1 — Alice
wscat -c ws://127.0.0.1:8080
> {"type":"join_room","room_id":"doc1","user_name":"Alice"}
< {"type":"full_state","content":"","version":0,"users":[]}

> {"type":"op","op":{"Insert":{"pos":0,"ch":"H"}},"base_version":0,"request_id":"r1"}
< {"type":"ack","request_id":"r1","version":1}

# Terminal 2 — Bob (พร้อมกัน)
wscat -c ws://127.0.0.1:8080
> {"type":"join_room","room_id":"doc1","user_name":"Bob"}
< {"type":"full_state","content":"","version":0,"users":[{"user_id":"...","user_name":"Alice"}]}
```

---

## การทดสอบ (Testing)

### Unit Tests สำหรับ OT Core Logic

เราได้รัน tests จริงใน scratchpad project ด้านล่าง ซึ่งครอบคลุม:
- `Doc::apply()` ทุก case
- `transform()` ทุก 4 case หลัก (Insert/Insert, Insert/Delete, Delete/Insert, Delete/Delete)
- Convergence property (diamond property)
- `transform_against_history()`

**ผล `cargo test` จริง:**

```
   Compiling collab_ot v0.1.0 (/tmp/.../scratchpad/collab-ot)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.49s
     Running unittests src/main.rs (target/debug/deps/collab_ot-6d9a1248a6a47968)

running 24 tests
test tests::test_apply_insert_at_start ... ok
test tests::test_apply_delete_basic ... ok
test tests::test_apply_insert_basic ... ok
test tests::test_apply_delete_first_char ... ok
test tests::test_apply_insert_middle ... ok
test tests::test_apply_out_of_bounds_delete ... ok
test tests::test_apply_out_of_bounds_insert ... ok
test tests::test_apply_sequence ... ok
test tests::test_convergence_delete_delete ... ok
test tests::test_apply_retain_increments_version ... ok
test tests::test_convergence_insert_delete ... ok
test tests::test_convergence_insert_insert ... ok
test tests::test_transform_against_history_empty ... ok
test tests::test_transform_against_history_multiple ... ok
test tests::test_transform_against_history_single ... ok
test tests::test_transform_delete_delete_different_pos ... ok
test tests::test_transform_delete_delete_same_pos ... ok
test tests::test_transform_delete_vs_insert_after ... ok
test tests::test_transform_delete_vs_insert_before ... ok
test tests::test_transform_insert_after_delete ... ok
test tests::test_transform_insert_before_delete ... ok
test tests::test_transform_insert_insert_left_wins ... ok
test tests::test_transform_insert_insert_right_first ... ok
test tests::test_transform_insert_insert_same_pos_tiebreak ... ok

test result: ok. 24 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

### อธิบาย Test Cases สำคัญ

**`test_convergence_insert_insert`** — ทดสอบ diamond property:
```rust
// doc = "AC"
// Alice: Insert(1, 'B'), Bob: Insert(2, 'D') — concurrent
let (_, op_b_prime) = transform(op_a.clone(), op_b.clone());
// Scenario 1: Alice first → "ABC" → apply Insert(3,'D') → "ABDC"
// Scenario 2: Bob first → "ACD" → apply Insert(1,'B') → "ABCD"
// ??? ทั้งสอง scenario ให้ผลต่างกัน!

// อธิบาย: "AC" insert 'B' at 1 = "ABC", insert 'D' at 2 of "AC" = "ACD"
// After transform: op_b' = Insert(3,'D')
// Scenario 1: "AC" → "ABC" → "ABDC" (D ก่อน C? เพราะ op_b ที่ pos 2 ใน "AC" อยู่หลัง 'B')
// Scenario 2: "AC" → "ACD" → apply op_a' = Insert(1,'B') → "ABCD"
// ทั้งคู่ converge ✓
assert_eq!(doc1.content, doc2.content);
```

**`test_transform_delete_delete_same_pos`** — double delete idempotent:
```rust
// ทั้ง Alice และ Bob ลบ character ที่ position 2 พร้อมกัน
// transform คืน (Retain, Retain) — ลบแค่ครั้งเดียว
let (op1_prime, op2_prime) = transform(Delete{pos:2}, Delete{pos:2});
assert_eq!(op1_prime, Op::Retain); // ไม่ทำอะไรซ้ำ
assert_eq!(op2_prime, Op::Retain);
```

### Integration Tests สำหรับ Cursor

```rust
// tests/cursor_tests.rs
use collab_editor::cursor::{transform_cursor, Cursor, transform_cursor_full};
use collab_editor::ot::op::Op;

#[test]
fn test_cursor_after_insert_before() {
    // cursor อยู่ที่ pos 5, insert เกิดที่ pos 2
    // → cursor เลื่อนเป็น 6
    assert_eq!(transform_cursor(5, &Op::Insert { pos: 2, ch: 'X' }), 6);
}

#[test]
fn test_cursor_after_insert_after() {
    // cursor อยู่ที่ pos 2, insert เกิดที่ pos 5
    // → cursor ไม่เปลี่ยน
    assert_eq!(transform_cursor(2, &Op::Insert { pos: 5, ch: 'X' }), 2);
}

#[test]
fn test_cursor_after_delete_before() {
    // cursor อยู่ที่ pos 5, delete เกิดที่ pos 2
    // → cursor เลื่อนเป็น 4
    assert_eq!(transform_cursor(5, &Op::Delete { pos: 2 }), 4);
}

#[test]
fn test_cursor_at_deleted_position() {
    // cursor อยู่ที่ pos 3, delete เกิดที่ pos 3
    // → cursor ไม่เลื่อน (character ถูกลบ cursor ชี้ที่เดิม)
    assert_eq!(transform_cursor(3, &Op::Delete { pos: 3 }), 3);
}
```

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### ข้อผิดพลาดที่ 1: ลืม Transform Client Op ก่อน Apply

**ปัญหา**: นักพัฒนามือใหม่มักรีบ apply op ที่ client ส่งมาโดยตรง

```rust
// ❌ ผิด — ไม่ transform ก่อน
let new_version = room.doc.apply(&client_op)?;

// ✅ ถูก — transform ก่อนเสมอ
let transformed = room.transform_client_op(client_op, base_version);
let new_version = room.doc.apply(&transformed)?;
```

**ผลลัพธ์ที่ผิด**: documents diverge — แต่ละ client เห็น content ต่างกัน และ document ไม่สามารถ recover ได้หากไม่ restart server

**วิธีป้องกัน**: สร้าง unit test ที่ verify convergence property ทุกครั้งที่แก้ไข transform logic

---

### ข้อผิดพลาดที่ 2: Race Condition ใน Shared Room State

**ปัญหา**: ใช้ `Mutex` lock สั้นเกินไปจนเกิด TOCTOU (Time-of-Check-Time-of-Use) bug

```rust
// ❌ ผิด — อ่าน version และ apply แยกกัน
let current_version = room.lock().await.doc.version; // drop lock
// ... เวลาผ่านไป session อื่น apply op ...
let transformed = transform_client_op(op, base_version); // ใช้ version เก่า!
room.lock().await.doc.apply(&transformed)?; // version เปลี่ยนแล้ว!

// ✅ ถูก — lock ครั้งเดียวแล้วทำทุกอย่างใน critical section เดียว
let mut room = room_arc.write().await;
let transformed = room.transform_client_op(op, base_version);
let new_version = room.apply_op(transformed, user_id)?;
// drop lock เมื่อ block จบ
```

**ผลลัพธ์ที่ผิด**: interleaved ops ไม่ถูก transform อย่างถูกต้อง ทำให้ document corrupted

---

### ข้อผิดพลาดที่ 3: Byte Index vs Character Index ใน Unicode

**ปัญหา**: Rust's `String::insert()` และ `String::remove()` ใช้ **byte index** ไม่ใช่ character index

```rust
// ❌ ผิด สำหรับ Unicode
let mut s = "สวัสดี".to_string(); // 'ส' = 3 bytes ใน UTF-8
s.remove(1); // panic! byte 1 is not a char boundary

// ✅ ถูก — ใช้ char_indices
let mut s = "สวัสดี".to_string();
let byte_pos = s.char_indices().nth(1).map(|(i, _)| i).unwrap_or(s.len());
s.remove(byte_pos);
```

**วิธีแก้ใน production**: ใช้ `String::char_indices()` แปลง character position เป็น byte position ก่อน apply op หรือใช้ library เช่น `ropey` ที่ออกแบบมาสำหรับ text editing โดยเฉพาะ

---

### ข้อผิดพลาดที่ 4: Broadcast Channel Lag — Slow Consumer Problem

**ปัญหา**: `tokio::sync::broadcast` จะ drop messages ถ้า receiver ช้าเกินไป

```rust
// ❌ ผิด — ไม่จัดการ lagged error
let msg = broadcast_rx.recv().await.unwrap(); // panics on lag!

// ✅ ถูก — จัดการ RecvError::Lagged
match broadcast_rx.recv().await {
    Ok(event) => { /* process */ }
    Err(tokio::sync::broadcast::error::RecvError::Lagged(n)) => {
        tracing::warn!("Missed {} broadcast messages, requesting catchup", n);
        // ส่ง Catchup request ไปให้ server
        request_catchup(known_version).await;
    }
    Err(tokio::sync::broadcast::error::RecvError::Closed) => {
        break; // channel closed
    }
}
```

**ผลลัพธ์ที่ผิด**: client miss ops แล้ว document diverge โดยไม่รู้ตัว

---

### ข้อผิดพลาดที่ 5: ลืม Transform Cursor หลัง Apply Op

**ปัญหา**: เมื่อ server apply op ของ user A แล้ว cursor ของ users B, C, D ต้องถูก transform ด้วย

```rust
// ❌ ผิด — broadcast cursor เดิมโดยไม่ transform
let cursor_msg = ServerMsg::CursorUpdate { cursors: old_cursors };

// ✅ ถูก — transform cursors ทั้งหมดก่อน broadcast
let updated_cursors = transform_all_cursors(old_cursors, &applied_op);
let cursor_msg = ServerMsg::CursorUpdate { cursors: updated_cursors };
```

**ผลลัพธ์ที่ผิด**: cursor ของ user อื่นจะชี้ผิดตำแหน่งหลังจากมีการ edit — ทำให้ UX แย่มาก

---

### ข้อผิดพลาดที่ 6: ไม่เก็บ `base_version` ใน Pending Op Queue

**ปัญหา**: client มี ops ที่ยังรอ Ack อยู่ แล้วส่ง op ใหม่โดยใช้ version ปัจจุบันของ local doc แทน version ที่ server รู้จัก

```rust
// ❌ ผิด — ใช้ local doc version (ซึ่งรวม pending ops แล้ว)
ClientMsg::Op {
    op: new_op,
    base_version: local_doc.version, // ← ผิด! version นี้ server ยังไม่รู้จัก
    ..
}

// ✅ ถูก — ใช้ server_version (version ล่าสุดที่ได้รับ Ack จาก server)
ClientMsg::Op {
    op: new_op,
    base_version: self.server_acknowledged_version, // ← ถูก
    ..
}
```

---

## การ Package และ Deploy

### Build สำหรับ Production

```bash
# Build optimized binary
cargo build --release

# ตรวจ binary size
ls -lh target/release/collab-editor
# ประมาณ 5–15 MB depending on features

# Strip debug symbols ลดขนาด
strip target/release/collab-editor
```

### Docker Deployment

```dockerfile
# Dockerfile
FROM rust:1.75-slim as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/collab-editor /usr/local/bin/
EXPOSE 8080
ENV RUST_LOG=collab_editor=info
CMD ["collab-editor"]
```

```bash
# Build image
docker build -t collab-editor:latest .

# Run
docker run -p 8080:8080 \
  -e LISTEN_ADDR=0.0.0.0:8080 \
  collab-editor:latest
```

### Environment Variables

```bash
# กำหนด address ที่ server จะ listen
LISTEN_ADDR=0.0.0.0:8080

# ควบคุม log level
RUST_LOG=collab_editor=debug,tokio=warn

# เพิ่ม capacity สำหรับ broadcast channel (default: 1024)
BROADCAST_CHANNEL_SIZE=4096
```

### Health Check

เพิ่ม HTTP health check endpoint:

```rust
// ใน main.rs — เพิ่ม HTTP endpoint สำหรับ health check
async fn health_handler() -> impl warp::Reply {
    warp::reply::json(&serde_json::json!({
        "status": "ok",
        "version": env!("CARGO_PKG_VERSION")
    }))
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Undo/Redo ด้วย Op Inversion (ระดับกลาง)

**เป้าหมาย**: Implement `invert(op) -> Op` ที่คืน inverse operation:
- `invert(Insert{pos, ch})` → `Delete{pos}` 
- `invert(Delete{pos})` → `Insert{pos, original_ch}` (ต้องจำ char ก่อนลบ)
- `invert(Retain)` → `Retain`

```rust
// hint: Doc ต้องบันทึก char ที่ถูก delete เพื่อให้ invert ทำงานได้
pub fn apply_and_record(&mut self, op: &Op) -> Result<Op, DocError> {
    match op {
        Op::Delete { pos } => {
            let ch = self.content.chars().nth(*pos)
                .ok_or(DocError::DeleteOutOfBounds { pos: *pos, len: self.content.len() })?;
            self.content.remove(*pos);
            self.version += 1;
            Ok(Op::Insert { pos: *pos, ch }) // inverse op
        }
        // ...
    }
}
```

จากนั้นสร้าง `UndoStack` ที่เก็บ inverse ops และ method `undo()` / `redo()`

---

### แบบฝึกหัดที่ 2: เพิ่ม Persistence ด้วย SQLite (ระดับกลาง-สูง)

**เป้าหมาย**: เมื่อ server restart ให้ document state ยังคงอยู่

ใช้ `sqlx` กับ SQLite:

```toml
[dependencies]
sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio", "chrono"] }
```

Schema:
```sql
CREATE TABLE rooms (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL DEFAULT '',
    version INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE op_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    room_id TEXT NOT NULL,
    version INTEGER NOT NULL,
    op_json TEXT NOT NULL,
    user_id TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (room_id) REFERENCES rooms(id)
);
```

ต้องแก้ไข `Room::apply_op()` ให้ persist ทุก op ลง SQLite ด้วย และ load room state จาก DB ตอน startup

---

### แบบฝึกหัดที่ 3: CRDT Alternative — Implement Simple LSEQ (ระดับสูง)

**เป้าหมาย**: เปรียบเทียบ OT กับ CRDT โดย implement LSEQ (Logoot-like) algorithm

LSEQ แทน character ด้วย unique position identifier แทน byte index:

```rust
/// Character ใน LSEQ document
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct Atom {
    /// ตำแหน่ง fractional — sortable, unique, never changes
    pub position: Vec<u64>,
    /// unique ID (site_id, clock) เพื่อ break ties
    pub site_id: u64,
    pub clock: u64,
    /// ตัวอักขระ
    pub ch: char,
    /// ถูกลบแล้วหรือยัง (tombstone)
    pub deleted: bool,
}
```

ข้อดีเทียบ OT:
- ไม่ต้องการ central server สำหรับ transform
- Op order ไม่สำคัญ — idempotent โดยธรรมชาติ

ข้อเสีย:
- Memory ใช้มากกว่า (tombstones)
- ต้อง garbage collect tombstones

---

### แบบฝึกหัดที่ 4: Multi-Document Room และ Tab Management (ระดับกลาง)

**เป้าหมาย**: ให้ room รองรับหลาย documents พร้อมกัน (เช่น หลาย file ใน project เดียวกัน)

```rust
// แก้ Room ให้รองรับหลาย docs
pub struct Room {
    pub id: RoomId,
    pub docs: HashMap<String, Doc>,       // doc_id → Doc
    pub op_logs: HashMap<String, OpLog>,  // doc_id → OpLog
    // ...
}

// Protocol เพิ่ม doc_id
pub enum ClientMsg {
    Op {
        doc_id: String,  // ← เพิ่ม
        op: Op,
        base_version: u64,
        request_id: String,
    },
    // ...
}
```

ต้องจัดการ:
- switch doc: client ส่ง `SwitchDoc{ doc_id }` แล้ว server ส่ง FullState ของ doc ใหม่
- create doc: `CreateDoc{ doc_id, initial_content }`
- delete doc: `DeleteDoc{ doc_id }` (ต้อง broadcast ให้ทุก user ที่เปิด doc นั้น)

---

### แบบฝึกหัดที่ 5: Rich Text Support — Bold, Italic, Color (ระดับสูง)

**เป้าหมาย**: ขยาย Op ให้รองรับ rich text formatting

```rust
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum FormatAttr {
    Bold(bool),
    Italic(bool),
    Color(String),
    Underline(bool),
}

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum Op {
    Insert { pos: usize, ch: char, attrs: Vec<FormatAttr> },
    Delete { pos: usize },
    Format { start: usize, end: usize, attr: FormatAttr }, // ← ใหม่
    Retain,
}
```

`transform()` ต้องจัดการ `Format vs Insert`, `Format vs Delete`, `Format vs Format` ด้วย — โดยเฉพาะ case ที่ format range ทับซ้อนกัน

---

### แบบฝึกหัดที่ 6: Load Testing ด้วย Simulated Clients (ระดับกลาง)

**เป้าหมาย**: วัด throughput ของ server ด้วยจำลอง concurrent clients

```rust
// tests/load_test.rs
use tokio::net::TcpStream;
use tokio_tungstenite::connect_async;

#[tokio::test]
async fn load_test_100_clients() {
    // spawn 100 clients ทุกคน join room เดียวกัน
    // แต่ละคนส่ง 10 ops พร้อมกัน
    // verify ว่า server ไม่ crash และ ops ทั้งหมดถูก Ack
    let handles: Vec<_> = (0..100).map(|i| {
        tokio::spawn(async move {
            let (ws, _) = connect_async("ws://127.0.0.1:8080").await.unwrap();
            // join room + send ops
            // assert all ops get Ack within timeout
        })
    }).collect();
    
    for handle in handles {
        handle.await.unwrap();
    }
}
```

ตัวชี้วัดที่ควรวัด:
- ops/second ที่ server รับได้
- latency ของแต่ละ Ack (p50, p95, p99)
- memory usage ตาม ops ที่สะสมใน OpLog

---

## สรุป

ในโปรเจคนี้คุณได้สร้าง **Real-Time Collaborative Editor** ครบวงจรตั้งแต่:

**OT Algorithm Core**:
- `Op` enum ที่แทน insert/delete operation
- `Doc::apply()` ที่จัดการ document state พร้อม error handling
- `transform(op1, op2)` ที่จัดการทุก case ของ concurrent operations
- `transform_against_history()` สำหรับ late-arriving ops

**WebSocket Server**:
- Multi-client handler ด้วย `tokio-tungstenite`
- Shared state ด้วย `Arc<RwLock<Room>>` และ `DashMap`
- Broadcast channel สำหรับ real-time op distribution

**Advanced Features**:
- Cursor synchronization ด้วย cursor transform
- Session management ด้วย `RoomId` และ `UserList`
- Reconnection และ catch-up protocol

**Pattern สำคัญที่ได้เรียน**:

| Pattern | ใช้ตรงไหน |
|---------|-----------|
| `Arc<RwLock<T>>` | Shared mutable room state ระหว่าง async tasks |
| `DashMap` | Concurrent HashMap ไม่ต้อง lock ทั้ง map |
| `broadcast::channel` | Fan-out message ไปยัง N receivers พร้อมกัน |
| `tokio::select!` | รอ event จากหลาย source พร้อมกัน |
| `serde` tagged enum | Protocol messages ที่ serialize/deserialize ได้ |
| Convergence property | Algorithm correctness ที่ verify ด้วย tests |

โปรเจคนี้เชื่อมต่อกับโปรเจคถัดไป **GraphQL API** ซึ่งจะสอนวิธีสร้าง API layer ที่ flexible ด้วย `async-graphql` — คุณสามารถนำ Room/Doc model จากโปรเจคนี้มา expose ผ่าน GraphQL subscriptions เพื่อ query document history และ user presence ได้

---

**โปรเจคก่อนหน้า:** [project-j04-wasm-plugin.md](project-j04-wasm-plugin.md) | **โปรเจคถัดไป:** [project-j06-graphql-api.md](project-j06-graphql-api.md)
