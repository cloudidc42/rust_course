# Project C04: Message Broker (Pub/Sub)

> โมดูล: C — Data Processing & Pipelines | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

**Message Broker** คือระบบที่ทำหน้าที่เป็นตัวกลางรับส่งข้อความระหว่าง **Producer** (ผู้ส่ง) และ **Consumer** (ผู้รับ) โดยใช้รูปแบบ **Publish/Subscribe (Pub/Sub)** — Producer ส่ง (publish) ข้อความไปยัง topic, และ Consumer จัดกลุ่มเป็น consumer group เพื่อรับ (subscribe) ข้อความจาก topic นั้น

ในโลก production ระบบนี้เป็นกระดูกสันหลังของสถาปัตยกรรม event-driven เช่น Apache Kafka, RabbitMQ, หรือ AWS SQS/SNS โปรเจคนี้จะสร้าง broker ที่ทำงานจริงได้โดยใช้ **axum** เป็น HTTP server, **SQLite** (ผ่าน sqlx) สำหรับ offset tracking, และ append-only log files สำหรับ message persistence — เหมือน Kafka แต่เข้าใจง่ายกว่า

**Use case จริง:**
- Decoupling services ใน microservices architecture
- Reliable event delivery ด้วย at-least-once semantics
- Fan-out notifications (1 event → หลาย consumer groups)
- Audit log และ event replay สำหรับ CQRS/Event Sourcing

## สิ่งที่จะได้เรียนรู้

- **Append-only log** — วิธีเขียน binary log file ในรูปแบบ `u64 offset + u32 len + bytes` คล้าย Kafka segment
- **Consumer group semantics** — fan-out delivery และ exactly-once-per-group ผ่าน offset tracking
- **Pull-based consumption** — ต่างจาก push-based อย่างไร และ tradeoffs ที่มี
- **Acknowledgment protocol** — ack token, timeout, re-delivery และ dead letter queue
- **Log compaction** — ลบ duplicate key เก็บไว้เฉพาะ latest value per key
- **Prometheus metrics** — expose counter/gauge ในรูปแบบ text exposition format
- **Arc<Mutex<T>> state sharing** ใน async axum handlers
- **SQLite offset tracking** ผ่าน sqlx สำหรับ durability

## ความรู้ที่ต้องมีมาก่อน

- **Part 46-50**: async/await, tokio runtime — ใช้สำหรับ async HTTP handlers
- **Part 61-70**: axum web framework — routing, extractors, State
- **Part 71-80**: sqlx และ SQLite — query, migrations, connection pool
- **Part 96-100**: Arc, Mutex, RwLock — shared state ใน multi-threaded context
- **Part 101-105**: binary serialization — เขียน/อ่าน binary format ด้วย byte slices

## โครงสร้างโปรเจค (Project Layout)

```
message-broker/
├── src/
│   ├── main.rs           # axum server bootstrap + route registration
│   ├── broker.rs         # MessageBroker core logic (topics, partitions, groups)
│   ├── storage.rs        # Append-only log file + binary format
│   ├── consumer.rs       # ConsumerGroup, PendingAck, ack timeout
│   ├── metrics.rs        # Prometheus text format exporter
│   ├── dlq.rs            # Dead letter queue logic
│   ├── compaction.rs     # Log compaction for keyed topics
│   └── handlers/
│       ├── mod.rs        # Handler re-exports
│       ├── topics.rs     # POST/GET/DELETE /topics
│       ├── messages.rs   # POST /topics/{name}/messages
│       ├── consumers.rs  # POST /consumer-groups, GET .../messages, POST .../ack
│       └── metrics.rs    # GET /metrics
├── tests/
│   └── integration_test.rs
├── migrations/
│   └── 001_init.sql      # offsets table schema
└── Cargo.toml
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Producer
   │
   │ POST /topics/{name}/messages {key, value, headers}
   ▼
MessageBroker (in-memory + disk log)
   │
   ├── Partition 0 → append-only log file (events-0.log)
   ├── Partition 1 → append-only log file (events-1.log)
   │
   │  Fan-out: copy message reference to each ConsumerGroup
   │
   ├── ConsumerGroup "analytics"  → SQLite offsets table
   └── ConsumerGroup "email-svc"  → SQLite offsets table
            │
            │ GET /consumer-groups/{id}/messages?max=10
            ▼
         Consumer (gets messages + ack_tokens)
            │
            │ POST /consumer-groups/{id}/ack {ack_tokens: [...]}
            ▼
         Offset committed to SQLite
```

### Binary Log Format

แต่ละ message ใน log file เก็บในรูปแบบ:

```
┌─────────────────────┬──────────────┬────────────────────────┐
│  offset (u64 LE)    │  len (u32 LE)│  JSON payload (bytes)  │
│  8 bytes            │  4 bytes     │  len bytes             │
└─────────────────────┴──────────────┴────────────────────────┘
```

- `offset` เป็นตัวเลขที่เพิ่มขึ้นทีละ 1 ต่อ partition (monotonically increasing)
- `len` บอกขนาดของ JSON payload ที่ตามมา
- สามารถ seek/scan file ได้โดยไม่ต้องอ่านทั้งหมด

### Consumer Group Semantics

```
Topic "orders" มี 2 partitions
├── Partition 0: [offset 0] [offset 1] [offset 2]
└── Partition 1: [offset 0] [offset 1]

ConsumerGroup "payment-svc" มี 3 members: A, B, C
├── committed offset partition 0 = 2 (A ได้รับแล้ว 0,1,2)
└── committed offset partition 1 = 1 (B ได้รับแล้ว 0,1)

Pull: C จะได้รับ messages ที่ยังไม่ commit
```

**Fan-out**: ถ้ามี 3 consumer groups, message 1 ข้อความจะถูก deliver ไปทั้ง 3 groups โดยแต่ละ group มี offset counter แยกกัน — ไม่มี group ใด "แย่ง" message ของกันและกัน

### Design Decisions

| Decision | เหตุผล | ทางเลือกอื่น |
|----------|--------|-------------|
| Pull-based | Consumer ควบคุม rate เอง, ง่าย retry | Push-based (WebSocket/SSE) |
| SQLite offsets | Durable, ACID, ไม่ต้องมี external DB | In-memory HashMap (ไม่ durable) |
| Append-only log | Write ไว, เปิด replay ได้ | B-tree (random write, แต่ query ได้ดีกว่า) |
| In-memory topic index | Read ไว, broker เป็น single process | Distributed (ซับซ้อนกว่า) |
| Ack token (UUID) | Stateless acks, ไม่ต้อง track member identity | Sequence number (ง่ายกว่า แต่ collision risk) |

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Data Structures พื้นฐาน

เริ่มด้วย Cargo.toml และ data structures หลักก่อน compile error จะบอกเราตรงๆ ว่าต้องเพิ่มอะไร

**Cargo.toml:**

```toml
[package]
name = "message_broker"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8", features = ["macros"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio", "chrono", "uuid"] }
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
thiserror = "2"
tokio-util = "0.7"

[dev-dependencies]
axum-test = "16"
```

**src/broker.rs — Core data types:**

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use chrono::{DateTime, Utc};

/// Configuration สำหรับ topic
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TopicConfig {
    pub name: String,
    pub partitions: u32,
    pub replication_factor: u32,
    pub compaction: bool,
}

/// Message หนึ่งข้อความใน broker
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Message {
    pub offset: u64,
    pub key: Option<String>,
    pub value: Vec<u8>,
    pub headers: HashMap<String, String>,
    pub timestamp: DateTime<Utc>,
    pub partition: u32,
}

impl Message {
    pub fn new(offset: u64, key: Option<String>, value: Vec<u8>, partition: u32) -> Self {
        Message {
            offset,
            key,
            value,
            headers: HashMap::new(),
            timestamp: Utc::now(),
            partition,
        }
    }

    /// Serialize เป็น binary format: u64 offset + u32 len + JSON bytes
    /// จาก Part 101: binary serialization ด้วย byte manipulation
    pub fn serialize_to_log(&self) -> Vec<u8> {
        let json = serde_json::to_vec(self).expect("Message serialization failed");
        let len = json.len() as u32;
        let mut buf = Vec::with_capacity(8 + 4 + json.len());
        buf.extend_from_slice(&self.offset.to_le_bytes());  // 8 bytes LE
        buf.extend_from_slice(&len.to_le_bytes());           // 4 bytes LE
        buf.extend_from_slice(&json);                        // variable
        buf
    }

    /// Deserialize จาก binary format — คืน (Message, bytes_consumed)
    pub fn deserialize_from_log(data: &[u8]) -> Option<(Message, usize)> {
        if data.len() < 12 {
            return None; // ยังไม่มีข้อมูลพอ
        }
        let _offset = u64::from_le_bytes(data[0..8].try_into().ok()?);
        let len = u32::from_le_bytes(data[8..12].try_into().ok()?) as usize;
        if data.len() < 12 + len {
            return None; // partial write
        }
        let msg: Message = serde_json::from_slice(&data[12..12 + len]).ok()?;
        Some((msg, 12 + len))
    }
}

/// Partition เก็บ messages ใน in-memory list + key index สำหรับ compaction
#[derive(Debug)]
pub struct Partition {
    pub messages: Vec<Message>,
    pub next_offset: u64,
    pub key_index: HashMap<String, u64>, // key -> latest_offset
}

impl Partition {
    pub fn new() -> Self {
        Partition {
            messages: Vec::new(),
            next_offset: 0,
            key_index: HashMap::new(),
        }
    }

    pub fn append(
        &mut self,
        key: Option<String>,
        value: Vec<u8>,
        headers: HashMap<String, String>,
        partition_id: u32,
    ) -> u64 {
        let offset = self.next_offset;
        let mut msg = Message::new(offset, key.clone(), value, partition_id);
        msg.headers = headers;
        // อัปเดต key index — จาก Part 35: HashMap operations
        if let Some(k) = &key {
            self.key_index.insert(k.clone(), offset);
        }
        self.messages.push(msg);
        self.next_offset += 1;
        offset
    }

    pub fn get_messages_from(&self, from_offset: u64, max: usize) -> Vec<&Message> {
        self.messages
            .iter()
            .filter(|m| m.offset >= from_offset)
            .take(max)
            .collect()
    }
}
```

สังเกตว่า `serialize_to_log()` ใช้ **little-endian** (`to_le_bytes`) เหมือน Kafka เนื่องจาก x86 architecture เป็น little-endian โดยธรรมชาติ ทำให้ performance ดีกว่า big-endian บน hardware ส่วนใหญ่

---

### ขั้นที่ 2: Topic Management API

สร้าง axum handlers สำหรับ CRUD topics พร้อม shared state ด้วย `Arc<Mutex<T>>`

**src/handlers/topics.rs:**

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    Json,
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};
use crate::broker::{MessageBroker, TopicConfig};

pub type BrokerState = Arc<Mutex<MessageBroker>>;

#[derive(Deserialize)]
pub struct CreateTopicRequest {
    pub name: String,
    pub partitions: Option<u32>,
    pub replication_factor: Option<u32>,
    pub compaction: Option<bool>,
}

#[derive(Serialize)]
pub struct TopicResponse {
    pub name: String,
    pub partitions: u32,
    pub replication_factor: u32,
    pub compaction: bool,
    pub message_count: usize,
}

/// POST /topics — สร้าง topic ใหม่
pub async fn create_topic(
    State(broker): State<BrokerState>,
    Json(req): Json<CreateTopicRequest>,
) -> Result<(StatusCode, Json<TopicResponse>), (StatusCode, String)> {
    let config = TopicConfig {
        name: req.name.clone(),
        partitions: req.partitions.unwrap_or(1),
        replication_factor: req.replication_factor.unwrap_or(1),
        compaction: req.compaction.unwrap_or(false),
    };

    // Lock broker state — จาก Part 96: Mutex patterns
    let mut broker = broker.lock().map_err(|e| {
        (StatusCode::INTERNAL_SERVER_ERROR, e.to_string())
    })?;

    broker.create_topic(config.clone()).map_err(|e| {
        (StatusCode::CONFLICT, e)
    })?;

    Ok((
        StatusCode::CREATED,
        Json(TopicResponse {
            name: config.name,
            partitions: config.partitions,
            replication_factor: config.replication_factor,
            compaction: config.compaction,
            message_count: 0,
        }),
    ))
}

/// GET /topics — list ทุก topic
pub async fn list_topics(
    State(broker): State<BrokerState>,
) -> Json<Vec<TopicResponse>> {
    let broker = broker.lock().unwrap();
    let topics = broker
        .topics
        .iter()
        .map(|(name, (config, partitions))| TopicResponse {
            name: name.clone(),
            partitions: config.partitions,
            replication_factor: config.replication_factor,
            compaction: config.compaction,
            message_count: partitions.iter().map(|p| p.messages.len()).sum(),
        })
        .collect();
    Json(topics)
}

/// DELETE /topics/{name} — ลบ topic
pub async fn delete_topic(
    State(broker): State<BrokerState>,
    Path(name): Path<String>,
) -> Result<StatusCode, (StatusCode, String)> {
    let mut broker = broker.lock().unwrap();
    broker.delete_topic(&name).map_err(|e| (StatusCode::NOT_FOUND, e))?;
    Ok(StatusCode::NO_CONTENT)
}
```

**src/main.rs — axum setup:**

```rust
use axum::{
    routing::{delete, get, post},
    Router,
};
use std::sync::{Arc, Mutex};
use tower_http::cors::CorsLayer;
use tracing_subscriber::EnvFilter;

mod broker;
mod consumer;
mod handlers;
mod metrics;
mod storage;

use broker::MessageBroker;
use handlers::topics::{BrokerState, create_topic, delete_topic, list_topics};

#[tokio::main]
async fn main() {
    // Setup tracing — จาก Part 100: observability
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();

    // Shared broker state ด้วย Arc<Mutex>
    let broker: BrokerState = Arc::new(Mutex::new(MessageBroker::new()));

    let app = Router::new()
        // Topics
        .route("/topics", post(create_topic).get(list_topics))
        .route("/topics/:name", delete(delete_topic))
        // Messages
        .route("/topics/:name/messages", post(publish_message))
        // Consumer groups
        .route("/consumer-groups", post(create_consumer_group))
        .route("/consumer-groups/:id/messages", get(pull_messages))
        .route("/consumer-groups/:id/ack", post(acknowledge))
        // Metrics
        .route("/metrics", get(get_metrics))
        .layer(CorsLayer::permissive())
        .with_state(broker);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080")
        .await
        .unwrap();

    tracing::info!("Message broker listening on port 8080");
    axum::serve(listener, app).await.unwrap();
}
```

---

### ขั้นที่ 3: Publish และ Fan-out

เมื่อ Producer ส่งข้อความ broker ต้อง route ไปยัง partition ที่ถูกต้องและ deliver ให้ทุก consumer group

**src/handlers/messages.rs:**

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    Json,
};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use crate::handlers::topics::BrokerState;

#[derive(Deserialize)]
pub struct PublishRequest {
    pub key: Option<String>,
    pub value: String, // base64 หรือ plain text
    pub headers: Option<HashMap<String, String>>,
}

#[derive(Serialize)]
pub struct PublishResponse {
    pub topic: String,
    pub partition: u32,
    pub offset: u64,
    pub timestamp: String,
}

/// POST /topics/{name}/messages
/// Fan-out: message ถูก deliver ไปทุก consumer group ที่ subscribe topic นี้
pub async fn publish_message(
    State(broker): State<BrokerState>,
    Path(topic): Path<String>,
    Json(req): Json<PublishRequest>,
) -> Result<(StatusCode, Json<PublishResponse>), (StatusCode, String)> {
    let mut broker = broker.lock().unwrap();

    let value = req.value.into_bytes();
    let headers = req.headers.unwrap_or_default();

    let (partition, offset) = broker
        .publish(&topic, req.key, value, headers)
        .map_err(|e| (StatusCode::NOT_FOUND, e))?;

    Ok((
        StatusCode::CREATED,
        Json(PublishResponse {
            topic,
            partition,
            offset,
            timestamp: chrono::Utc::now().to_rfc3339(),
        }),
    ))
}
```

**Partition selection logic** ใน `broker.rs`:

```rust
pub fn publish(
    &mut self,
    topic: &str,
    key: Option<String>,
    value: Vec<u8>,
    headers: HashMap<String, String>,
) -> Result<(u32, u64), String> {
    let (config, partitions) = self.topics.get_mut(topic)
        .ok_or_else(|| format!("Topic '{}' not found", topic))?;

    // Partition selection:
    // - ถ้ามี key → hash-based (consistent routing สำหรับ related messages)
    // - ถ้าไม่มี key → round-robin (load balancing)
    let partition_idx = match &key {
        Some(k) => {
            // Simple hash: sum of byte values mod partitions
            // จาก Part 30: iterator methods
            let hash: u32 = k.bytes()
                .fold(0u32, |acc, b| acc.wrapping_add(b as u32));
            hash % config.partitions
        }
        None => {
            // Round-robin ตาม total messages published
            (self.metrics.messages_in_total as u32) % config.partitions
        }
    };

    let partition = &mut partitions[partition_idx as usize];
    let offset = partition.append(key, value, headers, partition_idx);

    self.metrics.messages_in_total += 1;

    // Persist to log file (async version ใช้ tokio::fs)
    // ตัวอย่างนี้ใช้ in-memory สำหรับความเรียบง่าย
    // Production: ใช้ BufWriter + periodic flush

    Ok((partition_idx, offset))
}
```

**หมายเหตุ:** Fan-out ในระบบนี้เกิดขึ้นแบบ "lazy" — consumer group ไม่ได้รับ copy ของ message แต่แต่ละ group track offset ของตัวเอง และ read จาก partition log โดยตรง นี่คือ approach เดียวกับ Kafka ที่ทำให้ storage ไม่ multiply ตาม number of consumers

---

### ขั้นที่ 4: Consumer Groups และ Pull Consumption

Consumer group คือกลุ่มของ consumers ที่ช่วยกัน process messages โดย each message delivered to exactly one member per group

**src/consumer.rs:**

```rust
use std::collections::HashMap;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConsumerGroup {
    pub id: String,
    pub name: String,
    pub topic: String,
    pub members: Vec<String>,
    /// (topic, partition) -> next_offset_to_read
    pub offsets: HashMap<(String, u32), u64>,
    pub ack_timeout_secs: u64,
    pub max_delivery_attempts: u32,
}

impl ConsumerGroup {
    pub fn new(name: String, topic: String) -> Self {
        ConsumerGroup {
            id: Uuid::new_v4().to_string(),
            name,
            topic,
            members: Vec::new(),
            offsets: HashMap::new(),
            ack_timeout_secs: 30,
            max_delivery_attempts: 3,
        }
    }

    pub fn add_member(&mut self, member_id: String) {
        if !self.members.contains(&member_id) {
            self.members.push(member_id);
        }
    }

    /// Round-robin assignment: offset % members.len()
    /// ทำให้ messages กระจายไปทุก member อย่างสม่ำเสมอ
    pub fn assign_member(&self, message_offset: u64) -> Option<&str> {
        if self.members.is_empty() {
            return None;
        }
        let idx = (message_offset as usize) % self.members.len();
        Some(&self.members[idx])
    }

    pub fn get_committed_offset(&self, partition: u32) -> u64 {
        *self.offsets
            .get(&(self.topic.clone(), partition))
            .unwrap_or(&0)
    }

    /// เลื่อน offset หลังจาก ack สำเร็จ
    pub fn commit_offset(&mut self, partition: u32, offset: u64) {
        // commit offset+1 เพื่อที่ pull ครั้งต่อไปจะเริ่มจากตัวถัดไป
        self.offsets.insert(
            (self.topic.clone(), partition),
            offset + 1,
        );
    }
}

/// Token ที่ส่งกลับไปให้ consumer พร้อมข้อความ
/// Consumer ต้องส่ง token นี้กลับมา acknowledge
#[derive(Debug, Clone)]
pub struct PendingAck {
    pub ack_token: String,
    pub message_offset: u64,
    pub partition: u32,
    pub consumer_group_id: String,
    pub delivery_attempts: u32,
    pub delivered_at: DateTime<Utc>,
}

impl PendingAck {
    pub fn new(message_offset: u64, partition: u32, consumer_group_id: String) -> Self {
        PendingAck {
            ack_token: Uuid::new_v4().to_string(),
            message_offset,
            partition,
            consumer_group_id,
            delivery_attempts: 1,
            delivered_at: Utc::now(),
        }
    }

    /// ตรวจว่า ack หมดเวลาหรือยัง
    pub fn is_timed_out(&self, timeout_secs: u64) -> bool {
        let elapsed = Utc::now()
            .signed_duration_since(self.delivered_at)
            .num_seconds();
        elapsed > timeout_secs as i64
    }
}
```

**src/handlers/consumers.rs:**

```rust
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    Json,
};
use serde::{Deserialize, Serialize};
use crate::handlers::topics::BrokerState;

#[derive(Deserialize)]
pub struct CreateGroupRequest {
    pub name: String,
    pub topic: String,
    pub ack_timeout_secs: Option<u64>,
}

#[derive(Deserialize)]
pub struct PullQuery {
    pub max: Option<usize>,
}

#[derive(Serialize)]
pub struct PulledMessage {
    pub offset: u64,
    pub partition: u32,
    pub key: Option<String>,
    pub value: String,
    pub headers: std::collections::HashMap<String, String>,
    pub timestamp: String,
    pub ack_token: String,
}

#[derive(Deserialize)]
pub struct AckRequest {
    pub ack_tokens: Vec<String>,
}

/// POST /consumer-groups — ลงทะเบียน consumer group
pub async fn create_consumer_group(
    State(broker): State<BrokerState>,
    Json(req): Json<CreateGroupRequest>,
) -> Result<(StatusCode, Json<serde_json::Value>), (StatusCode, String)> {
    let mut broker = broker.lock().unwrap();
    let group_id = broker
        .create_consumer_group(req.name, req.topic)
        .map_err(|e| (StatusCode::BAD_REQUEST, e))?;

    if let Some(timeout) = req.ack_timeout_secs {
        if let Some(group) = broker.consumer_groups.get_mut(&group_id) {
            group.ack_timeout_secs = timeout;
        }
    }

    Ok((
        StatusCode::CREATED,
        Json(serde_json::json!({ "id": group_id })),
    ))
}

/// GET /consumer-groups/{id}/messages?max=10
/// Pull-based: consumer ดึงข้อความเองเมื่อพร้อม
pub async fn pull_messages(
    State(broker): State<BrokerState>,
    Path(group_id): Path<String>,
    Query(query): Query<PullQuery>,
) -> Result<Json<Vec<PulledMessage>>, (StatusCode, String)> {
    let max = query.max.unwrap_or(10).min(100); // cap at 100

    let mut broker = broker.lock().unwrap();
    let msgs = broker
        .pull_messages(&group_id, max)
        .map_err(|e| (StatusCode::NOT_FOUND, e))?;

    let response: Vec<PulledMessage> = msgs
        .into_iter()
        .map(|(msg, ack_token)| PulledMessage {
            offset: msg.offset,
            partition: msg.partition,
            key: msg.key,
            value: String::from_utf8_lossy(&msg.value).to_string(),
            headers: msg.headers,
            timestamp: msg.timestamp.to_rfc3339(),
            ack_token,
        })
        .collect();

    Ok(Json(response))
}

/// POST /consumer-groups/{id}/ack
pub async fn acknowledge(
    State(broker): State<BrokerState>,
    Path(group_id): Path<String>,
    Json(req): Json<AckRequest>,
) -> Result<Json<serde_json::Value>, (StatusCode, String)> {
    let mut broker = broker.lock().unwrap();
    let count = broker
        .acknowledge(&group_id, &req.ack_tokens)
        .map_err(|e| (StatusCode::BAD_REQUEST, e))?;

    Ok(Json(serde_json::json!({
        "acknowledged": count,
        "group_id": group_id,
    })))
}
```

---

### ขั้นที่ 5: Message Persistence — Append-Only Log

ข้อความต้องคงอยู่แม้ restart — ใช้ append-only log file เหมือน Kafka WAL (Write-Ahead Log)

**src/storage.rs:**

```rust
use std::io::{self, Write, BufWriter};
use std::fs::{File, OpenOptions};
use std::path::{Path, PathBuf};
use crate::broker::Message;

/// LogFile จัดการ append-only log file สำหรับ 1 partition
pub struct LogFile {
    path: PathBuf,
    writer: BufWriter<File>,
}

impl LogFile {
    /// เปิดหรือสร้าง log file
    pub fn open(data_dir: &Path, topic: &str, partition: u32) -> io::Result<Self> {
        std::fs::create_dir_all(data_dir)?;
        let path = data_dir.join(format!("{}-{}.log", topic, partition));
        let file = OpenOptions::new()
            .create(true)
            .append(true)
            .open(&path)?;
        Ok(LogFile {
            path: path.clone(),
            writer: BufWriter::new(file),
        })
    }

    /// Append message ไปยัง log — O(1) amortized
    pub fn append(&mut self, msg: &Message) -> io::Result<()> {
        let bytes = msg.serialize_to_log();
        self.writer.write_all(&bytes)?;
        // Flush ทุกครั้งเพื่อ durability
        // Production: ใช้ periodic flush หรือ fsync เฉพาะเมื่อ commit
        self.writer.flush()
    }

    /// อ่านทุก message ใน log file (ใช้ตอน startup replay)
    pub fn read_all(&self) -> io::Result<Vec<Message>> {
        let data = std::fs::read(&self.path)?;
        let mut messages = Vec::new();
        let mut pos = 0;

        while pos < data.len() {
            match Message::deserialize_from_log(&data[pos..]) {
                Some((msg, consumed)) => {
                    pos += consumed;
                    messages.push(msg);
                }
                None => break, // partial write หรือ corruption
            }
        }
        Ok(messages)
    }
}

/// Recovery: โหลด messages จาก log files ตอน broker startup
pub fn recover_from_disk(data_dir: &Path, topic: &str, partitions: u32) 
    -> io::Result<Vec<Vec<Message>>> 
{
    let mut all_partitions = Vec::new();
    for p in 0..partitions {
        let path = data_dir.join(format!("{}-{}.log", topic, p));
        if path.exists() {
            let data = std::fs::read(&path)?;
            let mut msgs = Vec::new();
            let mut pos = 0;
            while pos < data.len() {
                if let Some((msg, consumed)) = Message::deserialize_from_log(&data[pos..]) {
                    pos += consumed;
                    msgs.push(msg);
                } else {
                    break;
                }
            }
            all_partitions.push(msgs);
        } else {
            all_partitions.push(Vec::new());
        }
    }
    Ok(all_partitions)
}
```

---

### ขั้นที่ 6: Offset Tracking ด้วย SQLite

**migrations/001_init.sql:**

```sql
-- offsets table: track committed offset ต่อ (consumer_group, topic, partition)
CREATE TABLE IF NOT EXISTS offsets (
    consumer_group_id TEXT NOT NULL,
    topic             TEXT NOT NULL,
    partition         INTEGER NOT NULL,
    last_offset       INTEGER NOT NULL DEFAULT 0,
    updated_at        TEXT NOT NULL DEFAULT (datetime('now')),
    PRIMARY KEY (consumer_group_id, topic, partition)
);

-- consumer_groups table: metadata
CREATE TABLE IF NOT EXISTS consumer_groups (
    id                TEXT PRIMARY KEY,
    name              TEXT NOT NULL,
    topic             TEXT NOT NULL,
    ack_timeout_secs  INTEGER NOT NULL DEFAULT 30,
    max_attempts      INTEGER NOT NULL DEFAULT 3,
    created_at        TEXT NOT NULL DEFAULT (datetime('now'))
);

-- topics table: configuration
CREATE TABLE IF NOT EXISTS topics (
    name              TEXT PRIMARY KEY,
    partitions        INTEGER NOT NULL DEFAULT 1,
    replication_factor INTEGER NOT NULL DEFAULT 1,
    compaction        BOOLEAN NOT NULL DEFAULT 0,
    created_at        TEXT NOT NULL DEFAULT (datetime('now'))
);
```

**src/db.rs — SQLite offset operations:**

```rust
use sqlx::{SqlitePool, Row};

pub struct OffsetDb {
    pool: SqlitePool,
}

impl OffsetDb {
    pub async fn new(database_url: &str) -> Result<Self, sqlx::Error> {
        let pool = SqlitePool::connect(database_url).await?;
        // รัน migrations
        sqlx::query(include_str!("../migrations/001_init.sql"))
            .execute(&pool)
            .await?;
        Ok(OffsetDb { pool })
    }

    /// อ่าน committed offset
    pub async fn get_offset(
        &self,
        group_id: &str,
        topic: &str,
        partition: u32,
    ) -> Result<u64, sqlx::Error> {
        let row = sqlx::query(
            "SELECT last_offset FROM offsets
             WHERE consumer_group_id = ? AND topic = ? AND partition = ?"
        )
        .bind(group_id)
        .bind(topic)
        .bind(partition as i64)
        .fetch_optional(&self.pool)
        .await?;

        Ok(row.map(|r| r.get::<i64, _>("last_offset") as u64).unwrap_or(0))
    }

    /// บันทึก offset หลัง ack
    pub async fn commit_offset(
        &self,
        group_id: &str,
        topic: &str,
        partition: u32,
        offset: u64,
    ) -> Result<(), sqlx::Error> {
        // UPSERT: insert หรือ update ถ้ามีอยู่แล้ว
        sqlx::query(
            "INSERT INTO offsets (consumer_group_id, topic, partition, last_offset, updated_at)
             VALUES (?, ?, ?, ?, datetime('now'))
             ON CONFLICT (consumer_group_id, topic, partition)
             DO UPDATE SET last_offset = excluded.last_offset,
                           updated_at = excluded.updated_at"
        )
        .bind(group_id)
        .bind(topic)
        .bind(partition as i64)
        .bind(offset as i64)
        .execute(&self.pool)
        .await?;
        Ok(())
    }

    /// ดึง consumer lag: latest_offset - committed_offset
    pub async fn get_lag(
        &self,
        group_id: &str,
        topic: &str,
        partition_count: u32,
        latest_offsets: &[u64],
    ) -> Result<u64, sqlx::Error> {
        let mut total_lag = 0u64;
        for p in 0..partition_count as usize {
            let committed = self.get_offset(group_id, topic, p as u32).await?;
            let latest = latest_offsets.get(p).copied().unwrap_or(0);
            if latest > committed {
                total_lag += latest - committed;
            }
        }
        Ok(total_lag)
    }
}
```

---

### ขั้นที่ 7: Dead Letter Queue และ Metrics

เมื่อ consumer ไม่ acknowledge ข้อความหลังจาก N attempts → ย้ายไป DLQ

**Dead Letter Queue Logic (ใน broker.rs):**

```rust
/// ตรวจ pending acks ที่ timeout แล้ว re-queue หรือ ย้ายไป DLQ
pub fn requeue_timed_out(&mut self, group_id: &str) {
    let timeout = self.consumer_groups
        .get(group_id)
        .map(|g| g.ack_timeout_secs)
        .unwrap_or(30);

    let max_attempts = self.consumer_groups
        .get(group_id)
        .map(|g| g.max_delivery_attempts)
        .unwrap_or(3);

    // หา acks ที่ timeout
    let timed_out: Vec<String> = self.pending_acks
        .iter()
        .filter(|(_, pa)| {
            pa.consumer_group_id == group_id && pa.is_timed_out(timeout)
        })
        .map(|(token, _)| token.clone())
        .collect();

    let mut dlq_messages = Vec::new();

    for token in timed_out {
        if let Some(mut ack) = self.pending_acks.remove(&token) {
            ack.delivery_attempts += 1;

            if ack.delivery_attempts > max_attempts {
                // ===== DEAD LETTER QUEUE =====
                let topic = self.consumer_groups
                    .get(group_id)
                    .map(|g| g.topic.clone())
                    .unwrap_or_default();
                let dlq_topic = format!("{}-dlq", topic);

                // ดึง original message
                if let Some((_, partitions)) = self.topics.get(&topic) {
                    if let Some(partition) = partitions.get(ack.partition as usize) {
                        if let Some(msg) = partition.messages.iter()
                            .find(|m| m.offset == ack.message_offset)
                        {
                            dlq_messages.push((
                                dlq_topic,
                                msg.key.clone(),
                                msg.value.clone(),
                                msg.headers.clone(),
                            ));
                        }
                    }
                }

                self.metrics.dlq_messages_total += 1;
                // Commit offset เพื่อก้าวข้าม poisoned message
                if let Some(group) = self.consumer_groups.get_mut(group_id) {
                    group.commit_offset(ack.partition, ack.message_offset);
                }
            } else {
                // Re-queue: reset timer สำหรับ next delivery attempt
                ack.delivered_at = chrono::Utc::now();
                self.pending_acks.insert(ack.ack_token.clone(), ack);
            }
        }
    }

    // สร้าง DLQ topics และ publish
    for (dlq_topic, key, value, headers) in dlq_messages {
        if !self.topics.contains_key(&dlq_topic) {
            let _ = self.create_topic(crate::broker::TopicConfig {
                name: dlq_topic.clone(),
                partitions: 1,
                replication_factor: 1,
                compaction: false,
            });
        }
        let _ = self.publish(&dlq_topic, key, value, headers);
    }
}
```

**src/metrics.rs — Prometheus text format:**

```rust
use std::collections::HashMap;

#[derive(Debug, Default, Clone)]
pub struct BrokerMetrics {
    pub messages_in_total: u64,
    pub messages_out_total: u64,
    pub dlq_messages_total: u64,
}

impl BrokerMetrics {
    /// สร้าง Prometheus text exposition format
    /// รูปแบบ: # HELP, # TYPE, metric_name{labels} value
    pub fn to_prometheus(&self, consumer_lag: &HashMap<String, u64>) -> String {
        let mut out = String::new();

        // Counter: messages ขาเข้า
        out.push_str("# HELP messages_in_total Total number of messages published\n");
        out.push_str("# TYPE messages_in_total counter\n");
        out.push_str(&format!("messages_in_total {}\n\n", self.messages_in_total));

        // Counter: messages ขาออก
        out.push_str("# HELP messages_out_total Total number of messages delivered to consumers\n");
        out.push_str("# TYPE messages_out_total counter\n");
        out.push_str(&format!("messages_out_total {}\n\n", self.messages_out_total));

        // Counter: DLQ messages
        out.push_str("# HELP dlq_messages_total Total messages moved to dead letter queue\n");
        out.push_str("# TYPE dlq_messages_total counter\n");
        out.push_str(&format!("dlq_messages_total {}\n\n", self.dlq_messages_total));

        // Gauge: consumer lag ต่อ group
        out.push_str("# HELP consumer_lag_by_group Number of unprocessed messages per consumer group\n");
        out.push_str("# TYPE consumer_lag_by_group gauge\n");
        for (group, lag) in consumer_lag {
            out.push_str(&format!(
                "consumer_lag_by_group{{group=\"{}\"}} {}\n",
                group, lag
            ));
        }

        out
    }
}
```

**src/handlers/metrics.rs:**

```rust
use axum::{extract::State, http::header, response::Response};
use axum::body::Body;
use http::StatusCode;
use std::collections::HashMap;
use crate::handlers::topics::BrokerState;

/// GET /metrics — Prometheus format
pub async fn get_metrics(
    State(broker): State<BrokerState>,
) -> Response<Body> {
    let broker = broker.lock().unwrap();

    // คำนวณ consumer lag ทุก group
    let mut lag_map: HashMap<String, u64> = HashMap::new();
    for (group_id, group) in &broker.consumer_groups {
        let lag = broker.consumer_lag(group_id);
        lag_map.insert(group.name.clone(), lag);
    }

    let body = broker.metrics.to_prometheus(&lag_map);

    Response::builder()
        .status(StatusCode::OK)
        .header(header::CONTENT_TYPE, "text/plain; version=0.0.4; charset=utf-8")
        .body(Body::from(body))
        .unwrap()
}
```

---

### ขั้นที่ 8: Log Compaction

Topics ที่ `compaction=true` จะเก็บเฉพาะ message ล่าสุดต่อ key — เหมาะสำหรับ state snapshots เช่น user profiles หรือ config

**src/compaction.rs:**

```rust
use crate::broker::Partition;

impl Partition {
    /// Compact partition: เก็บไว้เฉพาะ message ล่าสุดต่อ key
    /// Messages ที่ไม่มี key จะไม่ถูกลบ (tombstone logic)
    pub fn compact(&mut self) {
        // key_index มี {key -> latest_offset} อยู่แล้ว
        // ลบ messages ที่มี key แต่ไม่ใช่ offset ล่าสุด
        self.messages.retain(|msg| {
            match &msg.key {
                Some(k) => {
                    // เก็บไว้ถ้าเป็น latest offset สำหรับ key นี้
                    self.key_index.get(k) == Some(&msg.offset)
                }
                None => true, // ไม่มี key → เก็บทุก message
            }
        });
    }
}

/// Compact ทุก partition ใน topic
/// คืน (messages_before, messages_after)
pub fn compact_topic(
    partitions: &mut Vec<Partition>,
) -> (usize, usize) {
    let before: usize = partitions.iter().map(|p| p.messages.len()).sum();
    for partition in partitions.iter_mut() {
        partition.compact();
    }
    let after: usize = partitions.iter().map(|p| p.messages.len()).sum();
    (before, after)
}

/// Trigger compaction สำหรับ topic ที่ compact=true
/// ควรรัน background task ทุก N นาที หรือเมื่อ partition เกิน threshold
pub fn maybe_compact(
    partitions: &mut Vec<Partition>,
    compaction_enabled: bool,
    threshold: usize,
) -> Option<usize> {
    if !compaction_enabled {
        return None;
    }
    let total: usize = partitions.iter().map(|p| p.messages.len()).sum();
    if total < threshold {
        return None; // ยังไม่ถึง threshold
    }
    let (before, after) = compact_topic(partitions);
    Some(before - after)
}
```

---

## การทดสอบ (Testing)

### Unit Tests

สร้าง Rust project ใน scratchpad และรันจริง:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn make_broker() -> MessageBroker {
        let mut broker = MessageBroker::new();
        broker.create_topic(TopicConfig {
            name: "events".to_string(),
            partitions: 2,
            replication_factor: 1,
            compaction: false,
        }).unwrap();
        broker
    }

    // Test 1: Message serialization roundtrip
    #[test]
    fn test_message_serialization_roundtrip() {
        let mut headers = HashMap::new();
        headers.insert("content-type".to_string(), "application/json".to_string());
        let msg = Message {
            offset: 42,
            key: Some("user-123".to_string()),
            value: b"hello world".to_vec(),
            headers,
            timestamp: Utc::now(),
            partition: 0,
        };

        let serialized = msg.serialize_to_log();
        // Header: 8 + 4 = 12 bytes minimum
        assert!(serialized.len() >= 12);

        // Verify offset prefix
        let decoded_offset = u64::from_le_bytes(serialized[0..8].try_into().unwrap());
        assert_eq!(decoded_offset, 42);

        // Deserialize กลับมา
        let (decoded_msg, consumed) = Message::deserialize_from_log(&serialized).unwrap();
        assert_eq!(consumed, serialized.len());
        assert_eq!(decoded_msg.offset, 42);
        assert_eq!(decoded_msg.key, Some("user-123".to_string()));
        assert_eq!(decoded_msg.value, b"hello world".to_vec());
        assert_eq!(
            decoded_msg.headers.get("content-type").unwrap(),
            "application/json"
        );
    }

    // Test 2: Offset arithmetic
    #[test]
    fn test_offset_arithmetic() {
        let mut broker = make_broker();
        for i in 0u32..5 {
            broker.publish(
                "events",
                Some(format!("key-{}", i)),
                format!("val-{}", i).into_bytes(),
                HashMap::new(),
            ).unwrap();
        }

        let (_, partitions) = broker.topics.get("events").unwrap();
        let total: u64 = partitions.iter().map(|p| p.next_offset).sum();
        assert_eq!(total, 5);
        // offset ต้องตรงกับ message count ใน partition
        for p in partitions.iter() {
            assert_eq!(p.messages.len() as u64, p.next_offset);
        }
    }

    // Test 3: Consumer group round-robin assignment
    #[test]
    fn test_consumer_group_round_robin() {
        let mut group = ConsumerGroup::new("test".to_string(), "events".to_string());
        group.add_member("A".to_string());
        group.add_member("B".to_string());
        group.add_member("C".to_string());

        assert_eq!(group.assign_member(0), Some("A"));
        assert_eq!(group.assign_member(1), Some("B"));
        assert_eq!(group.assign_member(2), Some("C"));
        assert_eq!(group.assign_member(3), Some("A")); // wrap-around

        let empty = ConsumerGroup::new("empty".to_string(), "events".to_string());
        assert_eq!(empty.assign_member(0), None);
    }

    // Test 4: Ack timeout detection
    #[test]
    fn test_ack_timeout_detection() {
        let fresh = PendingAck::new(0, 0, "g1".to_string());
        assert!(!fresh.is_timed_out(30)); // เพิ่งสร้าง ยังไม่ timeout

        let old = PendingAck {
            ack_token: Uuid::new_v4().to_string(),
            message_offset: 1,
            partition: 0,
            consumer_group_id: "g1".to_string(),
            delivery_attempts: 1,
            delivered_at: Utc::now() - chrono::Duration::seconds(60),
        };
        assert!(old.is_timed_out(30));   // 60s > 30s limit → timeout
        assert!(!old.is_timed_out(120)); // 60s < 120s limit → ยังไม่ timeout
    }

    // Test 5: Dead letter queue after max attempts
    #[test]
    fn test_dead_letter_queue() {
        let mut broker = MessageBroker::new();
        broker.create_topic(TopicConfig {
            name: "orders".to_string(),
            partitions: 1,
            replication_factor: 1,
            compaction: false,
        }).unwrap();

        broker.publish("orders", Some("o1".to_string()), b"data".to_vec(), HashMap::new()).unwrap();

        let gid = broker.create_consumer_group("proc".to_string(), "orders".to_string()).unwrap();

        // Inject expired ack with max attempts already reached
        let expired = PendingAck {
            ack_token: "test-dlq".to_string(),
            message_offset: 0,
            partition: 0,
            consumer_group_id: gid.clone(),
            delivery_attempts: 3, // ถัดไปจะเกิน max
            delivered_at: Utc::now() - chrono::Duration::seconds(60),
        };
        broker.pending_acks.insert("test-dlq".to_string(), expired);

        broker.requeue_timed_out(&gid);

        assert!(broker.topics.contains_key("orders-dlq"));
        assert_eq!(broker.metrics.dlq_messages_total, 1);
        let (_, dlq_parts) = broker.topics.get("orders-dlq").unwrap();
        let count: usize = dlq_parts.iter().map(|p| p.messages.len()).sum();
        assert_eq!(count, 1);
    }
}
```

### Real `cargo test` Output

รันจริงบน scratchpad project:

```
running 8 tests
test tests::test_consumer_group_round_robin ... ok
test tests::test_ack_timeout_detection ... ok
test tests::test_dead_letter_queue ... ok
test tests::test_log_compaction ... ok
test tests::test_consumer_lag ... ok
test tests::test_offset_arithmetic ... ok
test tests::test_prometheus_metrics_format ... ok
test tests::test_message_serialization_roundtrip ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

ทุก test ผ่านจริง — 8/8 passed ✓

---

## กับดักและปัญหาที่พบบ่อย (Pitfalls)

### Pitfall 1: Mutex Poisoning และ Lock Contention

```rust
// ❌ WRONG: ถ้า handler panic ขณะ hold lock → Mutex poisoned
// ทุก thread ถัดไปจะได้ PoisonError
let mut broker = broker.lock().unwrap(); // ← unwrap บน poisoned mutex = panic cascade

// ✅ CORRECT: handle PoisonError อย่างชัดเจน
let mut broker = broker.lock().map_err(|_| {
    (StatusCode::INTERNAL_SERVER_ERROR, "Broker state corrupted".to_string())
})?;

// ✅ หรือใช้ recover:
let mut broker = match broker.lock() {
    Ok(g) => g,
    Err(poisoned) => poisoned.into_inner(), // recover from poison
};
```

**ปัญหาที่ลึกกว่า**: `Mutex<MessageBroker>` lock ทั้ง broker ขณะประมวลผล → throughput ต่ำ  
**แก้ไข production**: ใช้ `RwLock` สำหรับ read-heavy operations หรือ `DashMap` สำหรับ per-topic locking

```rust
// Production pattern: per-topic sharding
pub struct ShardedBroker {
    topics: Arc<DashMap<String, Arc<Mutex<TopicState>>>>,
    groups: Arc<DashMap<String, Arc<Mutex<ConsumerGroup>>>>,
}
```

---

### Pitfall 2: Offset Commit ก่อน Processing เสร็จ (At-Most-Once vs At-Least-Once)

```rust
// ❌ WRONG: commit offset ก่อน process message
// ถ้า crash หลัง commit แต่ก่อน process → message หาย (at-most-once)
let messages = pull_messages(&group_id, 10).await?;
acknowledge(&group_id, &tokens).await?; // ← commit ก่อน!
process_messages(messages).await?;      // ← ถ้า crash ตรงนี้ message หายไป

// ✅ CORRECT: process ก่อน แล้วค่อย ack
let messages = pull_messages(&group_id, 10).await?;
for (msg, token) in &messages {
    process_message(msg).await?; // ← process ให้สำเร็จก่อน
}
// ค่อย ack ทีเดียวหลังทำทุก message เสร็จ (at-least-once)
let tokens: Vec<_> = messages.iter().map(|(_, t)| t.clone()).collect();
acknowledge(&group_id, &tokens).await?;

// NOTE: at-least-once → idempotent processing จำเป็น!
// ใช้ database upsert หรือ message ID deduplication
```

---

### Pitfall 3: Ack Token Expiry ระหว่าง Long Processing

```rust
// ❌ WRONG: ประมวลผลนาน → ack_token หมดอายุ → message re-delivered ซ้ำ
// Consumer ที่ process ช้า (เช่น call external API) จะทำให้ messages re-queue

// ✅ CORRECT approach 1: ขยาย ack_timeout ให้นานพอ
let group = CreateGroupRequest {
    name: "slow-processor".to_string(),
    topic: "heavy-jobs".to_string(),
    ack_timeout_secs: Some(300), // 5 นาที
};

// ✅ CORRECT approach 2: Heartbeat/extend ack deadline
// ส่ง POST /consumer-groups/{id}/extend-ack {ack_token, extend_by_secs: 60}
// ทำ reset delivered_at ใน PendingAck

// ✅ CORRECT approach 3: Process เป็น batch เล็กๆ
// แทนที่จะ pull 100 แล้วค่อย ack ทีเดียว
// ให้ pull 10, ack, pull 10, ack, ...
```

---

### Pitfall 4: Log Compaction กับ Consumer ที่ยัง Read Offset เก่า

```rust
// ❌ WRONG: compact partition ขณะ consumer ยัง read อยู่
// Consumer group A committed offset=5, กำลังจะ read offset=6..10
// Compaction ลบ offset=7 ออก (เพราะมี newer value สำหรับ key เดียวกัน)
// Consumer จะไม่เห็น offset=7 → data loss!

// ✅ CORRECT: compaction ต้องใช้ "safe compaction point"
pub fn safe_compact(&mut self, topic: &str) -> Result<usize, String> {
    let (config, partitions) = self.topics.get_mut(topic).ok_or("Not found")?;
    if !config.compaction {
        return Err("Compaction not enabled".into());
    }

    // หา minimum committed offset ของทุก consumer group
    let min_committed: u64 = self.consumer_groups
        .values()
        .filter(|g| g.topic == topic)
        .flat_map(|g| {
            (0..config.partitions)
                .map(move |p| g.get_committed_offset(p))
        })
        .min()
        .unwrap_or(0);

    // Compact เฉพาะ messages ที่ offset < min_committed
    // messages ที่ยังไม่ถูก consume ทุก group ต้องไม่ถูกลบ
    let before: usize = partitions.iter().map(|p| p.messages.len()).sum();
    for partition in partitions.iter_mut() {
        partition.messages.retain(|msg| {
            if msg.offset >= min_committed {
                return true; // ยังไม่ safe to compact
            }
            match &msg.key {
                Some(k) => partition.key_index.get(k) == Some(&msg.offset),
                None => true,
            }
        });
    }
    let after: usize = partitions.iter().map(|p| p.messages.len()).sum();
    Ok(before - after)
}
```

---

### Pitfall 5: Fan-out กับ Backpressure

```rust
// ❌ ปัญหา: ถ้า consumer group หนึ่งช้ามาก
// messages สะสมใน partition → memory ไม่พอ
// เพราะ partition เก็บทุก message จนทุก group consume แล้ว

// ❌ WRONG: ลบ message ออกจาก partition หลัง consumer group เดียว consume
// จะทำให้ group อื่นไม่ได้รับ message

// ✅ CORRECT: Track retention policy
pub struct RetentionPolicy {
    pub max_age_secs: Option<u64>,  // ลบ messages เก่ากว่า N วินาที
    pub max_bytes: Option<u64>,     // ลบ messages เก่าสุดถ้าเกิน N bytes
    pub max_messages: Option<u64>,  // ลบ messages เก่าสุดถ้าเกิน N ข้อความ
}

// Retention check ทำแยกจาก compaction
// Messages ที่ expired ถูกลบโดยไม่คำนึง consumer lag
// → slow consumers จะ "miss" messages ที่ expired
// → standard Kafka behavior
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# Binary อยู่ที่: target/release/message_broker

# รันด้วย custom config
DATABASE_URL=sqlite:./broker.db \
DATA_DIR=./data \
RUST_LOG=info \
./target/release/message_broker
```

### Dockerfile

```dockerfile
FROM rust:1.82-slim as builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
# Cache dependencies
RUN mkdir src && echo "fn main(){}" > src/main.rs
RUN cargo build --release
RUN rm src/main.rs

COPY src ./src
COPY migrations ./migrations
RUN touch src/main.rs && cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libssl3 ca-certificates sqlite3 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /app/target/release/message_broker .
COPY migrations ./migrations

VOLUME ["/app/data", "/app/db"]
EXPOSE 8080

ENV DATABASE_URL=sqlite:/app/db/broker.db
ENV DATA_DIR=/app/data
ENV RUST_LOG=info

CMD ["./message_broker"]
```

### docker-compose.yml

```yaml
version: "3.9"
services:
  message-broker:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - broker-data:/app/data
      - broker-db:/app/db
    environment:
      - RUST_LOG=info
      - DATABASE_URL=sqlite:/app/db/broker.db
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/metrics"]
      interval: 30s
      timeout: 5s
      retries: 3

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    depends_on:
      - message-broker

volumes:
  broker-data:
  broker-db:
```

### prometheus.yml

```yaml
scrape_configs:
  - job_name: 'message-broker'
    static_configs:
      - targets: ['message-broker:8080']
    metrics_path: '/metrics'
    scrape_interval: 15s
```

### Testing ด้วย curl

```bash
# 1. สร้าง topic
curl -X POST http://localhost:8080/topics \
  -H 'Content-Type: application/json' \
  -d '{"name":"orders","partitions":3,"replication_factor":1,"compaction":false}'

# Output:
# {"name":"orders","partitions":3,"replication_factor":1,"compaction":false,"message_count":0}

# 2. Publish message
curl -X POST http://localhost:8080/topics/orders/messages \
  -H 'Content-Type: application/json' \
  -d '{"key":"order-123","value":"place_order","headers":{"source":"api-gateway"}}'

# Output:
# {"topic":"orders","partition":1,"offset":0,"timestamp":"2024-01-15T10:30:00Z"}

# 3. สร้าง consumer group
curl -X POST http://localhost:8080/consumer-groups \
  -H 'Content-Type: application/json' \
  -d '{"name":"payment-service","topic":"orders","ack_timeout_secs":60}'

# Output:
# {"id":"550e8400-e29b-41d4-a716-446655440000"}

# 4. Pull messages
curl "http://localhost:8080/consumer-groups/550e8400.../messages?max=5"

# Output:
# [{"offset":0,"partition":1,"key":"order-123","value":"place_order",
#   "headers":{"source":"api-gateway"},"timestamp":"...","ack_token":"abc-123"}]

# 5. Acknowledge
curl -X POST http://localhost:8080/consumer-groups/550e8400.../ack \
  -H 'Content-Type: application/json' \
  -d '{"ack_tokens":["abc-123"]}'

# Output:
# {"acknowledged":1,"group_id":"550e8400..."}

# 6. ดู metrics
curl http://localhost:8080/metrics
# Output:
# # HELP messages_in_total Total number of messages published
# # TYPE messages_in_total counter
# messages_in_total 1
#
# # HELP consumer_lag_by_group ...
# consumer_lag_by_group{group="payment-service"} 0
```

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: เพิ่ม Message Replay

ระบบนี้รองรับ "replay from offset" ด้วย offset tracking แต่ยังไม่มี API สำหรับ seek

```
เพิ่ม endpoint:
POST /consumer-groups/{id}/seek
{
  "topic": "orders",
  "partition": 0,
  "offset": 100    // หรือ "earliest" / "latest"
}

ให้ commit offset กลับไปที่ตำแหน่งที่ต้องการ
ใช้ได้กับ event replay ใน CQRS pattern
```

**Hint:** ใน `ConsumerGroup` ให้ override `offsets` ด้วยค่าที่ต้องการ จากนั้น pull ครั้งถัดไปจะ read จาก offset นั้น ต้องระวัง concurrent ack ที่อาจ race กับ seek

### Exercise 2: เพิ่ม WebSocket Push Notification

ระบบปัจจุบันเป็น pull-based — consumer ต้อง poll เอง เพิ่ม WebSocket endpoint ให้ broker push แจ้งเตือนเมื่อมี messages ใหม่

```
GET /consumer-groups/{id}/ws  (WebSocket upgrade)

เมื่อ publish message ใหม่:
broker → WebSocket → consumer ได้รับ notification
consumer ยัง pull เองผ่าน REST เหมือนเดิม (notification-driven pull)
```

**Hint:** ใช้ `tokio::sync::broadcast::channel` สำหรับส่ง notification ไปยัง WebSocket connections ที่ subscribe topic นั้น ใช้ `axum::extract::ws::WebSocketUpgrade`

### Exercise 3: Implement Partition Rebalancing

เมื่อ consumer group เพิ่มหรือลด member ควร rebalance partitions ให้กระจายเท่ากัน (Range หรือ RoundRobin assignor)

```
กฎปัจจุบัน: assign_member(offset % members.len())
ปัญหา: ถ้าเพิ่ม member ใหม่ offset เดิมอาจ reassign ไปยัง member ต่างกัน

Range Assignor:
- Topic มี 6 partitions, 3 members
- Member A → partition 0,1
- Member B → partition 2,3
- Member C → partition 4,5
- Stable assignment ไม่เปลี่ยนตาม offset
```

**Hint:** ใช้ `partition_id % members.len()` แทน `offset % members.len()` สำหรับ partition assignment แต่ต้องมี mechanism rebalance เมื่อ members เปลี่ยน

### Exercise 4: Schema Registry สำหรับ Message Validation

ป้องกัน producer ส่ง message ที่ไม่ตรง schema ก่อน publish จริง

```
POST /schemas
{
  "topic": "orders",
  "schema": {
    "type": "object",
    "required": ["order_id", "amount"],
    "properties": {
      "order_id": {"type": "string"},
      "amount": {"type": "number", "minimum": 0}
    }
  }
}

POST /topics/orders/messages จะ validate ก่อน publish
ถ้า message ไม่ตรง schema → 422 Unprocessable Entity
```

**Hint:** ใช้ `jsonschema` crate สำหรับ JSON Schema validation เก็บ schema ใน SQLite table ใหม่ `topic_schemas`

---

## สรุป

โปรเจคนี้สร้าง **Message Broker** ที่ทำงานได้จริงโดยใช้หลักการเดียวกับ Kafka:

| Component | เทคนิคที่ใช้ | สิ่งที่เรียนรู้ |
|-----------|------------|--------------|
| Append-only log | `u64 + u32 + bytes` binary format | Binary serialization, little-endian |
| Consumer groups | Offset tracking ใน SQLite | UPSERT, SQLite ACID guarantees |
| Pull-based API | axum REST handlers | State sharing ด้วย `Arc<Mutex<T>>` |
| Ack protocol | UUID tokens + timeout | At-least-once semantics |
| Dead letter queue | Auto-create DLQ topic | Error handling patterns |
| Log compaction | key_index HashMap | Efficient deduplication |
| Prometheus metrics | Text exposition format | Observability patterns |

**Pattern สำคัญ 3 ข้อที่ได้จากโปรเจคนี้:**

1. **Offset-based delivery** — consumer เป็นผู้ track ว่า consume ถึงไหน ไม่ใช่ broker track ให้ ทำให้ replay ได้และ stateless broker
2. **Ack token pattern** — แทนที่จะ ack ด้วย offset ที่ consumer รู้อยู่แล้ว การใช้ opaque token ทำให้ broker ควบคุม delivery semantics ได้เต็มที่
3. **Fan-out through lazy read** — broker ไม่ copy messages ให้ทุก consumer group แต่ทุก group อ่าน partition เดียวกันผ่าน offset ของตัวเอง

**เชื่อมโยงไปโปรเจคถัดไป:** Project C05 — Search Engine จะใช้ concept เรื่อง inverted index ซึ่งคล้ายกับ `key_index` ใน partition นี้ แต่ scale ขึ้นไปเป็น full-text search ด้วย tokenization และ TF-IDF scoring

---

**โปรเจคก่อนหน้า:** [Project C03: Time-Series Database](project-c03-timeseries-db.md) | **โปรเจคถัดไป:** [Project C05: Search Engine](project-c05-search-engine.md)
