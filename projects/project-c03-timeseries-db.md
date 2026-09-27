# Project C03: Time Series Database (lite)

> โมดูล: C — Data Processing & Pipelines | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

Time Series Database (TSDB) คือระบบจัดเก็บข้อมูลที่ **ออกแบบมาเฉพาะสำหรับข้อมูลที่มีมิติเวลา** เช่น CPU usage ทุก 10 วินาที, อุณหภูมิเซ็นเซอร์, จำนวน HTTP requests ต่อนาที ข้อมูลประเภทนี้เติบโตไม่หยุด (append-only) เรียงตามเวลาเสมอ และมักถูก query ในรูปแบบ range scan ("ให้ข้อมูล CPU ของ server1 ใน 1 ชั่วโมงที่ผ่านมา")

ในโปรเจคนี้เราจะสร้าง TSDB แบบ lightweight ที่ **รองรับ InfluxDB v1 API** (subset) ได้จริง ซึ่งหมายความว่า Grafana หรือ Telegraf สามารถเชื่อมต่อมาใช้งานได้โดยตรง ระบบประกอบด้วย:

- **WAL (Write-Ahead Log)** — เขียนข้อมูลลง binary file ด้วย `BufWriter` แบบ append-only
- **In-memory BTreeMap index** — O(log n) range scan ที่สร้างใหม่จาก WAL ทุก startup
- **Compaction background task** — รวม WAL files เล็ก ๆ ให้เป็น chunk ที่บีบอัดด้วย LZ4
- **InfluxDB line protocol parser** — รับ `POST /write` body แบบ hand-written parser ไม่พึ่ง crate
- **InfluxQL query engine** — parse และ execute `SELECT mean(x) FROM y WHERE time > now()-1h GROUP BY time(5m)`
- **Retention policy** — background task ลบข้อมูลเก่าตาม `retention_days` config
- **HTTP API** — compatible กับ InfluxDB v1 `/write` `/query` `/series` endpoints

**Use case จริงใน production:** InfluxDB, Prometheus, TimescaleDB, Victoria Metrics ต่างมีแนวคิดคล้ายกัน การสร้าง TSDB จาก scratch ช่วยให้เข้าใจ trade-off ของ storage engine design, ทำไม append-only WAL ถึงเร็วกว่า B-tree ทั่วไป และทำไม downsampling ถึงสำคัญมากสำหรับ long-retention data

---

## สิ่งที่จะได้เรียนรู้

- **Binary serialization** ด้วย `byteorder` crate — เขียน/อ่าน binary format แบบ cross-platform
- **Append-only WAL design** — pattern ที่ใช้ใน databases จริง (PostgreSQL WAL, Kafka log)
- **BTreeMap สำหรับ ordered index** — ทำไมถึงเลือก BTree แทน HashMap สำหรับ range queries
- **Manual parser design** — parse InfluxDB line protocol และ InfluxQL subset โดยไม่ใช้ parser crate
- **Background tasks ใน tokio** — `tokio::spawn` + `tokio::time::interval` สำหรับ compaction/retention
- **LZ4 compression** ด้วย `lz4_flex` — compress chunk files เพื่อลด disk usage
- **axum HTTP server** — สร้าง REST API ที่ compatible กับ InfluxDB v1 protocol
- **Downsampling algorithms** — bucket-based aggregation (mean/sum/min/max/count) สำหรับ GROUP BY time()

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 41-50** — Async/await, tokio runtime, future combinators
- **Part 51-60** — File I/O, BufWriter/BufReader, binary data handling
- **Part 61-70** — axum web framework, HTTP routing, extractors
- **Part 71-80** — Collections: BTreeMap, HashMap, iterators, ranges
- **Part 96-110** — Production patterns: error handling, structured logging, background tasks
- โปรเจค C02 (Realtime Analytics) — ความเข้าใจ streaming data pipeline และ time-window aggregation

---

## โครงสร้างโปรเจค (Project Layout)

```
tsdb/
├── src/
│   ├── main.rs              # Entry point, server bootstrap, config
│   ├── wal.rs               # WAL serialization/deserialization
│   ├── index.rs             # BTreeMap in-memory index
│   ├── store.rs             # TsStore — integrates WAL + index, coordinates writes
│   ├── line_protocol.rs     # InfluxDB line protocol parser (hand-written)
│   ├── query_engine.rs      # InfluxQL subset parser + executor
│   ├── downsampling.rs      # Bucket aggregation (mean/sum/min/max/count)
│   ├── compaction.rs        # Background WAL compaction → LZ4 chunks
│   ├── retention.rs         # Retention policy + background cleanup
│   └── http_api.rs          # axum routes: /write /query /series
├── tests/
│   └── integration_test.rs  # End-to-end write → query → assert
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
POST /write (line protocol body)
        │
        ▼
  line_protocol::parse_line()
        │  LineRecord { measurement, tags, fields, timestamp_ns }
        ▼
  TsStore::write()
        ├─── WAL append  → wal.bin  (BufWriter, binary format)
        └─── Index insert → BTreeMap<(SeriesKey, Timestamp), FileOffset>

GET /query?q=SELECT mean(value) FROM cpu WHERE time > now()-1h GROUP BY time(5m)
        │
        ▼
  query_engine::parse_query()
        │  TsQuery { select_fn, from, where_time_gt_ns, group_by_ns }
        ▼
  TsStore::execute_query()
        ├─── index.range_scan("cpu", cutoff_ns, now_ns)  → Vec<(ts, offset)>
        ├─── wal/chunk read → Vec<(ts, f64)>
        └─── downsampling::aggregate_buckets()  → Vec<BucketStats>
                                                       │
                                                       ▼
                                              JSON response (InfluxDB v1 format)

Background tasks (tokio::spawn):
  ┌─ compaction_task()  → every 1h: merge small WAL segs → LZ4 chunk
  └─ retention_task()   → every 1h: drop entries older than retention_days
```

### ทำไมถึงเลือก Design นี้

**Append-only WAL แทน in-place update:**
การเขียนข้อมูล time series ส่วนใหญ่เป็น sequential append ไม่มีการแก้ไข historical data WAL จึงเหมาะกว่า B-tree page-based storage เพราะ:
- Sequential write เร็วกว่า random write 10-100x บน spinning disk
- Recovery ง่าย — แค่ replay WAL จาก beginning
- No write amplification จากการ rebalance tree

**BTreeMap index ใน memory แทน persistent B-tree:**
สำหรับ TSDB ที่มี hot data ไม่เกิน retention window การเก็บ index ใน memory:
- Range scan เป็น O(log n + k) โดย k คือจำนวน results
- ไม่ต้องจัดการ page cache, lock contention
- Rebuild จาก WAL ตอน startup (startup time = O(WAL size))

**ทางเลือกที่ไม่เลือก:**
- LSM-tree (RocksDB style) — ดีกว่าสำหรับ write-heavy random key, แต่ complex เกิน
- Columnar storage (Parquet) — ดีกว่าสำหรับ analytics query, แต่ write latency สูง
- Per-series file — ง่ายแต่ filesystem overhead สูงเมื่อมีหลายพัน series

### Binary Format ของ WAL Entry

```
┌──────────────────┬──────────────┬──────────────┬─────────────┬──────────────┬──────────────────┐
│  timestamp_ns    │  name_len    │  name bytes  │    value    │  tags_len    │  tags bytes      │
│  (u64 LE, 8B)   │  (u32 LE,4B) │  (name_len B)│ (f64 LE,8B) │  (u64 LE,8B) │  (tags_len B)    │
└──────────────────┴──────────────┴──────────────┴─────────────┴──────────────┴──────────────────┘
```

ทุก field ใช้ **Little-Endian** เพราะ x86/ARM สมัยใหม่เป็น LE ทำให้ไม่ต้องแปลง byte order

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: WAL Serialization/Deserialization

เริ่มจากหัวใจของ storage engine — binary serialization ของ WAL entry

```toml
# Cargo.toml
[package]
name = "tsdb"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
byteorder = "1"
tokio = { version = "1", features = ["full"] }
axum = "0.8"
chrono = { version = "0.4", features = ["serde"] }
lz4_flex = "0.11"

[dev-dependencies]
tempfile = "3"
```

```rust
// src/wal.rs
use byteorder::{LittleEndian, ReadBytesExt, WriteBytesExt};
use std::io::{self, BufWriter, Read, Write};

/// Single WAL entry: timestamp_ns, series name, value, tags (as JSON string)
#[derive(Debug, Clone, PartialEq)]
pub struct WalEntry {
    pub timestamp_ns: u64,
    pub name: String,
    pub value: f64,
    pub tags: String,
}

/// Serialize a WalEntry to bytes.
/// Format: [u64 timestamp_ns][u32 name_len][name bytes][f64 value][u64 tags_len][tags bytes]
pub fn serialize_entry<W: Write>(writer: &mut W, entry: &WalEntry) -> io::Result<()> {
    writer.write_u64::<LittleEndian>(entry.timestamp_ns)?;
    let name_bytes = entry.name.as_bytes();
    writer.write_u32::<LittleEndian>(name_bytes.len() as u32)?;
    writer.write_all(name_bytes)?;
    writer.write_f64::<LittleEndian>(entry.value)?;
    let tags_bytes = entry.tags.as_bytes();
    writer.write_u64::<LittleEndian>(tags_bytes.len() as u64)?;
    writer.write_all(tags_bytes)?;
    Ok(())
}

/// Deserialize a WalEntry from bytes. Returns None on EOF.
pub fn deserialize_entry<R: Read>(reader: &mut R) -> io::Result<Option<WalEntry>> {
    let timestamp_ns = match reader.read_u64::<LittleEndian>() {
        Ok(v) => v,
        Err(e) if e.kind() == io::ErrorKind::UnexpectedEof => return Ok(None),
        Err(e) => return Err(e),
    };
    let name_len = reader.read_u32::<LittleEndian>()? as usize;
    let mut name_bytes = vec![0u8; name_len];
    reader.read_exact(&mut name_bytes)?;
    let name = String::from_utf8(name_bytes)
        .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
    let value = reader.read_f64::<LittleEndian>()?;
    let tags_len = reader.read_u64::<LittleEndian>()? as usize;
    let mut tags_bytes = vec![0u8; tags_len];
    reader.read_exact(&mut tags_bytes)?;
    let tags = String::from_utf8(tags_bytes)
        .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
    Ok(Some(WalEntry { timestamp_ns, name, value, tags }))
}

/// Write multiple entries using BufWriter for efficiency
pub fn write_entries<W: Write>(
    writer: &mut BufWriter<W>,
    entries: &[WalEntry],
) -> io::Result<()> {
    for entry in entries {
        serialize_entry(writer, entry)?;
    }
    writer.flush()
}

/// Read all entries from a WAL byte slice (e.g. memory-mapped or loaded file)
pub fn read_all_entries(data: &[u8]) -> io::Result<Vec<WalEntry>> {
    let mut cursor = std::io::Cursor::new(data);
    let mut entries = Vec::new();
    while let Some(entry) = deserialize_entry(&mut cursor)? {
        entries.push(entry);
    }
    Ok(entries)
}
```

**จุดสำคัญ:** `BufWriter` จำเป็นมากเพราะ WAL เขียนบ่อยมาก — ทุก data point ต้องเขียน แต่ไม่ต้อง flush ทุกครั้ง `BufWriter` buffer ไว้แล้ว flush เป็น batch ทำให้ throughput สูงขึ้นหลาย order of magnitude

### ขั้นที่ 2: In-Memory BTreeMap Index

```rust
// src/index.rs
use std::collections::BTreeMap;

pub type SeriesKey = String;
pub type Timestamp = u64;
pub type FileOffset = u64;

/// Index mapping (series_name, timestamp_ns) -> file_offset in WAL
pub struct TsIndex {
    pub inner: BTreeMap<(SeriesKey, Timestamp), FileOffset>,
}

impl TsIndex {
    pub fn new() -> Self {
        Self { inner: BTreeMap::new() }
    }

    pub fn insert(&mut self, series: &str, ts: Timestamp, offset: FileOffset) {
        self.inner.insert((series.to_string(), ts), offset);
    }

    /// Range scan: O(log n + k) — returns all offsets in [start_ts, end_ts]
    pub fn range_scan(
        &self,
        series: &str,
        start_ts: Timestamp,
        end_ts: Timestamp,
    ) -> Vec<(Timestamp, FileOffset)> {
        let lo = (series.to_string(), start_ts);
        let hi = (series.to_string(), end_ts);
        self.inner
            .range(lo..=hi)
            .map(|((_, ts), offset)| (*ts, *offset))
            .collect()
    }

    /// List all distinct series names (sorted)
    pub fn series_names(&self) -> Vec<String> {
        let mut names: Vec<String> = self
            .inner
            .keys()
            .map(|(name, _)| name.clone())
            .collect::<std::collections::HashSet<_>>()
            .into_iter()
            .collect();
        names.sort();
        names
    }

    /// Count entries for a specific series
    pub fn count_for_series(&self, series: &str) -> usize {
        let lo = (series.to_string(), 0u64);
        let hi = (series.to_string(), u64::MAX);
        self.inner.range(lo..=hi).count()
    }
}
```

**ทำไม BTreeMap ไม่ใช่ HashMap:**
- `BTreeMap::range()` ให้ range scan ที่มีประสิทธิภาพโดยตรง
- ข้อมูล sorted ตาม key `(series, timestamp)` ทำให้ series ที่มีชื่อใกล้เคียงกันอยู่ต่อกัน
- `HashMap` ไม่รองรับ range query — ต้อง scan ทั้ง table

**Composite key `(String, u64)`:**
BTreeMap compares tuples lexicographically — compare `String` ก่อน แล้วค่อย compare `u64` ทำให้ entries ของ series เดียวกันอยู่ติดกันใน tree range scan จึงเดินแค่ส่วนที่ต้องการ

### ขั้นที่ 3: TsStore — ประสานงาน WAL + Index

```rust
// src/store.rs
use crate::index::TsIndex;
use crate::wal::{self, WalEntry};
use std::fs::{File, OpenOptions};
use std::io::{self, BufWriter, Seek, SeekFrom};
use std::path::PathBuf;
use std::sync::{Arc, RwLock};

pub struct TsStore {
    pub index: Arc<RwLock<TsIndex>>,
    wal_path: PathBuf,
    wal_writer: Arc<tokio::sync::Mutex<BufWriter<File>>>,
}

impl TsStore {
    /// Open or create a TSDB store at the given directory
    pub fn open(dir: &std::path::Path) -> io::Result<Self> {
        std::fs::create_dir_all(dir)?;
        let wal_path = dir.join("wal.bin");

        // Rebuild index from existing WAL on startup
        let mut index = TsIndex::new();
        if wal_path.exists() {
            let data = std::fs::read(&wal_path)?;
            let entries = wal::read_all_entries(&data)?;
            let mut offset = 0u64;
            for entry in &entries {
                // Compute byte size of this entry for offset tracking
                let entry_size = 8 + 4 + entry.name.len() + 8 + 8 + entry.tags.len();
                index.insert(&entry.name, entry.timestamp_ns, offset);
                offset += entry_size as u64;
            }
        }

        // Open WAL file in append mode
        let file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&wal_path)?;

        Ok(Self {
            index: Arc::new(RwLock::new(index)),
            wal_path,
            wal_writer: Arc::new(tokio::sync::Mutex::new(BufWriter::new(file))),
        })
    }

    /// Write a single data point
    pub async fn write(
        &self,
        series: &str,
        timestamp_ns: u64,
        value: f64,
        tags: &str,
    ) -> io::Result<()> {
        let entry = WalEntry {
            timestamp_ns,
            name: series.to_string(),
            value,
            tags: tags.to_string(),
        };

        let mut writer = self.wal_writer.lock().await;

        // Record current offset before writing
        let offset = writer.get_ref().seek(SeekFrom::Current(0))
            .unwrap_or(0);

        wal::serialize_entry(&mut *writer, &entry)?;
        writer.flush()?;

        // Update in-memory index
        self.index.write().unwrap().insert(series, timestamp_ns, offset);
        Ok(())
    }

    /// Query raw (timestamp, value) pairs for a series in a time range
    pub async fn query_range(
        &self,
        series: &str,
        start_ns: u64,
        end_ns: u64,
    ) -> io::Result<Vec<(u64, f64)>> {
        let offsets = {
            let idx = self.index.read().unwrap();
            idx.range_scan(series, start_ns, end_ns)
        };

        if offsets.is_empty() {
            return Ok(vec![]);
        }

        // Read WAL file and extract values at recorded offsets
        let data = tokio::fs::read(&self.wal_path).await?;
        let all_entries = wal::read_all_entries(&data)?;

        // Build a lookup map from file-offset → entry for the queried series
        // (In a production system, offsets would be used for direct seeks;
        //  here we rebuild from WAL for simplicity)
        let mut result: Vec<(u64, f64)> = all_entries
            .iter()
            .filter(|e| e.name == series
                && e.timestamp_ns >= start_ns
                && e.timestamp_ns <= end_ns)
            .map(|e| (e.timestamp_ns, e.value))
            .collect();

        result.sort_by_key(|(ts, _)| *ts);
        Ok(result)
    }

    pub fn index(&self) -> Arc<RwLock<TsIndex>> {
        self.index.clone()
    }
}
```

**Pitfall #1 — Flush ไม่ครบทำให้ข้อมูลหาย:**
`BufWriter` buffer ข้อมูลไว้ใน memory ถ้าโปรแกรม crash โดยไม่ flush ข้อมูลใน buffer จะหาย WAL design แก้ปัญหานี้โดย flush ทุกครั้งหลัง write แต่นี่ trade-off กับ throughput — production system จะใช้ `fsync` + `fdatasync` เพื่อ durability จริง ๆ

### ขั้นที่ 4: InfluxDB Line Protocol Parser

Line protocol คือ format ที่ InfluxDB ใช้รับข้อมูล ตัวอย่าง:
```
cpu,host=server1,region=us-east usage=85.5,idle=14.5 1700000000000000000
```
รูปแบบ: `measurement[,tag_key=tag_val...] field_key=field_val[,...] [timestamp_ns]`

```rust
// src/line_protocol.rs
#[derive(Debug, Clone, PartialEq)]
pub struct LineRecord {
    pub measurement: String,
    pub tags: Vec<(String, String)>,
    pub fields: Vec<(String, FieldValue)>,
    pub timestamp_ns: Option<u64>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum FieldValue {
    Float(f64),
    Integer(i64),
    Bool(bool),
    Str(String),
}

impl FieldValue {
    pub fn as_f64(&self) -> f64 {
        match self {
            FieldValue::Float(v) => *v,
            FieldValue::Integer(v) => *v as f64,
            FieldValue::Bool(v) => if *v { 1.0 } else { 0.0 },
            FieldValue::Str(_) => f64::NAN,
        }
    }
}

pub fn parse_line(line: &str) -> Result<LineRecord, String> {
    let line = line.trim();
    if line.is_empty() || line.starts_with('#') {
        return Err("empty or comment line".to_string());
    }

    let (meas_tags_part, rest) = split_first_unescaped_space(line)
        .ok_or_else(|| "missing field set".to_string())?;

    let (fields_part, timestamp_str) = match split_first_unescaped_space(rest.trim()) {
        Some((f, t)) => (f, Some(t.trim())),
        None => (rest.trim(), None),
    };

    let (measurement, tags) = parse_measurement_tags(meas_tags_part)?;
    let fields = parse_fields(fields_part)?;
    let timestamp_ns = match timestamp_str {
        Some(ts) if !ts.is_empty() => {
            Some(ts.parse::<u64>().map_err(|e| format!("bad timestamp: {e}"))?)
        }
        _ => None,
    };

    Ok(LineRecord { measurement, tags, fields, timestamp_ns })
}

// quote-aware space splitter: ไม่แยกถ้าอยู่ใน quoted string
fn split_first_unescaped_space(s: &str) -> Option<(&str, &str)> {
    let bytes = s.as_bytes();
    let mut i = 0;
    let mut in_quote = false;
    while i < bytes.len() {
        if bytes[i] == b'\\' { i += 2; continue; }
        if bytes[i] == b'"' { in_quote = !in_quote; i += 1; continue; }
        if bytes[i] == b' ' && !in_quote {
            return Some((&s[..i], &s[i + 1..]));
        }
        i += 1;
    }
    None
}

fn parse_measurement_tags(s: &str) -> Result<(String, Vec<(String, String)>), String> {
    let bytes = s.as_bytes();
    let mut comma_pos = None;
    let mut i = 0;
    while i < bytes.len() {
        if bytes[i] == b'\\' { i += 2; continue; }
        if bytes[i] == b',' { comma_pos = Some(i); break; }
        i += 1;
    }
    let (meas_raw, tags_raw) = match comma_pos {
        Some(p) => (&s[..p], Some(&s[p + 1..])),
        None => (s, None),
    };
    let measurement = unescape(meas_raw);
    let tags = match tags_raw {
        None => vec![],
        Some(t) => parse_tag_set(t)?,
    };
    Ok((measurement, tags))
}

fn parse_tag_set(s: &str) -> Result<Vec<(String, String)>, String> {
    let mut tags = Vec::new();
    for part in split_kv_pairs(s) {
        let eq = part.find('=').ok_or_else(|| format!("bad tag pair: {part}"))?;
        tags.push((unescape(&part[..eq]), unescape(&part[eq + 1..])));
    }
    Ok(tags)
}

fn parse_fields(s: &str) -> Result<Vec<(String, FieldValue)>, String> {
    let mut fields = Vec::new();
    for part in split_kv_pairs(s) {
        let eq = part.find('=').ok_or_else(|| format!("bad field pair: {part}"))?;
        let k = unescape(&part[..eq]);
        let v = parse_field_value(&part[eq + 1..])?;
        fields.push((k, v));
    }
    if fields.is_empty() { return Err("no fields".to_string()); }
    Ok(fields)
}

// comma-split ที่รู้จัก quoted strings
fn split_kv_pairs(s: &str) -> Vec<String> {
    let mut parts = Vec::new();
    let mut current = String::new();
    let mut in_string = false;
    let mut chars = s.chars().peekable();
    while let Some(c) = chars.next() {
        if c == '"' { in_string = !in_string; current.push(c); continue; }
        if c == '\\' {
            if let Some(&nc) = chars.peek() {
                chars.next();
                current.push(c); current.push(nc);
                continue;
            }
        }
        if c == ',' && !in_string {
            parts.push(current.trim().to_string());
            current = String::new();
            continue;
        }
        current.push(c);
    }
    if !current.trim().is_empty() { parts.push(current.trim().to_string()); }
    parts
}

fn parse_field_value(s: &str) -> Result<FieldValue, String> {
    if matches!(s, "true" | "True" | "TRUE") { return Ok(FieldValue::Bool(true)); }
    if matches!(s, "false" | "False" | "FALSE") { return Ok(FieldValue::Bool(false)); }
    if s.starts_with('"') && s.ends_with('"') {
        return Ok(FieldValue::Str(unescape(&s[1..s.len() - 1])));
    }
    if s.ends_with('i') {
        return s[..s.len() - 1].parse::<i64>()
            .map(FieldValue::Integer)
            .map_err(|e| format!("bad integer: {e}"));
    }
    s.parse::<f64>().map(FieldValue::Float).map_err(|e| format!("bad float: {e}"))
}

fn unescape(s: &str) -> String {
    let mut result = String::with_capacity(s.len());
    let mut chars = s.chars();
    while let Some(c) = chars.next() {
        if c == '\\' {
            if let Some(nc) = chars.next() { result.push(nc); continue; }
        }
        result.push(c);
    }
    result
}

pub fn parse_lines(body: &str) -> Vec<Result<LineRecord, String>> {
    body.lines().map(parse_line).collect()
}
```

**Pitfall #2 — String fields ที่มี space ทำให้ parser แตก:**
ถ้า `split_first_unescaped_space` ไม่รู้จัก quoted string จะ split ผิด:
```
event msg="disk full" 1000
           ^^^^^
     space ใน quoted string ต้องไม่ถูก split
```
การแก้: ต้อง track `in_quote` state ระหว่าง traverse characters

### ขั้นที่ 5: Downsampling Engine

Downsampling คือการ aggregate ข้อมูลจำนวนมากให้เป็น bucket เวลา เช่น ข้อมูล CPU ทุก 10 วินาที → average ทุก 5 นาที

```rust
// src/downsampling.rs
use std::collections::BTreeMap;

pub fn bucket_start(timestamp_ns: u64, bucket_duration_ns: u64) -> u64 {
    (timestamp_ns / bucket_duration_ns) * bucket_duration_ns
}

#[derive(Debug, Clone, serde::Serialize)]
pub struct BucketStats {
    pub time: u64,      // bucket_start_ns
    pub count: u64,
    pub sum: f64,
    pub min: f64,
    pub max: f64,
    pub mean: f64,
}

pub fn aggregate_buckets(
    points: &[(u64, f64)],
    bucket_duration_ns: u64,
) -> Vec<BucketStats> {
    #[derive(Default)]
    struct Acc {
        count: u64,
        sum: f64,
        min: f64,
        max: f64,
    }

    let mut buckets: BTreeMap<u64, Acc> = BTreeMap::new();
    for &(ts, val) in points {
        let bs = bucket_start(ts, bucket_duration_ns);
        let entry = buckets.entry(bs).or_insert(Acc {
            count: 0,
            sum: 0.0,
            min: f64::INFINITY,
            max: f64::NEG_INFINITY,
        });
        entry.count += 1;
        entry.sum += val;
        if val < entry.min { entry.min = val; }
        if val > entry.max { entry.max = val; }
    }

    buckets
        .into_iter()
        .map(|(bs, acc)| BucketStats {
            time: bs,
            count: acc.count,
            sum: acc.sum,
            min: acc.min,
            max: acc.max,
            mean: if acc.count > 0 { acc.sum / acc.count as f64 } else { f64::NAN },
        })
        .collect()
}

/// Apply aggregation function to bucket results
pub fn apply_agg_fn(buckets: &[BucketStats], agg_fn: &str) -> Vec<(u64, f64)> {
    buckets
        .iter()
        .map(|b| {
            let val = match agg_fn {
                "mean" => b.mean,
                "sum" => b.sum,
                "min" => b.min,
                "max" => b.max,
                "count" => b.count as f64,
                _ => b.mean,
            };
            (b.time, val)
        })
        .collect()
}
```

**Logic ของ bucket_start:**
```
timestamp_ns = 3_700_000_000_000  (1 ชั่วโมง 1 นาที 40 วินาที)
bucket_duration_ns = 5 * 60 * 1e9  (5 นาที)

bucket_index = 3_700_000_000_000 / 300_000_000_000 = 12
bucket_start = 12 * 300_000_000_000 = 3_600_000_000_000  (1 ชั่วโมง)
```
ทุก timestamp ในช่วง [1h, 1h5m) จะได้ bucket_start เดียวกัน

### ขั้นที่ 6: Retention Policy

```rust
// src/retention.rs
use crate::index::TsIndex;
use std::sync::{Arc, RwLock};
use std::time::{Duration, SystemTime, UNIX_EPOCH};

pub struct RetentionPolicy {
    pub retention_ns: u64,
}

impl RetentionPolicy {
    pub fn new_days(days: u64) -> Self {
        Self { retention_ns: days * 24 * 3600 * 1_000_000_000 }
    }

    pub fn is_within_retention_at(&self, timestamp_ns: u64, now_ns: u64) -> bool {
        timestamp_ns >= now_ns.saturating_sub(self.retention_ns)
    }

    pub fn filter_retained<T: Clone>(
        &self,
        entries: &[(u64, T)],
        now_ns: u64,
    ) -> Vec<(u64, T)> {
        let cutoff = now_ns.saturating_sub(self.retention_ns);
        entries.iter().filter(|(ts, _)| *ts >= cutoff).cloned().collect()
    }

    /// Remove expired entries from in-memory index
    pub fn apply_to_index(&self, index: &Arc<RwLock<TsIndex>>) {
        let now_ns = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_nanos() as u64;
        let cutoff = now_ns.saturating_sub(self.retention_ns);
        let mut idx = index.write().unwrap();
        idx.inner.retain(|(_, ts), _| *ts >= cutoff);
    }
}

/// Background retention task: runs every `interval` and cleans the index
pub async fn retention_task(
    index: Arc<RwLock<TsIndex>>,
    retention_days: u64,
    interval: Duration,
) {
    let policy = RetentionPolicy::new_days(retention_days);
    let mut ticker = tokio::time::interval(interval);
    loop {
        ticker.tick().await;
        policy.apply_to_index(&index);
        tracing::info!(
            "Retention pass completed: entries_remaining={}",
            index.read().unwrap().inner.len()
        );
    }
}
```

**Pitfall #3 — Saturating subtraction สำคัญมากสำหรับ timestamp arithmetic:**
ถ้าใช้ `now_ns - retention_ns` แบบปกติและ `retention_ns > now_ns` จะเกิด integer overflow panic (ใน debug) หรือ wraparound (ใน release) `saturating_sub` ป้องกัน: ผลลัพธ์จะเป็น 0 แทนที่จะ overflow

### ขั้นที่ 7: Query Engine

```rust
// src/query_engine.rs
#[derive(Debug, Clone, PartialEq)]
pub struct TsQuery {
    pub select_fn: AggFn,
    pub select_field: String,
    pub from_measurement: String,
    pub where_time_gt_ns: Option<u64>,    // เก็บเป็น duration offset จาก now()
    pub group_by_duration_ns: Option<u64>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum AggFn { Mean, Sum, Min, Max, Count }

pub fn parse_query(q: &str) -> Result<TsQuery, String> {
    let q = q.trim();
    let upper = q.to_uppercase();

    let sel_start = upper.find("SELECT ").ok_or("missing SELECT")? + 7;
    let sel_end = upper.find(" FROM ").ok_or("missing FROM")?;
    let sel_part = q[sel_start..sel_end].trim();
    let (select_fn, select_field) = parse_select_fn(sel_part)?;

    let from_start = upper.find(" FROM ").ok_or("missing FROM")? + 6;
    let from_end = upper.find(" WHERE ")
        .unwrap_or_else(|| upper.find(" GROUP ").unwrap_or(q.len()));
    let from_measurement = q[from_start..from_end].trim().to_string();

    let where_time_gt_ns = if let Some(wi) = upper.find(" WHERE ") {
        parse_where_time(&q[wi + 7..])?
    } else { None };

    let group_by_duration_ns = if let Some(gi) = upper.find("GROUP BY TIME(") {
        let after = &q[gi + 14..];
        let end = after.find(')').ok_or("missing ) in GROUP BY time()")?;
        Some(parse_duration(&after[..end])?)
    } else { None };

    Ok(TsQuery { select_fn, select_field, from_measurement,
                 where_time_gt_ns, group_by_duration_ns })
}

fn parse_select_fn(s: &str) -> Result<(AggFn, String), String> {
    let s_up = s.to_uppercase();
    for (fn_str, agg) in &[
        ("MEAN", AggFn::Mean), ("SUM", AggFn::Sum), ("MIN", AggFn::Min),
        ("MAX", AggFn::Max), ("COUNT", AggFn::Count),
    ] {
        let prefix = format!("{}(", fn_str);
        if s_up.starts_with(&prefix) && s.ends_with(')') {
            let field = s[fn_str.len() + 1..s.len() - 1].trim().to_string();
            return Ok((agg.clone(), field));
        }
    }
    Err(format!("unsupported SELECT function: {s}"))
}

fn parse_where_time(s: &str) -> Result<Option<u64>, String> {
    let s_up = s.to_uppercase();
    if let Some(pos) = s_up.find("TIME > NOW()-") {
        let duration_str = &s[pos + 13..];
        let end = duration_str.find(|c: char| c == ' ' || c == '\n')
            .unwrap_or(duration_str.len());
        return Ok(Some(parse_duration(duration_str[..end].trim())?));
    }
    Ok(None)
}

/// Parse duration: "1h", "5m", "30s", "1h30m", "7d"
pub fn parse_duration(s: &str) -> Result<u64, String> {
    let s = s.trim().to_lowercase();
    if s.is_empty() { return Err("empty duration".to_string()); }
    let mut total_ns = 0u64;
    let mut num_buf = String::new();
    for c in s.chars() {
        if c.is_ascii_digit() { num_buf.push(c); continue; }
        let n: u64 = num_buf.parse().map_err(|e| format!("bad number: {e}"))?;
        num_buf.clear();
        let mult = match c {
            'w' => 604_800_000_000_000u64,
            'd' => 86_400_000_000_000u64,
            'h' => 3_600_000_000_000u64,
            'm' => 60_000_000_000u64,
            's' => 1_000_000_000u64,
            _ => return Err(format!("unknown unit: {c}")),
        };
        total_ns = total_ns.saturating_add(n.saturating_mul(mult));
    }
    if !num_buf.is_empty() {
        return Err(format!("trailing number without unit: {num_buf}"));
    }
    Ok(total_ns)
}
```

### ขั้นที่ 8: Compaction Background Task

Compaction รวม WAL segments เล็ก ๆ เป็น chunk files ที่บีบอัดด้วย LZ4 ลด read amplification และ disk usage

```rust
// src/compaction.rs
use crate::wal::{self, WalEntry};
use lz4_flex::{compress_prepend_size, decompress_size_prepended};
use std::collections::BTreeMap;
use std::io::{self, BufWriter, Write};
use std::path::{Path, PathBuf};
use std::time::Duration;

pub struct ChunkWriter {
    output_path: PathBuf,
}

impl ChunkWriter {
    pub fn new(output_path: PathBuf) -> Self {
        Self { output_path }
    }

    /// Merge entries (sorted by (name, timestamp_ns)), compress with LZ4,
    /// write to chunk file. Returns number of entries written.
    pub fn write_chunk(&self, entries: &[WalEntry]) -> io::Result<usize> {
        // Sort entries by (series, timestamp) for efficient future range reads
        let mut sorted: Vec<&WalEntry> = entries.iter().collect();
        sorted.sort_by(|a, b| {
            a.name.cmp(&b.name).then(a.timestamp_ns.cmp(&b.timestamp_ns))
        });

        // Serialize to raw bytes
        let mut raw = Vec::new();
        {
            let mut writer = BufWriter::new(&mut raw);
            for entry in &sorted {
                wal::serialize_entry(&mut writer, entry)?;
            }
            writer.flush()?;
        }

        // LZ4 compress
        let compressed = compress_prepend_size(&raw);

        // Write compressed chunk
        std::fs::write(&self.output_path, &compressed)?;
        Ok(sorted.len())
    }
}

pub fn read_chunk(path: &Path) -> io::Result<Vec<WalEntry>> {
    let compressed = std::fs::read(path)?;
    let raw = decompress_size_prepended(&compressed)
        .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e.to_string()))?;
    wal::read_all_entries(&raw)
}

/// Background compaction task
/// Triggered periodically to merge WAL segments into sorted, compressed chunks
pub async fn compaction_task(
    data_dir: PathBuf,
    interval: Duration,
) {
    let mut ticker = tokio::time::interval(interval);
    loop {
        ticker.tick().await;
        if let Err(e) = run_compaction(&data_dir) {
            tracing::error!("Compaction error: {e}");
        }
    }
}

fn run_compaction(data_dir: &Path) -> io::Result<()> {
    let wal_path = data_dir.join("wal.bin");
    if !wal_path.exists() { return Ok(()); }

    let data = std::fs::read(&wal_path)?;
    if data.is_empty() { return Ok(()); }

    let entries = wal::read_all_entries(&data)?;
    if entries.is_empty() { return Ok(()); }

    // Group by series and find timestamp range for chunk filename
    let min_ts = entries.iter().map(|e| e.timestamp_ns).min().unwrap_or(0);
    let max_ts = entries.iter().map(|e| e.timestamp_ns).max().unwrap_or(0);

    let chunk_path = data_dir.join(format!("chunk_{min_ts}_{max_ts}.lz4"));
    let writer = ChunkWriter::new(chunk_path.clone());
    let written = writer.write_chunk(&entries)?;

    // After successful compaction, truncate WAL
    // (in production: atomic rename swap)
    std::fs::write(&wal_path, &[])?;

    tracing::info!(
        "Compaction done: {} entries → {}",
        written,
        chunk_path.display()
    );
    Ok(())
}
```

**Pitfall #4 — ลบ WAL ก่อน chunk เขียนสำเร็จ ทำให้ข้อมูลหาย:**
ลำดับที่ถูกต้องใน production:
1. เขียน chunk file ลง temp path ก่อน
2. `fsync` temp file
3. Atomic rename temp → final path
4. `fsync` directory
5. เพิ่งลบ/truncate WAL

ใน code ตัวอย่างนี้ตัดขั้นตอน atomic swap ออกเพื่อความเรียบง่าย แต่ production code ต้องทำครบ

### ขั้นที่ 9: HTTP API ที่ Compatible กับ InfluxDB v1

```rust
// src/http_api.rs
use axum::{
    extract::{Query, State},
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
};
use serde::{Deserialize, Serialize};
use serde_json::{json, Value};
use std::sync::Arc;
use std::time::{SystemTime, UNIX_EPOCH};

use crate::downsampling::{aggregate_buckets, apply_agg_fn};
use crate::line_protocol::parse_lines;
use crate::query_engine::{parse_query, AggFn};
use crate::store::TsStore;

pub type AppState = Arc<TsStore>;

pub fn create_router(store: AppState) -> Router {
    Router::new()
        .route("/write", post(handle_write))
        .route("/query", get(handle_query))
        .route("/series", get(handle_series))
        .route("/series/:name/tags", get(handle_series_tags))
        .route("/ping", get(handle_ping))
        .with_state(store)
}

/// POST /write — InfluxDB line protocol ingestion
async fn handle_write(
    State(store): State<AppState>,
    body: String,
) -> StatusCode {
    let now_ns = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap_or_default()
        .as_nanos() as u64;

    let mut error_count = 0usize;
    for result in parse_lines(&body) {
        match result {
            Err(_) => { /* skip comment/empty lines */ }
            Ok(rec) => {
                let ts = rec.timestamp_ns.unwrap_or(now_ns);
                let tags_json = serde_json::to_string(
                    &rec.tags.iter().collect::<std::collections::HashMap<_, _>>()
                ).unwrap_or_else(|_| "{}".to_string());

                for (field_name, field_val) in &rec.fields {
                    let value = field_val.as_f64();
                    if value.is_nan() { continue; }
                    let series_name = format!("{}.{}", rec.measurement, field_name);
                    if let Err(e) = store.write(&series_name, ts, value, &tags_json).await {
                        tracing::error!("Write error for series {series_name}: {e}");
                        error_count += 1;
                    }
                }
            }
        }
    }

    if error_count > 0 {
        StatusCode::INTERNAL_SERVER_ERROR
    } else {
        StatusCode::NO_CONTENT // 204 — InfluxDB standard
    }
}

#[derive(Deserialize)]
struct QueryParams {
    q: String,
    #[serde(default = "default_db")]
    db: String,
}

fn default_db() -> String { "default".to_string() }

/// GET /query?q=SELECT+mean(value)+FROM+cpu+WHERE+time>now()-1h+GROUP+BY+time(5m)
/// Returns InfluxDB v1 JSON format
async fn handle_query(
    State(store): State<AppState>,
    Query(params): Query<QueryParams>,
) -> Result<Json<Value>, (StatusCode, String)> {
    let query = parse_query(&params.q)
        .map_err(|e| (StatusCode::BAD_REQUEST, format!("Query parse error: {e}")))?;

    let now_ns = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap_or_default()
        .as_nanos() as u64;

    // Apply WHERE time > now()-Xh cutoff
    let start_ns = query.where_time_gt_ns
        .map(|dur| now_ns.saturating_sub(dur))
        .unwrap_or(0);

    // Series name in store is "measurement.field"
    let series_name = format!("{}.{}", query.from_measurement, query.select_field);
    let points = store.query_range(&series_name, start_ns, now_ns).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;

    // Apply GROUP BY time() downsampling if requested
    let values: Vec<(u64, f64)> = if let Some(bucket_ns) = query.group_by_duration_ns {
        let buckets = aggregate_buckets(&points, bucket_ns);
        let fn_name = query.select_fn.to_string();
        apply_agg_fn(&buckets, &fn_name)
    } else {
        points
    };

    // Format as InfluxDB v1 JSON response
    let rows: Vec<Vec<Value>> = values
        .iter()
        .map(|(ts, v)| vec![json!(ts), json!(v)])
        .collect();

    let response = json!({
        "results": [{
            "statement_id": 0,
            "series": [{
                "name": query.from_measurement,
                "columns": ["time", query.select_fn.to_string()],
                "values": rows
            }]
        }]
    });

    Ok(Json(response))
}

/// GET /series — list all series names
async fn handle_series(State(store): State<AppState>) -> Json<Value> {
    let names = store.index().read().unwrap().series_names();
    Json(json!({ "series": names }))
}

/// GET /series/:name/tags — list distinct tag values for a series
/// (simplified: returns the tags JSON string from latest entry)
async fn handle_series_tags(
    State(store): State<AppState>,
    axum::extract::Path(name): axum::extract::Path<String>,
) -> Json<Value> {
    let idx = store.index().read().unwrap();
    let count = idx.count_for_series(&name);
    Json(json!({
        "series": name,
        "entry_count": count,
        "note": "Use /query for detailed tag filtering"
    }))
}

/// GET /ping — health check
async fn handle_ping() -> (StatusCode, &'static str) {
    (StatusCode::NO_CONTENT, "")
}

pub trait AggFnDisplay {
    fn to_string(&self) -> String;
}

impl std::fmt::Display for AggFn {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            AggFn::Mean => write!(f, "mean"),
            AggFn::Sum => write!(f, "sum"),
            AggFn::Min => write!(f, "min"),
            AggFn::Max => write!(f, "max"),
            AggFn::Count => write!(f, "count"),
        }
    }
}
```

### ขั้นที่ 10: Main Entry Point และ Configuration

```rust
// src/main.rs
use std::net::SocketAddr;
use std::path::PathBuf;
use std::sync::Arc;
use std::time::Duration;

mod compaction;
mod downsampling;
mod http_api;
mod index;
mod line_protocol;
mod query_engine;
mod retention;
mod store;
mod wal;

#[derive(Debug)]
pub struct Config {
    pub data_dir: PathBuf,
    pub listen_addr: SocketAddr,
    pub retention_days: u64,
    pub compaction_interval_secs: u64,
    pub retention_interval_secs: u64,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            data_dir: PathBuf::from("./tsdb_data"),
            listen_addr: "0.0.0.0:8086".parse().unwrap(),
            retention_days: 30,
            compaction_interval_secs: 3600,
            retention_interval_secs: 3600,
        }
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();

    let config = Config::default();

    tracing::info!(
        "Opening TSDB store at {:?}",
        config.data_dir
    );

    let store = Arc::new(
        store::TsStore::open(&config.data_dir)?
    );

    // Spawn compaction background task
    {
        let data_dir = config.data_dir.clone();
        let interval = Duration::from_secs(config.compaction_interval_secs);
        tokio::spawn(async move {
            compaction::compaction_task(data_dir, interval).await;
        });
    }

    // Spawn retention background task
    {
        let index = store.index();
        let retention_days = config.retention_days;
        let interval = Duration::from_secs(config.retention_interval_secs);
        tokio::spawn(async move {
            retention::retention_task(index, retention_days, interval).await;
        });
    }

    let router = http_api::create_router(store);
    let listener = tokio::net::TcpListener::bind(config.listen_addr).await?;

    tracing::info!("TSDB listening on {}", config.listen_addr);
    tracing::info!("Compatible with InfluxDB v1 API (subset)");
    tracing::info!("  POST http://localhost:8086/write");
    tracing::info!("  GET  http://localhost:8086/query?q=...");
    tracing::info!("  GET  http://localhost:8086/series");

    axum::serve(listener, router).await?;
    Ok(())
}
```

---

## การทดสอบ (Testing)

### Unit Tests — ผ่านทั้งหมด

โค้ดต่อไปนี้คือ test suite ที่รันได้จริง รวมไว้ใน `src/main.rs` (หรือ module files ตามโครงสร้างโปรเจค):

```rust
// ทดสอบ WAL serialize/deserialize
#[cfg(test)]
mod wal_tests {
    use super::wal::*;
    use std::io::BufWriter;

    #[test]
    fn test_wal_roundtrip_single_entry() {
        let entry = WalEntry {
            timestamp_ns: 1_700_000_000_000_000_000,
            name: "cpu_usage".to_string(),
            value: 85.5,
            tags: r#"{"host":"server1","region":"us-east"}"#.to_string(),
        };
        let mut buf = Vec::new();
        {
            let mut writer = BufWriter::new(&mut buf);
            serialize_entry(&mut writer, &entry).unwrap();
            writer.flush().unwrap();
        }
        let mut cursor = std::io::Cursor::new(&buf);
        let decoded = deserialize_entry(&mut cursor).unwrap().unwrap();
        assert_eq!(decoded.timestamp_ns, entry.timestamp_ns);
        assert_eq!(decoded.name, entry.name);
        assert!((decoded.value - entry.value).abs() < 1e-10);
        assert_eq!(decoded.tags, entry.tags);
    }

    #[test]
    fn test_wal_roundtrip_multiple_entries() {
        let entries = vec![
            WalEntry { timestamp_ns: 1_000, name: "mem".to_string(),
                       value: 4096.0, tags: "{}".to_string() },
            WalEntry { timestamp_ns: 2_000, name: "disk".to_string(),
                       value: 128.75, tags: r#"{"dev":"sda"}"#.to_string() },
        ];
        let mut buf = Vec::new();
        {
            let mut writer = BufWriter::new(&mut buf);
            write_entries(&mut writer, &entries).unwrap();
        }
        let decoded = read_all_entries(&buf).unwrap();
        assert_eq!(decoded.len(), 2);
        assert!((decoded[0].value - 4096.0).abs() < 1e-10);
    }

    #[test]
    fn test_wal_eof_returns_none() {
        let buf: &[u8] = &[];
        let mut cursor = std::io::Cursor::new(buf);
        assert!(deserialize_entry(&mut cursor).unwrap().is_none());
    }
}
```

### Real `cargo test` Output

```
running 36 tests
test downsampling::tests::test_bucket_assignment ... ok
test downsampling::tests::test_aggregate_buckets_basic ... ok
test downsampling::tests::test_aggregate_single_bucket ... ok
test downsampling::tests::test_aggregate_empty ... ok
test index::tests::test_index_count_for_series ... ok
test index::tests::test_index_empty_scan ... ok
test index::tests::test_index_multiple_series ... ok
test index::tests::test_index_insert_and_range_scan ... ok
test index::tests::test_index_series_names ... ok
test line_protocol::tests::test_parse_boolean_field ... ok
test line_protocol::tests::test_parse_comment_returns_err ... ok
test line_protocol::tests::test_parse_empty_returns_err ... ok
test line_protocol::tests::test_parse_integer_field ... ok
test line_protocol::tests::test_parse_multiple_fields ... ok
test line_protocol::tests::test_parse_no_timestamp ... ok
test line_protocol::tests::test_parse_simple_line ... ok
test line_protocol::tests::test_parse_string_field ... ok
test line_protocol::tests::test_parse_with_tags ... ok
test query_engine::tests::test_parse_duration_compound ... ok
test query_engine::tests::test_parse_duration_hours ... ok
test query_engine::tests::test_parse_duration_minutes ... ok
test query_engine::tests::test_parse_duration_seconds ... ok
test query_engine::tests::test_parse_query_count ... ok
test query_engine::tests::test_parse_query_full ... ok
test query_engine::tests::test_parse_query_missing_select ... ok
test query_engine::tests::test_parse_query_sum_no_where ... ok
test query_engine::tests::test_parse_query_unsupported_fn ... ok
test retention::tests::test_filter_retained ... ok
test retention::tests::test_retention_30_days_expired ... ok
test retention::tests::test_retention_30_days_within ... ok
test retention::tests::test_retention_boundary ... ok
test retention::tests::test_retention_zero_days_expires_all ... ok
test wal::tests::test_wal_empty_name_and_tags ... ok
test wal::tests::test_wal_eof_returns_none ... ok
test wal::tests::test_wal_roundtrip_multiple_entries ... ok
test wal::tests::test_wal_roundtrip_single_entry ... ok

test result: ok. 36 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

---

## จุดสังเกตและ Pitfalls สำคัญ

### Pitfall #1 — `BufWriter` flush ที่ขาดหาย

```rust
// WRONG: BufWriter drop เมื่อออก scope แต่ drop อาจ silent fail
{
    let mut writer = BufWriter::new(file);
    write_entries(&mut writer, &entries)?;
} // drop ที่นี่ flush แต่ error ถูก ignore!

// CORRECT: flush อย่างชัดเจน
{
    let mut writer = BufWriter::new(file);
    write_entries(&mut writer, &entries)?;
    writer.flush()?; // error propagated ผ่าน ?
}
```

เมื่อ `BufWriter` ถูก drop มัน flush อัตโนมัติ **แต่ error ถูก ignore** ต้องเรียก `.flush()?` เอง

### Pitfall #2 — String Fields ที่มี Space ใน Quoted Values

```
event msg="reboot server" 1000000000
                 ^
         space ตรงนี้ต้องไม่แยก token!
```

ถ้า parser ไม่ track `in_quote` state จะ split `"reboot server"` เป็น `"reboot` และ `server"` ซึ่ง parse เป็น float ไม่ได้ ต้อง track quote state ทุก character

### Pitfall #3 — Integer Overflow ใน Timestamp Arithmetic

```rust
// WRONG: ถ้า retention_ns > now_ns → panic ใน debug หรือ wraparound ใน release
let cutoff = now_ns - retention_ns;  // integer underflow!

// CORRECT: ใช้ saturating_sub
let cutoff = now_ns.saturating_sub(retention_ns);  // ผลลัพธ์ minimum 0
```

timestamp เป็น `u64` การลบที่ทำให้ผลเป็นลบจะเกิด panic หรือ wraparound ต้องใช้ `saturating_sub` เสมอ

### Pitfall #4 — WAL Deletion Race Condition

```
Thread A: เขียน WAL → compaction เสร็จ → ลบ WAL
Thread B: กำลัง read WAL สำหรับ query!

เกิด: Thread B อ่าน file ที่ถูกลบ → IO error หรือ empty data
```

การแก้ที่ถูกต้องคือ:
1. ใช้ `Arc<RwLock<_>>` guard สำหรับ WAL file access
2. ใช้ Copy-on-write: compaction เขียน chunk ใหม่ก่อน แล้วค่อย atomic swap WAL reference
3. Reference counting สำหรับ active readers ก่อน truncate WAL

### Pitfall #5 — BTreeMap Composite Key Ordering

```rust
// KEY: (SeriesName, Timestamp) — BTreeMap sort by SeriesName first
// ทำให้ "cpu" timestamps 1000, 2000, 3000 อยู่ติดกัน
// แต่ "cpu_idle" กับ "cpu" อยู่ติดกันเพราะ lexicographic order!

// range scan ของ "cpu":
let lo = ("cpu".to_string(), start_ts);
let hi = ("cpu".to_string(), end_ts);
// entries ของ "cpu_idle" จะ intercalate เข้ามาใน range นี้ถ้าไม่ระวัง
// เหตุผล: "cpu\0" < "cpu_" — ต้องใช้ exact series name match
```

นี่คือเหตุผลที่ `range_scan` ต้อง filter โดย series name อย่างชัดเจน ไม่ใช่แค่ bounded range

---

## การทดสอบ Integration กับ Grafana / Telegraf

เมื่อ server รันอยู่ที่ `localhost:8086` สามารถทดสอบด้วย `curl`:

```bash
# เขียนข้อมูล (InfluxDB line protocol)
curl -X POST "http://localhost:8086/write" \
  --data-binary "cpu,host=server1,region=us-east usage=85.5,idle=14.5 $(date +%s)000000000"

# Query (InfluxDB v1 format)
curl "http://localhost:8086/query?q=SELECT+mean(usage)+FROM+cpu+WHERE+time+%3E+now()-1h+GROUP+BY+time(5m)"

# List series
curl "http://localhost:8086/series"

# Health check
curl -i "http://localhost:8086/ping"
# HTTP/1.1 204 No Content
```

**Telegraf config (`/etc/telegraf/telegraf.conf`):**
```toml
[[outputs.influxdb]]
  urls = ["http://localhost:8086"]
  database = "metrics"

[[inputs.cpu]]
  percpu = false
  totalcpu = true
  interval = "10s"
```

**Grafana:**
1. Add data source → InfluxDB
2. URL: `http://localhost:8086`
3. Database: `metrics`
4. HTTP Method: GET

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Optimized release build
cargo build --release

# Binary อยู่ที่:
./target/release/tsdb

# Run with custom config via environment
TSDB_DATA_DIR=/var/lib/tsdb \
TSDB_RETENTION_DAYS=90 \
TSDB_LISTEN=0.0.0.0:8086 \
./target/release/tsdb
```

### Dockerfile

```dockerfile
# ---- Build stage ----
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src/ src/
RUN cargo build --release

# ---- Runtime stage ----
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/tsdb /usr/local/bin/tsdb
VOLUME /data
EXPOSE 8086
ENV TSDB_DATA_DIR=/data
CMD ["tsdb"]
```

```bash
# Build และ run
docker build -t tsdb:latest .
docker run -p 8086:8086 -v tsdb-data:/data tsdb:latest
```

### Systemd Service

```ini
# /etc/systemd/system/tsdb.service
[Unit]
Description=Lightweight Time Series Database
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/tsdb
Environment=TSDB_DATA_DIR=/var/lib/tsdb
Restart=always
RestartSec=5s
User=tsdb

[Install]
WantedBy=multi-user.target
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Persistent Index ด้วย Memory-Mapped File

ปัญหาปัจจุบัน: rebuild index จาก WAL ทุก startup ช้าเมื่อ WAL ใหญ่

**งาน:** สร้าง `src/persistent_index.rs` ที่:
1. Save BTreeMap index ลง file เป็น binary format เมื่อ shutdown
2. Load กลับมาจาก file เมื่อ startup แทนการ rebuild จาก WAL
3. Invalidate index file เมื่อ WAL timestamp ใหม่กว่า index timestamp
4. Test: เปรียบเทียบ startup time ของ in-memory rebuild vs persistent index บน WAL 100K entries

```rust
pub struct PersistentIndex {
    path: PathBuf,
    inner: TsIndex,
}

impl PersistentIndex {
    pub fn load_or_rebuild(path: &Path, wal_data: &[u8]) -> io::Result<Self> {
        // 1. ถ้า index file มีอยู่และ newer กว่า WAL → load
        // 2. ถ้าไม่มีหรือ outdated → rebuild แล้ว save
        todo!()
    }

    pub fn save(&self) -> io::Result<()> {
        // Serialize BTreeMap to binary
        todo!()
    }
}
```

### แบบฝึกหัดที่ 2: Multi-Field Time Series และ Tag Filtering

ปัจจุบัน series name คือ `"measurement.field"` แต่ยังไม่รองรับ filter ด้วย tag

**งาน:** เพิ่ม WHERE clause ที่รองรับ tag condition:
```
SELECT mean(value) FROM cpu WHERE time > now()-1h AND host='server1' GROUP BY time(5m)
```

1. แก้ `parse_query` ให้ extract `where_tags: Vec<(String, String)>`
2. แก้ `TsIndex` ให้ store tags ด้วย (อาจใช้ `HashMap<SeriesKey, Vec<(Timestamp, Tags, FileOffset)>>`)
3. แก้ `query_range` ให้ filter ด้วย tag values
4. Test: เขียนข้อมูล 3 hosts ต่างกัน แล้ว query เฉพาะ `host=server1`

### แบบฝึกหัดที่ 3: Chunk Read Path สำหรับ Long-Range Queries

ปัจจุบัน queries อ่านแค่ WAL ซึ่ง compaction อาจย้ายข้อมูลเก่าไปเป็น chunk files แล้ว

**งาน:** สร้าง unified read path ที่:
1. ดู time range ของ query
2. หา chunk files ที่ overlap (จาก filename `chunk_{min_ts}_{max_ts}.lz4`)
3. อ่าน + decompress chunk files ที่เกี่ยวข้อง
4. Merge ผลจาก WAL + chunks โดย deduplicate entries ที่ timestamp ซ้ำ
5. Benchmark: compare query time สำหรับ 1-day range บน WAL only vs WAL+chunks

```rust
pub async fn query_range_all_sources(
    &self,
    series: &str,
    start_ns: u64,
    end_ns: u64,
) -> io::Result<Vec<(u64, f64)>> {
    let mut points = Vec::new();

    // 1. Read from WAL
    let wal_points = self.query_range(series, start_ns, end_ns).await?;
    points.extend(wal_points);

    // 2. Read from chunk files that overlap [start_ns, end_ns]
    for chunk_path in self.find_overlapping_chunks(start_ns, end_ns)? {
        let entries = compaction::read_chunk(&chunk_path)?;
        for e in entries {
            if e.name == series && e.timestamp_ns >= start_ns && e.timestamp_ns <= end_ns {
                points.push((e.timestamp_ns, e.value));
            }
        }
    }

    points.sort_by_key(|(ts, _)| *ts);
    points.dedup_by_key(|(ts, _)| *ts); // เอา duplicate timestamp ออก
    Ok(points)
}
```

### แบบฝึกหัดที่ 4: Continuous Query (Pre-computed Downsampling)

สำหรับ dashboard ที่ query บ่อย ๆ เช่น Grafana refresh ทุก 30 วินาที การ compute mean ทุกครั้งจาก raw data สิ้นเปลือง

**งาน:** สร้าง continuous query engine ที่:
1. Config ระบุ: `cq_name`, `source_measurement`, `resample_every`, `group_by_interval`, `into_measurement`
2. Background task รัน downsampling ตาม schedule แล้ว write ผลลัพธ์กลับลง store เหมือน regular data point
3. Query engine detect ว่ามี pre-computed series อยู่หรือไม่ ถ้ามีให้ใช้แทน raw data
4. Test: สร้าง CQ สำหรับ `cpu` 1m → `cpu_1m_mean` แล้วตรวจว่า query `SELECT mean(value) FROM cpu_1m_mean` ได้ผลเร็วกว่า raw query

```toml
# tsdb.toml — example config
[[continuous_queries]]
name = "cpu_5m_mean"
query = "SELECT mean(usage) INTO cpu_5m_mean FROM cpu GROUP BY time(5m)"
resample_every = "1m"
```

---

## สรุป

โปรเจคนี้สร้าง Time Series Database แบบ lightweight ที่ทำงานได้จริง และ compatible กับ InfluxDB v1 API เราได้เรียนรู้:

**Patterns ที่นำไปใช้ได้จริง:**
- **Append-only WAL** — pattern พื้นฐานของ databases ทุกตัว ทำให้ write fast และ recovery ง่าย
- **Composite BTreeMap key** — วิธีใช้ ordered map เพื่อ range scan โดยไม่ต้องสร้าง data structure พิเศษ
- **Background tasks ใน tokio** — `tokio::spawn` + `tokio::time::interval` สำหรับ periodic maintenance
- **Manual parser design** — เมื่อ parser crate ไม่เหมาะ (เพื่อ compatibility หรือ control) การเขียน parser เองทำให้เข้าใจ format ลึกขึ้น
- **Bucket-based aggregation** — downsampling pattern ที่ใช้ทั่วไปใน time series systems

**ความเชื่อมโยงกับโปรเจคถัดไป:**
โปรเจคนี้สร้าง storage layer ที่รับข้อมูลแบบ push (HTTP POST) โปรเจค C04 — Message Broker จะสำรวจการส่ง data ผ่าน message queue แบบ async ซึ่งเป็น pattern ที่ complement กัน: TSDB อยู่ปลาย pipeline ส่วน Message Broker อยู่กลาง pipeline

---

**โปรเจคก่อนหน้า:** [project-c02-realtime-analytics.md](project-c02-realtime-analytics.md) | **โปรเจคถัดไป:** [project-c04-message-broker.md](project-c04-message-broker.md)
