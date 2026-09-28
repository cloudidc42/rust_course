# Project G05: Configuration Manager

> โมดูล: G — DevOps/Infra | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 10 ชั่วโมง

## ภาพรวมโปรเจค

ทุก service ที่ทำงานใน production ต้องการระบบ configuration ที่ยืดหยุ่น — อ่านจากหลายแหล่ง, แต่ละ environment (dev/staging/prod) ใช้ค่าต่างกัน, secrets ต้องซ่อนไม่ให้ log รั่วออกไป, และเมื่อ deploy ใหม่โดยไม่รีสตาร์ท service ระบบควร reload ค่าใหม่ได้ทันที

**Configuration Manager** ในโปรเจคนี้แก้ปัญหาทั้งหมดนั้นโดยสร้างระบบแบบ **layered** — หลาย source ถูก merge กันตามลำดับ priority เหมือนกับที่ Spring Boot, Viper (Go), และ dotenv ทำ:

```
Priority ต่ำ ──────────────────────────────── Priority สูง
    ┌──────────┐     ┌──────────┐     ┌──────────┐
    │ defaults │  ←  │  file    │  ←  │  env     │
    │ (code)   │     │ (TOML)   │     │  vars    │
    └──────────┘     └──────────┘     └──────────┘
                              ↓
                     merged config (dot-notation access)
```

**Use case จริงใน production:**

- **12-factor app** — environment variables สำหรับ secrets, ไฟล์สำหรับ non-sensitive config
- **Hot reload** — เปลี่ยน log level หรือ feature flag โดยไม่รีสตาร์ท service
- **Secret masking** — log dump ของ config ไม่รั่ว password หรือ API token ออกมา
- **Schema validation** — fail fast เมื่อ config ผิดรูปแบบ แทนที่จะ panic ตอน runtime

**What we build:** library แบบ production-grade พร้อม 41 tests ที่ผ่านจริง ครอบคลุม:

| Feature | ที่ใช้ |
|---|---|
| `ConfigSource` trait | TOML, JSON, EnvVar, Defaults |
| Layered merge | deep merge recursive, dot-notation |
| Type-safe access | `get::<T>(path)` ด้วย serde |
| Schema validation | required keys, type, range |
| Hot reload | `tokio::sync::watch` channel |
| Secret masking | `SecretString` newtype, pattern matching |

---

## สิ่งที่จะได้เรียนรู้

- **`trait` objects แบบ `dyn Trait`** — `Box<dyn ConfigSource>` เก็บ sources ต่างชนิดใน `Vec` เดียว
- **Newtype pattern สำหรับ safety** — `SecretString` ที่ override `Debug`/`Display` ป้องกัน accidental logging
- **serde generics** — `get::<T: DeserializeOwned>()` แปลง `serde_json::Value` เป็น arbitrary type ที่ compile time
- **`tokio::sync::watch`** — broadcast config changes ไปยัง subscribers หลายตัวพร้อมกัน
- **Builder pattern** — `.add_source().add_source().build()` ที่อ่านง่ายและ immutable ก่อน finalize
- **Recursive data structures** — deep merge ของ nested JSON/TOML maps
- **`std::any::type_name::<T>()`** — human-readable error messages สำหรับ type mismatches
- **Test isolation สำหรับ env vars** — inject vars ผ่าน `with_vars()` แทนการ mutate global state

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 11-20** — Traits, trait objects (`dyn Trait`), generics
- **Part 21-30** — Error handling, `Result<T, E>`, `From`/`Into`
- **Part 31-40** — Closures, iterators, `Box<T>`, `Arc<T>`
- **Part 41-50** — Async/await, tokio runtime (สำหรับ hot reload)
- **Part 51-60** — `serde`, derive macros, `Deserialize`
- **Part 96-110** — Production patterns: custom error types, library design
- โปรเจค G04 (Metrics Collector) — ความเข้าใจ trait-based design และ tokio

---

## โครงสร้างโปรเจค (Project Layout)

```
config-manager/
├── src/
│   ├── lib.rs          # Public API — re-exports
│   ├── error.rs        # ConfigError, ValidationError
│   ├── value.rs        # deep_merge(), get_by_path() utilities
│   ├── source.rs       # ConfigSource trait + DefaultsSource, FileSource, EnvSource
│   ├── layered.rs      # LayeredConfig — builder + merge + typed get
│   ├── schema.rs       # ConfigSchema, FieldSchema, TypeConstraint
│   ├── secret.rs       # SecretString newtype + mask_secrets()
│   ├── watch.rs        # ConfigWatcher + config_diff()
│   └── main.rs         # Demo binary
├── tests/
│   └── integration_tests.rs
└── Cargo.toml
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
┌───────────────────────────────────────────────────────────┐
│                    LayeredConfig                          │
│                                                           │
│  sources: Vec<Box<dyn ConfigSource>>                      │
│     [0] DefaultsSource  ──┐                               │
│     [1] FileSource(TOML)  ├──→ deep_merge() ──→ merged   │
│     [2] EnvSource         ┘         (Map<String, Value>)  │
└──────────────────────────────────────────┬────────────────┘
                                           │
                                  get::<T>("dot.path")
                                           │
                              ┌────────────▼────────────┐
                              │   serde_json::from_value │
                              │   → T: DeserializeOwned  │
                              └──────────────────────────┘
```

### Source Priority

Sources ถูกเพิ่มทีละตัวด้วย `.add_source()` และ merge จาก index 0 ไป index สุดท้าย แต่ละ source จะ **override** ค่าที่ source ก่อนหน้าตั้งไว้:

```
merged = {}
for source in sources:             # [defaults, file, env]
    deep_merge(&mut merged, layer)  # ทีหลัง override ทีแรก
```

ดังนั้น sources ที่ add **ทีหลัง** มี priority **สูงกว่า** — convention มาตรฐานคือ `defaults → file → env`

### deep_merge vs. shallow merge

**Shallow merge** (ทับทั้ง key):

```
base:    { server: { host: "a", port: 80 } }
overlay: { server: { port: 90 } }
result:  { server: { port: 90 } }   ← host หาย!
```

**Deep merge** (เข้าไป merge ข้างใน object):

```
base:    { server: { host: "a", port: 80 } }
overlay: { server: { port: 90 } }
result:  { server: { host: "a", port: 90 } }  ← host ยังอยู่
```

โปรเจคนี้ใช้ deep merge เสมอ ทำให้ file source สามารถ override แค่ค่าที่ต้องการโดยไม่ต้องระบุทุก field

### Internal Type: serde_json::Value

เหตุผลที่ใช้ `serde_json::Value` เป็น internal representation:
1. **Uniform API** — TOML, JSON, env vars ทั้งหมด convert มาเป็น `serde_json::Value` ก่อน merge
2. **serde integration** — `from_value::<T>()` แปลงเป็น arbitrary type ที่ implement `Deserialize` ได้ทันที
3. **No dependencies ซ้อน** — `serde_json` ถูกใช้อยู่แล้ว ไม่ต้องเพิ่ม crate แยก
4. **JSON pointer compatible** — dot-notation เทียบเท่า JSON pointer แบบ simplified

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Dependencies

สร้างโปรเจคใหม่ด้วย `cargo new config-manager --lib`:

**`Cargo.toml`:**

```toml
[package]
name = "config-manager"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio  = { version = "1", features = ["full"] }
serde  = { version = "1", features = ["derive"] }
serde_json = "1"
toml   = "0.8"

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

เหตุผลการเลือก crate:
- **`tokio`** — async runtime สำหรับ `watch` channel ใน hot reload
- **`serde` + `serde_json`** — de/serialization และ type-safe access ผ่าน `from_value::<T>()`
- **`toml`** — parse TOML config files ด้วย serde backend
- ไม่ใช้ `serde_yaml` ในเวอร์ชันนี้ (ดู Extensions) เพื่อไม่เพิ่ม dependency โดยไม่จำเป็น
- ไม่ใช้ `notify` crate (ดู Pitfall 4) — ใช้ polling หรือ broadcast แทน

---

### ขั้นที่ 2: Error Types

**`src/error.rs`** — custom error type ที่ cover ทุกกรณีที่ config อาจผิดพลาด:

```rust
use std::fmt;

/// ประเภทข้อผิดพลาดหลักของระบบ Configuration
#[derive(Debug)]
pub enum ConfigError {
    /// ไม่พบ key ที่ต้องการ
    KeyNotFound(String),
    /// ชนิดข้อมูลไม่ตรง
    TypeMismatch {
        key: String,
        expected: String,
        found: String,
    },
    /// แปลงค่าไม่ได้
    ParseError { key: String, source: String },
    /// IO error (เช่น อ่านไฟล์ไม่ได้)
    IoError(std::io::Error),
    /// ผิดกฎ schema validation
    ValidationError(Vec<ValidationError>),
    /// ข้อผิดพลาดจาก source (TOML/JSON parse error)
    SourceError(String),
}

/// ข้อผิดพลาดจาก schema validation — มี field path บอกตำแหน่ง
#[derive(Debug, Clone)]
pub struct ValidationError {
    pub field: String,
    pub message: String,
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::KeyNotFound(key) => write!(f, "key not found: '{key}'"),
            ConfigError::TypeMismatch { key, expected, found } => {
                write!(f, "type mismatch for '{key}': expected {expected}, found {found}")
            }
            ConfigError::ParseError { key, source } => {
                write!(f, "parse error for '{key}': {source}")
            }
            ConfigError::IoError(e) => write!(f, "IO error: {e}"),
            ConfigError::ValidationError(errs) => {
                write!(f, "validation failed ({} error(s)): ", errs.len())?;
                for e in errs {
                    write!(f, "[{}: {}] ", e.field, e.message)?;
                }
                Ok(())
            }
            ConfigError::SourceError(s) => write!(f, "source error: {s}"),
        }
    }
}

impl std::error::Error for ConfigError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            ConfigError::IoError(e) => Some(e),
            _ => None,
        }
    }
}

impl From<std::io::Error> for ConfigError {
    fn from(e: std::io::Error) -> Self {
        ConfigError::IoError(e)
    }
}
```

**Design decision:** `ValidationError` เป็น `Vec` ใน `ConfigError::ValidationError` เพื่อรวบรวม **ทุก error** ในการ validate ครั้งเดียว แทนที่จะ fail-fast ที่ error แรก — ทำให้ developer เห็นปัญหาทั้งหมดพร้อมกัน ไม่ต้อง run-fix-run-fix วนซ้ำ

---

### ขั้นที่ 3: Value Utilities — deep_merge และ get_by_path

**`src/value.rs`:**

```rust
use serde_json::{Map, Value};

/// Deep merge overlay เข้าไปใน base
/// ถ้าทั้งคู่เป็น object → merge recursive
/// อื่น ๆ → overlay ชนะ (replace)
pub fn deep_merge(base: &mut Map<String, Value>, overlay: &Map<String, Value>) {
    for (key, val) in overlay {
        match base.get_mut(key) {
            Some(base_val) if base_val.is_object() && val.is_object() => {
                let base_obj = base_val.as_object_mut().unwrap();
                let overlay_obj = val.as_object().unwrap();
                deep_merge(base_obj, overlay_obj);
            }
            _ => {
                base.insert(key.clone(), val.clone());
            }
        }
    }
}

/// ค้นหา value ด้วย dot-notation path
/// เช่น "server.tls.enabled" → map["server"]["tls"]["enabled"]
pub fn get_by_path<'a>(map: &'a Map<String, Value>, path: &str) -> Option<&'a Value> {
    let (head, tail) = match path.find('.') {
        Some(pos) => (&path[..pos], Some(&path[pos + 1..])),
        None => (path, None),
    };

    let val = map.get(head)?;

    match tail {
        Some(rest) => {
            if let Some(obj) = val.as_object() {
                get_by_path(obj, rest)
            } else {
                None
            }
        }
        None => Some(val),
    }
}
```

**ตัวอย่างการใช้งาน:**

```rust
use serde_json::{Map, Value, json};

let mut base: Map<String, Value> = serde_json::from_value(json!({
    "db": { "host": "localhost", "port": 5432 }
})).unwrap();

let overlay: Map<String, Value> = serde_json::from_value(json!({
    "db": { "port": 5433 }  // override แค่ port
})).unwrap();

deep_merge(&mut base, &overlay);
// ผล: { "db": { "host": "localhost", "port": 5433 } }

let port = get_by_path(&base, "db.port"); // Some(5433)
let host = get_by_path(&base, "db.host"); // Some("localhost")
let missing = get_by_path(&base, "db.timeout"); // None
```

---

### ขั้นที่ 4: ConfigSource Trait และ Source Implementations

**`src/source.rs`** — trait + สาม implementations:

```rust
use serde_json::{Map, Value};
use crate::error::ConfigError;

pub type ConfigMap = Map<String, Value>;

/// Trait หลักสำหรับทุก config source
pub trait ConfigSource: Send + Sync {
    fn load(&self) -> Result<ConfigMap, ConfigError>;
    fn name(&self) -> &str;
}
```

**DefaultsSource** — ค่า default ที่ hardcode ไว้ใน code:

```rust
pub struct DefaultsSource {
    defaults: ConfigMap,
    label: String,
}

impl DefaultsSource {
    pub fn new(defaults: ConfigMap) -> Self {
        Self { defaults, label: "defaults".into() }
    }

    pub fn with_label(mut self, label: impl Into<String>) -> Self {
        self.label = label.into();
        self
    }
}

impl ConfigSource for DefaultsSource {
    fn load(&self) -> Result<ConfigMap, ConfigError> {
        Ok(self.defaults.clone())
    }
    fn name(&self) -> &str { &self.label }
}
```

**FileSource** — อ่านจากไฟล์ TOML หรือ JSON:

```rust
pub enum FileFormat { Toml, Json }

pub struct FileSource {
    path: String,
    format: FileFormat,
}

impl FileSource {
    pub fn toml(path: impl Into<String>) -> Self {
        Self { path: path.into(), format: FileFormat::Toml }
    }
    pub fn json(path: impl Into<String>) -> Self {
        Self { path: path.into(), format: FileFormat::Json }
    }
}

impl ConfigSource for FileSource {
    fn load(&self) -> Result<ConfigMap, ConfigError> {
        let content = std::fs::read_to_string(&self.path)?;
        let val: Value = match self.format {
            FileFormat::Toml => toml::from_str(&content)
                .map_err(|e| ConfigError::SourceError(e.to_string()))?,
            FileFormat::Json => serde_json::from_str(&content)
                .map_err(|e| ConfigError::SourceError(e.to_string()))?,
        };
        val.as_object()
            .cloned()
            .ok_or_else(|| ConfigError::SourceError("config root must be a map/table".into()))
    }
    fn name(&self) -> &str { &self.path }
}
```

**EnvSource** — อ่าน environment variables พร้อม prefix stripping และ `__` → nested key:

```
APP_HOST              → host
APP_SERVER__PORT      → server.port
APP_DATABASE__TLS__ON → database.tls.on
```

```rust
pub struct EnvSource {
    prefix: String,
    vars_override: Option<Vec<(String, String)>>,
}

impl EnvSource {
    /// Production: อ่านจาก process environment จริง
    pub fn new(prefix: impl Into<String>) -> Self {
        Self { prefix: prefix.into().to_uppercase(), vars_override: None }
    }

    /// Testing: inject vars โดยตรงโดยไม่ต้อง mutate global state
    pub fn with_vars(prefix: impl Into<String>, vars: Vec<(String, String)>) -> Self {
        Self { prefix: prefix.into().to_uppercase(), vars_override: Some(vars) }
    }

    fn iter_vars(&self) -> Box<dyn Iterator<Item = (String, String)> + '_> {
        match &self.vars_override {
            Some(v) => Box::new(v.clone().into_iter()),
            None    => Box::new(std::env::vars()),
        }
    }
}

impl ConfigSource for EnvSource {
    fn load(&self) -> Result<ConfigMap, ConfigError> {
        let mut result = Map::new();
        let prefix_with_sep = format!("{}_", self.prefix);

        for (env_key, env_val) in self.iter_vars() {
            if !env_key.starts_with(&prefix_with_sep) { continue; }
            let stripped = env_key[prefix_with_sep.len()..].to_string();
            let parts: Vec<String> = stripped
                .split("__")
                .map(|s| s.to_lowercase())
                .collect();
            insert_nested(&mut result, &parts, Value::String(env_val));
        }
        Ok(result)
    }
    fn name(&self) -> &str { &self.prefix }
}

fn insert_nested(map: &mut Map<String, Value>, parts: &[String], val: Value) {
    if parts.is_empty() { return; }
    if parts.len() == 1 {
        map.insert(parts[0].clone(), val);
        return;
    }
    let key = parts[0].clone();
    let entry = map.entry(key).or_insert_with(|| Value::Object(Map::new()));
    if let Some(nested) = entry.as_object_mut() {
        insert_nested(nested, &parts[1..], val);
    }
}
```

**ทดสอบ EnvSource:**

```rust
let source = EnvSource::with_vars("APP", vec![
    ("APP_HOST".into(), "prod.example.com".into()),
    ("APP_SERVER__PORT".into(), "443".into()),
    ("UNRELATED".into(), "ignored".into()),
]);

let map = source.load().unwrap();
assert_eq!(map["host"], json!("prod.example.com"));
// map["server"]["port"] == "443"
```

---

### ขั้นที่ 5: LayeredConfig — Builder + Typed Access

**`src/layered.rs`:**

```rust
use serde::de::DeserializeOwned;
use serde_json::{Map, Value};
use crate::error::ConfigError;
use crate::source::ConfigSource;
use crate::value::{deep_merge, get_by_path};

pub struct LayeredConfig {
    sources: Vec<Box<dyn ConfigSource>>,
    merged: Map<String, Value>,
}

impl LayeredConfig {
    pub fn new() -> Self {
        Self { sources: Vec::new(), merged: Map::new() }
    }

    /// เพิ่ม source — sources ที่เพิ่มทีหลัง = priority สูงกว่า
    pub fn add_source(mut self, source: impl ConfigSource + 'static) -> Self {
        self.sources.push(Box::new(source));
        self
    }

    /// Finalize และ merge ทุก source
    pub fn build(mut self) -> Result<Self, ConfigError> {
        self.reload()?;
        Ok(self)
    }

    /// Reload ทุก source และ re-merge (ใช้สำหรับ hot reload)
    pub fn reload(&mut self) -> Result<(), ConfigError> {
        let mut merged = Map::new();
        for source in &self.sources {
            let layer = source.load()?;
            deep_merge(&mut merged, &layer);
        }
        self.merged = merged;
        Ok(())
    }

    /// ดึงค่าด้วย dot-notation path และแปลงเป็น type ที่ต้องการ
    ///
    /// ```ignore
    /// let port: u16 = config.get("server.port")?;
    /// let origins: Vec<String> = config.get("allowed_origins")?;
    /// let enabled: bool = config.get("feature.dark_mode")?;
    /// ```
    pub fn get<T: DeserializeOwned>(&self, path: &str) -> Result<T, ConfigError> {
        let val = get_by_path(&self.merged, path)
            .ok_or_else(|| ConfigError::KeyNotFound(path.to_string()))?;

        serde_json::from_value(val.clone()).map_err(|e| ConfigError::TypeMismatch {
            key: path.to_string(),
            expected: std::any::type_name::<T>().to_string(),
            found: e.to_string(),
        })
    }

    /// ดึง raw JSON value โดยไม่แปลง type
    pub fn get_raw(&self, path: &str) -> Option<&Value> {
        get_by_path(&self.merged, path)
    }

    pub fn merged(&self) -> &Map<String, Value> { &self.merged }
}
```

**ตัวอย่างการประกอบ config แบบสมบูรณ์:**

```rust
use config_manager::*;
use serde_json::json;

let defaults: serde_json::Map<_, _> = serde_json::from_value(json!({
    "server": { "host": "127.0.0.1", "port": 8080, "workers": 4 },
    "log":    { "level": "info", "format": "json" }
})).unwrap();

let config = LayeredConfig::new()
    .add_source(DefaultsSource::new(defaults))
    .add_source(FileSource::toml("config/app.toml"))    // override defaults
    .add_source(EnvSource::new("APP"))                   // override file
    .build()?;

let port:  u16    = config.get("server.port")?;
let level: String = config.get("log.level")?;
let workers: u32  = config.get("server.workers")?;
```

---

### ขั้นที่ 6: Schema Validation

**`src/schema.rs`** — validate config ก่อน service เริ่ม:

```rust
use serde_json::Value;
use crate::error::{ConfigError, ValidationError};
use crate::layered::LayeredConfig;

#[derive(Debug, Clone)]
pub enum TypeConstraint {
    String, Integer, Float, Bool, Array, Object,
}

#[derive(Debug, Clone)]
pub struct FieldSchema {
    pub key: String,
    pub required: bool,
    pub type_constraint: Option<TypeConstraint>,
    pub min: Option<f64>,
    pub max: Option<f64>,
}

impl FieldSchema {
    pub fn required(key: impl Into<String>) -> Self {
        Self { key: key.into(), required: true,
               type_constraint: None, min: None, max: None }
    }
    pub fn optional(key: impl Into<String>) -> Self {
        Self { key: key.into(), required: false,
               type_constraint: None, min: None, max: None }
    }
    pub fn with_type(mut self, tc: TypeConstraint) -> Self {
        self.type_constraint = Some(tc); self
    }
    pub fn with_range(mut self, min: f64, max: f64) -> Self {
        self.min = Some(min); self.max = Some(max); self
    }
}

pub struct ConfigSchema { fields: Vec<FieldSchema> }

impl ConfigSchema {
    pub fn new() -> Self { Self { fields: Vec::new() } }

    pub fn field(mut self, f: FieldSchema) -> Self {
        self.fields.push(f); self
    }

    pub fn validate(&self, config: &LayeredConfig) -> Result<(), ConfigError> {
        let mut errors = Vec::new();
        for field in &self.fields {
            match config.get_raw(&field.key) {
                None => {
                    if field.required {
                        errors.push(ValidationError {
                            field: field.key.clone(),
                            message: "required key is missing".into(),
                        });
                    }
                }
                Some(val) => {
                    if let Some(tc) = &field.type_constraint {
                        if !value_matches_type(val, tc) {
                            errors.push(ValidationError {
                                field: field.key.clone(),
                                message: format!("expected type {:?}", tc),
                            });
                        }
                    }
                    if let Some(n) = val.as_f64() {
                        if let Some(min) = field.min {
                            if n < min { errors.push(ValidationError {
                                field: field.key.clone(),
                                message: format!("value {n} is below minimum {min}"),
                            }); }
                        }
                        if let Some(max) = field.max {
                            if n > max { errors.push(ValidationError {
                                field: field.key.clone(),
                                message: format!("value {n} exceeds maximum {max}"),
                            }); }
                        }
                    }
                }
            }
        }
        if errors.is_empty() { Ok(()) }
        else { Err(ConfigError::ValidationError(errors)) }
    }
}
```

**ตัวอย่าง validation ใน application startup:**

```rust
fn validate_config(config: &LayeredConfig) -> Result<(), ConfigError> {
    ConfigSchema::new()
        .field(FieldSchema::required("server.host")
            .with_type(TypeConstraint::String))
        .field(FieldSchema::required("server.port")
            .with_type(TypeConstraint::Integer)
            .with_range(1024.0, 65535.0))
        .field(FieldSchema::required("database.url")
            .with_type(TypeConstraint::String))
        .field(FieldSchema::optional("server.workers")
            .with_range(1.0, 256.0))
        .validate(config)
}

#[tokio::main]
async fn main() {
    let config = LayeredConfig::new()
        .add_source(DefaultsSource::new(defaults()))
        .add_source(FileSource::toml("config.toml"))
        .add_source(EnvSource::new("APP"))
        .build()
        .expect("failed to load config");

    // Fail fast ถ้า config ไม่ครบหรือผิดรูปแบบ
    if let Err(e) = validate_config(&config) {
        eprintln!("Config validation failed: {e}");
        std::process::exit(1);
    }
    // ... start server
}
```

---

### ขั้นที่ 7: Secret Masking

**`src/secret.rs`** — newtype pattern สำหรับซ่อน secrets:

```rust
use std::fmt;
use serde_json::{Map, Value};

/// Newtype wrapper ที่ซ่อนค่าจริงใน Debug และ Display
#[derive(Clone, PartialEq, Eq)]
pub struct SecretString(String);

impl SecretString {
    pub fn new(s: impl Into<String>) -> Self { Self(s.into()) }

    /// expose() ต้องเรียกโดยตั้งใจ — ไม่ใช้โดยบังเอิญ
    pub fn expose(&self) -> &str { &self.0 }
}

/// *** แสดงใน debug output แทนค่าจริง
impl fmt::Debug for SecretString {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "SecretString(***)")
    }
}

/// *** แสดงใน display output แทนค่าจริง
impl fmt::Display for SecretString {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "***")
    }
}
```

**Pattern detection สำหรับ mask_secrets():**

```rust
const SECRET_PATTERNS: &[&str] = &[
    "password", "passwd", "token", "secret",
    "api_key", "apikey", "private_key", "credential", "auth_key",
];

pub fn is_secret_key(key: &str) -> bool {
    let lower = key.to_lowercase();
    SECRET_PATTERNS.iter().any(|p| lower.contains(p))
}

/// แทนที่ค่าของ keys ที่เป็น secret ด้วย "***" ใน map
/// ทำงาน recursive ลงไปใน nested objects ด้วย
pub fn mask_secrets(map: &mut Map<String, Value>) {
    let keys: Vec<String> = map.keys().cloned().collect();
    // ผ่าน 1: mask keys ที่เป็น secret
    for key in &keys {
        if is_secret_key(key) {
            map.insert(key.clone(), Value::String("***".into()));
        }
    }
    // ผ่าน 2: recursive ลงไปใน nested objects (เฉพาะ non-secret keys)
    for key in &keys {
        if !is_secret_key(key) {
            if let Some(Value::Object(nested)) = map.get_mut(key) {
                mask_secrets(nested);
            }
        }
    }
}
```

**ตัวอย่างการใช้งานใน logging:**

```rust
fn log_config(config: &LayeredConfig) {
    let mut displayable = config.merged().clone();
    mask_secrets(&mut displayable);  // ← mask ก่อน log เสมอ
    tracing::info!("Config loaded: {}", serde_json::to_string_pretty(&displayable).unwrap());
}
```

Output ที่ได้:
```json
{
  "database": {
    "host": "prod-db.internal",
    "password": "***",
    "port": 5432
  },
  "jwt_secret": "***",
  "server": { "host": "0.0.0.0", "port": 8080 }
}
```

---

### ขั้นที่ 8: Hot Reload ด้วย tokio::sync::watch

**`src/watch.rs`** — broadcast config updates ไปยัง subscribers หลายตัว:

```rust
use serde_json::{Map, Value};
use tokio::sync::watch;
use crate::layered::LayeredConfig;

pub struct ConfigWatcher {
    tx: watch::Sender<Map<String, Value>>,
}

impl ConfigWatcher {
    /// สร้าง watcher พร้อม initial value จาก config ปัจจุบัน
    pub fn new(config: &LayeredConfig) -> (Self, watch::Receiver<Map<String, Value>>) {
        let (tx, rx) = watch::channel(config.merged().clone());
        (Self { tx }, rx)
    }

    /// Broadcast config ใหม่ไปยัง subscribers ทั้งหมด
    pub fn broadcast(&self, new_config: Map<String, Value>) {
        let _ = self.tx.send(new_config);
    }

    pub fn receiver_count(&self) -> usize {
        self.tx.receiver_count()
    }
}

/// คำนวณ diff ระหว่าง config เก่าและใหม่
/// คืน vec ของ strings: "~key" = เปลี่ยน, "+key" = ใหม่, "-key" = ลบ
pub fn config_diff(
    old: &Map<String, Value>,
    new: &Map<String, Value>,
) -> Vec<String> {
    let mut changed = Vec::new();
    for (key, new_val) in new {
        match old.get(key) {
            None => changed.push(format!("+{key}")),
            Some(old_val) if old_val != new_val => changed.push(format!("~{key}")),
            _ => {}
        }
    }
    for key in old.keys() {
        if !new.contains_key(key) {
            changed.push(format!("-{key}"));
        }
    }
    changed
}
```

**Hot reload loop ใน production service:**

```rust
use tokio::time::{sleep, Duration};

async fn start_config_reloader(
    mut config: LayeredConfig,
    watcher: ConfigWatcher,
) {
    loop {
        sleep(Duration::from_secs(30)).await; // poll ทุก 30 วินาที

        let old = config.merged().clone();
        match config.reload() {
            Ok(()) => {
                let diff = config_diff(&old, config.merged());
                if !diff.is_empty() {
                    tracing::info!("Config reloaded, changes: {:?}", diff);
                    watcher.broadcast(config.merged().clone());
                }
            }
            Err(e) => {
                tracing::error!("Config reload failed: {e}");
                // ไม่ broadcast — ใช้ config เดิมต่อไป
            }
        }
    }
}

// ฝั่ง subscriber (เช่นใน request handler):
async fn handler(mut config_rx: watch::Receiver<Map<String, Value>>) {
    // รอ config เปลี่ยน
    while config_rx.changed().await.is_ok() {
        let cfg = config_rx.borrow();
        let log_level = cfg.get("log")
            .and_then(|v| v.get("level"))
            .and_then(|v| v.as_str())
            .unwrap_or("info");
        tracing::info!("Log level updated to: {log_level}");
    }
}
```

---

### ขั้นที่ 9: Public API และ Demo Binary

**`src/lib.rs`:**

```rust
pub mod error;
pub mod layered;
pub mod schema;
pub mod secret;
pub mod source;
pub mod value;
pub mod watch;

pub use error::{ConfigError, ValidationError};
pub use layered::LayeredConfig;
pub use schema::{ConfigSchema, FieldSchema, TypeConstraint};
pub use secret::{is_secret_key, mask_secrets, SecretString};
pub use source::{ConfigSource, DefaultsSource, EnvSource, FileFormat, FileSource};
pub use value::{deep_merge, get_by_path};
pub use watch::{config_diff, ConfigWatcher};
```

**`src/main.rs`** — demo binary แสดงทุก feature:

```rust
use config_manager::*;
use serde_json::json;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("=== Config Manager Demo ===\n");

    // 1. Layered config (defaults + env override)
    let defaults: serde_json::Map<String, serde_json::Value> =
        serde_json::from_value(json!({
            "server": { "host": "127.0.0.1", "port": 8080 },
            "log":    { "level": "info" },
            "database": { "pool_size": 5 }
        })).unwrap();

    let env_source = EnvSource::with_vars("DEMO", vec![
        ("DEMO_SERVER__PORT".into(), "9090".into()),
        ("DEMO_LOG__LEVEL".into(),   "debug".into()),
    ]);

    let config = LayeredConfig::new()
        .add_source(DefaultsSource::new(defaults))
        .add_source(env_source)
        .build()?;

    let host:      String = config.get("server.host")?;
    let port:      String = config.get("server.port")?;
    let log_level: String = config.get("log.level")?;

    println!("server.host = {host}");
    println!("server.port = {port}  (env override)");
    println!("log.level   = {log_level}  (env override)");

    // 2. Schema validation
    println!("\n--- Schema Validation ---");
    let schema = ConfigSchema::new()
        .field(FieldSchema::required("server.host").with_type(TypeConstraint::String))
        .field(FieldSchema::optional("database.pool_size").with_range(1.0, 100.0));

    match schema.validate(&config) {
        Ok(())  => println!("Config is valid"),
        Err(e)  => println!("Validation failed: {e}"),
    }

    // 3. SecretString
    println!("\n--- SecretString ---");
    let token = SecretString::new("sk-secret-token-12345");
    println!("Debug:   {token:?}");
    println!("Display: {token}");
    println!("Expose:  {}", token.expose());

    // 4. mask_secrets ก่อน log
    println!("\n--- Mask Secrets ---");
    let mut cfg_map: serde_json::Map<String, serde_json::Value> =
        serde_json::from_value(json!({
            "host": "db.example.com",
            "db_password": "hunter2",
            "api_token":   "tok_abc",
            "port": 5432
        })).unwrap();

    println!("Before: {cfg_map:?}");
    mask_secrets(&mut cfg_map);
    println!("After:  {cfg_map:?}");

    // 5. config_diff
    println!("\n--- Config Diff ---");
    let old: serde_json::Map<_, _> =
        serde_json::from_value(json!({ "port": 8080, "debug": true })).unwrap();
    let new: serde_json::Map<_, _> =
        serde_json::from_value(json!({ "port": 9090, "timeout": 30 })).unwrap();

    for change in config_diff(&old, &new) {
        println!("  {change}");
    }
    Ok(())
}
```

**Output จากการรัน `cargo run`:**

```
=== Config Manager Demo ===

server.host = 127.0.0.1
server.port = 9090  (env override)
log.level   = debug  (env override)

--- Schema Validation ---
Config is valid

--- SecretString ---
Debug:   SecretString(***)
Display: ***
Expose:  sk-secret-token-12345

--- Mask Secrets ---
Before: {"api_token": String("tok_abc"), "db_password": String("hunter2"), "host": String("db.example.com"), "port": Number(5432)}
After:  {"api_token": String("***"), "db_password": String("***"), "host": String("db.example.com"), "port": Number(5432)}

--- Config Diff ---
  ~port
  +timeout
  -debug
```

---

## การทดสอบ (Testing)

โปรเจคนี้มี **41 tests** — 27 unit tests ใน modules และ 14 integration tests ครอบคลุมทุก feature

### รัน Tests

```
cargo test
```

**Output จริงจากการรัน `cargo test`:**

```
   Compiling config-manager v0.1.0 (...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.15s
     Running unittests src/lib.rs (target/debug/deps/config_manager-765ba37dfb51fa47)

running 27 tests
test layered::tests::test_key_not_found_error ... ok
test layered::tests::test_dot_notation_get_nested ... ok
test layered::tests::test_layer_priority_higher_wins ... ok
test layered::tests::test_get_vec_of_strings ... ok
test layered::tests::test_reload_updates_merged ... ok
test layered::tests::test_type_mismatch_error ... ok
test schema::tests::test_schema_range_above_max ... ok
test schema::tests::test_schema_range_below_min ... ok
test schema::tests::test_schema_type_mismatch ... ok
test schema::tests::test_schema_required_key_missing ... ok
test schema::tests::test_schema_validation_passes ... ok
test secret::tests::test_is_secret_key_detects_patterns ... ok
test secret::tests::test_mask_secrets_nested ... ok
test secret::tests::test_mask_secrets_replaces_values ... ok
test secret::tests::test_secret_string_expose ... ok
test source::tests::test_defaults_source ... ok
test secret::tests::test_secret_string_debug_redacted ... ok
test secret::tests::test_secret_string_display_redacted ... ok
test source::tests::test_env_source_nested_double_underscore ... ok
test source::tests::test_env_source_prefix_strip ... ok
test value::tests::test_deep_merge_simple_override ... ok
test value::tests::test_deep_merge_nested_preserves_unmodified ... ok
test value::tests::test_get_by_path_deep_nested ... ok
test value::tests::test_get_by_path_top_level ... ok
test watch::tests::test_config_diff_detects_changes ... ok
test watch::tests::test_config_diff_no_changes ... ok
test watch::tests::test_config_watcher_broadcasts ... ok

test result: ok. 27 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/config_manager-3339ef413a84df94)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running tests/integration_tests.rs (target/debug/deps/integration_tests-d8f3259f6d07f782)

running 14 tests
test test_dot_notation_three_levels_deep ... ok
test test_get_u16_type_coercion ... ok
test test_env_source_double_underscore_nesting ... ok
test test_env_source_strips_prefix ... ok
test test_get_vec_string_coercion ... ok
test test_range_validation_below_min ... ok
test test_json_file_load ... ok
test test_required_key_missing_validation ... ok
test test_key_not_found_returns_correct_error ... ok
test test_reload_diff_catches_changes ... ok
test test_secret_masking_in_debug ... ok
test test_three_layer_priority_chain ... ok
test test_type_mismatch_returns_correct_error ... ok
test test_toml_file_load ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests config_manager

running 1 test
test src/layered.rs - layered::LayeredConfig::get (line 50) ... ignored

test result: ok. 0 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### อธิบาย Tests ที่สำคัญ

**Test: Layer Priority — env ชนะ file ชนะ defaults**

```rust
#[test]
fn test_three_layer_priority_chain() {
    let defaults = DefaultsSource::new(
        json!({ "host": "localhost", "port": 3000, "debug": false })
            .as_object().unwrap().clone(),
    );
    // file layer: override port
    let file_layer = DefaultsSource::new(json!({ "port": 8080 })
        .as_object().unwrap().clone()).with_label("file");
    // env layer: override port อีกครั้ง (highest priority)
    let env_layer = DefaultsSource::new(json!({ "port": 9090, "debug": true })
        .as_object().unwrap().clone()).with_label("env");

    let config = LayeredConfig::new()
        .add_source(defaults)
        .add_source(file_layer)
        .add_source(env_layer)
        .build().unwrap();

    assert_eq!(config.get::<i64>("port").unwrap(), 9090);     // env ชนะ
    assert_eq!(config.get::<bool>("debug").unwrap(), true);   // env override
    assert_eq!(config.get::<String>("host").unwrap(), "localhost"); // preserved
}
```

**Test: TOML File Load**

```rust
#[test]
fn test_toml_file_load() {
    let toml_content = r#"
[server]
host = "example.com"
port = 9000

[database]
url = "postgres://localhost/mydb"
max_connections = 20
"#;
    let path = write_temp_file(toml_content, "toml");
    let source = FileSource::toml(path.to_str().unwrap());
    let map = source.load().expect("TOML load should succeed");

    assert_eq!(map["server"].as_object().unwrap()["host"], json!("example.com"));
    assert_eq!(map["server"].as_object().unwrap()["port"], json!(9000));
    assert_eq!(map["database"].as_object().unwrap()["max_connections"], json!(20));

    let _ = std::fs::remove_file(&path);
}
```

**Test: Secret Masking ใน Debug Output**

```rust
#[test]
fn test_secret_masking_in_debug() {
    let secret = SecretString::new("hunter2-password");
    let debug_repr   = format!("{secret:?}");
    let display_repr = format!("{secret}");

    assert!(!debug_repr.contains("hunter2"), "must not leak in debug");
    assert!(debug_repr.contains("***"));
    assert_eq!(display_repr, "***");
    assert_eq!(secret.expose(), "hunter2-password");
}
```

**Test: config_diff ตรวจ reload changes**

```rust
#[test]
fn test_reload_diff_catches_changes() {
    let old = serde_json::from_value::<Map<String, Value>>(json!({
        "host": "localhost", "port": 8080, "debug": true
    })).unwrap();

    let new = serde_json::from_value::<Map<String, Value>>(json!({
        "host": "localhost", "port": 9090, "timeout": 30
    })).unwrap();

    let diff = config_diff(&old, &new);
    assert!(diff.contains(&"~port".to_string()));    // เปลี่ยน
    assert!(diff.contains(&"+timeout".to_string())); // ใหม่
    assert!(diff.contains(&"-debug".to_string()));   // ลบ
    assert!(!diff.iter().any(|d| d.contains("host"))); // ไม่เปลี่ยน
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/config-manager
```

### ใช้เป็น Library

เพิ่มใน `Cargo.toml` ของ project อื่น:

```toml
[dependencies]
config-manager = { path = "../config-manager" }
# หรือจาก crates.io:
# config-manager = "0.1"
```

### ตัวอย่าง Docker Integration

```dockerfile
FROM rust:1.75 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/my-service /usr/local/bin/
COPY config/defaults.toml /etc/my-service/config.toml

# Config ผ่าน env vars — override ค่าใน file ได้
ENV APP_SERVER__PORT=8080
ENV APP_DATABASE__URL=postgres://db:5432/prod

CMD ["/usr/local/bin/my-service"]
```

### Kubernetes ConfigMap Integration

```yaml
# k8s-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.toml: |
    [server]
    host = "0.0.0.0"
    port = 8080
    workers = 4

    [log]
    level = "info"
    format = "json"
```

```yaml
# k8s-deployment.yaml
spec:
  containers:
  - name: app
    image: my-service:latest
    volumeMounts:
    - name: config
      mountPath: /etc/app
    env:
    - name: APP_DATABASE__PASSWORD  # secret ผ่าน k8s Secret
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
  volumes:
  - name: config
    configMap:
      name: app-config
```

ระบบ config ที่สร้างในโปรเจคนี้ compatible กับ pattern นี้โดยตรง — `FileSource::toml("/etc/app/config.toml")` อ่านจาก mounted ConfigMap และ `EnvSource::new("APP")` อ่าน secrets จาก env vars ที่ inject มา

### Cargo Publish Checklist

```toml
[package]
name = "config-manager"
version = "0.1.0"
edition = "2021"
description = "Layered, hot-reloadable configuration manager for Rust"
license = "MIT OR Apache-2.0"
repository = "https://github.com/example/config-manager"
keywords = ["config", "configuration", "settings", "environment"]
categories = ["config", "data-structures"]
```

---

## หลุมพรางและข้อควรระวัง (Pitfalls)

### Pitfall 1: EnvSource ใช้ `std::env::set_var()` ใน Tests ที่ Run แบบ Parallel

```rust
// ❌ อันตราย — tests run แบบ parallel ใน thread เดียวกัน
#[test]
fn test_env_a() {
    std::env::set_var("APP_PORT", "8080");
    let source = EnvSource::new("APP");
    // ... test logic
    std::env::remove_var("APP_PORT"); // ถ้า test อื่น run ระหว่างนี้ จะเห็นค่าผิด
}
```

ตั้งแต่ Rust 1.81 เป็นต้นมา `std::env::set_var` และ `remove_var` ถูก mark ว่า deprecated เพราะ unsound ใน multi-threaded environment การ mutate global env state ระหว่าง parallel tests สร้าง **data race ใน C standard library** ที่ Rust ไม่สามารถป้องกันได้ด้วย borrow checker

```rust
// ✅ ถูกต้อง — inject vars ผ่าน with_vars() โดยไม่แตะ global state
#[test]
fn test_env_a() {
    let source = EnvSource::with_vars("APP", vec![
        ("APP_PORT".into(), "8080".into()),
    ]);
    let map = source.load().unwrap();
    assert_eq!(map["port"], json!("8080"));
}
```

`with_vars()` ที่เราสร้างขึ้นแก้ปัญหานี้ได้อย่างสมบูรณ์ — ไม่ใช้ global state เลย

---

### Pitfall 2: Deep Merge ทับ Leaf Node ด้วย Object (หรือกลับกัน)

สมมติ config file มี:
```toml
[server]
host = "localhost"
```

แต่ env var มี:
```
APP_SERVER = "production"  # server เป็น string แทน object!
```

ผล `deep_merge` จะ **ทับ object ทั้งหมด** ด้วย string:

```
base:    { server: { host: "localhost" } }
overlay: { server: "production" }
result:  { server: "production" }  ← host หาย!
```

เหตุผล: code ของเราใช้ `match base.get_mut(key)` — ถ้า overlay ไม่ใช่ object ก็ไป default arm ซึ่ง replace ทั้งหมด

**วิธีป้องกัน:** ถ้า production system ต้องการ detect ปัญหานี้ ให้ emit warning log เมื่อ overlay type ต่างจาก base type:

```rust
Some(base_val) => {
    if base_val.is_object() && !val.is_object() {
        tracing::warn!(
            "Config merge: key '{}' type changed from object to {:?}",
            key, val
        );
    }
    base.insert(key.clone(), val.clone());
}
```

---

### Pitfall 3: TOML Integer กับ JSON Integer ไม่ Compatible ทุกกรณี

TOML integer เป็น i64 เสมอ แต่ JSON number อาจเป็น i64, u64, หรือ f64 ขึ้นอยู่กับค่า เมื่อ `toml::from_str::<serde_json::Value>()` แปลง TOML ไป JSON, integer จะเป็น `Value::Number` ที่ represent แบบ i64 ภายใน

ปัญหาเกิดเมื่อต้องการ `u16` แต่ค่าถูกเก็บเป็น i64:

```rust
// TOML: port = 8080 → serde_json::Value::Number(8080i64)
let port: u16 = config.get("server.port")?;
// serde_json::from_value ทำ coercion ให้อัตโนมัติ → 8080u16  ✅
```

แต่ถ้า env var override ด้วย string:
```
APP_SERVER__PORT = "8080"  → Value::String("8080")
let port: u16 = config.get("server.port")?;
// ❌ TypeMismatch! String ไม่ coerce เป็น u16 โดยอัตโนมัติ
```

**วิธีแก้ใน application code:** ใช้ `.parse()` สำหรับ env vars ที่เป็น number:

```rust
let port: u16 = config.get::<String>("server.port")
    .or_else(|_| config.get::<u16>("server.port")
        .map(|p| p.to_string()))
    .unwrap()
    .parse()
    .expect("server.port must be a valid port number");
```

หรือเขียน custom deserializer ที่ handle ทั้ง string และ number (ดู Extension 2)

---

### Pitfall 4: การใช้ `notify` Crate สำหรับ File Watching

`notify` crate เป็นตัวเลือกยอดนิยมสำหรับ file system watching แต่มีหลุมพรางสำคัญ:

```rust
// ❌ ปัญหา 1: notify callback รันใน thread แยก ไม่สามารถ await ได้โดยตรง
use notify::{Watcher, RecursiveMode, watcher};
let (tx, rx) = std::sync::mpsc::channel();
let mut watcher = watcher(tx, Duration::from_secs(2)).unwrap();
watcher.watch("/etc/app/config.toml", RecursiveMode::NonRecursive).unwrap();

// ❌ ปัญหา 2: debounce ต้องจัดการเอง — editor อาจเขียนไฟล์หลายครั้งใน milliseconds
// ❌ ปัญหา 3: inotify/kqueue/FSEvents behave ต่างกันในแต่ละ OS
```

**วิธีที่ง่ายและ reliable กว่า** คือ polling ทุก N วินาที:

```rust
// ✅ ง่าย, predictable, ทำงานทุก OS
async fn poll_reload(
    config_path: &str,
    mut config: LayeredConfig,
    watcher: ConfigWatcher,
    interval: Duration,
) {
    let mut last_modified = std::fs::metadata(config_path)
        .ok()
        .and_then(|m| m.modified().ok());

    loop {
        tokio::time::sleep(interval).await;
        let current_modified = std::fs::metadata(config_path)
            .ok()
            .and_then(|m| m.modified().ok());

        if current_modified != last_modified {
            last_modified = current_modified;
            let old = config.merged().clone();
            if config.reload().is_ok() {
                let diff = config_diff(&old, config.merged());
                if !diff.is_empty() {
                    watcher.broadcast(config.merged().clone());
                }
            }
        }
    }
}
```

ถ้า latency ต้องการน้อยมาก (< 1 วินาที) ให้ใช้ `notify-debouncer-mini` crate ซึ่ง wrap `notify` และจัดการ debounce ให้

---

### Pitfall 5: SecretString ยัง Leak ได้ถ้า Clone แล้ว Log ผ่าน Struct อื่น

```rust
#[derive(Debug)]
struct DatabaseConfig {
    host: String,
    password: String,  // ❌ String ธรรมดา — Debug leak!
}
```

```rust
#[derive(Debug)]
struct DatabaseConfig {
    host: String,
    password: SecretString,  // ✅ SecretString — Debug แสดง ***
}

// การ log จะปลอดภัย:
// DatabaseConfig { host: "localhost", password: SecretString(***) }
```

การ mask ผ่าน `mask_secrets()` function ช่วยได้เฉพาะ `Map<String, Value>` — สำหรับ custom struct ที่มี secret field ต้องใช้ `SecretString` newtype โดยตรง

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: YAML Support ด้วย serde_yaml (ระดับง่าย)

เพิ่ม YAML format ใน `FileSource` โดยเพิ่ม dependency:

```toml
[dependencies]
serde_yaml = "0.9"
```

แก้ไข `FileFormat` enum และ `FileSource::load()`:

```rust
pub enum FileFormat { Toml, Json, Yaml }

// ใน load():
FileFormat::Yaml => serde_yaml::from_str::<serde_json::Value>(&content)
    .map_err(|e| ConfigError::SourceError(e.to_string()))?,
```

เพิ่ม constructor:
```rust
pub fn yaml(path: impl Into<String>) -> Self {
    Self { path: path.into(), format: FileFormat::Yaml }
}
```

สิ่งที่ได้เรียน: YAML มี datetime, anchor, alias ที่ serde_yaml จัดการให้ แต่ต้องระวัง YAML "Norway problem" (ค่า `NO` ถูก parse เป็น `false` ใน YAML 1.1)

---

### แบบฝึกหัดที่ 2: String-to-Number Coercion สำหรับ Env Vars (ระดับกลาง)

ปัญหา: env vars เป็น string เสมอ แต่ application ต้องการ `u16`, `bool`, `f64` สร้าง custom deserialize wrapper:

```rust
use serde::{Deserialize, Deserializer};

/// Deserialize จาก string หรือ number
fn deserialize_port<'de, D>(deserializer: D) -> Result<u16, D::Error>
where D: Deserializer<'de> {
    use serde::de::{self, Unexpected};
    match serde_json::Value::deserialize(deserializer)? {
        serde_json::Value::Number(n) => {
            n.as_u64()
                .and_then(|n| u16::try_from(n).ok())
                .ok_or_else(|| de::Error::invalid_value(
                    Unexpected::Other("number out of range"),
                    &"u16"
                ))
        }
        serde_json::Value::String(s) => {
            s.parse().map_err(|_| de::Error::invalid_value(
                Unexpected::Str(&s), &"a port number"
            ))
        }
        other => Err(de::Error::invalid_type(
            Unexpected::Other(&format!("{other:?}")),
            &"u16 or string"
        )),
    }
}

#[derive(Deserialize)]
struct ServerConfig {
    host: String,
    #[serde(deserialize_with = "deserialize_port")]
    port: u16,
}
```

สิ่งที่ได้เรียน: custom serde deserializer, `Unexpected` enum ใน serde error system

---

### แบบฝึกหัดที่ 3: inotify-based File Watcher (ระดับกลาง)

แทนที่ polling ด้วย file system events จริงโดยใช้ `notify-debouncer-mini`:

```toml
[dependencies]
notify-debouncer-mini = "0.4"
```

```rust
use notify_debouncer_mini::{new_debouncer, DebounceEventResult};
use std::time::Duration;

async fn watch_file(
    path: String,
    mut config: LayeredConfig,
    watcher: ConfigWatcher,
) {
    let (tx, mut rx) = tokio::sync::mpsc::channel(1);

    // ต้องรัน watcher ใน blocking thread เพราะมัน block
    let path_clone = path.clone();
    std::thread::spawn(move || {
        let (std_tx, std_rx) = std::sync::mpsc::channel();
        let mut debouncer = new_debouncer(
            Duration::from_millis(500),
            None,
            move |res: DebounceEventResult| {
                let _ = std_tx.send(res);
            },
        ).unwrap();
        debouncer.watcher().watch(
            std::path::Path::new(&path_clone),
            notify::RecursiveMode::NonRecursive,
        ).unwrap();
        loop {
            if let Ok(_events) = std_rx.recv() {
                let _ = tx.blocking_send(());
            }
        }
    });

    while let Some(()) = rx.recv().await {
        let old = config.merged().clone();
        if config.reload().is_ok() {
            let diff = config_diff(&old, config.merged());
            if !diff.is_empty() {
                watcher.broadcast(config.merged().clone());
                tracing::info!("Config reloaded: {:?}", diff);
            }
        }
    }
}
```

สิ่งที่ได้เรียน: bridging sync/async, `blocking_send` vs `send`, thread-to-async handoff

---

### แบบฝึกหัดที่ 4: Typed Config Struct ด้วย serde Deserialize (ระดับกลาง)

แทนที่การเรียก `config.get()` ทีละ key ด้วยการ deserialize ทั้ง struct พร้อมกัน:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct AppConfig {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub log: LogConfig,
}

#[derive(Debug, Deserialize)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
    pub workers: Option<u32>,
}

#[derive(Debug, Deserialize)]
pub struct DatabaseConfig {
    pub url: String,
    pub pool_size: u32,
    #[serde(default)]
    pub ssl: bool,
}

#[derive(Debug, Deserialize)]
pub struct LogConfig {
    pub level: String,
    #[serde(default = "default_format")]
    pub format: String,
}

fn default_format() -> String { "json".into() }
```

เพิ่ม method `into_typed<T: DeserializeOwned>()` ใน `LayeredConfig`:

```rust
impl LayeredConfig {
    pub fn into_typed<T: serde::de::DeserializeOwned>(
        &self,
    ) -> Result<T, ConfigError> {
        serde_json::from_value(serde_json::Value::Object(self.merged.clone()))
            .map_err(|e| ConfigError::SourceError(e.to_string()))
    }
}

// การใช้งาน:
let app_cfg: AppConfig = config.into_typed()?;
println!("Server: {}:{}", app_cfg.server.host, app_cfg.server.port);
```

สิ่งที่ได้เรียน: struct-based deserialization, `#[serde(default)]`, nested struct mapping

---

### แบบฝึกหัดที่ 5: Config Versioning และ Migration (ระดับสูง)

ในระบบจริงที่ evolve ตามเวลา config schema เปลี่ยนได้ สร้างระบบ migration:

```rust
pub trait ConfigMigration {
    /// Version ของ schema ที่ migration นี้ remap จาก
    fn from_version(&self) -> u32;
    /// Migration logic
    fn migrate(&self, config: &mut Map<String, Value>);
}

pub struct ConfigVersionManager {
    migrations: Vec<Box<dyn ConfigMigration>>,
}

impl ConfigVersionManager {
    pub fn migrate_to_latest(&self, config: &mut Map<String, Value>) {
        let version = config.get("_schema_version")
            .and_then(|v| v.as_u64())
            .unwrap_or(0) as u32;

        for migration in &self.migrations {
            if migration.from_version() >= version {
                migration.migrate(config);
            }
        }
    }
}
```

ตัวอย่าง migration จาก v1 (flat) ไป v2 (nested):
```rust
struct V1ToV2Migration;

impl ConfigMigration for V1ToV2Migration {
    fn from_version(&self) -> u32 { 1 }
    fn migrate(&self, config: &mut Map<String, Value>) {
        // เก่า: { "host": "x", "port": 8080 }
        // ใหม่: { "server": { "host": "x", "port": 8080 } }
        if let (Some(host), Some(port)) = (
            config.remove("host"),
            config.remove("port"),
        ) {
            let mut server = Map::new();
            server.insert("host".into(), host);
            server.insert("port".into(), port);
            config.insert("server".into(), Value::Object(server));
            config.insert("_schema_version".into(), json!(2));
        }
    }
}
```

สิ่งที่ได้เรียน: versioning strategies, backward compatibility, migration chaining

---

### แบบฝึกหัดที่ 6: Encrypted Secrets via KMS (ระดับ Expert)

ใน production หลายองค์กรใช้ AWS KMS, HashiCorp Vault, หรือ GCP Secret Manager เพื่อเก็บ encrypted secrets สร้าง `SecretSource` trait:

```rust
#[async_trait::async_trait]
pub trait SecretBackend: Send + Sync {
    async fn get_secret(&self, key: &str) -> Result<String, ConfigError>;
}

pub struct VaultSource {
    backend: Arc<dyn SecretBackend>,
    secret_keys: Vec<String>,
}

impl VaultSource {
    pub async fn load_async(&self) -> Result<ConfigMap, ConfigError> {
        let mut result = Map::new();
        for key in &self.secret_keys {
            let secret = self.backend.get_secret(key).await?;
            result.insert(key.clone(), Value::String(secret));
        }
        Ok(result)
    }
}
```

Mock backend สำหรับ test:
```rust
struct MockVaultBackend {
    secrets: HashMap<String, String>,
}

#[async_trait::async_trait]
impl SecretBackend for MockVaultBackend {
    async fn get_secret(&self, key: &str) -> Result<String, ConfigError> {
        self.secrets.get(key)
            .cloned()
            .ok_or_else(|| ConfigError::KeyNotFound(key.into()))
    }
}
```

สิ่งที่ได้เรียน: async traits, `async_trait` crate, mock pattern, secret management patterns

---

## สรุป

ในโปรเจคนี้เราได้สร้าง Configuration Manager ระดับ production ที่ครอบคลุม:

**Pattern และ Concepts สำคัญ:**

| สิ่งที่สร้าง | Rust Pattern |
|---|---|
| Multi-source config | `trait ConfigSource` + `Box<dyn T>` |
| Priority layering | `Vec<Box<dyn ConfigSource>>` + deep_merge |
| Type-safe access | `get::<T: DeserializeOwned>(path)` |
| Newtype secret masking | `struct SecretString(String)` + override `Debug` |
| Compile-time safety | Borrow checker ป้องกัน secret leak |
| Schema validation | Builder pattern + error accumulation |
| Hot reload | `tokio::sync::watch` channel |
| Test isolation | `with_vars()` แทน global env mutation |

**สิ่งที่ทำให้ implementation นี้ production-ready:**

1. **Test isolation** — `EnvSource::with_vars()` ไม่ mutate global state ทำให้ tests รัน parallel ได้ปลอดภัย
2. **Error accumulation** — `ValidationError` เป็น `Vec` รวมทุก error ไว้แทนที่จะ fail-fast
3. **Zero-cost abstraction** — `get::<T>()` ทำ type conversion ที่ compile time ไม่มี overhead ตอน runtime
4. **Explicit secret exposure** — `.expose()` method ทำให้ต้อง "ตั้งใจ" อ่านค่าจริง
5. **Deep merge ไม่ใช่ shallow** — override แค่ค่าที่ต้องการโดยไม่ต้อง specify ทุก key

โปรเจคถัดไป **G06: Health Checker** จะใช้ `LayeredConfig` เป็น config backend สำหรับระบบ health check endpoints — config จะกำหนด endpoints ที่ต้อง check, timeout, retry policy, และ alert thresholds ซึ่งสามารถ hot reload ได้โดยไม่รีสตาร์ท service

---

**โปรเจคก่อนหน้า:** [Project G04: Metrics Collector](project-g04-metrics-collector.md) | **โปรเจคถัดไป:** [Project G06: Health Checker](project-g06-health-checker.md)
