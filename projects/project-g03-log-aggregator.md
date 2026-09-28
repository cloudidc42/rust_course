# Project G03: Log Aggregation System

> โมดูล: G — DevOps & Infrastructure | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ในโลก production ทุก service ที่รันอยู่ต่างปล่อย log ออกมาอย่างต่อเนื่อง ไม่ว่าจะเป็น web server, background worker, database query หรือ scheduled job การที่จะ debug ปัญหาหรือ monitor สุขภาพของระบบได้จริง ต้องมีระบบ **Log Aggregation** ที่:

1. **รับ log จากหลายรูปแบบ** — structured JSON, plain text สไตล์ Apache, logfmt
2. **กรองและแปลง** ข้อมูลก่อนเก็บ — ตัด noise ออก, เพิ่ม metadata, unmask fields
3. **เก็บไว้ใน ring buffer** แบบ in-memory — ไม่รั่วหน่วยความจำเมื่อ log ทะลักเข้ามา
4. **ค้นหาและ query** ด้วย time range, full-text search, field filter
5. **วิเคราะห์ด้วย aggregation** — count, rate, top-k, error rate
6. **ส่งออกหลายรูปแบบ** — JSON Lines, logfmt, plain text

**Use case จริงในโลก production:**
- **Sidecar log collector**: ทำงานคู่กับ application container เพื่อ buffer และส่ง log ต่อไปยัง backend
- **Log tail & analysis**: CLI tool ที่ดูด log จาก file/socket แล้ว query แบบ real-time
- **Alerting pipeline**: กรอง error pattern แล้วส่ง notification เมื่อ error rate พุ่งสูง
- **Audit trail**: เก็บ security event ไว้ใน tamper-evident ring buffer สำหรับ compliance

**Learning value:**
โปรเจคนี้สอนการออกแบบ **trait-based pipeline architecture** ที่ทั้ง composable และ extensible ครอบคลุมตั้งแต่ parsing, filtering, transformation, buffering จนถึง querying และ aggregation — ทักษะที่ใช้ได้กับ event streaming, metrics collection และ data pipeline ทุกประเภท

## สิ่งที่จะได้เรียนรู้

- **Trait objects สำหรับ pipeline** — `Box<dyn Filter>`, `Box<dyn Transform>`, `Box<dyn Formatter>` ใช้แทนกันได้ในขณะ runtime
- **VecDeque เป็น ring buffer** — ควบคุมขนาด memory ด้วย overflow policy แบบ drop-oldest
- **Regex parsing** ด้วย `regex` crate — ดึงข้อมูลจาก unstructured text อย่างปลอดภัย
- **Builder pattern** สำหรับ query API — method chaining แบบ type-safe
- **HashMap-based aggregation** — count_by, top_k, rate calculation ล้วนทำจาก `HashMap`
- **serde สำหรับ JSON parsing แบบ dynamic** — `serde_json::Value` แทน fixed struct เมื่อ schema ไม่แน่นอน
- **chrono สำหรับ time-based query** — DateTime arithmetic และ time range filtering
- **Error propagation ด้วย `?`** — เปลี่ยน parse error เป็น `Result<T, String>` อย่างสะอาด

## ความรู้ที่ต้องมีมาก่อน

- **Part 15-20**: Traits, trait objects, `Box<dyn Trait>`, dynamic dispatch
- **Part 21-24**: Collections — `HashMap`, `VecDeque`, iterators
- **Part 25-30**: Error handling ด้วย `Result<T, E>`, `?` operator
- **Part 35-40**: Closures, `filter_map`, `collect`, iterator chains
- **Part 55-60**: `serde` ecosystem — Serialize/Deserialize, `serde_json`
- **Part 96-100**: `regex` crate, pattern matching บน unstructured data
- **Part 101-105**: Design patterns — builder, strategy, pipeline

## โครงสร้างโปรเจค (Project Layout)

```
log-aggregator/
├── src/
│   ├── lib.rs           # re-export ทุก module
│   ├── types.rs         # LogRecord, LogLevel, Value
│   ├── parser.rs        # LogParser trait, JsonLogParser, RegexLogParser
│   ├── filter.rs        # Filter trait, LevelFilter, RegexFilter, FieldExistsFilter
│   ├── transform.rs     # Transform trait, AddField, RenameField, ParseJsonField, MaskField
│   ├── buffer.rs        # LogBuffer (VecDeque ring buffer) + BufferStats
│   ├── query.rs         # LogQuery builder + TimeRange + FieldFilter
│   ├── aggregation.rs   # count_by, rate_per_minute, top_k, error_rate
│   ├── output.rs        # Formatter trait, JsonLinesFormatter, LogfmtFormatter, TextFormatter
│   └── pipeline.rs      # Pipeline builder: filter + transform chain
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ภาพรวม

```
[Raw Log String]
      │
      ▼ LogParser (JSON / Regex)
[LogRecord { timestamp, level, service, message, fields }]
      │
      ▼ Pipeline
  ┌───────────────────────────────────┐
  │  Filter₁ → Filter₂ → ... → Filterₙ  │  ← reject early
  │  Transform₁ → Transform₂ → ...    │  ← mutate record
  └───────────────────────────────────┘
      │ Some(record)  │ None (filtered out)
      ▼               ▼
[LogBuffer]         (discard)
(VecDeque, fixed capacity, drop-oldest)
      │
      ▼ LogQuery.execute(&buffer)
[Vec<LogRecord>]  ← filtered snapshot
      │
      ├──▶ Aggregation functions  →  HashMap<String, usize>, f64
      │
      └──▶ Formatter (JSONL / logfmt / text)  →  String output
```

### Core Abstractions

ระบบแบ่งออกเป็นชั้นที่แยกจากกันชัดเจน 3 ชั้น:

```
trait LogParser   → แปลง &str → Result<LogRecord, String>
trait Filter      → ตัดสินใจ: record นี้ผ่านหรือไม่
trait Transform   → แปลง record → record ใหม่ (immutable appearance, แต่ takes ownership)
trait Formatter   → แปลง &LogRecord → String สำหรับ output
```

**ทำไม `fields: HashMap<String, Value>` แทน typed struct?**

Log จากระบบต่างกันมี field ต่างกันโดยสิ้นเชิง เช่น HTTP log มี `status_code`, `method`, `path` แต่ database log มี `query_time_ms`, `rows_affected` การใช้ `HashMap<String, Value>` ทำให้ parse และจัดการ log ได้ทุกรูปแบบโดยไม่ต้องสร้าง struct ใหม่ทุกครั้ง

**ทำไมใช้ `Box<dyn Filter>` ไม่ใช่ generics?**

Pipeline ต้องเก็บ filters หลายชนิดใน `Vec` เดียวกัน เช่น `[LevelFilter, RegexFilter, FieldExistsFilter]` ถ้าใช้ generics จะต้อง encode ทุก type ใน type signature ของ Pipeline: `Pipeline<(LevelFilter, (RegexFilter, FieldExistsFilter))>` ซึ่งซับซ้อนมาก `Box<dyn Filter>` อ่านง่ายกว่ามาก และ overhead ของ vtable lookup นั้นน้อยมากเมื่อเทียบกับ I/O และ regex matching ที่ทำจริง

**VecDeque เป็น ring buffer:**

`VecDeque` รองรับ `pop_front()` และ `push_back()` ใน O(1) amortized — เหมาะมากสำหรับ FIFO queue ที่ต้องการ drop-oldest policy เมื่อ capacity เต็ม ต่างจาก `Vec` ที่ต้อง `drain(0..1)` แล้ว shift ทุกอย่าง O(n)

### ลำดับความสำคัญของ Design Decisions

| Decision | ทางเลือก | เหตุผลที่เลือก |
|----------|----------|----------------|
| Dynamic fields | `HashMap<String, Value>` | Schema ไม่แน่นอน, รองรับทุก log format |
| Pipeline storage | `Vec<Box<dyn Trait>>` | ต้องผสม types ใน Vec เดียว |
| Buffer | `VecDeque` | O(1) pop_front/push_back สำหรับ ring buffer |
| Error type | `String` | Simple, ไม่ต้อง dependency เพิ่ม |
| Time | `chrono::DateTime<Utc>` | UTC ทุกที่, ไม่มีปัญหา timezone |

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Types — LogRecord และ LogLevel

เริ่มจากโครงสร้างข้อมูลหลัก ทุกส่วนของระบบจะใช้ types เหล่านี้

สร้าง `Cargo.toml`:

```toml
[package]
name = "log-aggregator"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
regex = "1"
chrono = { version = "0.4", features = ["serde"] }

[lib]
name = "log_aggregator"
path = "src/lib.rs"
```

สร้าง `src/types.rs`:

```rust
use std::collections::HashMap;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// Log level — derive PartialOrd/Ord เพื่อเปรียบเทียบ level ได้ (Info < Warn < Error)
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
#[serde(rename_all = "UPPERCASE")]
pub enum LogLevel {
    Trace,
    Debug,
    Info,
    Warn,
    Error,
    Fatal,
}

impl std::fmt::Display for LogLevel {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            LogLevel::Trace => write!(f, "TRACE"),
            LogLevel::Debug => write!(f, "DEBUG"),
            LogLevel::Info  => write!(f, "INFO"),
            LogLevel::Warn  => write!(f, "WARN"),
            LogLevel::Error => write!(f, "ERROR"),
            LogLevel::Fatal => write!(f, "FATAL"),
        }
    }
}

impl std::str::FromStr for LogLevel {
    type Err = String;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s.to_uppercase().as_str() {
            "TRACE"              => Ok(LogLevel::Trace),
            "DEBUG"              => Ok(LogLevel::Debug),
            "INFO"               => Ok(LogLevel::Info),
            "WARN" | "WARNING"   => Ok(LogLevel::Warn),
            "ERROR"              => Ok(LogLevel::Error),
            "FATAL" | "CRITICAL" => Ok(LogLevel::Fatal),
            other => Err(format!("unknown log level: {}", other)),
        }
    }
}

/// Dynamic value type สำหรับ log fields
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(untagged)]
pub enum Value {
    Null,
    Bool(bool),
    Int(i64),
    Float(f64),
    String(String),
    Array(Vec<Value>),
    Object(HashMap<String, Value>),
}

impl std::fmt::Display for Value {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Value::Null        => write!(f, "null"),
            Value::Bool(b)     => write!(f, "{}", b),
            Value::Int(n)      => write!(f, "{}", n),
            Value::Float(n)    => write!(f, "{}", n),
            Value::String(s)   => write!(f, "{}", s),
            Value::Array(arr)  => {
                write!(f, "[")?;
                for (i, v) in arr.iter().enumerate() {
                    if i > 0 { write!(f, ",")?; }
                    write!(f, "{}", v)?;
                }
                write!(f, "]")
            }
            Value::Object(_)   => write!(f, "{{...}}"),
        }
    }
}

impl From<serde_json::Value> for Value {
    fn from(v: serde_json::Value) -> Self {
        match v {
            serde_json::Value::Null        => Value::Null,
            serde_json::Value::Bool(b)     => Value::Bool(b),
            serde_json::Value::Number(n)   => {
                if let Some(i) = n.as_i64() { Value::Int(i) }
                else { Value::Float(n.as_f64().unwrap_or(0.0)) }
            }
            serde_json::Value::String(s)   => Value::String(s),
            serde_json::Value::Array(arr)  => Value::Array(arr.into_iter().map(Value::from).collect()),
            serde_json::Value::Object(obj) => Value::Object(
                obj.into_iter().map(|(k, v)| (k, Value::from(v))).collect()
            ),
        }
    }
}

/// Log record เดียว — หน่วยพื้นฐานของระบบทั้งหมด
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LogRecord {
    pub timestamp: DateTime<Utc>,
    pub level:     LogLevel,
    pub service:   String,
    pub message:   String,
    pub fields:    HashMap<String, Value>,
}

impl LogRecord {
    pub fn new(
        timestamp: DateTime<Utc>,
        level: LogLevel,
        service: impl Into<String>,
        message: impl Into<String>,
    ) -> Self {
        LogRecord {
            timestamp,
            level,
            service: service.into(),
            message: message.into(),
            fields: HashMap::new(),
        }
    }

    pub fn with_field(mut self, key: impl Into<String>, value: Value) -> Self {
        self.fields.insert(key.into(), value);
        self
    }
}
```

จุดสำคัญ:
- `#[derive(PartialOrd, Ord)]` บน `LogLevel` ทำให้ `level >= LogLevel::Warn` ใช้ได้ทันที — enum variants ถูกเปรียบเทียบตามลำดับที่ประกาศ
- `#[serde(untagged)]` บน `Value` ทำให้ JSON number ถูก deserialize เป็น `Int` หรือ `Float` โดยอัตโนมัติ
- `From<serde_json::Value>` ช่วยแปลง JSON object ที่ parse ได้เป็น `HashMap<String, Value>` ของเราโดยตรง

### ขั้นที่ 2: Log Parsers — JSON และ Regex

สร้าง `src/parser.rs` — ทั้ง `JsonLogParser` และ `RegexLogParser` implement trait `LogParser` เดียวกัน:

```rust
use std::str::FromStr;
use chrono::{DateTime, Utc};
use regex::Regex;

use crate::types::{LogLevel, LogRecord, Value};

/// Trait สำหรับ parse log จาก raw string
pub trait LogParser: Send + Sync {
    fn parse(&self, input: &str) -> Result<LogRecord, String>;
    fn name(&self) -> &str;
}

/// Parse structured JSON log lines
/// รูปแบบ: {"timestamp":"...","level":"INFO","service":"api","message":"...","fields":{}}
pub struct JsonLogParser;

impl LogParser for JsonLogParser {
    fn name(&self) -> &str { "json" }

    fn parse(&self, input: &str) -> Result<LogRecord, String> {
        let v: serde_json::Value = serde_json::from_str(input)
            .map_err(|e| format!("JSON parse error: {}", e))?;

        let obj = v.as_object().ok_or("expected JSON object")?;

        let timestamp_str = obj.get("timestamp")
            .and_then(|v| v.as_str())
            .ok_or("missing 'timestamp'")?;

        let timestamp = DateTime::parse_from_rfc3339(timestamp_str)
            .map(|dt| dt.with_timezone(&Utc))
            .map_err(|e| format!("bad timestamp: {}", e))?;

        let level_str = obj.get("level")
            .and_then(|v| v.as_str())
            .ok_or("missing 'level'")?;

        let level = LogLevel::from_str(level_str)
            .map_err(|e| format!("bad level: {}", e))?;

        let service = obj.get("service")
            .and_then(|v| v.as_str())
            .unwrap_or("unknown")
            .to_string();

        let message = obj.get("message")
            .and_then(|v| v.as_str())
            .ok_or("missing 'message'")?
            .to_string();

        let fields = if let Some(fields_val) = obj.get("fields") {
            if let Some(fields_obj) = fields_val.as_object() {
                fields_obj.iter()
                    .map(|(k, v)| (k.clone(), Value::from(v.clone())))
                    .collect()
            } else {
                std::collections::HashMap::new()
            }
        } else {
            std::collections::HashMap::new()
        };

        Ok(LogRecord { timestamp, level, service, message, fields })
    }
}

/// Parse plain text log lines ด้วย regex
/// รูปแบบ: [2024-01-15T10:30:00Z] [INFO] [api] message text
pub struct RegexLogParser {
    pattern: Regex,
}

impl RegexLogParser {
    pub fn new() -> Self {
        let pattern = Regex::new(
            r"^\[(?P<timestamp>[^\]]+)\]\s+\[(?P<level>[^\]]+)\]\s+\[(?P<service>[^\]]+)\]\s+(?P<message>.+)$"
        ).expect("invalid regex");
        RegexLogParser { pattern }
    }

    pub fn with_pattern(pattern: &str) -> Result<Self, String> {
        let re = Regex::new(pattern)
            .map_err(|e| format!("regex error: {}", e))?;
        Ok(RegexLogParser { pattern: re })
    }
}

impl LogParser for RegexLogParser {
    fn name(&self) -> &str { "regex" }

    fn parse(&self, input: &str) -> Result<LogRecord, String> {
        let caps = self.pattern.captures(input)
            .ok_or_else(|| format!("no match for input: {}", input))?;

        let timestamp_str = caps.name("timestamp")
            .ok_or("missing timestamp group")?.as_str();

        let timestamp = DateTime::parse_from_rfc3339(timestamp_str)
            .map(|dt| dt.with_timezone(&Utc))
            .map_err(|e| format!("bad timestamp '{}': {}", timestamp_str, e))?;

        let level_str = caps.name("level")
            .ok_or("missing level group")?.as_str();

        let level = LogLevel::from_str(level_str)
            .map_err(|e| format!("bad level: {}", e))?;

        let service = caps.name("service")
            .map(|m| m.as_str().to_string())
            .unwrap_or_else(|| "unknown".to_string());

        let message = caps.name("message")
            .map(|m| m.as_str().to_string())
            .unwrap_or_default();

        Ok(LogRecord::new(timestamp, level, service, message))
    }
}
```

วิธีใช้ parsers:

```rust
// JSON parser
let parser = JsonLogParser;
let json_line = r#"{"timestamp":"2024-01-15T10:30:00Z","level":"ERROR","service":"db","message":"connection pool exhausted","fields":{"pool_size":10,"waiting":47}}"#;
let record = parser.parse(json_line).unwrap();
println!("Parsed: {} - {}", record.level, record.message);
// Output: Parsed: ERROR - connection pool exhausted

// Regex parser
let parser = RegexLogParser::new();
let text_line = "[2024-01-15T10:30:00Z] [WARN] [cache] eviction rate high";
let record = parser.parse(text_line).unwrap();
println!("Parsed: {} from {}", record.level, record.service);
// Output: Parsed: WARN from cache
```

จุดสำคัญเรื่อง regex named captures:
- `(?P<name>...)` สร้าง named capture group ที่ดึงได้ด้วย `caps.name("name")`
- Pattern ใช้ `[^\]]+` เพื่อ match ทุกอย่างยกเว้น `]` — อ่านง่ายกว่า `.+?` และ backtrack น้อยกว่า
- `ok_or_else(|| ...)` แทน `ok_or(...)` เพราะ `format!` ที่อยู่ใน closure ไม่ถูก evaluate เมื่อไม่จำเป็น

### ขั้นที่ 3: Filter และ Transform Pipeline

Filter กรอง record, Transform เปลี่ยน record — ทั้งคู่ทำงานเป็น chain ใน Pipeline

**`src/filter.rs`** — filters ที่ใช้บ่อย:

```rust
use regex::Regex;
use crate::types::{LogLevel, LogRecord};

pub trait Filter: Send + Sync {
    fn matches(&self, record: &LogRecord) -> bool;
    fn name(&self) -> &str;
}

/// เก็บ record ที่ level >= min_level
pub struct LevelFilter {
    min_level: LogLevel,
}

impl LevelFilter {
    pub fn new(min_level: LogLevel) -> Self { LevelFilter { min_level } }
}

impl Filter for LevelFilter {
    fn name(&self) -> &str { "level_filter" }
    fn matches(&self, record: &LogRecord) -> bool {
        record.level >= self.min_level
    }
}

/// เก็บ record ที่ message match pattern
pub struct RegexFilter {
    pattern: Regex,
    invert:  bool,
}

impl RegexFilter {
    pub fn new(pattern: &str) -> Result<Self, String> {
        Regex::new(pattern)
            .map(|re| RegexFilter { pattern: re, invert: false })
            .map_err(|e| format!("regex error: {}", e))
    }
    pub fn inverted(mut self) -> Self { self.invert = true; self }
}

impl Filter for RegexFilter {
    fn name(&self) -> &str { "regex_filter" }
    fn matches(&self, record: &LogRecord) -> bool {
        let matched = self.pattern.is_match(&record.message);
        if self.invert { !matched } else { matched }
    }
}

/// เก็บ record ที่มี field ชื่อนี้
pub struct FieldExistsFilter { field_name: String }

impl FieldExistsFilter {
    pub fn new(field_name: impl Into<String>) -> Self {
        FieldExistsFilter { field_name: field_name.into() }
    }
}

impl Filter for FieldExistsFilter {
    fn name(&self) -> &str { "field_exists_filter" }
    fn matches(&self, record: &LogRecord) -> bool {
        record.fields.contains_key(&self.field_name)
    }
}
```

**`src/transform.rs`** — transforms ที่ใช้บ่อย:

```rust
use crate::types::{LogRecord, Value};

pub trait Transform: Send + Sync {
    fn apply(&self, record: LogRecord) -> LogRecord;
    fn name(&self) -> &str;
}

/// เพิ่ม field คงที่ทุก record
pub struct AddFieldTransform { key: String, value: Value }

impl AddFieldTransform {
    pub fn new(key: impl Into<String>, value: Value) -> Self {
        AddFieldTransform { key: key.into(), value }
    }
}

impl Transform for AddFieldTransform {
    fn name(&self) -> &str { "add_field" }
    fn apply(&self, mut record: LogRecord) -> LogRecord {
        record.fields.insert(self.key.clone(), self.value.clone());
        record
    }
}

/// เปลี่ยนชื่อ field
pub struct RenameFieldTransform { from: String, to: String }

impl RenameFieldTransform {
    pub fn new(from: impl Into<String>, to: impl Into<String>) -> Self {
        RenameFieldTransform { from: from.into(), to: to.into() }
    }
}

impl Transform for RenameFieldTransform {
    fn name(&self) -> &str { "rename_field" }
    fn apply(&self, mut record: LogRecord) -> LogRecord {
        if let Some(val) = record.fields.remove(&self.from) {
            record.fields.insert(self.to.clone(), val);
        }
        record
    }
}

/// Parse string field เป็น JSON แล้ว expand เป็น fields
pub struct ParseJsonFieldTransform {
    field:  String,
    prefix: Option<String>,
}

impl ParseJsonFieldTransform {
    pub fn new(field: impl Into<String>) -> Self {
        ParseJsonFieldTransform { field: field.into(), prefix: None }
    }
    pub fn with_prefix(mut self, prefix: impl Into<String>) -> Self {
        self.prefix = Some(prefix.into());
        self
    }
}

impl Transform for ParseJsonFieldTransform {
    fn name(&self) -> &str { "parse_json_field" }
    fn apply(&self, mut record: LogRecord) -> LogRecord {
        if let Some(Value::String(s)) = record.fields.get(&self.field).cloned() {
            if let Ok(serde_json::Value::Object(obj)) = serde_json::from_str::<serde_json::Value>(&s) {
                for (k, v) in obj {
                    let key = match &self.prefix {
                        Some(p) => format!("{}_{}", p, k),
                        None    => k,
                    };
                    record.fields.insert(key, Value::from(v));
                }
                record.fields.remove(&self.field);
            }
        }
        record
    }
}

/// Redact field ที่มี sensitive data
pub struct MaskFieldTransform { field: String, mask: String }

impl MaskFieldTransform {
    pub fn new(field: impl Into<String>) -> Self {
        MaskFieldTransform {
            field: field.into(),
            mask: "***REDACTED***".to_string(),
        }
    }
}

impl Transform for MaskFieldTransform {
    fn name(&self) -> &str { "mask_field" }
    fn apply(&self, mut record: LogRecord) -> LogRecord {
        if record.fields.contains_key(&self.field) {
            record.fields.insert(
                self.field.clone(),
                Value::String(self.mask.clone()),
            );
        }
        record
    }
}
```

**`src/pipeline.rs`** — ประกอบ filters และ transforms เข้าด้วยกัน:

```rust
use crate::filter::Filter;
use crate::transform::Transform;
use crate::types::LogRecord;

pub struct Pipeline {
    filters:    Vec<Box<dyn Filter>>,
    transforms: Vec<Box<dyn Transform>>,
}

impl Pipeline {
    pub fn new() -> Self {
        Pipeline { filters: Vec::new(), transforms: Vec::new() }
    }

    pub fn filter(mut self, f: impl Filter + 'static) -> Self {
        self.filters.push(Box::new(f));
        self
    }

    pub fn transform(mut self, t: impl Transform + 'static) -> Self {
        self.transforms.push(Box::new(t));
        self
    }

    /// ส่ง record ผ่าน pipeline
    /// คืน None หาก filter ใด filter หนึ่ง reject
    pub fn process(&self, record: LogRecord) -> Option<LogRecord> {
        for f in &self.filters {
            if !f.matches(&record) { return None; }
        }
        let mut r = record;
        for t in &self.transforms {
            r = t.apply(r);
        }
        Some(r)
    }

    pub fn process_batch(&self, records: Vec<LogRecord>) -> Vec<LogRecord> {
        records.into_iter().filter_map(|r| self.process(r)).collect()
    }
}
```

ตัวอย่างการสร้าง pipeline แบบ fluent:

```rust
use log_aggregator::filter::{LevelFilter, RegexFilter};
use log_aggregator::transform::{AddFieldTransform, MaskFieldTransform};
use log_aggregator::types::{LogLevel, Value};
use log_aggregator::pipeline::Pipeline;

let pipeline = Pipeline::new()
    .filter(LevelFilter::new(LogLevel::Warn))
    .filter(RegexFilter::new(r"database|db").unwrap().inverted())  // exclude health checks
    .transform(AddFieldTransform::new("env", Value::String("production".to_string())))
    .transform(MaskFieldTransform::new("api_key"));
```

### ขั้นที่ 4: In-Memory Ring Buffer ด้วย VecDeque

`VecDeque` เป็น double-ended queue ที่ `push_back` และ `pop_front` ทำงานใน O(1) amortized — เหมาะมากสำหรับ ring buffer

สร้าง `src/buffer.rs`:

```rust
use std::collections::VecDeque;
use crate::types::LogRecord;

#[derive(Debug, Clone, Default)]
pub struct BufferStats {
    pub total_received: u64,
    pub total_dropped:  u64,
    pub current_size:   usize,
}

pub struct LogBuffer {
    inner:         VecDeque<LogRecord>,
    capacity:      usize,
    total_received: u64,
    total_dropped:  u64,
}

impl LogBuffer {
    pub fn new(capacity: usize) -> Self {
        assert!(capacity > 0, "capacity must be > 0");
        LogBuffer {
            inner:          VecDeque::with_capacity(capacity),
            capacity,
            total_received: 0,
            total_dropped:  0,
        }
    }

    /// Push record เข้า buffer
    /// ถ้า buffer เต็ม ลบ record เก่าที่สุดออกก่อน
    pub fn push(&mut self, record: LogRecord) {
        self.total_received += 1;
        if self.inner.len() >= self.capacity {
            self.inner.pop_front();
            self.total_dropped += 1;
        }
        self.inner.push_back(record);
    }

    /// ดู records ทั้งหมด (newest last)
    pub fn records(&self) -> Vec<&LogRecord> {
        self.inner.iter().collect()
    }

    /// เอา records ออกจาก buffer ทั้งหมด
    pub fn drain(&mut self) -> Vec<LogRecord> {
        self.inner.drain(..).collect()
    }

    pub fn len(&self)      -> usize { self.inner.len() }
    pub fn is_empty(&self) -> bool  { self.inner.is_empty() }
    pub fn capacity(&self) -> usize { self.capacity }

    pub fn stats(&self) -> BufferStats {
        BufferStats {
            total_received: self.total_received,
            total_dropped:  self.total_dropped,
            current_size:   self.inner.len(),
        }
    }
}
```

ตัวอย่างการใช้งาน:

```rust
use log_aggregator::buffer::LogBuffer;
use log_aggregator::types::{LogLevel, LogRecord};
use chrono::Utc;

let mut buffer = LogBuffer::new(1000);  // เก็บ log ล่าสุด 1000 รายการ

// Simulate log flood
for i in 0..1500 {
    let msg = format!("request #{}", i);
    let record = LogRecord::new(Utc::now(), LogLevel::Info, "api", msg);
    buffer.push(record);
}

let stats = buffer.stats();
println!("Received: {}", stats.total_received);  // 1500
println!("Dropped:  {}", stats.total_dropped);   // 500
println!("Current:  {}", stats.current_size);    // 1000
```

**Pitfall #1: `with_capacity` ไม่ใช่ capacity จริง**

```rust
// ผิด — with_capacity เป็น hint เท่านั้น, ไม่ได้ limit ขนาด
let mut deque: VecDeque<i32> = VecDeque::with_capacity(10);
for i in 0..100 {
    deque.push_back(i);  // จะโตเกิน 10 ได้
}
println!("{}", deque.len());  // 100, ไม่ใช่ 10!

// ถูก — ต้อง check ขนาดเองและ pop_front เมื่อเต็ม
if deque.len() >= 10 {
    deque.pop_front();
}
deque.push_back(new_value);
```

**Pitfall #2: Clone กับ `records()` ที่คืน reference**

```rust
// records() คืน Vec<&LogRecord> — reference เท่านั้น
// ถ้าต้องการ owned copy ต้อง .cloned() เอง
let snapshot: Vec<LogRecord> = buffer.records()
    .into_iter()
    .cloned()
    .collect();
```

### ขั้นที่ 5: Query Engine — ค้นหาด้วย Builder Pattern

`LogQuery` ใช้ builder pattern เพื่อให้สร้าง query ได้อย่างอ่านง่ายและ composable

สร้าง `src/query.rs`:

```rust
use chrono::{DateTime, Utc};
use crate::types::{LogLevel, LogRecord, Value};
use crate::buffer::LogBuffer;

#[derive(Debug, Clone, Default)]
pub struct TimeRange {
    pub from: Option<DateTime<Utc>>,
    pub to:   Option<DateTime<Utc>>,
}

impl TimeRange {
    pub fn from(from: DateTime<Utc>) -> Self {
        TimeRange { from: Some(from), to: None }
    }
    pub fn to(mut self, to: DateTime<Utc>) -> Self {
        self.to = Some(to); self
    }
    pub fn contains(&self, ts: &DateTime<Utc>) -> bool {
        self.from.as_ref().map_or(true, |f| ts >= f)
            && self.to.as_ref().map_or(true, |t| ts <= t)
    }
}

#[derive(Debug, Clone)]
pub enum FieldFilter {
    Equals(String, Value),
    NotEquals(String, Value),
    Exists(String),
}

impl FieldFilter {
    pub fn matches(&self, record: &LogRecord) -> bool {
        match self {
            FieldFilter::Equals(k, v)    => record.fields.get(k) == Some(v),
            FieldFilter::NotEquals(k, v) => record.fields.get(k) != Some(v),
            FieldFilter::Exists(k)       => record.fields.contains_key(k),
        }
    }
}

#[derive(Debug, Clone, Default)]
pub struct LogQuery {
    pub min_level:     Option<LogLevel>,
    pub services:      Vec<String>,
    pub time_range:    Option<TimeRange>,
    pub text_search:   Option<String>,
    pub field_filters: Vec<FieldFilter>,
    pub limit:         Option<usize>,
    pub sort_desc:     bool,
}

impl LogQuery {
    pub fn new() -> Self { LogQuery::default() }

    pub fn min_level(mut self, level: LogLevel)       -> Self { self.min_level = Some(level); self }
    pub fn service(mut self, svc: impl Into<String>)  -> Self { self.services.push(svc.into()); self }
    pub fn time_range(mut self, r: TimeRange)         -> Self { self.time_range = Some(r); self }
    pub fn text_search(mut self, t: impl Into<String>)-> Self { self.text_search = Some(t.into()); self }
    pub fn limit(mut self, n: usize)                  -> Self { self.limit = Some(n); self }
    pub fn sort_desc(mut self)                        -> Self { self.sort_desc = true; self }

    pub fn field_equals(mut self, key: impl Into<String>, value: Value) -> Self {
        self.field_filters.push(FieldFilter::Equals(key.into(), value));
        self
    }

    pub fn execute(&self, buffer: &LogBuffer) -> Vec<LogRecord> {
        let mut results: Vec<LogRecord> = buffer.records()
            .into_iter()
            .filter(|r| self.matches(r))
            .cloned()
            .collect();

        if self.sort_desc {
            results.sort_by(|a, b| b.timestamp.cmp(&a.timestamp));
        } else {
            results.sort_by(|a, b| a.timestamp.cmp(&b.timestamp));
        }

        if let Some(limit) = self.limit {
            results.truncate(limit);
        }
        results
    }

    fn matches(&self, r: &LogRecord) -> bool {
        if let Some(ml) = &self.min_level {
            if &r.level < ml { return false; }
        }
        if !self.services.is_empty() && !self.services.contains(&r.service) {
            return false;
        }
        if let Some(range) = &self.time_range {
            if !range.contains(&r.timestamp) { return false; }
        }
        if let Some(text) = &self.text_search {
            if !r.message.to_lowercase().contains(&text.to_lowercase()) {
                return false;
            }
        }
        for ff in &self.field_filters {
            if !ff.matches(r) { return false; }
        }
        true
    }
}
```

ตัวอย่างการ query:

```rust
use chrono::Utc;
use log_aggregator::query::{LogQuery, TimeRange};
use log_aggregator::types::{LogLevel, Value};

// Query: Error ใน 5 นาทีที่ผ่านมา จาก service "api" หรือ "payments"
let five_min_ago = Utc::now() - chrono::Duration::minutes(5);
let results = LogQuery::new()
    .min_level(LogLevel::Error)
    .service("api")
    .service("payments")
    .time_range(TimeRange::from(five_min_ago).to(Utc::now()))
    .sort_desc()        // newest first
    .limit(50)
    .execute(&buffer);

println!("Found {} recent errors", results.len());

// Query: Full-text search + field filter
let results = LogQuery::new()
    .text_search("timeout")
    .field_equals("retry_count", Value::Int(3))
    .execute(&buffer);
```

**Pitfall #3: Full-text search บน message เท่านั้น**

Query engine นี้ทำ full-text search เฉพาะ `message` field ไม่รวม dynamic `fields` ถ้าต้องการค้นใน fields ด้วย ต้องใช้ `FieldFilter::Equals` แยกต่างหาก หรือ extend `LogQuery` ให้ search ใน `fields.values()` ด้วย

### ขั้นที่ 6: Aggregation Functions

Aggregation ทำงานบน `&[LogRecord]` slice — รับ query result มาวิเคราะห์ต่อ

สร้าง `src/aggregation.rs`:

```rust
use std::collections::HashMap;
use chrono::Duration;
use crate::types::{LogLevel, LogRecord};

/// นับจำนวน record แต่ละ unique value ของ field
pub fn count_by(records: &[LogRecord], field: &str) -> HashMap<String, usize> {
    let mut counts: HashMap<String, usize> = HashMap::new();
    for r in records {
        let key = if field == "level" {
            r.level.to_string()
        } else if field == "service" {
            r.service.clone()
        } else if let Some(v) = r.fields.get(field) {
            v.to_string()
        } else {
            "__missing__".to_string()
        };
        *counts.entry(key).or_insert(0) += 1;
    }
    counts
}

/// คำนวณอัตรา events ต่อนาที ในช่วงเวลาที่กำหนด
pub fn rate_per_minute(records: &[LogRecord], window: Duration) -> f64 {
    if records.is_empty() { return 0.0; }
    let now = records.iter().map(|r| r.timestamp).max().unwrap();
    let from = now - window;
    let count = records.iter().filter(|r| r.timestamp >= from).count();
    let minutes = window.num_seconds() as f64 / 60.0;
    if minutes <= 0.0 { return 0.0; }
    count as f64 / minutes
}

/// Top-K values ที่พบบ่อยที่สุดสำหรับ field หนึ่ง
pub fn top_k(records: &[LogRecord], field: &str, k: usize) -> Vec<(String, usize)> {
    let counts = count_by(records, field);
    let mut entries: Vec<(String, usize)> = counts.into_iter().collect();
    entries.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    entries.truncate(k);
    entries
}

/// Error rate = errors / total ในช่วงเวลาที่กำหนด
pub fn error_rate(records: &[LogRecord], window: Duration) -> f64 {
    if records.is_empty() { return 0.0; }
    let now = records.iter().map(|r| r.timestamp).max().unwrap();
    let from = now - window;
    let windowed: Vec<&LogRecord> = records.iter()
        .filter(|r| r.timestamp >= from)
        .collect();
    if windowed.is_empty() { return 0.0; }
    let errors = windowed.iter().filter(|r| r.level >= LogLevel::Error).count();
    errors as f64 / windowed.len() as f64
}
```

ตัวอย่างการใช้ aggregation:

```rust
use chrono::Duration;
use log_aggregator::aggregation::{count_by, top_k, error_rate};
use log_aggregator::query::LogQuery;

// ดึง records ล่าสุด 1 ชั่วโมง
let recent = LogQuery::new()
    .time_range(TimeRange::from(Utc::now() - Duration::hours(1)).to(Utc::now()))
    .execute(&buffer);

// นับ errors ต่อ service
let by_service = count_by(&recent, "service");
for (svc, count) in &by_service {
    println!("  {}: {} events", svc, count);
}

// Top 5 services ที่ generate log มากที่สุด
let top5 = top_k(&recent, "service", 5);
println!("Top services: {:?}", top5);

// Error rate ใน 15 นาทีที่ผ่านมา
let err_rate = error_rate(&recent, Duration::minutes(15));
if err_rate > 0.1 {
    println!("ALERT: Error rate {:.1}% exceeds threshold!", err_rate * 100.0);
}
```

### ขั้นที่ 7: Output Formatters

Formatter ทำหน้าที่แปลง `LogRecord` เป็น string สำหรับเขียนออกไปยัง file หรือ stdout

สร้าง `src/output.rs`:

```rust
use crate::types::LogRecord;

pub trait Formatter: Send + Sync {
    fn format(&self, record: &LogRecord) -> String;
    fn name(&self) -> &str;
}

/// JSON Lines: หนึ่ง JSON object ต่อบรรทัด — เหมาะกับ log ingestion
pub struct JsonLinesFormatter;

impl Formatter for JsonLinesFormatter {
    fn name(&self) -> &str { "jsonl" }
    fn format(&self, record: &LogRecord) -> String {
        serde_json::to_string(record)
            .unwrap_or_else(|e| format!("{{\"error\":\"{}\"}}", e))
    }
}

/// logfmt: key=value pairs — เหมาะกับ systemd journal, Loki
pub struct LogfmtFormatter;

impl Formatter for LogfmtFormatter {
    fn name(&self) -> &str { "logfmt" }
    fn format(&self, record: &LogRecord) -> String {
        let mut parts = vec![
            format!("ts={}", record.timestamp.format("%Y-%m-%dT%H:%M:%SZ")),
            format!("level={}", record.level),
            format!("service={}", record.service),
            format!("msg={:?}", record.message),
        ];
        for (k, v) in &record.fields {
            let val_str = match v {
                crate::types::Value::String(s) if s.contains(' ') => format!("{:?}", s),
                other => other.to_string(),
            };
            parts.push(format!("{}={}", k, val_str));
        }
        parts.join(" ")
    }
}

/// Text formatter แบบ configurable template
pub struct TextFormatter { pub template: String }

impl TextFormatter {
    pub fn new() -> Self {
        TextFormatter {
            template: "[{ts}] [{level}] [{service}] {message}".to_string(),
        }
    }
}

impl Formatter for TextFormatter {
    fn name(&self) -> &str { "text" }
    fn format(&self, record: &LogRecord) -> String {
        self.template
            .replace("{ts}",      &record.timestamp.format("%Y-%m-%dT%H:%M:%SZ").to_string())
            .replace("{level}",   &record.level.to_string())
            .replace("{service}", &record.service)
            .replace("{message}", &record.message)
    }
}
```

ตัวอย่าง output แต่ละรูปแบบจากโค้ดเดียวกัน:

```
# JSON Lines
{"timestamp":"2024-01-15T10:30:00Z","level":"ERROR","service":"api","message":"database timeout","fields":{"query_time_ms":5000}}

# logfmt
ts=2024-01-15T10:30:00Z level=ERROR service=api msg="database timeout" query_time_ms=5000

# Text
[2024-01-15T10:30:00Z] [ERROR] [api] database timeout
```

**Pitfall #4: logfmt และ values ที่มีช่องว่าง**

```rust
// ผิด — "New York" ไม่มี quotes จะทำให้ parser อ่าน city=New เท่านั้น
format!("city={}", "New York")  // → city=New York  ← parse ผิด!

// ถูก — ต้อง quote values ที่มีช่องว่าง
format!("city={:?}", "New York")  // → city="New York"  ← parse ถูกต้อง
```

`{:?}` ใน Rust จะเพิ่ม `"..."` รอบ string โดยอัตโนมัติ — ใช้ได้สำหรับ logfmt quoting

## การทดสอบ (Testing)

### Unit Tests ทั้งหมด (42 tests)

Tests กระจายอยู่ในทุก module — รันด้วย `cargo test`:

```
running 42 tests
test aggregation::tests::test_count_by_field ... ok
test aggregation::tests::test_count_by_level ... ok
test aggregation::tests::test_count_by_service ... ok
test aggregation::tests::test_error_rate ... ok
test aggregation::tests::test_rate_per_minute_empty ... ok
test aggregation::tests::test_top_k_all ... ok
test aggregation::tests::test_top_k ... ok
test buffer::tests::test_buffer_basic_push ... ok
test buffer::tests::test_buffer_drain ... ok
test buffer::tests::test_buffer_overflow_drops_oldest ... ok
test buffer::tests::test_buffer_overflow_many ... ok
test filter::tests::test_level_filter_blocks_below ... ok
test filter::tests::test_field_exists_filter ... ok
test filter::tests::test_level_filter_passes_above ... ok
test filter::tests::test_inverted_regex_filter ... ok
test filter::tests::test_level_filter_passes_equal ... ok
test output::tests::test_jsonl_format ... ok
test output::tests::test_logfmt_format ... ok
test filter::tests::test_regex_filter_no_match ... ok
test output::tests::test_text_format ... ok
test parser::tests::test_json_parse_missing_message ... ok
test parser::tests::test_json_parse_valid ... ok
test parser::tests::test_json_parse_invalid_level ... ok
test parser::tests::test_json_parse_warning_alias ... ok
test filter::tests::test_regex_filter_matches ... ok
test parser::tests::test_regex_parse_no_match ... ok
test parser::tests::test_regex_parse_warn_alias ... ok
test parser::tests::test_regex_parse_valid ... ok
test pipeline::tests::test_pipeline_filter_and_transform ... ok
test query::tests::test_query_combined ... ok
test query::tests::test_query_field_equals ... ok
test query::tests::test_query_full_text_search ... ok
test query::tests::test_query_limit ... ok
test query::tests::test_query_min_level ... ok
test query::tests::test_query_service_filter ... ok
test query::tests::test_query_time_range ... ok
test transform::tests::test_add_field ... ok
test transform::tests::test_mask_field ... ok
test transform::tests::test_parse_json_field_with_prefix ... ok
test transform::tests::test_parse_json_field ... ok
test pipeline::tests::test_pipeline_empty_filters ... ok
test transform::tests::test_rename_field ... ok

test result: ok. 42 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests log_aggregator

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### การเขียน Tests ที่ดี

**Test pattern สำหรับ ring buffer overflow:**

```rust
#[test]
fn test_buffer_overflow_drops_oldest() {
    let mut buf = LogBuffer::new(3);
    buf.push(make_record("first"));
    buf.push(make_record("second"));
    buf.push(make_record("third"));
    buf.push(make_record("fourth")); // ดัน "first" ออก

    assert_eq!(buf.len(), 3);
    let stats = buf.stats();
    assert_eq!(stats.total_dropped, 1);

    let records = buf.records();
    assert_eq!(records[0].message, "second");  // เก่าสุดที่เหลือ
    assert_eq!(records[2].message, "fourth");  // ใหม่สุด
}
```

**Test pattern สำหรับ time range query:**

```rust
#[test]
fn test_query_time_range() {
    let buf = fill_buffer(); // records ที่ offset 0, 10, 20, 30, 40 วินาที
    let range = TimeRange::from(ts(5)).to(ts(25));
    let results = LogQuery::new().time_range(range).execute(&buf);
    assert_eq!(results.len(), 2); // offset 10 และ 20 เท่านั้น
}
```

**Test pattern สำหรับ aggregation:**

```rust
#[test]
fn test_count_by_service() {
    let records = make_records();
    let counts = count_by(&records, "service");
    assert_eq!(counts.get("api"), Some(&3));
    assert_eq!(counts.get("payments"), Some(&2));
}

#[test]
fn test_error_rate() {
    let records = make_records(); // 2 errors ใน 5 records
    let rate = error_rate(&records, Duration::seconds(3600));
    assert!((rate - 0.4).abs() < 1e-9); // 2/5 = 0.4 ± floating point
}
```

### Tips สำหรับ Testing

1. **ใช้ helper functions** สร้าง test data แทนที่จะ repeat ทุก test
2. **DateTime arithmetic** ในเทส — ใช้ `Utc.with_ymd_and_hms(2024, 1, 15, 10, 0, 0).unwrap() + Duration::seconds(n)` แทน `Utc::now()` เพื่อให้ deterministic
3. **Floating point comparison** — ใช้ `(a - b).abs() < 1e-9` ไม่ใช่ `==` เพราะ `f64` มี precision error
4. **Test boundary conditions** เสมอ — buffer ที่ว่าง, query ที่ไม่เจออะไร, top_k ที่ k > จำนวนจริง

## การ Package และ Deploy

### Build สำหรับ Production

```bash
# Release build พร้อม optimization
cargo build --release

# ขนาด binary หลัง strip
strip target/release/log-aggregator
ls -lh target/release/log-aggregator
```

### เพิ่ม Binary Target

เพิ่ม `src/main.rs` สำหรับ CLI tool:

```toml
# Cargo.toml
[[bin]]
name = "logagg"
path = "src/main.rs"
```

```rust
// src/main.rs
use log_aggregator::{
    buffer::LogBuffer,
    parser::{JsonLogParser, LogParser},
    query::{LogQuery, TimeRange},
    types::LogLevel,
    aggregation::count_by,
};
use chrono::{Utc, Duration};
use std::io::{self, BufRead};

fn main() {
    let mut buffer = LogBuffer::new(10_000);
    let parser = JsonLogParser;

    // อ่าน stdin ทีละบรรทัด
    let stdin = io::stdin();
    let mut parsed = 0usize;
    let mut failed = 0usize;

    for line in stdin.lock().lines() {
        let line = line.expect("readline failed");
        match parser.parse(&line) {
            Ok(record) => {
                buffer.push(record);
                parsed += 1;
            }
            Err(_) => failed += 1,
        }
    }

    eprintln!("Parsed: {}, Failed: {}", parsed, failed);

    // Query: errors ใน 1 ชั่วโมงที่ผ่านมา
    let results = LogQuery::new()
        .min_level(LogLevel::Error)
        .time_range(TimeRange::from(Utc::now() - Duration::hours(1)).to(Utc::now()))
        .sort_desc()
        .limit(100)
        .execute(&buffer);

    println!("=== Recent Errors ({}) ===", results.len());
    for r in &results {
        println!("[{}] {} - {}", r.level, r.service, r.message);
    }

    // Aggregation: top services
    let all = LogQuery::new().execute(&buffer);
    let top = log_aggregator::aggregation::top_k(&all, "service", 5);
    println!("\n=== Top Services ===");
    for (svc, count) in top {
        println!("  {}: {}", svc, count);
    }
}
```

### Docker Deployment

```dockerfile
FROM rust:1.75 AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/logagg /usr/local/bin/logagg
ENTRYPOINT ["logagg"]
```

Build และรัน:

```bash
docker build -t logagg .

# รับ log จาก Docker container อื่น
docker logs my-service --follow --timestamps | docker run -i logagg
```

### ใช้เป็น Library

```toml
# ในโปรเจคอื่น
[dependencies]
log-aggregator = { path = "../log-aggregator" }
# หรือจาก git
log-aggregator = { git = "https://github.com/your-org/log-aggregator" }
```

## จุดที่ต้องระวัง (Pitfalls)

### Pitfall #1: `VecDeque::with_capacity` ไม่ได้จำกัดขนาด

`with_capacity` เป็น performance hint เท่านั้น ไม่ใช่ hard limit:

```rust
// BUG: capacity เกินกว่าที่ตั้งได้เสมอ
let mut deque: VecDeque<i32> = VecDeque::with_capacity(5);
for i in 0..100 {
    deque.push_back(i);  // จะโตเกิน 5 ได้โดยไม่ error
}
assert_eq!(deque.len(), 100);  // ไม่ใช่ 5!

// FIX: ต้อง check ขนาดเองก่อน push
if deque.len() >= self.capacity {
    deque.pop_front();  // drop oldest
}
deque.push_back(new_item);
```

### Pitfall #2: `LogLevel` ordering ขึ้นกับลำดับ variant

`#[derive(PartialOrd, Ord)]` เปรียบเทียบ enum โดยใช้ลำดับที่ประกาศ variant:

```rust
// ถ้าประกาศผิดลำดับ:
enum LogLevel { Error, Info, Debug }  // Error < Info < Debug ← ผิด!

// ต้องประกาศจากน้อยไปมาก:
enum LogLevel { Trace, Debug, Info, Warn, Error, Fatal }  // ถูก

// จะได้:
assert!(LogLevel::Error > LogLevel::Warn);  // true
assert!(LogLevel::Info < LogLevel::Error);  // true
```

### Pitfall #3: Full-text search ไม่ทำ stemming

`contains` ในภาษาไทยหรือภาษาอังกฤษทำ exact substring match เท่านั้น:

```rust
// "timeout" จะไม่ match "timed out" หรือ "times out"
let found = records.iter().filter(|r| r.message.contains("timeout")).count();

// Solution: ใช้ regex สำหรับ fuzzy matching
let re = Regex::new(r"time[d]?\s*out").unwrap();
let found = records.iter().filter(|r| re.is_match(&r.message)).count();
```

### Pitfall #4: Clone ทั้ง Vec ใน query แพง

`LogQuery::execute` clone ทุก matching record — ถ้า buffer มี record ขนาดใหญ่จะใช้ memory มาก:

```rust
// ถ้า LogRecord มี fields หลายร้อยตัว การ clone ทุกชิ้นแพงมาก
let results = query.execute(&buffer);  // Vec<LogRecord> — cloned ทั้งหมด

// Alternative: คืน references แทน (ต้องการ lifetime)
pub fn execute<'a>(&self, buffer: &'a LogBuffer) -> Vec<&'a LogRecord> {
    buffer.records().into_iter().filter(|r| self.matches(r)).collect()
}

// หรือใช้ limit เสมอเมื่อ buffer ใหญ่
let results = LogQuery::new()
    .min_level(LogLevel::Error)
    .limit(100)  // จำกัดการ clone
    .execute(&buffer);
```

### Pitfall #5: Regex compilation ซ้ำใน hot path

```rust
// BUG: compile regex ทุกครั้งที่ filter ถูก call
fn matches(&self, record: &LogRecord) -> bool {
    let re = Regex::new(&self.pattern_str).unwrap(); // แพงมาก!
    re.is_match(&record.message)
}

// FIX: compile ครั้งเดียวตอน construction เก็บใน struct
pub struct RegexFilter {
    pattern: Regex,  // compiled ครั้งเดียว
}
impl RegexFilter {
    pub fn new(pattern: &str) -> Result<Self, String> {
        Ok(RegexFilter { pattern: Regex::new(pattern).map_err(|e| e.to_string())? })
    }
}
```

### Pitfall #6: HashMap iteration order ไม่ deterministic

`count_by` คืน `HashMap` ที่ iteration order ไม่แน่นอน — อย่าใช้เปรียบเทียบ Vec โดยตรง:

```rust
// BUG: HashMap iteration order ไม่แน่นอน
let counts = count_by(&records, "service");
let v: Vec<_> = counts.into_iter().collect();
assert_eq!(v, vec![("api", 3), ("payments", 2)]);  // อาจ fail!

// FIX: sort ก่อน compare หรือใช้ HashMap comparison
let mut v: Vec<_> = counts.into_iter().collect();
v.sort_by_key(|(k, _)| k.clone());
assert_eq!(v[0], ("api".to_string(), 3));
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Tail Mode

เพิ่ม function `tail(n)` ที่คืน n records ล่าสุดจาก buffer โดยไม่ต้อง query ทั้งหมด:

```rust
// ใน LogBuffer
pub fn tail(&self, n: usize) -> Vec<&LogRecord> {
    let len = self.inner.len();
    let start = len.saturating_sub(n);
    self.inner.range(start..).collect()
}
```

ทดสอบว่า `tail(5)` บน buffer ที่มี 100 records คืนแค่ 5 records ล่าสุด

### แบบฝึกหัดที่ 2: Log Rotation Writer

สร้าง `RotatingFileWriter` ที่เปิดไฟล์ใหม่เมื่อถึง size limit:

```rust
pub struct RotatingFileWriter {
    base_path:    String,
    max_size_mb:  u64,
    current_file: std::fs::File,
    current_size: u64,
    file_index:   u32,
}

impl RotatingFileWriter {
    pub fn write(&mut self, line: &str) -> std::io::Result<()> {
        if self.current_size >= self.max_size_mb * 1024 * 1024 {
            self.rotate()?;
        }
        // write line + newline
        // update current_size
        Ok(())
    }

    fn rotate(&mut self) -> std::io::Result<()> {
        // เปิดไฟล์ใหม่ชื่อ base_path.{index+1}.log
        // reset current_size = 0
        Ok(())
    }
}
```

### แบบฝึกหัดที่ 3: Apache Combined Log Parser

เพิ่ม `ApacheLogParser` สำหรับ parse Apache Combined Log Format:

```
127.0.0.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326 "http://www.example.com/start.html" "Mozilla/4.08 [en] (Win98; I ;Nav)"
```

ดึงข้อมูล: IP, method, path, status code, bytes, referrer, user agent เก็บเป็น fields

### แบบฝึกหัดที่ 4: Alerting Rules Engine

สร้าง `AlertRule` ที่ evaluate ต่อ `Vec<LogRecord>` และคืน alert:

```rust
pub struct AlertRule {
    pub name:      String,
    pub query:     LogQuery,
    pub threshold: usize,          // จำนวน records ที่ trigger alert
    pub window:    Duration,       // ช่วงเวลาที่นับ
}

pub struct Alert {
    pub rule_name: String,
    pub message:   String,
    pub count:     usize,
    pub timestamp: DateTime<Utc>,
}

impl AlertRule {
    pub fn evaluate(&self, buffer: &LogBuffer) -> Option<Alert> {
        let results = self.query.execute(buffer);
        if results.len() >= self.threshold {
            Some(Alert { ... })
        } else {
            None
        }
    }
}
```

### แบบฝึกหัดที่ 5: Metrics Export (Prometheus Format)

เพิ่ม function สร้าง Prometheus text format จาก aggregation results:

```rust
pub fn to_prometheus(records: &[LogRecord], service_label: &str) -> String {
    let by_level = count_by(records, "level");
    let mut lines = vec![
        "# HELP log_records_total Total log records by level".to_string(),
        "# TYPE log_records_total counter".to_string(),
    ];
    for (level, count) in &by_level {
        lines.push(format!(
            r#"log_records_total{{service="{}",level="{}"}} {}"#,
            service_label, level, count
        ));
    }
    lines.join("\n")
}

// Output:
// # HELP log_records_total Total log records by level
// # TYPE log_records_total counter
// log_records_total{service="api",level="ERROR"} 42
// log_records_total{service="api",level="INFO"} 1337
```

### แบบฝึกหัดที่ 6: Multi-source Aggregation พร้อม Async

เปลี่ยน pipeline ให้รับ log จากหลาย source พร้อมกัน ด้วย `tokio`:

```rust
use tokio::sync::mpsc;
use std::sync::Arc;
use tokio::sync::Mutex;

pub async fn run_aggregator(
    sources: Vec<Box<dyn AsyncLogSource>>,
    buffer:  Arc<Mutex<LogBuffer>>,
    pipeline: Arc<Pipeline>,
) {
    let (tx, mut rx) = mpsc::channel::<LogRecord>(1000);

    // spawn task สำหรับแต่ละ source
    for source in sources {
        let tx = tx.clone();
        tokio::spawn(async move {
            while let Some(raw) = source.next().await {
                if let Ok(record) = source.parse(&raw) {
                    let _ = tx.send(record).await;
                }
            }
        });
    }
    drop(tx);

    // รับและประมวลผล records ทีละชิ้น
    while let Some(record) = rx.recv().await {
        if let Some(processed) = pipeline.process(record) {
            buffer.lock().await.push(processed);
        }
    }
}
```

## สรุป

โปรเจคนี้สร้าง **Log Aggregation System** ที่ครบสมบูรณ์ตั้งแต่ ingestion ถึง output โดยใช้ pattern สำคัญ:

### Pattern ที่ได้เรียน

| Pattern | ใช้ที่ | ประโยชน์ |
|---------|--------|----------|
| **Trait objects** | `Box<dyn Filter>`, `Box<dyn Transform>` | ผสม types ต่างกันใน Vec เดียว |
| **Builder pattern** | `LogQuery::new().service("api").limit(50)` | API ที่อ่านง่ายและ type-safe |
| **VecDeque ring buffer** | `LogBuffer` | O(1) drop-oldest เมื่อ capacity เต็ม |
| **Owned transform** | `Transform::apply(record: LogRecord)` | ไม่ต้องการ lifetime, clone เฉพาะที่จำเป็น |
| **HashMap aggregation** | `count_by`, `top_k` | Aggregation ด้วย single-pass O(n) |
| **Dynamic dispatch** | `serde_json::Value` → `Value` | รองรับ schema ที่ไม่รู้ล่วงหน้า |

### เชื่อมต่อกับโปรเจคถัดไป

Log Aggregation เป็นพื้นฐานสำคัญของ observability stack ใน **G04: Metrics Collector** เราจะต่อยอดด้วยการเก็บ numeric metrics (gauges, counters, histograms) ที่ทำงานคู่กับ log aggregation — เมื่อ log บอกว่า "เกิดอะไรขึ้น" metrics จะบอกว่า "เกิดบ่อยแค่ไหนและมีผลกระทบมากเพียงใด"

แนวคิดที่ใช้ต่อ: trait-based pipeline, ring buffer สำหรับ time-series data, aggregation functions, และ Prometheus-compatible output format

---

**โปรเจคก่อนหน้า:** [Project G02: CI Runner](project-g02-ci-runner.md) | **โปรเจคถัดไป:** [Project G04: Metrics Collector](project-g04-metrics-collector.md)
