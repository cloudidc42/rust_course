# Project B04: RSS/Atom Feed Aggregator Service

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

## ภาพรวมโปรเจค

ในโปรเจคนี้เราจะสร้าง **RSS/Atom Feed Aggregator Service** ซึ่งเป็น backend service ที่รับ URL ของ feeds แล้ว ดึงบทความมาจัดเก็บไว้ใน PostgreSQL พร้อม API สำหรับอ่าน, ค้นหา, และจัดการข้อมูล โปรแกรมจะ poll feeds ในพื้นหลังอัตโนมัติ, กรองข้อมูลซ้ำด้วย deduplication, และ notify ไปยัง webhook เมื่อมีบทความใหม่ที่ตรงกับ keyword filter

Feed aggregator เป็น archetype ที่พบบ่อยในโลก real-world มาก ไม่ว่าจะเป็น Feedly, Newsblur, Miniflux, FreshRSS ล้วนเป็นโปรแกรมลักษณะนี้ทั้งสิ้น นอกจากนี้ pattern เดียวกันยังใช้กับ "event ingestion pipeline" ทั่วไป เช่น ดึง data จาก external sources มาเก็บไว้ใน DB แล้วส่ง notification

**Use cases จริงในโลก production:**
- Personal feed reader ที่ self-host ได้ ไม่ต้องพึ่ง service ภายนอก
- Content monitoring pipeline ที่ตามดูคู่แข่งหรือ industry news
- Alert system ที่ส่ง webhook ไปยัง Slack/Discord เมื่อมีบทความตรงตามเงื่อนไข
- Backend สำหรับ team feed aggregator ที่ทุกคนในทีมสามารถ subscribe feeds ร่วมกัน

## สิ่งที่จะได้เรียนรู้

- **Feed parsing** — ใช้ `feed-rs` crate ซึ่งรองรับ RSS 0.9/1.0/2.0, Atom 0.3/1.0, JSON Feed โดยไม่ต้องเขียน parser เอง
- **Background polling loop** — `tokio::time::interval` สำหรับ periodic task ที่รันคู่กับ HTTP server
- **Deduplication pattern** — ใช้ unique constraint บน `(feed_id, guid)` ใน PostgreSQL ป้องกัน duplicate insert
- **PostgreSQL full-text search** — `tsvector` + `GIN` index + `to_tsquery` สำหรับ keyword search ใน content
- **OPML XML parsing/generation** — ใช้ `quick-xml` อ่าน/เขียนไฟล์ OPML มาตรฐาน
- **Axum shared state** — ส่ง database pool และ config ผ่าน `Arc<AppState>` ไปยังทุก handler
- **HTTP Cache-Control parsing** — อ่าน `Cache-Control: max-age=N` และ `TTL` จาก HTTP response เพื่อ throttle polling
- **Webhook fanout** — ส่ง HTTP POST ไปยัง registered URLs เมื่อมีบทความใหม่ที่ตรงกับ keyword

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–40**: Rust basics — ownership, borrowing, structs, enums, `Result`, traits, generics
- **Part 41–50**: Async/await, `tokio` runtime, `Future`, `spawn`
- **Part 51–60**: `axum` web framework, routing, extractors, middleware
- **Part 61–70**: `sqlx` กับ PostgreSQL, migrations, connection pool
- **Part 71–80**: `serde` / `serde_json`, JSON serialization/deserialization
- **Part 81–90**: HTTP client ด้วย `reqwest`, headers, status codes
- **Part 91–95**: XML parsing ด้วย `quick-xml`, iterator pattern บน events

## โครงสร้างโปรเจค (Project Layout)

```
rss-aggregator/
├── src/
│   ├── main.rs            ← entry point: setup AppState, start server + background poller
│   ├── config.rs          ← Config struct อ่านจาก env vars
│   ├── db/
│   │   ├── mod.rs         ← re-exports
│   │   ├── feeds.rs       ← database queries เกี่ยวกับ feeds table
│   │   └── items.rs       ← database queries เกี่ยวกับ items table
│   ├── models.rs          ← shared data types (Feed, Item, Webhook, etc.)
│   ├── routes/
│   │   ├── mod.rs         ← router assembly
│   │   ├── feeds.rs       ← /feeds endpoint handlers
│   │   ├── items.rs       ← /items endpoint handlers
│   │   ├── opml.rs        ← /opml import/export handlers
│   │   └── webhooks.rs    ← /webhooks endpoint handlers
│   ├── fetcher.rs         ← background feed fetching loop
│   ├── parser.rs          ← wraps feed-rs, normalises entries to Item
│   └── opml.rs            ← OPML XML serialiser/deserialiser
├── migrations/
│   ├── 001_create_feeds.sql
│   ├── 002_create_items.sql
│   ├── 003_add_fts_index.sql
│   └── 004_create_webhooks.sql
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
                 POST /feeds (URL)
                       │
                       ▼
              ┌─────────────────┐
              │  Axum HTTP API  │
              └────────┬────────┘
                       │ INSERT feed
                       ▼
              ┌─────────────────┐
              │   PostgreSQL    │◄──── Background Fetcher (tokio interval)
              │  feeds + items  │      │  1. SELECT feeds WHERE next_fetch <= NOW()
              └─────────────────┘      │  2. HTTP GET feed URL (reqwest)
                       │               │  3. Parse (feed-rs)
           ┌───────────┴───────────┐   │  4. INSERT OR IGNORE items (dedup)
           │  GET /items?q=keyword  │  │  5. Fire webhooks if new items match filter
           │  (PostgreSQL FTS)      │  └──────────────────────────────────────────
           └────────────────────────┘
```

### ทำไมถึงใช้ `feed-rs` แทนการเขียน parser เอง?

RSS มีหลาย version ที่มี quirks ต่างกัน RSS 0.9 ใช้ namespace ต่างจาก RSS 2.0, Atom ใช้ `<entry>` แทน `<item>`, JSON Feed ใช้ JSON ทั้งหมด การ handle corner cases ทั้งหมดใช้เวลามาก `feed-rs` normalize ทุก format ออกมาเป็น struct เดียวกัน (`Feed` + `Entry`) ทำให้โค้ดของเราง่ายขึ้นมาก

### การออกแบบ Database Schema

```sql
-- feeds table: เก็บ URL และ metadata ของแต่ละ feed
CREATE TABLE feeds (
    id          BIGSERIAL PRIMARY KEY,
    url         TEXT NOT NULL UNIQUE,
    title       TEXT,
    description TEXT,
    fetch_interval_secs INTEGER NOT NULL DEFAULT 900,  -- default 15 นาที
    last_fetched_at     TIMESTAMPTZ,
    next_fetch_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    etag                TEXT,         -- สำหรับ HTTP conditional request
    last_modified       TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- items table: เก็บบทความที่ดึงมาจาก feeds
CREATE TABLE items (
    id           BIGSERIAL PRIMARY KEY,
    feed_id      BIGINT NOT NULL REFERENCES feeds(id) ON DELETE CASCADE,
    guid         TEXT NOT NULL,       -- item guid หรือ URL ใช้ deduplicate
    title        TEXT,
    url          TEXT,
    content      TEXT,
    published_at TIMESTAMPTZ,
    fetched_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    read_at      TIMESTAMPTZ,         -- NULL = ยังไม่ได้อ่าน
    search_vec   TSVECTOR,            -- full-text search vector
    UNIQUE (feed_id, guid)            -- deduplication constraint
);

-- GIN index สำหรับ full-text search
CREATE INDEX items_search_idx ON items USING GIN (search_vec);

-- webhooks table
CREATE TABLE webhooks (
    id         BIGSERIAL PRIMARY KEY,
    url        TEXT NOT NULL,
    keyword    TEXT NOT NULL,         -- filter keyword (empty = ทุก item)
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

ประเด็นสำคัญในการออกแบบ:
- `UNIQUE (feed_id, guid)` คือหัวใจของ deduplication — ถ้า INSERT ซ้ำ PostgreSQL จะ error ซึ่งเราจัดการด้วย `ON CONFLICT DO NOTHING`
- `TSVECTOR` column `search_vec` ถูก populate โดย trigger หรือโดย application code ตอน INSERT
- `next_fetch_at` ช่วยให้ background fetcher ไม่ต้องดึง feed ทุกตัวในรอบเดียว — เลือกเฉพาะที่ถึงเวลาแล้ว

### Shared State Pattern ใน Axum

```rust
// src/main.rs
use std::sync::Arc;
use sqlx::PgPool;

#[derive(Clone)]
pub struct AppState {
    pub db: PgPool,
    pub config: Arc<Config>,
    pub http: reqwest::Client,
}
```

เราส่ง `Arc<AppState>` เป็น Axum extension state ซึ่ง clone ได้ถูก (เพราะ `PgPool` และ `reqwest::Client` ถูก clone ได้อยู่แล้ว เนื่องจากภายในเป็น `Arc` อยู่แล้ว) Handler แต่ละตัวรับ `State(state): State<Arc<AppState>>` แล้วเข้าถึง pool, config, และ HTTP client ได้ทันที

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างพื้นฐาน — Cargo.toml, Models, Config

เริ่มจากการกำหนด dependencies และ data structures ที่จะใช้ตลอดทั้งโปรเจค

**Cargo.toml:**

```toml
[package]
name = "rss-aggregator"
version = "0.1.0"
edition = "2021"

[dependencies]
# Web framework
axum = { version = "0.8", features = ["multipart"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["trace", "cors"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Database
sqlx = { version = "0.8", features = ["runtime-tokio-native-tls", "postgres", "chrono", "uuid"] }

# Feed parsing
feed-rs = "2"

# HTTP client
reqwest = { version = "0.12", features = ["json", "gzip"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# XML (OPML)
quick-xml = { version = "0.37", features = ["serialize"] }

# Utilities
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1", features = ["v4", "serde"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
thiserror = "2"
anyhow = "1"
dotenvy = "0.15"
```

**src/config.rs:**

```rust
use std::env;

#[derive(Debug, Clone)]
pub struct Config {
    pub database_url: String,
    pub server_host: String,
    pub server_port: u16,
    /// Default interval ถ้า feed ไม่ได้ระบุ TTL (วินาที)
    pub default_fetch_interval: u64,
}

impl Config {
    pub fn from_env() -> anyhow::Result<Self> {
        dotenvy::dotenv().ok();
        Ok(Config {
            database_url: env::var("DATABASE_URL")
                .unwrap_or_else(|_| "postgres://localhost/rss_aggregator".into()),
            server_host: env::var("HOST").unwrap_or_else(|_| "0.0.0.0".into()),
            server_port: env::var("PORT")
                .ok()
                .and_then(|p| p.parse().ok())
                .unwrap_or(3000),
            default_fetch_interval: env::var("DEFAULT_FETCH_INTERVAL_SECS")
                .ok()
                .and_then(|s| s.parse().ok())
                .unwrap_or(900), // 15 นาที
        })
    }
}
```

**src/models.rs:**

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// Feed ที่ผู้ใช้ subscribe ไว้
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Feed {
    pub id: i64,
    pub url: String,
    pub title: Option<String>,
    pub description: Option<String>,
    pub fetch_interval_secs: i32,
    pub last_fetched_at: Option<DateTime<Utc>>,
    pub next_fetch_at: DateTime<Utc>,
    pub etag: Option<String>,
    pub last_modified: Option<String>,
    pub created_at: DateTime<Utc>,
}

/// Item (บทความ) ที่ดึงมาจาก feed
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Item {
    pub id: i64,
    pub feed_id: i64,
    pub guid: String,
    pub title: Option<String>,
    pub url: Option<String>,
    pub content: Option<String>,
    pub published_at: Option<DateTime<Utc>>,
    pub fetched_at: DateTime<Utc>,
    pub read_at: Option<DateTime<Utc>>,
}

/// Request body สำหรับ POST /feeds
#[derive(Debug, Deserialize)]
pub struct CreateFeedRequest {
    pub url: String,
    /// Override interval เป็น seconds (ถ้าไม่ระบุ ใช้ default จาก config)
    pub fetch_interval_secs: Option<i32>,
}

/// Query parameters สำหรับ GET /items
#[derive(Debug, Deserialize)]
pub struct ItemsQuery {
    pub feed_id: Option<i64>,
    pub unread: Option<bool>,
    pub q: Option<String>,           // full-text search keyword
    pub since: Option<DateTime<Utc>>,
    pub before: Option<DateTime<Utc>>,
    #[serde(default = "default_limit")]
    pub limit: i64,
    #[serde(default)]
    pub offset: i64,
}

fn default_limit() -> i64 { 20 }

/// Webhook ที่ผู้ใช้ register ไว้
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Webhook {
    pub id: i64,
    pub url: String,
    pub keyword: String,
    pub created_at: DateTime<Utc>,
}

/// Request body สำหรับ POST /webhooks
#[derive(Debug, Deserialize)]
pub struct CreateWebhookRequest {
    pub url: String,
    #[serde(default)]
    pub keyword: String,
}

/// Payload ที่ส่งไปยัง webhook URL
#[derive(Debug, Serialize)]
pub struct WebhookPayload<'a> {
    pub event: &'static str,
    pub item: &'a Item,
    pub feed: &'a Feed,
}

/// Error type ของ application
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),
    #[error("Feed parse error: {0}")]
    FeedParse(#[from] feed_rs::parser::ParseFeedError),
    #[error("Not found")]
    NotFound,
    #[error("Bad request: {0}")]
    BadRequest(String),
}

// impl IntoResponse เพื่อให้ Axum แปลง AppError เป็น HTTP response ได้อัตโนมัติ
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "Not found".to_string()),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            _ => (StatusCode::INTERNAL_SERVER_ERROR, self.to_string()),
        };
        (status, Json(serde_json::json!({ "error": message }))).into_response()
    }
}
```

### ขั้นที่ 2: Feed Parsing ด้วย `feed-rs`

`feed-rs` เป็น crate ที่ parse RSS และ Atom ได้หลาย version โดย normalize ผลลัพธ์ออกมาเป็น `feed_rs::model::Feed` ซึ่งมี `entries: Vec<Entry>` แต่ละ entry มี field ที่เราสนใจ ได้แก่ `id` (guid), `title`, `links`, `content`, `published`

**src/parser.rs:**

```rust
use chrono::{DateTime, Utc};
use feed_rs::{model::Entry, parser};

/// แปลง raw feed bytes เป็น Vec ของ normalized items ที่พร้อม insert
pub struct ParsedItem {
    pub guid: String,
    pub title: Option<String>,
    pub url: Option<String>,
    pub content: Option<String>,
    pub published_at: Option<DateTime<Utc>>,
}

/// Parse feed content (ไม่ว่าจะเป็น RSS หรือ Atom) และ return items
///
/// # Errors
/// คืนค่า `feed_rs::parser::ParseFeedError` ถ้า content ไม่ใช่ feed ที่ valid
pub fn parse_feed(bytes: &[u8]) -> Result<(Option<String>, Vec<ParsedItem>), feed_rs::parser::ParseFeedError> {
    let feed = parser::parse(bytes)?;

    let feed_title = feed.title.map(|t| t.content);

    let items = feed
        .entries
        .into_iter()
        .map(normalise_entry)
        .collect();

    Ok((feed_title, items))
}

fn normalise_entry(entry: Entry) -> ParsedItem {
    // guid: ใช้ entry.id ก่อน ถ้าไม่มีใช้ link แรก
    let guid = if !entry.id.is_empty() {
        entry.id.clone()
    } else {
        entry
            .links
            .first()
            .map(|l| l.href.clone())
            .unwrap_or_else(|| entry.id.clone())
    };

    // title
    let title = entry.title.map(|t| t.content);

    // url: ลิงก์แรกใน entry.links
    let url = entry.links.into_iter().next().map(|l| l.href);

    // content: ลอง summary ก่อน แล้วค่อยดู content
    let content = entry
        .summary
        .map(|s| s.content)
        .or_else(|| entry.content.and_then(|c| c.body));

    // published
    let published_at = entry.published.map(|dt| dt.into());

    ParsedItem { guid, title, url, content, published_at }
}
```

สังเกตว่าเราแยก `parser.rs` ไว้เฉพาะ เพราะ logic การ normalize `feed-rs` entry ค่อนข้างมีรายละเอียด และการแยกออกทำให้ test ได้ง่าย ไม่ต้องพึ่ง database หรือ HTTP

### ขั้นที่ 3: Database Layer — CRUD สำหรับ Feeds และ Items

**src/db/feeds.rs:**

```rust
use crate::models::{CreateFeedRequest, Feed};
use sqlx::PgPool;
use chrono::Utc;

/// สร้าง feed ใหม่ใน database
pub async fn create_feed(
    db: &PgPool,
    req: &CreateFeedRequest,
    default_interval: i32,
) -> Result<Feed, sqlx::Error> {
    let interval = req.fetch_interval_secs.unwrap_or(default_interval);
    sqlx::query_as!(
        Feed,
        r#"
        INSERT INTO feeds (url, fetch_interval_secs, next_fetch_at)
        VALUES ($1, $2, NOW())
        RETURNING *
        "#,
        req.url,
        interval,
    )
    .fetch_one(db)
    .await
}

/// ดึง feeds ทั้งหมด
pub async fn list_feeds(db: &PgPool) -> Result<Vec<Feed>, sqlx::Error> {
    sqlx::query_as!(Feed, "SELECT * FROM feeds ORDER BY created_at DESC")
        .fetch_all(db)
        .await
}

/// ดึง feed เดี่ยวตาม id
pub async fn get_feed(db: &PgPool, id: i64) -> Result<Option<Feed>, sqlx::Error> {
    sqlx::query_as!(Feed, "SELECT * FROM feeds WHERE id = $1", id)
        .fetch_optional(db)
        .await
}

/// ลบ feed และ cascade ลบ items ด้วย (ON DELETE CASCADE)
pub async fn delete_feed(db: &PgPool, id: i64) -> Result<bool, sqlx::Error> {
    let result = sqlx::query!("DELETE FROM feeds WHERE id = $1", id)
        .execute(db)
        .await?;
    Ok(result.rows_affected() > 0)
}

/// ดึง feeds ที่ถึงเวลา fetch แล้ว (next_fetch_at <= NOW())
pub async fn feeds_due_for_fetch(db: &PgPool) -> Result<Vec<Feed>, sqlx::Error> {
    sqlx::query_as!(
        Feed,
        "SELECT * FROM feeds WHERE next_fetch_at <= NOW() ORDER BY next_fetch_at ASC"
    )
    .fetch_all(db)
    .await
}

/// อัปเดต next_fetch_at, last_fetched_at, etag หลังดึง feed เสร็จ
pub async fn update_feed_after_fetch(
    db: &PgPool,
    id: i64,
    etag: Option<&str>,
    last_modified: Option<&str>,
    next_fetch_secs: i64,
) -> Result<(), sqlx::Error> {
    sqlx::query!(
        r#"
        UPDATE feeds
        SET last_fetched_at = NOW(),
            next_fetch_at   = NOW() + ($1 || ' seconds')::INTERVAL,
            etag            = $2,
            last_modified   = $3
        WHERE id = $4
        "#,
        next_fetch_secs.to_string(),
        etag,
        last_modified,
        id,
    )
    .execute(db)
    .await?;
    Ok(())
}
```

**src/db/items.rs:**

```rust
use crate::models::{Item, ItemsQuery};
use crate::parser::ParsedItem;
use sqlx::PgPool;

/// Insert items ใหม่ด้วย deduplication (ON CONFLICT DO NOTHING)
/// คืน ids ของ items ที่ถูก insert จริง ๆ (ไม่ใช่ duplicate)
pub async fn insert_items(
    db: &PgPool,
    feed_id: i64,
    items: &[ParsedItem],
) -> Result<Vec<Item>, sqlx::Error> {
    let mut inserted = Vec::new();

    for item in items {
        // สร้าง search_vec จาก title + content สำหรับ FTS
        let search_text = format!(
            "{} {}",
            item.title.as_deref().unwrap_or(""),
            item.content.as_deref().unwrap_or("")
        );

        let result = sqlx::query_as!(
            Item,
            r#"
            INSERT INTO items (feed_id, guid, title, url, content, published_at, search_vec)
            VALUES ($1, $2, $3, $4, $5, $6, to_tsvector('english', $7))
            ON CONFLICT (feed_id, guid) DO NOTHING
            RETURNING id, feed_id, guid, title, url, content, published_at, fetched_at, read_at
            "#,
            feed_id,
            item.guid,
            item.title,
            item.url,
            item.content,
            item.published_at,
            search_text,
        )
        .fetch_optional(db)
        .await?;

        if let Some(new_item) = result {
            inserted.push(new_item);
        }
    }

    Ok(inserted)
}

/// Query items ด้วย filter หลายแบบ รวมถึง full-text search
pub async fn query_items(
    db: &PgPool,
    q: &ItemsQuery,
) -> Result<Vec<Item>, sqlx::Error> {
    // เราใช้ dynamic query building ด้วย String
    // ในระบบ production อาจใช้ crate เช่น sea-query สำหรับ type-safe query building

    let mut conditions: Vec<String> = Vec::new();
    let mut param_idx: i32 = 1;

    // NOTE: เราใช้ format! เพื่อ build query structure เท่านั้น
    // ค่าจริงทั้งหมดส่งเป็น parameters เพื่อป้องกัน SQL injection

    if q.feed_id.is_some() {
        conditions.push(format!("feed_id = ${param_idx}"));
        param_idx += 1;
    }
    if q.unread == Some(true) {
        conditions.push("read_at IS NULL".to_string());
    }
    if q.since.is_some() {
        conditions.push(format!("published_at >= ${param_idx}"));
        param_idx += 1;
    }
    if q.before.is_some() {
        conditions.push(format!("published_at < ${param_idx}"));
        param_idx += 1;
    }
    if q.q.is_some() {
        conditions.push(format!("search_vec @@ to_tsquery('english', ${param_idx})"));
        param_idx += 1;
    }

    let where_clause = if conditions.is_empty() {
        String::new()
    } else {
        format!("WHERE {}", conditions.join(" AND "))
    };

    let sql = format!(
        r#"
        SELECT id, feed_id, guid, title, url, content, published_at, fetched_at, read_at
        FROM items
        {where_clause}
        ORDER BY COALESCE(published_at, fetched_at) DESC
        LIMIT ${param_idx} OFFSET ${}
        "#,
        param_idx + 1,
    );

    // ใช้ QueryBuilder แบบ manual bind
    let mut qb = sqlx::query_as::<_, Item>(&sql);
    if let Some(fid) = q.feed_id {
        qb = qb.bind(fid);
    }
    if let Some(since) = q.since {
        qb = qb.bind(since);
    }
    if let Some(before) = q.before {
        qb = qb.bind(before);
    }
    if let Some(ref keyword) = q.q {
        let tsquery = build_tsquery(keyword);
        qb = qb.bind(tsquery);
    }
    qb = qb.bind(q.limit).bind(q.offset);

    qb.fetch_all(db).await
}

/// Mark item ว่าอ่านแล้ว
pub async fn mark_item_read(db: &PgPool, id: i64) -> Result<bool, sqlx::Error> {
    let result = sqlx::query!(
        "UPDATE items SET read_at = NOW() WHERE id = $1 AND read_at IS NULL",
        id
    )
    .execute(db)
    .await?;
    Ok(result.rows_affected() > 0)
}

/// สร้าง PostgreSQL tsquery expression จาก keyword string ที่ผู้ใช้ป้อนมา
///
/// ตัวอย่าง: "rust async" -> "rust & async"
/// ตัวอย่าง: "async; DROP" -> "async & DROP"  (special chars ถูกกรองออก)
pub fn build_tsquery(input: &str) -> String {
    let terms: Vec<String> = input
        .split_whitespace()
        .filter(|s| !s.is_empty())
        .map(|s| {
            s.chars()
                .filter(|c| c.is_alphanumeric() || *c == '-')
                .collect::<String>()
                .to_lowercase()
        })
        .filter(|s| !s.is_empty())
        .collect();

    if terms.is_empty() {
        String::new()
    } else {
        terms.join(" & ")
    }
}
```

สังเกตจุดสำคัญ: เราใช้ `ON CONFLICT (feed_id, guid) DO NOTHING` แทนการ check ก่อน insert ซึ่งทำให้ atomic และ thread-safe กว่า ถ้า background fetcher หลาย instance รันพร้อมกันก็ไม่มีปัญหา race condition

### ขั้นที่ 4: HTTP API ด้วย Axum

**src/routes/feeds.rs:**

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::IntoResponse,
    Json,
};
use std::sync::Arc;

use crate::{
    db::feeds,
    models::{AppError, CreateFeedRequest},
    AppState,
};

/// POST /feeds — เพิ่ม feed URL ใหม่
pub async fn create_feed(
    State(state): State<Arc<AppState>>,
    Json(req): Json<CreateFeedRequest>,
) -> Result<impl IntoResponse, AppError> {
    // Validate URL format เบื้องต้น
    if !req.url.starts_with("http://") && !req.url.starts_with("https://") {
        return Err(AppError::BadRequest("URL must start with http:// or https://".into()));
    }

    let interval = state.config.default_fetch_interval as i32;
    let feed = feeds::create_feed(&state.db, &req, interval).await?;

    Ok((StatusCode::CREATED, Json(feed)))
}

/// GET /feeds — รายการ feeds ทั้งหมด
pub async fn list_feeds(
    State(state): State<Arc<AppState>>,
) -> Result<impl IntoResponse, AppError> {
    let feeds = feeds::list_feeds(&state.db).await?;
    Ok(Json(feeds))
}

/// DELETE /feeds/:id — ลบ feed (cascade ลบ items ด้วย)
pub async fn delete_feed(
    State(state): State<Arc<AppState>>,
    Path(id): Path<i64>,
) -> Result<impl IntoResponse, AppError> {
    let deleted = feeds::delete_feed(&state.db, id).await?;
    if deleted {
        Ok(StatusCode::NO_CONTENT.into_response())
    } else {
        Err(AppError::NotFound)
    }
}

/// GET /feeds/:id/items — รายการ items ของ feed นั้น (paginated)
pub async fn get_feed_items(
    State(state): State<Arc<AppState>>,
    Path(id): Path<i64>,
    axum::extract::Query(mut q): axum::extract::Query<crate::models::ItemsQuery>,
) -> Result<impl IntoResponse, AppError> {
    // ตรวจสอบว่า feed มีอยู่จริง
    feeds::get_feed(&state.db, id)
        .await?
        .ok_or(AppError::NotFound)?;

    q.feed_id = Some(id);
    let items = crate::db::items::query_items(&state.db, &q).await?;
    Ok(Json(items))
}
```

**src/routes/items.rs:**

```rust
use axum::{
    extract::{Path, Query, State},
    response::IntoResponse,
    Json,
};
use std::sync::Arc;

use crate::{
    db::items,
    models::{AppError, ItemsQuery},
    AppState,
};

/// GET /items — รายการ items พร้อม filtering และ full-text search
pub async fn list_items(
    State(state): State<Arc<AppState>>,
    Query(q): Query<ItemsQuery>,
) -> Result<impl IntoResponse, AppError> {
    let result = items::query_items(&state.db, &q).await?;
    Ok(Json(result))
}

/// PATCH /items/:id/read — mark item ว่าอ่านแล้ว
pub async fn mark_read(
    State(state): State<Arc<AppState>>,
    Path(id): Path<i64>,
) -> Result<impl IntoResponse, AppError> {
    let updated = items::mark_item_read(&state.db, id).await?;
    if updated {
        Ok(Json(serde_json::json!({ "ok": true })))
    } else {
        // ไม่ found หรือ already read
        Err(AppError::NotFound)
    }
}
```

**src/routes/mod.rs:**

```rust
use axum::{routing::{delete, get, patch, post}, Router};
use std::sync::Arc;
use crate::AppState;

pub fn app_router() -> Router<Arc<AppState>> {
    Router::new()
        // Feeds
        .route("/feeds",                post(feeds::create_feed).get(feeds::list_feeds))
        .route("/feeds/:id",            delete(feeds::delete_feed))
        .route("/feeds/:id/items",      get(feeds::get_feed_items))
        // Items
        .route("/items",                get(items::list_items))
        .route("/items/:id/read",       patch(items::mark_read))
        // OPML
        .route("/opml",                 post(opml::import_opml).get(opml::export_opml))
        // Webhooks
        .route("/webhooks",             post(webhooks::create_webhook).get(webhooks::list_webhooks))
        .route("/webhooks/:id",         delete(webhooks::delete_webhook))
}

mod feeds;
mod items;
mod opml;
mod webhooks;
```

### ขั้นที่ 5: Background Fetcher ด้วย `tokio::time::interval`

**src/fetcher.rs:**

```rust
use std::sync::Arc;
use std::time::Duration;
use tracing::{error, info, warn};

use crate::{
    db::{feeds, items},
    models::Webhook,
    parser,
    AppState,
};

/// Entry point ของ background fetcher loop
/// ควรเรียกด้วย `tokio::spawn(run_fetcher(state))` ใน main
pub async fn run_fetcher(state: Arc<AppState>) {
    // tick ทุก 60 วินาที แล้วตรวจว่า feed ไหนถึงเวลาดึงบ้าง
    let mut interval = tokio::time::interval(Duration::from_secs(60));

    loop {
        interval.tick().await;

        let due_feeds = match feeds::feeds_due_for_fetch(&state.db).await {
            Ok(f) => f,
            Err(e) => {
                error!("Failed to query feeds due for fetch: {e}");
                continue;
            }
        };

        if due_feeds.is_empty() {
            continue;
        }

        info!("Fetching {} feeds due for update", due_feeds.len());

        // spawn task แยกสำหรับแต่ละ feed เพื่อให้ parallel
        let handles: Vec<_> = due_feeds
            .into_iter()
            .map(|feed| {
                let state = Arc::clone(&state);
                tokio::spawn(async move {
                    if let Err(e) = fetch_one_feed(&state, &feed).await {
                        warn!("Error fetching feed {} ({}): {e}", feed.id, feed.url);
                    }
                })
            })
            .collect();

        // รอให้ทุก feed เสร็จก่อน tick ถัดไป (optional — ขึ้นอยู่กับ design)
        for h in handles {
            let _ = h.await;
        }
    }
}

/// ดึงและ process feed เดี่ยว
async fn fetch_one_feed(
    state: &Arc<AppState>,
    feed: &crate::models::Feed,
) -> anyhow::Result<()> {
    // สร้าง request พร้อม conditional headers เพื่อลด bandwidth
    let mut request = state.http.get(&feed.url);
    if let Some(etag) = &feed.etag {
        request = request.header("If-None-Match", etag);
    }
    if let Some(lm) = &feed.last_modified {
        request = request.header("If-Modified-Since", lm);
    }

    let response = request.send().await?;

    // 304 Not Modified: ไม่มีอะไรเปลี่ยนแปลง
    if response.status() == reqwest::StatusCode::NOT_MODIFIED {
        info!("Feed {} not modified (304)", feed.id);
        let next_secs = feed.fetch_interval_secs as i64;
        feeds::update_feed_after_fetch(&state.db, feed.id, None, None, next_secs).await?;
        return Ok(());
    }

    // อ่าน Cache-Control: max-age=N สำหรับ dynamic TTL
    let cache_max_age = parse_cache_control_max_age(
        response.headers().get("Cache-Control")
            .and_then(|v| v.to_str().ok())
            .unwrap_or("")
    );
    let next_fetch_secs = cache_max_age
        .unwrap_or(feed.fetch_interval_secs as u64)
        .max(60) as i64; // ขั้นต่ำ 60 วินาที

    // อ่าน etag และ last-modified สำหรับ conditional request ครั้งหน้า
    let new_etag = response.headers()
        .get("ETag")
        .and_then(|v| v.to_str().ok())
        .map(String::from);
    let new_lm = response.headers()
        .get("Last-Modified")
        .and_then(|v| v.to_str().ok())
        .map(String::from);

    let bytes = response.bytes().await?;

    // Parse feed
    let (feed_title, parsed_items) = parser::parse_feed(&bytes)?;

    // อัปเดต feed title ถ้ายังไม่มี
    if feed.title.is_none() {
        if let Some(title) = feed_title {
            sqlx::query!(
                "UPDATE feeds SET title = $1 WHERE id = $2 AND title IS NULL",
                title, feed.id
            )
            .execute(&state.db)
            .await?;
        }
    }

    // Insert items (dedup ด้วย ON CONFLICT DO NOTHING)
    let new_items = items::insert_items(&state.db, feed.id, &parsed_items).await?;

    info!(
        "Feed {}: fetched {} items, {} new",
        feed.id,
        parsed_items.len(),
        new_items.len()
    );

    // Fire webhooks สำหรับ items ใหม่
    if !new_items.is_empty() {
        fire_webhooks(state, feed, &new_items).await;
    }

    // อัปเดต feed metadata
    feeds::update_feed_after_fetch(
        &state.db,
        feed.id,
        new_etag.as_deref(),
        new_lm.as_deref(),
        next_fetch_secs,
    ).await?;

    Ok(())
}

/// Parse `Cache-Control: max-age=N` header
fn parse_cache_control_max_age(header: &str) -> Option<u64> {
    header
        .split(',')
        .map(str::trim)
        .find(|s| s.starts_with("max-age="))
        .and_then(|s| s["max-age=".len()..].parse().ok())
}

/// ส่ง webhook notification ไปยัง registered URLs
async fn fire_webhooks(
    state: &Arc<AppState>,
    feed: &crate::models::Feed,
    new_items: &[crate::models::Item],
) {
    let webhooks: Vec<Webhook> = match sqlx::query_as!(
        Webhook,
        "SELECT * FROM webhooks"
    )
    .fetch_all(&state.db)
    .await
    {
        Ok(w) => w,
        Err(e) => {
            error!("Failed to load webhooks: {e}");
            return;
        }
    };

    for item in new_items {
        for webhook in &webhooks {
            // ตรวจสอบ keyword filter
            if !webhook.keyword.is_empty() {
                let matches = item.title.as_deref().unwrap_or("")
                    .to_lowercase()
                    .contains(&webhook.keyword.to_lowercase())
                    || item.content.as_deref().unwrap_or("")
                        .to_lowercase()
                        .contains(&webhook.keyword.to_lowercase());
                if !matches {
                    continue;
                }
            }

            let payload = crate::models::WebhookPayload {
                event: "new_item",
                item,
                feed,
            };

            let wh_url = webhook.url.clone();
            let http = state.http.clone();
            tokio::spawn(async move {
                if let Err(e) = http.post(&wh_url).json(&payload).send().await {
                    warn!("Webhook POST to {wh_url} failed: {e}");
                }
            });
        }
    }
}
```

ประเด็นสำคัญเกี่ยวกับ `tokio::time::interval`:
- `interval.tick().await` จะ block จนกว่าจะถึงเวลา tick ถัดไป
- ถ้า logic ในแต่ละ tick ใช้เวลานานกว่า interval, ticks จะ "pile up" ใน Tokio's `MissedTickBehavior::Burst` (default) — ถ้าต้องการ skip แทนต้องตั้ง `interval.set_missed_tick_behavior(MissedTickBehavior::Skip)`
- เราแยก "check ว่า feed ไหนถึงเวลา" กับ "ดึง feed" ออกจากกัน ทำให้การเพิ่มนาทีหรือลดนาทีของ interval ไม่กระทบ logic

### ขั้นที่ 6: OPML Import/Export

OPML (Outline Processor Markup Language) เป็น XML format มาตรฐานสำหรับ export/import feed subscriptions ทำให้ผู้ใช้ย้าย subscriptions ระหว่าง RSS readers ได้

**src/opml.rs:**

```rust
use quick_xml::{events::Event, Reader, Writer};
use std::io::Cursor;

#[derive(Debug)]
pub struct OpmlFeed {
    pub title: String,
    pub xml_url: String,
    pub html_url: Option<String>,
}

/// Parse OPML XML bytes และคืนรายการ feeds
pub fn parse_opml(bytes: &[u8]) -> anyhow::Result<Vec<OpmlFeed>> {
    let mut reader = Reader::from_reader(bytes);
    reader.config_mut().trim_text(true);

    let mut feeds = Vec::new();
    let mut buf = Vec::new();

    loop {
        match reader.read_event_into(&mut buf) {
            Ok(Event::Empty(ref e)) | Ok(Event::Start(ref e)) => {
                // ค้นหา <outline type="rss" xmlUrl="..."> หรือ <outline type="atom" ...>
                let name = e.name();
                if name.as_ref() == b"outline" {
                    let mut feed_type = None;
                    let mut xml_url = None;
                    let mut title = None;
                    let mut html_url = None;

                    for attr in e.attributes().flatten() {
                        let key = std::str::from_utf8(attr.key.as_ref())
                            .unwrap_or("")
                            .to_lowercase();
                        let value = attr.unescape_value()
                            .map(|v| v.into_owned())
                            .unwrap_or_default();

                        match key.as_str() {
                            "type"    => feed_type = Some(value),
                            "xmlurl"  => xml_url = Some(value),
                            "title" | "text" => {
                                if title.is_none() {
                                    title = Some(value);
                                }
                            }
                            "htmlurl" => html_url = Some(value),
                            _ => {}
                        }
                    }

                    // outline ที่มี xmlUrl ถือว่าเป็น feed subscription
                    if let Some(url) = xml_url {
                        let is_feed = feed_type
                            .as_deref()
                            .map(|t| matches!(t, "rss" | "atom" | "feed" | "pie"))
                            .unwrap_or(true); // ถ้าไม่มี type attribute ให้ถือว่าเป็น feed

                        if is_feed && !url.is_empty() {
                            feeds.push(OpmlFeed {
                                title: title.unwrap_or_else(|| url.clone()),
                                xml_url: url,
                                html_url,
                            });
                        }
                    }
                }
            }
            Ok(Event::Eof) => break,
            Err(e) => return Err(anyhow::anyhow!("OPML parse error: {e}")),
            _ => {}
        }
        buf.clear();
    }

    Ok(feeds)
}

/// สร้าง OPML XML string จากรายการ feeds
pub fn generate_opml(feeds: &[crate::models::Feed]) -> anyhow::Result<String> {
    let mut writer = Writer::new_with_indent(Cursor::new(Vec::new()), b' ', 2);

    use quick_xml::events::{BytesDecl, BytesEnd, BytesStart, BytesText};

    // XML declaration
    writer.write_event(Event::Decl(BytesDecl::new("1.0", Some("UTF-8"), None)))?;

    // <opml version="2.0">
    let mut opml_elem = BytesStart::new("opml");
    opml_elem.push_attribute(("version", "2.0"));
    writer.write_event(Event::Start(opml_elem))?;

    // <head><title>RSS Aggregator Export</title></head>
    writer.write_event(Event::Start(BytesStart::new("head")))?;
    writer.write_event(Event::Start(BytesStart::new("title")))?;
    writer.write_event(Event::Text(BytesText::new("RSS Aggregator Export")))?;
    writer.write_event(Event::End(BytesEnd::new("title")))?;
    writer.write_event(Event::End(BytesEnd::new("head")))?;

    // <body>
    writer.write_event(Event::Start(BytesStart::new("body")))?;

    for feed in feeds {
        let mut outline = BytesStart::new("outline");
        outline.push_attribute(("type", "rss"));
        outline.push_attribute(("xmlUrl", feed.url.as_str()));
        let title = feed.title.as_deref().unwrap_or(&feed.url);
        outline.push_attribute(("title", title));
        outline.push_attribute(("text", title));
        if let Some(desc) = &feed.description {
            outline.push_attribute(("description", desc.as_str()));
        }
        writer.write_event(Event::Empty(outline))?;
    }

    writer.write_event(Event::End(BytesEnd::new("body")))?;
    writer.write_event(Event::End(BytesEnd::new("opml")))?;

    let result = writer.into_inner().into_inner();
    Ok(String::from_utf8(result)?)
}
```

**src/routes/opml.rs:**

```rust
use axum::{
    body::Bytes,
    extract::State,
    http::{header, StatusCode},
    response::{IntoResponse, Response},
    Json,
};
use std::sync::Arc;
use crate::{db::feeds, models::AppError, opml, AppState};

/// POST /opml — import feeds จาก OPML file (multipart/form-data หรือ raw bytes)
pub async fn import_opml(
    State(state): State<Arc<AppState>>,
    body: Bytes,
) -> Result<impl IntoResponse, AppError> {
    let parsed_feeds = opml::parse_opml(&body)
        .map_err(|e| AppError::BadRequest(e.to_string()))?;

    if parsed_feeds.is_empty() {
        return Err(AppError::BadRequest("No valid feeds found in OPML".into()));
    }

    let interval = state.config.default_fetch_interval as i32;
    let mut imported = 0usize;

    for feed in parsed_feeds {
        let req = crate::models::CreateFeedRequest {
            url: feed.xml_url,
            fetch_interval_secs: Some(interval),
        };
        // INSERT OR IGNORE: ถ้า URL ซ้ำ skip ไป
        match feeds::create_feed(&state.db, &req, interval).await {
            Ok(_) => imported += 1,
            Err(sqlx::Error::Database(e)) if e.is_unique_violation() => {
                // Feed URL ซ้ำ — ข้ามไป
            }
            Err(e) => return Err(AppError::Database(e)),
        }
    }

    Ok(Json(serde_json::json!({
        "imported": imported,
        "message": format!("Imported {imported} new feeds")
    })))
}

/// GET /opml — export ทุก feed เป็น OPML XML
pub async fn export_opml(
    State(state): State<Arc<AppState>>,
) -> Result<Response, AppError> {
    let all_feeds = feeds::list_feeds(&state.db).await?;
    let xml = opml::generate_opml(&all_feeds)
        .map_err(|e| AppError::BadRequest(e.to_string()))?;

    Ok((
        StatusCode::OK,
        [(header::CONTENT_TYPE, "text/xml; charset=utf-8"),
         (header::CONTENT_DISPOSITION, "attachment; filename=\"feeds.opml\"")],
        xml,
    )
        .into_response())
}
```

### ขั้นที่ 7: Main Entry Point และ Startup

**src/main.rs:**

```rust
use std::sync::Arc;
use tracing::info;
use tracing_subscriber::EnvFilter;

mod config;
mod db;
mod fetcher;
mod models;
mod opml;
mod parser;
mod routes;

pub use config::Config;

#[derive(Clone)]
pub struct AppState {
    pub db: sqlx::PgPool,
    pub config: Arc<Config>,
    pub http: reqwest::Client,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Setup logging
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info"))
        )
        .init();

    let config = Arc::new(Config::from_env()?);
    info!("Starting RSS Aggregator on {}:{}", config.server_host, config.server_port);

    // Database connection pool
    let db = sqlx::PgPool::connect(&config.database_url).await?;
    sqlx::migrate!("./migrations").run(&db).await?;

    // HTTP client สำหรับ fetching feeds
    let http = reqwest::Client::builder()
        .user_agent("RssAggregator/1.0 (Rust)")
        .timeout(std::time::Duration::from_secs(30))
        .gzip(true)
        .build()?;

    let state = Arc::new(AppState { db, config: Arc::clone(&config), http });

    // Start background fetcher ใน separate task
    let fetcher_state = Arc::clone(&state);
    tokio::spawn(async move {
        fetcher::run_fetcher(fetcher_state).await;
    });

    // Build and run Axum server
    let addr = format!("{}:{}", config.server_host, config.server_port);
    let listener = tokio::net::TcpListener::bind(&addr).await?;
    info!("Listening on {addr}");

    axum::serve(listener, routes::app_router().with_state(state)).await?;

    Ok(())
}
```

---

## การทดสอบ (Testing)

### Verification Tests (รันจริงใน Scratchpad)

โปรเจคนี้ต้องการ PostgreSQL สำหรับ integration tests เต็มรูปแบบ แต่เราสามารถ test logic หลักที่สำคัญที่สุด 3 อย่างได้โดยไม่ต้องพึ่ง database ได้แก่:

1. **Feed parsing** — parse RSS 2.0 และ Atom XML string แล้วตรวจจำนวน items และ title
2. **Deduplication logic** — simulate การ deduplicate ด้วย HashSet
3. **Full-text search query building** — `build_tsquery` function

ด้านล่างเป็น test file ที่รันได้จริง (ไม่ต้องการ database):

```toml
# Cargo.toml สำหรับ verify project
[package]
name = "rss_agg_verify"
version = "0.1.0"
edition = "2021"

[dependencies]
feed-rs = "2"
```

```rust
// src/main.rs — ไม่ต้องการ database
use feed_rs::parser;
use std::collections::HashSet;

fn main() {
    println!("RSS Aggregator verification — running tests...");
}

fn build_tsquery(input: &str) -> String {
    let terms: Vec<String> = input
        .split_whitespace()
        .filter(|s| !s.is_empty())
        .map(|s| {
            s.chars()
                .filter(|c| c.is_alphanumeric() || *c == '-')
                .collect::<String>()
                .to_lowercase()
        })
        .filter(|s| !s.is_empty())
        .collect();

    if terms.is_empty() {
        String::new()
    } else {
        terms.join(" & ")
    }
}

#[derive(Debug, PartialEq, Eq, Hash)]
struct ItemKey {
    feed_id: u64,
    guid_or_url: String,
}

fn deduplicate_items(items: Vec<ItemKey>) -> Vec<ItemKey> {
    let mut seen = HashSet::new();
    let mut result = Vec::new();
    for item in items {
        if seen.insert(format!("{}:{}", item.feed_id, item.guid_or_url)) {
            result.push(item);
        }
    }
    result
}

#[cfg(test)]
mod tests {
    use super::*;

    const RSS2_SAMPLE: &str = r#"<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0">
  <channel>
    <title>The Rust Blog</title>
    <link>https://blog.rust-lang.org</link>
    <description>News and articles from the Rust programming language team</description>
    <item>
      <title>Announcing Rust 1.80.0</title>
      <link>https://blog.rust-lang.org/2024/07/25/Rust-1.80.0.html</link>
      <description>The Rust team is happy to announce a new version of Rust, 1.80.0.</description>
      <pubDate>Thu, 25 Jul 2024 00:00:00 +0000</pubDate>
      <guid>https://blog.rust-lang.org/2024/07/25/Rust-1.80.0.html</guid>
    </item>
    <item>
      <title>Announcing Rust 1.79.0</title>
      <link>https://blog.rust-lang.org/2024/06/13/Rust-1.79.0.html</link>
      <description>The Rust team is happy to announce a new version of Rust, 1.79.0.</description>
      <pubDate>Thu, 13 Jun 2024 00:00:00 +0000</pubDate>
      <guid>https://blog.rust-lang.org/2024/06/13/Rust-1.79.0.html</guid>
    </item>
    <item>
      <title>Rustconf 2024 Recap</title>
      <link>https://blog.rust-lang.org/2024/09/15/rustconf-2024.html</link>
      <description>A recap of Rustconf 2024 in Montreal.</description>
      <pubDate>Sun, 15 Sep 2024 00:00:00 +0000</pubDate>
      <guid>https://blog.rust-lang.org/2024/09/15/rustconf-2024.html</guid>
    </item>
  </channel>
</rss>"#;

    const ATOM_SAMPLE: &str = r#"<?xml version="1.0" encoding="utf-8"?>
<feed xmlns="http://www.w3.org/2005/Atom">
  <title>This Week in Rust</title>
  <link href="https://this-week-in-rust.org"/>
  <updated>2024-07-24T00:00:00Z</updated>
  <id>https://this-week-in-rust.org/atom.xml</id>
  <entry>
    <title>This Week in Rust 558</title>
    <link href="https://this-week-in-rust.org/blog/2024/07/24/this-week-in-rust-558/"/>
    <id>https://this-week-in-rust.org/blog/2024/07/24/this-week-in-rust-558/</id>
    <updated>2024-07-24T00:00:00Z</updated>
    <summary>Hello and welcome to another issue of This Week in Rust!</summary>
  </entry>
  <entry>
    <title>This Week in Rust 557</title>
    <link href="https://this-week-in-rust.org/blog/2024/07/17/this-week-in-rust-557/"/>
    <id>https://this-week-in-rust.org/blog/2024/07/17/this-week-in-rust-557/</id>
    <updated>2024-07-17T00:00:00Z</updated>
    <summary>Hello and welcome to another issue of This Week in Rust!</summary>
  </entry>
</feed>"#;

    #[test]
    fn test_rss2_parse_item_count_and_title() {
        let feed = parser::parse(RSS2_SAMPLE.as_bytes()).expect("RSS2 parse failed");
        assert_eq!(feed.entries.len(), 3);
        let first_title = feed.entries[0].title.as_ref()
            .map(|t| t.content.as_str()).unwrap_or("");
        assert_eq!(first_title, "Announcing Rust 1.80.0");
    }

    #[test]
    fn test_rss2_parse_feed_title() {
        let feed = parser::parse(RSS2_SAMPLE.as_bytes()).expect("RSS2 parse failed");
        let title = feed.title.as_ref().map(|t| t.content.as_str()).unwrap_or("");
        assert_eq!(title, "The Rust Blog");
    }

    #[test]
    fn test_atom_parse_item_count_and_title() {
        let feed = parser::parse(ATOM_SAMPLE.as_bytes()).expect("Atom parse failed");
        assert_eq!(feed.entries.len(), 2);
        let first_title = feed.entries[0].title.as_ref()
            .map(|t| t.content.as_str()).unwrap_or("");
        assert_eq!(first_title, "This Week in Rust 558");
    }

    #[test]
    fn test_deduplication_same_item_twice() {
        let items = vec![
            ItemKey { feed_id: 1, guid_or_url: "https://example.com/post/1".into() },
            ItemKey { feed_id: 1, guid_or_url: "https://example.com/post/2".into() },
            ItemKey { feed_id: 1, guid_or_url: "https://example.com/post/1".into() }, // ซ้ำ
        ];
        let deduped = deduplicate_items(items);
        assert_eq!(deduped.len(), 2);
    }

    #[test]
    fn test_deduplication_different_feeds() {
        let items = vec![
            ItemKey { feed_id: 1, guid_or_url: "https://example.com/post/1".into() },
            ItemKey { feed_id: 2, guid_or_url: "https://example.com/post/1".into() },
        ];
        let deduped = deduplicate_items(items);
        assert_eq!(deduped.len(), 2); // ต่าง feed = ไม่ duplicate
    }

    #[test]
    fn test_build_tsquery_single_word() {
        assert_eq!(build_tsquery("rust"), "rust");
    }

    #[test]
    fn test_build_tsquery_multiple_words() {
        assert_eq!(build_tsquery("async programming"), "async & programming");
    }

    #[test]
    fn test_build_tsquery_strips_special_chars() {
        let q = build_tsquery("rust; DROP TABLE items--");
        let parts: Vec<&str> = q.split(" & ").collect();
        assert!(parts.contains(&"rust"));
        assert!(!q.contains(';'));
    }

    #[test]
    fn test_build_tsquery_empty_input() {
        assert_eq!(build_tsquery("   "), "");
    }

    #[test]
    fn test_rss2_item_links_present() {
        let feed = parser::parse(RSS2_SAMPLE.as_bytes()).expect("parse failed");
        for entry in &feed.entries {
            assert!(!entry.links.is_empty(), "ทุก item ต้องมีอย่างน้อยหนึ่ง link");
        }
    }
}
```

### ผลลัพธ์จากการรัน `cargo test` จริง

```
running 10 tests
test tests::test_build_tsquery_empty_input ... ok
test tests::test_build_tsquery_single_word ... ok
test tests::test_build_tsquery_strips_special_chars ... ok
test tests::test_build_tsquery_multiple_words ... ok
test tests::test_deduplication_different_feeds ... ok
test tests::test_deduplication_same_item_twice ... ok
test tests::test_atom_parse_item_count_and_title ... ok
test tests::test_rss2_parse_item_count_and_title ... ok
test tests::test_rss2_parse_feed_title ... ok
test tests::test_rss2_item_links_present ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

### Integration Tests ด้วย Database จริง

สำหรับ integration tests ที่ต้องการ PostgreSQL ใช้ `sqlx::test` macro:

```rust
// tests/integration_test.rs
#[cfg(test)]
mod tests {
    use sqlx::PgPool;

    // sqlx::test จะสร้าง fresh database สำหรับแต่ละ test อัตโนมัติ
    // ต้องตั้ง DATABASE_URL env variable

    #[sqlx::test(migrations = "./migrations")]
    async fn test_create_and_list_feeds(db: PgPool) {
        use rss_aggregator::db::feeds;
        use rss_aggregator::models::CreateFeedRequest;

        let req = CreateFeedRequest {
            url: "https://blog.rust-lang.org/feed.xml".into(),
            fetch_interval_secs: Some(600),
        };

        let feed = feeds::create_feed(&db, &req, 900).await
            .expect("create_feed should succeed");

        assert_eq!(feed.url, req.url);
        assert_eq!(feed.fetch_interval_secs, 600);

        let all = feeds::list_feeds(&db).await.expect("list_feeds should succeed");
        assert_eq!(all.len(), 1);
    }

    #[sqlx::test(migrations = "./migrations")]
    async fn test_item_deduplication_via_database(db: PgPool) {
        use rss_aggregator::{db::{feeds, items}, models::CreateFeedRequest, parser::ParsedItem};

        let feed = feeds::create_feed(&db, &CreateFeedRequest {
            url: "https://example.com/feed.xml".into(),
            fetch_interval_secs: None,
        }, 900).await.unwrap();

        let item = ParsedItem {
            guid: "unique-guid-001".into(),
            title: Some("Test Item".into()),
            url: Some("https://example.com/1".into()),
            content: Some("Test content".into()),
            published_at: None,
        };

        // Insert ครั้งแรก — ต้องสำเร็จ
        let inserted_first = items::insert_items(&db, feed.id, &[item]).await.unwrap();
        assert_eq!(inserted_first.len(), 1, "First insert should succeed");

        let item_dup = ParsedItem {
            guid: "unique-guid-001".into(), // guid เดิม
            title: Some("Test Item (Updated Title)".into()),
            url: Some("https://example.com/1".into()),
            content: Some("Updated content".into()),
            published_at: None,
        };

        // Insert ซ้ำ — ON CONFLICT DO NOTHING จะทำให้ไม่มีแถวใหม่
        let inserted_dup = items::insert_items(&db, feed.id, &[item_dup]).await.unwrap();
        assert_eq!(inserted_dup.len(), 0, "Duplicate insert should be ignored");
    }
}
```

---

## กับดักที่พบบ่อย (Common Pitfalls)

### Pitfall 1: RSS Quirks — `guid` อาจไม่ใช่ URL

หลาย feed ใช้ `<guid>` ที่ไม่ใช่ URL จริง เช่น `<guid>12345</guid>` หรือ `<guid isPermaLink="false">some-internal-id</guid>` ถ้าเราสมมติว่า guid คือ URL แล้ว fetch ไปจะ error:

```rust
// ผิด: สมมติว่า guid เป็น URL เสมอ
let url = entry.id.clone(); // entry.id อาจเป็นแค่ "12345"

// ถูก: ใช้ links ก่อน ถ้าไม่มีค่อยใช้ id
let url = entry.links
    .first()
    .map(|l| l.href.clone())
    .or_else(|| {
        if entry.id.starts_with("http") {
            Some(entry.id.clone())
        } else {
            None
        }
    });
```

นอกจากนี้ `feed-rs` normalize `entry.id` ให้แล้ว แต่ในบาง edge case อาจเป็น string ว่าง — ควร fallback ไปใช้ URL แทน

### Pitfall 2: tokio::time::interval กับ MissedTickBehavior

เมื่อ logic ใน loop ใช้เวลานานกว่า interval (เช่น fetch 100 feeds ใช้เวลา 2 นาที แต่ interval คือ 1 นาที) ticks ที่ missไปจะ "burst" ออกมาทันทีในรอบถัดไป ทำให้ระบบ fetch ถี่กว่าที่ตั้งใจ:

```rust
// ปัญหา: default MissedTickBehavior::Burst
let mut interval = tokio::time::interval(Duration::from_secs(60));
// ถ้า logic ใช้เวลา 3 นาที จะได้ tick burst 3 ครั้งทันที

// วิธีแก้: ใช้ Skip เพื่อข้าม ticks ที่ missed ไป
let mut interval = tokio::time::interval(Duration::from_secs(60));
interval.set_missed_tick_behavior(tokio::time::MissedTickBehavior::Skip);
```

### Pitfall 3: SQL Injection ใน Dynamic Query Building

เมื่อสร้าง WHERE clause แบบ dynamic เป็นเรื่องปกติที่จะสร้าง `$1, $2, $3...` แบบ dynamic ซึ่งมีโอกาสผิดพลาดได้:

```rust
// ผิด: เอา keyword เข้าไปใน query string โดยตรง
let sql = format!("SELECT * FROM items WHERE content ILIKE '%{}%'", keyword);
// ถ้า keyword = "'; DROP TABLE items; --" จะเป็น SQL injection

// ถูก: ใช้ parameterized query เสมอ
let sql = "SELECT * FROM items WHERE search_vec @@ to_tsquery('english', $1)";
let tsquery = build_tsquery(keyword); // กรอง special chars ออกก่อน
sqlx::query_as::<_, Item>(sql).bind(tsquery).fetch_all(db).await?;
```

ฟังก์ชัน `build_tsquery` ที่เราเขียนก็กรอง semicolons และ special chars ออกอยู่แล้ว แต่ถ้า PostgreSQL `to_tsquery` ยังมีปัญหา ให้ใช้ `websearch_to_tsquery` แทน ซึ่ง lenient กว่าและ accept input ที่ไม่ perfect ได้

### Pitfall 4: reqwest gzip decompression กับ `Content-Encoding`

`reqwest` ถ้าตั้ง `.gzip(true)` จะ decompresses response โดยอัตโนมัติ แต่ถ้า server ส่ง `Content-Encoding: gzip` และ `reqwest` ก็ decode แล้ว ค่า `Content-Length` header จะไม่ตรงกับ bytes จริง อย่าใช้ `Content-Length` สำหรับ size check:

```rust
// ผิด: ใช้ Content-Length เป็น bytes size
let size = response.headers()
    .get("Content-Length")
    .and_then(|v| v.to_str().ok())
    .and_then(|s| s.parse::<usize>().ok())
    .unwrap_or(0);

// ถูก: ใช้ bytes จริงที่ได้รับมา
let bytes = response.bytes().await?;
let size = bytes.len(); // size จริงหลัง decompression
```

นอกจากนี้บาง feed server ส่ง `Content-Encoding: gzip` โดยไม่มี `Accept-Encoding` ใน request ซึ่ง technically ผิด RFC แต่ `reqwest` จัดการให้อยู่แล้ว

---

## การ Package และ Deploy

### Docker

```dockerfile
# Build stage
FROM rust:1.80-slim as builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
# Pre-build dependencies เพื่อ cache layer
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm src/main.rs

# Build application
COPY src ./src
COPY migrations ./migrations
RUN touch src/main.rs && cargo build --release

# Runtime stage — minimal image
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libssl3 ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/rss-aggregator /usr/local/bin/
COPY --from=builder /app/migrations /migrations
EXPOSE 3000
CMD ["/usr/local/bin/rss-aggregator"]
```

### docker-compose.yml

```yaml
version: "3.9"
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: rss_aggregator
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build: .
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://postgres:secret@db/rss_aggregator
      HOST: 0.0.0.0
      PORT: 3000
      DEFAULT_FETCH_INTERVAL_SECS: 900
      RUST_LOG: info
    ports:
      - "3000:3000"

volumes:
  pgdata:
```

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่:
./target/release/rss-aggregator

# ตรวจ binary size
du -sh ./target/release/rss-aggregator
# ประมาณ 8-15 MB ก่อน strip

# Strip debug symbols เพื่อลด size
strip ./target/release/rss-aggregator
# ลดลงเหลือประมาณ 4-7 MB
```

### Database Migrations

เราใช้ `sqlx migrate` ซึ่ง embedded ใน binary โดย `sqlx::migrate!("./migrations")` จะ compile migrations เข้าไปใน binary ทำให้ไม่ต้องนำ migration files ติดไปด้วย:

```bash
# รัน migrations ด้วย sqlx CLI (สำหรับ development)
sqlx migrate run --database-url "postgres://localhost/rss_aggregator"

# สร้าง migration ใหม่
sqlx migrate add add_webhook_last_triggered
```

---

## กับดักเพิ่มเติม: Webhook Retry Loop

เมื่อ webhook endpoint ของ user ล่มชั่วคราว การส่ง fire-and-forget โดยไม่มี retry จะทำให้ notifications หาย แต่การ retry ง่ายเกินไปก็ทำให้ระบบส่ง spam ได้:

```rust
// ระบบ retry พื้นฐานด้วย exponential backoff
async fn send_webhook_with_retry(
    client: &reqwest::Client,
    url: &str,
    payload: &serde_json::Value,
) -> Result<(), reqwest::Error> {
    let mut delay = std::time::Duration::from_secs(1);
    for attempt in 0..3 {
        match client.post(url).json(payload).send().await {
            Ok(resp) if resp.status().is_success() => return Ok(()),
            Ok(resp) => {
                tracing::warn!(
                    "Webhook {url} returned HTTP {}: attempt {}/3",
                    resp.status(), attempt + 1
                );
            }
            Err(e) => {
                tracing::warn!("Webhook {url} failed: {e}: attempt {}/3", attempt + 1);
            }
        }
        tokio::time::sleep(delay).await;
        delay *= 2; // exponential backoff: 1s, 2s, 4s
    }
    Ok(()) // ยอมแพ้หลัง 3 ครั้ง — ไม่ propagate error ไปยัง caller
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม `feed_type` Detection อัตโนมัติ

ขณะนี้เราต้องการให้ผู้ใช้ระบุ URL ของ feed โดยตรง แต่ในโลกจริง user มักรู้แค่ URL ของ website ไม่ใช่ feed URL โจทย์: เพิ่ม logic ใน `POST /feeds` ที่ถ้า URL ชี้ไปยัง HTML page จะ:
1. Fetch HTML page นั้น
2. ค้นหา `<link rel="alternate" type="application/rss+xml">` หรือ `<link rel="alternate" type="application/atom+xml">` ใน `<head>`
3. ใช้ URL จาก `href` attribute นั้นเป็น feed URL แทน

hint: ใช้ `scraper` crate หรือ `quick-xml` อ่าน HTML แล้ว find link elements

### แบบฝึกหัดที่ 2: เพิ่ม Authentication ด้วย API Key

ขณะนี้ API ยังไม่มี authentication ทำให้ใครก็ได้ที่เข้าถึง server สามารถเพิ่ม/ลบ feeds ได้ โจทย์: เพิ่ม Axum middleware ที่:
1. ตรวจสอบ `Authorization: Bearer <api-key>` header ในทุก request
2. เปรียบเทียบกับ `API_KEY` env variable (hash ด้วย `sha256` ก่อนเปรียบ)
3. Return 401 ถ้าไม่มีหรือ key ไม่ถูก

hint: ดูตัวอย่าง Axum middleware ใน `tower::ServiceBuilder` หรือใช้ `axum::middleware::from_fn`

### แบบฝึกหัดที่ 3: เพิ่ม Feed Statistics Endpoint

โจทย์: สร้าง endpoint `GET /feeds/:id/stats` ที่คืนข้อมูลสถิติของ feed นั้น ประกอบด้วย:
- จำนวน items ทั้งหมด
- จำนวน items ที่อ่านแล้ว / ยังไม่ได้อ่าน
- วันที่ของ item ล่าสุดที่ดึงมา (`MAX(fetched_at)`)
- ความถี่การ publish โดยเฉลี่ย (items ต่อวัน ใน 30 วันที่ผ่านมา)

hint: ใช้ SQL aggregate functions: `COUNT(*)`, `COUNT(*) FILTER (WHERE read_at IS NULL)`, `AVG`

### แบบฝึกหัดที่ 4: เพิ่ม Category/Tag System

ขณะนี้ feeds ทั้งหมดอยู่รวมกัน ในโลกจริงผู้ใช้ต้องการจัด feeds เป็นหมวดหมู่ เช่น Tech, Science, News โจทย์:
1. เพิ่ม `categories` table และ `feed_categories` junction table
2. เพิ่ม `POST /categories`, `GET /categories`, `DELETE /categories/:id`
3. เพิ่ม `POST /feeds/:id/categories/:category_id` สำหรับ assign category
4. เพิ่ม query parameter `category_id` ใน `GET /items` เพื่อ filter items ตาม category

hint: ใช้ JOIN ระหว่าง `items`, `feeds`, `feed_categories` ใน `query_items`

---

## สรุป

ในโปรเจคนี้เราได้สร้าง RSS/Atom Feed Aggregator Service ครบวงจร ซึ่งครอบคลุม pattern สำคัญหลายอย่างที่ใช้บ่อยในระบบ production:

**Pattern ที่ได้เรียน:**

1. **Feed normalization** — `feed-rs` แสดงให้เห็นว่า crate ที่ดีช่วยซ่อน complexity ของ format quirks ไปให้เราได้มาก ทำให้โค้ดของเราโฟกัสที่ business logic
2. **Background worker + shared state** — `tokio::spawn` + `Arc<AppState>` เป็น pattern พื้นฐานของ concurrent Rust services ที่ต้องมีทั้ง HTTP server และ background task
3. **Database-level deduplication** — `UNIQUE constraint` + `ON CONFLICT DO NOTHING` เป็น pattern ที่ atomic และ thread-safe กว่าการ check ก่อน insert
4. **Full-text search** — `tsvector` + `GIN index` ใน PostgreSQL ให้ performance ดีมากสำหรับ keyword search โดยไม่ต้องใช้ Elasticsearch สำหรับข้อมูลขนาดกลาง
5. **HTTP Conditional Requests** — `ETag` / `If-None-Match` เป็น pattern สำคัญสำหรับ reducing server load เมื่อ poll external endpoints ซ้ำๆ
6. **Webhook fanout** — fire-and-forget notification เป็น pattern ที่ใช้กับระบบ alert/monitoring ทุกประเภท

**เชื่อมโยงไปโปรเจคถัดไป:**

Project B05 (Webhook Relay Service) จะนำ webhook notification pattern ที่เราเรียนไปต่อยอด — แทนที่จะ fire webhooks โดยตรง เราจะสร้าง relay layer ที่รับ webhook เข้ามา transform payload แล้ว forward ไปยัง multiple destinations พร้อม retry queue และ delivery guarantees

---

**โปรเจคก่อนหน้า:** [project-b03-image-processing-api.md](project-b03-image-processing-api.md) | **โปรเจคถัดไป:** [project-b05-webhook-relay.md](project-b05-webhook-relay.md)
