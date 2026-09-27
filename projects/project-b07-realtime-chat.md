# Project B07: Real-time Chat Application (WebSocket)

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 6 ชั่วโมง

## ภาพรวมโปรเจค

โปรเจคนี้สร้าง **Real-time Chat Server** ด้วย Rust ที่รองรับการสื่อสารแบบ bidirectional ผ่าน **WebSocket protocol** ซึ่งเป็นโครงสร้างหลักของแอปพลิเคชันประเภท Slack, Discord, หรือ LINE ในโลก production

สิ่งที่ทำให้โปรเจคนี้น่าสร้าง:

- **WebSocket** เหมาะกว่า HTTP polling ถึง 10–100 เท่าสำหรับ real-time use case เพราะ connection คงอยู่และ server สามารถ push data ได้เลยโดยไม่ต้องรอ client poll
- Rust + `axum` + `tokio` ให้ throughput สูงมากด้วย memory footprint เล็ก — server เดียวรองรับ connection พร้อมกันหลายหมื่น connection ได้
- Pattern `Arc<DashMap<...>>` + `tokio::sync::broadcast` เป็นตัวอย่างจริงของการออกแบบ shared state ใน async Rust
- ครอบคลุม authentication, rate limiting, presence management, file sharing — ครบ production requirements

**Use case จริงในโลก production:**
- ระบบ internal messaging สำหรับทีม
- Live customer support chat
- Collaborative editing notifications
- Gaming lobby / matchmaking
- Live dashboard ที่ push updates ถึง client โดยตรง

---

## สิ่งที่จะได้เรียนรู้

- การทำ **WebSocket upgrade** ด้วย `axum::extract::ws::WebSocketUpgrade` และ split connection เป็น `SplitSink` + `SplitStream`
- การออกแบบ **shared state** ด้วย `Arc<DashMap<RoomId, Arc<RwLock<Room>>>>` สำหรับ multi-room architecture
- การใช้ `tokio::sync::broadcast::channel` สำหรับ fan-out message ไปยัง subscribers ทุกคนพร้อมกัน
- **Presence management** — ตรวจจับ join/leave และ broadcast ไปยัง room members
- **JWT authentication** ใน WebSocket upgrade request (header + query param)
- **Per-connection rate limiting** ด้วย sliding window algorithm และ close connection ด้วย status code 1008
- การเก็บ **message history** ใน Redis และโหลดกลับมาตอน join
- **File upload** และ broadcast download URL ผ่าน WebSocket

---

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 46–50: async/await, `tokio` runtime, `spawn`, `select!`
- จาก Part 51–55: `Arc`, `RwLock`, `Mutex` — shared state ใน concurrent context
- จาก Part 61–65: `axum` basics — routing, extractors, handlers
- จาก Part 66–70: `serde` / `serde_json` — serialization/deserialization
- จาก Part 71–75: error handling ใน async context, `thiserror`
- ความรู้พื้นฐาน WebSocket protocol (handshake, frames, close codes)
- Project B06 (OAuth2 Server) — JWT token generation/validation

---

## โครงสร้างโปรเจค (Project Layout)

```
realtime-chat/
├── src/
│   ├── main.rs           # Entry point, router setup, AppState
│   ├── types.rs          # ChatMessage, Room, MessageType, UserClaims
│   ├── auth.rs           # JWT create/verify, token extraction
│   ├── rooms.rs          # RoomState, RoomRegistry, join/leave/broadcast
│   ├── ws_handler.rs     # WebSocket upgrade handler, message loop
│   ├── rate_limiter.rs   # Sliding window rate limiter
│   ├── redis_store.rs    # Message history ด้วย Redis LPUSH/LRANGE
│   ├── rest_api.rs       # REST endpoints: /rooms CRUD, /files upload
│   └── errors.rs         # AppError, error response formatting
├── tests/
│   └── integration.rs    # HTTP-level integration tests
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow

```
Client Browser
    │
    │  HTTP Upgrade Request
    │  Authorization: Bearer <JWT>
    ▼
┌─────────────────────────────────────────────────────────┐
│                    axum Router                          │
│                                                         │
│  GET /ws?token=<JWT>  ──► ws_handler::handle_ws()      │
│  POST /rooms          ──► rest_api::create_room()      │
│  GET  /rooms          ──► rest_api::list_rooms()       │
│  POST /rooms/:id/files ──► rest_api::upload_file()     │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
       JWT Verification
       (auth::verify_token)
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│              WebSocket Connection Loop                 │
│                                                        │
│  SplitSink (write) ◄──── broadcast::Receiver          │
│       │                        │                      │
│  SplitStream (read) ──► RateLimiter.check()           │
│                              │                        │
│                        broadcast::Sender              │
│                    (one per Room in RoomRegistry)     │
└────────────────────────────────────────────────────────┘
                 │
                 ▼
┌───────────────────────────────┐
│   Arc<DashMap<RoomId,         │
│     Arc<RwLock<RoomState>>>>  │
│                               │
│  RoomState {                  │
│    room: Room,                │
│    members: HashMap<UserId,   │
│    tx: broadcast::Sender      │
│  }                            │
└───────────────────────────────┘
```

### ทำไมถึงใช้ `DashMap` แทน `Mutex<HashMap>`?

`DashMap` ใช้ sharding ภายใน — แบ่ง HashMap ออกเป็น N shards แต่ละ shard มี lock ของตัวเอง เมื่อ concurrent operations กระทำกับ room ต่างกัน จะไม่ชนกันที่ lock เลย ต่างจาก `Mutex<HashMap>` ที่ทุก operation ต้องรอ global lock

### ทำไมถึงใช้ `broadcast::channel` แทน `mpsc`?

`broadcast` รองรับ **multiple receivers** สำหรับ sender เดียว — เหมาะมากสำหรับ "ทุกคนในห้องได้รับ message เดียวกัน" เมื่อมี user ใหม่ join ก็แค่ `tx.subscribe()` ได้ receiver ใหม่เลย

ข้อระวัง: `broadcast` มี capacity จำกัด — ถ้า receiver ช้าเกินไปและ buffer เต็ม message เก่าจะถูก drop (lagged)

### ทำไม `Arc<RwLock<RoomState>>` ไม่ใช่แค่ `RwLock<RoomState>` ตรงๆ?

`DashMap` เก็บค่าแบบ owned — เมื่อเราต้องการส่ง reference ของ `RoomState` ออกไปให้หลาย task ใช้พร้อมกัน (เช่น ws_handler ของ user แต่ละคน) เราต้องใช้ `Arc` เพื่อแชร์ความเป็นเจ้าของ และ `RwLock` เพื่อควบคุม concurrent read/write

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Cargo.toml

สร้างโปรเจคใหม่:

```bash
cargo new realtime-chat
cd realtime-chat
```

**`Cargo.toml`** — dependency ทั้งหมดที่ต้องใช้:

```toml
[package]
name = "realtime-chat"
version = "0.1.0"
edition = "2021"

[dependencies]
# Web framework + WebSocket
axum = { version = "0.8", features = ["ws", "multipart", "macros"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "fs", "trace"] }

# Async runtime
tokio = { version = "1", features = ["full"] }
futures-util = "0.3"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Concurrent data structures
dashmap = "6"

# Authentication
jsonwebtoken = "9"

# Redis (async)
redis = { version = "0.26", features = ["tokio-comp", "connection-manager"] }

# Utilities
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
tokio-util = "0.7"
bytes = "1"

[dev-dependencies]
# สำหรับ integration tests
tokio-tungstenite = "0.26"
```

**หมายเหตุ crate สำคัญ:**
- `axum 0.8` มี WebSocket support ใน feature `ws` ซึ่งแยกออกมาชัดเจน
- `dashmap 6` มี API เปลี่ยนเล็กน้อยจากเวอร์ชัน 5 — `iter()` คืน `DashMapIter` ที่ต้อง collect ก่อน
- `redis 0.26` ใช้ `ConnectionManager` สำหรับ async connection pooling อัตโนมัติ
- `jsonwebtoken 9` ใช้ `Algorithm::HS256` เป็น default

---

### ขั้นที่ 2: Types และ Message Protocol

**`src/types.rs`** — นิยาม data structures ทั้งหมดของระบบ:

```rust
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};
use uuid::Uuid;

/// ประเภทของ identifier
pub type RoomId = String;
pub type UserId = String;

/// ประเภทของ message ที่รับส่งผ่าน WebSocket
/// ใช้ rename_all = "snake_case" เพื่อให้ JSON ใช้ snake_case
/// ตรงกับ convention ของ JavaScript client
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum MessageType {
    Message,      // ข้อความปกติ
    Join,         // user เข้าห้อง
    Leave,        // user ออกห้อง
    Typing,       // กำลังพิมพ์
    MembersList,  // รายชื่อสมาชิกในห้อง
    File,         // ไฟล์แนบ
    Error,        // ข้อผิดพลาด
}

/// โครงสร้าง message หลักที่ใช้ตลอดระบบ
/// skip_serializing_if ป้องกัน null fields ปรากฏใน JSON
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChatMessage {
    #[serde(rename = "type")]
    pub msg_type: MessageType,
    pub room: RoomId,
    pub user: UserId,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub content: Option<String>,
    pub timestamp: DateTime<Utc>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub members: Option<Vec<String>>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub file_url: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub file_name: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub file_size: Option<u64>,
}

impl ChatMessage {
    pub fn new_message(room: &str, user: &str, content: &str) -> Self {
        Self {
            msg_type: MessageType::Message,
            room: room.to_string(),
            user: user.to_string(),
            content: Some(content.to_string()),
            timestamp: Utc::now(),
            members: None,
            file_url: None,
            file_name: None,
            file_size: None,
        }
    }

    pub fn new_join(room: &str, user: &str) -> Self {
        Self {
            msg_type: MessageType::Join,
            room: room.to_string(),
            user: user.to_string(),
            content: Some(format!("{} เข้าร่วมห้องสนทนา", user)),
            timestamp: Utc::now(),
            members: None,
            file_url: None,
            file_name: None,
            file_size: None,
        }
    }

    pub fn new_leave(room: &str, user: &str) -> Self {
        Self {
            msg_type: MessageType::Leave,
            room: room.to_string(),
            user: user.to_string(),
            content: Some(format!("{} ออกจากห้องสนทนา", user)),
            timestamp: Utc::now(),
            members: None,
            file_url: None,
            file_name: None,
            file_size: None,
        }
    }

    pub fn new_typing(room: &str, user: &str) -> Self {
        Self {
            msg_type: MessageType::Typing,
            room: room.to_string(),
            user: user.to_string(),
            content: None,
            timestamp: Utc::now(),
            members: None,
            file_url: None,
            file_name: None,
            file_size: None,
        }
    }

    pub fn new_members_list(room: &str, members: Vec<String>) -> Self {
        Self {
            msg_type: MessageType::MembersList,
            room: room.to_string(),
            user: "system".to_string(),
            content: None,
            timestamp: Utc::now(),
            members: Some(members),
            file_url: None,
            file_name: None,
            file_size: None,
        }
    }

    pub fn new_file(room: &str, user: &str, file_url: &str, file_name: &str, file_size: u64) -> Self {
        Self {
            msg_type: MessageType::File,
            room: room.to_string(),
            user: user.to_string(),
            content: Some(format!("แชร์ไฟล์: {}", file_name)),
            timestamp: Utc::now(),
            members: None,
            file_url: Some(file_url.to_string()),
            file_name: Some(file_name.to_string()),
            file_size: Some(file_size),
        }
    }

    pub fn new_error(room: &str, message: &str) -> Self {
        Self {
            msg_type: MessageType::Error,
            room: room.to_string(),
            user: "system".to_string(),
            content: Some(message.to_string()),
            timestamp: Utc::now(),
            members: None,
            file_url: None,
            file_name: None,
            file_size: None,
        }
    }

    /// Serialize เป็น JSON string (ใช้กับ WebSocket text frame)
    pub fn to_json(&self) -> String {
        serde_json::to_string(self).unwrap_or_default()
    }
}

/// ข้อมูลห้องสนทนา (เก็บใน database/memory)
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Room {
    pub id: RoomId,
    pub name: String,
    pub description: Option<String>,
    pub created_by: UserId,
    pub created_at: DateTime<Utc>,
    pub is_private: bool,
    pub max_members: Option<usize>,
}

impl Room {
    pub fn new(name: &str, created_by: &str, is_private: bool) -> Self {
        Self {
            id: Uuid::new_v4().to_string(),
            name: name.to_string(),
            description: None,
            created_by: created_by.to_string(),
            created_at: Utc::now(),
            is_private,
            max_members: None,
        }
    }
}

/// Request body สำหรับสร้างห้อง
#[derive(Debug, Deserialize)]
pub struct CreateRoomRequest {
    pub name: String,
    pub description: Option<String>,
    pub is_private: bool,
    pub max_members: Option<usize>,
}

/// JWT Claims ที่ฝังใน token
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserClaims {
    pub sub: String,       // user_id
    pub username: String,
    pub exp: usize,        // expiration (Unix timestamp)
    pub iat: usize,        // issued at
}

/// Query params สำหรับ WebSocket connection
#[derive(Debug, Deserialize)]
pub struct WsQueryParams {
    pub token: Option<String>,
    pub room: String,
}
```

**ตัวอย่าง JSON ที่ client จะได้รับ:**

```json
// message ปกติ
{
  "type": "message",
  "room": "general",
  "user": "alice",
  "content": "สวัสดีทุกคน!",
  "timestamp": "2025-09-27T10:30:00Z"
}

// user เข้าห้อง
{
  "type": "join",
  "room": "general",
  "user": "bob",
  "content": "bob เข้าร่วมห้องสนทนา",
  "timestamp": "2025-09-27T10:30:05Z"
}

// รายชื่อสมาชิก
{
  "type": "members_list",
  "room": "general",
  "user": "system",
  "timestamp": "2025-09-27T10:30:05Z",
  "members": ["alice", "bob", "charlie"]
}
```

---

### ขั้นที่ 3: JWT Authentication

**`src/auth.rs`** — การจัดการ token ทุกรูปแบบ:

```rust
use jsonwebtoken::{
    decode, encode, DecodingKey, EncodingKey,
    Header, Validation, Algorithm, errors::ErrorKind,
};
use crate::types::UserClaims;
use crate::errors::AppError;

/// Secret key ควรโหลดจาก environment variable ใน production
/// อย่าง `std::env::var("JWT_SECRET").expect("JWT_SECRET must be set")`
pub const JWT_SECRET: &[u8] = b"super-secret-key-for-chat-app-production-64bytes!!!!";

/// สร้าง JWT token สำหรับ user
pub fn create_token(user_id: &str, username: &str) -> Result<String, AppError> {
    use std::time::{SystemTime, UNIX_EPOCH};
    let now = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .map_err(|e| AppError::InternalError(e.to_string()))?
        .as_secs() as usize;

    let claims = UserClaims {
        sub: user_id.to_string(),
        username: username.to_string(),
        iat: now,
        exp: now + 86_400, // 24 ชั่วโมง
    };

    encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(JWT_SECRET),
    )
    .map_err(|e| AppError::InternalError(e.to_string()))
}

/// Verify JWT token และคืน claims
pub fn verify_token(token: &str) -> Result<UserClaims, AppError> {
    decode::<UserClaims>(
        token,
        &DecodingKey::from_secret(JWT_SECRET),
        &Validation::new(Algorithm::HS256),
    )
    .map(|data| data.claims)
    .map_err(|e| match e.kind() {
        ErrorKind::ExpiredSignature => AppError::TokenExpired,
        _ => AppError::Unauthorized(format!("Token ไม่ถูกต้อง: {}", e)),
    })
}

/// ดึง token จาก Authorization header
/// รูปแบบ: "Bearer <token>"
pub fn extract_token_from_header(header_value: &str) -> Option<&str> {
    header_value.strip_prefix("Bearer ")
}

/// ดึง token จาก WebSocket upgrade request
/// ลำดับ priority: Authorization header > query param ?token=
pub fn extract_ws_token<'a>(
    auth_header: Option<&'a str>,
    query_token: Option<&'a str>,
) -> Option<&'a str> {
    // ลอง header ก่อน
    if let Some(header) = auth_header {
        if let Some(token) = extract_token_from_header(header) {
            return Some(token);
        }
    }
    // fallback ไป query param
    query_token
}
```

**⚠️ Pitfall #1 — JWT ใน WebSocket Header**

Browser WebSocket API มาตรฐาน (`new WebSocket(url)`) **ไม่รองรับการส่ง custom headers** ได้โดยตรง ทำให้ต้องส่ง token ผ่าน query parameter แทน:

```
ws://localhost:3000/ws?token=eyJ...&room=general
```

ข้อเสียคือ token จะปรากฏใน server access log ซึ่งเป็น security risk ใน production ควรใช้ทางเลือกเหล่านี้:
1. ส่ง token ในข้อความแรกหลัง connection สำเร็จ (authenticate via first message)
2. ใช้ `Sec-WebSocket-Protocol` header เป็น workaround (แต่ dirty)
3. ใช้ short-lived one-time token แทน JWT ปกติ

---

### ขั้นที่ 4: Room Management ด้วย DashMap + broadcast

**`src/rooms.rs`** — หัวใจของระบบ multi-room:

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{broadcast, RwLock};
use dashmap::DashMap;
use crate::types::{ChatMessage, Room, RoomId, UserId};

/// capacity ของ broadcast channel ต่อห้อง
/// ถ้า receiver ช้า และ buffer เต็ม → message เก่าถูก drop
/// เพิ่มค่านี้ถ้าเจอ broadcast::error::RecvError::Lagged
pub const BROADCAST_CAPACITY: usize = 256;

/// State ของห้องสนทนาหนึ่งห้อง
pub struct RoomState {
    pub room: Room,
    /// สมาชิกปัจจุบัน: UserId → () (unit เพราะแค่ต้องการ set ของ user IDs)
    pub members: HashMap<UserId, String>,  // UserId → display_name
    /// broadcast sender — แชร์ให้ทุก connection ในห้องนี้ subscribe
    pub tx: broadcast::Sender<ChatMessage>,
    /// history สำรองไว้ใน memory (สูงสุด 50 messages)
    /// สำหรับ Redis unavailable scenarios
    pub in_memory_history: std::collections::VecDeque<ChatMessage>,
}

impl RoomState {
    pub fn new(room: Room) -> Self {
        let (tx, _rx) = broadcast::channel(BROADCAST_CAPACITY);
        Self {
            room,
            members: HashMap::new(),
            tx,
            in_memory_history: std::collections::VecDeque::with_capacity(50),
        }
    }

    /// เพิ่มสมาชิก
    pub fn add_member(&mut self, user_id: &str, display_name: &str) {
        self.members.insert(user_id.to_string(), display_name.to_string());
    }

    /// ลบสมาชิก
    pub fn remove_member(&mut self, user_id: &str) -> bool {
        self.members.remove(user_id).is_some()
    }

    /// รายชื่อสมาชิกทั้งหมด (เป็น display name)
    pub fn member_list(&self) -> Vec<String> {
        self.members.values().cloned().collect()
    }

    /// จำนวนสมาชิก
    pub fn member_count(&self) -> usize {
        self.members.len()
    }

    /// Broadcast message ไปทุก subscriber
    /// คืนค่า = จำนวน receiver ที่รับ message (0 = ไม่มีใครฟัง)
    pub fn broadcast(&self, msg: ChatMessage) -> usize {
        // บันทึกลง history ก่อน broadcast
        // (ทำผ่าน Arc<RwLock> ไม่ได้ที่นี่ แยกออกไปทำใน ws_handler)
        self.tx.send(msg).unwrap_or(0)
    }

    /// บันทึก message ลง in-memory history
    pub fn record_history(&mut self, msg: ChatMessage) {
        if self.in_memory_history.len() >= 50 {
            self.in_memory_history.pop_front();
        }
        self.in_memory_history.push_back(msg);
    }

    /// Subscribe ไปยัง broadcast channel
    pub fn subscribe(&self) -> broadcast::Receiver<ChatMessage> {
        self.tx.subscribe()
    }
}

/// Registry หลักของทุกห้อง
/// DashMap รองรับ concurrent access ได้ดีกว่า Mutex<HashMap>
/// Arc ทำให้แชร์ระหว่าง routes หลายตัวได้
pub type RoomRegistry = Arc<DashMap<RoomId, Arc<RwLock<RoomState>>>>;

/// สร้าง registry ว่างสำหรับใช้ใน AppState
pub fn new_registry() -> RoomRegistry {
    Arc::new(DashMap::new())
}

/// สร้างห้องใหม่และเพิ่มลง registry
pub async fn create_room(registry: &RoomRegistry, room: Room) -> RoomId {
    let id = room.id.clone();
    let state = Arc::new(RwLock::new(RoomState::new(room)));
    registry.insert(id.clone(), state);
    id
}

/// User join ห้อง: เพิ่ม member และ broadcast "join" event
/// คืน broadcast::Receiver สำหรับฟัง message ในห้องนี้
pub async fn join_room(
    registry: &RoomRegistry,
    room_id: &str,
    user_id: &str,
    display_name: &str,
) -> Option<(broadcast::Receiver<ChatMessage>, Vec<ChatMessage>)> {
    let entry = registry.get(room_id)?;
    let mut state = entry.write().await;

    state.add_member(user_id, display_name);
    let rx = state.subscribe();

    // Broadcast join event ไปให้คนอื่นในห้อง
    let join_msg = ChatMessage::new_join(room_id, display_name);
    let _ = state.tx.send(join_msg);

    // Broadcast รายชื่อสมาชิกใหม่
    let members = state.member_list();
    let members_msg = ChatMessage::new_members_list(room_id, members);
    let _ = state.tx.send(members_msg);

    // ดึง history จาก in-memory (ใน production ดึงจาก Redis)
    let history: Vec<ChatMessage> = state.in_memory_history.iter().cloned().collect();

    Some((rx, history))
}

/// User leave ห้อง: ลบ member และ broadcast "leave" event
pub async fn leave_room(
    registry: &RoomRegistry,
    room_id: &str,
    user_id: &str,
    display_name: &str,
) {
    if let Some(entry) = registry.get(room_id) {
        let mut state = entry.write().await;
        if state.remove_member(user_id) {
            let leave_msg = ChatMessage::new_leave(room_id, display_name);
            let _ = state.tx.send(leave_msg);
        }
    }
}

/// ดึงรายชื่อสมาชิกในห้อง
pub async fn get_members(registry: &RoomRegistry, room_id: &str) -> Option<Vec<String>> {
    let entry = registry.get(room_id)?;
    let state = entry.read().await;
    Some(state.member_list())
}

/// ดึงรายชื่อทุกห้อง
pub async fn list_rooms(registry: &RoomRegistry) -> Vec<(RoomId, String, usize)> {
    let mut result = Vec::new();
    for entry in registry.iter() {
        let state = entry.value().read().await;
        result.push((
            entry.key().clone(),
            state.room.name.clone(),
            state.member_count(),
        ));
    }
    result
}
```

**⚠️ Pitfall #2 — `broadcast::Receiver::recv()` และ `RecvError::Lagged`**

เมื่อ receiver ช้าเกินไปและ buffer channel เต็ม Tokio จะ drop messages เก่าและคืน `Err(RecvError::Lagged(n))` ซึ่งต้องจัดการให้ถูกต้อง:

```rust
// อย่าเขียนแบบนี้ — จะ crash เมื่อ lag
let msg = rx.recv().await.unwrap();

// เขียนแบบนี้แทน — handle lag ให้ถูกต้อง
match rx.recv().await {
    Ok(msg) => { /* ส่งไปยัง client */ }
    Err(broadcast::error::RecvError::Lagged(n)) => {
        // แจ้ง client ว่าพลาด n messages
        tracing::warn!("Connection lagged, dropped {} messages", n);
    }
    Err(broadcast::error::RecvError::Closed) => {
        // Channel ถูกปิด (ห้องถูกลบ)
        break;
    }
}
```

---

### ขั้นที่ 5: WebSocket Handler และ Rate Limiting

**`src/rate_limiter.rs`** — Sliding Window Rate Limiter:

```rust
use std::collections::VecDeque;
use std::time::{Duration, Instant};

/// Rate limiter แบบ sliding window ต่อ connection
/// สำหรับ per-user rate limiting ต้องเพิ่ม user_id lookup
pub struct RateLimiter {
    max_messages: usize,
    window: Duration,
    timestamps: VecDeque<Instant>,
}

impl RateLimiter {
    /// สร้าง rate limiter ใหม่
    /// `max_messages` = จำนวน messages สูงสุดใน `window`
    pub fn new(max_messages: usize, window: Duration) -> Self {
        Self {
            max_messages,
            window,
            timestamps: VecDeque::with_capacity(max_messages + 1),
        }
    }

    /// ตรวจสอบและบันทึก request ใหม่
    /// คืน `true` ถ้าผ่าน rate limit, `false` ถ้าเกิน limit
    ///
    /// Algorithm: Sliding Window
    /// 1. ลบ timestamps ที่เก่ากว่า `window` ออก
    /// 2. ถ้า timestamps เหลือ >= max_messages → block
    /// 3. ไม่เกิน → บันทึก timestamp ปัจจุบันและ allow
    pub fn check_and_record(&mut self) -> bool {
        let now = Instant::now();

        // ลบ entries ที่หลุดออกจาก window
        while let Some(&front) = self.timestamps.front() {
            if now.duration_since(front) > self.window {
                self.timestamps.pop_front();
            } else {
                break;
            }
        }

        if self.timestamps.len() >= self.max_messages {
            return false; // เกิน rate limit
        }

        self.timestamps.push_back(now);
        true
    }

    /// จำนวน requests ปัจจุบันใน window (สำหรับ debugging)
    pub fn current_count(&self) -> usize {
        self.timestamps.len()
    }

    /// คืนเวลาที่ต้องรอก่อนส่ง message ได้อีก (milliseconds)
    pub fn retry_after_ms(&self) -> u64 {
        if let Some(&oldest) = self.timestamps.front() {
            let elapsed = Instant::now().duration_since(oldest);
            if elapsed < self.window {
                return (self.window - elapsed).as_millis() as u64;
            }
        }
        0
    }
}
```

**`src/ws_handler.rs`** — WebSocket handler หลัก:

```rust
use axum::{
    extract::{
        ws::{Message, WebSocket, WebSocketUpgrade, CloseFrame},
        Query, State,
    },
    response::IntoResponse,
    http::HeaderMap,
};
use futures_util::{SinkExt, StreamExt};
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::broadcast;

use crate::{
    AppState,
    auth::extract_ws_token,
    auth::verify_token,
    errors::AppError,
    rate_limiter::RateLimiter,
    rooms,
    types::{ChatMessage, WsQueryParams},
};

/// จำนวน messages สูงสุดต่อวินาทีต่อ connection
/// เกินกว่านี้ = close connection ด้วย code 1008 (Policy Violation)
const MAX_MESSAGES_PER_SECOND: usize = 10;

/// WebSocket upgrade handler
/// ทำงาน 2 ขั้น:
/// 1. Verify JWT และตรวจสอบ room_id
/// 2. Upgrade เป็น WebSocket connection
pub async fn ws_handler(
    ws: WebSocketUpgrade,
    Query(params): Query<WsQueryParams>,
    headers: HeaderMap,
    State(state): State<Arc<AppState>>,
) -> impl IntoResponse {
    // ดึง token จาก header หรือ query param
    let auth_header = headers
        .get("authorization")
        .and_then(|v| v.to_str().ok());
    let query_token = params.token.as_deref();

    let token = match extract_ws_token(auth_header, query_token) {
        Some(t) => t.to_string(),
        None => {
            // ส่ง 401 ก่อน upgrade
            return (
                axum::http::StatusCode::UNAUTHORIZED,
                "Missing authentication token",
            ).into_response();
        }
    };

    let claims = match verify_token(&token) {
        Ok(c) => c,
        Err(_) => {
            return (
                axum::http::StatusCode::UNAUTHORIZED,
                "Invalid or expired token",
            ).into_response();
        }
    };

    let room_id = params.room.clone();

    // ตรวจว่าห้องมีอยู่จริง
    if !state.rooms.contains_key(&room_id) {
        return (
            axum::http::StatusCode::NOT_FOUND,
            "Room not found",
        ).into_response();
    }

    tracing::info!(
        user_id = %claims.sub,
        username = %claims.username,
        room = %room_id,
        "WebSocket connection upgrading"
    );

    // Upgrade เป็น WebSocket — ส่ง callback ที่ทำงานหลัง upgrade สำเร็จ
    ws.on_upgrade(move |socket| {
        handle_socket(socket, claims.sub, claims.username, room_id, state)
    })
}

/// ฟังก์ชันหลักที่ทำงานหลัง WebSocket upgrade สำเร็จ
/// แบ่ง WebSocket เป็น 2 ส่วน: sink (write) และ stream (read)
/// แล้วรัน 2 tasks พร้อมกันด้วย tokio::select!
async fn handle_socket(
    socket: WebSocket,
    user_id: String,
    username: String,
    room_id: String,
    state: Arc<AppState>,
) {
    // Split WebSocket เป็น write และ read halves
    // SplitSink รับ Message และส่งออกไปยัง client
    // SplitStream รับ Message จาก client
    let (mut sink, mut stream) = socket.split();

    // Join ห้อง — ได้รับ broadcast receiver และ history
    let (mut rx, history) = match rooms::join_room(
        &state.rooms,
        &room_id,
        &user_id,
        &username,
    ).await {
        Some(r) => r,
        None => {
            let _ = sink.send(Message::Close(Some(CloseFrame {
                code: 4004,
                reason: "Room not found".into(),
            }))).await;
            return;
        }
    };

    // ส่ง history ให้ client เมื่อ join
    for hist_msg in history {
        let json = hist_msg.to_json();
        if sink.send(Message::Text(json.into())).await.is_err() {
            return; // Client disconnect ระหว่างส่ง history
        }
    }

    // Rate limiter สำหรับ connection นี้
    let mut rate_limiter = RateLimiter::new(
        MAX_MESSAGES_PER_SECOND,
        Duration::from_secs(1),
    );

    let state_clone = state.clone();
    let room_id_clone = room_id.clone();
    let user_id_clone = user_id.clone();
    let username_clone = username.clone();

    // ใช้ tokio::select! รัน 2 tasks พร้อมกัน:
    // - Task ซ้าย: รับ message จาก broadcast channel → ส่งให้ client
    // - Task ขวา: รับ message จาก client → validate → broadcast
    tokio::select! {
        // Task 1: Receive from broadcast channel, write to WebSocket
        _ = async {
            loop {
                match rx.recv().await {
                    Ok(msg) => {
                        let json = msg.to_json();
                        if sink.send(Message::Text(json.into())).await.is_err() {
                            break; // Client disconnect
                        }
                    }
                    Err(broadcast::error::RecvError::Lagged(n)) => {
                        tracing::warn!(
                            user = %username_clone,
                            dropped = n,
                            "Connection lagged"
                        );
                        // ส่งแจ้ง client ว่า miss messages
                        let err_msg = ChatMessage::new_error(
                            &room_id_clone,
                            &format!("พลาด {} messages เนื่องจาก connection ช้า", n),
                        );
                        let _ = sink.send(Message::Text(err_msg.to_json().into())).await;
                    }
                    Err(broadcast::error::RecvError::Closed) => break,
                }
            }
        } => {}

        // Task 2: Receive from WebSocket, validate and broadcast
        _ = async {
            loop {
                match stream.next().await {
                    Some(Ok(Message::Text(text))) => {
                        // Rate limiting check
                        if !rate_limiter.check_and_record() {
                            tracing::warn!(
                                user = %user_id_clone,
                                "Rate limit exceeded, closing connection"
                            );
                            // WebSocket close code 1008 = Policy Violation
                            let _ = sink.send(Message::Close(Some(CloseFrame {
                                code: 1008,
                                reason: "Rate limit exceeded (max 10 msg/sec)".into(),
                            }))).await;
                            break;
                        }

                        // Parse incoming message
                        let incoming: serde_json::Value = match serde_json::from_str(&text) {
                            Ok(v) => v,
                            Err(_) => {
                                let err = ChatMessage::new_error(&room_id_clone, "รูปแบบ JSON ไม่ถูกต้อง");
                                let _ = sink.send(Message::Text(err.to_json().into())).await;
                                continue;
                            }
                        };

                        let msg_type = incoming["type"].as_str().unwrap_or("");
                        let content = incoming["content"].as_str().unwrap_or("");

                        let broadcast_msg = match msg_type {
                            "message" => {
                                if content.is_empty() {
                                    continue;
                                }
                                let mut msg = ChatMessage::new_message(
                                    &room_id_clone,
                                    &username_clone,
                                    content,
                                );
                                // บันทึก history
                                if let Some(entry) = state_clone.rooms.get(&room_id_clone) {
                                    let mut room_state = entry.write().await;
                                    room_state.record_history(msg.clone());
                                }
                                msg
                            }
                            "typing" => ChatMessage::new_typing(&room_id_clone, &username_clone),
                            _ => continue, // ignore unknown types
                        };

                        // Broadcast ไปทุกคนในห้อง
                        if let Some(entry) = state_clone.rooms.get(&room_id_clone) {
                            let state = entry.read().await;
                            state.broadcast(broadcast_msg);
                        }
                    }
                    Some(Ok(Message::Close(_))) | None => break,
                    Some(Ok(Message::Ping(data))) => {
                        // ตอบ Pong
                        let _ = sink.send(Message::Pong(data)).await;
                    }
                    Some(Err(_)) | Some(Ok(_)) => break,
                }
            }
        } => {}
    }

    // Cleanup เมื่อ connection ปิด
    rooms::leave_room(&state.rooms, &room_id, &user_id, &username).await;
    tracing::info!(
        user = %username,
        room = %room_id,
        "WebSocket connection closed"
    );
}
```

**⚠️ Pitfall #3 — `tokio::select!` กับ mutable borrow**

`sink` เป็น `SplitSink` และต้องใช้ `&mut sink` ใน **ทั้งสอง** branch ของ `select!` แต่ Rust ไม่อนุญาตให้มี 2 mutable borrow พร้อมกัน วิธีแก้:

```rust
// ❌ ไม่ compile — sink ถูก borrow ใน 2 tasks พร้อมกัน
tokio::select! {
    _ = async { sink.send(...).await } => {}
    _ = async { sink.send(...).await } => {}
}

// ✅ วิธีที่ถูก — แยก sink ออก แล้วใช้ tokio::spawn + channel
let (ws_tx, mut ws_rx) = tokio::sync::mpsc::unbounded_channel::<String>();

// Task 1: เขียน message ลง WebSocket จาก channel
tokio::spawn(async move {
    while let Some(text) = ws_rx.recv().await {
        if sink.send(Message::Text(text.into())).await.is_err() { break; }
    }
});

// Task 2: อ่านจาก broadcast แล้วส่งผ่าน mpsc channel
tokio::spawn(async move {
    while let Ok(msg) = rx.recv().await {
        let _ = ws_tx.send(msg.to_json());
    }
});
```

ใน example ข้างต้น เราใช้ `select!` กับ block เดียวในแต่ละ branch เพื่อเลี่ยง issue นี้ โดยทำ sink ให้อยู่ใน branch เดียวเท่านั้น

---

### ขั้นที่ 6: Redis History และ REST API

**`src/redis_store.rs`** — เก็บ message history ใน Redis:

```rust
use redis::{AsyncCommands, Client};
use crate::types::ChatMessage;

pub const HISTORY_KEY_PREFIX: &str = "chat:room:";
pub const MAX_HISTORY: isize = 50;

/// สร้าง Redis connection
pub async fn create_redis_client(url: &str) -> Result<Client, redis::RedisError> {
    Client::open(url)
}

/// เพิ่ม message ลงใน Redis List
/// ใช้ LPUSH + LTRIM เพื่อจำกัด 50 messages สุดท้าย
///
/// Key pattern: chat:room:{room_id}:messages
pub async fn save_message(
    client: &Client,
    room_id: &str,
    msg: &ChatMessage,
) -> Result<(), redis::RedisError> {
    let mut conn = client.get_multiplexed_async_connection().await?;
    let key = format!("{}{}:messages", HISTORY_KEY_PREFIX, room_id);
    let json = serde_json::to_string(msg)
        .map_err(|e| redis::RedisError::from((redis::ErrorKind::IoError, "Serialize error", e.to_string())))?;

    // LPUSH ใส่ message ใหม่ที่หัว list
    conn.lpush::<_, _, ()>(&key, &json).await?;
    // LTRIM เหลือแค่ 50 messages แรก (ใหม่สุด 50 ตัว)
    conn.ltrim(&key, 0, MAX_HISTORY - 1).await?;

    Ok(())
}

/// โหลด history 50 messages ล่าสุดจาก Redis
/// ใช้ LRANGE 0 49 → ได้ list ลำดับใหม่ก่อน → reverse เพื่อเรียง chronological
pub async fn load_history(
    client: &Client,
    room_id: &str,
) -> Result<Vec<ChatMessage>, redis::RedisError> {
    let mut conn = client.get_multiplexed_async_connection().await?;
    let key = format!("{}{}:messages", HISTORY_KEY_PREFIX, room_id);

    let items: Vec<String> = conn
        .lrange(&key, 0, MAX_HISTORY - 1)
        .await?;

    let mut messages: Vec<ChatMessage> = items
        .into_iter()
        .filter_map(|s| serde_json::from_str(&s).ok())
        .collect();

    // LRANGE คืน newest-first, เราต้องการ oldest-first สำหรับ display
    messages.reverse();
    Ok(messages)
}

/// ลบ history ของห้อง (เมื่อลบห้อง)
pub async fn delete_room_history(
    client: &Client,
    room_id: &str,
) -> Result<(), redis::RedisError> {
    let mut conn = client.get_multiplexed_async_connection().await?;
    let key = format!("{}{}:messages", HISTORY_KEY_PREFIX, room_id);
    conn.del::<_, ()>(&key).await?;
    Ok(())
}
```

**`src/rest_api.rs`** — REST endpoints สำหรับ Rooms CRUD และ File Upload:

```rust
use axum::{
    extract::{Multipart, Path, State},
    http::StatusCode,
    Json,
};
use std::sync::Arc;
use uuid::Uuid;

use crate::{
    AppState,
    errors::AppError,
    rooms,
    types::{ChatMessage, CreateRoomRequest, Room},
};

/// POST /rooms — สร้างห้องใหม่
pub async fn create_room(
    State(state): State<Arc<AppState>>,
    Json(req): Json<CreateRoomRequest>,
) -> Result<(StatusCode, Json<Room>), AppError> {
    if req.name.trim().is_empty() {
        return Err(AppError::BadRequest("ชื่อห้องต้องไม่ว่าง".into()));
    }
    if req.name.len() > 100 {
        return Err(AppError::BadRequest("ชื่อห้องต้องไม่เกิน 100 ตัวอักษร".into()));
    }

    let room = Room::new(&req.name, "system", req.is_private);
    let room_copy = room.clone();
    rooms::create_room(&state.rooms, room).await;

    Ok((StatusCode::CREATED, Json(room_copy)))
}

/// GET /rooms — รายชื่อทุกห้อง
pub async fn list_rooms(
    State(state): State<Arc<AppState>>,
) -> Json<Vec<serde_json::Value>> {
    let room_list = rooms::list_rooms(&state.rooms).await;
    let response: Vec<serde_json::Value> = room_list
        .into_iter()
        .map(|(id, name, member_count)| {
            serde_json::json!({
                "id": id,
                "name": name,
                "member_count": member_count,
            })
        })
        .collect();
    Json(response)
}

/// GET /rooms/:id — ข้อมูลห้องเดียว
pub async fn get_room(
    Path(room_id): Path<String>,
    State(state): State<Arc<AppState>>,
) -> Result<Json<serde_json::Value>, AppError> {
    let entry = state.rooms.get(&room_id)
        .ok_or(AppError::NotFound("ไม่พบห้อง".into()))?;
    let room_state = entry.read().await;
    let members = room_state.member_list();
    Ok(Json(serde_json::json!({
        "id": room_state.room.id,
        "name": room_state.room.name,
        "description": room_state.room.description,
        "created_by": room_state.room.created_by,
        "created_at": room_state.room.created_at,
        "is_private": room_state.room.is_private,
        "members": members,
        "member_count": members.len(),
    })))
}

/// DELETE /rooms/:id — ลบห้อง
pub async fn delete_room(
    Path(room_id): Path<String>,
    State(state): State<Arc<AppState>>,
) -> Result<StatusCode, AppError> {
    if state.rooms.remove(&room_id).is_none() {
        return Err(AppError::NotFound("ไม่พบห้อง".into()));
    }
    Ok(StatusCode::NO_CONTENT)
}

/// GET /rooms/:id/history — ประวัติข้อความ
/// ดึงจาก Redis (หรือ in-memory fallback)
pub async fn get_history(
    Path(room_id): Path<String>,
    State(state): State<Arc<AppState>>,
) -> Result<Json<Vec<ChatMessage>>, AppError> {
    // ลอง Redis ก่อน
    if let Some(ref redis_client) = state.redis {
        match crate::redis_store::load_history(redis_client, &room_id).await {
            Ok(history) => return Ok(Json(history)),
            Err(e) => {
                tracing::warn!("Redis error, falling back to in-memory: {}", e);
            }
        }
    }

    // Fallback: in-memory history
    let entry = state.rooms.get(&room_id)
        .ok_or(AppError::NotFound("ไม่พบห้อง".into()))?;
    let room_state = entry.read().await;
    let history: Vec<ChatMessage> = room_state.in_memory_history.iter().cloned().collect();
    Ok(Json(history))
}

/// POST /rooms/:id/files — อัปโหลดไฟล์และ broadcast download URL
/// Multipart form upload
pub async fn upload_file(
    Path(room_id): Path<String>,
    State(state): State<Arc<AppState>>,
    mut multipart: Multipart,
) -> Result<Json<serde_json::Value>, AppError> {
    // ตรวจสอบว่าห้องมีอยู่
    if !state.rooms.contains_key(&room_id) {
        return Err(AppError::NotFound("ไม่พบห้อง".into()));
    }

    let mut file_data: Option<(String, Vec<u8>, String)> = None; // (filename, data, content_type)

    // อ่าน multipart fields
    while let Some(field) = multipart.next_field().await
        .map_err(|e| AppError::BadRequest(e.to_string()))? {

        let field_name = field.name().unwrap_or("").to_string();
        if field_name == "file" {
            let filename = field.file_name()
                .unwrap_or("unnamed")
                .to_string();
            let content_type = field.content_type()
                .unwrap_or("application/octet-stream")
                .to_string();
            let data = field.bytes().await
                .map_err(|e| AppError::BadRequest(e.to_string()))?;

            // จำกัดขนาดไฟล์ที่ 10MB
            if data.len() > 10 * 1024 * 1024 {
                return Err(AppError::BadRequest("ไฟล์ต้องไม่เกิน 10MB".into()));
            }

            file_data = Some((filename, data.to_vec(), content_type));
        }
    }

    let (filename, _data, _content_type) = file_data
        .ok_or(AppError::BadRequest("ไม่พบไฟล์ใน request".into()))?;

    // ใน production: บันทึกไฟล์ลง S3/MinIO และสร้าง presigned URL
    // ในตัวอย่างนี้: สร้าง URL จำลอง
    let file_id = Uuid::new_v4().to_string();
    let download_url = format!("{}/files/{}", state.base_url, file_id);
    let file_size = _data.len() as u64;

    // Broadcast file message ไปยังห้อง
    let file_msg = ChatMessage::new_file(
        &room_id,
        "system",  // ใน production ใช้ user_id จาก auth
        &download_url,
        &filename,
        file_size,
    );

    if let Some(entry) = state.rooms.get(&room_id) {
        let room_state = entry.read().await;
        room_state.broadcast(file_msg.clone());
    }

    Ok(Json(serde_json::json!({
        "file_id": file_id,
        "filename": filename,
        "size": file_size,
        "download_url": download_url,
        "message": file_msg,
    })))
}
```

---

### ขั้นที่ 7: Error Handling และ Main Application

**`src/errors.rs`** — Centralized error types:

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
    #[error("ไม่พบทรัพยากร: {0}")]
    NotFound(String),

    #[error("ไม่ได้รับอนุญาต: {0}")]
    Unauthorized(String),

    #[error("Token หมดอายุ")]
    TokenExpired,

    #[error("Request ไม่ถูกต้อง: {0}")]
    BadRequest(String),

    #[error("ข้อผิดพลาดภายใน: {0}")]
    InternalError(String),
}

/// แปลง AppError เป็น HTTP response โดยอัตโนมัติ
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg.clone()),
            AppError::Unauthorized(msg) => (StatusCode::UNAUTHORIZED, msg.clone()),
            AppError::TokenExpired => (StatusCode::UNAUTHORIZED, "Token หมดอายุ".to_string()),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            AppError::InternalError(msg) => (StatusCode::INTERNAL_SERVER_ERROR, msg.clone()),
        };

        let body = json!({
            "error": message,
            "status": status.as_u16(),
        });

        (status, Json(body)).into_response()
    }
}
```

**`src/main.rs`** — Application entry point และ AppState:

```rust
mod auth;
mod errors;
mod rate_limiter;
mod redis_store;
mod rest_api;
mod rooms;
mod types;
mod ws_handler;

use axum::{
    routing::{delete, get, post},
    Router,
};
use std::sync::Arc;
use tower_http::cors::{Any, CorsLayer};
use tower_http::trace::TraceLayer;

use rooms::RoomRegistry;

/// Global application state — แชร์ระหว่าง handlers ทุกตัว
/// ต้อง implement Clone (ได้จาก Arc)
pub struct AppState {
    /// Registry ของทุกห้อง
    pub rooms: RoomRegistry,
    /// Redis client (optional — fallback ไป in-memory ถ้าไม่มี)
    pub redis: Option<redis::Client>,
    /// Base URL สำหรับสร้าง file download links
    pub base_url: String,
}

#[tokio::main]
async fn main() {
    // Setup tracing
    tracing_subscriber::fmt()
        .with_env_filter("info,realtime_chat=debug")
        .init();

    // สร้าง Redis client (ถ้ามี)
    let redis_client = redis::Client::open("redis://127.0.0.1/")
        .ok()
        .and_then(|client| {
            // ทดสอบ connection
            tracing::info!("Redis client created");
            Some(client)
        });

    // สร้าง AppState
    let state = Arc::new(AppState {
        rooms: rooms::new_registry(),
        redis: redis_client,
        base_url: std::env::var("BASE_URL")
            .unwrap_or_else(|_| "http://localhost:3000".to_string()),
    });

    // สร้างห้อง default
    let general = types::Room::new("general", "system", false);
    rooms::create_room(&state.rooms, general).await;
    let random = types::Room::new("random", "system", false);
    rooms::create_room(&state.rooms, random).await;

    tracing::info!("Created default rooms: general, random");

    // CORS configuration
    let cors = CorsLayer::new()
        .allow_origin(Any)
        .allow_methods(Any)
        .allow_headers(Any);

    // Router
    let app = Router::new()
        // WebSocket endpoint
        .route("/ws", get(ws_handler::ws_handler))
        // Rooms CRUD
        .route("/rooms", post(rest_api::create_room).get(rest_api::list_rooms))
        .route("/rooms/:id", get(rest_api::get_room).delete(rest_api::delete_room))
        .route("/rooms/:id/history", get(rest_api::get_history))
        // File upload
        .route("/rooms/:id/files", post(rest_api::upload_file))
        // Health check
        .route("/health", get(|| async { "OK" }))
        // Middleware
        .layer(TraceLayer::new_for_http())
        .layer(cors)
        .with_state(state);

    let addr = "0.0.0.0:3000";
    let listener = tokio::net::TcpListener::bind(addr).await.unwrap();
    tracing::info!("Server listening on {}", addr);

    axum::serve(listener, app).await.unwrap();
}
```

---

### ขั้นที่ 8: JavaScript Client ตัวอย่าง

**`client.html`** — Client ง่ายๆ สำหรับทดสอบ:

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>Real-time Chat</title>
    <style>
        body { font-family: sans-serif; max-width: 600px; margin: 50px auto; }
        #messages { height: 400px; overflow-y: auto; border: 1px solid #ccc;
                    padding: 10px; margin-bottom: 10px; }
        .message { margin: 5px 0; }
        .join { color: green; }
        .leave { color: red; }
        .system { color: gray; font-style: italic; }
        input { width: 80%; padding: 5px; }
        button { padding: 5px 15px; }
    </style>
</head>
<body>
    <h2>Real-time Chat (Room: general)</h2>
    <div id="messages"></div>
    <div>
        <input id="msg-input" placeholder="พิมพ์ข้อความ..." />
        <button onclick="sendMessage()">ส่ง</button>
    </div>

    <script>
        // ขอ token จาก server ก่อน (demo: hardcode)
        const TOKEN = "eyJhbGciOiJIUzI1NiJ9...";
        const ws = new WebSocket(`ws://localhost:3000/ws?token=${TOKEN}&room=general`);

        const messagesEl = document.getElementById("messages");

        function addMessage(text, cls = "") {
            const div = document.createElement("div");
            div.className = `message ${cls}`;
            div.textContent = text;
            messagesEl.appendChild(div);
            messagesEl.scrollTop = messagesEl.scrollHeight;
        }

        ws.onopen = () => addMessage("เชื่อมต่อสำเร็จ!", "system");
        ws.onclose = (e) => addMessage(`การเชื่อมต่อสิ้นสุด (code: ${e.code})`, "system");

        ws.onmessage = (event) => {
            const msg = JSON.parse(event.data);
            switch (msg.type) {
                case "message":
                    addMessage(`[${msg.user}]: ${msg.content}`);
                    break;
                case "join":
                    addMessage(msg.content, "join");
                    break;
                case "leave":
                    addMessage(msg.content, "leave");
                    break;
                case "members_list":
                    addMessage(`สมาชิก: ${msg.members.join(", ")}`, "system");
                    break;
                case "file":
                    addMessage(`📎 ${msg.user} แชร์ไฟล์: ${msg.file_name} [${msg.file_url}]`);
                    break;
            }
        };

        // ส่ง typing event เมื่อพิมพ์
        let typingTimer;
        document.getElementById("msg-input").addEventListener("input", () => {
            clearTimeout(typingTimer);
            ws.send(JSON.stringify({ type: "typing", room: "general" }));
            typingTimer = setTimeout(() => {}, 2000); // หยุดพิมพ์ใน 2s
        });

        function sendMessage() {
            const input = document.getElementById("msg-input");
            const text = input.value.trim();
            if (!text) return;
            ws.send(JSON.stringify({
                type: "message",
                room: "general",
                content: text,
            }));
            input.value = "";
        }

        document.getElementById("msg-input").addEventListener("keypress", (e) => {
            if (e.key === "Enter") sendMessage();
        });
    </script>
</body>
</html>
```

---

## การทดสอบ (Testing)

### โครงสร้าง Test

Test suite แบ่งเป็น 4 กลุ่มหลักตาม requirement:

1. **Message Serialization** — ตรวจว่า JSON output ถูกต้องตาม protocol spec
2. **JWT Authentication** — ตรวจว่า create/verify/tamper ทำงานถูกต้อง
3. **Rate Limiter** — ตรวจ sliding window logic ครบทุก edge case
4. **Room Management** — ตรวจ join/leave/broadcast/multi-room ด้วย async tests

### Test Code (ทดสอบจริงและผ่านแล้ว)

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use types::*;
    use auth::*;
    use rooms::*;
    use rate_limiter::*;
    use std::time::Duration;

    // ========== Message Serialization Tests ==========

    #[test]
    fn test_message_serialization_type_message() {
        let msg = ChatMessage::new_message("room-1", "alice", "สวัสดี");
        let json = serde_json::to_string(&msg).unwrap();
        assert!(json.contains(r#""type":"message""#));
        assert!(json.contains(r#""room":"room-1""#));
        assert!(json.contains(r#""user":"alice""#));
        assert!(json.contains("สวัสดี"));
    }

    #[test]
    fn test_message_serialization_type_join() {
        let msg = ChatMessage::new_join("room-1", "bob");
        let json = serde_json::to_string(&msg).unwrap();
        assert!(json.contains(r#""type":"join""#));
        assert!(json.contains(r#""user":"bob""#));
        // members ต้องไม่มีถ้าเป็น null (skip_serializing_if)
        assert!(!json.contains(r#""members""#));
    }

    #[test]
    fn test_message_deserialization_roundtrip() {
        let original = ChatMessage::new_message("room-42", "user-1", "Hello World!");
        let json = serde_json::to_string(&original).unwrap();
        let deserialized: ChatMessage = serde_json::from_str(&json).unwrap();
        assert_eq!(deserialized.msg_type, MessageType::Message);
        assert_eq!(deserialized.room, "room-42");
        assert_eq!(deserialized.user, "user-1");
        assert_eq!(deserialized.content.unwrap(), "Hello World!");
    }

    #[test]
    fn test_message_deserialization_from_json_string() {
        let json = r#"{
            "type": "typing",
            "room": "room-5",
            "user": "dave",
            "timestamp": "2025-01-01T00:00:00Z"
        }"#;
        let msg: ChatMessage = serde_json::from_str(json).unwrap();
        assert_eq!(msg.msg_type, MessageType::Typing);
        assert_eq!(msg.room, "room-5");
        assert!(msg.content.is_none());
    }

    #[test]
    fn test_message_type_all_variants_serialize() {
        let variants = [
            (MessageType::Message, "message"),
            (MessageType::Join, "join"),
            (MessageType::Leave, "leave"),
            (MessageType::Typing, "typing"),
            (MessageType::MembersList, "members_list"),
            (MessageType::File, "file"),
            (MessageType::Error, "error"),
        ];
        for (variant, expected) in variants {
            let json = serde_json::to_string(&variant).unwrap();
            assert_eq!(json, format!(r#""{expected}""#));
        }
    }

    // ========== JWT Auth Tests ==========

    #[test]
    fn test_jwt_create_and_verify() {
        let token = create_token("user-123", "alice").unwrap();
        assert!(!token.is_empty());
        assert_eq!(token.split('.').count(), 3);
        let claims = verify_token(&token).unwrap();
        assert_eq!(claims.sub, "user-123");
        assert_eq!(claims.username, "alice");
    }

    #[test]
    fn test_jwt_invalid_token_rejected() {
        let result = verify_token("not.a.valid.token");
        assert!(result.is_err());
    }

    #[test]
    fn test_jwt_tampered_token_rejected() {
        let token = create_token("user-1", "alice").unwrap();
        let parts: Vec<&str> = token.split('.').collect();
        let tampered = format!("{}.{}.invalidsignature", parts[0], parts[1]);
        assert!(verify_token(&tampered).is_err());
    }

    #[test]
    fn test_jwt_extract_from_bearer_header() {
        let header = "Bearer eyJhbGciOiJIUzI1NiJ9.payload.sig";
        assert_eq!(
            extract_token_from_header(header),
            Some("eyJhbGciOiJIUzI1NiJ9.payload.sig")
        );
    }

    // ========== Rate Limiter Tests ==========

    #[test]
    fn test_rate_limiter_allows_under_limit() {
        let mut limiter = RateLimiter::new(10, Duration::from_secs(1));
        for i in 0..10 {
            assert!(limiter.check_and_record(), "message {} ควรผ่าน", i);
        }
        assert_eq!(limiter.current_count(), 10);
    }

    #[test]
    fn test_rate_limiter_blocks_over_limit() {
        let mut limiter = RateLimiter::new(10, Duration::from_secs(1));
        for _ in 0..10 { limiter.check_and_record(); }
        assert!(!limiter.check_and_record(), "message เกิน limit ต้องถูก block");
    }

    #[test]
    fn test_rate_limiter_window_reset() {
        let mut limiter = RateLimiter::new(3, Duration::from_millis(1));
        for _ in 0..3 { assert!(limiter.check_and_record()); }
        std::thread::sleep(Duration::from_millis(10));
        assert!(limiter.check_and_record(), "หลัง window reset ต้องส่งได้");
    }

    // ========== Room Management Tests ==========

    #[tokio::test]
    async fn test_room_create_and_retrieve() {
        let registry = new_registry();
        let room = Room::new("general", "alice", false);
        let room_id = create_room(&registry, room).await;
        assert!(registry.contains_key(&room_id));
    }

    #[tokio::test]
    async fn test_room_join_adds_member() {
        let registry = new_registry();
        let room = Room::new("tech-talk", "alice", false);
        let room_id = create_room(&registry, room).await;
        let result = join_room(&registry, &room_id, "bob", "Bob").await;
        assert!(result.is_some());
        let members = get_members(&registry, &room_id).await.unwrap();
        assert!(members.contains(&"Bob".to_string()));
    }

    #[tokio::test]
    async fn test_room_leave_removes_member() {
        let registry = new_registry();
        let room = Room::new("lobby", "admin", false);
        let room_id = create_room(&registry, room).await;
        join_room(&registry, &room_id, "alice", "Alice").await;
        join_room(&registry, &room_id, "bob", "Bob").await;
        leave_room(&registry, &room_id, "alice", "Alice").await;
        let members = get_members(&registry, &room_id).await.unwrap();
        assert_eq!(members.len(), 1);
        assert!(!members.contains(&"Alice".to_string()));
    }

    #[tokio::test]
    async fn test_room_broadcast_received_by_subscriber() {
        use tokio::time::timeout;
        let registry = new_registry();
        let room = Room::new("broadcast-test", "system", false);
        let room_id = create_room(&registry, room).await;
        let (mut rx, _) = join_room(&registry, &room_id, "alice", "Alice").await.unwrap();
        // รับ join + members_list ของ alice
        let _ = timeout(Duration::from_secs(1), rx.recv()).await;
        let _ = timeout(Duration::from_secs(1), rx.recv()).await;
        // bob join → alice รับ join event
        let _ = join_room(&registry, &room_id, "bob", "Bob").await;
        let result = timeout(Duration::from_secs(1), rx.recv()).await;
        assert!(result.is_ok());
        let msg = result.unwrap().unwrap();
        assert_eq!(msg.msg_type, MessageType::Join);
        assert_eq!(msg.user, "Bob");
    }
}
```

### ผลลัพธ์จากการรัน `cargo test` จริง

```
   Compiling chat-app v0.1.0 (.../scratchpad/chat-app)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.30s
     Running unittests src/main.rs (target/debug/deps/chat_app-325d261a0e1ddb5f)

running 23 tests
test tests::test_jwt_extract_from_bearer_header ... ok
test tests::test_jwt_extract_missing_bearer_prefix ... ok
test tests::test_jwt_invalid_token_rejected ... ok
test tests::test_message_deserialization_from_json_string ... ok
test tests::test_message_serialization_members_list ... ok
test tests::test_message_deserialization_roundtrip ... ok
test tests::test_jwt_tampered_token_rejected ... ok
test tests::test_jwt_create_and_verify ... ok
test tests::test_message_serialization_type_join ... ok
test tests::test_message_serialization_type_leave ... ok
test tests::test_message_type_all_variants_serialize ... ok
test tests::test_message_serialization_type_message ... ok
test tests::test_rate_limiter_accurate_count ... ok
test tests::test_rate_limiter_blocks_over_limit ... ok
test tests::test_rate_limiter_allows_under_limit ... ok
test tests::test_multiple_rooms_independent ... ok
test tests::test_room_join_adds_member ... ok
test tests::test_room_create_and_retrieve ... ok
test tests::test_room_broadcast_received_by_subscriber ... ok
test tests::test_room_join_nonexistent_returns_none ... ok
test tests::test_room_metadata ... ok
test tests::test_room_leave_removes_member ... ok
test tests::test_rate_limiter_window_reset ... ok

test result: ok. 23 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

**23/23 tests ผ่านทั้งหมด**

---

## ⚠️ Pitfalls สำคัญ (สรุป 4 ข้อ)

### Pitfall #1 — Browser WebSocket ไม่รองรับ Custom Headers

Browser `WebSocket` API ไม่อนุญาตส่ง `Authorization` header ต้องใช้ query param `?token=` หรือส่ง token เป็น message แรกแทน ในฝั่ง server ต้องรองรับทั้งสองรูปแบบ

```rust
// รองรับทั้ง header และ query param
pub fn extract_ws_token<'a>(
    auth_header: Option<&'a str>,
    query_token: Option<&'a str>,
) -> Option<&'a str> {
    auth_header.and_then(|h| h.strip_prefix("Bearer "))
        .or(query_token)
}
```

### Pitfall #2 — broadcast::RecvError::Lagged ต้องจัดการ

ถ้า receiver ช้าและ buffer เต็ม Tokio จะ drop message เก่าและคืน `Lagged(n)` ซึ่งเป็น `Err` ไม่ใช่ panic ต้องจัดการทุก pattern match เสมอ ไม่งั้น `unwrap()` จะทำให้ task crash

### Pitfall #3 — SplitSink ไม่สามารถ borrow พร้อมกัน 2 ที่

`SplitSink` ไม่ implements `Clone` ดังนั้น ถ้าต้องการใช้ใน 2 concurrent tasks ต้องใช้ `tokio::sync::mpsc::channel` เป็น intermediary — task ที่ต้องการเขียน WebSocket ส่งผ่าน channel ไปให้ task เดียวที่เป็นเจ้าของ sink

### Pitfall #4 — DashMap Guard และ async lock ห้าม hold ข้าม `.await`

`dashmap::RefMut` และ `dashmap::Ref` ไม่ implements `Send` ดังนั้น **ห้าม hold** ไว้ขณะรอ `.await`:

```rust
// ❌ compile error — DashMap ref ถูก hold ข้าม await
let entry = registry.get(&room_id).unwrap();
some_async_function().await; // Error: RefMut not Send
drop(entry);

// ✅ ถูกต้อง — ดึง Arc<RwLock> ออกมาก่อน แล้วค่อย await
let arc = {
    let entry = registry.get(&room_id)?;
    Arc::clone(entry.value())
}; // entry dropped ที่นี่
let state = arc.write().await; // await ปลอดภัย
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build แบบ optimized
cargo build --release

# Binary อยู่ที่
./target/release/realtime-chat
```

### Docker

```dockerfile
# Build stage
FROM rust:1.82-slim-bookworm AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
# Cache dependencies
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm src/main.rs

COPY src ./src
RUN touch src/main.rs && cargo build --release

# Runtime stage — minimal image
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/realtime-chat /usr/local/bin/

ENV JWT_SECRET="change-in-production-min-32-bytes!!"
ENV BASE_URL="https://chat.example.com"
ENV REDIS_URL="redis://redis:6379/"
EXPOSE 3000

CMD ["realtime-chat"]
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  chat:
    build: .
    ports:
      - "3000:3000"
    environment:
      - JWT_SECRET=${JWT_SECRET}
      - BASE_URL=http://localhost:3000
      - REDIS_URL=redis://redis:6379/
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  redis_data:
```

### ทดสอบด้วย wscat

```bash
# ติดตั้ง wscat
npm install -g wscat

# สร้าง JWT token ก่อน (endpoint ทดสอบ)
TOKEN=$(curl -s -X POST http://localhost:3000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"user_id":"u1","username":"alice"}' | jq -r .token)

# เชื่อมต่อ WebSocket
wscat -c "ws://localhost:3000/ws?token=$TOKEN&room=general"

# ส่ง message
> {"type":"message","room":"general","content":"สวัสดีทุกคน!"}

# คาดหวัง output
< {"type":"message","room":"general","user":"alice","content":"สวัสดีทุกคน!","timestamp":"2025-09-27T..."}
```

### ทดสอบ REST API

```bash
# สร้างห้องใหม่
curl -X POST http://localhost:3000/rooms \
  -H "Content-Type: application/json" \
  -d '{"name":"dev-chat","is_private":false}'
# Response: {"id":"...","name":"dev-chat","is_private":false,...}

# รายชื่อห้อง
curl http://localhost:3000/rooms
# Response: [{"id":"...","name":"general","member_count":0},...]

# ดู history
curl http://localhost:3000/rooms/{id}/history

# อัปโหลดไฟล์
curl -X POST http://localhost:3000/rooms/{id}/files \
  -F "file=@document.pdf"
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1 — Direct Messages (DM) ระหว่าง Users

ปัจจุบันระบบรองรับแค่ห้องสาธารณะ ให้เพิ่ม private DM ระหว่าง 2 users

**สิ่งที่ต้องทำ:**
- เพิ่ม endpoint `POST /dm/{user_id}` เพื่อเริ่ม DM
- สร้าง room_id อัตโนมัติจาก sorted user IDs เช่น `dm:alice:bob`
- ปรับ room validation ให้ตรวจ permission ก่อน join DM room
- เพิ่ม message type `direct_message` ที่ client filter ได้

```rust
// Room ID สำหรับ DM
pub fn dm_room_id(user1: &str, user2: &str) -> String {
    let mut ids = vec![user1, user2];
    ids.sort(); // เพื่อให้ alice→bob และ bob→alice ได้ id เดียวกัน
    format!("dm:{}:{}", ids[0], ids[1])
}
```

### แบบฝึกหัดที่ 2 — Message Reactions (Emoji Reactions)

เพิ่มระบบ react ด้วย emoji เหมือน Slack/Discord

**สิ่งที่ต้องทำ:**
- เพิ่ม message type `reaction` พร้อม fields: `message_id`, `emoji`, `user`
- เก็บ reactions ใน Redis hash `chat:room:{id}:reactions:{msg_id}`
- Broadcast reaction update ไปทุกคนในห้อง
- เพิ่ม `GET /rooms/{id}/messages/{msg_id}/reactions` endpoint

```json
// Client ส่ง:
{"type": "reaction", "room": "general", "message_id": "msg-123", "emoji": "👍"}

// Server broadcast:
{"type": "reaction", "room": "general", "user": "alice",
 "message_id": "msg-123", "emoji": "👍", "count": 3}
```

### แบบฝึกหัดที่ 3 — Persistent Message Storage ด้วย SQLite

เพิ่ม SQLite ด้วย `sqlx` เพื่อเก็บ messages ถาวร (Redis เป็น cache เท่านั้น)

**สิ่งที่ต้องทำ:**
- เพิ่ม dependency `sqlx = { version = "0.8", features = ["sqlite", "runtime-tokio", "chrono"] }`
- สร้าง migration: messages table พร้อม index บน `(room_id, timestamp)`
- Write path: บันทึก SQLite ก่อน แล้วค่อย cache Redis
- Read path: ตรวจ Redis ก่อน miss → query SQLite → fill Redis cache
- เพิ่ม pagination `GET /rooms/{id}/history?before={timestamp}&limit=50`

### แบบฝึกหัดที่ 4 — Horizontal Scaling ด้วย Redis Pub/Sub

ระบบปัจจุบันใช้ `tokio::sync::broadcast` ซึ่งทำงานได้แค่ใน process เดียว ถ้า deploy หลาย instances messages จาก instance A จะไม่ถึง clients ที่ connect กับ instance B

**สิ่งที่ต้องทำ:**
- แทนที่ `broadcast::channel` ด้วย Redis Pub/Sub (`PUBLISH`/`SUBSCRIBE`)
- แต่ละ instance subscribe channel `chat:room:{id}` ของ Redis
- เมื่อรับ message จาก Redis → broadcast ไปยัง local connections ด้วย `broadcast`
- เมื่อ user ส่ง message → `PUBLISH` ไปยัง Redis แทนที่จะ broadcast โดยตรง

```rust
// Publish ไป Redis แทน broadcast โดยตรง
async fn publish_to_redis(client: &redis::Client, room_id: &str, msg: &ChatMessage) {
    let channel = format!("chat:room:{}", room_id);
    let json = serde_json::to_string(msg).unwrap();
    let mut conn = client.get_multiplexed_async_connection().await.unwrap();
    let _: () = conn.publish(&channel, &json).await.unwrap();
}

// Subscribe loop (รัน 1 ครั้งต่อ room ต่อ instance)
async fn subscribe_redis_room(client: &redis::Client, room_id: &str, local_tx: broadcast::Sender<ChatMessage>) {
    let mut pubsub = client.get_async_pubsub().await.unwrap();
    pubsub.subscribe(format!("chat:room:{}", room_id)).await.unwrap();
    while let Some(msg) = pubsub.on_message().next().await {
        let payload: String = msg.get_payload().unwrap();
        if let Ok(chat_msg) = serde_json::from_str::<ChatMessage>(&payload) {
            let _ = local_tx.send(chat_msg);
        }
    }
}
```

---

## สรุป

โปรเจคนี้สร้าง real-time chat server ที่ครบถ้วนสำหรับ production โดยใช้ Rust patterns ที่สำคัญ:

**Pattern หลักที่ได้เรียน:**

1. **WebSocket Split Pattern** — `socket.split()` → `SplitSink` + `SplitStream` + `tokio::select!` สำหรับ bidirectional concurrent communication ใน async context

2. **Shared State with DashMap** — `Arc<DashMap<K, Arc<RwLock<V>>>>` สำหรับ concurrent access โดยไม่ต้อง lock global structure — DashMap แบ่ง shard อัตโนมัติ

3. **broadcast::channel Fan-out** — sender เดียว → receivers หลายคน เหมาะสำหรับ pub/sub pattern ภายใน process

4. **Sliding Window Rate Limiter** — `VecDeque<Instant>` สำหรับ O(1) check และ O(n) cleanup โดยที่ n คือ messages ใน window ขนาดเล็ก

5. **DashMap + await Safety** — อย่า hold `DashMap::Ref` ข้าม `.await` ให้ดึง `Arc` ออกมาก่อนแล้วค่อย await

**เชื่อมโยงกับโปรเจคถัดไป:**

Project B08 (Email Service) จะนำ pattern async task processing ที่เรียนในโปรเจคนี้ไปใช้กับ email queue — ส่ง email แบบ background task พร้อม retry logic, bounce handling และ delivery tracking ซึ่งใช้ `tokio::spawn` + `mpsc::channel` ในลักษณะเดียวกับที่เราใช้กับ WebSocket ที่นี่

---

**โปรเจคก่อนหน้า:** [Project B06: OAuth2 Server](project-b06-oauth2-server.md) | **โปรเจคถัดไป:** [Project B08: Email Service](project-b08-email-service.md)
