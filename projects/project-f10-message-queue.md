# Project F10: Persistent Message Queue

> โมดูล: F — Distributed Systems | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

## ภาพรวมโปรเจค

ในระบบ distributed สมัยใหม่ **message queue** คือกาวที่เชื่อม services เข้าหากัน ไม่ว่าจะเป็น
e-commerce ที่ต้องกระจาย order events ไปยัง inventory, billing, และ notification service พร้อมกัน,
ระบบ analytics ที่ต้องรวบรวม clickstream จาก millions ของ users, หรือ IoT platform ที่รับ sensor
data จาก devices นับพัน

**Apache Kafka** กลายเป็น standard de facto สำหรับ use case เหล่านี้ ด้วย design ที่เรียบง่ายแต่
ทรงพลัง: log ที่ append-only, partitioned เพื่อ scalability, replicated เพื่อ durability และ consumer
groups ที่อ่านแบบ pull-based เพื่อ backpressure โดยธรรมชาติ

โปรเจคนี้จะสร้าง **Persistent Message Queue** ที่ได้แรงบันดาลใจจาก Kafka ทำงานบน local filesystem
ประกอบด้วย:

- **Log Segment** — append-only file พร้อม sparse index สำหรับ seek ที่มีประสิทธิภาพ
- **Topic/Partition** — จัดกลุ่ม messages ตาม topic และกระจายงานผ่าน partition
- **Producer** — เลือก partition ด้วย `hash(key) % partition_count` และเขียนลง log
- **Consumer Group** — ติดตาม offset แต่ละ `(topic, partition)` และ assign งานแบบ round-robin
- **Pull Consumer** — `poll()` อ่านจาก committed offset, `commit_offsets()` บันทึกความคืบหน้า
- **Retention Policy** — background task ลบ segments ที่เก่าเกินกำหนดอัตโนมัติ

**Use case จริงในโลก production:**
- **Event streaming**: กระจาย domain events (OrderCreated, PaymentCompleted) ข้าม microservices
- **Log aggregation**: รวบรวม application logs จากหลาย instances เพื่อ centralized processing
- **Activity tracking**: เก็บ user clickstream สำหรับ real-time analytics
- **Change Data Capture (CDC)**: stream database changes ไปยัง downstream systems
- **Work queue**: กระจาย heavy jobs ให้ worker pool ด้วย at-least-once guarantee

**ทำไมไม่ใช้ Kafka โดยตรง?** การสร้างจากศูนย์ทำให้เข้าใจ internals ที่ Kafka ซ่อนไว้:
ทำไม consumer ต้อง pull (ไม่ใช่ push), ทำไม offset จึงเป็นแค่ integer เดียว,
ทำไม partition จึง immutable, และ retention ทำงานอย่างไรโดยไม่ lock consumer

## สิ่งที่จะได้เรียนรู้

- **Append-only log** — design พื้นฐานของ Kafka, LevelDB, และ write-ahead logs ใน databases
- **Sparse index** — ลด index size ด้วยการบันทึก `(offset, file_position)` ทุก N bytes แทนทุก message
- **Length-prefix framing** — encode binary records ด้วย `u64::to_le_bytes()` นำหน้า JSON payload
- **Pull-based consumer model** — consumer ควบคุม pace เอง (ต่างจาก push ที่ producer ควบคุม)
- **Consumer group semantics** — offset commit/fetch, round-robin partition assignment, consumer rebalance
- **Segment rolling** — เปิด segment ใหม่เมื่อขนาดเกิน threshold เพื่อ bounded file sizes
- **Retention enforcement** — ลบ old data โดยไม่กระทบ active consumers (เก็บ segment ล่าสุดเสมอ)
- **Tokio async runtime** — รัน background retention task ควบคู่กับ producer/consumer logic

## ความรู้ที่ต้องมีมาก่อน

- **Part 13–17**: Generic types, traits, trait bounds — `Segment::new<P: AsRef<Path>>()`
- **Part 18–22**: Closures, `Fn` traits — ใช้ใน background task
- **Part 23–27**: `Box<dyn Trait>`, `Arc`, dynamic dispatch — ใช้ใน concurrent components
- **Part 35–40**: Error handling, `Result<T, E>`, `?` operator — ใช้ทั่วทั้งโปรเจค
- **Part 46–50**: Iterators, `filter`, `take`, `flat_map` — ใช้ใน read/retention logic
- **Part 55–60**: `serde`, `serde_json` — serialize Message เป็น JSON payload
- **Part 61–65**: File I/O (`std::fs`, `std::io`) — อ่านเขียน log files จริง
- **Part 96–100**: `Arc<Mutex<T>>`, `tokio::spawn` — background retention task
- **Part 101–105**: `tokio` async runtime — `#[tokio::main]`, `tokio::time::interval`

## โครงสร้างโปรเจค (Project Layout)

```
message-queue/
├── src/
│   ├── lib.rs          # module root + unit tests (17 tests)
│   ├── main.rs         # demo binary
│   ├── segment.rs      # Message, IndexEntry, Segment (append-only log)
│   ├── partition.rs    # Partition, Topic
│   ├── producer.rs     # Producer, RecordMetadata
│   ├── consumer.rs     # ConsumerGroup, Consumer, ConsumerRecord
│   └── retention.rs    # RetentionConfig, cleanup_topic
└── Cargo.toml
```

**หมายเหตุเรื่อง design:** โปรเจคนี้ใช้ synchronous file I/O (`std::fs`) เพื่อความเรียบง่าย
ระบบ production จะใช้ `tokio::fs` หรือ `io_uring` (crate `tokio-uring`) เพื่อ non-blocking I/O
แต่ต้องเพิ่มความซับซ้อนมากขึ้น — design นี้เหมาะกับการเรียนรู้ core concepts ก่อน

## การออกแบบ (Architecture & Design)

### Data Flow Overview

```
Producer
    │ send(topic, key, value)
    │ compute_partition = hash(key) % N
    ▼
Topic["orders"]
    ├── Partition[0]
    │       ├── Segment(base=0)   [00000000000000000000.log]
    │       │        ├── msg@offset=0  ← written to end of file
    │       │        └── msg@offset=1
    │       └── Segment(base=2)   [00000000000000000002.log]  ← rolled when prev exceeded limit
    │                └── msg@offset=2
    ├── Partition[1]
    │       └── Segment(base=0)
    └── Partition[2]
            └── Segment(base=0)

Consumer Group "order-processors"
    ├── offsets: { ("orders",0)→3, ("orders",1)→1, ("orders",2)→5 }
    ├── members: ["processor-A", "processor-B"]
    └── assignment: { "processor-A"→[(orders,0),(orders,2)], "processor-B"→[(orders,1)] }

Consumer "processor-A"
    │ poll(max=10, timeout=100ms)
    │   → reads partition 0 from offset 3
    │   → reads partition 2 from offset 5
    │   → returns Vec<ConsumerRecord>
    │ commit_offsets(records)
    │   → group.offsets["orders",0] = max_offset+1
    │   → group.offsets["orders",2] = max_offset+1
    ▼
```

### Binary Format ของ Log File

แต่ละ record ใน `.log` file มีรูปแบบ:

```
┌──────────────────────┬──────────────────────────────────────────┐
│  length: u64 (8 B)   │  JSON payload (length bytes)             │
│  little-endian       │  { offset, timestamp, key, value, hdrs } │
└──────────────────────┴──────────────────────────────────────────┘
```

การใช้ length-prefix framing ทำให้:
- อ่านได้โดยไม่ต้อง scan หา delimiter
- รองรับ binary payload ใน `key` และ `value` (ไม่ต้อง escape)
- Parse เร็วกว่า line-delimited JSON

### Sparse Index

แทนที่จะบันทึก `(offset → file_position)` ทุก record (ซึ่งจะใหญ่มากถ้ามี billions of messages)
เราบันทึกทุก ๆ N bytes เท่านั้น:

```
สมมติ sparse_interval = 512 bytes:

position 0    → IndexEntry { offset: 0,  position: 0   }
position 512  → IndexEntry { offset: 8,  position: 512  }
position 1024 → IndexEntry { offset: 17, position: 1024 }
...

เมื่อต้องการหา offset 10:
1. หา entry ที่ offset ≤ 10 → IndexEntry{ offset:8, position:512 }
2. Seek file ไปที่ position 512
3. อ่าน sequential จนถึง offset 10
```

ใน implementation นี้เราอ่าน sequential ทั้ง segment เพื่อความเรียบง่าย
แต่ sparse index เก็บไว้เพื่อ illustrate concept และสามารถ optimize ได้ใน extension

### Segment Rolling

เมื่อ `Partition::append()` ถูกเรียก จะตรวจสอบ:
```
last_segment.size_bytes >= segment_size_limit
    → สร้าง Segment ใหม่ด้วย base_offset = last_segment.next_offset
```

ขนาด segment แนะนำใน production: **1 GB** (Kafka default)
ในโปรเจคนี้ใช้ค่าเล็กสำหรับ test

### Offset Semantics

offset ที่ commit ใน Consumer Group คือ **"offset ถัดไปที่จะอ่าน"** (Kafka convention)
ไม่ใช่ offset สุดท้ายที่อ่าน:

```
อ่าน messages ที่ offset 0, 1, 2
commit_offsets → บันทึก offset = 3
poll ครั้งถัดไป → เริ่มอ่านจาก offset 3
```

### Retention Strategy

Retention ทำงานที่ระดับ segment (ไม่ใช่ message):
```
ลบ segment ถ้า: segment.created_at < now - max_age
เงื่อนไข: เก็บ segment ล่าสุดเสมอ (active write segment)
```

การทำงานที่ segment boundary ทำให้ delete เป็น O(1) file delete
แทนที่จะต้อง compact file (ซึ่งจะกระทบ read performance)

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างพื้นฐาน — `Cargo.toml` และ `Message`

เริ่มต้นด้วย data structure หลักและ dependencies

**`Cargo.toml`**

```toml
[package]
name = "message-queue"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
tempfile = "3"
```

เลือก crates เหล่านี้เพราะ:
- `serde` + `serde_json` — serialize `Message` struct เป็น JSON payload สำหรับ portability
  (สามารถเปลี่ยนเป็น `bincode` ภายหลังถ้าต้องการ performance สูงขึ้น)
- `tokio` — async runtime สำหรับ background retention task
- `tempfile` — สร้าง temp directory สำหรับ tests (ลบอัตโนมัติเมื่อ test จบ)

**`src/segment.rs`** — ส่วนที่ 1: Message struct

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::time::{SystemTime, UNIX_EPOCH};

/// ข้อความเดียวที่ถูกบันทึกลงใน log segment
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct Message {
    pub offset: u64,
    pub timestamp: u64,          // milliseconds since Unix epoch
    pub key: Vec<u8>,            // partition key (binary)
    pub value: Vec<u8>,          // message payload (binary)
    pub headers: HashMap<String, String>, // metadata
}

impl Message {
    pub fn new(offset: u64, key: Vec<u8>, value: Vec<u8>) -> Self {
        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_millis() as u64;
        Message {
            offset,
            timestamp,
            key,
            value,
            headers: HashMap::new(),
        }
    }
}
```

**ทำไม `key` และ `value` เป็น `Vec<u8>` ไม่ใช่ `String`?**
เพราะ message payload ในระบบจริงอาจเป็น binary ได้ เช่น Protocol Buffers หรือ Avro
`Vec<u8>` รองรับทั้ง text และ binary ได้ในแบบเดียวกัน

**Sparse Index Entry:**

```rust
/// Sparse index entry: บันทึก (offset, byte_position) ทุก N bytes
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct IndexEntry {
    pub offset: u64,
    pub position: u64,   // byte offset ใน log file
}
```

---

### ขั้นที่ 2: Append-Only Segment พร้อม File I/O

นี่คือส่วนสำคัญที่สุดของระบบ — การเขียนและอ่าน messages จาก disk

**`src/segment.rs`** — ส่วนที่ 2: Segment struct และ append

```rust
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Write};
use std::path::{Path, PathBuf};

pub struct Segment {
    pub base_offset: u64,
    pub log_path: PathBuf,       // path ของ .log file
    pub index_path: PathBuf,     // path ของ .index file
    pub sparse_interval: u64,    // บันทึก index ทุกกี่ bytes
    pub sparse_index: Vec<IndexEntry>,
    pub next_offset: u64,        // offset ถัดไปที่จะเขียน
    pub size_bytes: u64,         // ขนาดรวมของ log file
    pub created_at: u64,         // Unix timestamp (seconds) เมื่อสร้าง
}

impl Segment {
    /// สร้างหรือเปิด Segment จาก base_offset
    pub fn new(dir: &Path, base_offset: u64, sparse_interval: u64) -> io::Result<Self> {
        let log_path = dir.join(format!("{:020}.log", base_offset));
        let index_path = dir.join(format!("{:020}.index", base_offset));

        let created_at = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();

        let mut sparse_index = Vec::new();
        let mut next_offset = base_offset;
        let mut size_bytes = 0u64;

        // โหลด index ถ้ามีอยู่แล้ว (reopen existing segment)
        if index_path.exists() {
            if let Ok(data) = std::fs::read_to_string(&index_path) {
                if let Ok(entries) = serde_json::from_str::<Vec<IndexEntry>>(&data) {
                    sparse_index = entries;
                }
            }
        }

        // สแกน log file เพื่อหา next_offset
        if log_path.exists() {
            let meta = std::fs::metadata(&log_path)?;
            size_bytes = meta.len();
            let msgs = Self::read_all_from_path(&log_path)?;
            if let Some(last) = msgs.last() {
                next_offset = last.offset + 1;
            }
        }

        Ok(Segment {
            base_offset,
            log_path,
            index_path,
            sparse_interval,
            sparse_index,
            next_offset,
            size_bytes,
            created_at,
        })
    }
```

**Method `append` — หัวใจของ Segment:**

```rust
    /// เขียน Message ใหม่ลงท้าย log file (append-only)
    /// Format: [8 bytes length (LE)] [JSON bytes]
    pub fn append(
        &mut self,
        key: Vec<u8>,
        value: Vec<u8>,
        headers: HashMap<String, String>,
    ) -> io::Result<u64> {
        let offset = self.next_offset;
        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_millis() as u64;

        let msg = Message { offset, timestamp, key, value, headers };

        // Serialize เป็น JSON
        let json = serde_json::to_vec(&msg)
            .map_err(|e| io::Error::new(io::ErrorKind::Other, e))?;
        let len = json.len() as u64;

        // เปิด file ด้วย append mode
        let mut file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&self.log_path)?;

        // เขียน length prefix (8 bytes, little-endian)
        file.write_all(&len.to_le_bytes())?;
        // เขียน JSON payload
        file.write_all(&json)?;
        file.flush()?;

        let prev_size = self.size_bytes;
        self.size_bytes += 8 + len;

        // ตรวจว่าถึงรอบ sparse index หรือยัง
        let prev_bucket = prev_size / self.sparse_interval.max(1);
        let new_bucket = self.size_bytes / self.sparse_interval.max(1);

        if self.sparse_index.is_empty() || new_bucket > prev_bucket {
            self.sparse_index.push(IndexEntry {
                offset,
                position: prev_size,
            });
            // บันทึก index ลง disk
            let json_idx = serde_json::to_string(&self.sparse_index)
                .map_err(|e| io::Error::new(io::ErrorKind::Other, e))?;
            std::fs::write(&self.index_path, json_idx)?;
        }

        self.next_offset = offset + 1;
        Ok(offset)
    }
```

**Method `read_from_offset`:**

```rust
    /// อ่าน messages จาก start_offset เป็นต้นไป (จำกัด max_records)
    pub fn read_from_offset(
        &self,
        start_offset: u64,
        max_records: usize,
    ) -> io::Result<Vec<Message>> {
        if !self.log_path.exists() {
            return Ok(Vec::new());
        }
        let all = Self::read_all_from_path(&self.log_path)?;
        Ok(all
            .into_iter()
            .filter(|m| m.offset >= start_offset)
            .take(max_records)
            .collect())
    }

    /// อ่าน messages ทั้งหมดจาก file path
    pub fn read_all_from_path(path: &Path) -> io::Result<Vec<Message>> {
        if !path.exists() {
            return Ok(Vec::new());
        }

        let mut file = File::open(path)?;
        let mut messages = Vec::new();

        loop {
            // อ่าน length prefix
            let mut len_bytes = [0u8; 8];
            match file.read_exact(&mut len_bytes) {
                Ok(_) => {}
                Err(e) if e.kind() == io::ErrorKind::UnexpectedEof => break,
                Err(e) => return Err(e),
            }

            let len = u64::from_le_bytes(len_bytes) as usize;
            let mut json_bytes = vec![0u8; len];
            file.read_exact(&mut json_bytes)?;

            let msg: Message = serde_json::from_slice(&json_bytes)
                .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
            messages.push(msg);
        }

        Ok(messages)
    }
}
```

**ข้อควรระวัง:** `OpenOptions::new().append(true)` ไม่ใช่ `.write(true)` เพราะ `write(true)`
จะ overwrite จาก position 0 ถ้าไม่ได้ `seek` ก่อน ใช้ `.append(true)` เสมอสำหรับ log files

---

### ขั้นที่ 3: Partition และ Topic

**`src/partition.rs`** — Partition จัดการ segments หลายตัว

```rust
use crate::segment::{Message, Segment};
use std::collections::HashMap;
use std::fs;
use std::io;
use std::path::{Path, PathBuf};

pub struct Partition {
    pub id: usize,
    pub dir: PathBuf,
    pub segments: Vec<Segment>,         // segments เรียงตาม base_offset
    pub segment_size_limit: u64,        // rolling threshold (bytes)
    pub sparse_interval: u64,
}

impl Partition {
    pub fn new(dir: &Path, id: usize, segment_size_limit: u64) -> io::Result<Self> {
        fs::create_dir_all(dir)?;
        let sparse_interval = 512;

        // โหลด existing segments จาก .log files ใน directory
        let mut base_offsets: Vec<u64> = Vec::new();
        for entry in fs::read_dir(dir)? {
            let entry = entry?;
            let name = entry.file_name().to_string_lossy().to_string();
            if name.ends_with(".log") {
                if let Ok(offset) = name.trim_end_matches(".log").parse::<u64>() {
                    base_offsets.push(offset);
                }
            }
        }
        base_offsets.sort_unstable();

        let mut segments = Vec::new();
        for base in base_offsets {
            segments.push(Segment::new(dir, base, sparse_interval)?);
        }

        // สร้าง initial segment ถ้ายังไม่มี
        if segments.is_empty() {
            segments.push(Segment::new(dir, 0, sparse_interval)?);
        }

        Ok(Partition {
            id,
            dir: dir.to_path_buf(),
            segments,
            segment_size_limit,
            sparse_interval,
        })
    }

    /// เพิ่ม message เข้า partition (ม้วน segment ใหม่ถ้าจำเป็น)
    pub fn append(
        &mut self,
        key: Vec<u8>,
        value: Vec<u8>,
        headers: HashMap<String, String>,
    ) -> io::Result<u64> {
        // ตรวจว่า active segment เต็มหรือยัง
        let should_roll = self
            .segments
            .last()
            .map(|s| s.size_bytes >= self.segment_size_limit)
            .unwrap_or(false);

        if should_roll {
            let new_base = self.segments.last().unwrap().next_offset;
            self.segments.push(Segment::new(&self.dir, new_base, self.sparse_interval)?);
        }

        self.segments.last_mut().unwrap().append(key, value, headers)
    }

    /// อ่าน messages จาก start_offset ข้ามหลาย segments
    pub fn read_from_offset(
        &self,
        start_offset: u64,
        max_records: usize,
    ) -> io::Result<Vec<Message>> {
        let mut results = Vec::new();

        for seg in &self.segments {
            // ข้าม segments ที่จบก่อน start_offset
            if seg.next_offset <= start_offset {
                continue;
            }
            let remaining = max_records.saturating_sub(results.len());
            if remaining == 0 { break; }
            results.extend(seg.read_from_offset(start_offset, remaining)?);
        }

        Ok(results)
    }

    pub fn next_offset(&self) -> u64 {
        self.segments.last().map(|s| s.next_offset).unwrap_or(0)
    }

    pub fn segment_count(&self) -> usize {
        self.segments.len()
    }

    /// ลบ segments ที่สร้างก่อน before_timestamp (Unix seconds)
    /// เก็บ segment ล่าสุดไว้เสมอ (active write segment)
    pub fn delete_segments_before(&mut self, before_timestamp: u64) -> usize {
        if self.segments.len() <= 1 {
            return 0;
        }
        let last_idx = self.segments.len() - 1;
        let to_delete: Vec<usize> = self.segments
            .iter()
            .enumerate()
            .filter(|(i, s)| *i < last_idx && s.created_at < before_timestamp)
            .map(|(i, _)| i)
            .collect();

        let mut sorted = to_delete;
        sorted.sort_unstable_by(|a, b| b.cmp(a)); // reverse เพื่อลบจากหลังมาหน้า

        let count = sorted.len();
        for i in sorted {
            let seg = self.segments.remove(i);
            let _ = fs::remove_file(&seg.log_path);
            let _ = fs::remove_file(&seg.index_path);
        }
        count
    }
}
```

**`Topic` struct — grouping ของ partitions:**

```rust
/// Topic ประกอบด้วย partitions หลาย ๆ ตัว
pub struct Topic {
    pub name: String,
    pub partitions: Vec<Partition>,
}

impl Topic {
    /// สร้าง Topic หรือเปิด Topic ที่มีอยู่แล้ว
    pub fn new(
        base_dir: &Path,
        name: &str,
        partition_count: usize,
        segment_size_limit: u64,
    ) -> io::Result<Self> {
        let mut partitions = Vec::new();
        for i in 0..partition_count {
            let partition_dir = base_dir.join(name).join(format!("partition-{}", i));
            partitions.push(Partition::new(&partition_dir, i, segment_size_limit)?);
        }
        Ok(Topic {
            name: name.to_string(),
            partitions,
        })
    }

    pub fn partition_count(&self) -> usize {
        self.partitions.len()
    }
}
```

**Directory layout จริงบน disk หลังจาก produce:**

```
/tmp/mq-demo/
└── orders/
    ├── partition-0/
    │   ├── 00000000000000000000.log
    │   └── 00000000000000000000.index
    ├── partition-1/
    │   ├── 00000000000000000000.log
    │   └── 00000000000000000000.index
    └── partition-2/
        ├── 00000000000000000000.log
        └── 00000000000000000000.index
```

ชื่อ file เป็น 20-digit zero-padded offset เพื่อให้ sort ตาม lexicographic order ได้ถูกต้อง

---

### ขั้นที่ 4: Producer — ส่ง Message และเลือก Partition

**`src/producer.rs`**

```rust
use crate::partition::Topic;
use std::collections::hash_map::DefaultHasher;
use std::collections::HashMap;
use std::hash::{Hash, Hasher};
use std::io;
use std::time::{SystemTime, UNIX_EPOCH};

/// Metadata ที่ส่งกลับหลัง produce สำเร็จ
#[derive(Debug, Clone)]
pub struct RecordMetadata {
    pub topic: String,
    pub partition: usize,
    pub offset: u64,
    pub timestamp: u64,
}

pub struct Producer;

impl Producer {
    pub fn new() -> Self { Producer }

    /// ส่ง message — เลือก partition โดยอัตโนมัติจาก hash(key)
    pub fn send(
        &self,
        topic: &mut Topic,
        key: &[u8],
        value: &[u8],
    ) -> io::Result<RecordMetadata> {
        self.send_with_headers(topic, key, value, HashMap::new())
    }

    /// ส่ง message พร้อม custom headers
    pub fn send_with_headers(
        &self,
        topic: &mut Topic,
        key: &[u8],
        value: &[u8],
        headers: HashMap<String, String>,
    ) -> io::Result<RecordMetadata> {
        let partition_count = topic.partitions.len();
        let partition_id = Self::compute_partition(key, partition_count);

        let offset = topic.partitions[partition_id]
            .append(key.to_vec(), value.to_vec(), headers)?;

        let timestamp = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_millis() as u64;

        Ok(RecordMetadata {
            topic: topic.name.clone(),
            partition: partition_id,
            offset,
            timestamp,
        })
    }

    /// hash(key) % partition_count — deterministic partition selection
    pub fn compute_partition(key: &[u8], partition_count: usize) -> usize {
        if partition_count == 0 { return 0; }
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        (hasher.finish() % partition_count as u64) as usize
    }
}

impl Default for Producer {
    fn default() -> Self { Self::new() }
}
```

**ทำไม hash(key) ถึงสำคัญ?** Keys ที่เหมือนกัน (เช่น `user-1001`) จะไปที่ partition เดียวกันเสมอ
ทำให้ messages ของ user คนเดิมถูกอ่านตามลำดับเสมอ — นี่คือ **ordering guarantee ภายใน partition**

**Demo การใช้งาน:**

```rust
let mut topic = Topic::new(&base_dir, "orders", 3, 1024 * 1024)?;
let producer = Producer::new();

// user-1001 จะไปที่ partition เดิมเสมอ
let m1 = producer.send(&mut topic, b"user-1001", b"{\"order\": 1}")?;
let m2 = producer.send(&mut topic, b"user-1001", b"{\"order\": 2}")?;
let m3 = producer.send(&mut topic, b"user-1002", b"{\"order\": 3}")?;

// m1.partition == m2.partition (same key)
// m3.partition อาจต่างจาก m1.partition
println!("user-1001 → partition {}, offset {}", m1.partition, m1.offset);
println!("user-1001 → partition {}, offset {}", m2.partition, m2.offset);
println!("user-1002 → partition {}", m3.partition);
```

---

### ขั้นที่ 5: Consumer Group — Offset Management และ Partition Assignment

**`src/consumer.rs`** — ส่วนที่ 1: ConsumerGroup

```rust
use std::collections::HashMap;

pub type TopicPartition = (String, usize);

/// Consumer Group จัดการ offset และการกระจาย partition
#[derive(Debug, Clone)]
pub struct ConsumerGroup {
    pub group_id: String,
    pub members: Vec<String>,
    /// committed offsets: (topic, partition) → offset ถัดไปที่จะอ่าน
    pub offsets: HashMap<TopicPartition, u64>,
    /// assignment: member_id → Vec<(topic, partition)>
    assignment: HashMap<String, Vec<TopicPartition>>,
}

impl ConsumerGroup {
    pub fn new(group_id: &str) -> Self {
        ConsumerGroup {
            group_id: group_id.to_string(),
            members: Vec::new(),
            offsets: HashMap::new(),
            assignment: HashMap::new(),
        }
    }

    pub fn join(&mut self, member_id: &str) {
        if !self.members.contains(&member_id.to_string()) {
            self.members.push(member_id.to_string());
        }
    }

    pub fn leave(&mut self, member_id: &str) {
        self.members.retain(|m| m != member_id);
        self.assignment.remove(member_id);
    }

    /// Assign partitions แบบ round-robin ให้ทุก member
    /// partition 0 → member 0, partition 1 → member 1, ...
    /// partition N → member (N % member_count)
    pub fn assign_partitions(&mut self, topic: &str, partition_count: usize) {
        if self.members.is_empty() { return; }

        for assignments in self.assignment.values_mut() {
            assignments.retain(|(t, _)| t != topic);
        }

        for i in 0..partition_count {
            let member = self.members[i % self.members.len()].clone();
            self.assignment
                .entry(member)
                .or_insert_with(Vec::new)
                .push((topic.to_string(), i));
        }
    }

    /// บันทึก offset (offset ถัดไปที่จะอ่าน)
    pub fn commit_offset(&mut self, topic: &str, partition: usize, offset: u64) {
        self.offsets.insert((topic.to_string(), partition), offset);
    }

    /// ดึง committed offset (default = 0 ถ้าไม่เคย commit)
    pub fn fetch_offset(&self, topic: &str, partition: usize) -> u64 {
        *self.offsets.get(&(topic.to_string(), partition)).unwrap_or(&0)
    }

    pub fn assigned_partitions(&self, member_id: &str) -> Vec<TopicPartition> {
        self.assignment.get(member_id).cloned().unwrap_or_default()
    }
}
```

**ตัวอย่าง round-robin assignment กับ 3 members และ 6 partitions:**

```
members = ["worker-0", "worker-1", "worker-2"]

partition 0 → worker-0   (0 % 3 = 0)
partition 1 → worker-1   (1 % 3 = 1)
partition 2 → worker-2   (2 % 3 = 2)
partition 3 → worker-0   (3 % 3 = 0)
partition 4 → worker-1   (4 % 3 = 1)
partition 5 → worker-2   (5 % 3 = 2)

ผลลัพธ์:
  worker-0 → [partition-0, partition-3]
  worker-1 → [partition-1, partition-4]
  worker-2 → [partition-2, partition-5]
```

---

### ขั้นที่ 6: Consumer — Poll และ Commit

**`src/consumer.rs`** — ส่วนที่ 2: ConsumerRecord และ Consumer

```rust
/// Record ที่ Consumer ได้รับ
#[derive(Debug, Clone)]
pub struct ConsumerRecord {
    pub topic: String,
    pub partition: usize,
    pub offset: u64,
    pub timestamp: u64,
    pub key: Vec<u8>,
    pub value: Vec<u8>,
    pub headers: HashMap<String, String>,
}

pub struct Consumer {
    pub member_id: String,
    group: ConsumerGroup,
}

impl Consumer {
    pub fn new(member_id: &str, group: ConsumerGroup) -> Self {
        Consumer { member_id: member_id.to_string(), group }
    }

    pub fn group(&self) -> &ConsumerGroup { &self.group }
    pub fn group_mut(&mut self) -> &mut ConsumerGroup { &mut self.group }

    /// Pull messages จาก assigned partitions เริ่มจาก committed offset
    pub fn poll(
        &mut self,
        topic: &Topic,
        max_records: usize,
        _timeout: Duration,
    ) -> io::Result<Vec<ConsumerRecord>> {
        let mut records = Vec::new();
        let assigned = self.group.assigned_partitions(&self.member_id);

        // ถ้าไม่มี assignment → อ่านจากทุก partition ของ topic นี้
        let relevant: Vec<(String, usize)> = if assigned.is_empty() {
            topic.partitions.iter().enumerate()
                .map(|(i, _)| (topic.name.clone(), i))
                .collect()
        } else {
            assigned.into_iter()
                .filter(|(t, _)| t == &topic.name)
                .collect()
        };

        for (topic_name, partition_id) in relevant {
            if partition_id >= topic.partitions.len() { continue; }
            let committed = self.group.fetch_offset(&topic_name, partition_id);
            let remaining = max_records.saturating_sub(records.len());
            if remaining == 0 { break; }

            let msgs = topic.partitions[partition_id]
                .read_from_offset(committed, remaining)?;

            for msg in msgs {
                records.push(ConsumerRecord {
                    topic: topic_name.clone(),
                    partition: partition_id,
                    offset: msg.offset,
                    timestamp: msg.timestamp,
                    key: msg.key,
                    value: msg.value,
                    headers: msg.headers,
                });
            }
        }

        Ok(records)
    }

    /// Commit offsets หลังประมวลผล records เสร็จ
    /// บันทึก max(offset) + 1 ต่อ partition
    pub fn commit_offsets(&mut self, records: &[ConsumerRecord]) {
        let mut max_offsets: HashMap<(String, usize), u64> = HashMap::new();
        for record in records {
            let key = (record.topic.clone(), record.partition);
            let cur = max_offsets.entry(key).or_insert(0);
            if record.offset + 1 > *cur {
                *cur = record.offset + 1;
            }
        }
        for ((topic, partition), next_offset) in max_offsets {
            self.group.commit_offset(&topic, partition, next_offset);
        }
    }
}
```

**ตัวอย่าง poll-commit cycle:**

```rust
let mut consumer = Consumer::new("processor-A", group);

loop {
    // Step 1: poll messages
    let records = consumer.poll(&topic, 100, Duration::from_millis(500))?;
    if records.is_empty() {
        // ไม่มี messages ใหม่ → รอสักครู่
        std::thread::sleep(Duration::from_millis(100));
        continue;
    }

    // Step 2: process messages
    for record in &records {
        let value = String::from_utf8_lossy(&record.value);
        println!("Processing: topic={} partition={} offset={} value={}",
            record.topic, record.partition, record.offset, value);
    }

    // Step 3: commit หลังจาก process สำเร็จ (at-least-once guarantee)
    consumer.commit_offsets(&records);
}
```

**At-least-once vs Exactly-once:** เราเลือก commit หลัง process
ถ้า process ล้มเหลวก่อน commit → poll ครั้งถัดไปจะได้ records เดิม (at-least-once)
ถ้า commit ก่อน process → ถ้า crash อาจเสีย records (at-most-once)

---

### ขั้นที่ 7: Retention Policy

**`src/retention.rs`**

```rust
use crate::partition::Topic;
use std::time::{Duration, SystemTime, UNIX_EPOCH};

pub struct RetentionConfig {
    pub max_age: Duration,              // อายุสูงสุดของ segment
    pub max_total_size_bytes: u64,      // ขนาดรวมสูงสุดของทั้ง topic
}

impl RetentionConfig {
    pub fn new(max_age: Duration, max_total_size_bytes: u64) -> Self {
        RetentionConfig { max_age, max_total_size_bytes }
    }
}

/// ล้าง segments ที่เก่าเกิน retention policy
/// คืนค่า: จำนวน segments ที่ถูกลบ
pub fn cleanup_topic(topic: &mut Topic, config: &RetentionConfig) -> usize {
    let now_secs = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap_or_default()
        .as_secs();

    let cutoff = now_secs.saturating_sub(config.max_age.as_secs());

    let mut total_deleted = 0;
    for partition in &mut topic.partitions {
        total_deleted += partition.delete_segments_before(cutoff);
    }
    total_deleted
}

/// คำนวณขนาดรวมของ topic (bytes)
pub fn total_topic_size(topic: &Topic) -> u64 {
    topic.partitions.iter()
        .flat_map(|p| p.segments.iter())
        .map(|s| s.size_bytes)
        .sum()
}
```

---

### ขั้นที่ 8: Background Retention Task ด้วย Tokio

ในระบบจริง retention ต้องทำงาน background อัตโนมัติ ไม่ใช่เรียกด้วยตัวเอง
ตัวอย่างนี้แสดง pattern ที่ใช้ `Arc<Mutex<Topic>>` และ `tokio::time::interval`:

```rust
use message_queue::partition::Topic;
use message_queue::retention::{cleanup_topic, RetentionConfig};
use std::sync::{Arc, Mutex};
use std::time::Duration;
use tokio::time;

/// รัน retention cleanup เป็น background task
async fn retention_task(
    topic: Arc<Mutex<Topic>>,
    config: RetentionConfig,
    interval: Duration,
) {
    let mut ticker = time::interval(interval);
    loop {
        ticker.tick().await;
        let deleted = {
            let mut t = topic.lock().unwrap();
            cleanup_topic(&mut t, &config)
        };
        if deleted > 0 {
            println!("[retention] ลบ {} segments", deleted);
        }
    }
}

#[tokio::main]
async fn main() {
    let dir = std::env::temp_dir().join("mq-async-demo");
    std::fs::create_dir_all(&dir).unwrap();

    let topic = Topic::new(&dir, "events", 3, 1024 * 1024).unwrap();
    let shared_topic = Arc::new(Mutex::new(topic));

    // สร้าง retention config: ลบ segments เก่ากว่า 7 วัน, ทุก 1 ชั่วโมง
    let config = RetentionConfig::new(
        Duration::from_secs(86400 * 7),
        10 * 1024 * 1024 * 1024, // 10 GB
    );

    let topic_clone = Arc::clone(&shared_topic);
    tokio::spawn(async move {
        retention_task(topic_clone, config, Duration::from_secs(3600)).await;
    });

    // ... producer/consumer logic ...
    println!("Background retention task กำลังทำงาน");
    time::sleep(Duration::from_millis(100)).await;
}
```

**Pitfall:** `Arc<Mutex<Topic>>` จะทำให้ producer/consumer ต้อง lock mutex ทุกครั้ง
ในระบบ high-throughput ใช้ `tokio::sync::RwLock` แทน (อนุญาต concurrent reads)
หรือแยก per-partition lock เพื่อลด contention

---

## การทดสอบ (Testing)

**`src/lib.rs`** รวม 17 unit tests ที่ครอบคลุม:

```rust
#[cfg(test)]
mod tests {
    use crate::consumer::{Consumer, ConsumerGroup};
    use crate::partition::Topic;
    use crate::producer::Producer;
    use crate::retention::{cleanup_topic, total_topic_size, RetentionConfig};
    use crate::segment::Segment;
    use std::collections::HashMap;
    use std::time::Duration;
    use tempfile::tempdir;

    // Test 1: เขียนและอ่าน message เดียว
    #[test]
    fn test_segment_append_and_read_single() {
        let dir = tempdir().unwrap();
        let mut seg = Segment::new(dir.path(), 0, 512).unwrap();

        let offset = seg
            .append(b"key1".to_vec(), b"value1".to_vec(), HashMap::new())
            .unwrap();

        assert_eq!(offset, 0);
        let messages = seg.read_from_offset(0, 10).unwrap();
        assert_eq!(messages.len(), 1);
        assert_eq!(messages[0].key, b"key1");
        assert_eq!(messages[0].value, b"value1");
        assert_eq!(messages[0].offset, 0);
    }

    // Test 2: เขียนหลาย messages และตรวจ offset
    #[test]
    fn test_segment_append_multiple_and_offsets() {
        let dir = tempdir().unwrap();
        let mut seg = Segment::new(dir.path(), 0, 512).unwrap();

        for i in 0u64..5 {
            let offset = seg
                .append(
                    format!("key{}", i).into_bytes(),
                    format!("value{}", i).into_bytes(),
                    HashMap::new(),
                )
                .unwrap();
            assert_eq!(offset, i);
        }

        let messages = seg.read_from_offset(0, 10).unwrap();
        assert_eq!(messages.len(), 5);
        assert_eq!(messages[2].offset, 2);
        assert_eq!(messages[4].key, b"key4");
    }

    // Test 3: อ่านจาก offset กลาง
    #[test]
    fn test_segment_read_from_middle_offset() {
        let dir = tempdir().unwrap();
        let mut seg = Segment::new(dir.path(), 0, 512).unwrap();

        for i in 0u64..10 {
            seg.append(b"k".to_vec(), format!("v{}", i).into_bytes(), HashMap::new())
                .unwrap();
        }

        let messages = seg.read_from_offset(5, 10).unwrap();
        assert_eq!(messages.len(), 5);
        assert_eq!(messages[0].offset, 5);
        assert_eq!(messages[4].offset, 9);
    }

    // Test 4: base_offset ที่ไม่ใช่ 0
    #[test]
    fn test_segment_nonzero_base_offset() {
        let dir = tempdir().unwrap();
        let mut seg = Segment::new(dir.path(), 100, 512).unwrap();

        assert_eq!(seg.next_offset, 100);
        seg.append(b"k".to_vec(), b"v".to_vec(), HashMap::new()).unwrap();
        assert_eq!(seg.next_offset, 101);
    }

    // Test 5: Message มี timestamp ที่ถูกต้อง
    #[test]
    fn test_message_has_timestamp() {
        let dir = tempdir().unwrap();
        let mut seg = Segment::new(dir.path(), 0, 512).unwrap();

        let before = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH).unwrap()
            .as_millis() as u64;

        seg.append(b"k".to_vec(), b"v".to_vec(), HashMap::new()).unwrap();

        let after = std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH).unwrap()
            .as_millis() as u64;

        let msgs = seg.read_from_offset(0, 1).unwrap();
        assert!(msgs[0].timestamp >= before && msgs[0].timestamp <= after);
    }

    // Test 6: Message มี custom headers
    #[test]
    fn test_message_with_headers() {
        let dir = tempdir().unwrap();
        let mut seg = Segment::new(dir.path(), 0, 512).unwrap();

        let mut headers = HashMap::new();
        headers.insert("content-type".to_string(), "application/json".to_string());
        headers.insert("trace-id".to_string(), "abc-123".to_string());

        seg.append(b"key".to_vec(), b"value".to_vec(), headers).unwrap();

        let msgs = seg.read_from_offset(0, 1).unwrap();
        assert_eq!(msgs[0].headers.get("content-type").unwrap(), "application/json");
        assert_eq!(msgs[0].headers.get("trace-id").unwrap(), "abc-123");
    }

    // Test 7: Producer เลือก partition แบบ deterministic
    #[test]
    fn test_producer_partition_selection_deterministic() {
        let p1 = Producer::compute_partition(b"order-123", 4);
        let p2 = Producer::compute_partition(b"order-123", 4);
        assert_eq!(p1, p2, "same key → same partition every time");
        assert!(p1 < 4);
    }

    // Test 8: Producer กระจาย keys ครบทุก partition
    #[test]
    fn test_producer_partition_distribution_covers_all() {
        let partitions: Vec<usize> = (0..100)
            .map(|i| Producer::compute_partition(format!("key-{}", i).as_bytes(), 4))
            .collect();

        assert!(partitions.iter().all(|&p| p < 4));
        let unique: std::collections::HashSet<usize> = partitions.into_iter().collect();
        assert_eq!(unique.len(), 4);
    }

    // Test 9: Producer send คืน RecordMetadata ที่ถูกต้อง
    #[test]
    fn test_producer_send_returns_correct_metadata() {
        let dir = tempdir().unwrap();
        let mut topic = Topic::new(dir.path(), "test-topic", 3, 1024 * 1024).unwrap();
        let producer = Producer::new();

        let meta = producer.send(&mut topic, b"mykey", b"myvalue").unwrap();
        assert_eq!(meta.topic, "test-topic");
        assert!(meta.partition < 3);
        assert_eq!(meta.offset, 0);
    }

    // Test 10: Offset เพิ่มขึ้นทีละ 1 ต่อ partition
    #[test]
    fn test_producer_offset_increments_per_partition() {
        let dir = tempdir().unwrap();
        let mut topic = Topic::new(dir.path(), "orders", 1, 1024 * 1024).unwrap();
        let producer = Producer::new();

        let m1 = producer.send(&mut topic, b"key1", b"v1").unwrap();
        let m2 = producer.send(&mut topic, b"key1", b"v2").unwrap();
        let m3 = producer.send(&mut topic, b"key1", b"v3").unwrap();

        assert_eq!(m1.partition, m2.partition);
        assert_eq!(m1.offset, 0);
        assert_eq!(m2.offset, 1);
        assert_eq!(m3.offset, 2);
    }

    // Test 11: Consumer อ่านจาก committed offset
    #[test]
    fn test_consumer_reads_from_committed_offset() {
        let dir = tempdir().unwrap();
        let mut topic = Topic::new(dir.path(), "events", 1, 1024 * 1024).unwrap();
        let producer = Producer::new();

        for i in 0..5 {
            producer.send(&mut topic, b"key", format!("msg-{}", i).as_bytes()).unwrap();
        }

        let mut group = ConsumerGroup::new("test-group");
        group.join("consumer-1");
        group.assign_partitions("events", 1);
        group.commit_offset("events", 0, 3); // อ่านจาก offset 3

        let mut consumer = Consumer::new("consumer-1", group);
        let records = consumer.poll(&topic, 10, Duration::from_millis(100)).unwrap();

        assert_eq!(records.len(), 2); // offset 3 และ 4
        assert_eq!(records[0].offset, 3);
        assert_eq!(records[1].offset, 4);
    }

    // Test 12: commit_offsets แล้ว poll ครั้งถัดไปไม่อ่านซ้ำ
    #[test]
    fn test_consumer_commit_and_resume() {
        let dir = tempdir().unwrap();
        let mut topic = Topic::new(dir.path(), "logs", 1, 1024 * 1024).unwrap();
        let producer = Producer::new();

        for i in 0..6 {
            producer.send(&mut topic, b"key", format!("log-{}", i).as_bytes()).unwrap();
        }

        let mut group = ConsumerGroup::new("log-group");
        group.join("worker-1");
        group.assign_partitions("logs", 1);
        let mut consumer = Consumer::new("worker-1", group);

        // Poll แรก: อ่าน 3 records
        let records = consumer.poll(&topic, 3, Duration::from_millis(100)).unwrap();
        assert_eq!(records.len(), 3);
        assert_eq!(records[0].offset, 0);
        consumer.commit_offsets(&records);

        // Poll สอง: ต้องเริ่มจาก offset 3
        let records2 = consumer.poll(&topic, 10, Duration::from_millis(100)).unwrap();
        assert_eq!(records2.len(), 3);
        assert_eq!(records2[0].offset, 3);
        assert_eq!(records2[2].offset, 5);
    }

    // Test 13: ConsumerGroup commit/fetch offset
    #[test]
    fn test_consumer_group_offset_commit_fetch() {
        let mut group = ConsumerGroup::new("my-group");

        group.commit_offset("topic-a", 0, 42);
        group.commit_offset("topic-a", 1, 100);
        group.commit_offset("topic-b", 0, 7);

        assert_eq!(group.fetch_offset("topic-a", 0), 42);
        assert_eq!(group.fetch_offset("topic-a", 1), 100);
        assert_eq!(group.fetch_offset("topic-b", 0), 7);
        assert_eq!(group.fetch_offset("topic-a", 99), 0); // ไม่มี → 0
    }

    // Test 14: Round-robin partition assignment
    #[test]
    fn test_consumer_group_round_robin_assignment() {
        let mut group = ConsumerGroup::new("workers");
        group.join("worker-0");
        group.join("worker-1");
        group.join("worker-2");
        group.assign_partitions("orders", 6);

        let a0 = group.assigned_partitions("worker-0");
        let a1 = group.assigned_partitions("worker-1");
        let a2 = group.assigned_partitions("worker-2");

        assert_eq!(a0.len(), 2);
        assert_eq!(a1.len(), 2);
        assert_eq!(a2.len(), 2);

        let total = a0.len() + a1.len() + a2.len();
        assert_eq!(total, 6);

        // ไม่มี partition ซ้ำกัน
        let all: Vec<usize> = a0.iter().chain(a1.iter()).chain(a2.iter())
            .map(|(_, p)| *p).collect();
        let unique: std::collections::HashSet<usize> = all.into_iter().collect();
        assert_eq!(unique.len(), 6);
    }

    // Test 15: Segment rolling เมื่อขนาดเกิน limit
    #[test]
    fn test_partition_segment_rolling() {
        let dir = tempdir().unwrap();
        let mut topic = Topic::new(dir.path(), "events", 1, 150).unwrap();
        let producer = Producer::new();

        for i in 0..15 {
            producer.send(&mut topic, b"key",
                format!("value-payload-number-{:04}", i).as_bytes()).unwrap();
        }

        let seg_count = topic.partitions[0].segment_count();
        assert!(seg_count > 1, "expected multiple segments, got {}", seg_count);
    }

    // Test 16: Retention cleanup ลบ segments เก่า
    #[test]
    fn test_retention_cleanup_deletes_old_segments() {
        let dir = tempdir().unwrap();
        let mut topic = Topic::new(dir.path(), "old-topic", 1, 60).unwrap();
        let producer = Producer::new();

        for i in 0..20 {
            producer.send(&mut topic, b"k", format!("data-{}", i).as_bytes()).unwrap();
        }

        let seg_count_before = topic.partitions[0].segment_count();
        assert!(seg_count_before > 1);

        // ปรับ created_at เป็น 0 (เก่ามาก) ให้ทุก segment ยกเว้น segment สุดท้าย
        {
            let segs = &mut topic.partitions[0].segments;
            let last = segs.len() - 1;
            for i in 0..last {
                segs[i].created_at = 0;
            }
        }

        let config = RetentionConfig::new(Duration::from_secs(1), u64::MAX);
        let deleted = cleanup_topic(&mut topic, &config);

        assert!(deleted > 0, "should have deleted at least one segment");
        assert_eq!(topic.partitions[0].segment_count(), 1);
    }

    // Test 17: total_topic_size คำนวณถูกต้อง
    #[test]
    fn test_total_topic_size() {
        let dir = tempdir().unwrap();
        let mut topic = Topic::new(dir.path(), "sized", 2, 1024 * 1024).unwrap();
        let producer = Producer::new();

        assert_eq!(total_topic_size(&topic), 0);

        producer.send(&mut topic, b"key1", b"some value here").unwrap();
        producer.send(&mut topic, b"key2", b"another value").unwrap();

        assert!(total_topic_size(&topic) > 0);
    }
}
```

### ผลลัพธ์ `cargo test` จริง

```
running 17 tests
test tests::test_consumer_group_offset_commit_fetch ... ok
test tests::test_consumer_group_round_robin_assignment ... ok
test tests::test_consumer_commit_and_resume ... ok
test tests::test_consumer_reads_from_committed_offset ... ok
test tests::test_message_with_headers ... ok
test tests::test_message_has_timestamp ... ok
test tests::test_producer_partition_distribution_covers_all ... ok
test tests::test_producer_partition_selection_deterministic ... ok
test tests::test_producer_offset_increments_per_partition ... ok
test tests::test_producer_send_returns_correct_metadata ... ok
test tests::test_segment_append_and_read_single ... ok
test tests::test_segment_nonzero_base_offset ... ok
test tests::test_partition_segment_rolling ... ok
test tests::test_total_topic_size ... ok
test tests::test_segment_append_multiple_and_offsets ... ok
test tests::test_segment_read_from_middle_offset ... ok
test tests::test_retention_cleanup_deletes_old_segments ... ok

test result: ok. 17 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### ผลลัพธ์ `cargo run` จริง (Demo Binary)

```
=== Persistent Message Queue Demo ===

--- Producer: ส่ง messages ---
  key=user-1001 → partition=2, offset=0
  key=user-1002 → partition=1, offset=0
  key=user-1003 → partition=0, offset=0
  key=user-1001 → partition=2, offset=1
  key=user-1002 → partition=1, offset=1
  key=user-1001 → partition=2, offset=2

--- Consumer Group: อ่าน messages ---
  processor-A assigned: [("orders", 0), ("orders", 2)]
  processor-B assigned: [("orders", 1)]

  processor-A ได้รับ 4 records:
    partition=0, offset=0, value={"order_id": 1002}
    partition=2, offset=0, value={"order_id": 1000}
    partition=2, offset=1, value={"order_id": 1003}
    partition=2, offset=2, value={"order_id": 1005}

  committed offsets สำเร็จ

--- Retention: ล้าง segments เก่า ---
  ลบ 0 segments (ทุก segment ยังใหม่อยู่)

Demo เสร็จสิ้น!
```

**สังเกตจาก demo output:**
- `user-1001` ไปที่ partition 2 ทุกครั้ง (offset 0, 1, 2) — ordering guarantee ภายใน partition
- `user-1002` ไปที่ partition 1 ทุกครั้ง (offset 0, 1)
- `processor-A` ได้รับ partitions 0 และ 2 (round-robin 2 members, 3 partitions → A ได้ 2 partitions)
- messages ของ `user-1001` ถูกอ่านตามลำดับ offset 0 → 1 → 2 เสมอ

---

## ข้อผิดพลาดที่พบบ่อย (Common Pitfalls)

### Pitfall 1: ใช้ `.write(true)` แทน `.append(true)`

**ผิด:**
```rust
// นี่จะ overwrite data จาก position 0!
let mut file = OpenOptions::new()
    .create(true)
    .write(true)  // ← ผิด!
    .open(&self.log_path)?;
```

**ถูก:**
```rust
let mut file = OpenOptions::new()
    .create(true)
    .append(true)  // ← ถูก: เขียนต่อท้ายเสมอ
    .open(&self.log_path)?;
```

**อธิบาย:** `.write(true)` เปิด file ที่ position 0 ทำให้การเขียนครั้งถัดไป overwrite data เก่า
`.append(true)` บังคับให้ kernel seek ไปที่ end of file ก่อนทุก write operation
ซึ่ง **atomic** บน Linux/macOS ต่างจาก seek+write ที่ต้องใช้ lock ป้องกัน

---

### Pitfall 2: Committed Offset Off-by-One

**ผิด:**
```rust
// commit offset ที่อ่านล่าสุด (ทำให้อ่านซ้ำ!)
fn commit_offsets(&mut self, records: &[ConsumerRecord]) {
    for record in records {
        self.group.commit_offset(&record.topic, record.partition, record.offset); // ← ผิด!
    }
}
```

**ถูก:**
```rust
// commit = max(offset) + 1 (offset ถัดไปที่จะอ่าน)
fn commit_offsets(&mut self, records: &[ConsumerRecord]) {
    let mut max_offsets: HashMap<(String, usize), u64> = HashMap::new();
    for record in records {
        let key = (record.topic.clone(), record.partition);
        let cur = max_offsets.entry(key).or_insert(0);
        if record.offset + 1 > *cur {
            *cur = record.offset + 1; // ← ถูก: next offset = last_read + 1
        }
    }
    for ((topic, partition), next_offset) in max_offsets {
        self.group.commit_offset(&topic, partition, next_offset);
    }
}
```

**อธิบาย:** Kafka convention กำหนดว่า committed offset = "offset ถัดไปที่จะอ่าน"
ถ้า commit ค่า offset ที่อ่านมา (ไม่บวก 1) poll ครั้งถัดไปจะอ่าน record เดิมซ้ำทุกครั้ง

---

### Pitfall 3: ลบ Active Segment

**ผิด:**
```rust
// ลบทุก segment โดยไม่เก็บ segment สุดท้ายไว้
fn delete_segments_before(&mut self, before_timestamp: u64) -> usize {
    let to_delete: Vec<usize> = self.segments
        .iter()
        .enumerate()
        .filter(|(_, s)| s.created_at < before_timestamp)
        .map(|(i, _)| i)
        .collect();
    // ← ผิด: อาจลบ active segment ที่กำลัง write อยู่!
```

**ถูก:**
```rust
fn delete_segments_before(&mut self, before_timestamp: u64) -> usize {
    if self.segments.len() <= 1 { return 0; } // ← ป้องกัน
    let last_idx = self.segments.len() - 1;
    let to_delete: Vec<usize> = self.segments
        .iter()
        .enumerate()
        .filter(|(i, s)| *i < last_idx && s.created_at < before_timestamp) // ← ไม่แตะ segment สุดท้าย
        .map(|(i, _)| i)
        .collect();
    // ...
}
```

**อธิบาย:** Active segment (segment ล่าสุด) ยังถูก write อยู่ ถ้าถูกลบทิ้งจะทำให้ append ครั้งต่อไป panic
หรือสร้าง gap ใน offset sequence กฎ: **เก็บ segment สุดท้ายเสมอ**

---

### Pitfall 4: Race Condition ใน Concurrent Producer

**ผิด:**
```rust
// ไม่ thread-safe — หลาย threads แข่งกัน append ในเวลาเดียวกัน
static mut TOPIC: Option<Topic> = None;

async fn handler(req: Request) {
    unsafe {
        if let Some(ref mut topic) = TOPIC {
            producer.send(topic, key, value); // ← ผิด: data race!
        }
    }
}
```

**ถูก:**
```rust
// ใช้ Arc<Mutex<Topic>> หรือ Arc<RwLock<Topic>>
let topic = Arc::new(Mutex::new(Topic::new(&dir, "events", 3, 1024 * 1024)?));

async fn handler(req: Request, topic: Arc<Mutex<Topic>>) {
    let mut t = topic.lock().unwrap(); // ← acquire lock ก่อน
    producer.send(&mut t, key, value)?;
    // lock released เมื่อ `t` drop
}
```

**อธิบาย:** `Topic` และ `Partition` ไม่ใช่ `Send + Sync` โดย default
ต้องใช้ `Arc<Mutex<T>>` เพื่อแชร์ข้ามหลาย threads
ใน high-throughput ควรใช้ `RwLock` แทนเพราะ consumer อ่านพร้อมกันได้

---

### Pitfall 5: ไม่ `flush()` หลัง `write_all()`

**ผิด:**
```rust
let mut file = OpenOptions::new().create(true).append(true).open(&path)?;
file.write_all(&len.to_le_bytes())?;
file.write_all(&json)?;
// ← ไม่ flush! ข้อมูลอาจค้างอยู่ใน BufWriter buffer
```

**ถูก:**
```rust
file.write_all(&len.to_le_bytes())?;
file.write_all(&json)?;
file.flush()?; // ← บังคับให้ flush ไปที่ OS buffer
// ถ้าต้องการ durability สูงสุด ใช้ file.sync_all()?
```

**อธิบาย:** Rust's `File` มี internal buffering บางส่วน แต่แม้แต่ OS buffer ก็ยังไม่ทนต่อ
power failure ถ้าต้องการ durability ให้ใช้ `file.sync_all()` (เรียก `fsync(2)`)
แต่ `sync_all()` ช้ากว่า `flush()` มาก — ใน production มักทำ `sync_all()` เป็น batch

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
./target/release/message-queue
```

### ใช้เป็น Library ใน Workspace

```toml
# ใน Cargo.toml ของ workspace root
[workspace]
members = [
    "message-queue",
    "order-service",   # ← service ที่ใช้ mq
    "analytics-service",
]

# ใน order-service/Cargo.toml
[dependencies]
message-queue = { path = "../message-queue" }
```

### Configuration สำหรับ Production

```rust
pub struct QueueConfig {
    pub data_dir: PathBuf,
    pub segment_size_bytes: u64,      // ขนาด segment (แนะนำ: 1 GB)
    pub retention_hours: u64,         // อายุ segment (แนะนำ: 168 h = 7 วัน)
    pub max_topic_size_bytes: u64,    // ขนาดรวมสูงสุดของ topic
    pub default_partition_count: usize, // partition count ต่อ topic ใหม่
}

impl Default for QueueConfig {
    fn default() -> Self {
        QueueConfig {
            data_dir: PathBuf::from("/var/lib/message-queue"),
            segment_size_bytes: 1024 * 1024 * 1024, // 1 GB
            retention_hours: 168,                    // 7 วัน
            max_topic_size_bytes: 50 * 1024 * 1024 * 1024, // 50 GB
            default_partition_count: 8,
        }
    }
}
```

### Docker Deployment

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN mkdir -p /var/lib/message-queue /etc/message-queue
COPY --from=builder /app/target/release/message-queue /usr/local/bin/

# Volume สำหรับ persistent data
VOLUME ["/var/lib/message-queue"]
EXPOSE 9092

CMD ["message-queue", "--config", "/etc/message-queue/config.toml"]
```

```yaml
# docker-compose.yml
services:
  message-queue:
    build: .
    ports:
      - "9092:9092"
    volumes:
      - mq_data:/var/lib/message-queue
    environment:
      MQ_DATA_DIR: /var/lib/message-queue
      MQ_RETENTION_HOURS: "168"
      MQ_SEGMENT_SIZE_MB: "1024"

volumes:
  mq_data:
    driver: local
```

### Health Check Endpoint (ตัวอย่างด้วย Axum)

```rust
use axum::{Json, Router, routing::get};
use serde::Serialize;

#[derive(Serialize)]
struct HealthResponse {
    status: String,
    topics: Vec<TopicInfo>,
}

#[derive(Serialize)]
struct TopicInfo {
    name: String,
    partition_count: usize,
    total_size_bytes: u64,
}

async fn health(State(topics): State<Arc<TopicRegistry>>) -> Json<HealthResponse> {
    let topics_info = topics.list().iter().map(|t| TopicInfo {
        name: t.name.clone(),
        partition_count: t.partition_count(),
        total_size_bytes: total_topic_size(t),
    }).collect();

    Json(HealthResponse {
        status: "ok".to_string(),
        topics: topics_info,
    })
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Offset Persistence (ระดับ: ⭐⭐)

ปัจจุบัน `ConsumerGroup.offsets` อยู่ใน memory เท่านั้น ถ้า process restart offsets จะหายไป
ให้เพิ่ม persistence layer สำหรับ offsets:

```rust
// บันทึก offsets ลง file ทุกครั้งที่ commit
pub struct PersistentConsumerGroup {
    inner: ConsumerGroup,
    offsets_path: PathBuf,
}

impl PersistentConsumerGroup {
    pub fn new(dir: &Path, group_id: &str) -> io::Result<Self> {
        let offsets_path = dir.join(format!("{}.offsets.json", group_id));
        let mut group = ConsumerGroup::new(group_id);

        // โหลด offsets จาก disk ถ้ามี
        if offsets_path.exists() {
            let data = std::fs::read_to_string(&offsets_path)?;
            let saved: HashMap<(String, usize), u64> = serde_json::from_str(&data)?;
            group.offsets = saved;
        }

        Ok(PersistentConsumerGroup { inner: group, offsets_path })
    }

    pub fn commit_offset(&mut self, topic: &str, partition: usize, offset: u64) -> io::Result<()> {
        self.inner.commit_offset(topic, partition, offset);
        // บันทึกลง disk ทันที
        let json = serde_json::to_string(&self.inner.offsets)?;
        std::fs::write(&self.offsets_path, json)?;
        Ok(())
    }
}
```

**เงื่อนไข:** เขียน test ที่ simulate process restart โดยสร้าง `PersistentConsumerGroup` ใหม่
และตรวจว่า offsets ถูกโหลดมาถูกต้อง

---

### แบบฝึกหัดที่ 2: Network Protocol (ระดับ: ⭐⭐⭐)

เพิ่ม TCP server ด้วย `tokio::net::TcpListener` เพื่อให้ producer/consumer ต่างกระบวนการ
ส่ง commands ผ่าน network ได้ กำหนด simple binary protocol:

```
Request format:
  [1 byte command] [2 bytes topic_len] [topic_len bytes topic] [payload...]

Commands:
  0x01 PRODUCE  → [2 bytes key_len][key_len bytes key][4 bytes value_len][value bytes]
  0x02 CONSUME  → [2 bytes group_len][group bytes][4 bytes partition][8 bytes offset][4 bytes max]
  0x03 COMMIT   → [2 bytes group_len][group bytes][4 bytes partition][8 bytes offset]

Response format:
  [1 byte status] [payload...]

Status:
  0x00 OK
  0x01 ERROR  → [4 bytes msg_len][msg bytes]
```

**เงื่อนไข:** handle concurrent connections ด้วย `tokio::spawn` ต่อ connection
ใช้ `Arc<RwLock<HashMap<String, Topic>>>` สำหรับ topic registry

---

### แบบฝึกหัดที่ 3: Message Compaction (ระดับ: ⭐⭐⭐)

Kafka มี **log compaction** mode — เก็บแค่ message ล่าสุดของแต่ละ key (แทน retention by time)
มีประโยชน์สำหรับ state snapshots เช่น user preferences, product prices

```rust
/// Compact partition: เก็บแค่ message ล่าสุดต่อ key
pub fn compact_partition(partition: &mut Partition) -> io::Result<usize> {
    // Step 1: อ่าน messages ทั้งหมด
    let all_messages = /* อ่านจากทุก segments */;

    // Step 2: เก็บแค่ message ล่าสุดต่อ key
    let mut latest: HashMap<Vec<u8>, Message> = HashMap::new();
    for msg in all_messages {
        latest.insert(msg.key.clone(), msg);
    }

    // Step 3: เขียน compacted messages ลง segment ใหม่
    let compacted: Vec<Message> = latest.into_values().collect();
    // ...

    // Step 4: ลบ segments เก่า
    let deleted = compacted.len(); // แสดงจำนวนที่ลบ
    Ok(deleted)
}
```

**ความท้าทาย:** ต้อง reassign offsets หลัง compact หรือ keep original offsets?
(Kafka เลือก keep offsets เพื่อให้ consumers ไม่ต้อง reset)

---

### แบบฝึกหัดที่ 4: Replication (ระดับ: ⭐⭐⭐⭐)

เพิ่ม replication ด้วย leader-follower pattern:

```rust
pub struct ReplicatedPartition {
    leader: Partition,
    followers: Vec<RemotePartition>, // partitions บน nodes อื่น
    replication_factor: usize,
    acks: AcksPolicy,
}

pub enum AcksPolicy {
    Zero,           // ไม่รอ ack ใด ๆ (fastest, least durable)
    One,            // รอ leader ack (default)
    All,            // รอทุก replica ack (slowest, most durable)
}

impl ReplicatedPartition {
    pub async fn append(&mut self, key: Vec<u8>, value: Vec<u8>) -> io::Result<u64> {
        let offset = self.leader.append(key.clone(), value.clone(), HashMap::new())?;

        match self.acks {
            AcksPolicy::Zero => {}
            AcksPolicy::One => { /* leader เขียนแล้ว */ }
            AcksPolicy::All => {
                // replicate ไปทุก follower แล้วรอ ack
                let futs: Vec<_> = self.followers.iter_mut()
                    .map(|f| f.replicate(offset, &key, &value))
                    .collect();
                futures::future::join_all(futs).await;
            }
        }

        Ok(offset)
    }
}
```

---

### แบบฝึกหัดที่ 5: Metrics และ Monitoring (ระดับ: ⭐⭐)

เพิ่ม metrics สำหรับ monitoring:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

pub struct TopicMetrics {
    pub messages_produced: AtomicU64,
    pub messages_consumed: AtomicU64,
    pub bytes_written: AtomicU64,
    pub bytes_read: AtomicU64,
    pub consumer_lag: HashMap<(String, usize), AtomicU64>, // partition lag
}

impl TopicMetrics {
    pub fn record_produce(&self, bytes: u64) {
        self.messages_produced.fetch_add(1, Ordering::Relaxed);
        self.bytes_written.fetch_add(bytes, Ordering::Relaxed);
    }

    pub fn consumer_lag(&self, group: &ConsumerGroup, partition: usize) -> u64 {
        let topic_name = /* ... */;
        let committed = group.fetch_offset(topic_name, partition);
        let end_offset = /* partition.next_offset() */;
        end_offset.saturating_sub(committed)
    }
}
```

**consumer lag** คือความแตกต่างระหว่าง end offset กับ committed offset
ถ้า lag สูงแสดงว่า consumer ประมวลผลช้ากว่า producer ส่ง → ต้องเพิ่ม consumer instances

---

### แบบฝึกหัดที่ 6: Schema Registry ขนาดเล็ก (ระดับ: ⭐⭐⭐)

เพิ่ม type safety ให้ messages ด้วย schema validation:

```rust
pub struct SchemaRegistry {
    schemas: HashMap<String, Schema>, // topic → schema
}

pub struct Schema {
    pub version: u32,
    pub required_headers: Vec<String>,
    pub value_format: ValueFormat,
}

pub enum ValueFormat {
    Json { schema: serde_json::Value },
    Binary,
    Utf8Text,
}

impl Producer {
    pub fn send_validated(
        &self,
        topic: &mut Topic,
        key: &[u8],
        value: &[u8],
        registry: &SchemaRegistry,
    ) -> Result<RecordMetadata, SchemaError> {
        if let Some(schema) = registry.schemas.get(&topic.name) {
            schema.validate(value)?;
        }
        Ok(self.send(topic, key, value)?)
    }
}
```

**เชื่อมต่อกับ Project C10 (Schema Registry)** ที่สร้าง full schema registry ด้วย version evolution

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **Persistent Message Queue** ที่ inspired by Kafka ประกอบด้วย:

**สิ่งที่สร้าง:**
- `Segment` — append-only log file พร้อม sparse index ใช้ length-prefix framing (`u64::to_le_bytes`)
- `Partition` — จัดการ segments หลายตัว, segment rolling เมื่อขนาดเกิน threshold
- `Topic` — grouping ของ partitions, สร้าง directory layout บน filesystem จริง
- `Producer` — `hash(key) % N` สำหรับ deterministic partition selection
- `ConsumerGroup` — offset management + round-robin partition assignment
- `Consumer` — pull-based poll, commit_offsets, at-least-once semantics
- `RetentionConfig` + `cleanup_topic` — ลบ old segments อัตโนมัติ

**Pattern สำคัญที่ได้เรียน:**
1. **Append-only log** — immutable past, mutable future: ทำให้ concurrent reads ปลอดภัยโดยไม่ต้องใช้ read lock
2. **Length-prefix framing** — `u64::to_le_bytes()` + JSON payload: เรียบง่าย portable และ self-describing
3. **Pull-based consumption** — consumer ควบคุม pace ตัวเอง ป้องกัน producer overwhelm consumer
4. **Offset-as-cursor** — offset เดียวต่อ partition แทน message-level acknowledgment: scalable ที่สุด
5. **Segment-level retention** — delete เป็น O(1) ไม่กระทบ performance ของ active segments
6. **Round-robin assignment** — กระจาย load อย่างสม่ำเสมอโดยไม่ต้องรู้ workload ล่วงหน้า

**ความแตกต่างจากโปรเจคก่อนหน้า:**
- **F09 Distributed Tracing**: เก็บ spans แบบ write-once-read-later
  ส่วน Message Queue เป็น stream ที่ consumers ประมวลผลตามลำดับ
- Message Queue เพิ่ม **consumer group semantics** ที่ F09 ไม่มี — หลาย workers
  ร่วมมือกันประมวลผล topic เดียวโดยไม่ duplicate งาน

**โปรเจคถัดไป — G01 Docker Builder** จะนำ knowledge ด้าน file I/O และ process management
ที่ได้เรียนในโปรเจคนี้ไปใช้ในการ build, tag, และ push Docker images แบบ programmatic
ผ่าน Rust โดยไม่ต้องเรียก `docker` CLI โดยตรง

---

**โปรเจคก่อนหน้า:** [Project F09: Distributed Tracing](project-f09-dist-tracing.md) | **โปรเจคถัดไป:** [Project G01: Docker Builder](project-g01-docker-builder.md)
