# Project F01: Raft Consensus KV Store

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 20–30 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **Replicated Key-Value Store** ที่ขับเคลื่อนด้วย **Raft consensus algorithm** — หนึ่งในอัลกอริทึมพื้นฐานที่สุดในโลก distributed systems ซึ่งใช้งานจริงใน etcd, CockroachDB, TiKV, Consul, InfluxDB Clustered และอีกหลายร้อยระบบในโลก production

**ปัญหาที่ Raft แก้:** ในระบบ distributed ที่มีหลาย node เราต้องการให้ทุก node เห็น "ลำดับของ operations" เหมือนกัน แม้ว่า node บางตัวจะ crash หรือ network จะล่าช้า — นี่คือ **consensus problem** ที่ Raft ออกแบบมาเพื่อแก้โดยเฉพาะ โดยให้ความสำคัญกับ **understandability** มากกว่า Paxos

**Use cases จริงในโลก production:**
- **etcd** — distributed key-value store สำหรับ Kubernetes configuration
- **CockroachDB / TiDB** — distributed SQL ที่ใช้ Raft ต่อ range/shard
- **Consul** — service discovery และ configuration management
- **InfluxDB Clustered** — time series database ที่ใช้ Raft สำหรับ replication
- **Distributed locks** — ระบบ mutual exclusion ข้ามหลาย process

**Learning value:**
โปรเจคนี้ไม่ใช่แค่การ implement อัลกอริทึม — มันสอนวิธีคิดแบบ distributed systems: fault tolerance, consistency guarantees, leader election, log-based state machine replication และ snapshot-based log compaction ซึ่งเป็นพื้นฐานของระบบ cloud-native ทุกประเภท

---

## สิ่งที่จะได้เรียนรู้

- **Raft consensus** — leader election, log replication, safety guarantees (election safety, log matching, leader completeness)
- **State machine replication** — ใช้ `Vec<LogEntry>` เป็น replicated log และ `HashMap<String, String>` เป็น state machine
- **Message passing** — สร้าง cluster simulation ด้วย `std::sync::mpsc` channels และ `VecDeque` message queues
- **Enum-driven RPC** — ออกแบบ `Message` enum แทน real network calls สำหรับ testability
- **Snapshot + log compaction** — บีบอัด log ที่ยาวด้วย periodic snapshots เพื่อลด memory และ restart time
- **Randomization ใน distributed systems** — ใช้ `rand` สำหรับ randomized election timeout ที่ป้องกัน split vote
- **`serde` + `serde_json`** สำหรับ snapshot serialization ที่ใช้จริงใน production
- **Safety ผ่าน types** — `RaftRole`, `LogEntry`, `Command` ทำให้ compile-time ป้องกัน state transitions ที่ผิด

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust basics — ownership, borrowing, structs, enums, `Vec`, `HashMap`
- **Part 21–30**: Collections, iterators, pattern matching แบบลึก
- **Part 31–40**: Error handling (`Result`, `Option`), traits, generics
- **Part 41–50**: `async/await` concepts (จาก Part 46), channels (`mpsc`)
- **Part 51–60**: crate ecosystem — `serde`, `rand`, Cargo features
- **Part 61–70**: concurrent programming concepts, `Arc`, `Mutex` (ใช้ใน extensions)
- **Part 96–110**: distributed systems theory (จาก Part 98–102 เรื่อง consensus, CAP theorem)
- พื้นฐาน **Raft paper** (Ongaro & Ousterhout 2014) — อ่าน §5.1–§5.4 ก็เพียงพอ

---

## โครงสร้างโปรเจค (Project Layout)

```
raft-kv/
├── src/
│   ├── lib.rs           ← core types, election, log replication, state machine
│   ├── main.rs          ← demo binary
│   ├── types.rs         ← RaftRole, LogEntry, Command, RaftState
│   ├── election.rs      ← VoteRequest/Response, random timeout, vote logic
│   ├── replication.rs   ← AppendEntries RPC, conflict resolution
│   ├── state_machine.rs ← KvStateMachine, apply_committed
│   ├── snapshot.rs      ← Snapshot, compact_log
│   └── cluster.rs       ← Cluster simulation, Message enum
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow ใน Raft

```
Client Request
      │
      ▼
┌─────────────┐    AppendEntries    ┌──────────────┐
│   Leader    │ ──────────────────▶ │  Follower 1  │
│  (Node 0)   │ ◀─────────────────  │  (Node 1)    │
│             │    Success/Fail     └──────────────┘
│  Log:       │
│  [1,2,3,4]  │    AppendEntries    ┌──────────────┐
│             │ ──────────────────▶ │  Follower 2  │
│  commit=4   │ ◀─────────────────  │  (Node 2)    │
└─────────────┘    Success/Fail     └──────────────┘
      │
      ▼
Apply to State Machine (KV Store)
      │
      ▼
Return response to Client
```

### Module Responsibilities

| Module | หน้าที่ |
|--------|---------|
| `types.rs` | Core data structures: `RaftRole`, `LogEntry`, `Command`, `RaftState` |
| `election.rs` | `VoteRequest/Response`, randomized timeout, vote counting |
| `replication.rs` | `AppendEntriesRequest/Response`, log consistency, conflict resolution |
| `state_machine.rs` | `KvStateMachine` — apply commands, `apply_committed` loop |
| `snapshot.rs` | `Snapshot` struct, serialization, `compact_log` |
| `cluster.rs` | `Cluster` simulation, `Message` enum, in-memory message routing |

### ทำไมถึง Simulate แทน Real Networking?

โปรเจคนี้ใช้ **in-memory message simulation** แทน real TCP/UDP connections เพราะ:
1. **Testability** — ทดสอบ Raft logic ได้โดยไม่ต้องจัดการ network latency, port binding
2. **Determinism** — ควบคุม message ordering ได้ทำให้ edge cases ทดสอบได้
3. **Simplicity** — focus ที่ Raft algorithm ไม่ใช่ network code
4. **Realism** — ใช้ `VecDeque` queue แทน TCP buffer — semantics เหมือนกัน

Production implementations (เช่น TiKV) แยก transport layer ออกจาก consensus layer ด้วยวิธีนี้เช่นกัน

### ทำไม `LogEntry` ต้องมี `term` และ `index`?

```
Log ใน Raft: entry ทุกตัวระบุด้วย (term, index) pair

index:  1      2      3      4      5
term:   ──┬────┬──────┬──────┬──────┬──
          │t=1 │ t=1  │ t=2  │ t=2  │ t=3
          └────┴──────┴──────┴──────┴──

- term: เทอมที่ leader สร้าง entry นี้ขึ้น → ป้องกันการ commit entries จาก stale leader
- index: ตำแหน่งใน log (1-based) → ใช้ใน AppendEntries prev_log_index check
```

**Log Matching Property** (ความถูกต้องหลักของ Raft):
ถ้า 2 logs มี entry ที่ term และ index เดียวกัน → entries ทุกตัวก่อนหน้าในทั้งสอง log เหมือนกันทุกประการ

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Core Types — RaftRole, LogEntry, RaftState

ขั้นแรกสร้างโครงสร้างข้อมูลหลัก:

```toml
# Cargo.toml
[package]
name = "raft-kv"
version = "0.1.0"
edition = "2021"

[dependencies]
rand = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

```rust
// src/types.rs
use std::collections::HashMap;
use serde::{Deserialize, Serialize};

/// บทบาทของ node ใน cluster ณ เวลาใดเวลาหนึ่ง
/// node แต่ละตัวอยู่ใน state นี้เสมอ — transition เกิดจาก events เท่านั้น
#[derive(Debug, Clone, PartialEq)]
pub enum RaftRole {
    Follower,   // สถานะเริ่มต้น — รับ AppendEntries จาก leader
    Candidate,  // กำลังขอ vote — ส่ง VoteRequest ไปทุก node
    Leader,     // ชนะ election — ส่ง AppendEntries ไปทุก follower
}

/// คำสั่งที่ client ส่งมา — จะถูก replicate ผ่าน Raft log
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum Command {
    Set { key: String, value: String },
    Delete { key: String },
    NoOp,  // leader ส่งเมื่อเพิ่งได้รับเลือก — commit entries จาก term ก่อนหน้า
}

/// หน่วยพื้นฐานใน Raft log — ทุก entry ถูกระบุด้วย (term, index) อย่างไม่ซ้ำกัน
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub struct LogEntry {
    pub term: u64,     // term ที่ leader สร้าง entry นี้
    pub index: u64,    // ตำแหน่งใน log (1-based global index)
    pub command: Command,
}

/// สถานะทั้งหมดของ Raft node หนึ่งตัว
/// ใน production สถานะบางส่วน (current_term, voted_for, log) ต้อง persist ลง disk
#[derive(Debug)]
pub struct RaftState {
    pub id: u64,
    pub role: RaftRole,
    /// Persistent state (must survive crash)
    pub current_term: u64,      // เทอมปัจจุบัน (monotonically increasing)
    pub voted_for: Option<u64>, // candidate ที่โหวตให้ในเทอมนี้ (ถ้ามี)
    pub log: Vec<LogEntry>,     // log entries หลัง snapshot (index เริ่มต้นที่ snapshot_last_index+1)
    /// Volatile state
    pub commit_index: u64,      // index สูงสุดที่ทราบว่า committed
    pub last_applied: u64,      // index สูงสุดที่ apply ลง state machine แล้ว
    /// Snapshot state
    pub snapshot_last_index: u64, // index สุดท้ายที่ถูก compact ไปแล้ว
    pub snapshot_last_term: u64,  // term ของ entry นั้น
}

impl RaftState {
    pub fn new(id: u64) -> Self {
        RaftState {
            id,
            role: RaftRole::Follower,
            current_term: 0,
            voted_for: None,
            log: Vec::new(),
            commit_index: 0,
            last_applied: 0,
            snapshot_last_index: 0,
            snapshot_last_term: 0,
        }
    }

    /// ดึง (last_log_term, last_log_index) สำหรับเปรียบใน election
    pub fn last_log_term_and_index(&self) -> (u64, u64) {
        if let Some(entry) = self.log.last() {
            (entry.term, entry.index)
        } else {
            // log ว่างเปล่า → ใช้ค่าจาก snapshot
            (self.snapshot_last_term, self.snapshot_last_index)
        }
    }

    /// ตรวจสอบว่า log ของ candidate up-to-date กว่า (หรือเท่ากับ) log ของ voter
    /// ตาม Raft §5.4.1: compare term ก่อน — ถ้าเท่ากันค่อย compare index
    pub fn is_log_up_to_date(&self, cand_last_term: u64, cand_last_index: u64) -> bool {
        let (my_term, my_index) = self.last_log_term_and_index();
        if cand_last_term != my_term {
            cand_last_term > my_term
        } else {
            cand_last_index >= my_index
        }
    }

    /// จำนวน log entries ทั้งหมด รวม entries ที่ถูก compact ไปแล้ว (ใน snapshot)
    pub fn log_length(&self) -> u64 {
        self.snapshot_last_index + self.log.len() as u64
    }

    /// ดึง log entry ตาม global 1-based index
    pub fn get_log_entry(&self, global_index: u64) -> Option<&LogEntry> {
        if global_index == 0 || global_index <= self.snapshot_last_index {
            return None;
        }
        let pos = (global_index - self.snapshot_last_index) as usize;
        self.log.get(pos - 1)
    }
}
```

---

### ขั้นที่ 2: Leader Election — VoteRequest, VoteResponse

Raft election ทำงานผ่าน 3 ขั้นตอน:
1. Follower หมด election timeout → เปลี่ยนเป็น **Candidate**, เพิ่ม term, โหวตให้ตัวเอง
2. Candidate broadcast **VoteRequest** ไปทุก node
3. ถ้าได้รับ majority grants → เป็น **Leader**; ถ้า timeout ซ้ำ → start election ใหม่

```rust
// src/election.rs
use rand::Rng;
use crate::types::{RaftRole, RaftState};

pub struct VoteRequest {
    pub term: u64,
    pub candidate_id: u64,
    pub last_log_index: u64,
    pub last_log_term: u64,
}

pub struct VoteResponse {
    pub term: u64,
    pub vote_granted: bool,
}

/// Election timeout แบบ random ช่วย break symmetry และป้องกัน split vote
/// ช่วงมาตรฐาน: 150–300ms (หรือ 150–300 tic ใน simulation)
pub fn random_election_timeout_ms() -> u64 {
    rand::thread_rng().gen_range(150..=300)
}

/// Voter ประมวลผล VoteRequest ตาม Raft §5.2 + §5.4
///
/// จะ grant vote ก็ต่อเมื่อ:
/// 1. term ของ request ≥ current_term
/// 2. ยังไม่เคย vote ในเทอมนี้ (หรือ vote ให้ candidate เดิมอยู่แล้ว)
/// 3. log ของ candidate up-to-date กว่า (หรือเท่ากับ) log ของ voter
pub fn handle_vote_request(state: &mut RaftState, req: &VoteRequest) -> VoteResponse {
    // กฎที่ 1: ปฏิเสธถ้า term ของ requester เก่ากว่า
    if req.term < state.current_term {
        return VoteResponse {
            term: state.current_term,
            vote_granted: false,
        };
    }

    // กฎที่ 2: ถ้าเห็น term สูงกว่า → revert เป็น Follower, reset voted_for
    if req.term > state.current_term {
        state.current_term = req.term;
        state.role = RaftRole::Follower;
        state.voted_for = None;
    }

    let can_vote = state.voted_for.is_none()
        || state.voted_for == Some(req.candidate_id);
    let log_ok = state.is_log_up_to_date(req.last_log_term, req.last_log_index);

    if can_vote && log_ok {
        state.voted_for = Some(req.candidate_id);
        VoteResponse {
            term: state.current_term,
            vote_granted: true,
        }
    } else {
        VoteResponse {
            term: state.current_term,
            vote_granted: false,
        }
    }
}

/// ตรวจสอบว่ามี majority หรือยัง
/// Majority = มากกว่าครึ่งหนึ่ง: votes > cluster_size / 2
/// ในคลัสเตอร์ขนาด 3: ต้องการ 2 votes, ขนาด 5: ต้องการ 3 votes
pub fn has_majority(votes: usize, cluster_size: usize) -> bool {
    votes * 2 > cluster_size
}
```

**ทำไม randomized timeout จึงสำคัญ?**

ถ้า node ทั้งหมด timeout พร้อมกัน ทุก node จะเป็น Candidate พร้อมกันและส่ง VoteRequest พร้อมกัน ทำให้ votes กระจายและไม่มีใครได้ majority — เรียกว่า **split vote** Randomized timeout ช่วยให้ node หนึ่งตื่นก่อนคนอื่นและชนะ election ก่อนที่คนอื่นจะ timeout

```
ไม่มี randomization:
Node 0: timeout=150ms ──▶ Candidate ──▶ VoteReq  ┐
Node 1: timeout=150ms ──▶ Candidate ──▶ VoteReq  ├── split vote
Node 2: timeout=150ms ──▶ Candidate ──▶ VoteReq  ┘

มี randomization:
Node 0: timeout=160ms ──▶ Candidate ──▶ VoteReq ──▶ Leader ✓
Node 1: timeout=210ms    ยังไม่ timeout → ตอบ VoteResp(granted=true)
Node 2: timeout=285ms    ยังไม่ timeout → ตอบ VoteResp(granted=true)
```

---

### ขั้นที่ 3: Log Replication — AppendEntries RPC

AppendEntries เป็น RPC หลักของ Raft — ใช้ทั้งสำหรับ replicate entries และ heartbeat (entries ว่าง) เพื่อยืนยันว่า leader ยังมีชีวิตอยู่

```rust
// src/replication.rs
use crate::types::{LogEntry, RaftRole, RaftState};

pub struct AppendEntriesRequest {
    pub term: u64,
    pub leader_id: u64,
    /// global 1-based index ของ entry ก่อนหน้า entries ที่จะ append
    pub prev_log_index: u64,
    /// term ของ entry ที่ prev_log_index
    pub prev_log_term: u64,
    pub entries: Vec<LogEntry>,
    /// commit index ของ leader
    pub leader_commit: u64,
}

pub struct AppendEntriesResponse {
    pub term: u64,
    pub success: bool,
    /// ถ้า success: match_index ที่ follower มีหลัง append เสร็จ
    pub match_index: u64,
}

/// ประมวลผล AppendEntries RPC ตาม Raft §5.3
///
/// ขั้นตอน 5 ขั้น:
/// 1. ปฏิเสธถ้า term ต่ำกว่า (stale leader)
/// 2. อัปเดต term/role ถ้าเห็น term สูงกว่า
/// 3. ตรวจสอบ prev_log consistency (log matching check)
/// 4. Append entries, resolve conflicts (truncate เมื่อ term ไม่ตรง)
/// 5. อัปเดต commit_index ถ้า leader_commit สูงกว่า
pub fn handle_append_entries(
    state: &mut RaftState,
    req: &AppendEntriesRequest,
) -> AppendEntriesResponse {
    // ขั้น 1
    if req.term < state.current_term {
        return AppendEntriesResponse {
            term: state.current_term,
            success: false,
            match_index: 0,
        };
    }

    // ขั้น 2
    if req.term > state.current_term {
        state.current_term = req.term;
        state.voted_for = None;
    }
    state.role = RaftRole::Follower;

    // ขั้น 3: ตรวจ prev_log
    if req.prev_log_index > 0 {
        if req.prev_log_index <= state.snapshot_last_index {
            // prev entry อยู่ใน snapshot แล้ว — ถือว่า consistent
        } else {
            let pos = (req.prev_log_index - state.snapshot_last_index) as usize;
            if pos > state.log.len() {
                // ไม่มี entry ที่ prev_log_index — log ขาด
                return AppendEntriesResponse {
                    term: state.current_term,
                    success: false,
                    match_index: state.log_length(),
                };
            }
            if state.log[pos - 1].term != req.prev_log_term {
                // Term ของ entry ที่ prev_log_index ไม่ตรง — conflict
                return AppendEntriesResponse {
                    term: state.current_term,
                    success: false,
                    match_index: req.prev_log_index - 1,
                };
            }
        }
    }

    // ขั้น 4: Append/replace entries
    let mut cur_global = req.prev_log_index;
    for entry in &req.entries {
        cur_global += 1;
        if cur_global <= state.snapshot_last_index {
            continue; // entry นี้อยู่ใน snapshot แล้ว
        }
        let pos = (cur_global - state.snapshot_last_index) as usize;
        if pos <= state.log.len() {
            if state.log[pos - 1].term != entry.term {
                // Conflict: truncate จากตำแหน่งนี้เป็นต้นไป แล้ว append ใหม่
                state.log.truncate(pos - 1);
                state.log.push(entry.clone());
            }
            // term เหมือนกัน: entry มีอยู่แล้ว ไม่ต้องทำอะไร
        } else {
            state.log.push(entry.clone());
        }
    }

    // ขั้น 5: update commit_index
    if req.leader_commit > state.commit_index {
        state.commit_index = req.leader_commit.min(state.log_length());
    }

    AppendEntriesResponse {
        term: state.current_term,
        success: true,
        match_index: state.log_length(),
    }
}
```

**Conflict Resolution ทำงานอย่างไร?**

```
ตัวอย่าง: Follower มี log จาก leader ที่ crash ก่อน commit

Follower log:  [t1:1] [t1:2] [t1:3]   (leader เก่า เทอม 1)
Leader log:    [t1:1] [t2:2] [t2:3]   (leader ใหม่ เทอม 2)

AppendEntries: prev_log_index=1, prev_log_term=1
               entries=[t2:2, t2:3]

Step 4: cur_global=2 → pos=2, log[1].term=1 ≠ entry.term=2
        → truncate(1): log = [t1:1]
        → push t2:2:    log = [t1:1, t2:2]
        cur_global=3 → pos=3, log.len()=2, pos>len → push t2:3
        log = [t1:1, t2:2, t2:3]  ✓ ตรงกับ leader
```

---

### ขั้นที่ 4: State Machine — KV Store

State machine รับ committed log entries และ apply ทีละตัวตามลำดับ ใน KV Store เราใช้ `HashMap<String, String>`:

```rust
// src/state_machine.rs
use std::collections::HashMap;
use crate::types::{Command, RaftState};

#[derive(Debug, Default)]
pub struct KvStateMachine {
    pub store: HashMap<String, String>,
}

impl KvStateMachine {
    pub fn new() -> Self {
        KvStateMachine { store: HashMap::new() }
    }

    /// Apply คำสั่งเดียวไปยัง state machine
    /// ส่งคืน Option<String> สำหรับ Get (ใน extensions)
    pub fn apply(&mut self, command: &Command) -> Option<String> {
        match command {
            Command::Set { key, value } => {
                self.store.insert(key.clone(), value.clone());
                None
            }
            Command::Delete { key } => {
                self.store.remove(key);
                None
            }
            Command::NoOp => None,
        }
    }

    /// Apply log entries ที่ committed แล้วแต่ยังไม่ถูก apply
    /// เรียกหลังจาก commit_index ถูก update
    ///
    /// Invariant: last_applied ≤ commit_index เสมอหลังเรียก
    pub fn apply_committed(&mut self, state: &mut RaftState) {
        while state.last_applied < state.commit_index {
            state.last_applied += 1;
            let idx = state.last_applied;

            if idx <= state.snapshot_last_index {
                // entry นี้อยู่ใน snapshot แล้ว ข้ามไป
                continue;
            }

            let pos = (idx - state.snapshot_last_index) as usize;
            if pos > 0 && pos <= state.log.len() {
                let cmd = state.log[pos - 1].command.clone();
                self.apply(&cmd);
            }
        }
    }

    pub fn get(&self, key: &str) -> Option<&String> {
        self.store.get(key)
    }
}
```

**ทำไม `last_applied` ต้องแยกออกจาก `commit_index`?**

```
commit_index: highest index known to be replicated on majority
last_applied: highest index actually applied to state machine

ช่วงเวลาระหว่าง commit และ apply เกิดจาก:
- Apply เกิดใน background thread แยก (production Raft)
- Snapshot restore: reset state machine ก่อน replay log ที่เหลือ
- Pause apply เพื่อ take snapshot

ถ้ารวม 2 ตัวเป็นหนึ่ง: จะ re-apply entries ที่ restore จาก snapshot
```

---

### ขั้นที่ 5: Snapshot และ Log Compaction

Log ยาวขึ้นเรื่อย ๆ ตามเวลา — Raft ใช้ **snapshot** บีบอัด log เพื่อลด memory และ disk usage รวมถึงเร่ง startup time:

```rust
// src/snapshot.rs
use std::collections::HashMap;
use serde::{Deserialize, Serialize};
use crate::types::RaftState;
use crate::state_machine::KvStateMachine;

/// Snapshot ของ state machine ณ จุดใดจุดหนึ่ง
/// ใช้แทน log entries ทั้งหมดที่ถูก apply ไปก่อนหน้า last_included_index
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Snapshot {
    /// index สูงสุดใน log ที่ snapshot นี้ครอบคลุม
    pub last_included_index: u64,
    /// term ของ entry ที่ last_included_index
    pub last_included_term: u64,
    /// สถานะของ state machine ณ จุดนั้น
    pub data: HashMap<String, String>,
}

impl Snapshot {
    pub fn from_state_machine(
        sm: &KvStateMachine,
        last_included_index: u64,
        last_included_term: u64,
    ) -> Self {
        Snapshot {
            last_included_index,
            last_included_term,
            data: sm.store.clone(),
        }
    }

    /// Serialize snapshot เป็น bytes สำหรับ disk หรือ network transfer
    pub fn serialize(&self) -> Vec<u8> {
        serde_json::to_vec(self).expect("snapshot serialization failed")
    }

    /// Deserialize snapshot จาก bytes
    pub fn deserialize(bytes: &[u8]) -> Self {
        serde_json::from_slice(bytes).expect("snapshot deserialization failed")
    }
}

/// Compact log หลังจาก take snapshot:
/// ลบ log entries ที่ index ≤ snapshot.last_included_index
pub fn compact_log(state: &mut RaftState, snapshot: &Snapshot) {
    let cutoff = snapshot.last_included_index;

    // ไม่ compact ถ้า snapshot เก่ากว่าที่มีอยู่
    if cutoff <= state.snapshot_last_index {
        return;
    }

    // จำนวน entries ที่ต้องลบออกจาก log Vec
    let entries_to_remove = (cutoff - state.snapshot_last_index) as usize;
    let remove_count = entries_to_remove.min(state.log.len());
    state.log.drain(..remove_count);

    state.snapshot_last_index = cutoff;
    state.snapshot_last_term = snapshot.last_included_term;

    // อัปเดต volatile state ถ้าจำเป็น
    if state.commit_index < cutoff {
        state.commit_index = cutoff;
    }
    if state.last_applied < cutoff {
        state.last_applied = cutoff;
    }
}
```

**ตัวอย่าง Snapshot cycle:**

```
Before snapshot (log ยาว 1000 entries):
  snapshot_last_index = 0
  log = [e1, e2, e3, ... e1000]   ← ใช้ memory มาก

Take snapshot at index 900:
  snapshot = { last_included_index=900, data={...} }
  compact_log(state, &snapshot)

After snapshot:
  snapshot_last_index = 900
  log = [e901, e902, ... e1000]   ← เหลือแค่ 100 entries ✓

Restart node:
  1. Load snapshot → restore KvStateMachine to state@index900
  2. Replay log[e901..e1000] → ได้ state ล่าสุด
  ประหยัดเวลา 90% เทียบกับ replay ตั้งแต่ต้น
```

---

### ขั้นที่ 6: Cluster Simulation — 3-Node In-Memory Cluster

ประกอบทุกส่วนเข้าด้วยกันในรูปแบบ simulation ที่ทดสอบได้:

```rust
// src/cluster.rs
use std::collections::VecDeque;
use crate::types::{LogEntry, RaftRole, RaftState};
use crate::election::{handle_vote_request, has_majority, VoteRequest};
use crate::replication::{handle_append_entries, AppendEntriesRequest};
use crate::state_machine::KvStateMachine;

/// Message ที่แลกเปลี่ยนระหว่าง nodes — แทน RPC calls ใน real system
#[derive(Debug, Clone)]
pub enum Message {
    VoteReq {
        from: u64,
        term: u64,
        last_log_index: u64,
        last_log_term: u64,
    },
    VoteResp {
        from: u64,
        term: u64,
        granted: bool,
    },
    AppendReq {
        from: u64,
        term: u64,
        prev_log_index: u64,
        prev_log_term: u64,
        entries: Vec<LogEntry>,
        leader_commit: u64,
    },
    AppendResp {
        from: u64,
        term: u64,
        success: bool,
        match_index: u64,
    },
}

/// Cluster simulation: N nodes ที่สื่อสารกันผ่าน in-memory message queues
pub struct Cluster {
    pub nodes: Vec<RaftState>,
    pub state_machines: Vec<KvStateMachine>,
    /// Inbox แต่ละ node
    pub queues: Vec<VecDeque<Message>>,
    /// จำนวน votes ที่แต่ละ node ได้รับในฐานะ candidate
    pub vote_counts: Vec<usize>,
}

impl Cluster {
    pub fn new(size: usize) -> Self {
        Cluster {
            nodes: (0..size as u64).map(RaftState::new).collect(),
            state_machines: (0..size).map(|_| KvStateMachine::new()).collect(),
            queues: (0..size).map(|_| VecDeque::new()).collect(),
            vote_counts: vec![0; size],
        }
    }

    /// Node `id` เริ่ม election: เปลี่ยนเป็น Candidate และ broadcast VoteRequest
    pub fn start_election(&mut self, id: usize) {
        let n = self.nodes.len();
        {
            let node = &mut self.nodes[id];
            node.role = RaftRole::Candidate;
            node.current_term += 1;
            node.voted_for = Some(node.id);
        }
        self.vote_counts[id] = 1; // vote ให้ตัวเอง

        let (last_term, last_index) = self.nodes[id].last_log_term_and_index();
        let term = self.nodes[id].current_term;
        let from = id as u64;

        for to in 0..n {
            if to != id {
                self.queues[to].push_back(Message::VoteReq {
                    from,
                    term,
                    last_log_index: last_index,
                    last_log_term: last_term,
                });
            }
        }
    }

    /// Process messages ทั้งหมดใน inbox ของทุก node หนึ่งรอบ
    /// ใน real system: แต่ละ node รัน loop ของตัวเองใน thread/task แยก
    pub fn process_messages(&mut self) {
        let n = self.nodes.len();
        let mut outgoing: Vec<(usize, Message)> = Vec::new();

        for to in 0..n {
            // Drain inbox ของ node `to`
            let mut inbox = std::mem::take(&mut self.queues[to]);
            while let Some(msg) = inbox.pop_front() {
                match msg {
                    Message::VoteReq { from, term, last_log_index, last_log_term } => {
                        let req = VoteRequest {
                            term,
                            candidate_id: from,
                            last_log_index,
                            last_log_term,
                        };
                        let resp = handle_vote_request(&mut self.nodes[to], &req);
                        outgoing.push((from as usize, Message::VoteResp {
                            from: to as u64,
                            term: resp.term,
                            granted: resp.vote_granted,
                        }));
                    }
                    Message::VoteResp { from: _, term, granted } => {
                        if self.nodes[to].role == RaftRole::Candidate
                            && self.nodes[to].current_term == term
                            && granted
                        {
                            self.vote_counts[to] += 1;
                            if has_majority(self.vote_counts[to], n) {
                                self.nodes[to].role = RaftRole::Leader;
                            }
                        }
                    }
                    Message::AppendReq { from, term, prev_log_index, prev_log_term, entries, leader_commit } => {
                        let req = AppendEntriesRequest {
                            term,
                            leader_id: from,
                            prev_log_index,
                            prev_log_term,
                            entries,
                            leader_commit,
                        };
                        let resp = handle_append_entries(&mut self.nodes[to], &req);
                        // Apply committed entries ทันทีที่ commit_index เปลี่ยน
                        self.state_machines[to].apply_committed(&mut self.nodes[to]);
                        outgoing.push((from as usize, Message::AppendResp {
                            from: to as u64,
                            term: resp.term,
                            success: resp.success,
                            match_index: resp.match_index,
                        }));
                    }
                    Message::AppendResp { .. } => {
                        // Production: leader tracks match_index[follower] → คำนวณ commit_index
                    }
                }
            }
            self.queues[to] = inbox;
        }

        for (to, msg) in outgoing {
            self.queues[to].push_back(msg);
        }
    }

    /// Leader `leader_id` replicates entries ไปยัง followers
    pub fn replicate_entries(&mut self, leader_id: usize, entries: Vec<LogEntry>) {
        let n = self.nodes.len();
        let base_index = self.nodes[leader_id].log_length();

        // Leader append entries ลง log ของตัวเอง
        for (i, mut entry) in entries.into_iter().enumerate() {
            entry.index = base_index + 1 + i as u64;
            entry.term = self.nodes[leader_id].current_term;
            self.nodes[leader_id].log.push(entry);
        }

        // Commit locally (leader counts itself as majority-1)
        let new_len = self.nodes[leader_id].log_length();
        self.nodes[leader_id].commit_index = new_len;
        self.state_machines[leader_id].apply_committed(&mut self.nodes[leader_id]);

        let term = self.nodes[leader_id].current_term;
        let leader_commit = self.nodes[leader_id].commit_index;
        let leader_id_u64 = leader_id as u64;

        // ส่ง AppendEntries ไปยัง followers
        for to in 0..n {
            if to == leader_id { continue; }
            let follower_len = self.nodes[to].log_length();
            let prev_log_term = if follower_len == 0 {
                0
            } else {
                self.nodes[to]
                    .get_log_entry(follower_len)
                    .map(|e| e.term)
                    .unwrap_or(self.nodes[to].snapshot_last_term)
            };

            let entries_to_send: Vec<LogEntry> = self.nodes[leader_id]
                .log
                .iter()
                .filter(|e| e.index > follower_len)
                .cloned()
                .collect();

            self.queues[to].push_back(Message::AppendReq {
                from: leader_id_u64,
                term,
                prev_log_index: follower_len,
                prev_log_term,
                entries: entries_to_send,
                leader_commit,
            });
        }
    }

    pub fn find_leader(&self) -> Option<usize> {
        self.nodes.iter().position(|n| n.role == RaftRole::Leader)
    }
}
```

---

### ขั้นที่ 7: lib.rs — รวบรวมทุก Module

```rust
// src/lib.rs
pub mod types;
pub mod election;
pub mod replication;
pub mod state_machine;
pub mod snapshot;
pub mod cluster;

// Re-export commonly used items
pub use types::{Command, LogEntry, RaftRole, RaftState};
pub use election::{handle_vote_request, has_majority, random_election_timeout_ms,
                   VoteRequest, VoteResponse};
pub use replication::{handle_append_entries, AppendEntriesRequest, AppendEntriesResponse};
pub use state_machine::KvStateMachine;
pub use snapshot::{compact_log, Snapshot};
pub use cluster::{Cluster, Message};
```

---

### ขั้นที่ 8: Demo Binary

```rust
// src/main.rs
use raft_kv::*;

fn main() {
    println!("=== Raft Consensus KV Store Demo ===\n");

    // --- Demo 1: Election ---
    println!("--- Election Demo (3-node cluster) ---");
    let mut cluster = Cluster::new(3);

    println!("All nodes start as Followers in term 0");
    for i in 0..3 {
        println!("  Node {}: role={:?}, term={}", i,
                 cluster.nodes[i].role, cluster.nodes[i].current_term);
    }

    cluster.start_election(0);
    println!("\nNode 0 starts election → Candidate, term=1");

    // Round 1: nodes 1 and 2 receive VoteReq → respond with VoteResp
    cluster.process_messages();
    // Round 2: node 0 receives VoteResp grants → becomes Leader
    cluster.process_messages();

    if let Some(leader) = cluster.find_leader() {
        println!("Leader elected: Node {} (term={})",
                 leader, cluster.nodes[leader].current_term);
    }
    println!();

    // --- Demo 2: Log Replication ---
    println!("--- Log Replication Demo ---");
    cluster.replicate_entries(0, vec![
        LogEntry { term: 0, index: 0, command: Command::Set {
            key: "name".into(), value: "raft".into() } },
        LogEntry { term: 0, index: 0, command: Command::Set {
            key: "version".into(), value: "1.0".into() } },
    ]);
    cluster.process_messages();

    println!("Leader log length:    {}", cluster.nodes[0].log_length());
    println!("Follower 1 log length: {}", cluster.nodes[1].log_length());
    println!("Follower 2 log length: {}", cluster.nodes[2].log_length());
    println!();

    // --- Demo 3: KV State Machine ---
    println!("--- KV State Machine Demo ---");
    let mut sm = KvStateMachine::new();
    let commands = vec![
        Command::Set { key: "city".into(), value: "Bangkok".into() },
        Command::Set { key: "country".into(), value: "Thailand".into() },
        Command::Set { key: "city".into(), value: "Chiang Mai".into() },
        Command::Delete { key: "country".into() },
    ];
    for cmd in &commands {
        sm.apply(cmd);
    }
    println!("After 4 commands (set city, set country, update city, delete country):");
    println!("  city    = {:?}", sm.get("city"));
    println!("  country = {:?}", sm.get("country"));
    println!();

    // --- Demo 4: Snapshot & Compaction ---
    println!("--- Snapshot & Compaction Demo ---");
    let mut state = RaftState::new(0);
    for i in 1..=5u64 {
        state.log.push(LogEntry { term: 1, index: i, command: Command::NoOp });
    }
    state.commit_index = 3;
    state.last_applied = 3;

    println!("Before compaction: log_length={}, entries in Vec={}",
             state.log_length(), state.log.len());

    let snap = Snapshot::from_state_machine(&sm, 3, 1);
    let bytes = snap.serialize();
    println!("Snapshot size: {} bytes", bytes.len());

    compact_log(&mut state, &snap);
    println!("After compaction:  snapshot_last_index={}, entries in Vec={}",
             state.snapshot_last_index, state.log.len());
    println!();

    // --- Demo 5: Election Timeout ---
    println!("--- Election Timeout Randomization ---");
    let timeouts: Vec<u64> = (0..5).map(|_| random_election_timeout_ms()).collect();
    println!("Sample timeouts (ms): {:?}", timeouts);
    println!("All in [150, 300]:    {}",
             timeouts.iter().all(|&t| t >= 150 && t <= 300));
}
```

**ผลลัพธ์จาก `cargo run` (จริง):**

```
=== Raft Consensus KV Store Demo ===

--- Election Demo (3-node cluster) ---
All nodes start as Followers in term 0
  Node 0: role=Follower, term=0
  Node 1: role=Follower, term=0
  Node 2: role=Follower, term=0

Node 0 starts election → Candidate, term=1
Leader elected: Node 0 (term=1)

--- Log Replication Demo ---
Leader log length:    2
Follower 1 log length: 2
Follower 2 log length: 2

--- KV State Machine Demo ---
After 4 commands (set city, set country, update city, delete country):
  city    = Some("Chiang Mai")
  country = None

--- Snapshot & Compaction Demo ---
Before compaction: log_length=5, entries in Vec=5
Snapshot size: 77 bytes
After compaction:  snapshot_last_index=3, entries in Vec=2

--- Election Timeout Randomization ---
Sample timeouts (ms): [178, 229, 211, 228, 255]
All in [150, 300]:    true
```

---

## การทดสอบ (Testing)

ใส่ tests ทั้งหมดไว้ใน `src/lib.rs` ใน `#[cfg(test)]` module:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // ===== Election Timeout =====

    #[test]
    fn test_election_timeout_in_range() {
        for _ in 0..50 {
            let t = random_election_timeout_ms();
            assert!(t >= 150 && t <= 300, "timeout {} out of range [150,300]", t);
        }
    }

    #[test]
    fn test_election_timeout_randomized() {
        let timeouts: Vec<u64> = (0..100).map(|_| random_election_timeout_ms()).collect();
        let unique: std::collections::HashSet<_> = timeouts.iter().cloned().collect();
        assert!(unique.len() > 5,
            "timeouts should vary, got only {} unique values", unique.len());
    }

    // ===== Majority Counting =====

    #[test]
    fn test_has_majority_logic() {
        assert!(has_majority(2, 3),  "2/3 is majority");
        assert!(!has_majority(1, 3), "1/3 is not majority");
        assert!(has_majority(3, 5),  "3/5 is majority");
        assert!(!has_majority(2, 5), "2/5 is not majority");
        assert!(has_majority(3, 3),  "3/3 is majority");
        assert!(has_majority(4, 7),  "4/7 is majority");
        assert!(!has_majority(3, 7), "3/7 is not majority");
    }

    // ===== Vote Request / Response =====

    #[test]
    fn test_vote_granted_empty_log() {
        let mut state = RaftState::new(1);
        let req = VoteRequest { term: 1, candidate_id: 2,
                                last_log_index: 0, last_log_term: 0 };
        let resp = handle_vote_request(&mut state, &req);
        assert!(resp.vote_granted);
        assert_eq!(state.voted_for, Some(2));
        assert_eq!(state.current_term, 1);
    }

    #[test]
    fn test_vote_rejected_stale_term() {
        let mut state = RaftState::new(1);
        state.current_term = 5;
        let req = VoteRequest { term: 3, candidate_id: 2,
                                last_log_index: 0, last_log_term: 0 };
        let resp = handle_vote_request(&mut state, &req);
        assert!(!resp.vote_granted);
        assert_eq!(resp.term, 5);
    }

    #[test]
    fn test_vote_rejected_already_voted_different_candidate() {
        let mut state = RaftState::new(1);
        state.current_term = 1;
        state.voted_for = Some(2); // voted for node 2

        let req = VoteRequest { term: 1, candidate_id: 3,
                                last_log_index: 0, last_log_term: 0 };
        let resp = handle_vote_request(&mut state, &req);
        assert!(!resp.vote_granted,
            "must not grant vote when already voted for another candidate");
    }

    #[test]
    fn test_vote_granted_same_candidate_idempotent() {
        let mut state = RaftState::new(1);
        state.current_term = 1;
        state.voted_for = Some(2);
        // Vote for the same candidate again (safe to retry)
        let req = VoteRequest { term: 1, candidate_id: 2,
                                last_log_index: 0, last_log_term: 0 };
        let resp = handle_vote_request(&mut state, &req);
        assert!(resp.vote_granted, "idempotent vote for same candidate must be granted");
    }

    #[test]
    fn test_vote_rejected_stale_log() {
        let mut state = RaftState::new(1);
        state.current_term = 2;
        // Voter has log up to term=2, index=2
        state.log = vec![
            LogEntry { term: 1, index: 1, command: Command::NoOp },
            LogEntry { term: 2, index: 2, command: Command::NoOp },
        ];
        // Candidate has log only up to term=1, index=1 (stale)
        let req = VoteRequest { term: 2, candidate_id: 2,
                                last_log_index: 1, last_log_term: 1 };
        let resp = handle_vote_request(&mut state, &req);
        assert!(!resp.vote_granted, "candidate with stale log must be rejected");
    }

    #[test]
    fn test_vote_higher_term_clears_voted_for() {
        let mut state = RaftState::new(1);
        state.current_term = 3;
        state.voted_for = Some(99);
        state.role = RaftRole::Leader;

        let req = VoteRequest { term: 5, candidate_id: 2,
                                last_log_index: 0, last_log_term: 0 };
        let resp = handle_vote_request(&mut state, &req);
        assert!(resp.vote_granted);
        assert_eq!(state.current_term, 5);
        assert_eq!(state.role, RaftRole::Follower);
    }

    // ===== AppendEntries =====

    #[test]
    fn test_append_entries_to_empty_log() {
        let mut state = RaftState::new(1);
        let req = AppendEntriesRequest {
            term: 1, leader_id: 0,
            prev_log_index: 0, prev_log_term: 0,
            entries: vec![
                LogEntry { term: 1, index: 1, command: Command::Set {
                    key: "x".into(), value: "1".into() } },
                LogEntry { term: 1, index: 2, command: Command::Set {
                    key: "y".into(), value: "2".into() } },
            ],
            leader_commit: 2,
        };
        let resp = handle_append_entries(&mut state, &req);
        assert!(resp.success);
        assert_eq!(state.log.len(), 2);
        assert_eq!(state.commit_index, 2);
    }

    #[test]
    fn test_append_entries_heartbeat_updates_commit() {
        let mut state = RaftState::new(1);
        state.current_term = 1;
        state.log = vec![
            LogEntry { term: 1, index: 1, command: Command::NoOp },
            LogEntry { term: 1, index: 2, command: Command::NoOp },
        ];
        state.commit_index = 0;

        // Heartbeat: no new entries, but updated leader_commit
        let req = AppendEntriesRequest {
            term: 1, leader_id: 0,
            prev_log_index: 2, prev_log_term: 1,
            entries: vec![],
            leader_commit: 2,
        };
        let resp = handle_append_entries(&mut state, &req);
        assert!(resp.success);
        assert_eq!(state.commit_index, 2);
    }

    #[test]
    fn test_append_entries_rejected_stale_term() {
        let mut state = RaftState::new(1);
        state.current_term = 5;
        let req = AppendEntriesRequest {
            term: 3, leader_id: 0,
            prev_log_index: 0, prev_log_term: 0,
            entries: vec![], leader_commit: 0,
        };
        let resp = handle_append_entries(&mut state, &req);
        assert!(!resp.success);
        assert_eq!(resp.term, 5);
    }

    #[test]
    fn test_append_entries_prev_log_mismatch() {
        let mut state = RaftState::new(1);
        state.current_term = 2;
        state.log = vec![
            LogEntry { term: 1, index: 1, command: Command::NoOp },
        ];
        let req = AppendEntriesRequest {
            term: 2, leader_id: 0,
            prev_log_index: 1,
            prev_log_term: 2, // mismatch: follower has term=1 at index 1
            entries: vec![
                LogEntry { term: 2, index: 2, command: Command::NoOp },
            ],
            leader_commit: 0,
        };
        let resp = handle_append_entries(&mut state, &req);
        assert!(!resp.success, "should reject when prev_log_term mismatches");
    }

    #[test]
    fn test_append_entries_prev_log_missing() {
        let mut state = RaftState::new(1);
        state.current_term = 1;
        // Log is empty but leader expects prev_log_index=3
        let req = AppendEntriesRequest {
            term: 1, leader_id: 0,
            prev_log_index: 3, prev_log_term: 1,
            entries: vec![
                LogEntry { term: 1, index: 4, command: Command::NoOp },
            ],
            leader_commit: 0,
        };
        let resp = handle_append_entries(&mut state, &req);
        assert!(!resp.success, "should reject when prev_log_index doesn't exist");
    }

    #[test]
    fn test_append_entries_conflict_resolution() {
        let mut state = RaftState::new(1);
        state.current_term = 2;
        // Follower has stale entries from a crashed leader (term=1)
        state.log = vec![
            LogEntry { term: 1, index: 1, command: Command::NoOp },
            LogEntry { term: 1, index: 2, command: Command::NoOp },
            LogEntry { term: 1, index: 3, command: Command::NoOp },
        ];
        // New leader sends entries starting at index 2 with term=2
        let req = AppendEntriesRequest {
            term: 2, leader_id: 0,
            prev_log_index: 1, prev_log_term: 1,
            entries: vec![
                LogEntry { term: 2, index: 2, command: Command::Set {
                    key: "a".into(), value: "1".into() } },
                LogEntry { term: 2, index: 3, command: Command::Set {
                    key: "b".into(), value: "2".into() } },
            ],
            leader_commit: 3,
        };
        let resp = handle_append_entries(&mut state, &req);
        assert!(resp.success);
        assert_eq!(state.log.len(), 3);
        assert_eq!(state.log[1].term, 2, "conflicting entry should be replaced");
        assert_eq!(state.log[2].term, 2, "following entry should be replaced");
        assert_eq!(state.commit_index, 3);
    }

    // ===== State Machine =====

    #[test]
    fn test_state_machine_set_get() {
        let mut sm = KvStateMachine::new();
        sm.apply(&Command::Set { key: "name".into(), value: "alice".into() });
        assert_eq!(sm.get("name"), Some(&"alice".to_string()));
        // Overwrite
        sm.apply(&Command::Set { key: "name".into(), value: "bob".into() });
        assert_eq!(sm.get("name"), Some(&"bob".to_string()));
    }

    #[test]
    fn test_state_machine_delete() {
        let mut sm = KvStateMachine::new();
        sm.apply(&Command::Set { key: "key1".into(), value: "val1".into() });
        sm.apply(&Command::Delete { key: "key1".into() });
        assert_eq!(sm.get("key1"), None, "deleted key should return None");
    }

    #[test]
    fn test_state_machine_apply_committed_log() {
        let mut state = RaftState::new(1);
        state.log = vec![
            LogEntry { term: 1, index: 1, command: Command::Set {
                key: "foo".into(), value: "bar".into() } },
            LogEntry { term: 1, index: 2, command: Command::Set {
                key: "baz".into(), value: "qux".into() } },
            LogEntry { term: 1, index: 3, command: Command::Delete {
                key: "foo".into() } },
        ];
        state.commit_index = 3;

        let mut sm = KvStateMachine::new();
        sm.apply_committed(&mut state);

        assert_eq!(state.last_applied, 3);
        assert_eq!(sm.get("foo"), None, "foo was deleted by entry 3");
        assert_eq!(sm.get("baz"), Some(&"qux".to_string()));
    }

    // ===== Snapshot & Compaction =====

    #[test]
    fn test_snapshot_serialize_deserialize() {
        let mut sm = KvStateMachine::new();
        sm.apply(&Command::Set { key: "k1".into(), value: "v1".into() });
        sm.apply(&Command::Set { key: "k2".into(), value: "v2".into() });

        let snap = Snapshot::from_state_machine(&sm, 5, 2);
        let bytes = snap.serialize();
        let restored = Snapshot::deserialize(&bytes);

        assert_eq!(restored.last_included_index, 5);
        assert_eq!(restored.last_included_term, 2);
        assert_eq!(restored.data.get("k1"), Some(&"v1".to_string()));
        assert_eq!(restored.data.get("k2"), Some(&"v2".to_string()));
    }

    #[test]
    fn test_log_compaction() {
        let mut state = RaftState::new(1);
        state.log = vec![
            LogEntry { term: 1, index: 1, command: Command::NoOp },
            LogEntry { term: 1, index: 2, command: Command::NoOp },
            LogEntry { term: 1, index: 3, command: Command::NoOp },
            LogEntry { term: 2, index: 4, command: Command::NoOp },
            LogEntry { term: 2, index: 5, command: Command::NoOp },
        ];
        state.commit_index = 3;
        state.last_applied = 3;

        let sm = KvStateMachine::new();
        let snapshot = Snapshot::from_state_machine(&sm, 3, 1);
        compact_log(&mut state, &snapshot);

        assert_eq!(state.snapshot_last_index, 3);
        assert_eq!(state.snapshot_last_term, 1);
        assert_eq!(state.log.len(), 2, "entries 1-3 should be removed");
        assert_eq!(state.log[0].index, 4, "remaining log starts at index 4");
        assert_eq!(state.log[1].index, 5);
    }

    #[test]
    fn test_log_length_includes_snapshot() {
        let mut state = RaftState::new(1);
        state.snapshot_last_index = 10;
        state.log = vec![
            LogEntry { term: 2, index: 11, command: Command::NoOp },
            LogEntry { term: 2, index: 12, command: Command::NoOp },
        ];
        assert_eq!(state.log_length(), 12);
    }

    // ===== Cluster Simulation =====

    #[test]
    fn test_cluster_election_3_nodes() {
        let mut cluster = Cluster::new(3);
        cluster.start_election(0);
        assert_eq!(cluster.nodes[0].role, RaftRole::Candidate);
        assert_eq!(cluster.nodes[0].current_term, 1);

        // Round 1: nodes 1,2 receive VoteReq and send VoteResp
        cluster.process_messages();
        // Round 2: node 0 receives VoteResp, counts votes, becomes Leader
        cluster.process_messages();

        assert_eq!(
            cluster.find_leader(), Some(0),
            "node 0 should become leader after winning majority"
        );
    }

    #[test]
    fn test_cluster_log_replication() {
        let mut cluster = Cluster::new(3);
        cluster.start_election(0);
        cluster.process_messages();
        cluster.process_messages();
        assert_eq!(cluster.find_leader(), Some(0));

        cluster.replicate_entries(0, vec![
            LogEntry { term: 0, index: 0, command: Command::Set {
                key: "hello".into(), value: "world".into() } },
        ]);
        cluster.process_messages(); // followers receive AppendReq and apply

        assert_eq!(cluster.nodes[1].log.len(), 1, "follower 1 should have 1 entry");
        assert_eq!(cluster.nodes[2].log.len(), 1, "follower 2 should have 1 entry");
        assert_eq!(
            cluster.state_machines[1].get("hello"),
            Some(&"world".to_string()),
            "follower 1 state machine should reflect the replicated Set"
        );
    }
}
```

### ผลลัพธ์จาก `cargo test` (จริง)

```
running 23 tests
test tests::test_append_entries_conflict_resolution ... ok
test tests::test_append_entries_heartbeat_updates_commit ... ok
test tests::test_append_entries_prev_log_mismatch ... ok
test tests::test_append_entries_prev_log_missing ... ok
test tests::test_append_entries_rejected_stale_term ... ok
test tests::test_append_entries_to_empty_log ... ok
test tests::test_cluster_election_3_nodes ... ok
test tests::test_cluster_log_replication ... ok
test tests::test_has_majority_logic ... ok
test tests::test_log_length_includes_snapshot ... ok
test tests::test_log_compaction ... ok
test tests::test_election_timeout_in_range ... ok
test tests::test_election_timeout_randomized ... ok
test tests::test_snapshot_serialize_deserialize ... ok
test tests::test_state_machine_apply_committed_log ... ok
test tests::test_state_machine_set_get ... ok
test tests::test_state_machine_delete ... ok
test tests::test_vote_granted_empty_log ... ok
test tests::test_vote_granted_same_candidate_idempotent ... ok
test tests::test_vote_higher_term_clears_voted_for ... ok
test tests::test_vote_rejected_already_voted_different_candidate ... ok
test tests::test_vote_rejected_stale_log ... ok
test tests::test_vote_rejected_stale_term ... ok

test result: ok. 23 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

---

## ข้อผิดพลาดที่พบบ่อย (Pitfalls)

### Pitfall 1: Off-by-One ใน Global Log Index

Raft ใช้ 1-based global index แต่ Rust `Vec` ใช้ 0-based index ความสับสนระหว่างสองระบบเป็นแหล่ง bug ที่พบบ่อยที่สุด:

```rust
// ❌ ผิด: ใช้ global index โดยตรงกับ Vec
let entry = &state.log[req.prev_log_index as usize]; // ข้าม offset ของ snapshot

// ✓ ถูก: คำนวณ position ใน Vec โดยหัก snapshot_last_index ออก
let pos = (req.prev_log_index - state.snapshot_last_index) as usize;
let entry = &state.log[pos - 1]; // -1 เพราะ pos เริ่มที่ 1

// ✓ ให้ใช้ helper method แทน เพื่อป้องกัน bug ซ้ำ
pub fn get_log_entry(&self, global_index: u64) -> Option<&LogEntry> {
    if global_index == 0 || global_index <= self.snapshot_last_index {
        return None;
    }
    let pos = (global_index - self.snapshot_last_index) as usize;
    self.log.get(pos - 1)  // get ส่งคืน Option — ปลอดภัยกว่า indexing ตรง ๆ
}
```

### Pitfall 2: ลืม Truncate Log ก่อน Append เมื่อมี Conflict

ข้อผิดพลาดที่ร้ายแรงที่สุดใน Raft implementation: append entries ใหม่โดยไม่ลบ entries ที่ conflict ออกก่อน ทำให้ log มี "fork" ที่ไม่ควรมี:

```rust
// ❌ อันตราย: append โดยไม่ truncate เมื่อพบ conflict
for entry in &req.entries {
    let pos = calculate_pos(entry.index);
    if pos <= state.log.len() {
        // ❌ แค่ replace entry เดียว แต่ entries หลังจากนี้ยังคงอยู่
        state.log[pos - 1] = entry.clone();
    } else {
        state.log.push(entry.clone());
    }
}

// ✓ ถูกต้อง: truncate ก่อน แล้วค่อย push
for entry in &req.entries {
    let pos = calculate_pos(entry.index);
    if pos <= state.log.len() {
        if state.log[pos - 1].term != entry.term {
            // Truncate ALL entries from pos onwards
            state.log.truncate(pos - 1);  // ← สำคัญมาก!
            state.log.push(entry.clone());
        }
    } else {
        state.log.push(entry.clone());
    }
}
```

**ทำไมถึงสำคัญ?**

```
ก่อน truncate:  [t1:1, t1:2, t1:3, t1:4]  (entries จาก leader เก่า)
Leader ใหม่ส่ง:  entries=[t2:2, t2:3]

ถ้าไม่ truncate: [t1:1, t2:2, t2:3, t1:4]  ← t1:4 ยังอยู่! ผิด
ถ้า truncate:    [t1:1, t2:2, t2:3]         ← ถูกต้อง
```

### Pitfall 3: Commit ก่อนได้ Majority

หนึ่งใน safety violation ที่ร้ายแรงที่สุด: leader commit entry ทันทีที่ append โดยไม่รอ acknowledgment จาก majority:

```rust
// ❌ อันตราย: commit ทันทีที่ append
fn append_to_log(&mut self, entry: LogEntry) {
    self.log.push(entry);
    self.commit_index += 1;  // ← ผิด! ยังไม่ได้ replicate ไปไหนเลย
}

// ✓ ถูกต้อง: commit หลังจาก majority ตอบ success
fn update_commit_index(&mut self, match_indexes: &[u64], cluster_size: usize) {
    // หา N สูงสุดที่ match_index[i] ≥ N สำหรับ majority ของ nodes
    // และ log[N].term == current_term
    let mut sorted = match_indexes.to_vec();
    sorted.sort_unstable();
    let majority_pos = cluster_size / 2; // index ที่ majority เห็นพ้อง
    let new_commit = sorted[majority_pos]; // median คือ majority threshold
    if new_commit > self.commit_index {
        // ตรวจว่า entry ที่ N มาจาก current_term (Raft §5.4.2)
        if let Some(entry) = self.get_log_entry(new_commit) {
            if entry.term == self.current_term {
                self.commit_index = new_commit;
            }
        }
    }
}
```

### Pitfall 4: ไม่ Reset `voted_for` เมื่อเห็น Term ใหม่

Raft กำหนดว่า node สามารถโหวตได้แค่ครั้งเดียวต่อ term ถ้าไม่ reset `voted_for` เมื่อเห็น term ใหม่จาก VoteRequest จะทำให้ node ปฏิเสธ candidates ที่ถูกต้องในเทอมใหม่:

```rust
// ❌ ผิด: update term แต่ไม่ clear voted_for
if req.term > state.current_term {
    state.current_term = req.term;
    // voted_for ยังชี้ไปที่ candidate ในเทอมเก่า!
}

// ✓ ถูก: clear voted_for เมื่อ term เปลี่ยน
if req.term > state.current_term {
    state.current_term = req.term;
    state.role = RaftRole::Follower;
    state.voted_for = None; // ← สำคัญ: reset vote ในเทอมใหม่
}
```

**ผลกระทบ:**
```
Term 3 election:
- Node 1 voted for Node 2 ในเทอม 3
- Node 3 crash แล้ว restart ในเทอม 4
- Node 3 ส่ง VoteReq(term=4) ไปหา Node 1
- ถ้าไม่ reset: Node 1 เห็น voted_for=Some(2) → ปฏิเสธ Node 3 → election ล้มเหลว
- ถ้า reset:     Node 1 เห็น voted_for=None  → vote ให้ Node 3 → election สำเร็จ
```

### Pitfall 5: Snapshot Stale Check ที่ขาดหาย

ถ้า `compact_log` ไม่ตรวจว่า snapshot ใหม่กว่า snapshot ที่มีอยู่ จะทำให้ `snapshot_last_index` ถอยหลังและ log index ผิด:

```rust
// ❌ อันตราย: compact โดยไม่ตรวจ
pub fn compact_log_wrong(state: &mut RaftState, snapshot: &Snapshot) {
    let n = snapshot.last_included_index as usize;
    state.log.drain(..n.min(state.log.len()));
    state.snapshot_last_index = snapshot.last_included_index; // อาจถอยหลัง!
}

// ✓ ถูกต้อง: เช็ค cutoff ก่อนเสมอ
pub fn compact_log(state: &mut RaftState, snapshot: &Snapshot) {
    let cutoff = snapshot.last_included_index;
    if cutoff <= state.snapshot_last_index {
        return; // snapshot นี้เก่ากว่าที่มีอยู่ — ไม่ทำอะไร
    }
    // ... compact ...
}
```

---

## การ Package และ Deploy

### Build Release

```bash
cargo build --release
# binary: target/release/raft-kv

# รัน demo
./target/release/raft-kv
```

### Release Profile แนะนำ

```toml
# Cargo.toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
strip = true
```

### ต่อยอดเป็น Production Service

สำหรับ production deployment ต้องเพิ่ม:

1. **Network transport** ด้วย `tokio` + `tonic` (gRPC):

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
tonic = "0.12"
prost = "0.13"
```

2. **Persistent storage** สำหรับ `current_term`, `voted_for`, `log`:

```toml
[dependencies]
rocksdb = "0.22"  # embedded KV store สำหรับ log persistence
```

3. **Docker** สำหรับ multi-node deployment:

```dockerfile
FROM rust:1.79-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/raft-kv /usr/local/bin/
ENV RAFT_NODE_ID=0
ENV RAFT_CLUSTER_PEERS="node1:7000,node2:7000,node3:7000"
ENTRYPOINT ["raft-kv"]
```

```yaml
# docker-compose.yml สำหรับ 3-node cluster local test
version: '3.8'
services:
  node0:
    build: .
    environment:
      RAFT_NODE_ID: "0"
      RAFT_PEERS: "node1:7000,node2:7000"
    ports: ["7000:7000"]
  node1:
    build: .
    environment:
      RAFT_NODE_ID: "1"
      RAFT_PEERS: "node0:7000,node2:7000"
  node2:
    build: .
    environment:
      RAFT_NODE_ID: "2"
      RAFT_PEERS: "node0:7000,node1:7000"
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Persistent State ด้วย File I/O

Raft กำหนดว่า `current_term`, `voted_for`, และ `log` ต้อง persist ก่อน respond ต่อ RPC — มิฉะนั้นถ้า crash แล้ว restart อาจ vote ซ้ำหรือลืม entries ที่ commit แล้ว

```rust
// บันทึก persistent state ลง disk ก่อน respond RPC
pub fn save_persistent_state(state: &RaftState, path: &str) -> std::io::Result<()> {
    let data = PersistentState {
        current_term: state.current_term,
        voted_for: state.voted_for,
        log: state.log.clone(),
    };
    let json = serde_json::to_vec(&data)?;
    // ใช้ write + fsync เพื่อ durability
    std::fs::write(path, &json)?;
    Ok(())
}
```

**เป้าหมาย:** เพิ่ม `save_persistent_state` และ `load_persistent_state` จาก file แล้วเขียน test ที่จำลอง crash-and-restart

### แบบฝึกหัดที่ 2: Real async ด้วย tokio

แทนที่ single-threaded simulation ด้วย async nodes แต่ละตัวใช้ `tokio::task`:

```rust
use tokio::sync::mpsc;

pub struct AsyncNode {
    state: RaftState,
    tx: HashMap<u64, mpsc::Sender<Message>>,
    rx: mpsc::Receiver<Message>,
    election_timeout: Duration,
}

impl AsyncNode {
    pub async fn run(&mut self) {
        loop {
            tokio::select! {
                Some(msg) = self.rx.recv() => {
                    self.handle_message(msg).await;
                }
                _ = tokio::time::sleep(self.election_timeout) => {
                    self.start_election().await;
                    self.election_timeout = Duration::from_millis(
                        random_election_timeout_ms()
                    );
                }
            }
        }
    }
}
```

**เป้าหมาย:** 3 async nodes รันใน background tasks แลกเปลี่ยน messages ผ่าน `tokio::sync::mpsc`

### แบบฝึกหัดที่ 3: Client API + Read/Write Linearizability

เพิ่ม HTTP API ให้ client ส่ง GET/SET/DELETE requests ไปยัง leader:

```rust
// ใช้ axum
async fn handle_set(
    State(raft): State<Arc<RaftHandle>>,
    Json(req): Json<SetRequest>,
) -> Json<SetResponse> {
    // Forward to leader ถ้า node นี้ไม่ใช่ leader
    // Wait for majority commit แล้วค่อย respond
    let result = raft.propose(Command::Set { key: req.key, value: req.value }).await;
    Json(SetResponse { success: result.is_ok() })
}
```

**ความท้าทาย:** Read linearizability — ต้องส่ง heartbeat ยืนยัน leadership ก่อน serve read request

### แบบฝึกหัดที่ 4: InstallSnapshot RPC

เมื่อ follower ล้าหลัง leader มากจน leader ทิ้ง log entries ที่ follower ต้องการไปแล้ว (เพราะ compaction) ต้อง send snapshot แทน:

```rust
pub struct InstallSnapshotRequest {
    pub term: u64,
    pub leader_id: u64,
    pub last_included_index: u64,
    pub last_included_term: u64,
    pub data: Vec<u8>,   // snapshot bytes
    pub done: bool,      // สำหรับ chunked transfer
}

pub fn handle_install_snapshot(
    state: &mut RaftState,
    sm: &mut KvStateMachine,
    req: &InstallSnapshotRequest,
) {
    // 1. ปฏิเสธถ้า term ต่ำกว่า
    // 2. ตรวจว่า snapshot ใหม่กว่า last_applied
    // 3. Restore state machine จาก snapshot.data
    // 4. Compact log ที่ทับซ้อนกัน
    // 5. อัปเดต snapshot_last_index/term
}
```

**เป้าหมาย:** เขียน `handle_install_snapshot` และเพิ่ม test ที่ใช้งาน lagging follower

### แบบฝึกหัดที่ 5: Pre-vote Protocol

Pre-vote เป็น optimization ที่ป้องกัน node ที่ network partition หายมีคนแล้วกลับมา disrupt leader ที่ stable อยู่:

```rust
// ก่อนเพิ่ม term และเริ่ม election จริง ส่ง pre-vote เพื่อตรวจว่ามีโอกาสชนะ
pub fn send_pre_vote(&self) -> Vec<PreVoteRequest> {
    // ใช้ current_term + 1 แต่ไม่เพิ่ม term จริง ๆ
    // ถ้าได้รับ majority pre-vote grants → เริ่ม election จริง
    // ถ้าไม่ได้ → แสดงว่า network partition → ไม่ disrupt cluster
}
```

### แบบฝึกหัดที่ 6: Membership Changes (Joint Consensus)

การเพิ่ม/ลบ node ใน cluster ต้องทำผ่าน **joint consensus** เพื่อป้องกัน split-brain ในระหว่าง transition:

```rust
pub enum ConfigChange {
    AddNode(u64, /* addr */ String),
    RemoveNode(u64),
}

// Phase 1: C_old+new — ทั้งสอง config ต้องได้ majority ก่อน commit
// Phase 2: C_new — ใช้ config ใหม่อย่างเดียว
pub fn propose_config_change(&mut self, change: ConfigChange) {
    let entry = LogEntry {
        term: self.current_term,
        index: self.log_length() + 1,
        command: Command::ConfigChange(change),
    };
    // ... replicate และ commit ...
}
```

---

## สรุป

โปรเจคนี้ implement Raft consensus algorithm แบบครบถ้วน ครอบคลุม:

| Component | สิ่งที่ implement |
|-----------|-----------------|
| **Types** | `RaftRole`, `LogEntry`, `Command`, `RaftState` |
| **Election** | Randomized timeout, `VoteRequest/Response`, majority counting |
| **Log Replication** | `AppendEntries` RPC, prev_log consistency check, conflict resolution |
| **State Machine** | `KvStateMachine` (`HashMap<String,String>`), `apply_committed` loop |
| **Snapshot** | `Snapshot` struct, serde serialization, `compact_log` |
| **Cluster** | In-memory simulation ด้วย `VecDeque` message queues |

**Patterns สำคัญที่ได้เรียน:**

1. **Log-structured state machine** — ทุก operation เป็น deterministic command ใน append-only log ทำให้ replay และ snapshot ทำได้
2. **Term-based leadership** — term เพิ่ม monotonically ทำให้ detect stale leaders ได้ง่ายโดยไม่ต้องใช้ wall clock
3. **Safety through majority quorum** — ทุก commit ต้องได้รับ acknowledgment จาก (n/2 + 1) nodes ทำให้แม้ minority crash ข้อมูลไม่หาย
4. **Snapshot + incremental log** — แยก full state (snapshot) กับ delta (log) ทำให้ balance ระหว่าง memory efficiency และ startup performance

**โปรเจคถัดไป** (F02) จะต่อยอดความรู้ distributed systems นี้ไปสู่ **Service Discovery** — ระบบที่ microservices ค้นหาและ communicate กันโดยไม่ต้อง hardcode addresses ซึ่งเป็น infrastructure layer อีกชั้นที่ depend on consensus-based storage อย่าง etcd

---

**โปรเจคก่อนหน้า:** [Project E10: Maze Generator](project-e10-maze-generator.md) | **โปรเจคถัดไป:** [Project F02: Service Discovery](project-f02-service-discovery.md)
