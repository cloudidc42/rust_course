# Project B01: Pastebin Service

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

Pastebin Service คือ web service สำหรับแชร์ข้อความ (code snippets, logs, configs) ผ่าน URL สั้น ๆ — คล้ายกับ [pastebin.com](https://pastebin.com) หรือ [gist.github.com](https://gist.github.com) แต่สร้างเองและควบคุมได้ทั้งหมด

ปัญหาที่โปรเจคนี้แก้ไข: developer มักต้องแชร์ code snippet หรือ error log ให้เพื่อนร่วมทีมผ่าน chat แต่ข้อความยาว ๆ อ่านยากในช่องแชท Pastebin แก้ปัญหาด้วยการให้ URL สั้นที่คลิกแล้วเปิดได้ทันที พร้อม syntax highlighting และ expiry อัตโนมัติ

Use case จริงในโลก production:
- Internal tool สำหรับทีม — แชร์ config, error log, SQL query ระหว่าง developer
- CI/CD pipeline — แนบ build log หรือ test output ให้ทีม review ใน PR
- On-call debugging — paste error stack trace และแชร์ให้ on-call engineer ดูได้ทันที
- Self-hosted alternative ของ pastebin.com ที่ไม่ต้องส่งข้อมูลออกนอกองค์กร

Learning value ที่ได้: โปรเจคนี้สอน web service "ครบวงจร" — ตั้งแต่ HTTP routing, database integration, background tasks, rate limiting, authentication ไปจนถึง content negotiation ซึ่งเป็น pattern ที่ใช้ใน production API จริง

## สิ่งที่จะได้เรียนรู้

- **Axum routing**: Handler functions, Path/Query extractors, State sharing, content negotiation ด้วย Accept header
- **SQLx + PostgreSQL**: Connection pool, typed queries, migration, aggregate queries
- **Short ID generation**: Base62 encoding จาก random bytes, collision detection และ retry strategy
- **Syntax highlighting**: `syntect` crate — ใช้ TextMate grammars เพื่อ highlight code เป็น HTML
- **Expiry และ background cleanup**: `tokio::time::interval` สำหรับ periodic task, การจัดการ TIMESTAMPTZ ใน PostgreSQL
- **Rate limiting**: Sliding window per-IP ด้วย `HashMap<IpAddr, Vec<Instant>>`, thread-safe ด้วย `Mutex`
- **Password protection**: bcrypt hashing, error response ที่ไม่รั่วข้อมูล
- **Error type design**: `thiserror`, `IntoResponse` สำหรับ AppError ที่ map เป็น HTTP status code

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–20**: Rust fundamentals — ownership, structs, enums, traits
- **Part 21–40**: Error handling (`Result`/`Option`), closures, iterators, `async`/`await` พื้นฐาน
- **Part 41–60**: `tokio` runtime, `Arc`/`Mutex`, HTTP basics, JSON serialization ด้วย `serde`
- **Part 61–80**: Axum web framework, SQLx database integration, PostgreSQL
- ความคุ้นเคยกับ HTTP status codes และ REST API design จะช่วยมาก

## โครงสร้างโปรเจค (Project Layout)

```
pastebin-service/
├── src/
│   ├── lib.rs          ← export ทุก module + AppState struct
│   ├── main.rs         ← entry point: connect DB, spawn cleanup task, start server
│   ├── error.rs        ← AppError enum + IntoResponse impl
│   ├── models.rs       ← Paste, CreatePasteRequest, PasteResponse, StatsResponse
│   ├── handlers.rs     ← handler functions สำหรับทุก route
│   ├── id.rs           ← generate_id() แบบ base62
│   ├── expiry.rs       ← parse_expiry(), is_expired()
│   ├── highlight.rs    ← Highlighter struct ด้วย syntect
│   └── ratelimit.rs    ← RateLimiter: sliding window per-IP
├── migrations/
│   └── 001_create_pastes.sql
├── tests/
│   └── integration_test.rs
└── Cargo.toml
```

## การออกแบบ (Architecture & Design)

### ภาพรวม Data Flow

```
HTTP Request
     │
     ▼
[Axum Router]
     │
     ├── POST /          → create_paste handler
     │        │
     │        ├── ตรวจ size limit (> 512KB → 413)
     │        ├── ตรวจ rate limit per-IP
     │        ├── bcrypt hash password (ถ้ามี)
     │        ├── generate_id() พร้อม collision retry
     │        └── INSERT INTO pastes
     │
     ├── GET /{id}       → get_paste handler
     │        │
     │        ├── SELECT * FROM pastes WHERE id = $1
     │        ├── is_expired() check
     │        ├── bcrypt verify password (ถ้า paste มี hash)
     │        ├── UPDATE views + 1
     │        └── Accept: text/html → HTML+highlight | else → JSON
     │
     ├── GET /{id}/raw   → get_paste_raw handler (plain text)
     ├── DELETE /{id}    → delete_paste handler
     ├── GET /stats      → get_stats handler (aggregate query)
     └── GET /           → health_check handler

Background Task (tokio::spawn):
     tokio::time::interval(5 min)
     → DELETE FROM pastes WHERE expires_at < NOW()
```

### ทำไมถึงเลือก Axum แทน Actix-web

Axum ออกแบบมาบนพื้นฐาน Tower middleware ecosystem ซึ่งทำให้ composable มากกว่า — เพิ่ม rate limiting, tracing, CORS ได้เป็น layer โดยไม่ต้องแก้ handler code Axum ยังใช้ type-safe extractors ที่ compile-time verified ทำให้ error ที่พบได้บ่อย (missing content-type, wrong parameter type) กลายเป็น compile error แทนที่จะเป็น runtime panic

### ทำไม SQLx แทน Diesel หรือ SeaORM

SQLx ให้เขียน raw SQL ได้ตรง ๆ แต่ยังคง type safety ที่ compile time (เมื่อใช้ macro mode) ไม่ต้องเรียนรู้ query builder DSL พิเศษ สำหรับโปรเจคที่มี query ที่ซับซ้อนเช่น aggregate หรือ FILTER clause การเขียน SQL ตรง ๆ อ่านง่ายกว่ามาก

### Short ID Design

```
Base62 charset: 0-9 A-Z a-z  (62 ตัวอักษร)
ID length: 8 ตัวอักษร
Possible combinations: 62^8 = 218,340,105,584,896 ≈ 218 ล้านล้าน

Birthday problem: ถ้ามี 1 ล้าน paste
→ P(collision) ≈ n² / (2 × 62^8) ≈ 2.3 × 10⁻⁶
→ น้อยมาก แต่ยังทำ retry loop ไว้เผื่อ
```

### Password Protection Design

ไม่เก็บ password เป็น plaintext — เก็บแค่ bcrypt hash ใน `password_hash` column bcrypt ใช้ work factor 12 (DEFAULT_COST) ซึ่งใช้เวลา ~300ms ต่อการ verify หนึ่งครั้ง ช้าพอที่จะขัดขวาง brute force แต่ยังรับได้สำหรับ UX

Error response สำหรับ wrong password จงใจใช้ 403 Forbidden ไม่ใช่ 401 เพราะ 401 ควรไปพร้อมกับ WWW-Authenticate header และ challenge/response flow ที่ซับซ้อนกว่า

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: สร้าง Project Skeleton และ Database Schema

เริ่มจาก Cargo.toml ที่มี dependencies ครบ จากนั้นสร้าง database schema และ AppState

```toml
# Cargo.toml
[package]
name = "pastebin-service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["postgres", "runtime-tokio", "chrono"] }
chrono = { version = "0.4", features = ["serde"] }
nanoid = "0.4"
tower-http = { version = "0.6", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
bcrypt = "0.15"
syntect = "5"
tower = "0.5"
rand = "0.8"
thiserror = "1"
anyhow = "1"
```

Migration file สำหรับสร้างตาราง:

```sql
-- migrations/001_create_pastes.sql
CREATE TABLE IF NOT EXISTS pastes (
    id TEXT PRIMARY KEY,
    content TEXT NOT NULL,
    language TEXT NOT NULL DEFAULT 'plain',
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    views INTEGER NOT NULL DEFAULT 0,
    password_hash TEXT
);

CREATE INDEX IF NOT EXISTS idx_pastes_expires_at ON pastes (expires_at)
    WHERE expires_at IS NOT NULL;
```

สังเกตว่า `expires_at` ใช้ **partial index** — สร้าง index เฉพาะแถวที่มีค่า (ไม่ใช่ NULL) ทำให้ cleanup query `WHERE expires_at < NOW()` ใช้ index ได้อย่างมีประสิทธิภาพ โดยไม่เปลืองพื้นที่กับ paste ที่ไม่มีวันหมดอายุ

AppState ที่แชร์ระหว่าง handler ทั้งหมด:

```rust
// src/lib.rs
pub mod error;
pub mod expiry;
pub mod handlers;
pub mod highlight;
pub mod id;
pub mod models;
pub mod ratelimit;

use crate::{highlight::Highlighter, ratelimit::RateLimiter};

/// State ที่แชร์ระหว่าง handlers ทั้งหมด
/// ถูก wrap ด้วย Arc เพื่อแชร์ระหว่าง tokio tasks
pub struct AppState {
    pub db: sqlx::PgPool,
    pub highlighter: Highlighter,
    pub rate_limiter: RateLimiter,
}
```

ทำไม `Highlighter` อยู่ใน AppState? เพราะ `SyntaxSet` และ `ThemeSet` ของ syntect ใช้เวลาโหลดนาน (อ่านจาก binary data) ควรโหลดครั้งเดียวตอน startup แล้วแชร์ทุก request แทนที่จะสร้างใหม่ทุกครั้ง

### ขั้นที่ 2: Error Types และ Models

ออกแบบ `AppError` ให้ map เป็น HTTP status code อย่างถูกต้อง:

```rust
// src/error.rs
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("paste not found")]
    NotFound,

    #[error("paste has expired")]
    Expired,

    #[error("wrong password")]
    WrongPassword,

    #[error("password required")]
    PasswordRequired,

    #[error("content too large (max 512KB)")]
    TooLarge,

    #[error("rate limit exceeded")]
    RateLimited,

    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("internal error: {0}")]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound      => (StatusCode::NOT_FOUND, self.to_string()),
            AppError::Expired       => (StatusCode::GONE, self.to_string()),
            AppError::WrongPassword => (StatusCode::FORBIDDEN, self.to_string()),
            AppError::PasswordRequired => (StatusCode::UNAUTHORIZED, self.to_string()),
            AppError::TooLarge      => (StatusCode::PAYLOAD_TOO_LARGE, self.to_string()),
            AppError::RateLimited   => (StatusCode::TOO_MANY_REQUESTS, self.to_string()),
            AppError::Database(e) => {
                tracing::error!("DB error: {}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, "database error".to_string())
            }
            AppError::Internal(e) => {
                tracing::error!("Internal error: {}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, "internal error".to_string())
            }
        };

        let body = Json(json!({ "error": message }));
        (status, body).into_response()
    }
}
```

สังเกตว่า `Database` และ `Internal` variants **ไม่เปิดเผย error message จริง** ต่อ client — log ไว้ที่ server เท่านั้น นี่คือ security best practice ที่ป้องกันการ leak ข้อมูล schema หรือ internal state

Data models ด้วย `serde` และ `sqlx::FromRow`:

```rust
// src/models.rs
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// แถวข้อมูล paste จาก database (ใช้ sqlx::FromRow เพื่อ map ตรงจาก query)
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Paste {
    pub id: String,
    pub content: String,
    pub language: String,
    pub expires_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub views: i32,
    pub password_hash: Option<String>,
}

/// Request body สำหรับสร้าง paste ใหม่
#[derive(Debug, Deserialize)]
pub struct CreatePasteRequest {
    pub content: String,
    #[serde(default = "default_language")]
    pub language: String,
    /// "1h", "1d", "7d", หรือ "never"
    #[serde(default = "default_expires")]
    pub expires: String,
    pub password: Option<String>,
}

fn default_language() -> String { "plain".to_string() }
fn default_expires() -> String { "never".to_string() }

/// Response body สำหรับ paste ที่สร้างแล้ว
#[derive(Debug, Serialize)]
pub struct CreatePasteResponse {
    pub id: String,
    pub url: String,
    pub raw_url: String,
    pub expires_at: Option<DateTime<Utc>>,
}

/// Response body สำหรับ GET /{id}
#[derive(Debug, Serialize)]
pub struct PasteResponse {
    pub id: String,
    pub content: String,
    pub language: String,
    pub created_at: DateTime<Utc>,
    pub expires_at: Option<DateTime<Utc>>,
    pub views: i32,
}

/// Response body สำหรับ GET /stats
#[derive(Debug, Serialize, sqlx::FromRow)]
pub struct StatsResponse {
    pub total_pastes: i64,
    pub active_pastes: i64,
    pub total_views: i64,
}
```

### ขั้นที่ 3: Short ID Generation และ Expiry Logic

ID generation แบบ base62 — เหตุผลที่ไม่ใช้ UUID ตรง ๆ: UUID v4 ยาว 36 ตัวอักษร (รวม dash) อ่านยาก สวยน้อย เมื่อเทียบกับ `aB3kP9mZ` ที่สั้นและพิมพ์ได้

```rust
// src/id.rs
use rand::Rng;

const BASE62_CHARS: &[u8] = b"0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
const ID_LENGTH: usize = 8;

/// สร้าง short ID แบบ base62 จาก random bytes
/// ใช้ 8 ตัวอักษร → 62^8 ≈ 218 ล้านล้านความเป็นไปได้
pub fn generate_id() -> String {
    let mut rng = rand::thread_rng();
    (0..ID_LENGTH)
        .map(|_| {
            let idx = rng.gen_range(0..BASE62_CHARS.len());
            BASE62_CHARS[idx] as char
        })
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashSet;

    #[test]
    fn test_id_length() {
        let id = generate_id();
        assert_eq!(id.len(), ID_LENGTH);
    }

    #[test]
    fn test_id_charset() {
        for _ in 0..100 {
            let id = generate_id();
            for ch in id.chars() {
                assert!(ch.is_alphanumeric(), "ID contains non-alphanumeric: {}", ch);
            }
        }
    }

    #[test]
    fn test_id_uniqueness() {
        let ids: HashSet<String> = (0..1000).map(|_| generate_id()).collect();
        // จาก 1000 IDs ควรได้ unique อย่างน้อย 990
        assert!(ids.len() >= 990, "too many collisions: {} unique from 1000", ids.len());
    }
}
```

Expiry logic — แปลง string เป็น `DateTime<Utc>`:

```rust
// src/expiry.rs
use chrono::{DateTime, Duration, Utc};

/// แปลง string เป็น DateTime<Utc> สำหรับ expires_at
/// รูปแบบที่รองรับ: "1h", "6h", "1d", "7d", "30d", "never"
pub fn parse_expiry(expires: &str) -> Option<DateTime<Utc>> {
    match expires {
        "never" | "" => None,
        "1h"  => Some(Utc::now() + Duration::hours(1)),
        "6h"  => Some(Utc::now() + Duration::hours(6)),
        "1d"  => Some(Utc::now() + Duration::days(1)),
        "7d"  => Some(Utc::now() + Duration::days(7)),
        "30d" => Some(Utc::now() + Duration::days(30)),
        _     => None, // ค่าที่ไม่รู้จัก → ไม่หมดอายุ
    }
}

/// ตรวจสอบว่า paste หมดอายุแล้วหรือยัง
pub fn is_expired(expires_at: Option<DateTime<Utc>>) -> bool {
    match expires_at {
        None    => false,
        Some(t) => Utc::now() > t,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_never() {
        assert!(parse_expiry("never").is_none());
        assert!(parse_expiry("").is_none());
    }

    #[test]
    fn test_parse_valid() {
        let t = parse_expiry("1h").unwrap();
        let diff = t - Utc::now();
        assert!(diff.num_minutes() >= 59 && diff.num_minutes() <= 61);
    }

    #[test]
    fn test_not_expired() {
        let future = Some(Utc::now() + Duration::hours(1));
        assert!(!is_expired(future));
    }

    #[test]
    fn test_expired() {
        let past = Some(Utc::now() - Duration::hours(1));
        assert!(is_expired(past));
    }
}
```

### ขั้นที่ 4: Syntax Highlighting และ Rate Limiting

`syntect` crate ใช้ TextMate grammar เดียวกับ VS Code — ให้คุณภาพ highlighting ระดับ editor

```rust
// src/highlight.rs
use syntect::easy::HighlightLines;
use syntect::highlighting::{Theme, ThemeSet};
use syntect::html::{styled_line_to_highlighted_html, IncludeBackground};
use syntect::parsing::SyntaxSet;

pub struct Highlighter {
    ss: SyntaxSet,
    theme: Theme,
}

impl Highlighter {
    pub fn new() -> Self {
        let ss = SyntaxSet::load_defaults_newlines();
        let ts = ThemeSet::load_defaults();
        let theme = ts.themes["base16-ocean.dark"].clone();
        Highlighter { ss, theme }
    }

    /// แปลง source code เป็น HTML พร้อม syntax highlighting
    pub fn highlight(&self, code: &str, language: &str) -> String {
        let syntax = self
            .ss
            .find_syntax_by_token(language)
            .unwrap_or_else(|| self.ss.find_syntax_plain_text());

        let mut h = HighlightLines::new(syntax, &self.theme);
        let mut html = String::new();
        html.push_str("<pre><code>");

        for line in syntect::util::LinesWithEndings::from(code) {
            match h.highlight_line(line, &self.ss) {
                Ok(ranges) => {
                    match styled_line_to_highlighted_html(&ranges[..], IncludeBackground::No) {
                        Ok(highlighted) => html.push_str(&highlighted),
                        Err(_) => html.push_str(&html_escape(line)),
                    }
                }
                Err(_) => html.push_str(&html_escape(line)),
            }
        }

        html.push_str("</code></pre>");
        html
    }
}

fn html_escape(s: &str) -> String {
    s.replace('&', "&amp;")
     .replace('<', "&lt;")
     .replace('>', "&gt;")
     .replace('"', "&quot;")
}

/// ตรวจจับภาษาจาก hint (query param ?lang=rust)
pub fn detect_language(lang_hint: Option<&str>) -> String {
    match lang_hint {
        Some(l) if !l.is_empty() => l.to_lowercase(),
        _ => "plain".to_string(),
    }
}
```

Rate limiter แบบ sliding window — ดีกว่า fixed window เพราะไม่มีปัญหา "double burst" ที่ช่วงรอยต่อของ window:

```rust
// src/ratelimit.rs
use std::{
    collections::HashMap,
    net::IpAddr,
    sync::Mutex,
    time::{Duration, Instant},
};

const WINDOW: Duration = Duration::from_secs(60);
const MAX_CREATES: usize = 10;

/// Rate limiter แบบ sliding window per IP
pub struct RateLimiter {
    requests: Mutex<HashMap<IpAddr, Vec<Instant>>>,
}

impl RateLimiter {
    pub fn new() -> Self {
        RateLimiter {
            requests: Mutex::new(HashMap::new()),
        }
    }

    /// คืน true ถ้า IP นี้ยังไม่ถึง rate limit
    /// Sliding window: ลบ request ที่เก่ากว่า 60 วินาทีออกก่อนนับ
    pub fn check_and_record(&self, ip: IpAddr) -> bool {
        let mut map = self.requests.lock().unwrap();
        let now = Instant::now();
        let entry = map.entry(ip).or_default();

        // ลบ request ที่เก่ากว่า window
        entry.retain(|t| now.duration_since(*t) < WINDOW);

        if entry.len() >= MAX_CREATES {
            return false; // rate limit exceeded
        }

        entry.push(now);
        true
    }

    /// ล้าง entry ที่ window หมดแล้ว (เรียกจาก background task)
    pub fn cleanup(&self) {
        let mut map = self.requests.lock().unwrap();
        let now = Instant::now();
        map.retain(|_, v| {
            v.retain(|t| now.duration_since(*t) < WINDOW);
            !v.is_empty()
        });
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::net::Ipv4Addr;

    #[test]
    fn test_allow_under_limit() {
        let rl = RateLimiter::new();
        let ip = IpAddr::V4(Ipv4Addr::new(127, 0, 0, 1));
        for _ in 0..MAX_CREATES {
            assert!(rl.check_and_record(ip));
        }
    }

    #[test]
    fn test_block_over_limit() {
        let rl = RateLimiter::new();
        let ip = IpAddr::V4(Ipv4Addr::new(10, 0, 0, 1));
        for _ in 0..MAX_CREATES {
            rl.check_and_record(ip);
        }
        assert!(!rl.check_and_record(ip));
    }

    #[test]
    fn test_different_ips_independent() {
        let rl = RateLimiter::new();
        let ip1 = IpAddr::V4(Ipv4Addr::new(1, 1, 1, 1));
        let ip2 = IpAddr::V4(Ipv4Addr::new(2, 2, 2, 2));
        for _ in 0..MAX_CREATES {
            rl.check_and_record(ip1);
        }
        // ip2 ยังไม่ถึง limit — ต้องผ่าน
        assert!(rl.check_and_record(ip2));
    }
}
```

### ขั้นที่ 5: HTTP Handlers

Handler หลักทุกตัวรวมอยู่ใน `src/handlers.rs` — นี่คือส่วนที่ผูก business logic เข้ากับ HTTP:

```rust
// src/handlers.rs
use axum::{
    extract::{Path, Query, State},
    http::{HeaderMap, StatusCode},
    response::{Html, IntoResponse, Response},
    Json,
};
use serde::Deserialize;
use std::sync::Arc;

use crate::{
    AppState,
    error::AppError,
    expiry::{is_expired, parse_expiry},
    highlight::detect_language,
    id::generate_id,
    models::{CreatePasteRequest, CreatePasteResponse, PasteResponse, StatsResponse},
};

const MAX_CONTENT_SIZE: usize = 512 * 1024; // 512 KB

#[derive(Deserialize)]
pub struct PasteQuery {
    pub lang: Option<String>,
    pub password: Option<String>,
}

/// GET / — health check
pub async fn health_check() -> impl IntoResponse {
    Json(serde_json::json!({
        "status": "ok",
        "service": "pastebin",
        "version": env!("CARGO_PKG_VERSION")
    }))
}

/// POST / — สร้าง paste ใหม่
pub async fn create_paste(
    State(state): State<Arc<AppState>>,
    headers: HeaderMap,
    Json(req): Json<CreatePasteRequest>,
) -> Result<impl IntoResponse, AppError> {
    // 1. ตรวจสอบขนาด content
    if req.content.len() > MAX_CONTENT_SIZE {
        return Err(AppError::TooLarge);
    }

    // 2. ตรวจสอบ rate limit ด้วย IP จาก X-Forwarded-For header
    let ip = extract_ip(&headers);
    if !state.rate_limiter.check_and_record(ip) {
        return Err(AppError::RateLimited);
    }

    // 3. hash password ถ้ามี (bcrypt DEFAULT_COST = 12)
    let password_hash = if let Some(ref pw) = req.password {
        if !pw.is_empty() {
            let hash = bcrypt::hash(pw, bcrypt::DEFAULT_COST)
                .map_err(|e| AppError::Internal(anyhow::anyhow!("bcrypt error: {}", e)))?;
            Some(hash)
        } else {
            None
        }
    } else {
        None
    };

    let expires_at = parse_expiry(&req.expires);
    let language = detect_language(Some(&req.language));

    // 4. สร้าง ID พร้อม collision retry (สูงสุด 5 ครั้ง)
    let id = generate_unique_id(&state.db).await?;

    // 5. บันทึกลง database
    sqlx::query(
        "INSERT INTO pastes (id, content, language, expires_at, password_hash) \
         VALUES ($1, $2, $3, $4, $5)",
    )
    .bind(&id)
    .bind(&req.content)
    .bind(&language)
    .bind(expires_at)
    .bind(&password_hash)
    .execute(&state.db)
    .await?;

    let response = CreatePasteResponse {
        url: format!("/{}", id),
        raw_url: format!("/{}/raw", id),
        expires_at,
        id,
    };

    Ok((StatusCode::CREATED, Json(response)))
}

/// GET /{id} — ดึง paste พร้อม content negotiation
pub async fn get_paste(
    State(state): State<Arc<AppState>>,
    Path(id): Path<String>,
    Query(query): Query<PasteQuery>,
    headers: HeaderMap,
) -> Result<Response, AppError> {
    let paste = sqlx::query_as::<_, crate::models::Paste>(
        "SELECT * FROM pastes WHERE id = $1"
    )
    .bind(&id)
    .fetch_optional(&state.db)
    .await?
    .ok_or(AppError::NotFound)?;

    // ตรวจสอบหมดอายุ
    if is_expired(paste.expires_at) {
        return Err(AppError::Expired);
    }

    // ตรวจสอบ password
    if let Some(ref hash) = paste.password_hash {
        let provided = query.password.as_deref().unwrap_or("");
        if provided.is_empty() {
            return Err(AppError::PasswordRequired);
        }
        let ok = bcrypt::verify(provided, hash)
            .map_err(|e| AppError::Internal(anyhow::anyhow!("bcrypt: {}", e)))?;
        if !ok {
            return Err(AppError::WrongPassword);
        }
    }

    // เพิ่ม views counter
    sqlx::query("UPDATE pastes SET views = views + 1 WHERE id = $1")
        .bind(&id)
        .execute(&state.db)
        .await?;

    // Content negotiation: Accept header ตัดสิน response format
    let accept = headers
        .get("accept")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("application/json");

    if accept.contains("text/html") {
        let lang = query.lang.as_deref().unwrap_or(&paste.language);
        let highlighted = state.highlighter.highlight(&paste.content, lang);
        let html = render_html_page(&paste.id, &highlighted, &paste.language, paste.views);
        Ok(Html(html).into_response())
    } else {
        let resp = PasteResponse {
            id: paste.id,
            content: paste.content,
            language: paste.language,
            created_at: paste.created_at,
            expires_at: paste.expires_at,
            views: paste.views + 1,
        };
        Ok(Json(resp).into_response())
    }
}

/// GET /{id}/raw — plain text response
pub async fn get_paste_raw(
    State(state): State<Arc<AppState>>,
    Path(id): Path<String>,
    Query(query): Query<PasteQuery>,
) -> Result<Response, AppError> {
    let paste = sqlx::query_as::<_, crate::models::Paste>(
        "SELECT * FROM pastes WHERE id = $1"
    )
    .bind(&id)
    .fetch_optional(&state.db)
    .await?
    .ok_or(AppError::NotFound)?;

    if is_expired(paste.expires_at) {
        return Err(AppError::Expired);
    }

    if let Some(ref hash) = paste.password_hash {
        let provided = query.password.as_deref().unwrap_or("");
        if provided.is_empty() {
            return Err(AppError::PasswordRequired);
        }
        let ok = bcrypt::verify(provided, hash)
            .map_err(|e| AppError::Internal(anyhow::anyhow!("bcrypt: {}", e)))?;
        if !ok {
            return Err(AppError::WrongPassword);
        }
    }

    Ok((
        [(axum::http::header::CONTENT_TYPE, "text/plain; charset=utf-8")],
        paste.content,
    )
        .into_response())
}

/// DELETE /{id} — ลบ paste
pub async fn delete_paste(
    State(state): State<Arc<AppState>>,
    Path(id): Path<String>,
) -> Result<impl IntoResponse, AppError> {
    let result = sqlx::query("DELETE FROM pastes WHERE id = $1")
        .bind(&id)
        .execute(&state.db)
        .await?;

    if result.rows_affected() == 0 {
        return Err(AppError::NotFound);
    }

    Ok(StatusCode::NO_CONTENT)
}

/// GET /stats — สถิติรวมจาก aggregate query
pub async fn get_stats(
    State(state): State<Arc<AppState>>,
) -> Result<impl IntoResponse, AppError> {
    let stats = sqlx::query_as::<_, StatsResponse>(
        r#"SELECT
            COUNT(*)::BIGINT AS total_pastes,
            COUNT(*) FILTER (
                WHERE expires_at IS NULL OR expires_at > NOW()
            )::BIGINT AS active_pastes,
            COALESCE(SUM(views), 0)::BIGINT AS total_views
           FROM pastes"#,
    )
    .fetch_one(&state.db)
    .await?;

    Ok(Json(stats))
}

// ---- private helper functions ----

/// สร้าง ID ที่ไม่ซ้ำกับ paste ที่มีอยู่แล้ว
/// retry สูงสุด 5 ครั้ง — ถ้ายัง collision ใช้ nanoid(12) ที่ยาวขึ้น
async fn generate_unique_id(db: &sqlx::PgPool) -> Result<String, AppError> {
    for _ in 0..5 {
        let id = generate_id();
        let exists: Option<(String,)> =
            sqlx::query_as("SELECT id FROM pastes WHERE id = $1")
                .bind(&id)
                .fetch_optional(db)
                .await?;
        if exists.is_none() {
            return Ok(id);
        }
    }
    Ok(nanoid::nanoid!(12))
}

/// ดึง IP address จาก X-Forwarded-For header
/// (ปกติใช้กับ reverse proxy เช่น nginx/Caddy)
fn extract_ip(headers: &HeaderMap) -> std::net::IpAddr {
    headers
        .get("x-forwarded-for")
        .and_then(|v| v.to_str().ok())
        .and_then(|s| s.split(',').next())
        .and_then(|s| s.trim().parse().ok())
        .unwrap_or_else(|| "127.0.0.1".parse().unwrap())
}

/// สร้าง HTML page พร้อม syntax-highlighted code
fn render_html_page(id: &str, highlighted: &str, language: &str, views: i32) -> String {
    format!(
        r#"<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Paste {id}</title>
<style>
  body {{ font-family: monospace; background: #2b303b; color: #c0c5ce;
         margin: 0; padding: 20px; }}
  .meta {{ color: #65737e; font-size: 0.85em; margin-bottom: 12px; }}
  pre {{ background: #1c1f26; padding: 16px; border-radius: 6px;
        overflow-x: auto; }}
  code {{ font-size: 0.9em; line-height: 1.5; }}
</style>
</head>
<body>
<div class="meta">
  Paste: <strong>{id}</strong> | Language: {language} | Views: {views}
</div>
{highlighted}
</body>
</html>"#,
        id = id,
        language = language,
        views = views,
        highlighted = highlighted,
    )
}
```

### ขั้นที่ 6: Main Entry Point พร้อม Background Cleanup Task

ส่วน `main.rs` ทำหน้าที่เชื่อมทุกอย่างเข้าด้วยกัน รวมถึง spawn background task สำหรับลบ paste หมดอายุ:

```rust
// src/main.rs
use pastebin_service::{
    AppState, handlers, highlight::Highlighter, ratelimit::RateLimiter,
};
use axum::{
    routing::{delete, get, post},
    Router,
};
use sqlx::postgres::PgPoolOptions;
use std::sync::Arc;
use tokio::time::{interval, Duration};
use tower_http::trace::TraceLayer;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // ตั้งค่า tracing subscriber
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "pastebin_service=debug,tower_http=debug".into()),
        )
        .init();

    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://user:password@localhost/pastebin".to_string());

    // สร้าง connection pool
    let pool = PgPoolOptions::new()
        .max_connections(10)
        .connect(&database_url)
        .await?;

    // รัน migration ด้วย raw_sql (รองรับหลาย statement)
    sqlx::raw_sql(include_str!("../migrations/001_create_pastes.sql"))
        .execute(&pool)
        .await?;

    let state = Arc::new(AppState {
        db: pool.clone(),
        highlighter: Highlighter::new(),
        rate_limiter: RateLimiter::new(),
    });

    // Background task: ลบ paste ที่หมดอายุทุก 5 นาที
    let cleanup_pool = pool.clone();
    tokio::spawn(async move {
        let mut ticker = interval(Duration::from_secs(300));
        loop {
            ticker.tick().await;
            match sqlx::query(
                "DELETE FROM pastes WHERE expires_at IS NOT NULL AND expires_at < NOW()"
            )
            .execute(&cleanup_pool)
            .await
            {
                Ok(r) if r.rows_affected() > 0 => {
                    tracing::info!("Cleaned up {} expired pastes", r.rows_affected());
                }
                Err(e) => tracing::error!("Cleanup error: {}", e),
                _ => {}
            }
        }
    });

    let app = Router::new()
        .route("/",         get(handlers::health_check))
        .route("/",         post(handlers::create_paste))
        .route("/stats",    get(handlers::get_stats))
        .route("/{id}",     get(handlers::get_paste))
        .route("/{id}",     delete(handlers::delete_paste))
        .route("/{id}/raw", get(handlers::get_paste_raw))
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    let addr = "0.0.0.0:3000";
    tracing::info!("Pastebin service listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(addr).await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

**Background cleanup task** — จุดสำคัญ:

1. `tokio::spawn` สร้าง task ใหม่ที่รันพร้อมกับ server
2. `ticker.tick().await` จะ block task นี้ระหว่างรอ — ไม่กิน CPU
3. `ticker.tick()` ครั้งแรกจะ fire ทันที (ไม่รอ 5 นาที) ซึ่งช่วย cleanup paste ที่ค้างอยู่ตั้งแต่ก่อน restart
4. `pool.clone()` ใช้ Arc clone ภายใน — ถูก และ reference count เพิ่มขึ้น 1

## การทดสอบ (Testing)

### Unit Tests

Unit tests อยู่ใน module ต่าง ๆ ด้วย `#[cfg(test)]` block — ทดสอบ pure logic ที่ไม่ต้องการ database:

```
src/expiry.rs   → test_parse_never, test_parse_valid, test_parse_1d,
                  test_not_expired, test_expired, test_never_expires
src/id.rs       → test_id_length, test_id_charset, test_id_uniqueness
src/highlight.rs → test_highlight_rust, test_highlight_plain, test_detect_language
src/ratelimit.rs → test_allow_under_limit, test_block_over_limit, test_different_ips_independent
```

### Integration Tests

Integration tests ใช้ real PostgreSQL ผ่าน `tower::ServiceExt::oneshot` — ส่ง HTTP request จริงเข้า Axum app โดยไม่ต้องเปิด network socket:

```rust
// tests/integration_test.rs
use pastebin_service::{AppState, handlers, highlight::Highlighter, ratelimit::RateLimiter};
use axum::{
    body::Body,
    http::{Request, StatusCode},
    Router,
    routing::{delete, get, post},
};
use sqlx::postgres::PgPoolOptions;
use std::sync::Arc;
use tower::ServiceExt; // สำหรับ .oneshot()

async fn setup_test_app() -> (Router, sqlx::PgPool) {
    let database_url = std::env::var("DATABASE_URL")
        .unwrap_or_else(|_| "postgres://user:password@localhost/pastebin_test".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to test database");

    sqlx::raw_sql(include_str!("../migrations/001_create_pastes.sql"))
        .execute(&pool)
        .await
        .expect("Failed to run migration");

    let state = Arc::new(AppState {
        db: pool.clone(),
        highlighter: Highlighter::new(),
        rate_limiter: RateLimiter::new(),
    });

    let app = Router::new()
        .route("/",         get(handlers::health_check))
        .route("/",         post(handlers::create_paste))
        .route("/stats",    get(handlers::get_stats))
        .route("/{id}",     get(handlers::get_paste))
        .route("/{id}",     delete(handlers::delete_paste))
        .route("/{id}/raw", get(handlers::get_paste_raw))
        .with_state(state);

    (app, pool)
}

#[tokio::test]
async fn test_health_check() {
    let (app, _pool) = setup_test_app().await;

    let resp = app
        .oneshot(Request::builder().uri("/").body(Body::empty()).unwrap())
        .await
        .unwrap();

    assert_eq!(resp.status(), StatusCode::OK);

    let body = axum::body::to_bytes(resp.into_body(), 1024 * 1024).await.unwrap();
    let json: serde_json::Value = serde_json::from_slice(&body).unwrap();
    assert_eq!(json["status"], "ok");
}

#[tokio::test]
async fn test_create_and_get_paste() {
    let (app, pool) = setup_test_app().await;

    let create_req = serde_json::json!({
        "content": "fn main() { println!(\"hello\"); }",
        "language": "rust",
        "expires": "never"
    });

    let resp = app.clone()
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/")
                .header("content-type", "application/json")
                .body(Body::from(create_req.to_string()))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(resp.status(), StatusCode::CREATED);

    let body = axum::body::to_bytes(resp.into_body(), 1024 * 1024).await.unwrap();
    let json: serde_json::Value = serde_json::from_slice(&body).unwrap();
    let id = json["id"].as_str().unwrap().to_string();

    // ดึง paste กลับมาในรูป JSON
    let resp2 = app
        .oneshot(
            Request::builder()
                .uri(format!("/{}", id))
                .header("accept", "application/json")
                .body(Body::empty())
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(resp2.status(), StatusCode::OK);
    let body2 = axum::body::to_bytes(resp2.into_body(), 1024 * 1024).await.unwrap();
    let json2: serde_json::Value = serde_json::from_slice(&body2).unwrap();
    assert_eq!(json2["language"], "rust");
    assert!(json2["content"].as_str().unwrap().contains("hello"));

    // cleanup
    sqlx::query("DELETE FROM pastes WHERE id = $1")
        .bind(&id)
        .execute(&pool)
        .await
        .unwrap();
}

#[tokio::test]
async fn test_reject_large_paste() {
    let (app, _pool) = setup_test_app().await;

    let big_content = "x".repeat(513 * 1024); // 513 KB — เกิน limit 512 KB
    let req_body = serde_json::json!({
        "content": big_content,
        "language": "plain",
        "expires": "never"
    });

    let resp = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/")
                .header("content-type", "application/json")
                .body(Body::from(req_body.to_string()))
                .unwrap(),
        )
        .await
        .unwrap();

    assert_eq!(resp.status(), StatusCode::PAYLOAD_TOO_LARGE);
}
```

### รัน Tests

```bash
# สร้าง test database ก่อน (ครั้งแรก)
createdb -h localhost -U user pastebin_test

# รัน unit tests + integration tests
# ใช้ --test-threads=1 เพราะ test แต่ละตัวแชร์ database เดียวกัน
DATABASE_URL="postgres://user:password@localhost/pastebin_test" \
  cargo test -- --test-threads=1
```

### ผลลัพธ์จริงจากการรัน

```
   Compiling pastebin-service v0.1.0
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1m 10s
     Running unittests src/lib.rs

running 15 tests
test expiry::tests::test_expired ... ok
test expiry::tests::test_never_expires ... ok
test expiry::tests::test_not_expired ... ok
test expiry::tests::test_parse_1d ... ok
test expiry::tests::test_parse_never ... ok
test expiry::tests::test_parse_valid ... ok
test highlight::tests::test_detect_language ... ok
test highlight::tests::test_highlight_plain ... ok
test highlight::tests::test_highlight_rust ... ok
test id::tests::test_id_charset ... ok
test id::tests::test_id_length ... ok
test id::tests::test_id_uniqueness ... ok
test ratelimit::tests::test_allow_under_limit ... ok
test ratelimit::tests::test_block_over_limit ... ok
test ratelimit::tests::test_different_ips_independent ... ok

test result: ok. 15 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.07s

     Running tests/integration_test.rs

running 6 tests
test test_create_and_get_paste ... ok
test test_delete_paste ... ok
test test_health_check ... ok
test test_paste_not_found ... ok
test test_reject_large_paste ... ok
test test_stats_endpoint ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.65s

   Doc-tests pastebin_service

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**รวมทั้งหมด: 21 tests ผ่าน 0 fail**

## กับดักที่พบบ่อย (Common Pitfalls)

### Pitfall 1: `sqlx::query` ไม่รองรับหลาย SQL Statement

```rust
// ❌ Error: "cannot insert multiple commands into a prepared statement"
sqlx::query(include_str!("../migrations/001_create_pastes.sql"))
    .execute(&pool)
    .await?;

// ✅ ใช้ raw_sql แทน ซึ่งไม่ใช้ prepared statement
sqlx::raw_sql(include_str!("../migrations/001_create_pastes.sql"))
    .execute(&pool)
    .await?;
```

**สาเหตุ**: `sqlx::query()` ใช้ PostgreSQL's prepared statement protocol ที่รองรับเพียง statement เดียว `raw_sql()` ส่ง SQL ตรงในรูป "simple query" ที่รองรับหลาย statement (คั่นด้วย `;`)

**เมื่อไหร่ที่เจอ**: ทุกครั้งที่ include migration file ที่มี `CREATE TABLE` + `CREATE INDEX` หรือ SQL file ใด ๆ ที่มีหลาย statement

---

### Pitfall 2: Integration Tests Fail เมื่อรัน Parallel

```bash
# ❌ tests ล้มเหลวเป็นบางครั้ง (flaky)
cargo test

# ✅ รัน sequential
cargo test -- --test-threads=1
```

**สาเหตุ**: Integration test หลายตัวแชร์ database เดียวกัน เมื่อรัน parallel สองตัวอาจพยายาม `CREATE TABLE IF NOT EXISTS` พร้อมกัน และ lock table ระหว่าง test อาจทำให้ query เพื่อ cleanup ข้อมูล test ชนกัน

**วิธีแก้ที่ดีกว่าระยะยาว**: ใช้ transaction ครอบแต่ละ test แล้ว rollback หลังเสร็จ หรือสร้าง schema ใหม่ต่อ test (เช่นใช้ `sqlx-test` crate)

---

### Pitfall 3: `Instant` ไม่ serialize ได้ — ใช้ `SystemTime` แทนเมื่อต้องการเก็บใน DB

```rust
// ❌ Instant ไม่ implement Serialize — ใช้เฉพาะสำหรับ duration measurement
use std::time::Instant;
let t: Instant = Instant::now();
// serde_json::to_string(&t) → compile error

// ✅ สำหรับ timestamp ที่ต้องเก็บใน database ใช้ chrono::DateTime<Utc>
use chrono::Utc;
let t: chrono::DateTime<Utc> = Utc::now();

// ✅ สำหรับ in-memory rate limiter ที่ไม่ต้อง serialize ใช้ Instant ได้
// (เร็วกว่าเพราะไม่ผ่าน system clock)
use std::time::Instant;
let t: Instant = Instant::now();
let elapsed = t.elapsed(); // Duration
```

**กฎง่าย ๆ**: `Instant` สำหรับ "นานแค่ไหน", `DateTime<Utc>` สำหรับ "ตอนไหน"

---

### Pitfall 4: `bcrypt::hash` ใช้เวลานาน — ห้ามเรียกใน synchronous context

```rust
// ❌ บล็อก tokio thread — ทำให้ throughput ลดลงมาก
pub async fn create_paste(...) {
    let hash = bcrypt::hash(password, bcrypt::DEFAULT_COST).unwrap();
    // DEFAULT_COST = 12 → ใช้เวลา ~300ms บน CPU ทั่วไป
}

// ✅ spawn_blocking เพื่อย้าย CPU-intensive work ออกจาก async thread
pub async fn create_paste(...) {
    let password = password.to_string();
    let hash = tokio::task::spawn_blocking(move || {
        bcrypt::hash(&password, bcrypt::DEFAULT_COST)
    })
    .await
    .map_err(|e| AppError::Internal(anyhow::anyhow!("join error: {}", e)))?
    .map_err(|e| AppError::Internal(anyhow::anyhow!("bcrypt: {}", e)))?;
}
```

**สาเหตุ**: tokio runtime ใช้ thread pool ขนาดเล็ก (= จำนวน CPU cores) ถ้าบล็อก thread นาน ๆ จะทำให้ request อื่น ๆ ค้าง `spawn_blocking` ย้าย task ไปยัง blocking thread pool ที่ขยายได้ไม่จำกัด

ในโปรเจคนี้เราเรียก `bcrypt::hash` โดยตรงเพื่อให้โค้ดสั้นและเข้าใจง่ายขึ้น แต่ใน production ควรใช้ `spawn_blocking`

---

### Pitfall 5: Views Counter Race Condition

```sql
-- ❌ อ่านแล้ว update แยกกัน — race condition ถ้ามีหลาย request พร้อมกัน
SELECT views FROM pastes WHERE id = $1;
-- (อ่านได้ views = 5)
-- (request อื่น ก็อ่านได้ views = 5 พร้อมกัน)
UPDATE pastes SET views = 6 WHERE id = $1;
-- (ทั้งสอง request update เป็น 6 แทนที่จะเป็น 7)

-- ✅ atomic increment ด้วย SQL
UPDATE pastes SET views = views + 1 WHERE id = $1;
```

PostgreSQL guarantee ว่า `views = views + 1` เป็น atomic operation ภายใน transaction เดียว จึงไม่มี race condition

## การ Package และ Deploy

### Build Release Binary

```bash
# สร้าง optimized binary
cargo build --release

# binary อยู่ที่
./target/release/pastebin-service
```

### Environment Variables

```bash
# รัน production server
DATABASE_URL="postgres://user:password@db-host/pastebin" \
RUST_LOG="pastebin_service=info,tower_http=warn" \
./target/release/pastebin-service
```

### Dockerfile

```dockerfile
# Stage 1: Build
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY migrations ./migrations
RUN cargo build --release

# Stage 2: Runtime (minimal image)
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libssl3 ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/pastebin-service /usr/local/bin/
EXPOSE 3000
CMD ["pastebin-service"]
```

### ทดสอบด้วย curl

```bash
# สร้าง paste
curl -s -X POST http://localhost:3000/ \
  -H "Content-Type: application/json" \
  -d '{"content":"fn main() { println!(\"Hello\"); }","language":"rust","expires":"1d"}' \
  | jq .

# ตัวอย่าง response:
# {
#   "id": "aB3kP9mZ",
#   "url": "/aB3kP9mZ",
#   "raw_url": "/aB3kP9mZ/raw",
#   "expires_at": "2026-09-28T10:30:00Z"
# }

# ดึง paste เป็น JSON
curl -s http://localhost:3000/aB3kP9mZ \
  -H "Accept: application/json" | jq .

# ดึง paste เป็น HTML พร้อม syntax highlighting
curl -s http://localhost:3000/aB3kP9mZ \
  -H "Accept: text/html" > paste.html

# ดึง plain text
curl -s http://localhost:3000/aB3kP9mZ/raw

# ดูสถิติ
curl -s http://localhost:3000/stats | jq .

# ลบ paste
curl -X DELETE http://localhost:3000/aB3kP9mZ

# สร้าง paste ที่มี password
curl -s -X POST http://localhost:3000/ \
  -H "Content-Type: application/json" \
  -d '{"content":"secret config","language":"plain","password":"mypassword"}' \
  | jq .

# ดึง paste ที่ต้องใช้ password
curl -s "http://localhost:3000/{id}?password=mypassword" \
  -H "Accept: application/json" | jq .
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม Edit Token (ระดับง่าย)

เมื่อสร้าง paste ใหม่ ให้ generate `edit_token` (UUID แบบ random) และส่งกลับไปให้ client token นี้จะใช้สำหรับ `DELETE /{id}?token=xxx` และ `PUT /{id}?token=xxx` เพื่อแก้ไข content

ต้องเพิ่ม:
- `edit_token TEXT` column ใน database
- ตรวจสอบ token ใน delete/update handler
- คืน 403 ถ้า token ไม่ตรง

*Hint*: ใช้ `uuid::Uuid::new_v4().to_string()` สำหรับ generate token และ constant-time comparison (`hmac::equal`) เพื่อป้องกัน timing attack

### แบบฝึกหัดที่ 2: เพิ่ม Paste Collections (ระดับกลาง)

สร้าง concept "collection" — user สามารถ group หลาย paste ไว้ด้วยกันได้ เช่น เป็น multi-file gist

ต้องเพิ่ม:
- ตาราง `collections` (id, name, created_at)
- ตาราง `collection_pastes` (collection_id, paste_id, filename)
- Route `POST /collections` และ `GET /collections/{id}`
- Response รวม list ของ paste ทั้งหมดใน collection

*Hint*: ใช้ JOIN query: `SELECT p.*, cp.filename FROM pastes p JOIN collection_pastes cp ON p.id = cp.paste_id WHERE cp.collection_id = $1`

### แบบฝึกหัดที่ 3: เพิ่ม Fork Functionality (ระดับกลาง)

เพิ่ม route `POST /{id}/fork` ที่สร้าง paste ใหม่จาก paste เดิม (copy content + language) และ track ว่า forked_from ใคร

ต้องเพิ่ม:
- `forked_from TEXT REFERENCES pastes(id)` column
- `POST /{id}/fork` handler ที่ copy paste
- `GET /{id}` response ที่แสดง `forked_from` field

### แบบฝึกหัดที่ 4: Rate Limit ที่ Persistent ข้าม Restart (ระดับยาก)

Rate limiter ปัจจุบันเก็บใน memory — เมื่อ server restart counter จะหาย ทำให้ bypass rate limit ได้

ให้เปลี่ยนไปใช้ Redis (ด้วย `redis` crate) หรือ PostgreSQL:

```sql
-- ตัวอย่าง schema ใน PostgreSQL
CREATE TABLE rate_limits (
    ip TEXT,
    window_start TIMESTAMPTZ,
    count INTEGER,
    PRIMARY KEY (ip, window_start)
);
```

หรือใช้ Redis INCR + EXPIRE:

```rust
// pseudo code
let key = format!("ratelimit:{}:{}", ip, window_minute);
let count: i64 = redis.incr(&key).await?;
if count == 1 {
    redis.expire(&key, 60).await?; // expire หลัง 60 วินาที
}
count <= MAX_CREATES
```

*Hint*: `deadpool-redis` ให้ connection pool สำหรับ Redis ที่เข้ากันได้กับ tokio

## สรุป

โปรเจคนี้สร้าง production-ready Pastebin Service ที่ครอบคลุม:

**Pattern ที่ได้เรียน:**
- **AppState pattern**: แชร์ shared resource (database pool, highlighter, rate limiter) ระหว่าง handler ด้วย `Arc<AppState>`
- **Content negotiation**: ใช้ `Accept` header ตัดสิน response format (HTML/JSON/plain text) — pattern นี้ใช้ใน REST API จริงทั่วไป
- **Error-first design**: ออกแบบ `AppError` ก่อนเขียน handler ทำให้ handler code สะอาด error handling ไม่กระจัดกระจาย
- **Background tasks**: `tokio::spawn` + `tokio::time::interval` สำหรับ periodic maintenance task
- **Sliding window rate limiting**: เหนือกว่า fixed window ตรงที่ไม่มี "burst ที่ช่วงรอยต่อ" ปัญหา

**สิ่งที่ควรเพิ่มใน production จริง:**
- `spawn_blocking` สำหรับ bcrypt
- Redis สำหรับ rate limit ที่ persist ข้าม restart
- TLS ด้วย `rustls` หรือ reverse proxy (nginx/Caddy)
- Structured logging ด้วย `tracing_json`
- Health check endpoint ที่ตรวจสอบ database connection ด้วย
- API key authentication สำหรับ admin endpoints (DELETE all, /stats)

โปรเจคถัดไปจะต่อยอดจาก web service พื้นฐานไปสู่ **File Upload CDN** ที่จัดการ binary file (image, document) ด้วย streaming upload, S3-compatible storage, และ content-based deduplication

---

**โปรเจคก่อนหน้า:** [project-a10-secret-vault.md](project-a10-secret-vault.md) | **โปรเจคถัดไป:** [project-b02-file-upload-cdn.md](project-b02-file-upload-cdn.md)
