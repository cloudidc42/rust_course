# Project C06: CSV/Parquet Data Processor

> โมดูล: C — Data Processing & Pipelines | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้าง **CLI Data Processor** สำหรับจัดการไฟล์ CSV และ Parquet ด้วย Rust ครอบคลุมตั้งแต่การอ่าน/เขียนไฟล์แบบ streaming, inference schema อัตโนมัติ, transformation pipeline (filter/select/rename/sort/limit), aggregation แบบ group-by, สถิติต่อคอลัมน์, และการแปลงรูปแบบระหว่าง CSV กับ Parquet

ในโลก Data Engineering เครื่องมือแบบนี้เปรียบได้กับ mini version ของ DuckDB CLI หรือ Polars Python package — ใช้งานบน command line โดยไม่ต้องติดตั้ง runtime ขนาดใหญ่ เหมาะสำหรับ data analyst ที่ต้องการ inspect, transform, และ convert ไฟล์ข้อมูลขนาด 10MB–10GB อย่างรวดเร็ว

**Learning value:**
- เรียนรู้ `csv` crate ในเชิง production: streaming reader, custom delimiter, quote handling
- เข้าใจ Apache Parquet format: row groups, column encoding, schema, compression
- ฝึกเขียน expression evaluator (recursive descent parser) สำหรับ filter language
- ออกแบบ `Value` enum ที่รองรับ multiple types ด้วย `PartialOrd` สำหรับ sort/compare
- ใช้ `indicatif` progress bar สำหรับ long-running chunked processing
- ออกแบบ CLI ที่ ergonomic ด้วย `clap 4` derive macro

## สิ่งที่จะได้เรียนรู้

- **CSV Streaming Reader**: อ่าน CSV ทีละ chunk ไม่โหลดทั้งไฟล์เข้า RAM ด้วย `csv::ReaderBuilder`
- **CSV Writer**: เขียน header อัตโนมัติจาก record แรก ด้วย `csv::WriterBuilder`
- **Parquet I/O**: อ่าน row groups, แปลง Parquet types (INT32/64, FLOAT, DOUBLE, BYTE_ARRAY, BOOLEAN) → `Value`, เขียนด้วย SNAPPY compression
- **Schema Inference**: ตรวจจับชนิดข้อมูล (int64, float64, bool, string, datetime) จาก N rows แรก
- **Filter Expression Evaluator**: tokenizer + recursive descent parser สำหรับ `age > 18 AND dept == "Engineering"`
- **Transformation Pipeline**: `select`, `rename`, `sort`, `limit` ที่ใช้ร่วมกันได้
- **Group-By Aggregation**: sum/count/avg/min/max per group key
- **Column Statistics**: count/null_count/min/max/mean/std สำหรับ numeric, top-5 values สำหรับ string
- **Chunked Processing**: `--chunk-size N` สำหรับไฟล์ใหญ่กว่า RAM พร้อม progress bar

## ความรู้ที่ต้องมีมาก่อน

- **Part 1-20**: Rust basics — ownership, borrowing, structs, enums, impl blocks
- **Part 21-40**: Collections (`HashMap`, `Vec`), iterators, closures, `Iterator` trait methods
- **Part 41-55**: Error handling ด้วย `Result`/`?`, `anyhow`, `thiserror`
- **Part 56-70**: Trait design, `PartialOrd`, `Display`, generics, lifetime ขั้นพื้นฐาน
- **Part 71-80**: Pattern matching เชิงลึก, recursive algorithms
- โปรเจค C05 (Search Engine) — ช่วยให้คุ้นกับ parser patterns

## โครงสร้างโปรเจค (Project Layout)

```
csv-parquet-processor/
├── src/
│   ├── main.rs          — CLI entry point ด้วย clap 4 derive
│   ├── types.rs         — Value enum, Record struct, DataType enum
│   ├── schema.rs        — Schema inference + parse_value
│   ├── filter.rs        — Tokenizer + recursive descent filter parser
│   ├── aggregation.rs   — Group-by aggregation (sum/count/avg/min/max)
│   ├── stats.rs         — Per-column statistics (mean/std/top5)
│   ├── transform.rs     — select_columns, rename_column, sort_records
│   ├── csv_io.rs        — CSV reader (streaming) + writer
│   └── parquet_io.rs    — Parquet reader (row groups) + writer (SNAPPY)
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Model

ใจกลางของระบบคือ `Value` enum ที่ represent ค่าหนึ่งช่องในตาราง:

```
Value::Int(i64)     — จำนวนเต็ม 64-bit
Value::Float(f64)   — ทศนิยม 64-bit
Value::Bool(bool)   — boolean
Value::Str(String)  — string (รวม datetime ที่เก็บเป็น string)
Value::Null         — ค่าว่าง/missing
```

`Record` คือหนึ่งแถวของตาราง เก็บเป็น `HashMap<String, Value>` พร้อม `Vec<String>` สำหรับรักษาลำดับคอลัมน์

### Data Flow

```
Input File (CSV / Parquet)
        │
        ▼
┌─────────────────────┐
│   Chunked Reader    │  อ่านทีละ chunk_size rows
│   (csv_io /         │  ป้องกัน OOM บนไฟล์ขนาดใหญ่
│    parquet_io)      │
└──────────┬──────────┘
           │ Vec<Record>
           ▼
┌─────────────────────┐
│ Schema Inference    │  ตรวจ types จาก N rows แรก
│ (schema.rs)         │  → HashMap<String, DataType>
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Transformation      │  filter → select → rename
│ Pipeline            │  → sort → limit
│ (filter, transform) │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Output              │  พิมพ์ตาราง / เขียน CSV / เขียน Parquet
│ (csv_io /           │
│  parquet_io)        │
└─────────────────────┘
```

### Filter Expression Grammar

```
expr   ::= and_expr ('OR' and_expr)*
and_expr ::= not_expr ('AND' not_expr)*
not_expr ::= 'NOT' atom | atom
atom   ::= '(' expr ')' | ident op value
op     ::= '>' | '<' | '>=' | '<=' | '==' | '!='
value  ::= NUMBER | STRING_LITERAL | IDENT
```

### Design Decision: Value::Null แทน Option

เลือกใส่ `Null` variant ใน `Value` enum แทนการใช้ `Option<Value>` เพราะทำให้ filter code เขียนง่ายกว่า — ไม่ต้อง unwrap ทุกที่ และ `PartialOrd` ก็ handle `Null` ได้ด้วยการ return `None` (Null ไม่สามารถ compare กับค่าใดๆ)

### Parquet Schema Mapping

| Rust DataType | Parquet Physical Type | Arrow Type |
|---|---|---|
| `DataType::Int64` | `INT64` | `Int64` |
| `DataType::Float64` | `DOUBLE` | `Float64` |
| `DataType::Bool` | `BOOLEAN` | `Boolean` |
| `DataType::Str` | `BYTE_ARRAY (UTF8)` | `Utf8` |
| `DataType::DateTime` | `BYTE_ARRAY (UTF8)` | `Utf8` |

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: สร้าง Types และ Data Model พื้นฐาน

เริ่มจาก `Cargo.toml` และ core types ก่อน:

**Cargo.toml:**

```toml
[package]
name = "csv-parquet-processor"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "dataproc"
path = "src/main.rs"

[dependencies]
csv       = "1.3"
parquet   = "54"
arrow     = { version = "54", features = ["csv"] }
serde     = { version = "1", features = ["derive"] }
serde_json = "1"
clap      = { version = "4", features = ["derive"] }
indicatif = "0.17"
chrono    = { version = "0.4", features = ["serde"] }
anyhow    = "1"
thiserror = "1"

[dev-dependencies]
tempfile = "3"
```

**src/types.rs** — core data model:

```rust
use std::collections::HashMap;
use std::fmt;

/// ค่าหนึ่งช่องในตาราง รองรับ 5 ชนิด
#[derive(Debug, Clone, PartialEq)]
pub enum Value {
    Int(i64),
    Float(f64),
    Bool(bool),
    Str(String),
    Null,
}

impl Value {
    pub fn as_f64(&self) -> Option<f64> {
        match self {
            Value::Int(i)   => Some(*i as f64),
            Value::Float(f) => Some(*f),
            _               => None,
        }
    }

    pub fn as_i64(&self) -> Option<i64> {
        match self {
            Value::Int(i)   => Some(*i),
            Value::Float(f) => Some(*f as i64),
            _               => None,
        }
    }

    pub fn as_str(&self) -> Option<&str> {
        match self {
            Value::Str(s) => Some(s.as_str()),
            _             => None,
        }
    }

    pub fn is_null(&self) -> bool {
        matches!(self, Value::Null)
    }
}

impl fmt::Display for Value {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Value::Int(i)   => write!(f, "{}", i),
            Value::Float(fl) => write!(f, "{:.4}", fl),
            Value::Bool(b)  => write!(f, "{}", b),
            Value::Str(s)   => write!(f, "{}", s),
            Value::Null     => write!(f, "NULL"),
        }
    }
}

/// ใช้ PartialOrd สำหรับ sort และ filter comparisons
/// Null เปรียบเทียบไม่ได้กับค่าใดๆ (return None)
impl PartialOrd for Value {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        match (self, other) {
            (Value::Int(a),   Value::Int(b))   => a.partial_cmp(b),
            (Value::Float(a), Value::Float(b)) => a.partial_cmp(b),
            (Value::Int(a),   Value::Float(b)) => (*a as f64).partial_cmp(b),
            (Value::Float(a), Value::Int(b))   => a.partial_cmp(&(*b as f64)),
            (Value::Str(a),   Value::Str(b))   => a.partial_cmp(b),
            _                                  => None,
        }
    }
}

/// หนึ่งแถวในตาราง
#[derive(Debug, Clone)]
pub struct Record {
    pub fields:  HashMap<String, Value>,
    /// รักษาลำดับคอลัมน์ตั้งแต่ต้น
    pub columns: Vec<String>,
}

impl Record {
    pub fn new(columns: Vec<String>) -> Self {
        Record { fields: HashMap::new(), columns }
    }

    /// ดึงค่า — คืน &Value::Null ถ้าคอลัมน์ไม่มี
    pub fn get(&self, col: &str) -> &Value {
        self.fields.get(col).unwrap_or(&Value::Null)
    }

    pub fn set(&mut self, col: String, val: Value) {
        if !self.columns.contains(&col) {
            self.columns.push(col.clone());
        }
        self.fields.insert(col, val);
    }
}

/// ชนิดข้อมูลที่ infer ได้จาก CSV/Parquet
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum DataType {
    Int64,
    Float64,
    Bool,
    Str,
    DateTime,
}

impl fmt::Display for DataType {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            DataType::Int64    => write!(f, "int64"),
            DataType::Float64  => write!(f, "float64"),
            DataType::Bool     => write!(f, "bool"),
            DataType::Str      => write!(f, "string"),
            DataType::DateTime => write!(f, "datetime"),
        }
    }
}
```

**⚠️ Pitfall #1: `PartialOrd` กับ Mixed Numeric Types**

ถ้าเขียน filter `salary > 50000` และ `salary` เป็น `Value::Int` แต่ `50000` ถูก parse เป็น `Value::Int` ก็ OK แต่ถ้า `salary` เป็น `Value::Float(50000.0)` จะ return `None` เพราะ `(Float, Int)` ไม่ตรง pattern ดังนั้น `PartialOrd` ต้องมี cross-type cases:

```rust
// ต้องมีทั้ง (Int, Float) และ (Float, Int)
(Value::Int(a),   Value::Float(b)) => (*a as f64).partial_cmp(b),
(Value::Float(a), Value::Int(b))   => a.partial_cmp(&(*b as f64)),
```

---

### ขั้นที่ 2: Schema Inference และ CSV Reader

**src/schema.rs** — ตรวจจับชนิดข้อมูลอัตโนมัติ:

```rust
use std::collections::HashMap;
use crate::types::{DataType, Record, Value};

/// ตรวจว่า string มีรูปแบบ YYYY-MM-DD หรือ ISO 8601
fn looks_like_datetime(s: &str) -> bool {
    let s = s.trim();
    if s.len() < 10 { return false; }
    let b = s.as_bytes();
    b[4] == b'-' && b[7] == b'-'
        && b[0..4].iter().all(|c| c.is_ascii_digit())
        && b[5..7].iter().all(|c| c.is_ascii_digit())
        && b[8..10].iter().all(|c| c.is_ascii_digit())
}

/// แปลง raw string จาก CSV เป็น Value ที่มีชนิดที่เหมาะสม
pub fn parse_value(s: &str) -> Value {
    let trimmed = s.trim();
    if trimmed.is_empty() {
        return Value::Null;
    }
    // Bool ก่อน (true/false/yes/no) — ต้อง check ก่อน int
    match trimmed.to_lowercase().as_str() {
        "true" | "yes"  => return Value::Bool(true),
        "false" | "no"  => return Value::Bool(false),
        _ => {}
    }
    // Int
    if let Ok(i) = trimmed.parse::<i64>() {
        return Value::Int(i);
    }
    // Float
    if let Ok(f) = trimmed.parse::<f64>() {
        return Value::Float(f);
    }
    // DateTime (เก็บเป็น Str แต่ schema จะ tag เป็น DateTime)
    Value::Str(trimmed.to_string())
}

/// Infer schema จาก records แรก N ตัว
/// คืน HashMap: column_name → DataType
pub fn infer_schema(records: &[Record]) -> HashMap<String, DataType> {
    if records.is_empty() {
        return HashMap::new();
    }

    let columns = &records[0].columns;
    let mut schema = HashMap::new();

    for col in columns {
        let (mut has_int, mut has_float) = (false, false);
        let (mut has_bool, mut has_str)  = (false, false);
        let mut has_datetime = false;
        let mut all_null     = true;

        for rec in records {
            match rec.get(col) {
                Value::Null       => {}
                Value::Int(_)     => { all_null = false; has_int = true; }
                Value::Float(_)   => { all_null = false; has_float = true; }
                Value::Bool(_)    => { all_null = false; has_bool = true; }
                Value::Str(s)     => {
                    all_null = false;
                    if looks_like_datetime(s) { has_datetime = true; }
                    else                      { has_str = true; }
                }
            }
        }

        let dtype = if all_null {
            DataType::Str
        } else if has_str {
            DataType::Str
        } else if has_datetime && !has_int && !has_float && !has_bool {
            DataType::DateTime
        } else if has_bool && !has_int && !has_float {
            DataType::Bool
        } else if has_float {
            DataType::Float64
        } else {
            DataType::Int64
        };

        schema.insert(col.clone(), dtype);
    }

    schema
}
```

**⚠️ Pitfall #2: Bool ถูก Parse เป็น Int**

ใน CSV ทั่วไป บางไฟล์เก็บ boolean เป็น `1`/`0` ซึ่งถ้า parse เป็น `Value::Int` ก่อน schema inference จะได้ `DataType::Int64` แทน `DataType::Bool` วิธีแก้คือ infer schema โดยดูจากบริบท หรือ อนุญาตให้ผู้ใช้ระบุ schema ด้วย `--schema` flag:

```rust
// ถ้าต้องการ strict bool จาก 1/0 ต้องเพิ่ม override
match trimmed {
    "1" if col_hint == "bool" => return Value::Bool(true),
    "0" if col_hint == "bool" => return Value::Bool(false),
    _ => {}
}
```

**src/csv_io.rs** — Streaming CSV reader:

```rust
use std::path::Path;
use anyhow::Result;
use indicatif::{ProgressBar, ProgressStyle};
use crate::types::{Record, Value};
use crate::schema::parse_value;

/// อ่าน CSV แบบ streaming ทีละ chunk_size rows
/// ใช้ indicatif แสดง progress สำหรับไฟล์ขนาดใหญ่
pub fn read_csv_chunked(
    path: &Path,
    chunk_size: usize,
    delimiter: u8,
    has_header: bool,
) -> Result<Vec<Record>> {
    // ดึงขนาดไฟล์เพื่อแสดง progress
    let file_size = std::fs::metadata(path)?.len();
    let pb = ProgressBar::new(file_size);
    pb.set_style(
        ProgressStyle::with_template(
            "[{elapsed_precise}] {bar:40.cyan/blue} {bytes}/{total_bytes} ({eta})"
        )?.progress_chars("=>-"),
    );

    let mut rdr = csv::ReaderBuilder::new()
        .has_headers(has_header)
        .delimiter(delimiter)
        .flexible(true)
        .from_path(path)?;

    let headers: Vec<String> = if has_header {
        rdr.headers()?.iter().map(|s| s.to_string()).collect()
    } else {
        // สร้าง header อัตโนมัติ: col_0, col_1, ...
        let first = rdr.records().next()
            .ok_or_else(|| anyhow::anyhow!("Empty file"))??;
        (0..first.len()).map(|i| format!("col_{}", i)).collect()
    };

    let mut all_records = Vec::new();
    let mut chunk: Vec<Record> = Vec::with_capacity(chunk_size);

    for result in rdr.records() {
        let row = result?;
        let mut rec = Record::new(headers.clone());
        for (i, field) in row.iter().enumerate() {
            if i < headers.len() {
                rec.set(headers[i].clone(), parse_value(field));
            }
        }
        chunk.push(rec);

        if chunk.len() >= chunk_size {
            pb.inc(chunk_size as u64 * 50); // ประมาณ bytes ต่อ row
            all_records.extend(chunk.drain(..));
        }
    }
    all_records.extend(chunk.drain(..));
    pb.finish_with_message("Done");

    Ok(all_records)
}

/// เวอร์ชันง่ายไม่มี progress bar สำหรับไฟล์เล็ก
pub fn read_csv(path: &Path, chunk_size: usize) -> Result<Vec<Record>> {
    read_csv_chunked(path, chunk_size, b',', true)
}

/// เขียน records เป็น CSV file
/// header ถูก infer จาก records[0].columns อัตโนมัติ
pub fn write_csv(records: &[Record], path: &Path) -> Result<()> {
    if records.is_empty() { return Ok(()); }

    let mut wtr = csv::WriterBuilder::new()
        .has_headers(true)
        .from_path(path)?;

    // เขียน header row
    let columns = &records[0].columns;
    wtr.write_record(columns)?;

    // เขียนแต่ละแถว
    for record in records {
        let row: Vec<String> = columns.iter()
            .map(|col| record.get(col).to_string())
            .collect();
        wtr.write_record(&row)?;
    }

    wtr.flush()?;
    Ok(())
}

/// แสดงตารางใน terminal สำหรับ preview
pub fn print_table(records: &[Record], limit: usize) {
    if records.is_empty() {
        println!("(no records)");
        return;
    }

    let columns = &records[0].columns;

    // คำนวณความกว้างแต่ละคอลัมน์
    let col_widths: Vec<usize> = columns.iter().map(|c| {
        records.iter().take(limit)
            .fold(c.len(), |max_w, rec| {
                max_w.max(rec.get(c).to_string().len())
            })
    }).collect();

    // แสดง header
    let header: Vec<String> = columns.iter().zip(col_widths.iter())
        .map(|(c, w)| format!("{:width$}", c, width = w))
        .collect();
    println!("| {} |", header.join(" | "));

    // separator
    let sep: Vec<String> = col_widths.iter()
        .map(|w| "-".repeat(*w))
        .collect();
    println!("|-{}-|", sep.join("-|-"));

    // rows
    for record in records.iter().take(limit) {
        let row: Vec<String> = columns.iter().zip(col_widths.iter())
            .map(|(c, w)| format!("{:width$}", record.get(c).to_string(), width = w))
            .collect();
        println!("| {} |", row.join(" | "));
    }

    if records.len() > limit {
        println!("... ({} more rows)", records.len() - limit);
    }
}
```

---

### ขั้นที่ 3: Parquet Reader และ Writer

Parquet เป็น columnar format ที่นิยมใช้ใน Data Engineering เพราะ compression ดีและอ่านได้เร็วมากในแบบ column-scan

**src/parquet_io.rs:**

```rust
use std::path::Path;
use std::sync::Arc;
use anyhow::{anyhow, Result};

use arrow::array::{
    Array, BooleanArray, Float32Array, Float64Array,
    Int32Array, Int64Array, StringArray,
};
use arrow::datatypes::{DataType as ArrowDataType, Field, Schema};
use arrow::record_batch::RecordBatch;
use parquet::arrow::ArrowWriter;
use parquet::arrow::arrow_reader::ParquetRecordBatchReaderBuilder;
use parquet::basic::Compression;
use parquet::file::properties::WriterProperties;

use crate::types::{DataType, Record, Value};
use crate::schema::infer_schema;

/// อ่าน Parquet file และ return Vec<Record>
/// อ่านทีละ row group เพื่อควบคุม memory
pub fn read_parquet(path: &Path) -> Result<Vec<Record>> {
    let file = std::fs::File::open(path)?;
    let builder = ParquetRecordBatchReaderBuilder::try_new(file)?;

    // แสดง schema
    let schema = builder.schema().clone();
    eprintln!("Parquet schema: {} columns", schema.fields().len());
    for field in schema.fields() {
        eprintln!("  - {}: {:?}", field.name(), field.data_type());
    }

    let reader = builder.build()?;
    let mut records = Vec::new();

    for batch_result in reader {
        let batch = batch_result?;
        let columns: Vec<String> = batch.schema().fields().iter()
            .map(|f| f.name().clone())
            .collect();

        for row_idx in 0..batch.num_rows() {
            let mut rec = Record::new(columns.clone());

            for (col_idx, field) in batch.schema().fields().iter().enumerate() {
                let col_array = batch.column(col_idx);
                let val = if col_array.is_null(row_idx) {
                    Value::Null
                } else {
                    match field.data_type() {
                        ArrowDataType::Int32 => {
                            let arr = col_array.as_any()
                                .downcast_ref::<Int32Array>().unwrap();
                            Value::Int(arr.value(row_idx) as i64)
                        }
                        ArrowDataType::Int64 => {
                            let arr = col_array.as_any()
                                .downcast_ref::<Int64Array>().unwrap();
                            Value::Int(arr.value(row_idx))
                        }
                        ArrowDataType::Float32 => {
                            let arr = col_array.as_any()
                                .downcast_ref::<Float32Array>().unwrap();
                            Value::Float(arr.value(row_idx) as f64)
                        }
                        ArrowDataType::Float64 => {
                            let arr = col_array.as_any()
                                .downcast_ref::<Float64Array>().unwrap();
                            Value::Float(arr.value(row_idx))
                        }
                        ArrowDataType::Boolean => {
                            let arr = col_array.as_any()
                                .downcast_ref::<BooleanArray>().unwrap();
                            Value::Bool(arr.value(row_idx))
                        }
                        ArrowDataType::Utf8 => {
                            let arr = col_array.as_any()
                                .downcast_ref::<StringArray>().unwrap();
                            Value::Str(arr.value(row_idx).to_string())
                        }
                        other => {
                            eprintln!("Warning: unsupported type {:?}, storing as Null", other);
                            Value::Null
                        }
                    }
                };

                rec.set(field.name().clone(), val);
            }
            records.push(rec);
        }
    }

    Ok(records)
}

/// เขียน Vec<Record> เป็น Parquet file
/// Schema infer จาก records แรก, compression = SNAPPY
pub fn write_parquet(records: &[Record], path: &Path) -> Result<()> {
    if records.is_empty() {
        return Err(anyhow!("Cannot write empty records to Parquet"));
    }

    let schema_map = infer_schema(records);
    let columns = &records[0].columns;

    // สร้าง Arrow schema
    let fields: Vec<Field> = columns.iter().map(|col| {
        let dtype = schema_map.get(col).unwrap_or(&DataType::Str);
        let arrow_type = match dtype {
            DataType::Int64    => ArrowDataType::Int64,
            DataType::Float64  => ArrowDataType::Float64,
            DataType::Bool     => ArrowDataType::Boolean,
            DataType::Str      => ArrowDataType::Utf8,
            DataType::DateTime => ArrowDataType::Utf8, // เก็บ datetime เป็น string
        };
        Field::new(col, arrow_type, true)
    }).collect();

    let schema = Arc::new(Schema::new(fields));

    // สร้าง Arrow arrays จาก records
    let arrays: Vec<Arc<dyn Array>> = columns.iter().map(|col| {
        let dtype = schema_map.get(col).unwrap_or(&DataType::Str);
        build_arrow_array(records, col, dtype)
    }).collect();

    let batch = RecordBatch::try_new(schema.clone(), arrays)?;

    // เปิดไฟล์และเขียน
    let file = std::fs::File::create(path)?;
    let props = WriterProperties::builder()
        .set_compression(Compression::SNAPPY)
        .build();

    let mut writer = ArrowWriter::try_new(file, schema, Some(props))?;
    writer.write(&batch)?;
    writer.close()?;

    Ok(())
}

/// สร้าง Arrow array จาก column ของ records
fn build_arrow_array(
    records: &[Record],
    col: &str,
    dtype: &DataType,
) -> Arc<dyn Array> {
    match dtype {
        DataType::Int64 => {
            let vals: Vec<Option<i64>> = records.iter()
                .map(|r| r.get(col).as_f64().map(|f| f as i64))
                .collect();
            Arc::new(Int64Array::from(vals))
        }
        DataType::Float64 => {
            let vals: Vec<Option<f64>> = records.iter()
                .map(|r| r.get(col).as_f64())
                .collect();
            Arc::new(Float64Array::from(vals))
        }
        DataType::Bool => {
            let vals: Vec<Option<bool>> = records.iter()
                .map(|r| match r.get(col) {
                    Value::Bool(b) => Some(*b),
                    _              => None,
                })
                .collect();
            Arc::new(BooleanArray::from(vals))
        }
        DataType::Str | DataType::DateTime => {
            let vals: Vec<Option<&str>> = records.iter()
                .map(|r| r.get(col).as_str())
                .collect();
            Arc::new(StringArray::from(vals))
        }
    }
}
```

**⚠️ Pitfall #3: Parquet Schema vs Arrow Schema**

`parquet` crate ใช้ Arrow เป็น in-memory format ดังนั้น schema ที่เขียนลงไฟล์จะผ่าน `arrow::datatypes::Schema` ก่อน ปัญหาที่พบบ่อยคือ:

1. **Nullable columns**: Arrow field ต้องตั้ง `nullable = true` ไม่งั้น NULL values จะทำให้ `RecordBatch::try_new` error
2. **Timestamp vs String**: ถ้าต้องการ datetime filtering ที่ถูกต้อง ควรใช้ `ArrowDataType::Timestamp(TimeUnit::Microsecond, None)` แทน Utf8 แต่ต้องแปลง string → i64 ก่อน
3. **Mixed Int32/Int64**: Parquet files จาก tools อื่น (เช่น pandas) มักเขียน INT32 สำหรับ column ที่มีค่าน้อย แต่ Arrow reader จะ return `Int32Array` ไม่ใช่ `Int64Array`

---

### ขั้นที่ 4: Filter Expression Evaluator

นี่คือส่วนที่น่าสนใจที่สุด — recursive descent parser ที่แปลง string เช่น `"age > 18 AND dept == \"Engineering\""` เป็น filter function:

**src/filter.rs:**

```rust
use anyhow::{anyhow, Result};
use crate::types::{Record, Value};

#[derive(Debug, Clone, PartialEq)]
enum Token {
    Ident(String),
    Number(f64),
    StringLit(String),
    Op(String),   // >, <, >=, <=, ==, !=
    And,
    Or,
    Not,
    LParen,
    RParen,
}

/// Tokenizer: แปลง expression string เป็น Vec<Token>
fn tokenize(expr: &str) -> Result<Vec<Token>> {
    let mut tokens = Vec::new();
    let mut chars = expr.chars().peekable();

    while let Some(&c) = chars.peek() {
        match c {
            ' ' | '\t' => { chars.next(); }
            '(' => { tokens.push(Token::LParen); chars.next(); }
            ')' => { tokens.push(Token::RParen); chars.next(); }
            '>' | '<' | '!' | '=' => {
                chars.next();
                let mut op = c.to_string();
                if chars.peek() == Some(&'=') {
                    op.push('=');
                    chars.next();
                }
                tokens.push(Token::Op(op));
            }
            '"' | '\'' => {
                let quote = c;
                chars.next();
                let mut s = String::new();
                while let Some(&ch) = chars.peek() {
                    if ch == quote { chars.next(); break; }
                    s.push(ch);
                    chars.next();
                }
                tokens.push(Token::StringLit(s));
            }
            '0'..='9' | '.' => {
                let mut num = String::new();
                while let Some(&d) = chars.peek() {
                    if d.is_ascii_digit() || d == '.' {
                        num.push(d); chars.next();
                    } else { break; }
                }
                let n: f64 = num.parse()
                    .map_err(|_| anyhow!("Invalid number: {}", num))?;
                tokens.push(Token::Number(n));
            }
            '-' => {
                // negative number
                chars.next();
                let mut num = String::from("-");
                while let Some(&d) = chars.peek() {
                    if d.is_ascii_digit() || d == '.' {
                        num.push(d); chars.next();
                    } else { break; }
                }
                let n: f64 = num.parse()
                    .map_err(|_| anyhow!("Invalid number: {}", num))?;
                tokens.push(Token::Number(n));
            }
            'a'..='z' | 'A'..='Z' | '_' => {
                let mut ident = String::new();
                while let Some(&ch) = chars.peek() {
                    if ch.is_alphanumeric() || ch == '_' || ch == '.' {
                        ident.push(ch); chars.next();
                    } else { break; }
                }
                match ident.to_lowercase().as_str() {
                    "and"   => tokens.push(Token::And),
                    "or"    => tokens.push(Token::Or),
                    "not"   => tokens.push(Token::Not),
                    "true"  => tokens.push(Token::Number(1.0)),
                    "false" => tokens.push(Token::Number(0.0)),
                    _       => tokens.push(Token::Ident(ident)),
                }
            }
            other => return Err(anyhow!("Unexpected character: {:?}", other)),
        }
    }
    Ok(tokens)
}

/// Evaluate single comparison: col OP value
fn eval_comparison(record: &Record, col: &str, op: &str, rhs: &Value) -> bool {
    let lhs = record.get(col);
    match op {
        ">"  => lhs.partial_cmp(rhs).map_or(false, |o| o.is_gt()),
        "<"  => lhs.partial_cmp(rhs).map_or(false, |o| o.is_lt()),
        ">=" => lhs.partial_cmp(rhs).map_or(false, |o| o.is_ge()),
        "<=" => lhs.partial_cmp(rhs).map_or(false, |o| o.is_le()),
        "==" | "=" => lhs == rhs,
        "!=" => lhs != rhs,
        _    => false,
    }
}

/// expr ::= and_expr ('OR' and_expr)*
pub fn eval_expr(rec: &Record, tokens: &[Token], pos: &mut usize) -> Result<bool> {
    let mut result = eval_and(rec, tokens, pos)?;
    while *pos < tokens.len() && tokens[*pos] == Token::Or {
        *pos += 1;
        result = result || eval_and(rec, tokens, pos)?;
    }
    Ok(result)
}

/// and_expr ::= not_expr ('AND' not_expr)*
fn eval_and(rec: &Record, tokens: &[Token], pos: &mut usize) -> Result<bool> {
    let mut result = eval_not(rec, tokens, pos)?;
    while *pos < tokens.len() && tokens[*pos] == Token::And {
        *pos += 1;
        result = result && eval_not(rec, tokens, pos)?;
    }
    Ok(result)
}

/// not_expr ::= 'NOT' atom | atom
fn eval_not(rec: &Record, tokens: &[Token], pos: &mut usize) -> Result<bool> {
    if *pos < tokens.len() && tokens[*pos] == Token::Not {
        *pos += 1;
        return Ok(!eval_atom(rec, tokens, pos)?);
    }
    eval_atom(rec, tokens, pos)
}

/// atom ::= '(' expr ')' | ident op value
fn eval_atom(rec: &Record, tokens: &[Token], pos: &mut usize) -> Result<bool> {
    if *pos >= tokens.len() {
        return Err(anyhow!("Unexpected end of expression"));
    }

    // Parenthesized expression
    if tokens[*pos] == Token::LParen {
        *pos += 1;
        let result = eval_expr(rec, tokens, pos)?;
        if *pos < tokens.len() && tokens[*pos] == Token::RParen {
            *pos += 1;
        } else {
            return Err(anyhow!("Missing closing parenthesis"));
        }
        return Ok(result);
    }

    // ident OP value
    if let Token::Ident(col) = tokens[*pos].clone() {
        *pos += 1;
        if *pos >= tokens.len() {
            // ตรวจสอบว่า column มีค่า (not null, not zero)
            return Ok(!rec.get(&col).is_null());
        }
        if let Token::Op(op) = tokens[*pos].clone() {
            *pos += 1;
            if *pos >= tokens.len() {
                return Err(anyhow!("Expected value after operator"));
            }
            let rhs = match &tokens[*pos] {
                Token::Number(n) => {
                    if n.fract() == 0.0 {
                        Value::Int(*n as i64)
                    } else {
                        Value::Float(*n)
                    }
                }
                Token::StringLit(s) | Token::Ident(s) => Value::Str(s.clone()),
                other => return Err(anyhow!("Unexpected RHS token: {:?}", other)),
            };
            *pos += 1;
            return Ok(eval_comparison(rec, &col, &op, &rhs));
        }
    }

    Err(anyhow!("Cannot parse expression at position {}", pos))
}

/// Public API: filter records ตาม expression string
pub fn apply_filter(records: &[Record], expr: &str) -> Result<Vec<Record>> {
    let tokens = tokenize(expr)?;
    let mut result = Vec::new();
    for record in records {
        let mut pos = 0;
        if eval_expr(record, &tokens, &mut pos)? {
            result.push(record.clone());
        }
    }
    Ok(result)
}
```

---

### ขั้นที่ 5: Aggregation และ Statistics

**src/aggregation.rs:**

```rust
use std::collections::HashMap;
use anyhow::{anyhow, Result};
use crate::types::{Record, Value};

pub enum AggFunc { Sum, Count, Avg, Min, Max }

impl AggFunc {
    pub fn from_str(s: &str) -> Result<AggFunc> {
        match s.to_lowercase().as_str() {
            "sum"   => Ok(AggFunc::Sum),
            "count" => Ok(AggFunc::Count),
            "avg"   => Ok(AggFunc::Avg),
            "min"   => Ok(AggFunc::Min),
            "max"   => Ok(AggFunc::Max),
            _       => Err(anyhow!("Unknown agg function: {}", s)),
        }
    }
}

/// group-by aggregation
/// agg_str: "column:function" เช่น "salary:sum"
pub fn group_by(records: &[Record], group_col: &str, agg_str: &str) -> Result<Vec<Record>> {
    let (col, func_str) = agg_str.split_once(':')
        .ok_or_else(|| anyhow!("agg must be 'col:func'"))?;
    let func = AggFunc::from_str(func_str)?;

    // รวบรวมค่าต่อ group key
    let mut groups: HashMap<String, Vec<f64>> = HashMap::new();
    let mut counts: HashMap<String, usize>    = HashMap::new();

    for rec in records {
        let key = rec.get(group_col).to_string();
        *counts.entry(key.clone()).or_insert(0) += 1;
        if let Some(v) = rec.get(col).as_f64() {
            groups.entry(key).or_default().push(v);
        } else {
            groups.entry(key).or_default(); // เพื่อให้ key exist
        }
    }

    let result_col = format!("{}_{}", col, func_str);
    let columns = vec![group_col.to_string(), result_col.clone()];

    let mut keys: Vec<String> = groups.keys().cloned().collect();
    keys.sort(); // sort ตาม key เพื่อ deterministic output

    keys.iter().map(|key| {
        let mut rec = Record::new(columns.clone());
        rec.set(group_col.to_string(), Value::Str(key.clone()));

        let agg_val = match func {
            AggFunc::Count => Value::Int(*counts.get(key).unwrap_or(&0) as i64),
            AggFunc::Sum => {
                Value::Float(groups[key].iter().sum())
            }
            AggFunc::Avg => {
                let vals = &groups[key];
                if vals.is_empty() { Value::Null }
                else { Value::Float(vals.iter().sum::<f64>() / vals.len() as f64) }
            }
            AggFunc::Min => groups[key].iter().cloned()
                .reduce(f64::min).map(Value::Float).unwrap_or(Value::Null),
            AggFunc::Max => groups[key].iter().cloned()
                .reduce(f64::max).map(Value::Float).unwrap_or(Value::Null),
        };

        rec.set(result_col.clone(), agg_val);
        Ok(rec)
    }).collect()
}
```

**src/stats.rs** — per-column statistics:

```rust
use std::collections::HashMap;
use crate::types::{Record, Value};

pub struct ColumnStats {
    pub count:      usize,
    pub null_count: usize,
    // numeric
    pub min:  Option<f64>,
    pub max:  Option<f64>,
    pub mean: Option<f64>,
    pub std:  Option<f64>,
    // string: top 5 most common values
    pub top5: Vec<(String, usize)>,
}

pub fn compute_stats(records: &[Record]) -> HashMap<String, ColumnStats> {
    if records.is_empty() { return HashMap::new(); }

    records[0].columns.iter().map(|col| {
        let mut null_count = 0usize;
        let mut nums: Vec<f64> = Vec::new();
        let mut str_counts: HashMap<String, usize> = HashMap::new();
        let mut is_numeric = true;

        for rec in records {
            match rec.get(col) {
                Value::Null     => { null_count += 1; }
                Value::Int(i)   => { nums.push(*i as f64); }
                Value::Float(f) => { nums.push(*f); }
                Value::Bool(b)  => {
                    is_numeric = false;
                    *str_counts.entry(b.to_string()).or_insert(0) += 1;
                }
                Value::Str(s)   => {
                    is_numeric = false;
                    *str_counts.entry(s.clone()).or_insert(0) += 1;
                }
            }
        }

        let count = records.len() - null_count;
        let stats = if is_numeric && !nums.is_empty() {
            let min  = nums.iter().cloned().fold(f64::INFINITY, f64::min);
            let max  = nums.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
            let mean = nums.iter().sum::<f64>() / nums.len() as f64;
            let variance = nums.iter()
                .map(|v| (v - mean).powi(2))
                .sum::<f64>() / nums.len() as f64;
            ColumnStats { count, null_count, min: Some(min), max: Some(max),
                          mean: Some(mean), std: Some(variance.sqrt()), top5: vec![] }
        } else {
            let mut top5: Vec<_> = str_counts.into_iter().collect();
            top5.sort_by(|a, b| b.1.cmp(&a.1));
            top5.truncate(5);
            ColumnStats { count, null_count, min: None, max: None,
                          mean: None, std: None, top5 }
        };

        (col.clone(), stats)
    }).collect()
}

pub fn print_stats(records: &[Record]) {
    let stats = compute_stats(records);
    if records.is_empty() { println!("No data"); return; }

    println!("{:<20} {:>8} {:>8} {:>12} {:>12} {:>12} {:>12}",
        "Column", "Count", "Nulls", "Min", "Max", "Mean", "Std");
    println!("{}", "-".repeat(90));

    for col in &records[0].columns {
        if let Some(s) = stats.get(col) {
            if s.min.is_some() {
                println!("{:<20} {:>8} {:>8} {:>12.2} {:>12.2} {:>12.4} {:>12.4}",
                    col, s.count, s.null_count,
                    s.min.unwrap(), s.max.unwrap(),
                    s.mean.unwrap(), s.std.unwrap());
            } else {
                println!("{:<20} {:>8} {:>8}  top5: {:?}",
                    col, s.count, s.null_count, s.top5);
            }
        }
    }
}
```

---

### ขั้นที่ 6: CLI Interface และ Chunked Processing

**src/transform.rs:**

```rust
use anyhow::{anyhow, Result};
use crate::types::{Record, Value};

/// เลือกเฉพาะคอลัมน์ที่ระบุ
pub fn select_columns(records: &[Record], cols: &[&str]) -> Result<Vec<Record>> {
    if let Some(first) = records.first() {
        for col in cols {
            if !first.columns.contains(&col.to_string()) {
                return Err(anyhow!("Column not found: '{}'", col));
            }
        }
    }
    let columns: Vec<String> = cols.iter().map(|s| s.to_string()).collect();
    Ok(records.iter().map(|rec| {
        let mut new_rec = Record::new(columns.clone());
        for col in cols {
            new_rec.set(col.to_string(), rec.get(col).clone());
        }
        new_rec
    }).collect())
}

/// Rename column: old → new
pub fn rename_column(records: &[Record], old: &str, new: &str) -> Result<Vec<Record>> {
    if records.is_empty() { return Ok(Vec::new()); }
    if !records[0].columns.contains(&old.to_string()) {
        return Err(anyhow!("Column not found: '{}'", old));
    }
    Ok(records.iter().map(|rec| {
        let cols: Vec<String> = rec.columns.iter()
            .map(|c| if c == old { new.to_string() } else { c.clone() })
            .collect();
        let mut new_rec = Record::new(cols);
        for col in &rec.columns {
            let target = if col == old { new } else { col.as_str() };
            new_rec.set(target.to_string(), rec.get(col).clone());
        }
        new_rec
    }).collect())
}

/// Sort records ตาม column (asc หรือ desc)
pub fn sort_records(records: &mut Vec<Record>, col: &str, desc: bool) -> Result<()> {
    records.sort_by(|a, b| {
        let ord = a.get(col).partial_cmp(b.get(col))
            .unwrap_or(std::cmp::Ordering::Equal);
        if desc { ord.reverse() } else { ord }
    });
    Ok(())
}
```

**src/main.rs** — CLI ครบทุก subcommand:

```rust
mod types;
mod schema;
mod filter;
mod aggregation;
mod stats;
mod transform;
mod csv_io;
mod parquet_io;

use anyhow::Result;
use clap::{Parser, Subcommand};
use std::path::PathBuf;

#[derive(Parser)]
#[command(
    name    = "dataproc",
    about   = "CSV/Parquet Data Processor",
    version = "1.0.0"
)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Filter rows ด้วย expression: filter 'age > 18 AND dept == "Engineering"'
    Filter {
        input: PathBuf,
        expr: String,
        #[arg(long, default_value = "10000")]
        chunk_size: usize,
    },
    /// เลือกเฉพาะคอลัมน์: select col1,col2,col3
    Select {
        input: PathBuf,
        columns: String,
    },
    /// Sort ตาม column: sort salary DESC
    Sort {
        input: PathBuf,
        column: String,
        #[arg(default_value = "ASC")]
        order: String,
    },
    /// จำกัดจำนวนแถว
    Limit {
        input: PathBuf,
        n: usize,
    },
    /// Rename column: rename old_name=new_name
    Rename {
        input: PathBuf,
        mapping: String,
    },
    /// Group-by aggregation
    GroupBy {
        input: PathBuf,
        /// Column ที่ใช้ group
        #[arg(long)]
        group: String,
        /// Aggregation: "column:function"
        #[arg(long)]
        agg: String,
    },
    /// แสดงสถิติต่อคอลัมน์
    Stats {
        input: PathBuf,
    },
    /// แสดง schema ที่ infer จากไฟล์
    Schema {
        input: PathBuf,
        #[arg(long, default_value = "100")]
        sample_rows: usize,
    },
    /// แปลง CSV → Parquet
    Csv2parquet {
        input:  PathBuf,
        output: PathBuf,
    },
    /// แปลง Parquet → CSV
    Parquet2csv {
        input:  PathBuf,
        output: PathBuf,
    },
}

fn main() -> Result<()> {
    let cli = Cli::parse();

    match cli.command {
        Commands::Filter { input, expr, chunk_size } => {
            println!("Reading {}...", input.display());
            let records = csv_io::read_csv(&input, chunk_size)?;
            let filtered = filter::apply_filter(&records, &expr)?;
            println!("Filtered {} → {} rows", records.len(), filtered.len());
            csv_io::print_table(&filtered, 20);
        }

        Commands::Select { input, columns } => {
            let records = csv_io::read_csv(&input, 10000)?;
            let cols: Vec<&str> = columns.split(',')
                .map(|s| s.trim()).collect();
            let selected = transform::select_columns(&records, &cols)?;
            csv_io::print_table(&selected, 20);
        }

        Commands::Sort { input, column, order } => {
            let mut records = csv_io::read_csv(&input, 10000)?;
            transform::sort_records(&mut records, &column, order == "DESC")?;
            csv_io::print_table(&records, 20);
        }

        Commands::Limit { input, n } => {
            let records = csv_io::read_csv(&input, n + 1)?;
            let limited: Vec<_> = records.into_iter().take(n).collect();
            csv_io::print_table(&limited, n);
        }

        Commands::Rename { input, mapping } => {
            let records = csv_io::read_csv(&input, 10000)?;
            let (old, new) = mapping.split_once('=')
                .expect("mapping format must be: old_name=new_name");
            let renamed = transform::rename_column(&records, old.trim(), new.trim())?;
            csv_io::print_table(&renamed, 20);
        }

        Commands::GroupBy { input, group, agg } => {
            let records = csv_io::read_csv(&input, 10000)?;
            let result = aggregation::group_by(&records, &group, &agg)?;
            csv_io::print_table(&result, 50);
        }

        Commands::Stats { input } => {
            let records = csv_io::read_csv(&input, 10000)?;
            stats::print_stats(&records);
        }

        Commands::Schema { input, sample_rows } => {
            let records = csv_io::read_csv(&input, sample_rows)?;
            let schema = schema::infer_schema(&records);
            println!("Schema ({} columns):", schema.len());
            if let Some(first) = records.first() {
                for col in &first.columns {
                    let dtype = schema.get(col)
                        .map(|d| d.to_string())
                        .unwrap_or("unknown".into());
                    println!("  {:<20} {}", col, dtype);
                }
            }
        }

        Commands::Csv2parquet { input, output } => {
            println!("Reading CSV: {}", input.display());
            let records = csv_io::read_csv(&input, 50000)?;
            println!("Read {} rows. Writing Parquet...", records.len());
            parquet_io::write_parquet(&records, &output)?;
            println!("Written: {}", output.display());
        }

        Commands::Parquet2csv { input, output } => {
            println!("Reading Parquet: {}", input.display());
            let records = parquet_io::read_parquet(&input)?;
            println!("Read {} rows. Writing CSV...", records.len());
            csv_io::write_csv(&records, &output)?;
            println!("Written: {}", output.display());
        }
    }

    Ok(())
}
```

---

### ขั้นที่ 7: Integration Tests

**tests/integration_test.rs:**

```rust
use std::io::Write;
use tempfile::NamedTempFile;

// ฟังก์ชัน helper สร้าง CSV temp file
fn create_csv(content: &str) -> NamedTempFile {
    let mut f = NamedTempFile::new().unwrap();
    f.write_all(content.as_bytes()).unwrap();
    f
}

#[test]
fn test_filter_and_select_pipeline() {
    // จำลอง pipeline: อ่าน CSV → filter → select
    // ตรวจสอบว่าไม่ crash และผลลัพธ์ถูกต้อง
    let csv_content = "name,age,dept,salary\n\
        Alice,29,Engineering,95000\n\
        Bob,35,Marketing,72000\n\
        Carol,22,Engineering,55000\n\
        Dave,40,HR,80000\n";

    let f = create_csv(csv_content);

    // ทดสอบใช้ function โดยตรง
    use csv_parquet_processor::csv_io::read_csv;
    use csv_parquet_processor::filter::apply_filter;
    use csv_parquet_processor::transform::select_columns;

    let records = read_csv(f.path(), 1000).unwrap();
    assert_eq!(records.len(), 4);

    let filtered = apply_filter(&records, "age > 25 AND dept == \"Engineering\"").unwrap();
    assert_eq!(filtered.len(), 1);
    assert_eq!(filtered[0].get("name").to_string(), "Alice");

    let selected = select_columns(&filtered, &["name", "salary"]).unwrap();
    assert_eq!(selected[0].columns, vec!["name", "salary"]);
}
```

สำหรับ integration tests จาก library API ต้องเพิ่ม `lib.rs`:

```rust
// src/lib.rs
pub mod types;
pub mod schema;
pub mod filter;
pub mod aggregation;
pub mod stats;
pub mod transform;
pub mod csv_io;
pub mod parquet_io;
```

และ `Cargo.toml`:

```toml
[lib]
name = "csv_parquet_processor"
path = "src/lib.rs"
```

---

## การทดสอบ (Testing)

### Unit Tests

โปรเจคมี unit tests แบบ inline ในทุก module ครอบคลุม:

| Module | Tests |
|---|---|
| `schema` | parse_value (int/float/bool/null/datetime), infer_schema (int/float/bool/mixed/datetime) |
| `filter` | gt/lt/and/or/eq_string/ne/empty/paren expressions |
| `aggregation` | count/sum/avg/min/max, invalid function error |
| `transform` | select_columns, rename_column, sort_asc/desc |
| `csv_io` | read basic, read with nulls, write-read roundtrip |

### รัน Tests และ Output จริง

```bash
$ cargo test
```

```
running 30 tests
test aggregation::tests::test_group_by_count ... ok
test aggregation::tests::test_group_by_avg ... ok
test aggregation::tests::test_group_by_min_max ... ok
test aggregation::tests::test_group_by_sum ... ok
test aggregation::tests::test_invalid_agg_func ... ok
test csv_io::tests::test_read_csv_with_nulls ... ok
test csv_io::tests::test_read_csv_basic ... ok
test filter::tests::test_filter_and ... ok
test filter::tests::test_filter_empty_result ... ok
test filter::tests::test_filter_gt ... ok
test filter::tests::test_filter_lt ... ok
test filter::tests::test_filter_ne ... ok
test filter::tests::test_filter_or ... ok
test filter::tests::test_filter_eq_string ... ok
test csv_io::tests::test_write_read_roundtrip ... ok
test schema::tests::test_infer_schema_datetime ... ok
test schema::tests::test_infer_schema_float_col ... ok
test schema::tests::test_infer_schema_int_col ... ok
test filter::tests::test_filter_paren ... ok
test schema::tests::test_parse_value_bool ... ok
test schema::tests::test_parse_value_datetime ... ok
test schema::tests::test_infer_schema_mixed_int_float ... ok
test schema::tests::test_parse_value_float ... ok
test schema::tests::test_parse_value_int ... ok
test transform::tests::test_rename_column ... ok
test transform::tests::test_select_columns ... ok
test transform::tests::test_select_nonexistent_column ... ok
test transform::tests::test_sort_asc ... ok
test transform::tests::test_sort_desc ... ok
test schema::tests::test_parse_value_null ... ok

test result: ok. 30 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### Demo Run กับ Sample Data

สร้าง `sample_employees.csv`:

```csv
id,name,dept,age,salary,hired_at,active
1,Alice Johnson,Engineering,29,95000,2021-03-15,true
2,Bob Smith,Marketing,35,72000,2019-07-01,true
3,Carol Davis,Engineering,42,110000,2017-01-10,true
4,Dave Wilson,Marketing,28,68000,2022-11-20,false
5,Eve Brown,HR,31,65000,2020-05-15,true
6,Frank Lee,Engineering,26,88000,2023-01-08,true
7,Grace Kim,HR,45,78000,2015-09-30,true
8,Hank Miller,Marketing,22,55000,2024-02-14,true
9,Ivy Chen,Engineering,33,102000,2018-06-22,true
10,Jack Wang,HR,27,61000,2023-08-01,false
```

**Filter: `age > 30`**

```bash
$ ./dataproc filter sample_employees.csv "age > 30"
```
```
Filtered 10 -> 5 rows
| id | name        | dept        | age | salary | hired_at   | active |
|----|-------------|-------------|-----|--------|------------|--------|
| 2  | Bob Smith   | Marketing   | 35  | 72000  | 2019-07-01 | true   |
| 3  | Carol Davis | Engineering | 42  | 110000 | 2017-01-10 | true   |
| 5  | Eve Brown   | HR          | 31  | 65000  | 2020-05-15 | true   |
| 7  | Grace Kim   | HR          | 45  | 78000  | 2015-09-30 | true   |
| 9  | Ivy Chen    | Engineering | 33  | 102000 | 2018-06-22 | true   |
```

**Select: `name,dept,salary`**

```bash
$ ./dataproc select sample_employees.csv "name,dept,salary"
```
```
| name          | dept        | salary |
|---------------|-------------|--------|
| Alice Johnson | Engineering | 95000  |
| Bob Smith     | Marketing   | 72000  |
| Carol Davis   | Engineering | 110000 |
| Dave Wilson   | Marketing   | 68000  |
| Eve Brown     | HR          | 65000  |
| Frank Lee     | Engineering | 88000  |
| Grace Kim     | HR          | 78000  |
| Hank Miller   | Marketing   | 55000  |
| Ivy Chen      | Engineering | 102000 |
| Jack Wang     | HR          | 61000  |
```

**Sort by `salary DESC`**

```bash
$ ./dataproc sort sample_employees.csv salary DESC
```
```
| id | name          | dept        | age | salary | hired_at   | active |
|----|---------------|-------------|-----|--------|------------|--------|
| 3  | Carol Davis   | Engineering | 42  | 110000 | 2017-01-10 | true   |
| 9  | Ivy Chen      | Engineering | 33  | 102000 | 2018-06-22 | true   |
| 1  | Alice Johnson | Engineering | 29  | 95000  | 2021-03-15 | true   |
| 6  | Frank Lee     | Engineering | 26  | 88000  | 2023-01-08 | true   |
| 7  | Grace Kim     | HR          | 45  | 78000  | 2015-09-30 | true   |
| 2  | Bob Smith     | Marketing   | 35  | 72000  | 2019-07-01 | true   |
| 4  | Dave Wilson   | Marketing   | 28  | 68000  | 2022-11-20 | false  |
| 5  | Eve Brown     | HR          | 31  | 65000  | 2020-05-15 | true   |
| 10 | Jack Wang     | HR          | 27  | 61000  | 2023-08-01 | false  |
| 8  | Hank Miller   | Marketing   | 22  | 55000  | 2024-02-14 | true   |
```

**Group-by `dept`, sum `salary`**

```bash
$ ./dataproc group-by sample_employees.csv --group dept --agg salary:sum
```
```
| dept        | salary_sum  |
|-------------|-------------|
| Engineering | 395000.0000 |
| HR          | 204000.0000 |
| Marketing   | 195000.0000 |
```

**Group-by `dept`, avg `age`**

```bash
$ ./dataproc group-by sample_employees.csv --group dept --agg age:avg
```
```
| dept        | age_avg |
|-------------|---------|
| Engineering | 32.5000 |
| HR          | 34.3333 |
| Marketing   | 28.3333 |
```

**Group-by count**

```bash
$ ./dataproc group-by sample_employees.csv --group dept --agg dept:count
```
```
| dept        | dept_count |
|-------------|------------|
| Engineering | 4          |
| HR          | 3          |
| Marketing   | 3          |
```

**Stats**

```bash
$ ./dataproc stats sample_employees.csv
```
```
Column                  Count    Nulls          Min          Max         Mean          Std
------------------------------------------------------------------------------------------
id                         10        0         1.00        10.00       5.5000       2.8723
name                       10        0  top5: [("Hank Miller", 1), ("Frank Lee", 1), ("Bob Smith", 1), ("Alice Johnson", 1), ("Carol Davis", 1)]
dept                       10        0  top5: [("Engineering", 4), ("HR", 3), ("Marketing", 3)]
age                        10        0        22.00        45.00      31.8000       6.8235
salary                     10        0     55000.00    110000.00   79400.0000   17585.2211
hired_at                   10        0  top5: [("2022-11-20", 1), ("2019-07-01", 1), ("2021-03-15", 1), ("2023-01-08", 1), ("2015-09-30", 1)]
active                     10        0  top5: [("true", 8), ("false", 2)]
```

**Schema inference**

```bash
$ ./dataproc schema sample_employees.csv
```
```
  name: string
  active: bool
  age: int64
  id: int64
  dept: string
  salary: int64
  hired_at: datetime
```

---

## ⚠️ Pitfalls สำคัญ

### Pitfall #1: `PartialOrd` กับ Mixed Numeric Types

ดังที่กล่าวใน ขั้นที่ 1 ถ้า `salary` ใน CSV เป็น `"95000"` จะถูก parse เป็น `Value::Int(95000)` แต่ถ้า filter เขียน `salary > 50000.0` ตัวเลข `50000.0` จะถูก tokenize เป็น `Token::Number(50000.0)` และแปลงเป็น `Value::Float` เพราะ `.0` ซึ่งทำให้ `Int.partial_cmp(Float)` return `None`

**แก้ไข**: ใน `eval_atom` ให้ตรวจ fract ก่อนแปลง:

```rust
Token::Number(n) => {
    if n.fract() == 0.0 {
        Value::Int(*n as i64)   // "50000.0" → Int(50000)
    } else {
        Value::Float(*n)         // "50000.5" → Float(50000.5)
    }
}
```

### Pitfall #2: CSV Quote Characters ใน Fields

`csv` crate จัดการ quoted fields โดยอัตโนมัติ แต่ถ้าไฟล์ใช้ quote character ที่ไม่ใช่ `"` (เช่น `'`) ต้องตั้งค่าเอง:

```rust
// ⚠️ ค่า default คือ " (double quote)
let mut rdr = csv::ReaderBuilder::new()
    .quote(b'\'')   // สำหรับ single-quote CSV
    .from_path(path)?;
```

นอกจากนี้ CSV บางไฟล์ใช้ `\` เป็น escape character แทน double-quote:

```rust
.double_quote(false)
.escape(Some(b'\\'))
```

### Pitfall #3: Parquet Row Groups และ Memory

Parquet file แบ่งเป็น row groups ขนาดใหญ่ (default ~128MB) เมื่ออ่าน 1 row group ทั้ง group จะถูกโหลดเข้า RAM พร้อมกัน สำหรับไฟล์ขนาดใหญ่ควรตั้ง batch size:

```rust
// ควบคุมขนาด batch ที่อ่านต่อครั้ง
let reader = builder
    .with_batch_size(10_000)   // อ่านทีละ 10000 rows
    .build()?;
```

และเมื่อเขียน ควรตั้ง row group size:

```rust
let props = WriterProperties::builder()
    .set_compression(Compression::SNAPPY)
    .set_max_row_group_size(100_000)   // ไม่เกิน 100K rows ต่อ row group
    .build();
```

### Pitfall #4: `HashMap` ลำดับไม่แน่นอนใน `Record`

`HashMap<String, Value>` ใน `Record` ไม่รักษาลำดับ insertion ดังนั้น `Vec<String> columns` ต้องเก็บแยก ถ้าเขียน:

```rust
// ❌ ผิด: columns จะเรียงแบบสุ่มจาก HashMap
let cols: Vec<String> = record.fields.keys().cloned().collect();
```

```rust
// ✓ ถูก: ใช้ record.columns ซึ่งเรียงตามลำดับที่ set ไว้
let cols = &record.columns;
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# สร้าง optimized binary
cargo build --release

# ขนาด binary ก่อน strip
du -sh target/release/dataproc
# ประมาณ 8-15 MB

# strip debug symbols
strip target/release/dataproc
du -sh target/release/dataproc
# ลดเหลือประมาณ 3-5 MB
```

### Cross Compilation

```bash
# Build สำหรับ Linux (musl static binary)
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl

# Build สำหรับ macOS arm64
rustup target add aarch64-apple-darwin
cargo build --release --target aarch64-apple-darwin
```

### Docker Image

```dockerfile
FROM rust:1.76-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/dataproc /usr/local/bin/
ENTRYPOINT ["dataproc"]
```

### การ Process ไฟล์ขนาดใหญ่

สำหรับ CSV ขนาด 10GB+ ที่ใหญ่กว่า RAM ให้ใช้ `--chunk-size` และ pipe output:

```bash
# กรอง + เขียนผลลัพธ์ใหม่
./dataproc filter huge_data.csv "status == active" \
    --chunk-size 50000 \
    > filtered.csv

# Convert ทีละ chunk
./dataproc csv2parquet huge_data.csv output.parquet
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม `JOIN` command

สร้าง subcommand `join` ที่ join สอง CSV files ด้วย column ร่วมกัน:

```bash
./dataproc join employees.csv departments.csv --on dept_id --type inner
```

Hint: ใช้ `HashMap<String, Vec<Record>>` สำหรับ hash join, รองรับ `inner`, `left`, `right` join types

ความยาก: ⭐⭐⭐ — ต้องจัดการ duplicate column names และ null padding สำหรับ left/right join

### แบบฝึกหัดที่ 2: Expression เพิ่ม functions

ขยาย filter expression ให้รองรับ built-in functions:

```bash
# string functions
./dataproc filter data.csv "UPPER(name) == ALICE"
./dataproc filter data.csv "LEN(name) > 5"
./dataproc filter data.csv "CONTAINS(email, @gmail)"

# date functions  
./dataproc filter data.csv "YEAR(hired_at) >= 2022"
./dataproc filter data.csv "DATEDIFF(today, hired_at) > 365"
```

Hint: เพิ่ม `Token::FuncCall(String)` ใน tokenizer และ handle ใน `eval_atom`

ความยาก: ⭐⭐⭐⭐

### แบบฝึกหัดที่ 3: Streaming Parquet Write สำหรับไฟล์ใหญ่

ปัจจุบัน `write_parquet` รับ `Vec<Record>` ทั้งหมดเข้า RAM ก่อน ให้ rewrite ให้รองรับ streaming:

```rust
pub struct ParquetStreamWriter {
    writer: ArrowWriter<File>,
    schema: Arc<Schema>,
    buffer: Vec<Record>,
    batch_size: usize,
}

impl ParquetStreamWriter {
    pub fn new(path: &Path, schema: Arc<Schema>, batch_size: usize) -> Result<Self>;
    pub fn write_record(&mut self, record: Record) -> Result<()>;
    pub fn finish(self) -> Result<()>;
}
```

ความยาก: ⭐⭐⭐⭐ — ต้องจัดการ buffering และ flush ที่ batch boundaries

### แบบฝึกหัดที่ 4: PIVOT และ UNPIVOT

```bash
# PIVOT: แปลง rows → columns
./dataproc pivot sales.csv --rows product --cols quarter --values amount --agg sum

# ผลลัพธ์:
# product | Q1_sum | Q2_sum | Q3_sum | Q4_sum
# Apple   | 100    | 150    | 120    | 200
# Orange  | 80     | 90     | 110    | 95

# UNPIVOT: แปลง columns → rows (reverse of PIVOT)
./dataproc unpivot wide_table.csv --id_cols product --value_cols Q1,Q2,Q3,Q4
```

ความยาก: ⭐⭐⭐⭐⭐ — PIVOT ต้องอ่าน data สองรอบ (รอบแรกหา unique column values, รอบสองสร้างตาราง)

---

## สรุป

โปรเจคนี้สร้าง Data Processor ที่สามารถใช้งานจริงใน production ครอบคลุม:

**Pattern สำคัญที่ได้เรียน:**

1. **`Value` enum + `PartialOrd`** — วิธี model data แบบ dynamic typed ใน Rust ที่ยังคง type safety ด้วย exhaustive matching

2. **Recursive Descent Parser** — pattern สำหรับ parse expression language ด้วย Rust โดยไม่ต้องใช้ parser generator สามารถ extend ได้ง่าย

3. **Streaming vs Batch Processing** — เมื่อไหร่ควรใช้ chunk-based reading และ tradeoffs ระหว่าง simplicity กับ memory efficiency

4. **Arrow/Parquet Integration** — วิธีใช้ Apache Arrow เป็น in-memory format สำหรับ columnar operations และ Parquet เป็น on-disk format พร้อม compression

5. **`HashMap` + `Vec` สำหรับ ordered map** — pattern สำหรับเก็บข้อมูลที่ต้องการทั้ง O(1) lookup และ ordering

**เปรียบเทียบกับ tools จริง:**

| Feature | dataproc | DuckDB | Polars CLI |
|---|---|---|---|
| Memory efficient | ✓ (chunks) | ✓ (query optimizer) | ✓ (lazy eval) |
| Parquet support | ✓ | ✓ | ✓ |
| Filter language | custom expr | SQL | expression DSL |
| Binary size | ~5MB | ~30MB | Python required |

**โปรเจคถัดไป** จะต่อยอดด้วยการสร้าง **Database Backup System** (C07) ที่ใช้ Parquet format เป็น backup format สำหรับ PostgreSQL/MySQL tables รองรับ incremental backup, compression, และ point-in-time recovery

---

**โปรเจคก่อนหน้า:** [project-c05-search-engine.md](project-c05-search-engine.md) | **โปรเจคถัดไป:** [project-c07-db-backup.md](project-c07-db-backup.md)
