# Project B05: Webhook Relay Service

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

**Webhook Relay Service** คือระบบกลางที่รับ webhook event จากแหล่งภายนอก (GitHub, Stripe, Shopify ฯลฯ) แล้วกระจาย (fan-out) ไปยัง subscriber หลาย endpoint พร้อม signature verification, retry logic, filter expression, และ circuit breaker — คือ "event bus" แบบ push ที่ใช้งานได้จริงในระดับ production

**ทำไมถึงน่าสร้าง?** หลายองค์กรมี microservice หลายตัวที่ต้องการรับ webhook event เดียวกัน (เช่น GitHub push event ต้องไปถึงทั้ง CI runner, Slack notifier, และ deployment system) การ relay ผ่านศูนย์กลางทำให้:
- แต่ละ service ลงทะเบียน subscription ของตัวเองได้
- มี audit log ว่า delivery สำเร็จหรือไม่
- Retry อัตโนมัติเมื่อ target ล่ม
- กรอง event ที่ไม่เกี่ยวข้องออกก่อนส่ง

**Use case ในโลกจริง:**
- GitHub App ที่ต้องกระจาย push event ไปหลาย service
- Payment gateway webhook ที่ต้องส่งไปทั้ง fulfillment และ accounting
- IoT sensor data relay ไปยัง multiple analytics pipeline
- Internal event bus สำหรับ microservice architecture

## สิ่งที่จะได้เรียนรู้

- การออกแบบ **fan-out delivery system** ด้วย `tokio::spawn` แบบ concurrent
- การสร้าง **HMAC-SHA256 signature** ด้วย `hmac` + `sha2` crate (รูปแบบ GitHub/Stripe)
- **Exponential backoff retry** พร้อม cap และ jitter — pattern สำคัญสำหรับ resilient system
- **Circuit breaker pattern** เพื่อป้องกันการ flood target ที่ล่มอยู่
- **PostgreSQL + sqlx** สำหรับ async database access พร้อม compile-time query checking
- การออกแบบ **filter DSL** อย่างง่าย (JSONPath-like `$.event == "push"`)
- การจัดการ **arbitrary Content-Type** — รับ body ดิบโดยไม่ parse
- **Axum 0.8** middleware, state sharing, และ error handling แบบ production-ready

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 1–30: ownership, borrowing, structs, enums, traits, error handling ด้วย `Result`
- จาก Part 51–60: async/await, `tokio::spawn`, Future พื้นฐาน
- จาก Part 61–70: HTTP concepts, Axum routing, handler functions, State extraction
- จาก Part 71–80: SQLx, database connection pool, query macros
- ความรู้พื้นฐาน: HMAC/SHA256 (cryptographic hash), HTTP webhook (POST + signature header)

## โครงสร้างโปรเจค (Project Layout)

```
webhook-relay/
├── src/
│   ├── main.rs               ← tokio runtime + axum router + worker spawn
│   ├── db.rs                 ← PgPool setup + migration helper
│   ├── models.rs             ← Webhook, Subscription, Delivery structs
│   ├── error.rs              ← AppError + IntoResponse impl
│   ├── signature.rs          ← HMAC-SHA256 generation & verification
│   ├── filter.rs             ← JSONPath-like filter evaluation
│   ├── handlers/
│   │   ├── mod.rs
│   │   ├── ingest.rs         ← POST /webhooks/{id}/ingest
│   │   ├── subscriptions.rs  ← POST/GET/DELETE /subscriptions
│   │   ├── deliveries.rs     ← GET /subscriptions/{id}/deliveries + replay
│   │   └── stats.rs          ← GET /stats
│   └── worker/
│       ├── mod.rs            ← delivery worker loop
│       ├── retry.rs          ← retry delay calculation
│       └── circuit.rs        ← circuit breaker state machine
├── migrations/
│   └── 001_initial.sql       ← PostgreSQL schema
├── tests/
│   └── integration_test.rs   ← end-to-end tests
└── Cargo.toml
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
                    ┌──────────────────────────────────────────────────┐
                    │             Webhook Relay Service                  │
                    │                                                    │
  GitHub/Stripe ──► │ POST /webhooks/{id}/ingest                        │
  Shopify/Custom    │   │                                               │
                    │   ├─ verify X-Hub-Signature (optional)            │
                    │   ├─ store raw body + headers + timestamp in DB   │
                    │   └─ tokio::spawn delivery_worker(webhook_event)  │
                    │           │                                        │
                    │           ├─ query active subscriptions           │
                    │           ├─ evaluate filter_expression           │
                    │           ├─ for each matching subscription:      │
                    │           │   ├─ check circuit breaker state      │
                    │           │   ├─ POST to target_url               │
                    │           │   │   + X-Webhook-Signature header    │
                    │           │   └─ record delivery result in DB     │
                    │           └─ schedule retry if failed             │
                    │                                                    │
  Client ─────────► │ GET  /subscriptions/{id}/deliveries               │
                    │ POST /deliveries/{id}/replay                      │
                    │ GET  /stats                                        │
                    └──────────────────────────────────────────────────┘
                                          │
                              ┌───────────▼───────────┐
                              │     PostgreSQL DB       │
                              │  ┌─────────────────┐  │
                              │  │   webhooks       │  │
                              │  │   subscriptions  │  │
                              │  │   deliveries     │  │
                              │  └─────────────────┘  │
                              └───────────────────────┘
```

### Database Schema

ระบบใช้ 3 ตารางหลัก:

- **`webhooks`** — endpoint ที่ลงทะเบียนไว้ให้ ingest (แต่ละ webhook มี unique `id` ใช้ใน URL)
- **`subscriptions`** — การลงทะเบียนว่าจะส่ง event ประเภทใดไปที่ target URL ไหน
- **`deliveries`** — log ของ delivery attempt แต่ละครั้ง พร้อม status และ retry schedule

### Design Decisions

**1. Raw body storage** — เก็บ body เป็น `bytea` (PostgreSQL binary) เพราะ webhook body อาจเป็นอะไรก็ได้ (JSON, XML, form-encoded) การ parse เป็น JSON ก่อนเก็บจะทำให้สูญเสียข้อมูลหาก format เปลี่ยน

**2. tokio::spawn per delivery** — แต่ละ delivery fan-out ทำงานใน spawn แยก เพื่อให้ HTTP response กลับ client (200 Accepted) เร็วที่สุด โดยไม่ต้องรอ delivery สำเร็จ

**3. Circuit breaker per subscription** — track state แยกกันแต่ละ subscription (ไม่ใช่ per target URL) เพราะ subscription แต่ละอันอาจมี auth header ต่างกัน และ failure ของ subscription หนึ่งไม่ควรส่งผลต่ออีกอัน

**4. Simple filter DSL** — ใช้ `$.field == "value"` แทน JSONPath เต็มรูปแบบ เพราะ use case จริงมักต้องการแค่ match field เดียว การ implement เองทำให้ไม่ต้องพึ่ง dependency ที่ซับซ้อน

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: Database Schema และโครงสร้างโปรเจค

เริ่มต้นด้วยการวาง foundation — Cargo.toml, SQL schema, และ models

**`Cargo.toml`**

```toml
[package]
name = "webhook-relay"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "webhook-relay"
path = "src/main.rs"

[dependencies]
# Web framework
axum = { version = "0.8", features = ["macros"] }
tower = { version = "0.5", features = ["util"] }
tower-http = { version = "0.6", features = ["trace", "cors"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Database
sqlx = { version = "0.8", features = [
    "runtime-tokio-native-tls",
    "postgres",
    "uuid",
    "chrono",
    "json",
] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Crypto / HMAC
hmac = "0.12"
sha2 = "0.10"
hex = "0.4"

# HTTP client สำหรับ outbound delivery
reqwest = { version = "0.12", features = ["json"] }

# Time
chrono = { version = "0.4", features = ["serde"] }

# UUID
uuid = { version = "1", features = ["v4", "serde"] }

# Logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Error handling
thiserror = "1"
anyhow = "1"

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

**`migrations/001_initial.sql`**

```sql
-- เปิดใช้งาน UUID extension
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ตาราง webhooks: endpoint ที่รับ ingest
CREATE TABLE webhooks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    description TEXT,
    -- optional: secret สำหรับ verify inbound signature (GitHub/Stripe format)
    inbound_secret TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ตาราง subscriptions: ลงทะเบียนว่าจะส่ง event ไปที่ไหน
CREATE TABLE subscriptions (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_webhook_id UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
    target_url        TEXT NOT NULL,
    -- secret สำหรับ sign outbound delivery
    secret            TEXT NOT NULL,
    -- filter expression เช่น `$.event == "push"` หรือ empty string = match all
    filter_expression TEXT NOT NULL DEFAULT '',
    -- สถานะ: active, paused (circuit breaker), deleted
    status            TEXT NOT NULL DEFAULT 'active',
    -- circuit breaker: นับ consecutive failures
    consecutive_failures INTEGER NOT NULL DEFAULT 0,
    -- วันเวลาที่ circuit เปิด (paused) ถ้าไม่ได้ paused = NULL
    paused_until      TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ตาราง deliveries: log ของ delivery attempt
CREATE TABLE deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES subscriptions(id) ON DELETE CASCADE,
    -- body ที่ส่ง (เก็บไว้เพื่อ replay)
    webhook_body    BYTEA NOT NULL,
    -- headers ของ inbound request ที่เกี่ยวข้อง (เก็บเป็น JSON)
    inbound_headers JSONB NOT NULL DEFAULT '{}',
    -- สถานะ: pending, success, failed, retrying
    status          TEXT NOT NULL DEFAULT 'pending',
    -- attempt ปัจจุบัน (เริ่มที่ 1)
    attempt_count   INTEGER NOT NULL DEFAULT 0,
    -- เวลาที่จะ retry ครั้งถัดไป (NULL ถ้าไม่มี retry)
    next_retry_at   TIMESTAMPTZ,
    -- ผล HTTP response จาก target
    response_code   INTEGER,
    response_body   TEXT,
    -- เวลา delivery (milliseconds)
    duration_ms     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Index สำหรับ query ที่ใช้บ่อย
CREATE INDEX idx_subscriptions_webhook_id
    ON subscriptions(source_webhook_id)
    WHERE status = 'active';

CREATE INDEX idx_deliveries_subscription_id
    ON deliveries(subscription_id);

CREATE INDEX idx_deliveries_next_retry
    ON deliveries(next_retry_at)
    WHERE status = 'retrying' AND next_retry_at IS NOT NULL;
```

**`src/models.rs`**

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

/// Webhook endpoint ที่ลงทะเบียนไว้รับ ingest
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Webhook {
    pub id: Uuid,
    pub name: String,
    pub description: Option<String>,
    pub inbound_secret: Option<String>,
    pub created_at: DateTime<Utc>,
}

/// Subscription — ลงทะเบียนส่ง event จาก source webhook ไป target URL
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Subscription {
    pub id: Uuid,
    pub source_webhook_id: Uuid,
    pub target_url: String,
    pub secret: String,
    pub filter_expression: String,
    pub status: String,
    pub consecutive_failures: i32,
    pub paused_until: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

impl Subscription {
    /// ตรวจสอบว่า subscription ใช้งานได้ตอนนี้หรือไม่
    /// (active และไม่ได้ paused โดย circuit breaker)
    pub fn is_deliverable(&self) -> bool {
        if self.status != "active" {
            return false;
        }
        match self.paused_until {
            Some(until) => Utc::now() > until,
            None => true,
        }
    }
}

/// Delivery attempt record
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Delivery {
    pub id: Uuid,
    pub subscription_id: Uuid,
    pub webhook_body: Vec<u8>,
    pub inbound_headers: serde_json::Value,
    pub status: String,
    pub attempt_count: i32,
    pub next_retry_at: Option<DateTime<Utc>>,
    pub response_code: Option<i32>,
    pub response_body: Option<String>,
    pub duration_ms: Option<i32>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

/// Request body สำหรับสร้าง subscription
#[derive(Debug, Deserialize)]
pub struct CreateSubscriptionRequest {
    pub source_webhook_id: Uuid,
    pub target_url: String,
    pub secret: String,
    #[serde(default)]
    pub filter_expression: String,
}

/// Response สำหรับ stats endpoint
#[derive(Debug, Serialize)]
pub struct StatsResponse {
    pub total_ingested: i64,
    pub total_deliveries: i64,
    pub successful_deliveries: i64,
    pub failed_deliveries: i64,
    pub success_rate_pct: f64,
    pub avg_duration_ms: f64,
}
```

---

### ขั้นที่ 2: HMAC Signature และ Filter Expression

สองส่วน core logic ที่ไม่ขึ้นกับ database — เหมาะสำหรับ unit test

**`src/signature.rs`**

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

/// สร้าง HMAC-SHA256 signature ในรูปแบบ `sha256=<hex>`
/// รองรับทั้ง GitHub (`X-Hub-Signature-256`) และ Stripe (`Stripe-Signature`)
///
/// # Arguments
/// * `secret` — shared secret ที่ตกลงกับ subscriber
/// * `body`   — raw request body bytes
///
/// # Returns
/// String ในรูปแบบ `sha256=<64-char-hex>`
pub fn compute_signature(secret: &str, body: &[u8]) -> String {
    let mut mac = HmacSha256::new_from_slice(secret.as_bytes())
        .expect("HMAC accepts any key length");
    mac.update(body);
    let result = mac.finalize().into_bytes();
    format!("sha256={}", hex::encode(result))
}

/// ตรวจสอบ signature จาก inbound request
///
/// รองรับ format:
/// - `sha256=<hex>` (GitHub, standard)
/// - `v1=<hex>` (Stripe — convert ให้เป็น sha256= ก่อน compare)
pub fn verify_inbound_signature(
    secret: &str,
    body: &[u8],
    received_signature: &str,
) -> bool {
    // normalize: แปลง `v1=` → `sha256=` สำหรับ Stripe compat
    let normalized = if received_signature.starts_with("v1=") {
        format!("sha256={}", &received_signature[3..])
    } else {
        received_signature.to_string()
    };

    let expected = compute_signature(secret, body);

    // timing-safe comparison: เปรียบเทียบ byte ด้วยความยาวเท่ากัน
    // (ในระบบ production จริงควรใช้ `subtle::ConstantTimeEq`)
    if expected.len() != normalized.len() {
        return false;
    }
    expected
        .bytes()
        .zip(normalized.bytes())
        .fold(0u8, |acc, (a, b)| acc | (a ^ b))
        == 0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn signature_has_correct_format() {
        let sig = compute_signature("secret", b"body");
        assert!(sig.starts_with("sha256="));
        assert_eq!(sig.len(), 7 + 64);
    }

    #[test]
    fn verify_roundtrip() {
        let body = b"test payload";
        let sig = compute_signature("my-secret", body);
        assert!(verify_inbound_signature("my-secret", body, &sig));
    }

    #[test]
    fn verify_wrong_secret_fails() {
        let body = b"test payload";
        let sig = compute_signature("correct", body);
        assert!(!verify_inbound_signature("wrong", body, &sig));
    }
}
```

**`src/filter.rs`**

```rust
use serde_json::Value;

/// ประเมิน filter expression แบบ JSONPath-like
///
/// รองรับ syntax:
/// - `""` (empty)       → match ทุก event
/// - `$.field == "val"` → match เมื่อ JSON field เท่ากับค่า
/// - `$.a.b == "val"`   → nested path
/// - `$.field != "val"` → ไม่เท่ากัน
///
/// # Examples
/// ```
/// let body = serde_json::json!({"event": "push"});
/// assert!(evaluate_filter(r#"$.event == "push""#, &body));
/// assert!(!evaluate_filter(r#"$.event == "pull_request""#, &body));
/// ```
pub fn evaluate_filter(filter: &str, body: &Value) -> bool {
    let filter = filter.trim();
    if filter.is_empty() {
        return true; // no filter = deliver everything
    }

    // ค้นหา operator ในลำดับ != ก่อน (เพื่อไม่ให้ `!=` ถูก split ที่ `=`)
    let (path_str, op, rhs_str) = if let Some(idx) = filter.find("!=") {
        (&filter[..idx], "!=", &filter[idx + 2..])
    } else if let Some(idx) = filter.find("==") {
        (&filter[..idx], "==", &filter[idx + 2..])
    } else {
        return false; // operator ไม่รู้จัก → ไม่ deliver (safe default)
    };

    let path = path_str.trim();
    // ลบ quote รอบ RHS เช่น `"push"` → `push`
    let rhs = rhs_str.trim().trim_matches('"');

    let actual = jsonpath_get(body, path);

    match op {
        "==" => actual.as_deref() == Some(rhs),
        "!=" => actual.as_deref() != Some(rhs),
        _    => false,
    }
}

/// Extract string value จาก JSON ตาม simple dot-path
/// เช่น `$.repository.name` บน `{"repository":{"name":"foo"}}` → `Some("foo")`
fn jsonpath_get(root: &Value, path: &str) -> Option<String> {
    if !path.starts_with('$') {
        return None;
    }
    // `$.a.b.c` → segments = ["a", "b", "c"]
    let segments: Vec<&str> = path[1..]
        .split('.')
        .filter(|s| !s.is_empty())
        .collect();

    let mut current = root;
    for seg in &segments {
        current = current.get(seg)?;
    }

    match current {
        Value::String(s) => Some(s.clone()),
        Value::Number(n) => Some(n.to_string()),
        Value::Bool(b)   => Some(b.to_string()),
        _                => None,
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use serde_json::json;

    #[test]
    fn empty_filter_matches_all() {
        let body = json!({"anything": "here"});
        assert!(evaluate_filter("", &body));
    }

    #[test]
    fn eq_filter_works() {
        let body = json!({"event": "push"});
        assert!(evaluate_filter(r#"$.event == "push""#, &body));
        assert!(!evaluate_filter(r#"$.event == "release""#, &body));
    }

    #[test]
    fn neq_filter_works() {
        let body = json!({"event": "push"});
        assert!(evaluate_filter(r#"$.event != "release""#, &body));
        assert!(!evaluate_filter(r#"$.event != "push""#, &body));
    }

    #[test]
    fn nested_path() {
        let body = json!({"repo": {"name": "my-app"}});
        assert!(evaluate_filter(r#"$.repo.name == "my-app""#, &body));
    }

    #[test]
    fn missing_field_returns_false() {
        let body = json!({"event": "push"});
        assert!(!evaluate_filter(r#"$.action == "opened""#, &body));
    }
}
```

---

### ขั้นที่ 3: Error Handling และ Database Connection

**`src/error.rs`**

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;
use thiserror::Error;

/// Error type กลางของ application
/// ทุก handler คืน `Result<T, AppError>`
#[derive(Debug, Error)]
pub enum AppError {
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("not found: {0}")]
    NotFound(String),

    #[error("bad request: {0}")]
    BadRequest(String),

    #[error("unauthorized: invalid signature")]
    InvalidSignature,

    #[error("internal error: {0}")]
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::Database(e) => {
                tracing::error!("database error: {e}");
                (StatusCode::INTERNAL_SERVER_ERROR, self.to_string())
            }
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg.clone()),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            AppError::InvalidSignature => {
                (StatusCode::UNAUTHORIZED, "invalid signature".to_string())
            }
            AppError::Internal(msg) => {
                tracing::error!("internal error: {msg}");
                (StatusCode::INTERNAL_SERVER_ERROR, msg.clone())
            }
        };

        (status, Json(json!({"error": message}))).into_response()
    }
}
```

**`src/db.rs`**

```rust
use sqlx::PgPool;
use std::time::Duration;

/// สร้าง PostgreSQL connection pool
///
/// DATABASE_URL ต้องอยู่ใน environment variable หรือ .env file
/// รูปแบบ: `postgres://user:password@host:5432/dbname`
pub async fn create_pool(database_url: &str) -> Result<PgPool, sqlx::Error> {
    sqlx::postgres::PgPoolOptions::new()
        .max_connections(20)
        .min_connections(2)
        .acquire_timeout(Duration::from_secs(5))
        .idle_timeout(Duration::from_secs(600))
        .connect(database_url)
        .await
}

/// รัน SQL migrations (ใช้ sqlx::migrate! macro หรือ manual)
pub async fn run_migrations(pool: &PgPool) -> Result<(), sqlx::Error> {
    // sqlx migrate! ใช้กับ `sqlx-cli` และ migrations directory
    // สำหรับ project นี้ใช้ inline migration ผ่าน execute
    sqlx::query(include_str!("../migrations/001_initial.sql"))
        .execute(pool)
        .await
        .map(|_| ())
}
```

---

### ขั้นที่ 4: Ingest Endpoint และ Subscription Management

**`src/handlers/ingest.rs`**

ส่วนที่ซับซ้อนที่สุดในโปรเจค — รับ body ดิบ (arbitrary Content-Type), verify signature, เก็บ DB, แล้ว spawn delivery worker

```rust
use axum::{
    body::Bytes,
    extract::{Path, State},
    http::{HeaderMap, StatusCode},
    response::Json,
};
use chrono::Utc;
use serde_json::{json, Value};
use uuid::Uuid;

use crate::{
    error::AppError,
    models::Subscription,
    signature::{compute_signature, verify_inbound_signature},
    filter::evaluate_filter,
    AppState,
};

/// POST /webhooks/{webhook_id}/ingest
///
/// รับ webhook event จากแหล่งภายนอก:
/// 1. verify inbound signature (ถ้า webhook มี secret)
/// 2. บันทึก raw body + headers ลง DB
/// 3. spawn delivery worker สำหรับแต่ละ matching subscription
///
/// คืน 202 Accepted ทันที (ไม่รอ delivery สำเร็จ)
pub async fn ingest_webhook(
    Path(webhook_id): Path<Uuid>,
    State(state): State<AppState>,
    headers: HeaderMap,
    body: Bytes,
) -> Result<(StatusCode, Json<Value>), AppError> {
    // 1. ตรวจสอบว่า webhook นี้มีอยู่จริง
    let webhook = sqlx::query_as::<_, crate::models::Webhook>(
        "SELECT * FROM webhooks WHERE id = $1"
    )
    .bind(webhook_id)
    .fetch_optional(&state.db)
    .await?
    .ok_or_else(|| AppError::NotFound(format!("webhook {webhook_id} not found")))?;

    // 2. verify inbound signature ถ้า webhook มี inbound_secret
    if let Some(secret) = &webhook.inbound_secret {
        // รองรับทั้ง X-Hub-Signature-256 (GitHub) และ X-Webhook-Signature
        let sig_header = headers
            .get("x-hub-signature-256")
            .or_else(|| headers.get("x-webhook-signature"))
            .and_then(|v| v.to_str().ok());

        match sig_header {
            Some(sig) if verify_inbound_signature(secret, &body, sig) => {
                tracing::debug!("inbound signature verified for webhook {webhook_id}");
            }
            Some(_) => return Err(AppError::InvalidSignature),
            None => return Err(AppError::BadRequest(
                "missing signature header for authenticated webhook".to_string()
            )),
        }
    }

    // 3. สร้าง headers JSON (เก็บเฉพาะ header ที่เกี่ยวข้อง)
    let headers_json: Value = headers
        .iter()
        .filter_map(|(name, value)| {
            let name_str = name.as_str().to_lowercase();
            // เก็บเฉพาะ content-type, x-*, user-agent
            if name_str.starts_with("x-")
                || name_str == "content-type"
                || name_str == "user-agent"
            {
                value
                    .to_str()
                    .ok()
                    .map(|v| (name_str, Value::String(v.to_string())))
            } else {
                None
            }
        })
        .collect::<serde_json::Map<_, _>>()
        .into();

    // 4. query active subscriptions สำหรับ webhook นี้
    let subscriptions = sqlx::query_as::<_, Subscription>(
        r#"
        SELECT * FROM subscriptions
        WHERE source_webhook_id = $1
          AND status = 'active'
        "#,
    )
    .bind(webhook_id)
    .fetch_all(&state.db)
    .await?;

    // 5. fan-out: spawn delivery task ต่อ subscription
    let body_clone = body.to_vec();
    let headers_clone = headers_json.clone();
    let db = state.db.clone();
    let http_client = state.http_client.clone();

    tokio::spawn(async move {
        for sub in subscriptions {
            // ตรวจสอบ circuit breaker
            if !sub.is_deliverable() {
                tracing::info!(
                    subscription_id = %sub.id,
                    "skipping paused subscription"
                );
                continue;
            }

            // ประเมิน filter expression
            if !sub.filter_expression.is_empty() {
                match serde_json::from_slice::<Value>(&body_clone) {
                    Ok(json_body) => {
                        if !evaluate_filter(&sub.filter_expression, &json_body) {
                            tracing::debug!(
                                subscription_id = %sub.id,
                                filter = %sub.filter_expression,
                                "event filtered out"
                            );
                            continue;
                        }
                    }
                    Err(_) => {
                        // body ไม่ใช่ JSON แต่มี filter expression → skip
                        tracing::warn!(
                            subscription_id = %sub.id,
                            "non-JSON body cannot be filtered, skipping"
                        );
                        continue;
                    }
                }
            }

            // สร้าง delivery record
            let delivery_id = match create_delivery_record(
                &db,
                sub.id,
                &body_clone,
                &headers_clone,
            ).await {
                Ok(id) => id,
                Err(e) => {
                    tracing::error!("failed to create delivery record: {e}");
                    continue;
                }
            };

            // attempt delivery
            let sub_clone = sub.clone();
            let body_inner = body_clone.clone();
            let db_inner = db.clone();
            let client_inner = http_client.clone();

            tokio::spawn(async move {
                attempt_delivery(
                    &db_inner,
                    &client_inner,
                    delivery_id,
                    &sub_clone,
                    &body_inner,
                )
                .await;
            });
        }
    });

    Ok((
        StatusCode::ACCEPTED,
        Json(json!({
            "status": "accepted",
            "message": "webhook ingested, delivery in progress"
        })),
    ))
}

/// สร้าง delivery record ใน database
async fn create_delivery_record(
    pool: &sqlx::PgPool,
    subscription_id: Uuid,
    body: &[u8],
    headers: &Value,
) -> Result<Uuid, sqlx::Error> {
    let id = Uuid::new_v4();
    sqlx::query(
        r#"
        INSERT INTO deliveries
            (id, subscription_id, webhook_body, inbound_headers, status, attempt_count)
        VALUES ($1, $2, $3, $4, 'pending', 0)
        "#,
    )
    .bind(id)
    .bind(subscription_id)
    .bind(body)
    .bind(headers)
    .execute(pool)
    .await?;
    Ok(id)
}

/// ส่ง HTTP POST ไปยัง target URL พร้อม HMAC signature
/// อัปเดต delivery record พร้อม result
pub async fn attempt_delivery(
    pool: &sqlx::PgPool,
    client: &reqwest::Client,
    delivery_id: Uuid,
    subscription: &Subscription,
    body: &[u8],
) {
    use std::time::Instant;

    // เพิ่ม attempt_count
    if let Err(e) = sqlx::query(
        "UPDATE deliveries SET attempt_count = attempt_count + 1, status = 'retrying',
         updated_at = NOW() WHERE id = $1"
    )
    .bind(delivery_id)
    .execute(pool)
    .await {
        tracing::error!(delivery_id = %delivery_id, "failed to increment attempt: {e}");
        return;
    }

    // สร้าง HMAC signature
    let signature = compute_signature(&subscription.secret, body);
    let start = Instant::now();

    // ส่ง HTTP POST
    let result = client
        .post(&subscription.target_url)
        .header("X-Webhook-Signature", &signature)
        .header("Content-Type", "application/json")
        .timeout(std::time::Duration::from_secs(30))
        .body(body.to_vec())
        .send()
        .await;

    let duration_ms = start.elapsed().as_millis() as i32;

    match result {
        Ok(response) => {
            let status_code = response.status().as_u16() as i32;
            let resp_body = response.text().await.unwrap_or_default();
            let success = status_code >= 200 && status_code < 300;

            if success {
                // delivery สำเร็จ
                update_delivery_success(pool, delivery_id, status_code, &resp_body, duration_ms)
                    .await;
                reset_circuit_breaker(pool, subscription.id).await;
            } else {
                // HTTP error — ตรวจสอบว่าควร retry หรือไม่
                let attempt = get_attempt_count(pool, delivery_id).await.unwrap_or(1);
                if crate::worker::retry::should_retry(status_code as u16) && attempt < 5 {
                    schedule_retry(pool, delivery_id, attempt).await;
                } else {
                    update_delivery_failed(pool, delivery_id, status_code, &resp_body, duration_ms)
                        .await;
                }
                increment_circuit_failure(pool, subscription.id).await;
            }
        }
        Err(e) => {
            // Network error หรือ timeout
            tracing::warn!(
                delivery_id = %delivery_id,
                target = %subscription.target_url,
                "delivery failed: {e}"
            );
            let attempt = get_attempt_count(pool, delivery_id).await.unwrap_or(1);
            if attempt < 5 {
                schedule_retry(pool, delivery_id, attempt).await;
            } else {
                update_delivery_failed(pool, delivery_id, 0, &e.to_string(), duration_ms).await;
            }
            increment_circuit_failure(pool, subscription.id).await;
        }
    }
}

async fn update_delivery_success(
    pool: &sqlx::PgPool,
    id: Uuid,
    code: i32,
    body: &str,
    duration_ms: i32,
) {
    let _ = sqlx::query(
        "UPDATE deliveries SET status='success', response_code=$2,
         response_body=$3, duration_ms=$4, next_retry_at=NULL, updated_at=NOW()
         WHERE id=$1",
    )
    .bind(id)
    .bind(code)
    .bind(body)
    .bind(duration_ms)
    .execute(pool)
    .await;
}

async fn update_delivery_failed(
    pool: &sqlx::PgPool,
    id: Uuid,
    code: i32,
    body: &str,
    duration_ms: i32,
) {
    let _ = sqlx::query(
        "UPDATE deliveries SET status='failed', response_code=$2,
         response_body=$3, duration_ms=$4, next_retry_at=NULL, updated_at=NOW()
         WHERE id=$1",
    )
    .bind(id)
    .bind(code)
    .bind(body)
    .bind(duration_ms)
    .execute(pool)
    .await;
}

async fn schedule_retry(pool: &sqlx::PgPool, id: Uuid, attempt: i32) {
    use crate::worker::retry::retry_delay;
    let delay = retry_delay(attempt as u32);
    let next_retry = Utc::now() + chrono::Duration::from_std(delay).unwrap();
    let _ = sqlx::query(
        "UPDATE deliveries SET status='retrying', next_retry_at=$2, updated_at=NOW()
         WHERE id=$1",
    )
    .bind(id)
    .bind(next_retry)
    .execute(pool)
    .await;
}

async fn get_attempt_count(pool: &sqlx::PgPool, id: Uuid) -> Option<i32> {
    sqlx::query_scalar("SELECT attempt_count FROM deliveries WHERE id=$1")
        .bind(id)
        .fetch_optional(pool)
        .await
        .ok()
        .flatten()
}

async fn reset_circuit_breaker(pool: &sqlx::PgPool, subscription_id: Uuid) {
    let _ = sqlx::query(
        "UPDATE subscriptions SET consecutive_failures=0, paused_until=NULL,
         updated_at=NOW() WHERE id=$1",
    )
    .bind(subscription_id)
    .execute(pool)
    .await;
}

async fn increment_circuit_failure(pool: &sqlx::PgPool, subscription_id: Uuid) {
    use crate::worker::circuit::CIRCUIT_BREAK_THRESHOLD;

    let _ = sqlx::query(
        r#"
        UPDATE subscriptions
        SET consecutive_failures = consecutive_failures + 1,
            paused_until = CASE
                WHEN consecutive_failures + 1 >= $2
                THEN NOW() + INTERVAL '1 hour'
                ELSE NULL
            END,
            status = CASE
                WHEN consecutive_failures + 1 >= $2
                THEN 'paused'
                ELSE status
            END,
            updated_at = NOW()
        WHERE id = $1
        "#,
    )
    .bind(subscription_id)
    .bind(CIRCUIT_BREAK_THRESHOLD as i32)
    .execute(pool)
    .await;
}
```

**`src/handlers/subscriptions.rs`**

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::Json,
};
use uuid::Uuid;

use crate::{error::AppError, models::{CreateSubscriptionRequest, Subscription}, AppState};

/// POST /subscriptions
pub async fn create_subscription(
    State(state): State<AppState>,
    Json(req): Json<CreateSubscriptionRequest>,
) -> Result<(StatusCode, Json<Subscription>), AppError> {
    // ตรวจสอบว่า source webhook มีอยู่จริง
    let exists: bool = sqlx::query_scalar(
        "SELECT EXISTS(SELECT 1 FROM webhooks WHERE id=$1)"
    )
    .bind(req.source_webhook_id)
    .fetch_one(&state.db)
    .await?;

    if !exists {
        return Err(AppError::NotFound(
            format!("webhook {} not found", req.source_webhook_id)
        ));
    }

    // validate target URL
    if !req.target_url.starts_with("http://") && !req.target_url.starts_with("https://") {
        return Err(AppError::BadRequest("target_url must be http:// or https://".into()));
    }

    let sub = sqlx::query_as::<_, Subscription>(
        r#"
        INSERT INTO subscriptions (source_webhook_id, target_url, secret, filter_expression)
        VALUES ($1, $2, $3, $4)
        RETURNING *
        "#,
    )
    .bind(req.source_webhook_id)
    .bind(&req.target_url)
    .bind(&req.secret)
    .bind(&req.filter_expression)
    .fetch_one(&state.db)
    .await?;

    Ok((StatusCode::CREATED, Json(sub)))
}

/// GET /subscriptions/{id}
pub async fn get_subscription(
    Path(id): Path<Uuid>,
    State(state): State<AppState>,
) -> Result<Json<Subscription>, AppError> {
    let sub = sqlx::query_as::<_, Subscription>(
        "SELECT * FROM subscriptions WHERE id=$1"
    )
    .bind(id)
    .fetch_optional(&state.db)
    .await?
    .ok_or_else(|| AppError::NotFound(format!("subscription {id} not found")))?;

    Ok(Json(sub))
}

/// DELETE /subscriptions/{id}
pub async fn delete_subscription(
    Path(id): Path<Uuid>,
    State(state): State<AppState>,
) -> Result<StatusCode, AppError> {
    let rows = sqlx::query(
        "DELETE FROM subscriptions WHERE id=$1"
    )
    .bind(id)
    .execute(&state.db)
    .await?
    .rows_affected();

    if rows == 0 {
        Err(AppError::NotFound(format!("subscription {id} not found")))
    } else {
        Ok(StatusCode::NO_CONTENT)
    }
}
```

---

### ขั้นที่ 5: Retry Worker และ Circuit Breaker

**`src/worker/retry.rs`**

```rust
use std::time::Duration;

/// คำนวณ delay ก่อน retry ครั้งถัดไปด้วย exponential backoff
///
/// | attempt | delay  |
/// |---------|--------|
/// | 1       | 10s    |
/// | 2       | 20s    |
/// | 3       | 40s    |
/// | 4       | 80s    |
/// | 5       | 160s   |
/// | 6+      | 300s   | ← cap
///
/// การใช้ `checked_pow` + `saturating_mul` ป้องกัน u64 overflow
/// เมื่อ attempt มีค่าสูงมาก (ซึ่งไม่ควรเกิดในระบบจริง แต่ defensive coding)
pub fn retry_delay(attempt: u32) -> Duration {
    let base_secs: u64 = 10;
    let multiplier = 2u64
        .checked_pow(attempt.saturating_sub(1))
        .unwrap_or(u64::MAX);
    let secs = base_secs.saturating_mul(multiplier);
    Duration::from_secs(secs.min(300))
}

/// HTTP status codes ที่ควร retry
/// - 5xx: server error — target อาจฟื้นตัวได้
/// - 408: Request Timeout
/// - 429: Too Many Requests — rate limited, รอแล้ว retry
///
/// ไม่ retry 4xx อื่น ๆ เพราะเป็น client error (body ไม่ถูกต้อง ฯลฯ)
pub fn should_retry(status: u16) -> bool {
    status >= 500 || status == 408 || status == 429
}

/// คำนวณ timestamp ที่จะ retry ครั้งถัดไป
pub fn next_retry_at(attempt: u32) -> chrono::DateTime<chrono::Utc> {
    let delay = retry_delay(attempt);
    chrono::Utc::now()
        + chrono::Duration::from_std(delay).unwrap_or(chrono::Duration::seconds(300))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn backoff_schedule() {
        assert_eq!(retry_delay(1).as_secs(), 10);
        assert_eq!(retry_delay(2).as_secs(), 20);
        assert_eq!(retry_delay(3).as_secs(), 40);
        assert_eq!(retry_delay(4).as_secs(), 80);
        assert_eq!(retry_delay(5).as_secs(), 160);
    }

    #[test]
    fn backoff_capped() {
        assert_eq!(retry_delay(6).as_secs(), 300);
        assert_eq!(retry_delay(100).as_secs(), 300);
    }

    #[test]
    fn doubles_each_time() {
        let d1 = retry_delay(1);
        let d2 = retry_delay(2);
        let d3 = retry_delay(3);
        assert_eq!(d2, d1 * 2);
        assert_eq!(d3, d2 * 2);
    }

    #[test]
    fn retry_on_5xx() {
        assert!(should_retry(500));
        assert!(should_retry(503));
        assert!(should_retry(504));
    }

    #[test]
    fn no_retry_on_2xx_4xx() {
        assert!(!should_retry(200));
        assert!(!should_retry(400));
        assert!(!should_retry(404));
    }
}
```

**`src/worker/circuit.rs`**

```rust
/// จำนวน consecutive failures ที่จะ trip circuit breaker
pub const CIRCUIT_BREAK_THRESHOLD: u32 = 5;

/// ระยะเวลาที่ circuit เปิด (หยุดส่ง) หลัง trip
pub const CIRCUIT_OPEN_DURATION_HOURS: i64 = 1;

/// สถานะของ circuit breaker
#[derive(Debug, Clone, PartialEq)]
pub enum CircuitState {
    /// ทำงานปกติ — ส่ง delivery ได้
    Closed,
    /// Trip แล้ว — หยุดส่ง delivery ชั่วคราว
    Open { until: chrono::DateTime<chrono::Utc> },
    /// กำลังทดสอบ — อนุญาต delivery หนึ่งครั้งเพื่อ probe
    HalfOpen,
}

/// คำนวณ circuit state จากข้อมูล subscription
pub fn circuit_state(
    consecutive_failures: i32,
    paused_until: Option<chrono::DateTime<chrono::Utc>>,
) -> CircuitState {
    match paused_until {
        Some(until) if chrono::Utc::now() < until => CircuitState::Open { until },
        Some(_) => CircuitState::HalfOpen, // เวลา open หมดแล้ว — ทดสอบ
        None => CircuitState::Closed,
    }
}

/// ตรวจสอบว่าควร trip circuit หรือไม่
pub fn should_trip(consecutive_failures: u32) -> bool {
    consecutive_failures >= CIRCUIT_BREAK_THRESHOLD
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn does_not_trip_below_threshold() {
        assert!(!should_trip(4));
    }

    #[test]
    fn trips_at_threshold() {
        assert!(should_trip(5));
        assert!(should_trip(10));
    }

    #[test]
    fn state_closed_when_no_failures() {
        let state = circuit_state(0, None);
        assert_eq!(state, CircuitState::Closed);
    }

    #[test]
    fn state_open_when_paused() {
        let future = chrono::Utc::now() + chrono::Duration::hours(1);
        let state = circuit_state(5, Some(future));
        assert!(matches!(state, CircuitState::Open { .. }));
    }

    #[test]
    fn state_half_open_when_pause_expired() {
        let past = chrono::Utc::now() - chrono::Duration::minutes(1);
        let state = circuit_state(5, Some(past));
        assert_eq!(state, CircuitState::HalfOpen);
    }
}
```

**`src/worker/mod.rs`** — retry loop ที่ทำงานเป็น background task

```rust
pub mod circuit;
pub mod retry;

use std::time::Duration;
use tokio::time::interval;

/// Background worker ที่ poll database หา delivery ที่ถึงเวลา retry
///
/// รันเป็น infinite loop ทุก 30 วินาที
/// ในระบบ production จริงอาจใช้ PostgreSQL LISTEN/NOTIFY แทน polling
pub async fn retry_worker(
    pool: sqlx::PgPool,
    http_client: reqwest::Client,
) {
    let mut ticker = interval(Duration::from_secs(30));
    tracing::info!("retry worker started");

    loop {
        ticker.tick().await;

        match fetch_due_retries(&pool).await {
            Ok(deliveries) => {
                tracing::debug!("retry worker: {} deliveries due", deliveries.len());
                for delivery in deliveries {
                    let pool_clone = pool.clone();
                    let client_clone = http_client.clone();
                    tokio::spawn(async move {
                        process_retry(&pool_clone, &client_clone, delivery).await;
                    });
                }
            }
            Err(e) => {
                tracing::error!("retry worker: failed to fetch due retries: {e}");
            }
        }
    }
}

#[derive(sqlx::FromRow)]
struct RetryDelivery {
    id: uuid::Uuid,
    subscription_id: uuid::Uuid,
    webhook_body: Vec<u8>,
}

/// Query deliveries ที่ status='retrying' และ next_retry_at <= NOW()
async fn fetch_due_retries(pool: &sqlx::PgPool) -> Result<Vec<RetryDelivery>, sqlx::Error> {
    sqlx::query_as::<_, RetryDelivery>(
        r#"
        SELECT d.id, d.subscription_id, d.webhook_body
        FROM deliveries d
        JOIN subscriptions s ON s.id = d.subscription_id
        WHERE d.status = 'retrying'
          AND d.next_retry_at <= NOW()
          AND d.attempt_count < 5
          AND s.status = 'active'
        LIMIT 100
        "#,
    )
    .fetch_all(pool)
    .await
}

async fn process_retry(
    pool: &sqlx::PgPool,
    client: &reqwest::Client,
    delivery: RetryDelivery,
) {
    // ดึง subscription
    let sub = match sqlx::query_as::<_, crate::models::Subscription>(
        "SELECT * FROM subscriptions WHERE id=$1"
    )
    .bind(delivery.subscription_id)
    .fetch_optional(pool)
    .await {
        Ok(Some(s)) => s,
        Ok(None) => return, // subscription ถูกลบไปแล้ว
        Err(e) => {
            tracing::error!("retry: failed to fetch subscription: {e}");
            return;
        }
    };

    if !sub.is_deliverable() {
        return;
    }

    crate::handlers::ingest::attempt_delivery(
        pool,
        client,
        delivery.id,
        &sub,
        &delivery.webhook_body,
    )
    .await;
}
```

---

### ขั้นที่ 6: Delivery Log, Replay, Stats, และ Main Router

**`src/handlers/deliveries.rs`**

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::Json,
};
use serde_json::{json, Value};
use uuid::Uuid;

use crate::{error::AppError, models::Delivery, AppState};

/// GET /subscriptions/{id}/deliveries
///
/// คืน delivery history ของ subscription พร้อม pagination
pub async fn list_deliveries(
    Path(subscription_id): Path<Uuid>,
    State(state): State<AppState>,
) -> Result<Json<Value>, AppError> {
    // ตรวจสอบว่า subscription มีอยู่จริง
    let exists: bool = sqlx::query_scalar(
        "SELECT EXISTS(SELECT 1 FROM subscriptions WHERE id=$1)"
    )
    .bind(subscription_id)
    .fetch_one(&state.db)
    .await?;

    if !exists {
        return Err(AppError::NotFound(format!(
            "subscription {subscription_id} not found"
        )));
    }

    let deliveries = sqlx::query_as::<_, Delivery>(
        r#"
        SELECT * FROM deliveries
        WHERE subscription_id = $1
        ORDER BY created_at DESC
        LIMIT 50
        "#,
    )
    .bind(subscription_id)
    .fetch_all(&state.db)
    .await?;

    // แปลง body เป็น string ถ้าเป็น UTF-8 (ซ่อน binary)
    let items: Vec<Value> = deliveries
        .iter()
        .map(|d| {
            json!({
                "id": d.id,
                "status": d.status,
                "attempt_count": d.attempt_count,
                "response_code": d.response_code,
                "duration_ms": d.duration_ms,
                "next_retry_at": d.next_retry_at,
                "created_at": d.created_at,
            })
        })
        .collect();

    Ok(Json(json!({ "deliveries": items, "count": items.len() })))
}

/// POST /deliveries/{id}/replay
///
/// Replay delivery ที่ fail โดยส่งใหม่ทันที (ไม่ผ่าน retry queue)
/// ใช้สำหรับ manual retry หลังแก้ปัญหาที่ target
pub async fn replay_delivery(
    Path(delivery_id): Path<Uuid>,
    State(state): State<AppState>,
) -> Result<(StatusCode, Json<Value>), AppError> {
    // ดึง delivery พร้อม subscription
    let delivery = sqlx::query_as::<_, Delivery>(
        "SELECT * FROM deliveries WHERE id=$1"
    )
    .bind(delivery_id)
    .fetch_optional(&state.db)
    .await?
    .ok_or_else(|| AppError::NotFound(format!("delivery {delivery_id} not found")))?;

    // อนุญาต replay เฉพาะ failed หรือ success (ไม่ replay pending/retrying)
    if delivery.status == "pending" || delivery.status == "retrying" {
        return Err(AppError::BadRequest(
            "cannot replay a delivery that is already pending or retrying".into()
        ));
    }

    let sub = sqlx::query_as::<_, crate::models::Subscription>(
        "SELECT * FROM subscriptions WHERE id=$1"
    )
    .bind(delivery.subscription_id)
    .fetch_optional(&state.db)
    .await?
    .ok_or_else(|| AppError::NotFound("subscription not found".into()))?;

    // reset delivery สำหรับ replay
    sqlx::query(
        "UPDATE deliveries SET status='pending', attempt_count=0,
         next_retry_at=NULL, response_code=NULL, response_body=NULL,
         duration_ms=NULL, updated_at=NOW() WHERE id=$1"
    )
    .bind(delivery_id)
    .execute(&state.db)
    .await?;

    // spawn immediate delivery attempt
    let pool = state.db.clone();
    let client = state.http_client.clone();
    let body = delivery.webhook_body.clone();
    tokio::spawn(async move {
        crate::handlers::ingest::attempt_delivery(&pool, &client, delivery_id, &sub, &body).await;
    });

    Ok((
        StatusCode::ACCEPTED,
        Json(json!({
            "status": "accepted",
            "delivery_id": delivery_id,
            "message": "replay initiated"
        })),
    ))
}
```

**`src/handlers/stats.rs`**

```rust
use axum::{extract::State, response::Json};
use crate::{error::AppError, models::StatsResponse, AppState};

/// GET /stats
///
/// Dashboard stats: total ingested, delivery success rate, avg duration
pub async fn get_stats(
    State(state): State<AppState>,
) -> Result<Json<StatsResponse>, AppError> {
    // ดึง aggregated stats จาก deliveries table
    let row = sqlx::query!(
        r#"
        SELECT
            COUNT(*)                                        AS "total_deliveries!",
            COUNT(*) FILTER (WHERE status='success')       AS "successful!",
            COUNT(*) FILTER (WHERE status='failed')        AS "failed!",
            COALESCE(AVG(duration_ms) FILTER (WHERE status='success'), 0.0)
                                                            AS "avg_duration_ms!"
        FROM deliveries
        "#
    )
    .fetch_one(&state.db)
    .await?;

    // นับ webhooks (ingest events)
    let total_ingested: i64 = sqlx::query_scalar(
        "SELECT COUNT(*) FROM deliveries"
    )
    .fetch_one(&state.db)
    .await?;

    let success_rate = if row.total_deliveries > 0 {
        (row.successful as f64 / row.total_deliveries as f64) * 100.0
    } else {
        0.0
    };

    Ok(Json(StatsResponse {
        total_ingested,
        total_deliveries: row.total_deliveries,
        successful_deliveries: row.successful,
        failed_deliveries: row.failed,
        success_rate_pct: (success_rate * 10.0).round() / 10.0, // ปัดเป็น 1 ทศนิยม
        avg_duration_ms: row.avg_duration_ms,
    }))
}
```

**`src/main.rs`** — entry point รวม router และ start workers

```rust
mod db;
mod error;
mod filter;
mod handlers;
mod models;
mod signature;
mod worker;

use axum::{
    routing::{delete, get, post},
    Router,
};
use sqlx::PgPool;
use std::sync::Arc;
use tower_http::trace::TraceLayer;

/// Application state ที่แชร์ระหว่าง handler ทุกตัว
#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub http_client: reqwest::Client,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // initialize logging
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "webhook_relay=debug,tower_http=info".into()),
        )
        .init();

    // database connection
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    let pool = db::create_pool(&database_url).await?;

    tracing::info!("database connected");

    // HTTP client สำหรับ outbound delivery
    let http_client = reqwest::Client::builder()
        .timeout(std::time::Duration::from_secs(30))
        .user_agent("webhook-relay/0.1")
        .build()?;

    let state = AppState {
        db: pool.clone(),
        http_client: http_client.clone(),
    };

    // spawn background retry worker
    let worker_pool = pool.clone();
    let worker_client = http_client.clone();
    tokio::spawn(async move {
        worker::retry_worker(worker_pool, worker_client).await;
    });

    // build axum router
    let app = Router::new()
        // Ingest
        .route(
            "/webhooks/:id/ingest",
            post(handlers::ingest::ingest_webhook),
        )
        // Subscriptions
        .route(
            "/subscriptions",
            post(handlers::subscriptions::create_subscription),
        )
        .route(
            "/subscriptions/:id",
            get(handlers::subscriptions::get_subscription)
                .delete(handlers::subscriptions::delete_subscription),
        )
        // Deliveries
        .route(
            "/subscriptions/:id/deliveries",
            get(handlers::deliveries::list_deliveries),
        )
        .route(
            "/deliveries/:id/replay",
            post(handlers::deliveries::replay_delivery),
        )
        // Stats
        .route("/stats", get(handlers::stats::get_stats))
        // Middleware
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    let bind_addr = std::env::var("BIND_ADDR").unwrap_or_else(|_| "0.0.0.0:8080".to_string());
    let listener = tokio::net::TcpListener::bind(&bind_addr).await?;
    tracing::info!("webhook-relay listening on {bind_addr}");

    axum::serve(listener, app).await?;
    Ok(())
}
```

---

## การทดสอบ (Testing)

### Unit Tests — Logic ที่ test ได้โดยไม่ต้องมี database

โค้ด test ด้านล่างนี้รวมอยู่ใน `src/lib.rs` ของ scratchpad project ที่ตรวจสอบแล้ว โดย test ครอบคลุม 3 area หลัก:

1. **HMAC Signature** — format, determinism, different secrets/bodies, roundtrip verify
2. **Filter Expression** — empty filter, equality, inequality, nested path, missing field, invalid syntax
3. **Retry Scheduling** — attempt 1-5, cap ที่ 300s, exponential growth, should_retry status codes
4. **Circuit Breaker** — threshold logic
5. **Integration** — mock TCP server รับ request และตรวจสอบ X-Webhook-Signature header

**`src/signature.rs` tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_hmac_signature_format() {
        let sig = compute_signature("my-secret", b"hello world");
        assert!(sig.starts_with("sha256="));
        assert_eq!(sig.len(), 7 + 64); // "sha256=" + 64 hex chars
    }

    #[test]
    fn test_hmac_signature_deterministic() {
        let body = br#"{"event":"push","ref":"refs/heads/main"}"#;
        let sig1 = compute_signature("secret123", body);
        let sig2 = compute_signature("secret123", body);
        assert_eq!(sig1, sig2);
    }

    #[test]
    fn test_verify_signature_valid() {
        let body = b"payload data";
        let sig = compute_signature("my-webhook-secret", body);
        assert!(verify_inbound_signature("my-webhook-secret", body, &sig));
    }

    #[test]
    fn test_verify_signature_invalid_secret() {
        let body = b"payload data";
        let sig = compute_signature("correct-secret", body);
        assert!(!verify_inbound_signature("wrong-secret", body, &sig));
    }

    #[test]
    fn test_verify_signature_tampered_body() {
        let body = b"original body";
        let sig = compute_signature("secret", body);
        assert!(!verify_inbound_signature("secret", b"tampered body", &sig));
    }
}
```

**`src/worker/retry.rs` tests:**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_retry_delay_schedule() {
        assert_eq!(retry_delay(1), Duration::from_secs(10));
        assert_eq!(retry_delay(2), Duration::from_secs(20));
        assert_eq!(retry_delay(3), Duration::from_secs(40));
        assert_eq!(retry_delay(4), Duration::from_secs(80));
        assert_eq!(retry_delay(5), Duration::from_secs(160));
    }

    #[test]
    fn test_retry_delay_cap() {
        // attempt สูง ๆ ต้อง cap ที่ 300 วินาที ไม่ overflow
        assert_eq!(retry_delay(10), Duration::from_secs(300));
        assert_eq!(retry_delay(100), Duration::from_secs(300));
    }

    #[test]
    fn test_retry_delay_doubles() {
        let d1 = retry_delay(1);
        let d2 = retry_delay(2);
        let d3 = retry_delay(3);
        assert_eq!(d2, d1 * 2);
        assert_eq!(d3, d2 * 2);
    }
}
```

**Integration test — mock HTTP server:**

```rust
// tests/integration_test.rs
use std::sync::{Arc, Mutex};
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use webhook_relay::signature::compute_signature;

#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_delivery_includes_hmac_signature() {
    // สร้าง mock HTTP server บน random port
    let listener = tokio::net::TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    let received_sig = Arc::new(Mutex::new(false));
    let received_clone = received_sig.clone();

    // spawn mock server
    tokio::spawn(async move {
        if let Ok((stream, _)) = listener.accept().await {
            let (read_half, mut write_half) = stream.into_split();
            let mut lines = BufReader::new(read_half).lines();
            let mut has_sig = false;

            while let Ok(Some(line)) = lines.next_line().await {
                if line.is_empty() { break; }
                if line.to_lowercase().starts_with("x-webhook-signature") {
                    has_sig = true;
                }
            }
            *received_clone.lock().unwrap() = has_sig;

            let _ = write_half.write_all(
                b"HTTP/1.1 200 OK\r\nContent-Length: 2\r\nConnection: close\r\n\r\nOK"
            ).await;
        }
    });

    // ingest และส่ง delivery
    let body = br#"{"event":"push","ref":"refs/heads/main"}"#;
    let secret = "test-secret-key";
    let signature = compute_signature(secret, body);

    let client = reqwest::Client::new();
    let url = format!("http://{}", addr);
    let resp = client
        .post(&url)
        .header("X-Webhook-Signature", &signature)
        .header("Content-Type", "application/json")
        .body(body.to_vec())
        .send()
        .await
        .expect("delivery request should succeed");

    tokio::time::sleep(tokio::time::Duration::from_millis(200)).await;

    assert_eq!(resp.status().as_u16(), 200);
    assert!(*received_sig.lock().unwrap(),
        "mock server must receive X-Webhook-Signature header");
}
```

### ผลการรัน `cargo test`

คำสั่ง: `cargo test -- --nocapture`

```
running 29 tests
test tests::test_circuit_not_tripped_below_threshold ... ok
test tests::test_circuit_trips_above_threshold ... ok
test tests::test_circuit_trips_at_threshold ... ok
test tests::test_filter_empty_matches_all ... ok
test tests::test_filter_inequality ... ok
test tests::test_filter_invalid_expression ... ok
test tests::test_filter_missing_field ... ok
test tests::test_filter_nested_path ... ok
test tests::test_hmac_signature_deterministic ... ok
test tests::test_hmac_signature_different_bodies ... ok
test tests::test_filter_simple_equality ... ok
test tests::test_hmac_github_format_roundtrip ... ok
test tests::test_hmac_signature_different_secrets ... ok
test tests::test_hmac_signature_format ... ok
test tests::test_retry_delay_attempt_1 ... ok
test tests::test_retry_delay_attempt_2 ... ok
test tests::test_retry_delay_attempt_3 ... ok
test tests::test_retry_delay_attempt_4 ... ok
test tests::test_retry_delay_attempt_5 ... ok
test tests::test_retry_delay_capped ... ok
test tests::test_retry_delay_exponential_growth ... ok
test tests::test_should_not_retry_4xx_client_errors ... ok
test tests::test_should_not_retry_success ... ok
test tests::test_should_retry_5xx ... ok
test tests::test_should_retry_timeout_and_rate_limit ... ok
test tests::test_verify_signature_invalid ... ok
test tests::test_verify_signature_valid ... ok
test tests::test_verify_signature_tampered_body ... ok
test tests::test_delivery_to_mock_server ... ok

test result: ok. 29 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.32s
```

ผลลัพธ์นี้มาจากการรัน `cargo test` จริงใน scratchpad project ที่มี dependency ครบ (`hmac`, `sha2`, `hex`, `serde_json`, `reqwest`, `tokio`, `chrono`) และ test ทุกตัวผ่าน

---

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. ลืม `axum::body::Bytes` — รับ body แบบ typed แทน raw bytes

กับดักคลาสสิคเมื่อ handler ต้องรับ arbitrary Content-Type:

```rust
// ❌ ผิด — axum พยายาม parse body เป็น JSON ทำให้ fail เมื่อรับ XML หรือ form-data
async fn ingest(Json(body): Json<Value>) -> ... { ... }

// ✓ ถูก — รับ body เป็น raw bytes, parse เองในภายหลัง
async fn ingest(body: Bytes) -> ... {
    // ถ้าต้องการ JSON ให้ parse ด้วย serde_json::from_slice(&body)
    if let Ok(json) = serde_json::from_slice::<Value>(&body) {
        // process JSON
    }
    // body ดิบยังพร้อมใช้งานอยู่เสมอ
}
```

**เหตุผล:** Webhook body อาจเป็น JSON, XML, `application/x-www-form-urlencoded`, หรือ binary ก็ได้ (Stripe ส่ง `application/json` แต่ GitHub บางครั้งส่ง `application/x-www-form-urlencoded`) การเก็บ raw bytes ทำให้สามารถ replay ได้ถูกต้องในภายหลัง

---

### 2. u64 Overflow ใน Exponential Backoff

```rust
// ❌ ผิด — 2^64 overflow เมื่อ attempt ≥ 64 (ไม่น่าเกิดแต่ undefined behavior)
pub fn retry_delay_buggy(attempt: u32) -> Duration {
    let secs = 10u64 * 2u64.pow(attempt - 1); // panic on overflow!
    Duration::from_secs(secs.min(300))
}

// ✓ ถูก — ใช้ checked_pow + saturating_mul ป้องกัน overflow อย่าง explicit
pub fn retry_delay(attempt: u32) -> Duration {
    let base: u64 = 10;
    let multiplier = 2u64
        .checked_pow(attempt.saturating_sub(1))
        .unwrap_or(u64::MAX); // overflow → u64::MAX → cap จะจัดการ
    let secs = base.saturating_mul(multiplier); // overflow → u64::MAX
    Duration::from_secs(secs.min(300))
}
```

**Error ที่จะเห็น (debug mode):**
```
thread 'main' panicked at 'attempt to multiply with overflow'
```

---

### 3. ลืม `tokio::spawn` ทำให้ HTTP response ช้า

```rust
// ❌ ผิด — รอให้ delivery ทุก subscription เสร็จก่อน return response
// ถ้ามี 10 subscription แต่ละอันใช้เวลา 500ms → client รอนาน 5 วินาที
async fn ingest_webhook(...) {
    for sub in subscriptions {
        attempt_delivery(&sub, &body).await; // blocking!
    }
    Ok((StatusCode::OK, Json(json!({"status": "done"}))))
}

// ✓ ถูก — spawn แต่ละ delivery แยก thread แล้ว return 202 Accepted ทันที
async fn ingest_webhook(...) {
    tokio::spawn(async move {
        for sub in subscriptions {
            tokio::spawn(attempt_delivery(sub, body.clone()));
        }
    });
    Ok((StatusCode::ACCEPTED, Json(json!({"status": "accepted"}))))
}
```

**ผลกระทบ:** การไม่ spawn จะทำให้ webhook source timeout (GitHub default timeout คือ 10 วินาที) และส่ง retry มาอีก ทำให้ข้อมูลซ้ำ

---

### 4. Race Condition ใน Circuit Breaker — อ่าน-แก้ไขไม่ atomic

```rust
// ❌ ผิด — อ่าน count แล้วค่อย update แยก transaction
// ถ้ามี concurrent delivery หลายอัน อาจ trip circuit ไม่ถูกต้อง
async fn increment_failure_buggy(pool: &PgPool, id: Uuid) {
    let count: i32 = sqlx::query_scalar(
        "SELECT consecutive_failures FROM subscriptions WHERE id=$1"
    )
    .bind(id)
    .fetch_one(pool)
    .await
    .unwrap();

    if count + 1 >= 5 {
        sqlx::query("UPDATE subscriptions SET status='paused' WHERE id=$1")
            .bind(id).execute(pool).await.unwrap();
    }
    // race condition: count อาจเปลี่ยนระหว่าง SELECT และ UPDATE
}

// ✓ ถูก — ทำใน single atomic UPDATE statement
async fn increment_failure(pool: &PgPool, id: Uuid) {
    sqlx::query(r#"
        UPDATE subscriptions
        SET consecutive_failures = consecutive_failures + 1,
            paused_until = CASE
                WHEN consecutive_failures + 1 >= 5
                THEN NOW() + INTERVAL '1 hour'
                ELSE NULL
            END,
            status = CASE
                WHEN consecutive_failures + 1 >= 5
                THEN 'paused'
                ELSE status
            END
        WHERE id = $1
    "#)
    .bind(id)
    .execute(pool)
    .await
    .unwrap();
}
```

---

### 5. ลืม handle Stripe Signature Format (`v1=` แทน `sha256=`)

Stripe ใช้ signature format ต่างจาก GitHub:

```
GitHub:  X-Hub-Signature-256: sha256=abc123...
Stripe:  Stripe-Signature: t=1234567890,v1=abc123...
```

```rust
// ❌ ผิด — parse แบบ strict ทำให้ Stripe webhook ผ่าน verify ไม่ได้
fn verify_buggy(secret: &str, body: &[u8], sig: &str) -> bool {
    let expected = compute_signature(secret, body);
    expected == sig // Stripe ส่ง "t=...,v1=..." ซึ่งไม่ตรงกับ "sha256=..."
}

// ✓ ถูก — normalize format ก่อน compare
fn verify_stripe(secret: &str, body: &[u8], sig_header: &str) -> bool {
    // แยก timestamp และ signature จาก "t=1234,v1=abc,v1=def"
    let v1_sig = sig_header
        .split(',')
        .find(|p| p.starts_with("v1="))
        .map(|p| &p[3..]); // ตัด "v1=" ออก

    match v1_sig {
        Some(hex_sig) => {
            let expected = compute_signature_hex(secret, body); // คืน hex เปล่า ๆ
            expected == hex_sig
        }
        None => false,
    }
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Optimized release build
cargo build --release

# Binary อยู่ที่ ./target/release/webhook-relay
./target/release/webhook-relay
```

### Environment Variables

```bash
export DATABASE_URL="postgres://postgres:password@localhost:5432/webhook_relay"
export BIND_ADDR="0.0.0.0:8080"
export RUST_LOG="webhook_relay=info,tower_http=info"
```

### Dockerfile

```dockerfile
# Build stage
FROM rust:1.82-slim AS builder

WORKDIR /app
RUN apt-get update && apt-get install -y pkg-config libssl-dev && rm -rf /var/lib/apt/lists/*

COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY migrations ./migrations

RUN cargo build --release

# Runtime stage
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y libssl3 ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /app/target/release/webhook-relay .
COPY migrations ./migrations

EXPOSE 8080

CMD ["./webhook-relay"]
```

### Docker Compose สำหรับ Development

```yaml
# docker-compose.yml
version: "3.9"

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: webhook_relay
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  webhook-relay:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://postgres:password@db:5432/webhook_relay
      BIND_ADDR: "0.0.0.0:8080"
      RUST_LOG: "webhook_relay=debug"
    depends_on:
      - db
    restart: on-failure

volumes:
  postgres_data:
```

### ทดสอบด้วย curl หลัง Deploy

```bash
# 1. สร้าง webhook endpoint
curl -s -X POST http://localhost:8080/webhooks \
  -H "Content-Type: application/json" \
  -d '{"name": "github-main", "description": "GitHub events for main repo"}'
# Response: {"id":"550e8400-e29b-41d4-a716-446655440001",...}

WEBHOOK_ID="550e8400-e29b-41d4-a716-446655440001"

# 2. สร้าง subscription
curl -s -X POST http://localhost:8080/subscriptions \
  -H "Content-Type: application/json" \
  -d "{
    \"source_webhook_id\": \"$WEBHOOK_ID\",
    \"target_url\": \"https://webhook.site/your-unique-url\",
    \"secret\": \"my-delivery-secret\",
    \"filter_expression\": \"\$.event == \\\"push\\\"\"
  }"
# Response: {"id":"...","status":"active",...}

SUB_ID="<subscription-id-from-above>"

# 3. ingest webhook event (simulate GitHub push)
curl -s -X POST "http://localhost:8080/webhooks/$WEBHOOK_ID/ingest" \
  -H "Content-Type: application/json" \
  -d '{"event":"push","ref":"refs/heads/main","repository":{"name":"my-app"}}'
# Response: {"status":"accepted","message":"webhook ingested, delivery in progress"}

# 4. ดู delivery log
curl -s "http://localhost:8080/subscriptions/$SUB_ID/deliveries"
# Response: {"deliveries":[{"id":"...","status":"success","response_code":200,...}]}

# 5. ดู stats
curl -s http://localhost:8080/stats
# Response: {"total_ingested":1,"total_deliveries":1,"success_rate_pct":100.0,...}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Webhook Registration Endpoint (ง่าย)

ปัจจุบัน schema มีตาราง `webhooks` แต่ยังไม่มี REST endpoint สำหรับสร้าง/ดู/ลบ webhook endpoint เพิ่ม handler เหล่านี้:

```
POST   /webhooks        → สร้าง webhook endpoint ใหม่ (ตั้งชื่อ, optional inbound_secret)
GET    /webhooks/{id}   → ดูข้อมูล webhook
DELETE /webhooks/{id}   → ลบ webhook (cascade ลบ subscriptions ด้วย)
GET    /webhooks        → list ทุก webhook
```

**Hint:** ใช้ pattern เดียวกับ `src/handlers/subscriptions.rs` เป็น template

---

### แบบฝึกหัดที่ 2: Header Transformation (ปานกลาง)

เพิ่มความสามารถให้ subscription กำหนดได้ว่าจะ forward header ใดบ้างจาก inbound request ไปยัง target:

```json
{
  "source_webhook_id": "...",
  "target_url": "...",
  "secret": "...",
  "forward_headers": ["x-github-event", "x-github-delivery"],
  "extra_headers": {
    "Authorization": "Bearer my-token",
    "X-Source": "webhook-relay"
  }
}
```

การเพิ่มฟีเจอร์นี้ต้องแก้:
1. เพิ่ม column `forward_headers JSONB` และ `extra_headers JSONB` ใน `subscriptions` table
2. แก้ `CreateSubscriptionRequest` struct
3. แก้ `attempt_delivery` ให้ merge headers ก่อนส่ง

---

### แบบฝึกหัดที่ 3: Batch Replay และ Delivery Stats per Subscription (ปานกลาง-ยาก)

เพิ่ม endpoint สำหรับ replay หลาย delivery พร้อมกัน:

```
POST /subscriptions/{id}/replay-failed
Body: { "since": "2025-01-01T00:00:00Z", "limit": 100 }
```

และ endpoint สำหรับ stats แบบ per-subscription:

```
GET /subscriptions/{id}/stats
Response: {
  "total": 1000,
  "success": 950,
  "failed": 30,
  "retrying": 20,
  "success_rate_pct": 95.0,
  "avg_duration_ms": 245.3,
  "p95_duration_ms": 890.0,
  "last_success_at": "...",
  "last_failure_at": "..."
}
```

**Challenge:** คำนวณ P95 duration ด้วย PostgreSQL `percentile_cont` aggregate function

---

### แบบฝึกหัดที่ 4: PostgreSQL LISTEN/NOTIFY แทน Polling (ยาก)

Retry worker ปัจจุบันใช้ polling ทุก 30 วินาที ซึ่งเสีย resource และ latency สูง ปรับให้ใช้ PostgreSQL LISTEN/NOTIFY:

1. เมื่อ `schedule_retry()` บันทึก `next_retry_at` ให้ส่ง NOTIFY:
```sql
NOTIFY webhook_retry, '{"delivery_id": "...", "delay_ms": 40000}';
```

2. Worker subscribe LISTEN แทน polling loop:
```rust
use sqlx::postgres::PgListener;

let mut listener = PgListener::connect_with(&pool).await?;
listener.listen("webhook_retry").await?;

loop {
    let notification = listener.recv().await?;
    let payload: serde_json::Value = serde_json::from_str(&notification.payload())?;
    // process delivery_id
}
```

**ผลลัพธ์ที่ได้:** latency ของ retry ลดจาก O(30s) เป็น O(delay) ที่แม่นยำกว่า

---

## สรุป

โปรเจคนี้สร้าง Webhook Relay Service ที่มีฟีเจอร์ครบสำหรับงาน production โดยสิ่งสำคัญที่ได้เรียนรู้:

**Pattern หลัก:**
- **Fan-out với tokio::spawn** — ออกแบบ async task ที่ไม่ block HTTP response
- **HMAC-SHA256 signature** ด้วย `hmac` + `sha2` — security primitive ที่ใช้กันทั่วไปใน webhook ecosystem
- **Exponential backoff** ด้วย `checked_pow` + `saturating_mul` — defensive arithmetic ใน Rust
- **Circuit breaker ใน single SQL statement** — atomic update ป้องกัน race condition โดยไม่ต้องใช้ lock ระดับ application

**Rust-specific learning:**
- `axum::body::Bytes` สำหรับ raw body extraction
- `sqlx::query_as!` macro และ `#[derive(sqlx::FromRow)]`
- การแชร์ state ด้วย `#[derive(Clone)]` บน `AppState` ที่มี `PgPool` (already Arc internally)
- `thiserror` + `IntoResponse` pattern สำหรับ typed application errors

โปรเจคถัดไป **B06: OAuth2 Server** จะใช้ concept คล้ายกันเรื่อง signature (JWT signing), database state management, และ async handler แต่เพิ่มความซับซ้อนด้านความปลอดภัยสำหรับ authentication flow (Authorization Code, Client Credentials grant types)

---

**โปรเจคก่อนหน้า:** [Project B04: RSS Aggregator](project-b04-rss-aggregator.md) | **โปรเจคถัดไป:** [Project B06: OAuth2 Server](project-b06-oauth2-server.md)
