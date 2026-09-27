# Project C01: ETL Pipeline Framework

> โมดูล: C — Data Processing & Pipelines | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 8 ชั่วโมง

## ภาพรวมโปรเจค

ETL (Extract, Transform, Load) Pipeline คือหัวใจของระบบ data engineering ทุกองค์กรที่ทำงานกับข้อมูลจำนวนมาก ไม่ว่าจะเป็นการย้ายข้อมูลจาก CSV ไปยังฐานข้อมูล, ทำความสะอาดข้อมูล, หรือรวมข้อมูลจากหลายแหล่ง

โปรเจคนี้จะสร้าง **ETL Pipeline Framework** ที่:
- อ่านข้อมูลจาก CSV, JSON (และ PostgreSQL เป็น optional)
- แปลงข้อมูลด้วย transform ที่ compose กันได้ (filter, rename, type cast, dedup)
- เขียนผลลัพธ์ไปยัง CSV, JSON
- กำหนด pipeline ผ่าน YAML config
- แสดง progress bar แบบ real-time ด้วย `indicatif`
- จัดการ error ได้ 3 โหมด: fail fast, skip row, write to error file

**Use case จริงในโลก production:**
- Data migration: ย้ายข้อมูลระหว่าง systems ที่มี schema ต่างกัน
- Data cleaning: กรองข้อมูลเสีย, เติม null, แปลงประเภทข้อมูล
- ETL jobs: นำเข้าข้อมูลจาก external sources เข้า data warehouse
- Data quality checks: ตรวจ schema validation บน ingest

**Learning value:**
เรียนรู้การออกแบบ trait-based abstraction ระดับ production พร้อม builder pattern, error handling ที่ configurable, และ parallel processing ด้วย `rayon`

## สิ่งที่จะได้เรียนรู้

- **Builder pattern** สำหรับ fluent API ที่อ่านง่ายและ type-safe
- **Trait objects** (`Box<dyn Source>`, `Box<dyn Transform>`, `Box<dyn Sink>`) — dynamic dispatch ในโลกจริง
- **Type coercion และ Schema validation** — แปลงข้อมูลระหว่าง types อย่างปลอดภัย
- **Configurable error handling** — `thiserror` + enum policy แทนที่ boolean flags
- **Progress reporting** ด้วย `indicatif` — UX ที่ดีสำหรับ long-running jobs
- **YAML-driven configuration** ด้วย `serde_yaml` — pipeline as code
- **Parallel processing** ด้วย `rayon` สำหรับ embarrassingly parallel transforms
- **Separation of concerns** — source/transform/sink แยกจากกันอย่างชัดเจน

## ความรู้ที่ต้องมีมาก่อน

- **Part 15-20**: Traits และ trait objects, `Box<dyn Trait>`
- **Part 25-30**: Error handling ด้วย `Result`, `?` operator, `thiserror`
- **Part 35-40**: Closures, `Fn`/`FnMut`/`FnOnce`, และ lifetimes
- **Part 46**: async/await และ `tokio::task::spawn_blocking`
- **Part 55-60**: `serde` ecosystem — serialize/deserialize JSON และ YAML
- **Part 96-100**: Performance — `rayon` parallel iterators, profiling
- **Part 105**: Design patterns — builder, strategy, chain of responsibility

## โครงสร้างโปรเจค (Project Layout)

```
etl-pipeline/
├── src/
│   ├── lib.rs           # re-export modules สำหรับ integration tests
│   ├── main.rs          # CLI entry point + config builder
│   ├── types.rs         # Record, Value, Schema, FieldDef
│   ├── error.rs         # PipelineError enum + ErrorPolicy
│   ├── source.rs        # Source trait + CsvSource, JsonSource
│   ├── transform.rs     # Transform trait + Filter, Map, Rename, TypeCast, Dedup
│   ├── sink.rs          # Sink trait + CsvSink, JsonSink, ErrorSink
│   ├── pipeline.rs      # Pipeline builder + runner + PipelineStats
│   └── config.rs        # YAML config structs + parser
├── tests/
│   └── integration_test.rs
├── Cargo.toml
├── sample_input.csv     # ตัวอย่างข้อมูล input
└── sample_pipeline.yaml # ตัวอย่าง config
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
[Source] ──read()──▶ [Record] ──▶ [Schema::coerce] ──▶ [Transform₁] ──▶ [Transform₂] ──▶ ... ──▶ [Sink]
                                        │                     │
                                   validation              filter/map
                                    error?                  error?
                                        │                     │
                                  [ErrorPolicy]         [ErrorPolicy]
                                 fail/skip/log         fail/skip/log
```

### Core Abstractions

สามชั้นหลักแยกจากกันอย่างชัดเจน:

```
trait Source   → อ่านข้อมูลทีละ Record จาก external system
trait Transform → แปลง/กรอง Record (pure function)
trait Sink     → เขียน Record ไปยัง external system
```

**ทำไมถึงเลือก dynamic dispatch แทน generics?**

Pipeline ต้องเก็บ transforms หลายชนิดใน `Vec` เดียวกัน ถ้าใช้ generics จะต้องระบุ type ทุกชั้น (`Pipeline<CsvSource, (Filter, (Rename, ()))>`) ซึ่งซับซ้อนมาก `Box<dyn Transform>` อ่านง่ายกว่าและ overhead ของ vtable เล็กมากเมื่อเทียบกับ I/O ที่ทำจริง

**ทำไม Record ใช้ `HashMap<String, Value>` แทน struct ที่ typed?**

CSV/JSON มี schema ที่เปลี่ยนได้ตอน runtime — ไม่รู้ column names ตอน compile time ดังนั้น dynamic map เหมาะกว่า typed struct แบบ Serde สำหรับ use case นี้

### Builder Pattern

```rust
Pipeline::builder()
    .source(CsvSource::new("input.csv")?)
    .transform(FilterTransform::int_gt("age", 18))
    .transform(RenameTransform::single("city", "location"))
    .sink(JsonSink::new("output.json")?)
    .error_policy(ErrorPolicy::SkipRow)
    .build()?
```

Builder แยก **construction** ออกจาก **execution** — ทำให้ validate config ก่อนรัน และ reuse builder ได้

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Core Types — Record, Value, Schema

เริ่มจาก data model ก่อน — ทุกอย่างในระบบจะใช้ types เหล่านี้

**สร้างโปรเจค:**

```bash
cargo new etl-pipeline
cd etl-pipeline
```

**`Cargo.toml`:**

```toml
[package]
name = "etl-pipeline"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "etl"
path = "src/main.rs"

[lib]
name = "etl_pipeline"
path = "src/lib.rs"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde_yaml = "0.9"
csv = "1"
rayon = "1"
indicatif = "0.17"
chrono = { version = "0.4", features = ["serde"] }
anyhow = "1"
thiserror = "1"
clap = { version = "4", features = ["derive"] }

[dev-dependencies]
tempfile = "3"
```

**`src/types.rs`** — ระบบ type ทั้งหมดอยู่ที่นี่:

```rust
use std::collections::HashMap;
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

/// ค่าข้อมูลทุกประเภทที่ ETL รองรับ
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(untagged)]
pub enum Value {
    String(String),
    Int(i64),
    Float(f64),
    Bool(bool),
    DateTime(DateTime<Utc>),
    Null,
}

impl Value {
    pub fn type_name(&self) -> &'static str {
        match self {
            Value::String(_) => "string",
            Value::Int(_) => "int",
            Value::Float(_) => "float",
            Value::Bool(_) => "bool",
            Value::DateTime(_) => "datetime",
            Value::Null => "null",
        }
    }

    pub fn as_str(&self) -> Option<&str> {
        match self { Value::String(s) => Some(s.as_str()), _ => None }
    }

    pub fn as_int(&self) -> Option<i64> {
        match self { Value::Int(i) => Some(*i), _ => None }
    }

    pub fn as_float(&self) -> Option<f64> {
        match self {
            Value::Float(f) => Some(*f),
            Value::Int(i) => Some(*i as f64),
            _ => None,
        }
    }

    pub fn is_null(&self) -> bool {
        matches!(self, Value::Null)
    }
}

impl std::fmt::Display for Value {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Value::String(s) => write!(f, "{}", s),
            Value::Int(i) => write!(f, "{}", i),
            Value::Float(fl) => write!(f, "{}", fl),
            Value::Bool(b) => write!(f, "{}", b),
            Value::DateTime(dt) => write!(f, "{}", dt.to_rfc3339()),
            Value::Null => write!(f, ""),
        }
    }
}

/// Record คือ 1 แถวข้อมูล — map ชื่อ field → ค่า
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct Record {
    pub fields: HashMap<String, Value>,
}

impl Record {
    pub fn new() -> Self {
        Self { fields: HashMap::new() }
    }

    pub fn with_fields(fields: HashMap<String, Value>) -> Self {
        Self { fields }
    }

    pub fn get(&self, key: &str) -> Option<&Value> {
        self.fields.get(key)
    }

    pub fn set(&mut self, key: impl Into<String>, value: Value) {
        self.fields.insert(key.into(), value);
    }

    pub fn remove(&mut self, key: &str) -> Option<Value> {
        self.fields.remove(key)
    }

    pub fn rename(&mut self, old_key: &str, new_key: &str) {
        if let Some(v) = self.fields.remove(old_key) {
            self.fields.insert(new_key.to_string(), v);
        }
    }
}

/// ประเภทข้อมูลที่รองรับใน Schema
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum FieldType {
    String,
    Int,
    Float,
    Bool,
    DateTime,
}

/// นิยาม field หนึ่งใน Schema
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FieldDef {
    pub name: String,
    #[serde(rename = "type")]
    pub field_type: FieldType,
    #[serde(default)]
    pub nullable: bool,
}

impl FieldDef {
    pub fn new(name: impl Into<String>, field_type: FieldType) -> Self {
        Self { name: name.into(), field_type, nullable: true }
    }

    pub fn required(name: impl Into<String>, field_type: FieldType) -> Self {
        Self { name: name.into(), field_type, nullable: false }
    }
}

/// Schema ของ dataset — กำหนด fields และ types
#[derive(Debug, Clone, Default, Serialize, Deserialize)]
pub struct Schema {
    pub fields: Vec<FieldDef>,
}

impl Schema {
    pub fn new() -> Self {
        Self { fields: Vec::new() }
    }

    pub fn add_field(mut self, field: FieldDef) -> Self {
        self.fields.push(field);
        self
    }

    /// Coerce ค่าใน Record ให้ตรงกับ Schema type
    pub fn coerce(&self, mut record: Record) -> Result<Record, String> {
        for field_def in &self.fields {
            let val = record.fields.get(&field_def.name).cloned();
            match val {
                None => {
                    if !field_def.nullable {
                        return Err(format!("required field '{}' is missing",
                            field_def.name));
                    }
                    record.fields.insert(field_def.name.clone(), Value::Null);
                }
                Some(Value::Null) if !field_def.nullable => {
                    return Err(format!("required field '{}' is null",
                        field_def.name));
                }
                Some(Value::String(s)) => {
                    let coerced = coerce_string(&s, &field_def.field_type)
                        .map_err(|e| format!("field '{}': {}", field_def.name, e))?;
                    record.fields.insert(field_def.name.clone(), coerced);
                }
                _ => {} // ค่าที่ type ตรงแล้ว ไม่ต้องทำอะไร
            }
        }
        Ok(record)
    }
}

fn coerce_string(s: &str, target: &FieldType) -> Result<Value, String> {
    match target {
        FieldType::String => Ok(Value::String(s.to_string())),
        FieldType::Int => s.trim().parse::<i64>()
            .map(Value::Int)
            .map_err(|_| format!("cannot parse '{}' as int", s)),
        FieldType::Float => s.trim().parse::<f64>()
            .map(Value::Float)
            .map_err(|_| format!("cannot parse '{}' as float", s)),
        FieldType::Bool => match s.trim().to_lowercase().as_str() {
            "true" | "1" | "yes" => Ok(Value::Bool(true)),
            "false" | "0" | "no" => Ok(Value::Bool(false)),
            _ => Err(format!("cannot parse '{}' as bool", s)),
        },
        FieldType::DateTime => s.parse::<DateTime<Utc>>()
            .map(Value::DateTime)
            .map_err(|_| format!("cannot parse '{}' as datetime", s)),
    }
}
```

**⚠️ Pitfall #1: `Value::Null` กับ empty string ต่างกัน**

CSV ที่มี field ว่างจะอ่านเป็น `Value::String("")` ไม่ใช่ `Value::Null` — ต้องระวังเมื่อใช้ `FilterTransform::not_null()` เพราะมันกรองเฉพาะ `Value::Null` ถ้าต้องการกรอง empty string ด้วยต้องเขียน filter ต่างหาก:

```rust
// กรองทั้ง null และ empty string
FilterTransform::new("email not empty", |r| {
    match r.get("email") {
        Some(Value::String(s)) => !s.is_empty(),
        Some(Value::Null) | None => false,
        _ => true,
    }
})
```

**Unit tests สำหรับ types:**

```rust
#[test]
fn test_schema_coerce_string_to_int() {
    let schema = Schema::new()
        .add_field(FieldDef::new("count", FieldType::Int));
    let mut r = Record::new();
    r.set("count", Value::String("42".into()));
    let coerced = schema.coerce(r).unwrap();
    assert_eq!(coerced.get("count"), Some(&Value::Int(42)));
}

#[test]
fn test_schema_required_field_missing() {
    let schema = Schema::new()
        .add_field(FieldDef::required("id", FieldType::Int));
    let r = Record::new();
    assert!(schema.coerce(r).is_err());
}
```

---

### ขั้นที่ 2: Error Handling — PipelineError และ ErrorPolicy

`src/error.rs` — ใช้ `thiserror` สร้าง error types ที่มีความหมาย:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum PipelineError {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("CSV error: {0}")]
    Csv(#[from] csv::Error),

    #[error("JSON error: {0}")]
    Json(#[from] serde_json::Error),

    #[error("YAML error: {0}")]
    Yaml(#[from] serde_yaml::Error),

    #[error("Schema validation error on row {row}: {message}")]
    SchemaValidation { row: usize, message: String },

    #[error("Transform error on row {row}: {message}")]
    TransformError { row: usize, message: String },

    #[error("Source error: {0}")]
    SourceError(String),

    #[error("Sink error: {0}")]
    SinkError(String),

    #[error("Config error: {0}")]
    ConfigError(String),
}

/// นโยบายเมื่อเกิด error ระหว่าง pipeline
#[derive(Debug, Clone, PartialEq)]
pub enum ErrorPolicy {
    /// หยุดทันทีเมื่อพบ error แรก
    FailFast,
    /// ข้ามแถวที่มี error แล้วทำต่อ
    SkipRow,
    /// บันทึก error แถวลงไฟล์แล้วทำต่อ
    WriteToErrorFile(String),
}

impl Default for ErrorPolicy {
    fn default() -> Self { ErrorPolicy::FailFast }
}

pub type Result<T> = std::result::Result<T, PipelineError>;
```

**⚠️ Pitfall #2: `#[from]` บน variants ที่ไม่ได้ implement `From`**

ถ้าใส่ `#[from]` บน variant ที่ error type นั้นไม่ได้ implement `std::error::Error` จะ compile ไม่ผ่าน ต้องใช้ `#[source]` แทน หรือ convert manually:

```rust
// ผิด — serde_yaml::Error implement Error แล้ว แต่ถ้าเอา type ที่ไม่ implement
#[error("...")]
MyError(#[from] SomeCustomType), // compile error ถ้า SomeCustomType ไม่ implement Error

// ถูก — ใช้ String แทน
#[error("Custom error: {0}")]
Custom(String),
```

---

### ขั้นที่ 3: Source Trait และ Implementations

**`src/source.rs`** — trait หลักและ implementations:

```rust
use std::fs::File;
use std::io::{BufRead, BufReader};
use std::collections::HashMap;
use crate::types::{Record, Schema, Value, FieldDef, FieldType};
use crate::error::{PipelineError, Result};

/// trait หลักสำหรับ data source — อ่านข้อมูลทีละ Record
pub trait Source: Send {
    fn read(&mut self) -> Option<Record>;
    fn schema(&self) -> Schema;
    /// จำนวน record โดยประมาณ (สำหรับ progress bar)
    fn estimated_count(&self) -> Option<usize> { None }
}

pub struct CsvSource {
    reader: csv::Reader<File>,
    headers: Vec<String>,
    schema: Schema,
    row_count: usize,
}

impl CsvSource {
    pub fn new(path: &str) -> Result<Self> {
        let file = File::open(path)
            .map_err(|e| PipelineError::SourceError(
                format!("cannot open {}: {}", path, e)
            ))?;
        let mut reader = csv::ReaderBuilder::new()
            .has_headers(true)
            .flexible(true)
            .from_reader(file);

        let headers: Vec<String> = reader
            .headers()
            .map_err(PipelineError::Csv)?
            .iter()
            .map(|s| s.to_string())
            .collect();

        // inferred schema — ทุก field เป็น String ก่อน
        let schema = Schema {
            fields: headers.iter()
                .map(|h| FieldDef::new(h.clone(), FieldType::String))
                .collect(),
        };

        Ok(Self { reader, headers, schema, row_count: 0 })
    }

    pub fn with_schema(mut self, schema: Schema) -> Self {
        self.schema = schema;
        self
    }
}

impl Source for CsvSource {
    fn read(&mut self) -> Option<Record> {
        let mut record = csv::StringRecord::new();
        match self.reader.read_record(&mut record) {
            Ok(true) => {
                self.row_count += 1;
                let mut fields = HashMap::new();
                for (i, header) in self.headers.iter().enumerate() {
                    let val = record.get(i).unwrap_or("").to_string();
                    fields.insert(header.clone(), Value::String(val));
                }
                Some(Record::with_fields(fields))
            }
            _ => None,
        }
    }

    fn schema(&self) -> Schema { self.schema.clone() }
}
```

**JsonSource** อ่าน newline-delimited JSON (NDJSON):

```rust
pub struct JsonSource {
    reader: BufReader<File>,
    schema: Schema,
    row_count: usize,
}

impl JsonSource {
    pub fn new(path: &str) -> Result<Self> {
        let file = File::open(path)
            .map_err(|e| PipelineError::SourceError(
                format!("cannot open {}: {}", path, e)
            ))?;
        Ok(Self {
            reader: BufReader::new(file),
            schema: Schema::new(),
            row_count: 0,
        })
    }
}

impl Source for JsonSource {
    fn read(&mut self) -> Option<Record> {
        let mut line = String::new();
        loop {
            line.clear();
            match self.reader.read_line(&mut line) {
                Ok(0) => return None, // EOF
                Ok(_) => {
                    let trimmed = line.trim();
                    if trimmed.is_empty() { continue; }
                    if let Ok(map) = serde_json::from_str::<
                        HashMap<String, serde_json::Value>
                    >(trimmed) {
                        self.row_count += 1;
                        let fields = map.into_iter()
                            .map(|(k, v)| (k, json_value_to_value(v)))
                            .collect();
                        return Some(Record::with_fields(fields));
                    }
                }
                Err(_) => return None,
            }
        }
    }

    fn schema(&self) -> Schema { self.schema.clone() }
}

fn json_value_to_value(v: serde_json::Value) -> Value {
    match v {
        serde_json::Value::Null => Value::Null,
        serde_json::Value::Bool(b) => Value::Bool(b),
        serde_json::Value::Number(n) => {
            if let Some(i) = n.as_i64() { Value::Int(i) }
            else if let Some(f) = n.as_f64() { Value::Float(f) }
            else { Value::String(n.to_string()) }
        }
        serde_json::Value::String(s) => Value::String(s),
        other => Value::String(other.to_string()),
    }
}
```

**PostgresSource** (โค้ดโครงสร้าง สำหรับผู้ที่ต้องการต่อยอด):

```rust
// ต้องเพิ่ม sqlx = { version = "0.7", features = ["postgres", "runtime-tokio"] }
// ใน Cargo.toml
//
// use sqlx::PgPool;
//
// pub struct PostgresSource {
//     pool: PgPool,
//     query: String,
//     rows: Option<Vec<sqlx::postgres::PgRow>>,
//     cursor: usize,
//     schema: Schema,
// }
//
// impl PostgresSource {
//     pub async fn new(url: &str, query: &str, schema: Schema)
//         -> Result<Self>
//     {
//         let pool = PgPool::connect(url).await
//             .map_err(|e| PipelineError::SourceError(e.to_string()))?;
//         // pre-fetch ทั้งหมดเก็บใน Vec (เหมาะกับ query ขนาดกลาง)
//         // สำหรับ dataset ขนาดใหญ่ ควรใช้ cursor streaming
//         Ok(Self { pool, query: query.into(), rows: None,
//                   cursor: 0, schema })
//     }
// }
```

---

### ขั้นที่ 4: Transform Trait และ Implementations

**`src/transform.rs`** — 5 transforms หลัก:

```rust
use std::collections::{HashMap, HashSet};
use crate::types::{Record, Value, FieldType};
use crate::error::PipelineError;

/// คืน None = ข้ามแถวนี้ (filter out)
pub trait Transform: Send + Sync {
    fn transform(&self, record: Record)
        -> Result<Option<Record>, PipelineError>;
    fn name(&self) -> &'static str;
}
```

**FilterTransform** — กรองด้วย predicate closure:

```rust
pub struct FilterTransform {
    predicate: Box<dyn Fn(&Record) -> bool + Send + Sync>,
    description: String,
}

impl FilterTransform {
    pub fn new<F>(description: impl Into<String>, f: F) -> Self
    where
        F: Fn(&Record) -> bool + Send + Sync + 'static,
    {
        Self { predicate: Box::new(f), description: description.into() }
    }

    pub fn field_equals(field: impl Into<String>, value: Value) -> Self {
        let field = field.into();
        Self::new(format!("{}=={}", field, value), move |r| {
            r.get(&field) == Some(&value)
        })
    }

    pub fn not_null(field: impl Into<String>) -> Self {
        let field = field.into();
        Self::new(format!("{} is not null", field), move |r| {
            !matches!(r.get(&field), Some(Value::Null) | None)
        })
    }

    pub fn int_gt(field: impl Into<String>, threshold: i64) -> Self {
        let field = field.into();
        Self::new(format!("{} > {}", field, threshold), move |r| {
            r.get(&field)
                .and_then(|v| v.as_int())
                .map_or(false, |i| i > threshold)
        })
    }
}
```

**RenameTransform:**

```rust
pub struct RenameTransform {
    mappings: HashMap<String, String>,
}

impl RenameTransform {
    pub fn new(mappings: HashMap<String, String>) -> Self {
        Self { mappings }
    }

    pub fn single(from: impl Into<String>, to: impl Into<String>) -> Self {
        let mut m = HashMap::new();
        m.insert(from.into(), to.into());
        Self::new(m)
    }
}

impl Transform for RenameTransform {
    fn transform(&self, mut record: Record)
        -> Result<Option<Record>, PipelineError>
    {
        for (old, new) in &self.mappings {
            record.rename(old, new);
        }
        Ok(Some(record))
    }
    fn name(&self) -> &'static str { "Rename" }
}
```

**TypeCastTransform:**

```rust
pub struct TypeCastTransform {
    casts: Vec<(String, FieldType)>,
}

impl TypeCastTransform {
    pub fn single(field: impl Into<String>, target: FieldType) -> Self {
        Self { casts: vec![(field.into(), target)] }
    }
}

impl Transform for TypeCastTransform {
    fn transform(&self, mut record: Record)
        -> Result<Option<Record>, PipelineError>
    {
        for (field, target_type) in &self.casts {
            if let Some(val) = record.fields.get(field).cloned() {
                let s = val.to_string();
                let new_val = cast_value(&s, target_type).map_err(|e| {
                    PipelineError::TransformError { row: 0, message: e }
                })?;
                record.fields.insert(field.clone(), new_val);
            }
        }
        Ok(Some(record))
    }
    fn name(&self) -> &'static str { "TypeCast" }
}
```

**DeduplicateTransform** — ใช้ `Mutex<HashSet>` เพื่อ thread-safe state:

```rust
pub struct DeduplicateTransform {
    key_fields: Vec<String>,
    seen: std::sync::Mutex<HashSet<String>>,
}

impl DeduplicateTransform {
    pub fn new(key_fields: Vec<String>) -> Self {
        Self {
            key_fields,
            seen: std::sync::Mutex::new(HashSet::new()),
        }
    }

    pub fn on_field(field: impl Into<String>) -> Self {
        Self::new(vec![field.into()])
    }

    fn make_key(&self, record: &Record) -> String {
        self.key_fields.iter()
            .map(|f| record.get(f)
                .map_or("".to_string(), |v| v.to_string()))
            .collect::<Vec<_>>()
            .join("|")
    }
}

impl Transform for DeduplicateTransform {
    fn transform(&self, record: Record)
        -> Result<Option<Record>, PipelineError>
    {
        let key = self.make_key(&record);
        let mut seen = self.seen.lock().unwrap();
        if seen.contains(&key) {
            Ok(None) // duplicate — ข้าม
        } else {
            seen.insert(key);
            Ok(Some(record))
        }
    }
    fn name(&self) -> &'static str { "Deduplicate" }
}
```

**⚠️ Pitfall #3: `DeduplicateTransform` ใน parallel context**

`DeduplicateTransform` ใช้ `Mutex<HashSet>` เพราะ `Transform` trait ต้อง `Sync` สำหรับ parallel execution ถ้าลืม `Mutex` แล้วใช้ `RefCell` หรือ `Cell` จะ compile error เมื่อใส่ใน `Vec<Box<dyn Transform>>` และรัน parallel:

```rust
// ผิด — RefCell ไม่ Sync
pub struct BrokenDedup {
    seen: RefCell<HashSet<String>>, // RefCell: !Sync
}
// error: `RefCell<HashSet<String>>` cannot be shared between threads safely

// ถูก
pub struct DeduplicateTransform {
    seen: std::sync::Mutex<HashSet<String>>, // Mutex: Sync
}
```

---

### ขั้นที่ 5: Sink Trait และ Implementations

**`src/sink.rs`:**

```rust
use std::fs::File;
use std::io::{BufWriter, Write};
use crate::types::{Record, Value};
use crate::error::{PipelineError, Result};

pub trait Sink: Send {
    fn write(&mut self, record: Record) -> Result<()>;
    fn flush(&mut self) -> Result<()>;
    fn records_written(&self) -> usize;
}

pub struct CsvSink {
    writer: csv::Writer<File>,
    headers: Vec<String>,
    headers_written: bool,
    count: usize,
}

impl CsvSink {
    pub fn new(path: &str) -> Result<Self> {
        let file = File::create(path)
            .map_err(|e| PipelineError::SinkError(
                format!("cannot create {}: {}", path, e)
            ))?;
        Ok(Self {
            writer: csv::WriterBuilder::new().from_writer(file),
            headers: Vec::new(),
            headers_written: false,
            count: 0,
        })
    }

    pub fn with_headers(mut self, headers: Vec<String>) -> Self {
        self.headers = headers;
        self
    }
}

impl Sink for CsvSink {
    fn write(&mut self, record: Record) -> Result<()> {
        // เขียน headers จาก record แรกถ้ายังไม่มี
        if !self.headers_written {
            if self.headers.is_empty() {
                self.headers = {
                    let mut ks: Vec<String> =
                        record.fields.keys().cloned().collect();
                    ks.sort(); // consistent ordering
                    ks
                };
            }
            self.writer.write_record(&self.headers)
                .map_err(PipelineError::Csv)?;
            self.headers_written = true;
        }
        let row: Vec<String> = self.headers.iter()
            .map(|h| record.get(h)
                .map_or("".to_string(), |v| v.to_string()))
            .collect();
        self.writer.write_record(&row).map_err(PipelineError::Csv)?;
        self.count += 1;
        Ok(())
    }

    fn flush(&mut self) -> Result<()> {
        self.writer.flush().map_err(|e| PipelineError::Io(e.into()))
    }

    fn records_written(&self) -> usize { self.count }
}

pub struct JsonSink {
    writer: BufWriter<File>,
    count: usize,
}

impl JsonSink {
    pub fn new(path: &str) -> Result<Self> {
        let file = File::create(path)
            .map_err(|e| PipelineError::SinkError(
                format!("cannot create {}: {}", path, e)
            ))?;
        Ok(Self { writer: BufWriter::new(file), count: 0 })
    }
}

impl Sink for JsonSink {
    fn write(&mut self, record: Record) -> Result<()> {
        let json_map: serde_json::Map<String, serde_json::Value> =
            record.fields.into_iter()
                .map(|(k, v)| (k, value_to_json(v)))
                .collect();
        let line = serde_json::to_string(&json_map)
            .map_err(PipelineError::Json)?;
        writeln!(self.writer, "{}", line)
            .map_err(PipelineError::Io)?;
        self.count += 1;
        Ok(())
    }

    fn flush(&mut self) -> Result<()> {
        self.writer.flush().map_err(PipelineError::Io)
    }

    fn records_written(&self) -> usize { self.count }
}

fn value_to_json(v: Value) -> serde_json::Value {
    match v {
        Value::Null => serde_json::Value::Null,
        Value::Bool(b) => serde_json::Value::Bool(b),
        Value::Int(i) => serde_json::Value::Number(i.into()),
        Value::Float(f) => serde_json::Number::from_f64(f)
            .map(serde_json::Value::Number)
            .unwrap_or(serde_json::Value::Null),
        Value::String(s) => serde_json::Value::String(s),
        Value::DateTime(dt) => serde_json::Value::String(dt.to_rfc3339()),
    }
}
```

**PostgresSink** (โครงสร้างสำหรับต่อยอด):

```rust
// pub struct PostgresSink {
//     pool: PgPool,
//     table: String,
//     columns: Vec<String>,
//     batch: Vec<Record>,
//     batch_size: usize,
//     count: usize,
// }
//
// impl PostgresSink {
//     pub async fn new(url: &str, table: &str,
//                      columns: Vec<String>) -> Result<Self>
//     {
//         let pool = PgPool::connect(url).await
//             .map_err(|e| PipelineError::SinkError(e.to_string()))?;
//         Ok(Self { pool, table: table.into(), columns,
//                   batch: Vec::new(), batch_size: 500, count: 0 })
//     }
//
//     async fn flush_batch(&mut self) -> Result<()> {
//         // INSERT INTO table (col1, col2, ...) VALUES ($1, $2, ...), ...
//         // ใช้ batch insert เพื่อประสิทธิภาพ
//         self.batch.clear();
//         Ok(())
//     }
// }
```

---

### ขั้นที่ 6: Pipeline Builder และ Runner

**`src/pipeline.rs`** — เชื่อมทุกส่วนเข้าด้วยกัน:

```rust
use indicatif::{ProgressBar, ProgressStyle};
use crate::source::Source;
use crate::transform::Transform;
use crate::sink::Sink;
use crate::error::{ErrorPolicy, PipelineError, Result};
use crate::types::Schema;

pub struct Pipeline {
    source: Box<dyn Source>,
    transforms: Vec<Box<dyn Transform>>,
    sink: Box<dyn Sink>,
    schema: Option<Schema>,
    error_policy: ErrorPolicy,
    show_progress: bool,
}

pub struct PipelineBuilder {
    source: Option<Box<dyn Source>>,
    transforms: Vec<Box<dyn Transform>>,
    sink: Option<Box<dyn Sink>>,
    schema: Option<Schema>,
    error_policy: ErrorPolicy,
    show_progress: bool,
}

impl PipelineBuilder {
    pub fn new() -> Self {
        Self {
            source: None, transforms: Vec::new(), sink: None,
            schema: None,
            error_policy: ErrorPolicy::default(),
            show_progress: false,
        }
    }

    pub fn source(mut self, s: impl Source + 'static) -> Self {
        self.source = Some(Box::new(s)); self
    }

    pub fn transform(mut self, t: impl Transform + 'static) -> Self {
        self.transforms.push(Box::new(t)); self
    }

    pub fn sink(mut self, s: impl Sink + 'static) -> Self {
        self.sink = Some(Box::new(s)); self
    }

    pub fn schema(mut self, s: Schema) -> Self {
        self.schema = Some(s); self
    }

    pub fn error_policy(mut self, p: ErrorPolicy) -> Self {
        self.error_policy = p; self
    }

    pub fn show_progress(mut self, v: bool) -> Self {
        self.show_progress = v; self
    }

    pub fn build(self) -> Result<Pipeline> {
        let source = self.source.ok_or_else(|| {
            PipelineError::ConfigError("source not set".into())
        })?;
        let sink = self.sink.ok_or_else(|| {
            PipelineError::ConfigError("sink not set".into())
        })?;
        Ok(Pipeline {
            source, transforms: self.transforms, sink,
            schema: self.schema,
            error_policy: self.error_policy,
            show_progress: self.show_progress,
        })
    }
}

#[derive(Debug, Default)]
pub struct PipelineStats {
    pub rows_read: usize,
    pub rows_written: usize,
    pub rows_skipped: usize,
    pub rows_errored: usize,
}

impl Pipeline {
    pub fn builder() -> PipelineBuilder { PipelineBuilder::new() }

    pub fn run(&mut self) -> Result<PipelineStats> {
        let mut stats = PipelineStats::default();

        let pb = if self.show_progress {
            let est = self.source.estimated_count();
            let bar = if let Some(n) = est {
                ProgressBar::new(n as u64)
            } else {
                ProgressBar::new_spinner()
            };
            bar.set_style(
                ProgressStyle::with_template(
                    "{spinner:.green} [{elapsed_precise}] \
                     {bar:40.cyan/blue} {pos}/{len} rows \
                     ({per_sec}) ETA:{eta} errors:{msg}"
                ).unwrap_or_else(|_| ProgressStyle::default_bar()),
            );
            bar.set_message("0");
            Some(bar)
        } else {
            None
        };

        loop {
            // 1. อ่าน record จาก source
            let raw_record = match self.source.read() {
                Some(r) => r,
                None => break, // EOF
            };
            stats.rows_read += 1;

            // 2. ตรวจสอบ/coerce schema ถ้ากำหนดไว้
            let record = match &self.schema {
                Some(schema) => match schema.coerce(raw_record) {
                    Ok(r) => r,
                    Err(e) => {
                        stats.rows_errored += 1;
                        match &self.error_policy {
                            ErrorPolicy::FailFast => return Err(
                                PipelineError::SchemaValidation {
                                    row: stats.rows_read,
                                    message: e,
                                }
                            ),
                            _ => {
                                if let Some(pb) = &pb {
                                    pb.set_message(
                                        stats.rows_errored.to_string()
                                    );
                                }
                                stats.rows_skipped += 1;
                                continue;
                            }
                        }
                    }
                },
                None => raw_record,
            };

            // 3. รัน transforms ตามลำดับ
            let mut current = Some(record);
            for transform in &self.transforms {
                if let Some(r) = current {
                    match transform.transform(r) {
                        Ok(result) => current = result,
                        Err(e) => {
                            stats.rows_errored += 1;
                            match &self.error_policy {
                                ErrorPolicy::FailFast => return Err(e),
                                _ => {
                                    current = None;
                                    if let Some(pb) = &pb {
                                        pb.set_message(
                                            stats.rows_errored.to_string()
                                        );
                                    }
                                    break;
                                }
                            }
                        }
                    }
                } else {
                    break; // filtered out
                }
            }

            // 4. เขียนลง sink ถ้ายังมี record
            match current {
                Some(r) => {
                    self.sink.write(r)?;
                    stats.rows_written += 1;
                }
                None => { stats.rows_skipped += 1; }
            }

            if let Some(pb) = &pb { pb.inc(1); }
        }

        self.sink.flush()?;
        if let Some(pb) = &pb {
            pb.finish_with_message(
                format!("done, {} errors", stats.rows_errored)
            );
        }
        Ok(stats)
    }
}
```

**Parallel Processing ด้วย Rayon:**

สำหรับงานที่ transform แต่ละ record เป็นอิสระจากกัน (embarrassingly parallel) สามารถใช้ `rayon` เพื่อเร่งความเร็ว:

```rust
use rayon::prelude::*;

/// รัน transforms แบบ parallel บน batch ของ records
/// เหมาะสำหรับ transform ที่ใช้ CPU สูงและไม่มี shared mutable state
pub fn transform_batch_parallel(
    records: Vec<Record>,
    transforms: &[Box<dyn Transform>],
) -> Vec<Option<Record>> {
    records
        .into_par_iter()
        .map(|record| {
            let mut current = Some(record);
            for t in transforms {
                if let Some(r) = current {
                    current = t.transform(r).ok().flatten();
                } else {
                    break;
                }
            }
            current
        })
        .collect()
}
```

สำหรับ I/O-bound operations ให้ใช้ `tokio::task::spawn_blocking`:

```rust
use tokio::task;

/// อ่าน source แบบ async โดยไม่ block event loop
pub async fn read_source_async(
    mut source: impl Source + Send + 'static,
) -> Vec<Record> {
    task::spawn_blocking(move || {
        let mut records = Vec::new();
        while let Some(r) = source.read() {
            records.push(r);
        }
        records
    })
    .await
    .unwrap_or_default()
}
```

---

### ขั้นที่ 7: YAML Config Pipeline

**`src/config.rs`** — config structs ที่ deserialize จาก YAML:

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use crate::error::{PipelineError, Result};

#[derive(Debug, Deserialize, Serialize)]
pub struct PipelineConfig {
    pub name: Option<String>,
    pub source: SourceConfig,
    pub transforms: Vec<TransformConfig>,
    pub sink: SinkConfig,
    #[serde(default)]
    pub error_policy: ErrorPolicyConfig,
    #[serde(default)]
    pub show_progress: bool,
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(tag = "type", rename_all = "lowercase")]
pub enum SourceConfig {
    Csv { path: String, schema: Option<Vec<FieldConfig>> },
    Json { path: String },
}

#[derive(Debug, Deserialize, Serialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum TransformConfig {
    Filter { field: String, op: FilterOp,
             value: Option<serde_yaml::Value> },
    Rename { mappings: HashMap<String, String> },
    TypeCast { field: String, target_type: String },
    Deduplicate { key_fields: Vec<String> },
    AddField { field: String, value: serde_yaml::Value },
    RemoveField { field: String },
}

#[derive(Debug, Deserialize, Serialize, Clone)]
#[serde(rename_all = "snake_case")]
pub enum FilterOp {
    Equals,
    NotNull,
    IntGt,
    IntLt,
}

#[derive(Debug, Deserialize, Serialize, Default)]
#[serde(rename_all = "snake_case")]
pub enum ErrorPolicyConfig {
    #[default]
    FailFast,
    SkipRow,
    WriteToErrorFile { path: String },
}

impl PipelineConfig {
    pub fn from_yaml_file(path: &str) -> Result<Self> {
        let content = std::fs::read_to_string(path)
            .map_err(|e| PipelineError::ConfigError(
                format!("cannot read {}: {}", path, e)
            ))?;
        serde_yaml::from_str(&content).map_err(PipelineError::Yaml)
    }
}
```

**ตัวอย่าง `pipeline.yaml`:**

```yaml
name: clean_customers
source:
  type: csv
  path: customers.csv
transforms:
  - type: filter
    field: email
    op: not_null
  - type: type_cast
    field: age
    target_type: int
  - type: filter
    field: age
    op: int_gt
    value: 18
  - type: deduplicate
    key_fields: [email]
  - type: rename
    mappings:
      city: location
      fullname: name
  - type: remove_field
    field: password_hash
sink:
  type: json
  path: output/customers_clean.json
error_policy: skip_row
show_progress: true
```

**⚠️ Pitfall #4: serde tag variant ต้อง match กับ `rename_all`**

เมื่อใช้ `#[serde(tag = "type", rename_all = "snake_case")]` บน enum variants YAML ต้องใช้ชื่อ snake_case ตรง เช่น `type_cast` (ไม่ใช่ `typecast` หรือ `TypeCast`):

```yaml
# ผิด
transforms:
  - type: TypeCast   # PascalCase ไม่ถูก
    field: age

# ผิด
  - type: typecast   # lowercase ไม่ถูกสำหรับ snake_case rename
    field: age

# ถูก — snake_case
  - type: type_cast
    field: age
```

ถ้าต้องการ debug ให้ทดสอบ parse YAML string ก่อนใช้จริง:

```rust
let yaml = "type: type_cast\nfield: age\ntarget_type: int\n";
let t: TransformConfig = serde_yaml::from_str(yaml).unwrap();
```

---

## การทดสอบ (Testing)

### Unit Tests

**tests ใน `types.rs`:**

```rust
#[test]
fn test_value_display() {
    assert_eq!(Value::String("hello".into()).to_string(), "hello");
    assert_eq!(Value::Int(42).to_string(), "42");
    assert_eq!(Value::Bool(true).to_string(), "true");
    assert_eq!(Value::Null.to_string(), "");
}

#[test]
fn test_record_operations() {
    let mut r = Record::new();
    r.set("name", Value::String("Alice".into()));
    r.rename("name", "full_name");
    assert!(r.get("name").is_none());
    assert!(r.get("full_name").is_some());
}

#[test]
fn test_schema_coerce_string_to_int() {
    let schema = Schema::new()
        .add_field(FieldDef::new("count", FieldType::Int));
    let mut r = Record::new();
    r.set("count", Value::String("42".into()));
    let coerced = schema.coerce(r).unwrap();
    assert_eq!(coerced.get("count"), Some(&Value::Int(42)));
}

#[test]
fn test_schema_required_field_missing() {
    let schema = Schema::new()
        .add_field(FieldDef::required("id", FieldType::Int));
    assert!(schema.coerce(Record::new()).is_err());
}
```

**tests ใน `transform.rs`:**

```rust
#[test]
fn test_filter_int_gt() {
    let f = FilterTransform::int_gt("age", 18);
    let r_pass = make_record(&[("age", Value::Int(25))]);
    let r_fail = make_record(&[("age", Value::Int(15))]);
    assert!(f.transform(r_pass).unwrap().is_some());
    assert!(f.transform(r_fail).unwrap().is_none());
}

#[test]
fn test_dedup_filters_second_occurrence() {
    let d = DeduplicateTransform::on_field("email");
    let r1 = make_record(&[
        ("email", Value::String("a@x.com".into()))
    ]);
    let r2 = make_record(&[
        ("email", Value::String("a@x.com".into()))
    ]);
    assert!(d.transform(r1).unwrap().is_some());
    assert!(d.transform(r2).unwrap().is_none()); // duplicate
}
```

### Integration Tests

**`tests/integration_test.rs`:**

```rust
use std::fs;
use std::io::Write;
use tempfile::NamedTempFile;

use etl_pipeline::types::{Schema, FieldDef, FieldType};
use etl_pipeline::source::CsvSource;
use etl_pipeline::sink::JsonSink;
use etl_pipeline::transform::{
    FilterTransform, RenameTransform,
    DeduplicateTransform, TypeCastTransform,
};
use etl_pipeline::pipeline::Pipeline;
use etl_pipeline::error::ErrorPolicy;

#[test]
fn test_filter_then_rename_pipeline() {
    let csv = "name,age,city\n\
               Alice,30,Bangkok\n\
               Bob,17,Chiang Mai\n\
               Carol,25,Phuket\n";
    let mut input = NamedTempFile::new().unwrap();
    write!(input, "{}", csv).unwrap();

    let output = NamedTempFile::new().unwrap();
    let out_path = output.path().to_str().unwrap().to_string();
    drop(output);

    let src = CsvSource::new(
        input.path().to_str().unwrap()
    ).unwrap();
    let sink = JsonSink::new(&out_path).unwrap();

    let mut pipeline = Pipeline::builder()
        .source(src)
        .transform(TypeCastTransform::single("age", FieldType::Int))
        .transform(FilterTransform::int_gt("age", 17))
        .transform(RenameTransform::single("city", "location"))
        .sink(sink)
        .build()
        .unwrap();

    let stats = pipeline.run().unwrap();
    assert_eq!(stats.rows_read, 3);
    assert_eq!(stats.rows_written, 2);  // Alice(30), Carol(25) ผ่าน
    assert_eq!(stats.rows_skipped, 1);  // Bob(17) ถูกกรอง

    let content = fs::read_to_string(&out_path).unwrap();
    let first: serde_json::Value =
        serde_json::from_str(content.lines().next().unwrap()).unwrap();
    assert!(first.get("location").is_some());
    assert!(first.get("city").is_none()); // rename ทำงาน

    let _ = fs::remove_file(&out_path);
}

#[test]
fn test_schema_validation_skip_row_policy() {
    let csv = "id,name,age\n1,Alice,30\n2,Bob,invalid\n3,Carol,25\n";
    let mut input = NamedTempFile::new().unwrap();
    write!(input, "{}", csv).unwrap();

    let output = NamedTempFile::new().unwrap();
    let out_path = output.path().to_str().unwrap().to_string();
    drop(output);

    let schema = Schema::new()
        .add_field(FieldDef::new("id", FieldType::Int))
        .add_field(FieldDef::new("name", FieldType::String))
        .add_field(FieldDef::new("age", FieldType::Int));

    let mut pipeline = Pipeline::builder()
        .source(CsvSource::new(
            input.path().to_str().unwrap()
        ).unwrap())
        .schema(schema)
        .sink(JsonSink::new(&out_path).unwrap())
        .error_policy(ErrorPolicy::SkipRow)
        .build()
        .unwrap();

    let stats = pipeline.run().unwrap();
    assert_eq!(stats.rows_read, 3);
    assert_eq!(stats.rows_written, 2); // Alice, Carol เท่านั้น
    assert_eq!(stats.rows_errored, 1); // Bob มี age="invalid"

    let _ = fs::remove_file(&out_path);
}
```

### รัน Tests จริง

```bash
cargo test
```

**Output จริงจากการรัน:**

```
running 18 tests
test config::tests::test_parse_yaml_config ... ok
test sink::tests::test_csv_sink_writes_headers_and_rows ... ok
test sink::tests::test_json_sink_writes_records ... ok
test source::tests::test_csv_source_reads_records ... ok
test transform::tests::test_dedup_filters_second_occurrence ... ok
test transform::tests::test_filter_int_gt ... ok
test transform::tests::test_filter_passes_matching ... ok
test transform::tests::test_filter_rejects_non_matching ... ok
test transform::tests::test_not_null_filter ... ok
test transform::tests::test_rename_transform ... ok
test transform::tests::test_typecast_string_to_int ... ok
test types::tests::test_record_operations ... ok
test types::tests::test_schema_coerce_invalid ... ok
test types::tests::test_schema_coerce_string_to_bool ... ok
test types::tests::test_schema_coerce_string_to_int ... ok
test types::tests::test_schema_required_field_missing ... ok
test types::tests::test_value_display ... ok
test source::tests::test_json_source_reads_records ... ok

test result: ok. 18 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_test.rs

running 4 tests
test test_dedup_pipeline ... ok
test test_empty_source ... ok
test test_filter_then_rename_pipeline ... ok
test test_schema_validation_skip_row_policy ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### รัน Pipeline จริง

สร้างไฟล์ `sample_input.csv`:

```csv
id,name,age,email,city,status
1,Alice Chen,32,alice@example.com,Bangkok,active
2,Bob Smith,17,bob@example.com,Chiang Mai,active
3,Carol Wang,28,,Phuket,inactive
4,David Park,45,david@example.com,Bangkok,active
5,Eve Lopez,22,eve@example.com,Pattaya,active
6,Alice Chen,32,alice@example.com,Bangkok,active
7,Frank Kim,15,frank@example.com,Chiang Rai,active
```

สร้าง `sample_pipeline.yaml`:

```yaml
name: sample_etl_pipeline
source:
  type: csv
  path: sample_input.csv
transforms:
  - type: filter
    field: email
    op: not_null
  - type: type_cast
    field: age
    target_type: int
  - type: filter
    field: age
    op: int_gt
    value: 18
  - type: deduplicate
    key_fields: [email]
  - type: rename
    mappings:
      city: location
  - type: remove_field
    field: status
sink:
  type: json
  path: sample_output.json
error_policy: skip_row
show_progress: false
```

```bash
cargo run --bin etl -- --pipeline sample_pipeline.yaml
```

**Output จริง:**

```
Pipeline complete:
  Rows read:    7
  Rows written: 4
  Rows skipped: 3
  Rows errored: 0
```

**`sample_output.json` ที่ได้:**

```json
{"age":32,"email":"alice@example.com","id":"1","location":"Bangkok","name":"Alice Chen"}
{"age":28,"email":"","id":"3","location":"Phuket","name":"Carol Wang"}
{"age":45,"email":"david@example.com","id":"4","location":"Bangkok","name":"David Park"}
{"age":22,"email":"eve@example.com","id":"5","location":"Pattaya","name":"Eve Lopez"}
```

**วิเคราะห์ผลลัพธ์:**
- **Bob (row 2)**: อายุ 17 — ถูกกรองโดย `int_gt 18`
- **Alice duplicate (row 6)**: email เหมือน row 1 — ถูก deduplicate
- **Frank (row 7)**: อายุ 15 — ถูกกรองโดย `int_gt 18`
- Carol (row 3) ผ่านเพราะ email ว่างเป็น `""` ไม่ใช่ `null` (ดู Pitfall #1)
- field `status` ถูกลบโดย `remove_field`
- field `city` ถูก rename เป็น `location`

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
ls -lh target/release/etl
# -rwxr-xr-x 1 user user 8.2M Sep 27 12:00 target/release/etl
```

### ใช้เป็น Library

เพิ่มใน `Cargo.toml` ของโปรเจคอื่น:

```toml
[dependencies]
etl-pipeline = { path = "../etl-pipeline" }
# หรือจาก git
# etl-pipeline = { git = "https://github.com/..." }
```

### Docker

```dockerfile
FROM rust:1.82-slim as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/etl /usr/local/bin/etl
ENTRYPOINT ["etl"]
```

```bash
docker build -t etl-pipeline .
docker run -v $(pwd)/data:/data etl-pipeline \
    --pipeline /data/pipeline.yaml
```

### CI Integration

เรียกใช้ใน CI pipeline:

```yaml
# .github/workflows/etl.yml
- name: Run ETL Pipeline
  run: |
    etl --pipeline pipeline.yaml
    echo "Processed records: $(wc -l < output.json)"
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม StringTransform

สร้าง `StringTransform` ที่รองรับ operations บน string values:

```rust
pub enum StringOp {
    Trim,
    Lowercase,
    Uppercase,
    Replace { from: String, to: String },
    Truncate { max_len: usize },
}

pub struct StringTransform {
    field: String,
    op: StringOp,
}
```

ตัวอย่างการใช้:
```rust
Pipeline::builder()
    .transform(StringTransform::new("name", StringOp::Trim))
    .transform(StringTransform::new("email", StringOp::Lowercase))
```

### แบบฝึกหัดที่ 2: Batch Processing ด้วย Rayon

เปลี่ยน `Pipeline::run()` ให้รองรับ batch mode — อ่าน N records พร้อมกัน แล้วรัน transforms แบบ parallel ด้วย `rayon`:

```rust
pub struct BatchPipeline {
    inner: Pipeline,
    batch_size: usize, // default: 1000
}

impl BatchPipeline {
    pub fn run(&mut self) -> Result<PipelineStats> {
        // อ่าน batch_size records
        // รัน transforms ด้วย par_iter()
        // เขียน sink แบบ sequential (sink ส่วนมากไม่ thread-safe)
    }
}
```

**Hint:** ระวัง `DeduplicateTransform` ใน parallel mode — state ของมันเป็น global ถ้าต้องการ deterministic dedup ใน parallel ต้องรัน dedup เป็น sequential step หลัง parallel transforms

### แบบฝึกหัดที่ 3: PostgreSQL Source และ Sink

เพิ่ม `PostgresSource` และ `PostgresSink` ด้วย `sqlx`:

```toml
sqlx = { version = "0.7", features = [
    "postgres", "runtime-tokio", "chrono"
] }
```

ทำ async pipeline ที่อ่านจาก PostgreSQL table หนึ่ง แปลงข้อมูล และเขียนลง table อื่น:

```rust
let src = PostgresSource::new(
    "postgres://user:pass@localhost/db",
    "SELECT id, name, age FROM users WHERE created_at > $1",
    schema,
).await?;
```

### แบบฝึกหัดที่ 4: Pipeline Validation และ Dry-Run Mode

เพิ่ม `--dry-run` flag ที่รันเฉพาะ transforms แต่ไม่เขียนลง sink จริง พร้อมรายงาน:
- จำนวน records ที่จะ pass แต่ละ step
- ตัวอย่าง 5 records แรกที่จะ output
- Error ที่พบใน 100 records แรก

```bash
etl --pipeline pipeline.yaml --dry-run --sample-size 100
```

Output:
```
=== Dry Run: clean_customers ===
Source: customers.csv (1,234 rows estimated)

Step 1 [filter email not_null]: 1,189 pass / 45 skip
Step 2 [type_cast age→int]: 1,185 pass / 4 error
Step 3 [filter age > 18]: 1,102 pass / 83 skip
Step 4 [deduplicate email]: 1,089 pass / 13 skip

Expected output: ~1,089 records

Sample output (first 3):
{"age":32,"email":"alice@...","name":"Alice Chen"}
...
```

---

## สรุป

โปรเจคนี้สร้าง ETL Pipeline Framework ที่มีความสมบูรณ์สำหรับงาน production โดยใช้ pattern สำคัญที่ Rust นิยมใช้:

1. **Trait-based abstraction** — `Source`, `Transform`, `Sink` เป็น traits ที่ extensible ไม่ผูกกับ implementation ใดๆ
2. **Builder pattern** — Pipeline DSL ที่อ่านง่าย type-safe และ validate ก่อน run
3. **Enum-driven error policy** — ยืดหยุ่นกว่า boolean flags ทั้ง behavior และ documentation
4. **Schema coercion** — แปลง types อย่างปลอดภัยพร้อม error message ที่ชัดเจน
5. **YAML-driven configuration** — pipeline as code ที่แก้ไขได้โดยไม่ต้อง compile
6. **Progress reporting** — UX ที่ดีสำหรับ long-running ETL jobs

**Pattern ที่ควรเอาไปใช้ต่อ:**
- การออกแบบ `Source`/`Transform`/`Sink` เป็นหัวใจของ stream processing ทุกชนิด
- `Box<dyn Trait>` + builder pattern เหมาะมากกับระบบที่ต้องการ configure ตอน runtime
- `Mutex` แทน `RefCell` เมื่อต้องการ shared state ใน multi-thread context

โปรเจคถัดไปจะนำ data processing concepts เหล่านี้ไปใช้กับ **Realtime Analytics** ที่ต้องประมวลผล streaming data จาก websocket และแสดงผลแบบ real-time

---

**โปรเจคก่อนหน้า:** [project-b10-screenshot-service.md](project-b10-screenshot-service.md) | **โปรเจคถัดไป:** [project-c02-realtime-analytics.md](project-c02-realtime-analytics.md)
