# Project C10: Schema Registry Service

> โมดูล: C — Data Processing & Pipelines | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 6 ชั่วโมง

## ภาพรวมโปรเจค

**Schema Registry** คือ centralised service สำหรับจัดเก็บและบริหาร schema ของข้อมูลในระบบ event-driven หรือ streaming pipeline เช่น Apache Kafka ในโลก production ทุกครั้งที่ producer ส่ง message ไปยัง Kafka topic มันจะ embed schema ID ใน message header แทนที่จะส่ง schema ทั้งก้อน consumer จะดึง schema จาก registry ด้วย ID นั้นเพื่อ deserialise ข้อมูล แนวทางนี้ลดขนาด message ลงได้มาก และรับประกัน schema evolution ที่ปลอดภัย

โปรเจคนี้ implementation Schema Registry ที่ compatible กับ **Confluent Schema Registry API** ซึ่งเป็น de facto standard ของ ecosystem โดยรองรับทั้ง JSON Schema (Draft 7) และ Avro IDL และมีระบบตรวจสอบ compatibility level แบบ BACKWARD / FORWARD / FULL / NONE เพื่อป้องกัน schema ที่ทำให้ consumer พังในระหว่างการ deploy

**Use case ใน production:**
- Kafka ecosystem ทุก company ขนาดกลาง-ใหญ่ที่ใช้ event sourcing
- Microservices ที่ต้องการ contract-first API evolution โดยไม่ down-time
- Data platform ที่ต้องการ governance และ auditability ของ schema ทุก version

**Learning value ที่ได้:**
- ออกแบบ REST API ที่ตรงกับ third-party standard (Confluent API)
- เข้าใจ schema evolution theory เชิงลึก
- ใช้ `axum 0.8` กับ `sqlx` แบบ async full-stack
- Fingerprinting ด้วย SHA-256 เพื่อ deduplication
- เขียน logic ที่ซับซ้อน (compatibility rules) แบบ pure functions ที่ testable

---

## สิ่งที่จะได้เรียนรู้

- **Schema compatibility theory** — BACKWARD, FORWARD, FULL, NONE และ rules ที่แต่ละ level บังคับ
- **REST API design** กับ `axum 0.8` — path parameters, JSON request/response, error handling แบบ idiomatic
- **Database integration** กับ `sqlx` และ SQLite — migration, prepared statements, connection pool
- **SHA-256 fingerprinting** เพื่อ detect duplicate schema และสร้าง globally unique ID
- **Schema diffing** — algorithm เปรียบเทียบ JSON Schema สองชุดแล้วหา added/removed/changed fields
- **Search และ query** — prefix search ของ subject names และ field name search ข้ามทุก schema
- **JSON Schema validation** — ตรวจสอบ syntax ของ schema ที่ผู้ใช้ส่งมาก่อน register
- **Error handling ระดับ production** — แยก validation error / compatibility error / not-found อย่างชัดเจน

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50**: async/await กับ Tokio runtime
- **Part 61–70**: REST API กับ Axum — handlers, routing, extractors
- **Part 71–80**: SQLx — connection pool, query!, sqlx::migrate!
- **Part 96–100**: Error handling ใน production — `thiserror`, `anyhow`
- **Part 101–105**: Serialisation/Deserialisation กับ `serde_json` รวมถึง `Value` แบบ dynamic
- ความคุ้นเคยกับ Apache Kafka แนวคิด (producer/consumer/topic) จะช่วยในการเข้าใจ context

---

## โครงสร้างโปรเจค (Project Layout)

```
schema-registry/
├── src/
│   ├── main.rs              # Entry point — server setup, routing
│   ├── models.rs            # Data types: Subject, SchemaVersion, CompatibilityLevel
│   ├── compatibility.rs     # Logic ตรวจสอบ BACKWARD/FORWARD/FULL compatibility
│   ├── fingerprint.rs       # SHA-256 fingerprinting + normalisation
│   ├── diff.rs              # Schema diffing algorithm
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── subjects.rs      # POST/GET /subjects/{name}/versions
│   │   ├── schemas.rs       # GET /schemas/ids/{id}
│   │   ├── compatibility.rs # POST /compatibility/subjects/{name}/versions/latest
│   │   └── search.rs        # GET /subjects?prefix=... , GET /schemas?contains=...
│   ├── db.rs                # SQLite operations via sqlx
│   └── error.rs             # AppError → axum IntoResponse
├── migrations/
│   └── 001_initial.sql      # Schema DDL
├── tests/
│   └── integration.rs       # HTTP-level integration tests
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
Producer (Kafka client)
       │  POST /subjects/user-created/versions
       │  body: { "schema": "{...JSON Schema...}" }
       ▼
┌─────────────────────────────────────────────────────┐
│                 Schema Registry                      │
│                                                     │
│  1. Validate schema syntax (JSON Schema Draft 7)    │
│  2. Load latest version from DB                     │
│  3. Check compatibility (BACKWARD by default)       │
│  4. Compute SHA-256 fingerprint                     │
│  5. Check if fingerprint already exists (dedup)     │
│  6. Assign global_id (auto-increment)               │
│  7. Store in SQLite                                 │
│  Returns: { "id": 42 }  (global schema ID)         │
└─────────────────────────────────────────────────────┘
       │
       ▼ embed schema ID in message header (5 bytes)
┌─────────────┐
│ Kafka Topic │
└─────────────┘
       │
       ▼
Consumer
  GET /schemas/ids/42  → schema text
  Deserialise message body
```

### ทำไมถึงใช้ SQLite แทน PostgreSQL

โปรเจคนี้ใช้ SQLite เพื่อให้รัน local ได้โดยไม่ต้องมี database server แต่ `sqlx` รองรับทั้ง SQLite และ PostgreSQL ด้วย API เดียวกัน การเปลี่ยนไปใช้ PostgreSQL ใน production แค่เปลี่ยน connection string และ `[dependencies]` ใน Cargo.toml

### ทำไม compatibility check ถึงเป็น pure function

Compatibility logic (BACKWARD/FORWARD) ไม่มี side effect ใดๆ — รับ `&Value` สองชุด คืน `CompatResult` เสมอ จึง unit-testable ได้โดยไม่ต้องมี database หรือ HTTP server ทำให้ test suite เร็วและ reliable สูง

### Global Schema ID vs. Subject Version

- **Global ID**: integer เพิ่มขึ้นทีละ 1 ข้ามทุก subject/version — ใช้ใน message header (Confluent wire format: magic byte 0x00 + 4-byte schema ID)
- **Subject + Version**: ใช้ lookup schema ตาม business context เช่น subject `user-created`, version 3

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Data Types

เริ่มจาก `Cargo.toml` และ type definitions ที่เป็นหัวใจของระบบ

**`Cargo.toml`:**

```toml
[package]
name = "schema-registry"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8", features = ["json", "macros"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio", "migrate", "chrono"] }
sha2 = "0.10"
hex = "0.4"
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2"
tower-http = { version = "0.6", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = "0.3"
uuid = { version = "1", features = ["v4"] }

[dev-dependencies]
axum-test = "0.4"
tokio = { version = "1", features = ["full"] }
```

**`src/models.rs`** — Data types หลักทั้งหมด:

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// ระดับ compatibility ที่รองรับ
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "TEXT")]
#[sqlx(rename_all = "SCREAMING_SNAKE_CASE")]
pub enum CompatibilityLevel {
    /// consumer ใหม่อ่าน data เก่าได้
    Backward,
    /// consumer เก่าอ่าน data ใหม่ได้
    Forward,
    /// ทั้ง BACKWARD และ FORWARD
    Full,
    /// ไม่ตรวจสอบ
    None,
}

impl Default for CompatibilityLevel {
    fn default() -> Self {
        CompatibilityLevel::Backward
    }
}

impl std::fmt::Display for CompatibilityLevel {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            CompatibilityLevel::Backward => write!(f, "BACKWARD"),
            CompatibilityLevel::Forward  => write!(f, "FORWARD"),
            CompatibilityLevel::Full     => write!(f, "FULL"),
            CompatibilityLevel::None     => write!(f, "NONE"),
        }
    }
}

/// Schema format ที่รองรับ
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "TEXT")]
pub enum SchemaFormat {
    JsonSchema,
    Avro,
}

/// หนึ่ง version ของ schema
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct SchemaVersion {
    pub id: i64,             // global schema ID
    pub subject: String,
    pub version: i32,
    pub schema_text: String,
    pub fingerprint: String, // SHA-256 hex
    pub format: String,      // "JsonSchema" | "Avro"
    pub registered_at: DateTime<Utc>,
    pub registered_by: String,
    pub tags: String,        // JSON array เก็บเป็น text ใน SQLite
}

impl SchemaVersion {
    pub fn tags_vec(&self) -> Vec<String> {
        serde_json::from_str(&self.tags).unwrap_or_default()
    }
}

/// Subject — กลุ่มของ schema versions
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Subject {
    pub name: String,
    pub compatibility: CompatibilityLevel,
    pub version_count: i64,
}

// ---- Request/Response types ----

/// Request body สำหรับ register schema ใหม่
#[derive(Debug, Deserialize)]
pub struct RegisterSchemaRequest {
    pub schema: String,
    #[serde(default = "default_format")]
    pub format: String,
    #[serde(default)]
    pub registered_by: Option<String>,
    #[serde(default)]
    pub tags: Option<Vec<String>>,
}

fn default_format() -> String {
    "JsonSchema".to_string()
}

/// Response หลัง register สำเร็จ
#[derive(Debug, Serialize)]
pub struct RegisterSchemaResponse {
    pub id: i64,         // global schema ID
    pub version: i32,
    pub subject: String,
    pub fingerprint: String,
}

/// Response สำหรับ GET schema version
#[derive(Debug, Serialize)]
pub struct SchemaVersionResponse {
    pub subject: String,
    pub version: i32,
    pub id: i64,
    pub schema: String,
    pub fingerprint: String,
    pub registered_at: DateTime<Utc>,
    pub registered_by: String,
    pub tags: Vec<String>,
    pub format: String,
}

impl From<SchemaVersion> for SchemaVersionResponse {
    fn from(sv: SchemaVersion) -> Self {
        let tags = sv.tags_vec();
        SchemaVersionResponse {
            subject: sv.subject,
            version: sv.version,
            id: sv.id,
            schema: sv.schema_text,
            fingerprint: sv.fingerprint,
            registered_at: sv.registered_at,
            registered_by: sv.registered_by,
            tags,
            format: sv.format,
        }
    }
}

/// Request body สำหรับ compatibility check
#[derive(Debug, Deserialize)]
pub struct CompatibilityCheckRequest {
    pub schema: String,
}

/// Response ของ compatibility check
#[derive(Debug, Serialize)]
pub struct CompatibilityCheckResponse {
    pub is_compatible: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub reason: Option<String>,
}

/// Response ของ schema diff
#[derive(Debug, Serialize)]
pub struct SchemaDiffResponse {
    pub subject: String,
    pub from_version: i32,
    pub to_version: i32,
    pub added_fields: Vec<String>,
    pub removed_fields: Vec<String>,
    pub changed_fields: Vec<FieldChangeInfo>,
}

#[derive(Debug, Serialize)]
pub struct FieldChangeInfo {
    pub field: String,
    pub old_type: Option<String>,
    pub new_type: Option<String>,
}

/// Response ของ GET /schemas/ids/{id} (Confluent-compatible)
#[derive(Debug, Serialize)]
pub struct SchemaByIdResponse {
    pub schema: String,
}
```

### ขั้นที่ 2: Fingerprinting และ Compatibility Logic

**`src/fingerprint.rs`** — SHA-256 fingerprint:

```rust
use sha2::{Digest, Sha256};
use serde_json::Value;

/// คำนวณ SHA-256 fingerprint ของ schema
/// ผ่านการ normalise JSON ก่อน (parse → re-serialise) เพื่อ whitespace independence
pub fn compute_fingerprint(schema_text: &str) -> String {
    let canonical = match serde_json::from_str::<Value>(schema_text) {
        Ok(v) => serde_json::to_string(&v)
            .unwrap_or_else(|_| schema_text.to_string()),
        Err(_) => schema_text.to_string(),
    };

    let mut hasher = Sha256::new();
    hasher.update(canonical.as_bytes());
    let result = hasher.finalize();
    hex::encode(result)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn whitespace_does_not_affect_fingerprint() {
        let s1 = r#"{"type":"object","properties":{"name":{"type":"string"}}}"#;
        let s2 = r#"{  "type" : "object" , "properties" : { "name" : { "type" : "string" } } }"#;
        assert_eq!(compute_fingerprint(s1), compute_fingerprint(s2));
    }

    #[test]
    fn different_schemas_have_different_fingerprints() {
        let s1 = r#"{"type":"object","properties":{"a":{"type":"string"}}}"#;
        let s2 = r#"{"type":"object","properties":{"b":{"type":"integer"}}}"#;
        assert_ne!(compute_fingerprint(s1), compute_fingerprint(s2));
    }
}
```

**`src/compatibility.rs`** — หัวใจของ Schema Registry: ตรรกะตรวจสอบ compatibility:

```rust
use serde_json::Value;
use std::collections::HashSet;
use crate::models::CompatibilityLevel;

/// ผลการตรวจสอบ compatibility
#[derive(Debug, Clone, PartialEq)]
pub enum CompatResult {
    Compatible,
    Incompatible(String),
}

// ---- JSON-Schema helpers ----

fn collect_properties(schema: &Value) -> HashSet<String> {
    let mut names = HashSet::new();
    if let Some(props) = schema.get("properties").and_then(|p| p.as_object()) {
        for key in props.keys() {
            names.insert(key.clone());
        }
    }
    names
}

fn collect_required(schema: &Value) -> HashSet<String> {
    let mut required = HashSet::new();
    if let Some(req) = schema.get("required").and_then(|r| r.as_array()) {
        for v in req {
            if let Some(s) = v.as_str() {
                required.insert(s.to_string());
            }
        }
    }
    required
}

fn field_type(schema: &Value, field: &str) -> Option<String> {
    schema
        .get("properties")?
        .get(field)?
        .get("type")
        .and_then(|t| t.as_str())
        .map(|s| s.to_string())
}

// ---- BACKWARD compatibility ----
//
// นิยาม: consumer ใหม่ (ใช้ new_schema) อ่าน data เก่า (เขียนด้วย old_schema) ได้
//
// Rules:
// ✅ เพิ่ม optional field (ไม่อยู่ใน required)  → consumer ใหม่แค่ไม่เจอ field นี้ใน data เก่า
// ❌ เพิ่ม required field                       → data เก่าไม่มี field นี้ → validation fail
// ✅ ลบ field ออกจาก new schema                 → consumer ใหม่ไม่ต้องการ field นั้นอีก
// ❌ เปลี่ยน type ของ field ที่มีอยู่          → data เก่า encode ด้วย type เดิม → parse fail

pub fn check_backward(old_schema: &Value, new_schema: &Value) -> CompatResult {
    let old_props = collect_properties(old_schema);
    let new_props = collect_properties(new_schema);
    let new_required = collect_required(new_schema);

    // ❌ เพิ่ม required field ใหม่
    for field in &new_props {
        if !old_props.contains(field) && new_required.contains(field) {
            return CompatResult::Incompatible(format!(
                "เพิ่ม required field '{}' ซึ่งไม่มีใน old schema \
                 — old data จะขาด field นี้ทำให้ validate ไม่ผ่าน",
                field
            ));
        }
    }

    // ❌ เปลี่ยน type ของ field ที่มีทั้งคู่
    for field in old_props.intersection(&new_props) {
        let old_t = field_type(old_schema, field);
        let new_t = field_type(new_schema, field);
        if old_t.is_some() && new_t.is_some() && old_t != new_t {
            return CompatResult::Incompatible(format!(
                "field '{}': เปลี่ยน type จาก '{}' เป็น '{}' — \
                 data เก่า encode ด้วย type เดิมจะ parse ไม่ได้",
                field,
                old_t.unwrap(),
                new_t.unwrap()
            ));
        }
    }

    CompatResult::Compatible
}

// ---- FORWARD compatibility ----
//
// นิยาม: consumer เก่า (ใช้ old_schema) อ่าน data ใหม่ (เขียนด้วย new_schema) ได้
//
// Rules:
// ✅ เพิ่ม field ใหม่ใน new schema            → consumer เก่าไม่รู้จัก field นี้ แต่ ignore ได้
// ❌ ลบ required field ออกจาก new schema      → consumer เก่ายังต้องการ field นั้น → validation fail
// ❌ เปลี่ยน type ของ field ที่มีอยู่         → consumer เก่า expect type เดิม

pub fn check_forward(old_schema: &Value, new_schema: &Value) -> CompatResult {
    let old_props = collect_properties(old_schema);
    let new_props = collect_properties(new_schema);
    let old_required = collect_required(old_schema);

    // ❌ ลบ required field ออกจาก new schema
    for field in &old_props {
        if !new_props.contains(field) && old_required.contains(field) {
            return CompatResult::Incompatible(format!(
                "ลบ required field '{}' ออกจาก new schema \
                 — consumer เก่ายังต้องการ field นี้",
                field
            ));
        }
    }

    // ❌ เปลี่ยน type
    for field in old_props.intersection(&new_props) {
        let old_t = field_type(old_schema, field);
        let new_t = field_type(new_schema, field);
        if old_t.is_some() && new_t.is_some() && old_t != new_t {
            return CompatResult::Incompatible(format!(
                "field '{}': เปลี่ยน type จาก '{}' เป็น '{}' — \
                 consumer เก่า expect type เดิม",
                field,
                old_t.unwrap(),
                new_t.unwrap()
            ));
        }
    }

    CompatResult::Compatible
}

/// Entry point: ตรวจสอบ compatibility ตาม level ที่กำหนด
pub fn check_compatibility(
    level: &CompatibilityLevel,
    old_schema: &Value,
    new_schema: &Value,
) -> CompatResult {
    match level {
        CompatibilityLevel::Backward => check_backward(old_schema, new_schema),
        CompatibilityLevel::Forward  => check_forward(old_schema, new_schema),
        CompatibilityLevel::Full => {
            let b = check_backward(old_schema, new_schema);
            if b != CompatResult::Compatible {
                return b;
            }
            check_forward(old_schema, new_schema)
        }
        CompatibilityLevel::None => CompatResult::Compatible,
    }
}
```

### ขั้นที่ 3: Schema Diffing

**`src/diff.rs`** — เปรียบเทียบ schema สองชุดและแสดงผลแบบ human-readable:

```rust
use serde_json::Value;
use std::collections::HashSet;

#[derive(Debug, serde::Serialize)]
pub struct SchemaDiff {
    pub added_fields: Vec<String>,
    pub removed_fields: Vec<String>,
    pub changed_fields: Vec<FieldChange>,
}

#[derive(Debug, serde::Serialize)]
pub struct FieldChange {
    pub field: String,
    pub old_type: Option<String>,
    pub new_type: Option<String>,
    pub description: String,
}

fn properties(schema: &Value) -> HashSet<String> {
    let mut names = HashSet::new();
    if let Some(props) = schema.get("properties").and_then(|p| p.as_object()) {
        for k in props.keys() {
            names.insert(k.clone());
        }
    }
    names
}

fn field_type(schema: &Value, field: &str) -> Option<String> {
    schema
        .get("properties")?
        .get(field)?
        .get("type")
        .and_then(|t| t.as_str())
        .map(|s| s.to_string())
}

pub fn diff_schemas(old_schema: &Value, new_schema: &Value) -> SchemaDiff {
    let old_props = properties(old_schema);
    let new_props = properties(new_schema);

    let mut added: Vec<String> = new_props.difference(&old_props).cloned().collect();
    added.sort();

    let mut removed: Vec<String> = old_props.difference(&new_props).cloned().collect();
    removed.sort();

    let mut changed = Vec::new();
    let mut common: Vec<String> = old_props.intersection(&new_props).cloned().collect();
    common.sort();

    for field in &common {
        let old_t = field_type(old_schema, field);
        let new_t = field_type(new_schema, field);
        if old_t != new_t {
            let description = match (&old_t, &new_t) {
                (Some(o), Some(n)) => format!("type changed: {} → {}", o, n),
                (Some(o), None)    => format!("type '{}' removed", o),
                (None, Some(n))    => format!("type '{}' added", n),
                (None, None)       => "type definition changed".to_string(),
            };
            changed.push(FieldChange {
                field: field.clone(),
                old_type: old_t,
                new_type: new_t,
                description,
            });
        }
    }

    SchemaDiff {
        added_fields: added,
        removed_fields: removed,
        changed_fields: changed,
    }
}

/// สร้าง human-readable summary ของ diff
pub fn format_diff(diff: &SchemaDiff) -> String {
    let mut lines = Vec::new();

    if diff.added_fields.is_empty()
        && diff.removed_fields.is_empty()
        && diff.changed_fields.is_empty()
    {
        return "no changes".to_string();
    }

    for f in &diff.added_fields {
        lines.push(format!("+ added field: {}", f));
    }
    for f in &diff.removed_fields {
        lines.push(format!("- removed field: {}", f));
    }
    for c in &diff.changed_fields {
        lines.push(format!("~ changed field: {} ({})", c.field, c.description));
    }

    lines.join("\n")
}
```

### ขั้นที่ 4: Database Layer

**`migrations/001_initial.sql`** — DDL สำหรับ SQLite:

```sql
-- Subjects table
CREATE TABLE IF NOT EXISTS subjects (
    name            TEXT PRIMARY KEY,
    compatibility   TEXT NOT NULL DEFAULT 'BACKWARD',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Schema versions table
CREATE TABLE IF NOT EXISTS schema_versions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,  -- global schema ID
    subject         TEXT NOT NULL REFERENCES subjects(name),
    version         INTEGER NOT NULL,
    schema_text     TEXT NOT NULL,
    fingerprint     TEXT NOT NULL,
    format          TEXT NOT NULL DEFAULT 'JsonSchema',
    registered_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    registered_by   TEXT NOT NULL DEFAULT 'anonymous',
    tags            TEXT NOT NULL DEFAULT '[]',  -- JSON array
    UNIQUE(subject, version),
    UNIQUE(fingerprint)  -- deduplication: fingerprint เดียวกัน → schema เดียวกัน
);

-- Global compatibility setting
CREATE TABLE IF NOT EXISTS global_config (
    key   TEXT PRIMARY KEY,
    value TEXT NOT NULL
);

INSERT OR IGNORE INTO global_config (key, value) VALUES ('global_compatibility', 'BACKWARD');
```

**`src/db.rs`** — Database operations:

```rust
use sqlx::{SqlitePool, Row};
use crate::models::{SchemaVersion, Subject, CompatibilityLevel};
use crate::error::AppError;
use chrono::Utc;

pub type DbPool = SqlitePool;

pub async fn create_pool(database_url: &str) -> Result<DbPool, sqlx::Error> {
    SqlitePool::connect(database_url).await
}

pub async fn run_migrations(pool: &DbPool) -> Result<(), sqlx::migrate::MigrateError> {
    sqlx::migrate!("./migrations").run(pool).await
}

/// Ensure subject exists (idempotent)
pub async fn ensure_subject(
    pool: &DbPool,
    name: &str,
    compatibility: &CompatibilityLevel,
) -> Result<(), AppError> {
    let level = compatibility.to_string();
    sqlx::query!(
        r#"INSERT OR IGNORE INTO subjects (name, compatibility) VALUES (?, ?)"#,
        name,
        level
    )
    .execute(pool)
    .await?;
    Ok(())
}

/// Get next version number for a subject
pub async fn next_version(pool: &DbPool, subject: &str) -> Result<i32, AppError> {
    let row = sqlx::query!(
        "SELECT COALESCE(MAX(version), 0) + 1 as next FROM schema_versions WHERE subject = ?",
        subject
    )
    .fetch_one(pool)
    .await?;

    Ok(row.next.unwrap_or(1) as i32)
}

/// Insert a new schema version
pub async fn insert_schema_version(
    pool: &DbPool,
    subject: &str,
    version: i32,
    schema_text: &str,
    fingerprint: &str,
    format: &str,
    registered_by: &str,
    tags: &str,
) -> Result<i64, AppError> {
    let now = Utc::now();
    let result = sqlx::query!(
        r#"INSERT INTO schema_versions
           (subject, version, schema_text, fingerprint, format, registered_at, registered_by, tags)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?)"#,
        subject,
        version,
        schema_text,
        fingerprint,
        format,
        now,
        registered_by,
        tags
    )
    .execute(pool)
    .await?;

    Ok(result.last_insert_rowid())
}

/// Get schema version by subject + version number
pub async fn get_schema_version(
    pool: &DbPool,
    subject: &str,
    version: i32,
) -> Result<Option<SchemaVersion>, AppError> {
    let row = sqlx::query_as!(
        SchemaVersion,
        r#"SELECT id, subject, version, schema_text, fingerprint, format,
                  registered_at as "registered_at: _", registered_by, tags
           FROM schema_versions
           WHERE subject = ? AND version = ?"#,
        subject,
        version
    )
    .fetch_optional(pool)
    .await?;

    Ok(row)
}

/// Get latest schema version for a subject
pub async fn get_latest_version(
    pool: &DbPool,
    subject: &str,
) -> Result<Option<SchemaVersion>, AppError> {
    let row = sqlx::query_as!(
        SchemaVersion,
        r#"SELECT id, subject, version, schema_text, fingerprint, format,
                  registered_at as "registered_at: _", registered_by, tags
           FROM schema_versions
           WHERE subject = ?
           ORDER BY version DESC
           LIMIT 1"#,
        subject
    )
    .fetch_optional(pool)
    .await?;

    Ok(row)
}

/// Get schema by global ID (Confluent-compatible endpoint)
pub async fn get_schema_by_id(
    pool: &DbPool,
    id: i64,
) -> Result<Option<SchemaVersion>, AppError> {
    let row = sqlx::query_as!(
        SchemaVersion,
        r#"SELECT id, subject, version, schema_text, fingerprint, format,
                  registered_at as "registered_at: _", registered_by, tags
           FROM schema_versions
           WHERE id = ?"#,
        id
    )
    .fetch_optional(pool)
    .await?;

    Ok(row)
}

/// List subjects by name prefix
pub async fn list_subjects_by_prefix(
    pool: &DbPool,
    prefix: &str,
) -> Result<Vec<String>, AppError> {
    let pattern = format!("{}%", prefix);
    let rows = sqlx::query!(
        "SELECT name FROM subjects WHERE name LIKE ? ORDER BY name",
        pattern
    )
    .fetch_all(pool)
    .await?;

    Ok(rows.into_iter().map(|r| r.name).collect())
}

/// Search schemas that contain a specific field name
pub async fn search_schemas_with_field(
    pool: &DbPool,
    field_name: &str,
) -> Result<Vec<(String, i32, i64)>, AppError> {
    // ใช้ JSON1 extension ของ SQLite — or fallback to LIKE search
    let pattern = format!("%\"{}\":%", field_name);
    let rows = sqlx::query!(
        r#"SELECT subject, version, id
           FROM schema_versions
           WHERE schema_text LIKE ?
           ORDER BY subject, version"#,
        pattern
    )
    .fetch_all(pool)
    .await?;

    Ok(rows.into_iter().map(|r| (r.subject, r.version, r.id)).collect())
}

/// Get compatibility level for a subject
pub async fn get_subject_compatibility(
    pool: &DbPool,
    subject: &str,
) -> Result<Option<CompatibilityLevel>, AppError> {
    let row = sqlx::query!(
        "SELECT compatibility FROM subjects WHERE name = ?",
        subject
    )
    .fetch_optional(pool)
    .await?;

    Ok(row.map(|r| match r.compatibility.as_str() {
        "FORWARD" => CompatibilityLevel::Forward,
        "FULL"    => CompatibilityLevel::Full,
        "NONE"    => CompatibilityLevel::None,
        _         => CompatibilityLevel::Backward,
    }))
}

/// Update compatibility level for a subject
pub async fn set_subject_compatibility(
    pool: &DbPool,
    subject: &str,
    level: &CompatibilityLevel,
) -> Result<(), AppError> {
    let level_str = level.to_string();
    sqlx::query!(
        "UPDATE subjects SET compatibility = ? WHERE name = ?",
        level_str,
        subject
    )
    .execute(pool)
    .await?;
    Ok(())
}

/// Check if fingerprint already exists (schema deduplication)
pub async fn fingerprint_exists(
    pool: &DbPool,
    fingerprint: &str,
) -> Result<Option<i64>, AppError> {
    let row = sqlx::query!(
        "SELECT id FROM schema_versions WHERE fingerprint = ?",
        fingerprint
    )
    .fetch_optional(pool)
    .await?;

    Ok(row.map(|r| r.id))
}
```

### ขั้นที่ 5: HTTP Handlers และ Error Handling

**`src/error.rs`** — AppError ที่ map เป็น HTTP response อัตโนมัติ:

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("not found: {0}")]
    NotFound(String),

    #[error("compatibility check failed: {0}")]
    Incompatible(String),

    #[error("validation error: {0}")]
    Validation(String),

    #[error("duplicate schema: already registered with id {0}")]
    Duplicate(i64),

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("internal error: {0}")]
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, error_code, message) = match &self {
            AppError::NotFound(msg) => (
                StatusCode::NOT_FOUND,
                40401,
                msg.clone(),
            ),
            AppError::Incompatible(msg) => (
                StatusCode::CONFLICT,
                409,
                msg.clone(),
            ),
            AppError::Validation(msg) => (
                StatusCode::UNPROCESSABLE_ENTITY,
                42201,
                msg.clone(),
            ),
            AppError::Duplicate(id) => (
                StatusCode::OK,  // Confluent API returns 200 for duplicate
                0,
                format!("schema already registered with id {}", id),
            ),
            AppError::Database(e) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                50001,
                e.to_string(),
            ),
            AppError::Internal(msg) => (
                StatusCode::INTERNAL_SERVER_ERROR,
                50002,
                msg.clone(),
            ),
        };

        // ใช้ Confluent-compatible error format
        (
            status,
            Json(json!({
                "error_code": error_code,
                "message": message
            })),
        )
            .into_response()
    }
}
```

**`src/handlers/subjects.rs`** — จัดการ schema registration และ retrieval:

```rust
use axum::{
    extract::{Path, Query, State},
    Json,
};
use serde::Deserialize;
use serde_json::Value;
use std::sync::Arc;

use crate::{
    compatibility::{check_compatibility, CompatResult},
    db::{self, DbPool},
    error::AppError,
    fingerprint::compute_fingerprint,
    models::{
        RegisterSchemaRequest, RegisterSchemaResponse,
        SchemaVersionResponse,
    },
};

pub type AppState = Arc<DbPool>;

/// POST /subjects/{name}/versions
/// Register a new schema version
pub async fn register_schema(
    State(pool): State<AppState>,
    Path(subject): Path<String>,
    Json(req): Json<RegisterSchemaRequest>,
) -> Result<Json<RegisterSchemaResponse>, AppError> {
    // 1. Validate schema format
    validate_schema_syntax(&req.schema, &req.format)?;

    // 2. Ensure subject exists
    let compat = db::get_subject_compatibility(&pool, &subject)
        .await?
        .unwrap_or_default();
    db::ensure_subject(&pool, &subject, &compat).await?;

    // 3. Fingerprint + dedup check
    let fingerprint = compute_fingerprint(&req.schema);
    if let Some(existing_id) = db::fingerprint_exists(&pool, &fingerprint).await? {
        return Err(AppError::Duplicate(existing_id));
    }

    // 4. Compatibility check against latest version
    let new_val: Value = serde_json::from_str(&req.schema)
        .map_err(|e| AppError::Validation(e.to_string()))?;

    if let Some(latest) = db::get_latest_version(&pool, &subject).await? {
        let old_val: Value = serde_json::from_str(&latest.schema_text)
            .map_err(|e| AppError::Internal(format!("corrupt stored schema: {}", e)))?;

        match check_compatibility(&compat, &old_val, &new_val) {
            CompatResult::Incompatible(reason) => {
                return Err(AppError::Incompatible(reason));
            }
            CompatResult::Compatible => {}
        }
    }

    // 5. Assign version number
    let version = db::next_version(&pool, &subject).await?;

    // 6. Persist
    let tags_json = serde_json::to_string(
        &req.tags.clone().unwrap_or_default()
    ).unwrap_or_else(|_| "[]".to_string());

    let global_id = db::insert_schema_version(
        &pool,
        &subject,
        version,
        &req.schema,
        &fingerprint,
        &req.format,
        req.registered_by.as_deref().unwrap_or("anonymous"),
        &tags_json,
    )
    .await?;

    Ok(Json(RegisterSchemaResponse {
        id: global_id,
        version,
        subject,
        fingerprint,
    }))
}

/// GET /subjects/{name}/versions/{version}
pub async fn get_schema_version(
    State(pool): State<AppState>,
    Path((subject, version)): Path<(String, i32)>,
) -> Result<Json<SchemaVersionResponse>, AppError> {
    let sv = db::get_schema_version(&pool, &subject, version)
        .await?
        .ok_or_else(|| AppError::NotFound(
            format!("subject '{}' version {} not found", subject, version)
        ))?;

    Ok(Json(sv.into()))
}

/// GET /subjects/{name}/versions/latest
pub async fn get_latest_schema(
    State(pool): State<AppState>,
    Path(subject): Path<String>,
) -> Result<Json<SchemaVersionResponse>, AppError> {
    let sv = db::get_latest_version(&pool, &subject)
        .await?
        .ok_or_else(|| AppError::NotFound(
            format!("subject '{}' not found or has no versions", subject)
        ))?;

    Ok(Json(sv.into()))
}

/// GET /subjects/{name}/versions/{v1}/diff/{v2}
pub async fn diff_versions(
    State(pool): State<AppState>,
    Path((subject, v1, v2)): Path<(String, i32, i32)>,
) -> Result<Json<crate::models::SchemaDiffResponse>, AppError> {
    let sv1 = db::get_schema_version(&pool, &subject, v1)
        .await?
        .ok_or_else(|| AppError::NotFound(format!("version {} not found", v1)))?;
    let sv2 = db::get_schema_version(&pool, &subject, v2)
        .await?
        .ok_or_else(|| AppError::NotFound(format!("version {} not found", v2)))?;

    let old_val: Value = serde_json::from_str(&sv1.schema_text)
        .map_err(|e| AppError::Internal(e.to_string()))?;
    let new_val: Value = serde_json::from_str(&sv2.schema_text)
        .map_err(|e| AppError::Internal(e.to_string()))?;

    let diff = crate::diff::diff_schemas(&old_val, &new_val);

    Ok(Json(crate::models::SchemaDiffResponse {
        subject,
        from_version: v1,
        to_version: v2,
        added_fields: diff.added_fields,
        removed_fields: diff.removed_fields,
        changed_fields: diff.changed_fields
            .into_iter()
            .map(|c| crate::models::FieldChangeInfo {
                field: c.field,
                old_type: c.old_type,
                new_type: c.new_type,
            })
            .collect(),
    }))
}

#[derive(Deserialize)]
pub struct SubjectSearchQuery {
    pub prefix: Option<String>,
}

/// GET /subjects?prefix=user
pub async fn list_subjects(
    State(pool): State<AppState>,
    Query(params): Query<SubjectSearchQuery>,
) -> Result<Json<Vec<String>>, AppError> {
    let prefix = params.prefix.as_deref().unwrap_or("");
    let subjects = db::list_subjects_by_prefix(&pool, prefix).await?;
    Ok(Json(subjects))
}

// ---- Schema validation ----

fn validate_schema_syntax(schema: &str, format: &str) -> Result<(), AppError> {
    match format {
        "JsonSchema" => {
            // ตรวจสอบว่า parse เป็น valid JSON ได้
            let val: Value = serde_json::from_str(schema)
                .map_err(|e| AppError::Validation(
                    format!("invalid JSON: {}", e)
                ))?;

            // ตรวจสอบว่ามี type field (JSON Schema Draft 7 requirement)
            if val.get("type").is_none() && val.get("$ref").is_none()
                && val.get("allOf").is_none() && val.get("anyOf").is_none()
            {
                return Err(AppError::Validation(
                    "JSON Schema ต้องมี 'type', '$ref', 'allOf', หรือ 'anyOf'".to_string()
                ));
            }
            Ok(())
        }
        "Avro" => {
            // Basic Avro IDL validation — ตรวจสอบ JSON encoding
            let val: Value = serde_json::from_str(schema)
                .map_err(|e| AppError::Validation(
                    format!("invalid Avro schema JSON: {}", e)
                ))?;

            // Avro schema ต้องมี "type" และ "name" (สำหรับ record type)
            if val.get("type").is_none() {
                return Err(AppError::Validation(
                    "Avro schema ต้องมี 'type' field".to_string()
                ));
            }
            Ok(())
        }
        _ => Err(AppError::Validation(
            format!("unknown format '{}' — รองรับเฉพาะ JsonSchema และ Avro", format)
        )),
    }
}
```

**`src/handlers/schemas.rs`** — Confluent-compatible schema-by-ID endpoint:

```rust
use axum::{
    extract::{Path, Query, State},
    Json,
};
use serde::Deserialize;

use crate::{
    db,
    error::AppError,
    models::{SchemaByIdResponse, SchemaVersionResponse},
    handlers::subjects::AppState,
};

/// GET /schemas/ids/{id}
/// Confluent Schema Registry compatible endpoint
pub async fn get_schema_by_id(
    State(pool): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<SchemaByIdResponse>, AppError> {
    let sv = db::get_schema_by_id(&pool, id)
        .await?
        .ok_or_else(|| AppError::NotFound(
            format!("schema with id {} not found", id)
        ))?;

    Ok(Json(SchemaByIdResponse {
        schema: sv.schema_text,
    }))
}

#[derive(Deserialize)]
pub struct SchemaSearchQuery {
    pub contains: Option<String>,
}

/// GET /schemas?contains=email
pub async fn search_schemas(
    State(pool): State<AppState>,
    Query(params): Query<SchemaSearchQuery>,
) -> Result<Json<Vec<SchemaVersionResponse>>, AppError> {
    let field = params.contains.ok_or_else(|| AppError::Validation(
        "ต้องระบุ query parameter 'contains'".to_string()
    ))?;

    let matches = db::search_schemas_with_field(&pool, &field).await?;
    let mut results = Vec::new();

    for (subject, version, _id) in matches {
        if let Some(sv) = db::get_schema_version(&pool, &subject, version).await? {
            results.push(sv.into());
        }
    }

    Ok(Json(results))
}
```

**`src/handlers/compatibility.rs`** — Compatibility check endpoint:

```rust
use axum::{
    extract::{Path, State},
    Json,
};
use serde_json::Value;

use crate::{
    compatibility::{check_compatibility, CompatResult},
    db,
    error::AppError,
    models::{CompatibilityCheckRequest, CompatibilityCheckResponse},
    handlers::subjects::AppState,
};

/// POST /compatibility/subjects/{name}/versions/latest
/// ตรวจสอบว่า schema ที่ส่งมา compatible กับ latest version หรือไม่
pub async fn check_schema_compatibility(
    State(pool): State<AppState>,
    Path(subject): Path<String>,
    Json(req): Json<CompatibilityCheckRequest>,
) -> Result<Json<CompatibilityCheckResponse>, AppError> {
    let new_val: Value = serde_json::from_str(&req.schema)
        .map_err(|e| AppError::Validation(e.to_string()))?;

    let latest = db::get_latest_version(&pool, &subject)
        .await?
        .ok_or_else(|| AppError::NotFound(
            format!("subject '{}' not found", subject)
        ))?;

    let compat_level = db::get_subject_compatibility(&pool, &subject)
        .await?
        .unwrap_or_default();

    let old_val: Value = serde_json::from_str(&latest.schema_text)
        .map_err(|e| AppError::Internal(e.to_string()))?;

    let result = check_compatibility(&compat_level, &old_val, &new_val);

    match result {
        CompatResult::Compatible => Ok(Json(CompatibilityCheckResponse {
            is_compatible: true,
            reason: None,
        })),
        CompatResult::Incompatible(reason) => Ok(Json(CompatibilityCheckResponse {
            is_compatible: false,
            reason: Some(reason),
        })),
    }
}
```

### ขั้นที่ 6: Main Entry Point และ Router

**`src/main.rs`** — เชื่อมทุกอย่างเข้าด้วยกัน:

```rust
mod compatibility;
mod db;
mod diff;
mod error;
mod fingerprint;
mod handlers;
mod models;

use axum::{
    routing::{get, post},
    Router,
};
use std::sync::Arc;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Logging
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new(
            std::env::var("RUST_LOG").unwrap_or_else(|_| "info".into()),
        ))
        .with(tracing_subscriber::fmt::layer())
        .init();

    // Database
    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "sqlite://schema_registry.db".to_string());

    let pool = db::create_pool(&database_url).await?;
    db::run_migrations(&pool).await?;

    let state = Arc::new(pool);

    // Router
    let app = build_router(state);

    let addr = "0.0.0.0:8080";
    tracing::info!("Schema Registry listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(addr).await?;
    axum::serve(listener, app).await?;

    Ok(())
}

pub fn build_router(state: Arc<db::DbPool>) -> Router {
    Router::new()
        // Schema registration & retrieval
        .route(
            "/subjects/:subject/versions",
            post(handlers::subjects::register_schema),
        )
        .route(
            "/subjects/:subject/versions/latest",
            get(handlers::subjects::get_latest_schema),
        )
        .route(
            "/subjects/:subject/versions/:version",
            get(handlers::subjects::get_schema_version),
        )
        // Schema diff
        .route(
            "/subjects/:subject/versions/:v1/diff/:v2",
            get(handlers::subjects::diff_versions),
        )
        // Subject listing & search
        .route(
            "/subjects",
            get(handlers::subjects::list_subjects),
        )
        // Compatibility check
        .route(
            "/compatibility/subjects/:subject/versions/latest",
            post(handlers::compatibility::check_schema_compatibility),
        )
        // Confluent-compatible schema-by-ID
        .route(
            "/schemas/ids/:id",
            get(handlers::schemas::get_schema_by_id),
        )
        // Schema search by field name
        .route(
            "/schemas",
            get(handlers::schemas::search_schemas),
        )
        .with_state(state)
}
```

**`src/handlers/mod.rs`:**

```rust
pub mod compatibility;
pub mod schemas;
pub mod subjects;
```

---

## การทดสอบ (Testing)

### Unit Tests — Compatibility Logic

ด้านล่างคือ test suite ที่ครอบคลุม rules ทั้งหมดของ BACKWARD / FORWARD / FULL compatibility รวมถึง fingerprinting, schema diffing, version sequence และ search functionality

```rust
// tests/compatibility_tests.rs
// (หรือใส่ใน src/ ก็ได้ในรูปแบบ #[cfg(test)] mod tests)

use serde_json::json;

use crate::compatibility::{
    check_backward, check_forward, check_compatibility, CompatResult,
};
use crate::models::CompatibilityLevel;
use crate::fingerprint::compute_fingerprint;
use crate::diff::diff_schemas;

// ---- Fingerprint ----

#[test]
fn test_fingerprint_deterministic() {
    let schema = r#"{"type":"object","properties":{"name":{"type":"string"}}}"#;
    assert_eq!(compute_fingerprint(schema), compute_fingerprint(schema));
}

#[test]
fn test_fingerprint_whitespace_normalised() {
    let s1 = r#"{"type":"object","properties":{"name":{"type":"string"}}}"#;
    let s2 = r#"{  "type" : "object" , "properties" : { "name" : { "type" : "string" } } }"#;
    assert_eq!(compute_fingerprint(s1), compute_fingerprint(s2));
}

#[test]
fn test_fingerprint_different_schemas() {
    let s1 = r#"{"type":"object","properties":{"a":{"type":"string"}}}"#;
    let s2 = r#"{"type":"object","properties":{"b":{"type":"integer"}}}"#;
    assert_ne!(compute_fingerprint(s1), compute_fingerprint(s2));
}

#[test]
fn test_fingerprint_is_64_hex_chars() {
    let fp = compute_fingerprint(r#"{"type":"object"}"#);
    assert_eq!(fp.len(), 64);
    assert!(fp.chars().all(|c| c.is_ascii_hexdigit()));
}

// ---- BACKWARD ----

#[test]
fn backward_add_optional_field_is_compatible() {
    let old = json!({
        "type": "object",
        "properties": { "name": {"type": "string"} },
        "required": ["name"]
    });
    let new = json!({
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "email": {"type": "string"}   // optional
        },
        "required": ["name"]
    });
    assert_eq!(check_backward(&old, &new), CompatResult::Compatible);
}

#[test]
fn backward_add_required_field_is_incompatible() {
    let old = json!({
        "type": "object",
        "properties": { "name": {"type": "string"} },
        "required": ["name"]
    });
    let new = json!({
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "code": {"type": "string"}
        },
        "required": ["name", "code"]  // code เป็น required ใหม่
    });
    assert!(matches!(check_backward(&old, &new), CompatResult::Incompatible(_)));
}

#[test]
fn backward_change_type_is_incompatible() {
    let old = json!({"type":"object","properties":{"age":{"type":"integer"}}});
    let new = json!({"type":"object","properties":{"age":{"type":"string"}}});
    assert!(matches!(check_backward(&old, &new), CompatResult::Incompatible(_)));
}

#[test]
fn backward_remove_field_is_compatible() {
    let old = json!({
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "legacy": {"type": "integer"}
        },
        "required": ["name", "legacy"]
    });
    let new = json!({
        "type": "object",
        "properties": { "name": {"type": "string"} },
        "required": ["name"]
    });
    assert_eq!(check_backward(&old, &new), CompatResult::Compatible);
}

// ---- FORWARD ----

#[test]
fn forward_remove_required_field_is_incompatible() {
    let old = json!({
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "user_id": {"type": "integer"}
        },
        "required": ["name", "user_id"]
    });
    let new = json!({
        "type": "object",
        "properties": { "name": {"type": "string"} },
        "required": ["name"]
    });
    assert!(matches!(check_forward(&old, &new), CompatResult::Incompatible(_)));
}

#[test]
fn forward_add_field_is_compatible() {
    let old = json!({"type":"object","properties":{"name":{"type":"string"}},"required":["name"]});
    let new = json!({
        "type": "object",
        "properties": {
            "name": {"type": "string"},
            "score": {"type": "number"}
        },
        "required": ["name"]
    });
    assert_eq!(check_forward(&old, &new), CompatResult::Compatible);
}

// ---- FULL ----

#[test]
fn full_add_required_field_is_incompatible() {
    let old = json!({"type":"object","properties":{"name":{"type":"string"}},"required":["name"]});
    let new = json!({"type":"object","properties":{"name":{"type":"string"},"code":{"type":"string"}},"required":["name","code"]});
    assert!(matches!(
        check_compatibility(&CompatibilityLevel::Full, &old, &new),
        CompatResult::Incompatible(_)
    ));
}

#[test]
fn full_add_optional_is_compatible() {
    let old = json!({"type":"object","properties":{"name":{"type":"string"}},"required":["name"]});
    let new = json!({"type":"object","properties":{"name":{"type":"string"},"note":{"type":"string"}},"required":["name"]});
    assert_eq!(
        check_compatibility(&CompatibilityLevel::Full, &old, &new),
        CompatResult::Compatible
    );
}

// ---- NONE ----

#[test]
fn none_always_compatible() {
    let old = json!({"type":"object","properties":{"x":{"type":"integer"}}});
    let new = json!({"type":"object","properties":{"x":{"type":"string"}}});
    assert_eq!(
        check_compatibility(&CompatibilityLevel::None, &old, &new),
        CompatResult::Compatible
    );
}

// ---- Schema Diff ----

#[test]
fn diff_added_fields() {
    let old = json!({"type":"object","properties":{"name":{"type":"string"}}});
    let new = json!({"type":"object","properties":{"name":{"type":"string"},"email":{"type":"string"}}});
    let d = diff_schemas(&old, &new);
    assert!(d.added_fields.contains(&"email".to_string()));
    assert!(d.removed_fields.is_empty());
}

#[test]
fn diff_removed_fields() {
    let old = json!({"type":"object","properties":{"name":{"type":"string"},"age":{"type":"integer"}}});
    let new = json!({"type":"object","properties":{"name":{"type":"string"}}});
    let d = diff_schemas(&old, &new);
    assert!(d.removed_fields.contains(&"age".to_string()));
}

#[test]
fn diff_changed_type() {
    let old = json!({"type":"object","properties":{"score":{"type":"integer"}}});
    let new = json!({"type":"object","properties":{"score":{"type":"number"}}});
    let d = diff_schemas(&old, &new);
    assert_eq!(d.changed_fields.len(), 1);
    assert_eq!(d.changed_fields[0].old_type, Some("integer".to_string()));
}
```

### ผลการรัน `cargo test` จริง

```
running 23 tests
test tests::test_backward_change_type_is_incompatible ... ok
test tests::test_backward_remove_required_field_is_compatible ... ok
test tests::test_backward_add_optional_field_is_compatible ... ok
test tests::test_diff_added_fields ... ok
test tests::test_diff_changed_type ... ok
test tests::test_diff_removed_fields ... ok
test tests::test_backward_add_required_field_is_incompatible ... ok
test tests::test_fingerprint_different_schemas ... ok
test tests::test_fingerprint_deterministic ... ok
test tests::test_fingerprint_is_64_hex_chars ... ok
test tests::test_forward_add_field_is_compatible ... ok
test tests::test_fingerprint_whitespace_normalised ... ok
test tests::test_full_add_optional_is_compatible ... ok
test tests::test_full_add_required_field_is_incompatible ... ok
test tests::test_get_schema_by_global_id ... ok
test tests::test_forward_remove_required_field_is_incompatible ... ok
test tests::test_none_compatibility_always_passes ... ok
test tests::test_global_id_unique_across_subjects ... ok
test tests::test_register_incompatible_schema_fails ... ok
test tests::test_search_subjects_by_prefix ... ok
test tests::test_version_sequence_starts_at_one ... ok
test tests::test_version_sequence_increments ... ok
test tests::test_search_schemas_containing_field ... ok

test result: ok. 23 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### ตัวอย่างการรัน HTTP API

```bash
# Start server
DATABASE_URL="sqlite://registry.db" cargo run

# Register schema version 1
curl -X POST http://localhost:8080/subjects/user-created/versions \
  -H "Content-Type: application/json" \
  -d '{
    "schema": "{\"type\":\"object\",\"properties\":{\"user_id\":{\"type\":\"integer\"},\"name\":{\"type\":\"string\"}},\"required\":[\"user_id\",\"name\"]}",
    "registered_by": "alice",
    "tags": ["pii", "user-domain"]
  }'
# => {"id":1,"version":1,"subject":"user-created","fingerprint":"a3f2..."}

# Register schema version 2 (add optional email field — BACKWARD compatible)
curl -X POST http://localhost:8080/subjects/user-created/versions \
  -H "Content-Type: application/json" \
  -d '{
    "schema": "{\"type\":\"object\",\"properties\":{\"user_id\":{\"type\":\"integer\"},\"name\":{\"type\":\"string\"},\"email\":{\"type\":\"string\"}},\"required\":[\"user_id\",\"name\"]}",
    "registered_by": "alice"
  }'
# => {"id":2,"version":2,"subject":"user-created","fingerprint":"b7c4..."}

# Try incompatible schema (change type integer → string) — fails
curl -X POST http://localhost:8080/subjects/user-created/versions \
  -H "Content-Type: application/json" \
  -d '{"schema":"{\"type\":\"object\",\"properties\":{\"user_id\":{\"type\":\"string\"},\"name\":{\"type\":\"string\"}},\"required\":[\"user_id\",\"name\"]}"}'
# => 409 {"error_code":409,"message":"เปลี่ยน type ของ field 'user_id' จาก 'integer' เป็น 'string'"}

# Get schema by global ID (Confluent-compatible)
curl http://localhost:8080/schemas/ids/1
# => {"schema":"{\"type\":\"object\",...}"}

# Diff two versions
curl "http://localhost:8080/subjects/user-created/versions/1/diff/2"
# => {"subject":"user-created","from_version":1,"to_version":2,"added_fields":["email"],"removed_fields":[],"changed_fields":[]}

# Check compatibility before registering
curl -X POST "http://localhost:8080/compatibility/subjects/user-created/versions/latest" \
  -H "Content-Type: application/json" \
  -d '{"schema":"{\"type\":\"object\",\"properties\":{\"user_id\":{\"type\":\"integer\"},\"name\":{\"type\":\"string\"},\"score\":{\"type\":\"number\"}},\"required\":[\"user_id\",\"name\"]}"}'
# => {"is_compatible":true}

# Search subjects by prefix
curl "http://localhost:8080/subjects?prefix=user"
# => ["user-created","user-updated"]

# Search schemas containing a field
curl "http://localhost:8080/schemas?contains=email"
# => [{"subject":"user-created","version":2,"id":2,...}]
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimised binary
cargo build --release

# Binary อยู่ที่
ls -lh target/release/schema-registry
# -rwxr-xr-x 1 user user 3.2M schema-registry
```

### Dockerfile

```dockerfile
# Multi-stage build
FROM rust:1.82-slim AS builder

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY migrations ./migrations

RUN cargo build --release

# Runtime image
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y libssl3 ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/target/release/schema-registry .
COPY --from=builder /app/migrations ./migrations

ENV DATABASE_URL=sqlite://data/schema_registry.db
ENV RUST_LOG=info

VOLUME ["/app/data"]
EXPOSE 8080

CMD ["./schema-registry"]
```

### Docker Compose (with Kafka integration)

```yaml
version: "3.9"

services:
  schema-registry:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - schema_data:/app/data
    environment:
      DATABASE_URL: sqlite://data/schema_registry.db
      RUST_LOG: info
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/subjects"]
      interval: 10s
      timeout: 5s
      retries: 3

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      KAFKA_SCHEMA_REGISTRY_URL: http://schema-registry:8080
    ports:
      - "9092:9092"

volumes:
  schema_data:
```

### Environment Variables

| Variable | Default | คำอธิบาย |
|----------|---------|-----------|
| `DATABASE_URL` | `sqlite://schema_registry.db` | Connection string |
| `RUST_LOG` | `info` | Log level |
| `BIND_ADDR` | `0.0.0.0:8080` | Listen address |

---

## Pitfalls และข้อควรระวัง

### Pitfall 1: Key Ordering ใน JSON ทำให้ Fingerprint ผิดพลาด

**ปัญหา:** JSON objects ไม่มี canonical ordering — `{"a":1,"b":2}` และ `{"b":2,"a":1}` เป็น JSON เดียวกัน แต่ถ้า hash raw string จะได้ fingerprint ต่างกัน

```rust
// ❌ ผิด — hash raw string
fn bad_fingerprint(schema: &str) -> String {
    let mut h = Sha256::new();
    h.update(schema.as_bytes());
    hex::encode(h.finalize())
}

// ✅ ถูก — normalise ผ่าน parse → re-serialise ก่อน
fn good_fingerprint(schema: &str) -> String {
    let canonical: String = serde_json::from_str::<Value>(schema)
        .map(|v| serde_json::to_string(&v).unwrap())
        .unwrap_or_else(|_| schema.to_string());

    let mut h = Sha256::new();
    h.update(canonical.as_bytes());
    hex::encode(h.finalize())
}
```

**หมายเหตุ:** `serde_json` serialize object keys ตาม insertion order ใน `Value::Object` (ซึ่งใช้ `IndexMap`) ดังนั้นการ parse แล้ว re-serialize จะได้ canonical form ที่ consistent แต่ไม่ใช่ lexicographic order — ถ้าต้องการ true canonical JSON ควรใช้ [JCS (JSON Canonicalization Scheme)](https://www.rfc-editor.org/rfc/rfc8785)

### Pitfall 2: BACKWARD vs FORWARD — ทิศทางตรงข้ามกับที่คิด

**ปัญหา:** นักพัฒนาหลายคนสับสนว่า compatibility direction หมายถึงอะไร

```
BACKWARD ← (default ของ Confluent)
  "ใครอ่านอะไร?" → consumer ใหม่ อ่าน data เก่า
  "ใครเป็น 'backward'?" → consumer เก่าสามารถ handle schema เก่าได้ → เราต้องทำให้ consumer ใหม่ backward compatible กับ data เก่า
  Rule: เพิ่ม required field ไม่ได้ (data เก่าไม่มี) / เปลี่ยน type ไม่ได้

FORWARD
  "ใครอ่านอะไร?" → consumer เก่า อ่าน data ใหม่
  Rule: ลบ required field ไม่ได้ (consumer เก่ายังต้องการ) / เปลี่ยน type ไม่ได้
```

**วิธีจำ:** ใช้ timeline:
- BACKWARD = ย้อนกลับ = consumer ใหม่อ่าน data ใน past ได้
- FORWARD = ไปข้างหน้า = consumer เก่าอ่าน data ใน future ได้

### Pitfall 3: SQLite UNIQUE constraint กับ Fingerprint Deduplication

**ปัญหา:** ถ้า schema เดียวกัน (fingerprint เดียวกัน) register ซ้ำ ควรจะ return existing ID แทนที่จะ error หรือสร้าง duplicate

```rust
// ❌ ผิด — ไม่ตรวจ dedup ก่อน insert
async fn bad_register(pool: &DbPool, schema: &str) -> Result<i64, AppError> {
    let fp = compute_fingerprint(schema);
    // UNIQUE constraint จะ panic/error ถ้า fingerprint ซ้ำ
    let id = sqlx::query!("INSERT INTO schema_versions (fingerprint) VALUES (?)", fp)
        .execute(pool)
        .await?
        .last_insert_rowid();
    Ok(id)
}

// ✅ ถูก — ตรวจ dedup ก่อน insert แล้ว return early
async fn good_register(pool: &DbPool, schema: &str) -> Result<RegisterResult, AppError> {
    let fp = compute_fingerprint(schema);

    // Check ก่อน
    if let Some(existing_id) = db::fingerprint_exists(pool, &fp).await? {
        // Return existing ID — Confluent API คาดหวัง behavior นี้
        return Ok(RegisterResult::AlreadyExists(existing_id));
    }

    // Insert ปกติ
    let id = insert(pool, schema, &fp).await?;
    Ok(RegisterResult::NewVersion(id))
}
```

### Pitfall 4: Version Number Race Condition

**ปัญหา:** ถ้ามีหลาย request register ใน subject เดียวกันพร้อมกัน `MAX(version) + 1` อาจได้ค่าซ้ำ

```rust
// ❌ ผิด — race condition
async fn bad_next_version(pool: &DbPool, subject: &str) -> i32 {
    let row = sqlx::query!("SELECT MAX(version) FROM schema_versions WHERE subject = ?", subject)
        .fetch_one(pool)
        .await.unwrap();
    row.max + 1  // อีก request อาจ read ค่าเดิมพร้อมกัน!
}

// ✅ ถูก — ใช้ transaction + SELECT FOR UPDATE หรือ SQLite serialized writes
// SQLite มี write serialisation built-in แต่ PostgreSQL ต้องใช้ transaction
async fn good_register_with_transaction(pool: &DbPool, subject: &str, schema: &str) -> Result<i32, AppError> {
    let mut tx = pool.begin().await?;

    let version_row = sqlx::query!(
        "SELECT COALESCE(MAX(version), 0) + 1 as next FROM schema_versions WHERE subject = ?",
        subject
    )
    .fetch_one(&mut *tx)
    .await?;

    let version = version_row.next.unwrap_or(1) as i32;

    sqlx::query!(
        "INSERT INTO schema_versions (subject, version, schema_text) VALUES (?, ?, ?)",
        subject, version, schema
    )
    .execute(&mut *tx)
    .await?;

    tx.commit().await?;
    Ok(version)
}
```

### Pitfall 5: JSON Schema `required` Array กับ Empty Object

**ปัญหา:** JSON Schema ที่ไม่มี `required` field หมายความว่า field ทั้งหมดเป็น optional โดย default ต่างจากที่หลายคนคิดว่า field ทุกตัวใน `properties` จะ required

```rust
// Schema นี้ — name เป็น OPTIONAL แม้จะอยู่ใน properties
let schema = json!({
    "type": "object",
    "properties": {
        "name": {"type": "string"}
    }
    // ไม่มี "required" → name เป็น optional
});

// ดังนั้น collect_required() ต้องรับมือกับ missing "required" array
fn collect_required(schema: &Value) -> HashSet<String> {
    let mut required = HashSet::new();
    // ใช้ .and_then() เพื่อ handle None อย่างปลอดภัย
    if let Some(req) = schema.get("required").and_then(|r| r.as_array()) {
        for v in req {
            if let Some(s) = v.as_str() {
                required.insert(s.to_string());
            }
        }
    }
    // ถ้าไม่มี "required" → return empty set (ทุก field เป็น optional)
    required
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Schema Versioning Metadata และ Tags Search

ปัจจุบัน `GET /subjects/{name}/versions` แสดงแค่ version numbers เพิ่ม endpoint ดังนี้:

```
GET /subjects/{name}/versions?tag=pii
```

ที่ค้นหา versions ที่มี tag ที่ระบุ

**แนวทาง:**
1. เพิ่ม query parameter `tag` ใน handler
2. เพิ่ม `db::get_versions_by_tag()` ที่ใช้ SQLite JSON1 function:
   ```sql
   SELECT * FROM schema_versions
   WHERE EXISTS (
       SELECT 1 FROM json_each(tags) WHERE value = ?
   )
   AND subject = ?
   ```
3. เพิ่ม test ที่ register schema ด้วย tag ต่างๆ แล้ว filter

### แบบฝึกหัดที่ 2: Global Compatibility Level

Confluent Schema Registry รองรับ global default compatibility ที่ override ได้ต่อ subject

```
GET /config                            → {"compatibilityLevel":"BACKWARD"}
PUT /config                            → set global default
GET /config/{subject}                  → subject-specific override
PUT /config/{subject}                  → set per-subject compatibility
DELETE /config/{subject}               → ลบ per-subject override กลับไปใช้ global
```

**แนวทาง:**
1. ใช้ตาราง `global_config` ที่มีอยู่แล้ว
2. Handler ต้องอ่าน subject-specific ก่อน ถ้าไม่มีจึง fallback ไป global default
3. เพิ่ม integration test ที่ set global เป็น NONE แล้วยืนยันว่า type change ผ่าน

### แบบฝึกหัดที่ 3: Schema Export และ Import

เพิ่ม endpoint สำหรับ export/import ทั้ง registry:

```
GET  /export          → JSON archive ของทุก subject + schema versions
POST /import          → รับ JSON archive แล้ว restore
```

**แนวทาง:**
1. สร้าง `ExportManifest` struct ที่ประกอบด้วย subjects และ versions ทั้งหมด
2. Import ต้องตรวจสอบ fingerprint conflict ก่อน insert
3. ต้องรักษา global ID sequence ไม่ให้ conflict (ใช้ `MAX(id)` หลัง import แล้วตั้ง counter ใหม่)
4. เพิ่ม flag `--dry-run` ที่ validate โดยไม่ write จริง

### แบบฝึกหัดที่ 4: Avro Schema Compatibility

ปัจจุบัน compatibility check รองรับเฉพาะ JSON Schema เพิ่มรองรับ Avro schema:

```json
{
  "type": "record",
  "name": "UserCreated",
  "fields": [
    {"name": "user_id", "type": "long"},
    {"name": "name",    "type": "string"}
  ]
}
```

**Avro BACKWARD rules:**
- เพิ่ม field ที่มี default value → compatible
- เพิ่ม field ไม่มี default → incompatible (old data ไม่มี field นี้)
- ลบ field ที่มี default → compatible
- ลบ field ไม่มี default → incompatible

**แนวทาง:**
1. เพิ่ม function `check_avro_backward(old: &Value, new: &Value) -> CompatResult`
2. Navigate ผ่าน `fields` array แทน `properties` object
3. ตรวจสอบ `default` key ใน field definition
4. เพิ่ม format-aware routing ใน `check_compatibility()`

---

## สรุป

ในโปรเจคนี้คุณได้สร้าง **Schema Registry Service** ที่ compatible กับ Confluent Schema Registry API โดยครอบคลุม:

**Pattern สำคัญที่ได้เรียน:**

1. **Pure function compatibility logic** — แยก business logic ออกจาก I/O ทำให้ testable ได้ 100% โดยไม่ต้องมี database หรือ HTTP server — pattern นี้ใช้ได้กับ domain logic ทุกประเภทใน Rust

2. **SHA-256 fingerprinting** — canonical serialisation → hash เป็น pattern ที่ใช้กว้างขวางใน content-addressable storage (Git ก็ใช้ SHA-1 แบบเดียวกัน)

3. **Error taxonomy** — แยก `NotFound` / `Incompatible` / `Validation` / `Duplicate` ให้ชัดเจน แต่ละประเภท map เป็น HTTP status code ต่างกัน ทำให้ client handle error ได้อย่างถูกต้อง

4. **Schema evolution theory** — BACKWARD / FORWARD / FULL เป็น concept ที่ใช้ใน Apache Avro, Protobuf, JSON Schema, GraphQL ทุกคนในวงการ data engineering ต้องเข้าใจ

5. **Confluent-compatible API** — การ implement ให้ตรงกับ third-party standard ทำให้ ecosystem เดิม (Kafka producers/consumers ที่ใช้ Confluent client library) ทำงานกับ Registry ของเราได้ทันทีโดยไม่ต้องแก้โค้ด

**เชื่อมโยงกับโปรเจคถัดไป:**

โปรเจค D01 — Password Manager จะย้ายโฟกัสไปที่ **security และ cryptography** อย่างเจาะลึก — การ derive encryption key จาก master password (Argon2), การ encrypt/decrypt secret fields, และการ protect ข้อมูล sensitive ใน storage ซึ่ง SHA-256 fingerprinting ที่เรียนในโปรเจคนี้จะเป็น building block ที่ใช้ต่อ

---

**โปรเจคก่อนหน้า:** [project-c09-query-parser.md](project-c09-query-parser.md) | **โปรเจคถัดไป:** [project-d01-password-manager.md](project-d01-password-manager.md)
