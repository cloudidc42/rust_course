# Project H08: Multi-Room TCP Chat Server

> โมดูล: H — Networking & Protocols | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 14 ชั่วโมง

## ภาพรวมโปรเจค

TCP chat server คือ classic networking project ที่สอนทุก pattern สำคัญของ async Rust networking พร้อมกัน ตั้งแต่ line-framed protocol ไปจนถึง broadcast channels และ concurrent shared state บน DashMap โปรเจคนี้สร้าง **multi-room TCP chat server** ที่รองรับ:

- **หลายห้องพร้อมกัน** — client เข้า/ออกห้องได้อิสระ, server broadcast ไปทุกคนในห้อง
- **JSON line protocol** — ทุก message เป็น JSON object ที่คั่นด้วย `\n` อ่านง่าย debug ง่าย
- **Direct messages (DMs)** — ส่งข้อความส่วนตัวข้ามห้องได้
- **Room history** — ห้องเก็บ 50 messages ล่าสุดไว้ให้ client ที่เพิ่งเข้ามาดูได้
- **Nickname management** — authenticate ด้วย nickname ก่อนใช้งาน, rename ได้ตลอด

ในโลก production ระบบ chat อย่าง **IRC**, **Slack (backend)**, **Discord** ล้วนใช้ architecture ที่ใกล้เคียงกัน แม้จะซับซ้อนกว่ามาก แต่ core pattern ที่เราจะสร้างคือพื้นฐานเดียวกัน

ทำไมโปรเจคนี้ถึงน่าสร้าง?

1. **tokio broadcast channel** คือหัวใจของ pub/sub pattern ใน async Rust — เข้าใจ `Lagged` error และ backpressure
2. **DashMap** แทน `Mutex<HashMap>` — ลด lock contention ใน concurrent read/write
3. **tokio split** — แยก reader กับ writer ออกจาก TCP stream เดียวกัน
4. **Framing protocol** — JSON lines แก้ปัญหา TCP stream ไม่มี message boundaries

## สิ่งที่จะได้เรียนรู้

- **JSON line framing:** ใช้ `AsyncBufReadExt::read_line` คั่น TCP stream เป็น messages ชัดเจน
- **tokio broadcast channel:** ส่ง event ไปยัง subscribers หลายคนพร้อมกัน, จัดการ `Lagged` error
- **DashMap concurrent access:** map ที่อ่าน/เขียนพร้อมกันได้โดยไม่ต้อง lock ทั้ง map
- **tokio split:** แยก read/write half เพื่อทำงานพร้อมกันในสอง task
- **mpsc unbounded channel:** ส่ง message จาก reader task ไปยัง writer task อย่างปลอดภัย
- **Connection state machine:** authenticate → join room → dispatch commands → cleanup
- **Arc shared state:** แชร์ server state ข้าม tasks โดยไม่ copy data
- **Graceful disconnect:** cleanup nickname, leave room, broadcast เมื่อ client หลุด

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50:** async/await fundamentals, tokio runtime, `spawn`
- **Part 51–55:** tokio channels — `mpsc`, `broadcast`, `oneshot`
- **Part 56–60:** `Arc<T>`, shared state ข้าม async tasks
- **Part 61–65:** `TcpListener`, `TcpStream`, `tokio::net`
- **Part 66–70:** `tokio::io::AsyncBufReadExt`, `AsyncWriteExt`, `BufReader`
- **Part 41–45:** `HashMap`, `HashSet`, `VecDeque` — collections ที่ใช้เป็น room state
- **Part 21–25:** enum, struct, pattern matching
- **Part 31–35:** `Result`, `?` operator, error handling

## โครงสร้างโปรเจค (Project Layout)

```
tcp-chat/
├── src/
│   ├── main.rs          ← tokio entry point, TCP acceptor, connection handler
│   ├── protocol.rs      ← ChatMessage enum, JSON line serialize/deserialize
│   ├── room.rs          ← RoomManager, Room struct, broadcast, history
│   ├── dm.rs            ← DmStore — direct messages ระหว่าง users
│   └── nickname.rs      ← NicknameStore — ClientId ↔ nickname mapping
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
TCP Client ──connect()──► tokio::spawn(handle_client)
                               │
                    ┌──────────▼───────────────────────┐
                    │  Authenticate: NICK <name>         │
                    │  NicknameStore::register()         │
                    └──────────┬───────────────────────┘
                               │ ClientId, nickname
                    ┌──────────▼───────────────────────┐
                    │  into_split() → (reader, writer)  │
                    │  mpsc::unbounded_channel()         │
                    │  tokio::spawn(writer task)         │
                    └──────────┬───────────────────────┘
                               │
                    ┌──────────▼───────────────────────┐
                    │  AsyncBufReadExt::read_line loop   │
                    ├──────────────────────────────────┤
                    │ /join <room>  → RoomManager::join  │
                    │               broadcast::subscribe │
                    │               tokio::spawn(recv)   │
                    │                                    │
                    │ /part         → leave_room()       │
                    │               broadcast Leave      │
                    │                                    │
                    │ /rooms        → list_rooms()       │
                    │               send RoomList        │
                    │                                    │
                    │ /users        → list_users()       │
                    │               send UserList        │
                    │                                    │
                    │ /rename <n>   → NicknameStore      │
                    │               ::rename()           │
                    │                                    │
                    │ /msg <n> <t>  → DmStore::store()   │
                    │                                    │
                    │ plain text    → push_history()     │
                    │               broadcast Message    │
                    └──────────────────────────────────┘
                               │ disconnect
                    ┌──────────▼───────────────────────┐
                    │  leave_room(), nicks.remove()      │
                    │  broadcast Leave                   │
                    └──────────────────────────────────┘
```

### Module Boundaries

```
┌─────────────────────────────────────────────────────────┐
│                     AppState (Arc)                       │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐ │
│  │ RoomManager  │  │  NicknameStore│  │  DmStore     │ │
│  │ DashMap<name,│  │  DashMap id→  │  │  DashMap     │ │
│  │  Room>       │  │    nick       │  │  (a,b)→msgs  │ │
│  │              │  │  DashMap nick→│  │              │ │
│  │ Room:        │  │    id         │  │              │ │
│  │  members:    │  └───────────────┘  └──────────────┘ │
│  │  HashSet<id> │                                       │
│  │  history:    │                                       │
│  │  VecDeque<M> │                                       │
│  │  sender:     │                                       │
│  │  broadcast   │                                       │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

### ทำไมถึงเลือก DashMap แทน Mutex<HashMap>

`Mutex<HashMap>` lock ทั้ง map ทุกครั้งที่ access แม้แต่ read operation `DashMap` ใช้ sharding — map ถูกแบ่งเป็น N shard แต่ละ shard มี lock อิสระ ดังนั้น concurrent reads บน different keys ไม่ block กัน เหมาะมากสำหรับ server ที่มี clients หลาย connections อ่าน/เขียนพร้อมกัน

### ทำไมถึงใช้ tokio broadcast channel แทน Vec<Sender>

`broadcast::Sender` มี semantics ที่ตรงกับ chat room มาก:
- ส่งครั้งเดียว ทุก subscriber ได้รับพร้อมกัน (zero-copy clone)
- handle `Lagged` error เมื่อ slow client ตาม buffer ไม่ทัน
- `receiver_count()` บอกจำนวน active subscribers

ทางเลือกอื่น คือ `Vec<mpsc::Sender<_>>` แต่ต้องจัดการ remove disconnected senders เองซึ่งซับซ้อนกว่า

### ทำไมถึงใช้ JSON Lines แทน Binary Protocol

JSON lines ดีกว่า binary สำหรับ learning project เพราะ:
1. Debug ได้ด้วย `telnet` หรือ `nc` โดยตรง
2. Parse ง่าย — `serde_json::from_str` บรรทัดเดียว
3. Schema-flexible — เพิ่ม field ได้โดยไม่ break backward compat
4. Human-readable ทำให้ trace ปัญหาได้เร็ว

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: Protocol Definition — ChatMessage enum

เริ่มจากสิ่งสำคัญที่สุด: กำหนด **wire protocol** ให้ชัดก่อนเขียน network code เพราะทุกอย่างอื่นต้องอ้างอิง enum นี้

จาก Part 46 เรื่อง serde และ Part 21 เรื่อง enum ใน Rust เราใช้ `#[serde(tag = "type", content = "data")]` ซึ่งทำให้ JSON ออกมาในรูป `{"type": "Message", "data": {...}}` — อ่านง่ายกว่า tuple variant และ debug ได้เร็วกว่า

**`src/protocol.rs`**

```rust
use serde::{Deserialize, Serialize};

/// ClientId เป็น UUID string ที่ unique ต่อ connection
pub type ClientId = String;

/// ChatMessage enum ครอบคลุม message ทุกชนิดที่ client ส่งหรือรับ
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
#[serde(tag = "type", content = "data")]
pub enum ChatMessage {
    /// Client ส่ง: JOIN ห้อง
    Join { room: String },
    /// Server แจ้ง: user ออกห้อง
    Leave { room: String, nickname: String },
    /// ข้อความปกติในห้อง
    Message {
        from: String,
        room: String,
        text: String,
    },
    /// ข้อความส่วนตัว
    PrivateMessage {
        from: String,
        to: String,
        text: String,
    },
    /// ขอรายการห้อง
    ListRooms,
    /// ขอรายการ users ในห้อง
    ListUsers { room: String },
    /// เปลี่ยนชื่อ
    Rename { new_nick: String },
    /// Error จาก server
    Error { code: String, message: String },
    /// Server response: รายการห้อง
    RoomList { rooms: Vec<RoomInfo> },
    /// Server response: รายการ users
    UserList { room: String, users: Vec<String> },
    /// Server notify: user เข้าห้อง
    UserJoined { room: String, nickname: String },
    /// Server notify: เปลี่ยนชื่อสำเร็จ
    RenameOk { old_nick: String, new_nick: String },
    /// Server acknowledge
    Ok { message: String },
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct RoomInfo {
    pub name: String,
    pub member_count: usize,
}

/// เก็บ DM message
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct StoredDm {
    pub from: ClientId,
    pub text: String,
}

impl ChatMessage {
    /// Serialize เป็น JSON line (มี \n ท้าย)
    pub fn to_json_line(&self) -> String {
        let mut s = serde_json::to_string(self).unwrap();
        s.push('\n');
        s
    }

    /// Parse จาก JSON string (ไม่จำเป็นต้องมี \n)
    pub fn from_json_line(line: &str) -> Result<Self, serde_json::Error> {
        serde_json::from_str(line.trim_end_matches('\n'))
    }
}
```

ตัวอย่าง JSON output ของ enum แต่ละ variant:

```json
{"type":"Message","data":{"from":"alice","room":"general","text":"hello"}}
{"type":"Join","data":{"room":"rust"}}
{"type":"Error","data":{"code":"NICK_TAKEN","message":"Nickname 'alice' already taken"}}
{"type":"RoomList","data":{"rooms":[{"name":"general","member_count":3}]}}
```

สังเกตว่า `ListRooms` ซึ่งไม่มี data จะ serialize เป็น:
```json
{"type":"ListRooms","data":null}
```

**Cargo.toml ที่ต้องใช้:**

```toml
[package]
name = "tcp-chat"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
dashmap = "5"
uuid = { version = "1", features = ["v4"] }
```

---

### ขั้นที่ 2: NicknameStore — Authentication Layer

ก่อน client จะทำอะไรได้ต้อง authenticate ด้วยการตั้ง nickname ก่อน `NicknameStore` ทำหน้าที่ 2 อย่าง:

1. **Enforce uniqueness** — ป้องกันสอง client ใช้ชื่อเดียวกัน
2. **Bi-directional lookup** — lookup ชื่อจาก ID หรือ ID จากชื่อได้ทั้งคู่

จาก Part 56 เรื่อง concurrent data structures เราใช้ `DashMap` สองอัน ทำ bi-directional mapping โดยไม่ต้อง lock

**`src/nickname.rs`**

```rust
use dashmap::DashMap;
use crate::protocol::ClientId;

pub struct NicknameStore {
    pub id_to_nick: DashMap<ClientId, String>,
    pub nick_to_id: DashMap<String, ClientId>,
}

impl NicknameStore {
    pub fn new() -> Self {
        NicknameStore {
            id_to_nick: DashMap::new(),
            nick_to_id: DashMap::new(),
        }
    }

    /// ลงทะเบียน nickname ใหม่
    /// คืน Err ถ้า nickname ถูกใช้แล้ว
    pub fn register(&self, id: &ClientId, nick: &str) -> Result<(), String> {
        if self.nick_to_id.contains_key(nick) {
            return Err(format!("Nickname '{}' already taken", nick));
        }
        self.id_to_nick.insert(id.clone(), nick.to_string());
        self.nick_to_id.insert(nick.to_string(), id.clone());
        Ok(())
    }

    /// เปลี่ยน nickname — ลบ old, เพิ่ม new
    pub fn rename(&self, id: &ClientId, new_nick: &str) -> Result<String, String> {
        if self.nick_to_id.contains_key(new_nick) {
            return Err(format!("Nickname '{}' already taken", new_nick));
        }
        let old = self
            .id_to_nick
            .get(id)
            .map(|v| v.clone())
            .unwrap_or_default();
        self.nick_to_id.remove(&old);
        self.id_to_nick.insert(id.clone(), new_nick.to_string());
        self.nick_to_id.insert(new_nick.to_string(), id.clone());
        Ok(old)
    }

    /// ลบ client ออก (เมื่อ disconnect)
    pub fn remove(&self, id: &ClientId) {
        if let Some((_, nick)) = self.id_to_nick.remove(id) {
            self.nick_to_id.remove(&nick);
        }
    }

    pub fn get_nick(&self, id: &ClientId) -> Option<String> {
        self.id_to_nick.get(id).map(|v| v.clone())
    }

    pub fn get_id(&self, nick: &str) -> Option<ClientId> {
        self.nick_to_id.get(nick).map(|v| v.clone())
    }
}
```

**Race condition ที่ต้องระวัง:** ใน `DashMap` การ check `contains_key` แล้ว `insert` ไม่ atomic ในทางทฤษฎีสอง client อาจ register ชื่อเดียวกันพร้อมกันได้ในกรณีที่ใช้ `DashMap` เปล่า ๆ วิธีแก้ใน production คือใช้ `entry().or_insert()` และตรวจสอบผล หรือเพิ่ม `Mutex` ครอบ register operation

---

### ขั้นที่ 3: Room State — DashMap + broadcast channel

ส่วนสำคัญที่สุดของ chat server คือ `RoomManager` ซึ่งจัดการ:

- **member tracking** — ใครอยู่ในห้องไหน
- **broadcast delivery** — ส่ง message ไปทุกคนในห้อง
- **history buffer** — เก็บ 50 messages ล่าสุด

จาก Part 51 เรื่อง broadcast channel — `broadcast::Sender` ใน Room ทำให้ server ส่งครั้งเดียวแล้วทุก subscriber ได้รับ

**`src/room.rs`**

```rust
use std::collections::{HashSet, VecDeque};
use dashmap::DashMap;
use tokio::sync::broadcast;
use crate::protocol::{ChatMessage, ClientId, RoomInfo};

pub const MAX_HISTORY: usize = 50;
pub const BROADCAST_CAPACITY: usize = 100;

#[derive(Debug, Clone)]
pub struct StoredMessage {
    pub from: String,
    pub text: String,
}

pub struct Room {
    pub name: String,
    pub members: HashSet<ClientId>,
    pub history: VecDeque<StoredMessage>,
    pub sender: broadcast::Sender<ChatMessage>,
}

impl Room {
    pub fn new(name: impl Into<String>) -> Self {
        let (sender, _) = broadcast::channel(BROADCAST_CAPACITY);
        Room {
            name: name.into(),
            members: HashSet::new(),
            history: VecDeque::new(),
            sender,
        }
    }

    /// เพิ่ม message ลง history — ถ้าเกิน MAX_HISTORY ให้ลบอันเก่าที่สุดออก
    pub fn push_history(&mut self, from: String, text: String) {
        if self.history.len() >= MAX_HISTORY {
            self.history.pop_front();
        }
        self.history.push_back(StoredMessage { from, text });
    }
}

pub struct RoomManager {
    pub rooms: DashMap<String, Room>,
}

impl RoomManager {
    pub fn new() -> Self {
        let mgr = RoomManager { rooms: DashMap::new() };
        mgr.rooms.insert("general".to_string(), Room::new("general"));
        mgr
    }

    pub fn list_rooms(&self) -> Vec<RoomInfo> {
        self.rooms
            .iter()
            .map(|e| RoomInfo {
                name: e.key().clone(),
                member_count: e.value().members.len(),
            })
            .collect()
    }

    /// เพิ่ม member เข้าห้อง — สร้างห้องใหม่ถ้ายังไม่มี
    /// คืน broadcast::Receiver สำหรับ client ที่เพิ่งเข้า
    pub fn join_room(
        &self,
        room_name: &str,
        client_id: &ClientId,
    ) -> broadcast::Receiver<ChatMessage> {
        let mut entry = self.rooms
            .entry(room_name.to_string())
            .or_insert_with(|| Room::new(room_name));
        entry.members.insert(client_id.clone());
        entry.sender.subscribe()
    }

    /// ลบ member ออกจากห้อง — คืน true ถ้าห้องว่างแล้ว
    pub fn leave_room(&self, room_name: &str, client_id: &ClientId) -> bool {
        if let Some(mut room) = self.rooms.get_mut(room_name) {
            room.members.remove(client_id);
            return room.members.is_empty();
        }
        false
    }

    /// Broadcast message ไปทุก subscriber ของห้อง
    pub fn broadcast(&self, room_name: &str, msg: ChatMessage) -> usize {
        if let Some(room) = self.rooms.get(room_name) {
            let _ = room.sender.send(msg);
            return room.sender.receiver_count();
        }
        0
    }

    pub fn push_history(&self, room_name: &str, from: String, text: String) {
        if let Some(mut room) = self.rooms.get_mut(room_name) {
            room.push_history(from, text);
        }
    }

    pub fn list_users(&self, room_name: &str) -> Vec<String> {
        if let Some(room) = self.rooms.get(room_name) {
            room.members.iter().cloned().collect()
        } else {
            vec![]
        }
    }
}
```

**จุดสำคัญของ `VecDeque` สำหรับ history:**

`VecDeque` เหมาะกับ bounded queue มากกว่า `Vec` เพราะ `pop_front` เป็น O(1) ใน `Vec` ต้อง shift elements ทั้งหมดซึ่งเป็น O(n) สำหรับ history buffer ที่ trim front บ่อย `VecDeque` จึงมีประสิทธิภาพกว่าชัดเจน

---

### ขั้นที่ 4: DmStore — Direct Messages ข้ามห้อง

Direct messages ระหว่าง users ต้องทำงานได้แม้สอง users อยู่คนละห้อง `DmStore` เก็บ conversation history ด้วย canonical key pair เพื่อกันซ้ำ

**`src/dm.rs`**

```rust
use dashmap::DashMap;
use crate::protocol::{ClientId, StoredDm};

pub struct DmStore {
    /// key: (a, b) โดย a <= b (canonical order)
    pub active: DashMap<(ClientId, ClientId), Vec<StoredDm>>,
}

impl DmStore {
    pub fn new() -> Self {
        DmStore { active: DashMap::new() }
    }

    /// Canonical key — เรียงให้ id เล็กก่อนเพื่อกัน duplicate key
    fn key(a: &ClientId, b: &ClientId) -> (ClientId, ClientId) {
        if a <= b {
            (a.clone(), b.clone())
        } else {
            (b.clone(), a.clone())
        }
    }

    pub fn store(&self, from: &ClientId, to: &ClientId, text: String) {
        let key = Self::key(from, to);
        self.active
            .entry(key)
            .or_insert_with(Vec::new)
            .push(StoredDm { from: from.clone(), text });
    }

    pub fn history(&self, a: &ClientId, b: &ClientId) -> Vec<StoredDm> {
        let key = Self::key(a, b);
        self.active.get(&key).map(|v| v.clone()).unwrap_or_default()
    }
}
```

---

### ขั้นที่ 5: AppState — Shared Server State

ทุก module รวมกันใน `AppState` ที่ wrap ด้วย `Arc` เพื่อแชร์ข้าม tasks

**`src/main.rs` (ส่วน state)**

```rust
mod protocol;
mod room;
mod dm;
mod nickname;

use std::sync::Arc;
use crate::room::RoomManager;
use crate::dm::DmStore;
use crate::nickname::NicknameStore;

pub struct AppState {
    pub rooms: RoomManager,
    pub dms: DmStore,
    pub nicks: NicknameStore,
}

impl AppState {
    pub fn new() -> Arc<Self> {
        Arc::new(AppState {
            rooms: RoomManager::new(),
            dms: DmStore::new(),
            nicks: NicknameStore::new(),
        })
    }
}
```

`Arc` ทำให้ clone ได้โดยไม่ copy data เพียงเพิ่ม reference count แต่ละ `tokio::spawn` task ได้รับ `Arc<AppState>` เดียวกัน ชี้ไปยัง state object เดียวกันใน heap

---

### ขั้นที่ 6: Connection Handler — Authentication + Reader Loop

`handle_client` คือ state machine ที่จัดการ lifecycle ของ client connection:

```
CONNECTED → AUTHENTICATING (รอ NICK) → ACTIVE (dispatch commands) → DISCONNECTED
```

**ส่วน authentication:**

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpStream;

async fn handle_client(socket: TcpStream, state: Arc<AppState>) {
    let client_id = uuid::Uuid::new_v4().to_string();
    let (reader, mut writer) = socket.into_split();
    let mut reader = BufReader::new(reader);

    writer
        .write_all(b"Welcome! Please set your nickname with: NICK <name>\n")
        .await
        .unwrap_or(());

    let mut line = String::new();
    loop {
        line.clear();
        match reader.read_line(&mut line).await {
            Ok(0) | Err(_) => return, // EOF หรือ error = disconnect
            Ok(_) => {}
        }
        let trimmed = line.trim();
        if trimmed.starts_with("NICK ") {
            let nick = trimmed[5..].trim();
            match state.nicks.register(&client_id, nick) {
                Ok(()) => {
                    let ok = ChatMessage::Ok {
                        message: format!("Nickname set to '{}'", nick)
                    };
                    writer.write_all(ok.to_json_line().as_bytes()).await.unwrap_or(());
                    break;
                }
                Err(e) => {
                    let err = ChatMessage::Error {
                        code: "NICK_TAKEN".to_string(),
                        message: e,
                    };
                    writer.write_all(err.to_json_line().as_bytes()).await.unwrap_or(());
                    // ไม่ break — รอให้ client ลองชื่อใหม่
                }
            }
        }
    }
    // ...
}
```

**จุดสำคัญ:** `socket.into_split()` แยก TCP stream เป็น `OwnedReadHalf` กับ `OwnedWriteHalf` ที่ส่งข้าม threads ได้ (ต่างจาก `split()` ที่ return borrow) นี่ทำให้เราสามารถ move write half เข้า writer task และ read half อยู่ใน reader loop ได้

**ส่วน writer task และ mpsc channel:**

```rust
use tokio::sync::mpsc;

// channel สำหรับส่ง message จาก reader loop ไปยัง writer
let (tx, mut rx) = mpsc::unbounded_channel::<ChatMessage>();

let _writer_task = {
    let mut writer = writer;
    tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            let _ = writer.write_all(msg.to_json_line().as_bytes()).await;
        }
        // channel ถูก drop → task จบ
    })
};
```

ทำไมต้องใช้ `mpsc` channel แทนส่ง `writer` โดยตรง? เพราะ reader loop ต้องการส่ง response ไปยัง writer แต่ writer ถูก move เข้า writer task ไปแล้ว `mpsc::Sender` clone ได้จึงส่งได้จากหลายที่ (เช่น broadcast receiver task ก็ต้องส่งด้วย)

---

### ขั้นที่ 7: Command Dispatcher

เมื่อ authenticate สำเร็จ reader loop ก็ dispatch commands ต่าง ๆ:

```rust
pub struct ClientState {
    pub id: ClientId,
    pub nickname: String,
    pub current_room: Option<String>,
}

// ใน reader loop:
loop {
    line.clear();
    match reader.read_line(&mut line).await {
        Ok(0) | Err(_) => break,
        Ok(_) => {}
    }
    let trimmed = line.trim();
    if trimmed.is_empty() { continue; }

    if trimmed.starts_with('/') {
        dispatch_command(trimmed, &mut client, &state, &tx).await;
    } else {
        // plain text → broadcast ไปห้องปัจจุบัน
        if let Some(room) = &client.current_room.clone() {
            let nick = state.nicks
                .get_nick(&client.id)
                .unwrap_or(client.nickname.clone());
            let msg = ChatMessage::Message {
                from: nick.clone(),
                room: room.clone(),
                text: trimmed.to_string(),
            };
            state.rooms.push_history(room, nick, trimmed.to_string());
            state.rooms.broadcast(room, msg);
        } else {
            let _ = tx.send(ChatMessage::Error {
                code: "NOT_IN_ROOM".to_string(),
                message: "Join a room first with /join <room>".to_string(),
            });
        }
    }
}
```

**Command dispatch function:**

```rust
async fn dispatch_command(
    line: &str,
    client: &mut ClientState,
    state: &Arc<AppState>,
    tx: &mpsc::UnboundedSender<ChatMessage>,
) {
    let parts: Vec<&str> = line.splitn(3, ' ').collect();
    match parts[0] {
        "/rooms" => {
            let rooms = state.rooms.list_rooms();
            let _ = tx.send(ChatMessage::RoomList { rooms });
        }
        "/users" => {
            if let Some(room) = &client.current_room {
                let ids = state.rooms.list_users(room);
                let users: Vec<String> = ids
                    .iter()
                    .filter_map(|id| state.nicks.get_nick(id))
                    .collect();
                let _ = tx.send(ChatMessage::UserList {
                    room: room.clone(),
                    users,
                });
            } else {
                let _ = tx.send(ChatMessage::Error {
                    code: "NOT_IN_ROOM".to_string(),
                    message: "Not in a room".to_string(),
                });
            }
        }
        "/join" => {
            if parts.len() < 2 {
                let _ = tx.send(ChatMessage::Error {
                    code: "BAD_ARGS".to_string(),
                    message: "/join requires room name".to_string(),
                });
                return;
            }
            let room_name = parts[1];
            // ออกจากห้องเดิมก่อน
            if let Some(old) = client.current_room.take() {
                state.rooms.leave_room(&old, &client.id);
                state.rooms.broadcast(
                    &old,
                    ChatMessage::Leave {
                        room: old.clone(),
                        nickname: client.nickname.clone(),
                    },
                );
            }
            let _rx = state.rooms.join_room(room_name, &client.id);
            client.current_room = Some(room_name.to_string());
            state.rooms.broadcast(
                room_name,
                ChatMessage::UserJoined {
                    room: room_name.to_string(),
                    nickname: client.nickname.clone(),
                },
            );
            let _ = tx.send(ChatMessage::Ok {
                message: format!("Joined #{}", room_name),
            });
        }
        "/part" => {
            if let Some(room) = client.current_room.take() {
                state.rooms.leave_room(&room, &client.id);
                state.rooms.broadcast(
                    &room,
                    ChatMessage::Leave {
                        room: room.clone(),
                        nickname: client.nickname.clone(),
                    },
                );
                let _ = tx.send(ChatMessage::Ok {
                    message: format!("Left #{}", room),
                });
            }
        }
        "/rename" => {
            if parts.len() < 2 {
                let _ = tx.send(ChatMessage::Error {
                    code: "BAD_ARGS".to_string(),
                    message: "/rename requires new nickname".to_string(),
                });
                return;
            }
            match state.nicks.rename(&client.id, parts[1]) {
                Ok(old_nick) => {
                    client.nickname = parts[1].to_string();
                    let _ = tx.send(ChatMessage::RenameOk {
                        old_nick,
                        new_nick: parts[1].to_string(),
                    });
                }
                Err(e) => {
                    let _ = tx.send(ChatMessage::Error {
                        code: "NICK_TAKEN".to_string(),
                        message: e,
                    });
                }
            }
        }
        "/msg" => {
            if parts.len() < 3 {
                let _ = tx.send(ChatMessage::Error {
                    code: "BAD_ARGS".to_string(),
                    message: "/msg requires target nick and message".to_string(),
                });
                return;
            }
            if let Some(target_id) = state.nicks.get_id(parts[1]) {
                state.dms.store(&client.id, &target_id, parts[2].to_string());
                let _ = tx.send(ChatMessage::Ok {
                    message: format!("DM sent to {}", parts[1]),
                });
            } else {
                let _ = tx.send(ChatMessage::Error {
                    code: "NO_SUCH_USER".to_string(),
                    message: format!("User '{}' not found", parts[1]),
                });
            }
        }
        "/quit" => {
            let _ = tx.send(ChatMessage::Ok { message: "Bye!".to_string() });
        }
        _ => {
            let _ = tx.send(ChatMessage::Error {
                code: "UNKNOWN_CMD".to_string(),
                message: format!("Unknown command: {}", parts[0]),
            });
        }
    }
}
```

---

### ขั้นที่ 8: Broadcast Receiver Task — จัดการ Lagged Error

เมื่อ client join ห้อง เราได้ `broadcast::Receiver` ซึ่งต้อง spawn task แยกเพื่อรับ messages และส่งไปยัง writer ผ่าน mpsc channel

```rust
use tokio::sync::broadcast;

// ใน /join handler หลังจาก join_room():
let mut bcast_rx = state.rooms.join_room(room_name, &client.id);
let tx_clone = tx.clone();
tokio::spawn(async move {
    loop {
        match bcast_rx.recv().await {
            Ok(msg) => {
                if tx_clone.send(msg).is_err() {
                    // mpsc receiver dropped → writer task จบแล้ว → exit
                    break;
                }
            }
            Err(broadcast::error::RecvError::Lagged(n)) => {
                // client ตาม buffer ไม่ทัน — แจ้งเตือนแล้วดำเนินต่อ
                let warn = ChatMessage::Error {
                    code: "LAGGED".to_string(),
                    message: format!("Missed {} messages (too slow)", n),
                };
                let _ = tx_clone.send(warn);
                // ไม่ break — client ยังรับต่อไปได้จาก current position
            }
            Err(broadcast::error::RecvError::Closed) => {
                // sender dropped → ห้องถูกปิด
                break;
            }
        }
    }
});
```

**`Lagged` error คืออะไร?**

`tokio::sync::broadcast` channel มี internal ring buffer ขนาดคงที่ (ในที่นี้ 100) ถ้า receiver อ่านช้าเกินไปและ buffer เต็ม sender จะ overwrite message เก่า ครั้งต่อไปที่ receiver เรียก `recv()` จะได้ `Err(RecvError::Lagged(n))` โดย `n` คือจำนวน messages ที่ miss ไป ในโปรเจคนี้เราแจ้งเตือน client แต่ไม่ kick ออก — แต่ใน production อาจ kick client ที่ lag เกินกำหนดได้

---

### ขั้นที่ 9: Main Entry Point และ Graceful Cleanup

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    let addr = "127.0.0.1:7878";
    let listener = TcpListener::bind(addr).await.unwrap();
    println!("TCP Chat Server listening on {}", addr);

    let state = AppState::new();

    loop {
        let (socket, peer) = listener.accept().await.unwrap();
        println!("New connection from {}", peer);
        let state = state.clone();
        tokio::spawn(async move {
            handle_client(socket, state).await;
        });
    }
}
```

**Graceful disconnect cleanup** ใน `handle_client` หลัง reader loop break:

```rust
// cleanup เมื่อ client disconnect
if let Some(room) = &client.current_room {
    state.rooms.leave_room(room, &client.id);
    state.rooms.broadcast(
        room,
        ChatMessage::Leave {
            room: room.clone(),
            nickname: client.nickname.clone(),
        },
    );
}
state.nicks.remove(&client.id);
println!("Client '{}' disconnected", client.nickname);
```

cleanup สำคัญมาก เพราะถ้าไม่ remove nickname จาก store ชื่อนั้นจะ "ถูกจอง" ค้างไว้ตลอดกาล ทำให้ user คนเดิมไม่สามารถกลับมา reconnect ด้วยชื่อเดิมได้

---

## การทดสอบ (Testing)

### Unit Tests ที่เขียนในโปรเจค

Tests แบ่งเป็น 4 module — protocol, room, dm, nickname — รวม 24 tests ที่ครอบคลุม:

**Protocol tests (8 tests):**
- `test_serialize_message` — ตรวจสอบ JSON structure
- `test_deserialize_join` — parse JSON line เป็น enum
- `test_deserialize_private_message` — destructure fields
- `test_roundtrip_error` — serialize แล้ว deserialize กลับได้เหมือนเดิม
- `test_roundtrip_list_rooms` — variant ไม่มี data
- `test_roundtrip_rename` — variant มี data
- `test_invalid_json_returns_error` — error handling
- `test_room_list_roundtrip` — nested Vec

**Room tests (7 tests):**
- `test_new_manager_has_general_room` — default room
- `test_join_creates_room` — auto-create on join
- `test_join_adds_member` — member tracking
- `test_leave_removes_member` — member cleanup
- `test_leave_returns_true_when_empty` — empty room detection
- `test_history_trimming` — MAX_HISTORY enforcement
- `test_broadcast_delivery` — `try_recv` ตรวจสอบ message received
- `test_list_rooms` — รายการห้องถูกต้อง

**DM tests (3 tests):**
- `test_dm_store_and_retrieve` — store messages แล้ว retrieve ได้
- `test_dm_canonical_key` — order ของ key ไม่ affect result
- `test_dm_empty_history` — empty case

**Nickname tests (5 tests):**
- `test_register_new_nick` — register สำเร็จ
- `test_duplicate_nick_rejected` — uniqueness enforcement
- `test_rename_success` — เปลี่ยนชื่อ + ปล่อย old nick
- `test_rename_to_taken_nick_rejected` — rename กันเอง
- `test_remove_frees_nick` — disconnect cleanup

### ผลลัพธ์ cargo test จริง

```
running 24 tests
test dm::tests::test_dm_empty_history ... ok
test dm::tests::test_dm_canonical_key ... ok
test nickname::tests::test_duplicate_nick_rejected ... ok
test dm::tests::test_dm_store_and_retrieve ... ok
test nickname::tests::test_rename_success ... ok
test nickname::tests::test_register_new_nick ... ok
test nickname::tests::test_remove_frees_nick ... ok
test nickname::tests::test_rename_to_taken_nick_rejected ... ok
test protocol::tests::test_deserialize_private_message ... ok
test protocol::tests::test_deserialize_join ... ok
test protocol::tests::test_room_list_roundtrip ... ok
test protocol::tests::test_roundtrip_list_rooms ... ok
test protocol::tests::test_roundtrip_rename ... ok
test protocol::tests::test_serialize_message ... ok
test protocol::tests::test_invalid_json_returns_error ... ok
test protocol::tests::test_roundtrip_error ... ok
test room::tests::test_broadcast_delivery ... ok
test room::tests::test_join_adds_member ... ok
test room::tests::test_history_trimming ... ok
test room::tests::test_join_creates_room ... ok
test room::tests::test_leave_removes_member ... ok
test room::tests::test_leave_returns_true_when_empty ... ok
test room::tests::test_list_rooms ... ok
test room::tests::test_new_manager_has_general_room ... ok

test result: ok. 24 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

### Integration Test ด้วย nc / telnet

เมื่อ start server (`cargo run`) ทดสอบได้ด้วย:

```bash
# Terminal 1 — Alice
nc 127.0.0.1 7878
Welcome! Please set your nickname with: NICK <name>
NICK alice
{"type":"Ok","data":{"message":"Nickname set to 'alice'"}}
/join general
{"type":"Ok","data":{"message":"Joined #general"}}
hello everyone
/rooms
{"type":"RoomList","data":{"rooms":[{"name":"general","member_count":1}]}}
```

```bash
# Terminal 2 — Bob
nc 127.0.0.1 7878
NICK bob
{"type":"Ok","data":{"message":"Nickname set to 'bob'"}}
/join general
{"type":"Ok","data":{"message":"Joined #general"}}
# ตอนนี้ Terminal 1 ได้รับ:
# {"type":"UserJoined","data":{"room":"general","nickname":"bob"}}
/msg alice hi there!
{"type":"Ok","data":{"message":"DM sent to alice"}}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
ls -lh target/release/tcp-chat
# -rwxr-xr-x ... 2.8M target/release/tcp-chat
```

### Dockerfile

```dockerfile
# Builder stage
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release

# Runtime stage — image เล็กลงมาก
FROM debian:bookworm-slim
COPY --from=builder /app/target/release/tcp-chat /usr/local/bin/tcp-chat
EXPOSE 7878
CMD ["tcp-chat"]
```

### Environment Variables

```bash
# กำหนด port และ address ผ่าน env
TCP_CHAT_ADDR=0.0.0.0:7878 cargo run --release

# ใน main.rs
let addr = std::env::var("TCP_CHAT_ADDR")
    .unwrap_or_else(|_| "127.0.0.1:7878".to_string());
```

### systemd Service

```ini
[Unit]
Description=TCP Chat Server
After=network.target

[Service]
ExecStart=/usr/local/bin/tcp-chat
Restart=always
RestartSec=5
Environment=TCP_CHAT_ADDR=0.0.0.0:7878

[Install]
WantedBy=multi-user.target
```

---

## Pitfalls ที่พบบ่อย

### Pitfall 1: ลืม cleanup nickname เมื่อ disconnect

**ปัญหา:** ถ้า `handle_client` return โดยไม่ call `nicks.remove()` nickname ของ client จะค้างอยู่ใน store ตลอดไป ทำให้:
- user reconnect ด้วยชื่อเดิมไม่ได้ (ได้ `NICK_TAKEN` error)
- `list_users` แสดง ghost users ที่ disconnect ไปแล้ว

**วิธีแก้:** ใส่ cleanup ในทุก exit path รวมถึง error cases:

```rust
// Bad — cleanup อยู่แค่ happy path
async fn handle_client(socket: TcpStream, state: Arc<AppState>) {
    // ... ถ้า auth fail แล้ว return ก่อน cleanup ไม่ถูก call
    state.nicks.remove(&client_id); // ✗ อาจไม่ถูก reach
}

// Good — ใช้ scopeguard หรือ RAII wrapper
// หรือใส่ cleanup ก่อน return ทุกจุด
```

ทางเลือกที่ดีกว่าคือสร้าง `ClientGuard` struct ที่ implement `Drop` เพื่อ auto-cleanup:

```rust
struct ClientGuard<'a> {
    id: ClientId,
    state: &'a AppState,
}

impl Drop for ClientGuard<'_> {
    fn drop(&mut self) {
        self.state.nicks.remove(&self.id);
    }
}
```

---

### Pitfall 2: DashMap deadlock จาก nested borrow

**ปัญหา:** การ hold `DashMap` entry (`.get()` หรือ `.get_mut()`) แล้วพยายาม borrow entry อื่นใน map เดียวกันทำให้ deadlock เพราะ `DashMap` ใช้ per-shard lock

```rust
// BAD — deadlock!
let room1 = self.rooms.get_mut("room1").unwrap();  // lock shard A
let room2 = self.rooms.get_mut("room2").unwrap();   // อาจ lock shard A อีกครั้ง
room1.members.insert("client1".to_string());
```

**วิธีแก้:** drop entry ก่อน borrow entry อื่น:

```rust
// Good — drop ก่อน borrow ครั้งที่สอง
{
    let mut room1 = self.rooms.get_mut("room1").unwrap();
    room1.members.insert("client1".to_string());
} // room1 dropped ที่นี่ — lock released
let mut room2 = self.rooms.get_mut("room2").unwrap(); // OK
```

---

### Pitfall 3: broadcast::Receiver drop ทำให้ miss messages

**ปัญหา:** `broadcast::Receiver` ที่ไม่ถูก spawn เข้า task จะ drop ทันที และ miss ทุก message ที่ส่งมาหลังจากนั้น

```rust
// BAD — receiver ถูก drop ทันที ไม่มีใครรับ broadcast
let _rx = state.rooms.join_room(room_name, &client.id);
// _rx drop ที่นี่!
```

**วิธีแก้:** spawn task ที่รับ receiver ทันที และส่งต่อผ่าน mpsc ไปยัง writer:

```rust
// Good — move receiver เข้า task
let mut rx = state.rooms.join_room(room_name, &client.id);
let tx_clone = tx.clone();
tokio::spawn(async move {
    loop {
        match rx.recv().await {
            Ok(msg) => { let _ = tx_clone.send(msg); }
            Err(_) => break,
        }
    }
});
```

---

### Pitfall 4: `splitn` ไม่เพียงพอสำหรับ command parsing

**ปัญหา:** ใช้ `split_whitespace` แบบ naive ทำให้ text ที่มี spaces โดนตัดผิด:

```rust
// BAD
let parts: Vec<&str> = line.split_whitespace().collect();
// "/msg alice hello world" → ["msg", "alice", "hello", "world"]
// parts[2] = "hello" ไม่ใช่ "hello world"!
```

**วิธีแก้:** ใช้ `splitn(3, ' ')` เพื่อ limit จำนวน split:

```rust
// Good
let parts: Vec<&str> = line.splitn(3, ' ').collect();
// "/msg alice hello world" → ["/msg", "alice", "hello world"]
// parts[2] = "hello world" ✓
```

---

### Pitfall 5: ไม่ handle `TcpListener::accept` error

**ปัญหา:** ถ้า `accept()` fail server จะ panic และ crash ทั้งหมด

```rust
// BAD
let (socket, peer) = listener.accept().await.unwrap();
```

**วิธีแก้:** handle error และ log แล้วดำเนินต่อ:

```rust
// Good
loop {
    match listener.accept().await {
        Ok((socket, peer)) => {
            println!("New connection from {}", peer);
            let state = state.clone();
            tokio::spawn(async move {
                handle_client(socket, state).await;
            });
        }
        Err(e) => {
            eprintln!("Accept error: {}", e);
            // อาจ sleep เล็กน้อยถ้า error เป็น resource exhaustion
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;
        }
    }
}
```

---

### Pitfall 6: ไม่ตรวจสอบ nickname ว่า valid

**ปัญหา:** client อาจส่ง nickname ที่มี space, `/`, หรือ special characters ที่ break command parsing:

```
NICK /badnick     <- ชื่อขึ้นต้นด้วย /
NICK alice bob    <- มี space ทำให้ split เป็น "alice" กับ "bob"
NICK              <- empty
```

**วิธีแก้:** validate nickname ก่อน register:

```rust
fn is_valid_nick(nick: &str) -> bool {
    !nick.is_empty()
        && nick.len() <= 32
        && !nick.starts_with('/')
        && nick.chars().all(|c| c.is_alphanumeric() || c == '_' || c == '-')
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: TLS Encryption ด้วย `tokio-rustls`

เพิ่ม TLS layer เพื่อ encrypt traffic:

1. เพิ่ม dependency `tokio-rustls = "0.24"` และ `rustls = "0.21"`
2. สร้าง self-signed cert ด้วย `rcgen`
3. Wrap `TcpStream` ด้วย `TlsAcceptor` ใน acceptor loop
4. Client ต้องใช้ `TlsConnector` แทน plain TCP

**Learning:** TLS handshake flow, certificate loading, async TLS streams

---

### แบบฝึกหัดที่ 2: Persistent History ด้วย SQLite

แทนที่ `VecDeque` ด้วย SQLite เพื่อเก็บ history ข้าม server restart:

1. เพิ่ม `sqlx = { version = "0.7", features = ["sqlite", "runtime-tokio"] }`
2. สร้าง table `messages(id, room, from_nick, text, created_at)`
3. แทน `push_history()` ด้วย async INSERT
4. เมื่อ client join ห้องให้ query 50 messages ล่าสุดและส่งให้ทันที

**Learning:** async SQLite กับ sqlx, connection pooling, migration

---

### แบบฝึกหัดที่ 3: Rate Limiting per Client

ป้องกัน spam โดย limit message rate:

1. เพิ่ม `last_message_at: Instant` และ `message_count: u32` ใน `ClientState`
2. ถ้า client ส่งเกิน 10 messages ใน 1 วินาทีให้ drop และส่ง warning
3. หรือใช้ `governor` crate ที่ implement token bucket algorithm

```rust
use governor::{Quota, RateLimiter};
use std::num::NonZeroU32;

let quota = Quota::per_second(NonZeroU32::new(10).unwrap());
let limiter = RateLimiter::direct(quota);
// ใน message handler:
if limiter.check().is_err() {
    let _ = tx.send(ChatMessage::Error {
        code: "RATE_LIMITED".to_string(),
        message: "Too many messages".to_string(),
    });
    continue;
}
```

**Learning:** token bucket algorithm, `governor` crate, per-connection state

---

### แบบฝึกหัดที่ 4: Admin Commands และ Moderation

เพิ่ม role system สำหรับ admin:

1. เพิ่ม `is_admin: bool` ใน `ClientState` — set จาก password หรือ config
2. เพิ่ม command `/kick <nick>` — force disconnect user
3. เพิ่ม command `/ban <nick>` — block IP/nickname จาก reconnecting
4. เพิ่ม command `/topic <room> <text>` — set room topic
5. Broadcast `SystemMessage` เมื่อ admin action เกิดขึ้น

**Learning:** role-based access control, force disconnect technique, broadcast system messages

---

### แบบฝึกหัดที่ 5: HTTP REST API สำหรับ Room Management

เพิ่ม Axum HTTP server คู่กับ TCP server:

```rust
use axum::{Router, routing::get, Json, extract::State};

async fn list_rooms_api(
    State(state): State<Arc<AppState>>,
) -> Json<Vec<RoomInfo>> {
    Json(state.rooms.list_rooms())
}

// ใน main() รัน TCP และ HTTP พร้อมกัน
tokio::join!(
    run_tcp_server(state.clone()),
    run_http_server(state.clone()),
);
```

**Learning:** multi-protocol server, `tokio::join!`, shared state ข้าม protocols

---

### แบบฝึกหัดที่ 6: WebSocket Gateway

เพิ่ม WebSocket endpoint ที่ bridge ไปยัง TCP server เพื่อให้ web browser เชื่อมต่อได้:

1. เพิ่ม `axum` + `tokio-tungstenite`
2. WebSocket client → JSON frames (เหมือน protocol เดิม)
3. Bridge WebSocket connection เข้า `handle_client` flow เดิม
4. Web UI แบบ simple ด้วย HTML + JavaScript

**Learning:** WebSocket protocol, protocol bridge pattern, reuse server logic

---

## สรุป

โปรเจคนี้สร้าง multi-room TCP chat server ที่สมบูรณ์โดยใช้ pattern สำคัญของ async Rust networking:

**Pattern ที่ได้เรียน:**

1. **JSON line framing** — แก้ปัญหา TCP stream boundaries ด้วยวิธีที่ simple และ debuggable
2. **tokio::io::into_split** — แยก read/write task เพื่อ concurrent I/O บน connection เดียว
3. **mpsc + broadcast channels** — mpsc สำหรับ write-to-client, broadcast สำหรับ room messages
4. **DashMap** — concurrent map ที่ลด lock contention ด้วย sharding
5. **Arc shared state** — pass server state ข้าม spawn tasks โดยไม่ copy
6. **Lagged error handling** — จัดการ slow consumer ใน broadcast channel
7. **Graceful cleanup** — cleanup state อย่างครบถ้วนเมื่อ client disconnect
8. **Bi-directional mapping** — สอง DashMap ทำ O(1) lookup ทั้ง forward และ reverse

**ความแตกต่างจาก production IRC/Discord:**

| Feature | โปรเจคนี้ | Production |
|---------|-----------|------------|
| Protocol | JSON lines | Binary framing (IRC, QUIC) |
| Auth | Nickname only | OAuth, tokens, passwords |
| Persistence | In-memory | Database (PostgreSQL) |
| History | 50 messages | Unlimited (object storage) |
| Scale | Single process | Cluster + pub/sub (Redis) |
| TLS | ไม่มี | บังคับ |

โปรเจคถัดไป **H09: QUIC Transport** จะยก networking stack ขึ้นอีกระดับ ด้วย QUIC protocol ที่แก้ปัญหา head-of-line blocking ของ TCP และ 0-RTT connection resumption สำหรับ mobile clients ที่ network เปลี่ยนบ่อย pattern ที่เรียนจาก H08 จะนำมาใช้ซ้ำทั้ง broadcast, shared state, และ connection handler

---

**โปรเจคก่อนหน้า:** [Project H07: gRPC Service](project-h07-grpc-service.md) | **โปรเจคถัดไป:** [Project H09: QUIC Transport](project-h09-quic-transport.md)
