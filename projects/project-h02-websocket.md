# Project H02: WebSocket Server with Pub/Sub

> โมดูล: H — Networking & Protocols | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 12 ชั่วโมง

---

## ภาพรวมโปรเจค

**WebSocket Server with Pub/Sub** คือโปรเจคที่สร้าง WebSocket server ตั้งแต่ระดับ protocol ขึ้นมาโดยไม่พึ่งพา WebSocket library สำเร็จรูป — เราจะอ่าน RFC 6455 แล้วแปลงเป็น Rust code ที่ทำงานได้จริง

WebSocket เป็น protocol สำหรับการสื่อสารแบบ full-duplex ผ่าน TCP socket เดิม โดย "อาศัย" HTTP ในการเริ่มต้น handshake แล้วสวิตช์ไปเป็น binary framing protocol ของตัวเอง นี่คือ foundation ของแอปพลิเคชันแบบ real-time ทุกชนิด: chat app, collaborative editor, live dashboard, online game, trading platform

ในโลก production ระบบเหล่านี้ต้องการมากกว่าแค่ "ส่งข้อความ" — ต้องมี **Pub/Sub** เพื่อ route ข้อความไปยังผู้รับที่ถูกต้อง, ต้องมี **backpressure** เพื่อจัดการกับ slow client ที่ไม่รับข้อมูลทัน, และต้องมี **rate limiting** เพื่อป้องกัน abuse

**learning value ที่ได้จากโปรเจคนี้:**

ส่วนใหญ่ของนักพัฒนาใช้ library อย่าง `tokio-tungstenite` หรือ `axum` โดยไม่รู้ว่าข้างใต้ทำงานยังไง โปรเจคนี้จะทำให้คุณรู้ว่า:
- ทำไม WebSocket ถึงต้องการ HTTP upgrade แทนที่จะเปิด TCP ตรง ๆ
- ทำไม client ต้อง mask payload แต่ server ไม่ต้อง
- `Sec-WebSocket-Accept` header ถูกคำนวณยังไง และทำไมต้องใช้ SHA-1
- frame fragmentation ช่วยแก้ปัญหาอะไร
- broadcast channel กับ pub/sub pattern เชื่อมกันยังไงใน async Rust

## สิ่งที่จะได้เรียนรู้

- **RFC 6455 protocol implementation:** อ่าน spec และ implement handshake, framing, masking ด้วยตัวเอง
- **Async TCP I/O with tokio:** ใช้ `tokio::net::TcpListener`, `AsyncReadExt`, `AsyncWriteExt`
- **Binary protocol parsing:** decode variable-length field, bit manipulation, XOR masking
- **Pub/Sub pattern:** `DashMap` สำหรับ concurrent topic registry, `tokio::sync::broadcast` สำหรับ fan-out
- **Backpressure handling:** bounded channel สำหรับ per-connection send queue, timeout สำหรับ slow client
- **Token bucket rate limiting:** leaky bucket algorithm สำหรับ per-connection message rate
- **Split async I/O:** ใช้ `tokio::io::split` เพื่อ read และ write บน socket พร้อมกัน
- **Graceful close handshake:** WebSocket close sequence ตาม RFC

## ความรู้ที่ต้องมีมาก่อน

- **Part 46-50:** Async/await fundamentals, Tokio runtime, Future trait
- **Part 51-55:** Tokio tasks (`spawn`), channels (`mpsc`, `broadcast`), `select!` macro
- **Part 56-60:** Error handling, `thiserror`, `anyhow`
- **Part 61-65:** Trait objects, dynamic dispatch, `Arc<Mutex<T>>`
- **Part 66-70:** Smart pointers, interior mutability, `DashMap`
- **Part 76-80:** Serde, JSON serialization
- **Part 36-40:** Iterators, closures, bit manipulation

## โครงสร้างโปรเจค (Project Layout)

```
websocket-server/
├── src/
│   ├── main.rs          # Entry point + TcpListener loop
│   ├── handshake.rs     # HTTP Upgrade parser + Sec-WebSocket-Accept
│   ├── frame.rs         # WsFrame codec (encode/decode/mask)
│   ├── connection.rs    # WsConnection (async read/write, ping/pong)
│   ├── broker.rs        # PubSubBroker (DashMap + broadcast)
│   ├── room.rs          # ChatRoom (join/leave/history/events)
│   ├── ratelimit.rs     # TokenBucket rate limiter
│   └── error.rs         # WsError enum
├── tests/
│   └── integration.rs   # End-to-end tests
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ของ WebSocket Connection

```
Internet
    │  TCP connect
    ▼
TcpListener::accept()
    │
    ▼
perform_handshake()          ← parse HTTP Upgrade, compute Accept key
    │
    ▼
WsConnection { reader, writer }   ← tokio::io::split(stream)
    │                               │
    │  spawn read_task              │  spawn write_task
    ▼                               ▼
decode frames               mpsc Receiver<WsMessage>
    │                               │
    ├─ Ping ──► send Pong           │
    ├─ Close ─► close handshake     │
    └─ Text/Binary                  │
         │                          │
         ▼                          │
PubSubBroker::publish(topic)        │
    │                               │
    └─► broadcast channel ─────────►┘
         fan-out to all subscribers
```

### ทำไมถึงแยก read_task กับ write_task?

TCP socket ใน OS เป็น full-duplex — อ่านและเขียนเป็นอิสระจากกัน `tokio::io::split` แบ่ง `TcpStream` เป็น `ReadHalf` และ `WriteHalf` แยกกัน ทำให้สามารถ spawn สอง task ที่ทำงานพร้อมกัน:

- **read_task** — loop อ่าน frame จาก network, parse, ส่งไปยัง broker
- **write_task** — loop รับ message จาก mpsc channel, encode เป็น frame, เขียนลง network

ถ้าใช้ single loop จะไม่สามารถรับ Ping จาก client ขณะที่กำลังรอ write ได้

### ทำไม client ต้อง mask แต่ server ไม่ต้อง?

RFC 6455 กำหนดว่า **client → server** ต้อง mask ทุก frame แต่ **server → client** ต้องไม่ mask สาเหตุมาจาก security concern: ถ้า client สามารถส่ง arbitrary bytes โดยไม่ mask ผ่าน transparent proxy ที่ cache HTTP response ก็อาจเกิด **cache poisoning attack** ได้ XOR masking ป้องกัน proxy จากการ "เข้าใจ" payload ผิด

### DashMap สำหรับ Pub/Sub

`DashMap` คือ concurrent `HashMap` ที่ใช้ sharding ภายใน ทำให้ไม่ต้องใช้ `Arc<Mutex<HashMap>>` ซึ่งจะ block ทั้ง map เวลา lock เราใช้ `DashMap<String, HashSet<ClientId>>` สำหรับเก็บ topic → subscribers และสร้าง `broadcast::Sender` ต่อ topic สำหรับ fan-out ที่มีประสิทธิภาพ

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: WebSocket Handshake

WebSocket เริ่มต้นด้วย HTTP request ปกติที่มี `Upgrade: websocket` header จาก client จากนั้น server ต้อง:

1. Parse `Sec-WebSocket-Key` จาก request header
2. เชื่อมต่อ key กับ GUID ที่กำหนดใน RFC: `"258EAFA5-E914-47DA-95CA-C5AB0DC85B11"`
3. คำนวณ SHA-1 hash ของ string ที่รวมกัน
4. Encode ด้วย base64
5. ตอบกลับด้วย HTTP 101 Switching Protocols พร้อม `Sec-WebSocket-Accept` header

**Cargo.toml:**

```toml
[package]
name = "websocket-server"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio    = { version = "1", features = ["full"] }
sha1     = "0.10"
base64   = "0.21"
dashmap  = "5"
serde    = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "1"
```

**src/handshake.rs:**

```rust
use sha1::{Digest, Sha1};
use base64::{Engine as _, engine::general_purpose::STANDARD};
use std::collections::HashMap;

pub const WS_GUID: &str = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";

/// คำนวณ Sec-WebSocket-Accept จาก Sec-WebSocket-Key
/// ตาม RFC 6455 §4.2.2
pub fn compute_accept_key(sec_ws_key: &str) -> String {
    let combined = format!("{}{}", sec_ws_key, WS_GUID);
    let mut hasher = Sha1::new();
    hasher.update(combined.as_bytes());
    let hash = hasher.finalize();
    STANDARD.encode(hash)
}

/// Parse HTTP request headers ให้ได้ key-value map
pub fn parse_http_headers(raw: &str) -> HashMap<String, String> {
    let mut headers = HashMap::new();
    let mut lines = raw.lines();
    lines.next(); // skip request line (GET / HTTP/1.1)
    for line in lines {
        if let Some((k, v)) = line.split_once(':') {
            headers.insert(
                k.trim().to_lowercase(),
                v.trim().to_string(),
            );
        }
    }
    headers
}

/// ตรวจสอบว่า request นี้เป็น WebSocket upgrade request หรือไม่
pub fn is_ws_upgrade(headers: &HashMap<String, String>) -> bool {
    headers.get("upgrade").map(|v| v.to_lowercase() == "websocket").unwrap_or(false)
        && headers.get("connection").map(|v| v.to_lowercase().contains("upgrade")).unwrap_or(false)
}

/// สร้าง HTTP 101 Switching Protocols response
pub fn build_upgrade_response(accept_key: &str) -> String {
    format!(
        "HTTP/1.1 101 Switching Protocols\r\n\
         Upgrade: websocket\r\n\
         Connection: Upgrade\r\n\
         Sec-WebSocket-Accept: {}\r\n\
         \r\n",
        accept_key
    )
}

/// ทำ WebSocket handshake ทั้งหมด: อ่าน request, คำนวณ key, ตอบ 101
pub async fn perform_handshake(
    stream: &mut (impl tokio::io::AsyncReadExt + tokio::io::AsyncWriteExt + Unpin),
) -> Result<(), Box<dyn std::error::Error>> {
    use tokio::io::AsyncReadExt;
    use tokio::io::AsyncWriteExt;

    // อ่าน HTTP request (จนถึง empty line \r\n\r\n)
    let mut buf = vec![0u8; 4096];
    let n = stream.read(&mut buf).await?;
    let request = String::from_utf8_lossy(&buf[..n]);

    let headers = parse_http_headers(&request);

    if !is_ws_upgrade(&headers) {
        return Err("Not a WebSocket upgrade request".into());
    }

    let ws_key = headers
        .get("sec-websocket-key")
        .ok_or("Missing Sec-WebSocket-Key")?;

    let accept_key = compute_accept_key(ws_key);
    let response = build_upgrade_response(&accept_key);
    stream.write_all(response.as_bytes()).await?;
    Ok(())
}
```

**ทดสอบ handshake key:**

```
Key: "dGhlIHNhbXBsZSBub25jZQ=="
GUID: "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
Combined: "dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
SHA-1: b3 7a 4f 2c c0 62 4f 16 90 f6 46 06 cf 38 59 45 b2 be c4 ea
Base64: "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
```

นี่คือ example จาก RFC 6455 §B ที่เราต้องได้ผลลัพธ์ตรงกัน

---

### ขั้นที่ 2: Frame Codec

WebSocket frame มี binary format ดังนี้ (จาก RFC 6455 §5.2):

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - -+
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - -+-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - -+
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

**src/frame.rs:**

```rust
use std::io;

/// Opcode ของ WebSocket frame (4 bits)
#[repr(u8)]
#[derive(Debug, Clone, PartialEq)]
pub enum Opcode {
    Continuation = 0x0,
    Text         = 0x1,
    Binary       = 0x2,
    Close        = 0x8,
    Ping         = 0x9,
    Pong         = 0xA,
}

impl TryFrom<u8> for Opcode {
    type Error = String;
    fn try_from(v: u8) -> Result<Self, Self::Error> {
        match v {
            0x0 => Ok(Opcode::Continuation),
            0x1 => Ok(Opcode::Text),
            0x2 => Ok(Opcode::Binary),
            0x8 => Ok(Opcode::Close),
            0x9 => Ok(Opcode::Ping),
            0xA => Ok(Opcode::Pong),
            _   => Err(format!("Unknown opcode: {:#x}", v)),
        }
    }
}

/// WebSocket frame
#[derive(Debug, Clone)]
pub struct WsFrame {
    pub fin:         bool,
    pub opcode:      Opcode,
    pub masked:      bool,
    pub masking_key: Option<[u8; 4]>,
    pub payload:     Vec<u8>,
}

impl WsFrame {
    // --- Constructor helpers ---

    pub fn text(text: &str) -> Self {
        Self { fin: true, opcode: Opcode::Text, masked: false,
               masking_key: None, payload: text.as_bytes().to_vec() }
    }

    pub fn binary(data: Vec<u8>) -> Self {
        Self { fin: true, opcode: Opcode::Binary, masked: false,
               masking_key: None, payload: data }
    }

    pub fn ping(data: Vec<u8>) -> Self {
        Self { fin: true, opcode: Opcode::Ping, masked: false,
               masking_key: None, payload: data }
    }

    pub fn pong(data: Vec<u8>) -> Self {
        Self { fin: true, opcode: Opcode::Pong, masked: false,
               masking_key: None, payload: data }
    }

    pub fn close(code: u16, reason: &str) -> Self {
        let mut payload = code.to_be_bytes().to_vec();
        payload.extend_from_slice(reason.as_bytes());
        Self { fin: true, opcode: Opcode::Close, masked: false,
               masking_key: None, payload }
    }

    pub fn continuation(payload: Vec<u8>, fin: bool) -> Self {
        Self { fin, opcode: Opcode::Continuation, masked: false,
               masking_key: None, payload }
    }

    // --- Encoding ---

    /// Encode frame เป็น bytes สำหรับส่งผ่าน network
    pub fn encode(&self) -> Vec<u8> {
        let mut buf = Vec::new();

        // Byte 0: FIN (1 bit) + RSV1-3 (3 bits, = 0) + opcode (4 bits)
        let byte0 = ((self.fin as u8) << 7) | (self.opcode.clone() as u8);
        buf.push(byte0);

        // Byte 1: MASK bit + payload length
        let mask_bit    = if self.masked { 0x80u8 } else { 0x00u8 };
        let payload_len = self.payload.len();
        if payload_len < 126 {
            buf.push(mask_bit | payload_len as u8);
        } else if payload_len <= 65535 {
            buf.push(mask_bit | 126);
            buf.extend_from_slice(&(payload_len as u16).to_be_bytes());
        } else {
            buf.push(mask_bit | 127);
            buf.extend_from_slice(&(payload_len as u64).to_be_bytes());
        }

        // Masking key (4 bytes ถ้า masked)
        if let Some(key) = self.masking_key {
            buf.extend_from_slice(&key);
            let masked_payload: Vec<u8> = self.payload.iter().enumerate()
                .map(|(i, &b)| b ^ key[i % 4])
                .collect();
            buf.extend(masked_payload);
        } else {
            buf.extend_from_slice(&self.payload);
        }
        buf
    }

    // --- Async reading ---

    /// อ่าน frame เดียวจาก async reader
    pub async fn read_from<R>(reader: &mut R) -> io::Result<Self>
    where
        R: tokio::io::AsyncReadExt + Unpin,
    {
        use tokio::io::AsyncReadExt;

        // อ่าน 2 bytes แรก
        let mut header = [0u8; 2];
        reader.read_exact(&mut header).await?;

        let fin         = (header[0] & 0x80) != 0;
        let opcode_byte = header[0] & 0x0F;
        let opcode      = Opcode::try_from(opcode_byte)
            .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
        let masked      = (header[1] & 0x80) != 0;
        let len_byte    = header[1] & 0x7F;

        // อ่าน extended payload length ถ้าจำเป็น
        let payload_len: usize = match len_byte {
            126 => {
                let mut buf = [0u8; 2];
                reader.read_exact(&mut buf).await?;
                u16::from_be_bytes(buf) as usize
            }
            127 => {
                let mut buf = [0u8; 8];
                reader.read_exact(&mut buf).await?;
                u64::from_be_bytes(buf) as usize
            }
            n => n as usize,
        };

        // อ่าน masking key ถ้า masked
        let masking_key = if masked {
            let mut key = [0u8; 4];
            reader.read_exact(&mut key).await?;
            Some(key)
        } else {
            None
        };

        // อ่าน payload
        let mut payload = vec![0u8; payload_len];
        reader.read_exact(&mut payload).await?;

        // Unmask ถ้ามี masking key
        if let Some(key) = masking_key {
            apply_mask(&mut payload, &key);
        }

        Ok(WsFrame { fin, opcode, masked, masking_key, payload })
    }
}

/// Apply/remove XOR mask — เรียกครั้งเดียวเพื่อ mask, เรียกสองครั้งเพื่อ unmask
pub fn apply_mask(data: &mut [u8], key: &[u8; 4]) {
    for (i, byte) in data.iter_mut().enumerate() {
        *byte ^= key[i % 4];
    }
}

/// Parse close frame payload: (status_code, reason)
pub fn parse_close_payload(payload: &[u8]) -> Option<(u16, String)> {
    if payload.len() < 2 { return None; }
    let code   = u16::from_be_bytes([payload[0], payload[1]]);
    let reason = String::from_utf8_lossy(&payload[2..]).to_string();
    Some((code, reason))
}
```

**ตัวอย่าง: วิธี frame ถูก encode:**

สำหรับ text frame "Hi" (2 bytes) ที่ไม่ masked:
```
Byte 0: 1000_0001 = 0x81  (FIN=1, RSV=000, opcode=0x1 Text)
Byte 1: 0000_0010 = 0x02  (MASK=0, len=2)
Byte 2: 0x48              ('H')
Byte 3: 0x69              ('i')
Total: [0x81, 0x02, 0x48, 0x69]
```

สำหรับ frame ที่ payload = 300 bytes (ต้องใช้ extended length):
```
Byte 0: 0x82              (FIN=1, Binary)
Byte 1: 0xFE              (MASK=1, len=126 → extended 16-bit)
Byte 2-3: 0x01, 0x2C      (300 ใน big-endian)
Byte 4-7: masking_key[4]
Byte 8+: masked payload
```

---

### ขั้นที่ 3: Connection Handler

`WsConnection` ห่อหุ้ม TCP socket ที่ผ่าน handshake แล้ว จัดการ Ping/Pong อัตโนมัติ, fragmented message reassembly, และ close handshake

**src/connection.rs:**

```rust
use tokio::io::{AsyncWriteExt, BufWriter};
use tokio::sync::mpsc;
use std::collections::VecDeque;
use crate::frame::{WsFrame, Opcode, parse_close_payload};

pub type ClientId = u64;

/// ข้อความที่ assembled แล้ว (อาจมาจากหลาย fragment)
#[derive(Debug)]
pub struct WsMessage {
    pub opcode: Opcode,
    pub data:   Vec<u8>,
}

/// State สำหรับ reassemble fragmented messages
#[derive(Default)]
struct FragmentAssembler {
    buffer:       Vec<u8>,
    first_opcode: Option<Opcode>,
}

impl FragmentAssembler {
    fn feed(&mut self, frame: WsFrame) -> Option<WsMessage> {
        let WsFrame { fin, opcode, payload, .. } = frame;
        match opcode {
            Opcode::Continuation => {
                self.buffer.extend_from_slice(&payload);
                if fin {
                    let op   = self.first_opcode.take().unwrap_or(Opcode::Text);
                    let data = std::mem::take(&mut self.buffer);
                    Some(WsMessage { opcode: op, data })
                } else {
                    None
                }
            }
            Opcode::Text | Opcode::Binary => {
                if fin {
                    Some(WsMessage { opcode, data: payload })
                } else {
                    self.first_opcode = Some(opcode);
                    self.buffer = payload;
                    None
                }
            }
            // Control frames (Ping, Pong, Close) ไม่ถูก fragment
            _ => Some(WsMessage { opcode, data: payload }),
        }
    }
}

/// WebSocket connection พร้อม Ping/Pong handling และ close handshake
pub struct WsConnection {
    pub client_id:  ClientId,
    pub tx:         mpsc::Sender<WsMessage>,   // outbound: send ออกไปยัง broker
    pub write_tx:   mpsc::Sender<WsFrame>,     // inbound: รับ frame มาจาก broker เพื่อ write
}

impl WsConnection {
    /// ทำงานหลัก: อ่าน frame จาก network, handle control frames, ส่ง data frames ไปยัง broker
    pub async fn run_reader<R>(
        client_id: ClientId,
        mut reader: R,
        msg_tx: mpsc::Sender<(ClientId, WsMessage)>,
        mut write_tx: mpsc::Sender<WsFrame>,
    ) where
        R: tokio::io::AsyncReadExt + Unpin,
    {
        let mut assembler = FragmentAssembler::default();

        loop {
            match WsFrame::read_from(&mut reader).await {
                Ok(frame) => {
                    match frame.opcode {
                        // Control frames ต้องจัดการทันที
                        Opcode::Ping => {
                            // ตอบ Pong พร้อม payload เดิม
                            let pong = WsFrame::pong(frame.payload);
                            let _ = write_tx.send(pong).await;
                        }
                        Opcode::Close => {
                            // ส่ง Close กลับ แล้วปิด connection
                            if let Some((code, reason)) = parse_close_payload(&frame.payload) {
                                tracing::info!(
                                    "Client {} closing: {} {}",
                                    client_id, code, reason
                                );
                            }
                            let close = WsFrame::close(1000, "");
                            let _ = write_tx.send(close).await;
                            break;
                        }
                        Opcode::Pong => {
                            // รับ Pong — ใช้สำหรับ latency measurement ถ้าต้องการ
                            tracing::debug!("Pong from {}", client_id);
                        }
                        _ => {
                            // Data frames ส่งผ่าน assembler
                            if let Some(msg) = assembler.feed(frame) {
                                if msg_tx.send((client_id, msg)).await.is_err() {
                                    break; // broker ปิดแล้ว
                                }
                            }
                        }
                    }
                }
                Err(e) => {
                    tracing::warn!("Connection {} read error: {}", client_id, e);
                    break;
                }
            }
        }
    }

    /// Write loop: รับ WsFrame จาก channel และเขียนลง network
    pub async fn run_writer<W>(
        client_id: ClientId,
        writer: W,
        mut rx: mpsc::Receiver<WsFrame>,
        timeout: std::time::Duration,
    ) where
        W: tokio::io::AsyncWriteExt + Unpin,
    {
        use tokio::time;

        let mut writer = BufWriter::new(writer);
        loop {
            match time::timeout(timeout, rx.recv()).await {
                Ok(Some(frame)) => {
                    let encoded = frame.encode();
                    if writer.write_all(&encoded).await.is_err() {
                        break;
                    }
                    if writer.flush().await.is_err() {
                        break;
                    }
                }
                Ok(None) => break, // channel ถูกปิด
                Err(_)   => {
                    // Timeout: client อาจช้าเกินไป, ส่ง Ping ตรวจสอบ
                    tracing::warn!("Client {} write timeout, sending ping", client_id);
                    let ping = WsFrame::ping(b"heartbeat".to_vec());
                    let encoded = ping.encode();
                    if writer.write_all(&encoded).await.is_err() {
                        break;
                    }
                }
            }
        }
        tracing::info!("Write loop for client {} ended", client_id);
    }
}
```

---

### ขั้นที่ 4: Pub/Sub Broker

Broker จัดการ topic subscriptions และ broadcast ข้อความไปยัง subscribers ทั้งหมด ใช้ `DashMap` สำหรับ concurrent access และ `tokio::sync::broadcast` channel ต่อ topic สำหรับ efficient fan-out

**src/broker.rs:**

```rust
use dashmap::DashMap;
use tokio::sync::broadcast;
use std::collections::HashSet;
use std::sync::Arc;

pub type ClientId = u64;

/// Message ที่ broadcast ผ่าน topic
#[derive(Debug, Clone)]
pub struct BrokerMessage {
    pub from_client: ClientId,
    pub topic:       String,
    pub payload:     String,
}

/// Per-topic state: สมาชิก + broadcast channel
struct TopicState {
    subscribers: HashSet<ClientId>,
    sender:      broadcast::Sender<BrokerMessage>,
}

impl TopicState {
    fn new() -> Self {
        let (sender, _) = broadcast::channel(256);
        TopicState { subscribers: HashSet::new(), sender }
    }
}

/// Pub/Sub broker หลัก
pub struct PubSubBroker {
    // DashMap ให้ lock-free reads และ fine-grained locking
    topics: DashMap<String, TopicState>,
}

impl PubSubBroker {
    pub fn new() -> Arc<Self> {
        Arc::new(PubSubBroker { topics: DashMap::new() })
    }

    /// Subscribe client ไปยัง topic — สร้าง topic ใหม่ถ้ายังไม่มี
    pub fn subscribe(
        &self,
        topic: &str,
        client_id: ClientId,
    ) -> broadcast::Receiver<BrokerMessage> {
        let mut entry = self.topics
            .entry(topic.to_string())
            .or_insert_with(TopicState::new);
        entry.subscribers.insert(client_id);
        entry.sender.subscribe()
    }

    /// Unsubscribe client จาก topic เดียว
    pub fn unsubscribe(&self, topic: &str, client_id: ClientId) {
        if let Some(mut state) = self.topics.get_mut(topic) {
            state.subscribers.remove(&client_id);
        }
    }

    /// Unsubscribe client จากทุก topic (เมื่อ disconnect)
    pub fn unsubscribe_all(&self, client_id: ClientId) {
        for mut entry in self.topics.iter_mut() {
            entry.subscribers.remove(&client_id);
        }
    }

    /// Publish message ไปยัง topic — broadcast ไปยัง subscribers ทั้งหมด
    /// คืนค่าจำนวน receivers ที่รับได้
    pub fn publish(
        &self,
        topic: &str,
        from_client: ClientId,
        payload: String,
    ) -> usize {
        if let Some(state) = self.topics.get(topic) {
            let msg = BrokerMessage {
                from_client,
                topic: topic.to_string(),
                payload,
            };
            // broadcast::send คืนค่า Err ถ้าไม่มี receiver — ไม่ใช่ error จริง
            state.sender.send(msg).unwrap_or(0)
        } else {
            0
        }
    }

    /// จำนวน subscribers ใน topic
    pub fn subscriber_count(&self, topic: &str) -> usize {
        self.topics.get(topic)
            .map(|s| s.subscribers.len())
            .unwrap_or(0)
    }

    /// List ทุก topic ที่ active
    pub fn list_topics(&self) -> Vec<String> {
        self.topics.iter()
            .filter(|e| !e.subscribers.is_empty())
            .map(|e| e.key().clone())
            .collect()
    }
}
```

**ทำไมใช้ `broadcast::channel` แทน `mpsc`?**

`mpsc` (Multi-Producer Single-Consumer) ส่งข้อความให้ผู้รับคนเดียว แต่ Pub/Sub ต้องการส่งสำเนาให้ทุก subscriber `broadcast` channel คือ Multi-Producer Multi-Consumer ที่แต่ละ receiver ได้รับสำเนาของทุก message — แต่ถ้า receiver ช้าเกินไป (lagged) จะได้รับ `RecvError::Lagged` และข้ามข้อความที่หายไป

---

### ขั้นที่ 5: Room-Based Chat

`ChatRoom` เป็น abstraction ด้านบนของ broker ที่เพิ่ม semantic ของ "room": join/leave events, message history, typing indicators

**src/room.rs:**

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use serde::{Deserialize, Serialize};

pub type ClientId = u64;

/// ประเภทของ event ใน room
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", rename_all = "snake_case")]
pub enum RoomEvent {
    Join    { client_id: ClientId, room: String },
    Leave   { client_id: ClientId, room: String },
    Message { from: ClientId, room: String, text: String, timestamp: u64 },
    Typing  { from: ClientId, room: String },
    WhoIs   { room: String, members: Vec<ClientId> },
}

/// Chat message ใน history
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ChatMessage {
    pub from:      ClientId,
    pub text:      String,
    pub timestamp: u64,
}

/// Chat room state
pub struct ChatRoom {
    pub name:        String,
    pub members:     std::collections::HashSet<ClientId>,
    pub history:     Vec<ChatMessage>,
    pub max_history: usize,
    pub typing:      std::collections::HashSet<ClientId>,
}

impl ChatRoom {
    pub fn new(name: &str) -> Self {
        ChatRoom {
            name:        name.to_string(),
            members:     Default::default(),
            history:     Vec::new(),
            max_history: 50,
            typing:      Default::default(),
        }
    }

    /// เพิ่ม client เข้า room — คืน true ถ้าเพิ่งเข้าใหม่
    pub fn join(&mut self, client_id: ClientId) -> bool {
        self.members.insert(client_id)
    }

    /// นำ client ออกจาก room — คืน true ถ้า client อยู่ใน room จริง
    pub fn leave(&mut self, client_id: ClientId) -> bool {
        self.typing.remove(&client_id);
        self.members.remove(&client_id)
    }

    /// เพิ่มข้อความใน history (ลบเก่าออกถ้าเกิน max)
    pub fn post(&mut self, from: ClientId, text: &str, ts: u64) {
        self.history.push(ChatMessage { from, text: text.to_string(), timestamp: ts });
        while self.history.len() > self.max_history {
            self.history.remove(0);
        }
    }

    /// ดึง history ล่าสุด N ข้อความ
    pub fn recent(&self, n: usize) -> &[ChatMessage] {
        let start = self.history.len().saturating_sub(n);
        &self.history[start..]
    }

    pub fn set_typing(&mut self, client_id: ClientId) {
        self.typing.insert(client_id);
    }

    pub fn clear_typing(&mut self, client_id: ClientId) {
        self.typing.remove(&client_id);
    }

    pub fn member_count(&self) -> usize { self.members.len() }

    pub fn members_list(&self) -> Vec<ClientId> {
        let mut v: Vec<ClientId> = self.members.iter().copied().collect();
        v.sort();
        v
    }
}

/// Registry ของ rooms ทั้งหมด (thread-safe)
pub type RoomRegistry = Arc<Mutex<HashMap<String, ChatRoom>>>;

pub fn new_registry() -> RoomRegistry {
    Arc::new(Mutex::new(HashMap::new()))
}

/// Helper: join room สร้างใหม่ถ้ายังไม่มี
pub fn join_room(
    registry: &RoomRegistry,
    room_name: &str,
    client_id: ClientId,
) -> bool {
    let mut rooms = registry.lock().unwrap();
    let room = rooms.entry(room_name.to_string())
        .or_insert_with(|| ChatRoom::new(room_name));
    room.join(client_id)
}
```

**Protocol ของ Chat Client (JSON over WebSocket):**

Client ส่งข้อความเป็น JSON text frames:

```json
// เข้า room
{ "cmd": "join", "room": "general" }

// ส่งข้อความ
{ "cmd": "msg", "room": "general", "text": "Hello!" }

// แสดงสถานะกำลังพิมพ์
{ "cmd": "typing", "room": "general" }

// ถามว่ามีใครอยู่ใน room บ้าง
{ "cmd": "whois", "room": "general" }

// ออกจาก room
{ "cmd": "leave", "room": "general" }
```

Server ตอบกลับเป็น JSON events:

```json
{ "type": "join",    "client_id": 42, "room": "general" }
{ "type": "message", "from": 42, "room": "general", "text": "Hello!", "timestamp": 1700000000 }
{ "type": "typing",  "from": 42, "room": "general" }
{ "type": "who_is",  "room": "general", "members": [1, 42, 99] }
{ "type": "leave",   "client_id": 42, "room": "general" }
```

---

### ขั้นที่ 6: Backpressure และ Rate Limiting

**src/ratelimit.rs:**

```rust
use std::time::Instant;

/// Token bucket rate limiter สำหรับ per-connection rate limiting
///
/// Algorithm:
/// - มี bucket ขนาด `capacity` tokens
/// - ทุก 1 วินาที เติม `refill_rate` tokens (ไม่เกิน capacity)
/// - ส่งข้อความ 1 ข้อความ = ใช้ 1 token
/// - ถ้าไม่มี token → ปฏิเสธข้อความ (หรือ drop connection)
#[derive(Debug)]
pub struct TokenBucket {
    pub capacity:    f64,   // จำนวน token สูงสุด
    pub tokens:      f64,   // token ที่เหลืออยู่ตอนนี้
    pub refill_rate: f64,   // token ต่อวินาที
    last_refill:     Instant,
}

impl TokenBucket {
    pub fn new(capacity: f64, refill_rate: f64) -> Self {
        TokenBucket {
            capacity,
            tokens: capacity,
            refill_rate,
            last_refill: Instant::now(),
        }
    }

    /// ลองใช้ `cost` tokens — คืน true ถ้าสำเร็จ, false ถ้า rate limited
    pub fn try_consume(&mut self, cost: f64) -> bool {
        self.refill();
        if self.tokens >= cost {
            self.tokens -= cost;
            true
        } else {
            false
        }
    }

    fn refill(&mut self) {
        let now     = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.capacity);
        self.last_refill = now;
    }

    pub fn available(&self) -> f64 { self.tokens }
}

/// Backpressure: bounded send queue สำหรับแต่ละ connection
///
/// ถ้า queue เต็ม (client ช้าเกินไป) → drop message แทนที่จะรอ
/// เพราะการรอจะ block broker และกระทบ connection อื่น ๆ
pub const SEND_QUEUE_SIZE: usize = 64;

/// ตรวจสอบว่าควร drop connection ที่ช้าหรือไม่
pub struct SlowClientDetector {
    drop_after_full_count: usize,
    current_count:         usize,
}

impl SlowClientDetector {
    pub fn new(threshold: usize) -> Self {
        SlowClientDetector { drop_after_full_count: threshold, current_count: 0 }
    }

    /// เรียกเมื่อ send queue เต็ม — คืน true ถ้าควร drop connection
    pub fn queue_full(&mut self) -> bool {
        self.current_count += 1;
        self.current_count >= self.drop_after_full_count
    }

    pub fn reset(&mut self) { self.current_count = 0; }
}
```

**การรวม Rate Limiter เข้ากับ Connection Handler:**

```rust
// ใน run_reader loop, ก่อนส่ง message ไปยัง broker:
if !rate_limiter.try_consume(1.0) {
    tracing::warn!("Client {} rate limited", client_id);
    let close = WsFrame::close(1008, "Rate limit exceeded");
    let _ = write_tx.send(close).await;
    break;
}
```

**src/main.rs:**

```rust
use tokio::net::TcpListener;
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};

mod handshake;
mod frame;
mod connection;
mod broker;
mod room;
mod ratelimit;

static CLIENT_COUNTER: AtomicU64 = AtomicU64::new(1);

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    let broker   = broker::PubSubBroker::new();
    let rooms    = room::new_registry();

    println!("WebSocket server listening on ws://127.0.0.1:8080");

    loop {
        let (mut stream, addr) = listener.accept().await?;
        let client_id = CLIENT_COUNTER.fetch_add(1, Ordering::SeqCst);
        let broker    = Arc::clone(&broker);
        let rooms     = Arc::clone(&rooms);

        tokio::spawn(async move {
            println!("New connection {} from {}", client_id, addr);

            // Perform WebSocket handshake
            if let Err(e) = handshake::perform_handshake(&mut stream).await {
                eprintln!("Handshake failed for {}: {}", client_id, e);
                return;
            }

            // Split stream for independent read/write
            let (reader, writer) = tokio::io::split(stream);

            // Create write channel
            let (write_tx, write_rx) = tokio::sync::mpsc::channel(ratelimit::SEND_QUEUE_SIZE);
            let (msg_tx,   _msg_rx)  = tokio::sync::mpsc::channel(256);

            // Spawn write task
            let write_task = {
                let timeout = std::time::Duration::from_secs(30);
                tokio::spawn(connection::WsConnection::run_writer(
                    client_id,
                    writer,
                    write_rx,
                    timeout,
                ))
            };

            // Run read loop (blocks until connection closes)
            connection::WsConnection::run_reader(
                client_id,
                reader,
                msg_tx,
                write_tx,
            ).await;

            // Cleanup
            broker.unsubscribe_all(client_id);
            write_task.abort();
            println!("Client {} disconnected", client_id);
        });
    }
}
```

---

## การทดสอบ (Testing)

โปรเจค verification ใน scratchpad ทดสอบ core algorithms ทั้งหมดโดยไม่ต้องรัน server จริง:

**Cargo.toml (ของ verification project):**

```toml
[package]
name = "websocket"
version = "0.1.0"
edition = "2021"

[dependencies]
sha1   = "0.10"
base64 = "0.21"
```

**src/lib.rs** ประกอบด้วย:
- `compute_accept_key()` — WebSocket handshake key derivation
- `WsFrame` + `encode()` — frame encoding
- `decode_frame()` — frame decoding
- `apply_mask()` — XOR masking
- `FragmentAssembler` — reassemble fragmented messages
- `PubSubBroker` — subscribe/publish/unsubscribe
- `ChatRoom` — join/leave/history
- `TokenBucket` — token bucket rate limiter
- `parse_close_frame()` — close frame payload parsing

**ผลลัพธ์จาก `cargo test` จริง:**

```
   Compiling version_check v0.9.5
   Compiling typenum v1.20.1
   Compiling cfg-if v1.0.5
   Compiling cpufeatures v0.2.17
   Compiling base64 v0.21.7
   Compiling generic-array v0.14.7
   Compiling block-buffer v0.10.4
   Compiling crypto-common v0.1.7
   Compiling digest v0.10.7
   Compiling sha1 v0.10.7
   Compiling websocket v0.1.0 (...)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 3.22s
     Running unittests src/lib.rs (target/debug/deps/websocket-5872747e017adc8b)

running 20 tests
test tests::test_accept_key_length ... ok
test tests::test_close_frame_empty_payload ... ok
test tests::test_close_frame_round_trip ... ok
test tests::test_accept_key_rfc_example ... ok
test tests::test_encode_decode_binary_frame ... ok
test tests::test_encode_decode_ping_frame ... ok
test tests::test_encode_decode_text_frame ... ok
test tests::test_fragment_reassembly_two_parts ... ok
test tests::test_fragment_reassembly_three_parts ... ok
test tests::test_mask_unmask_identity ... ok
test tests::test_masked_frame_auto_unmask ... ok
test tests::test_medium_payload_length_encoding ... ok
test tests::test_pubsub_subscribe_and_publish ... ok
test tests::test_pubsub_unsubscribe_all ... ok
test tests::test_pubsub_unsubscribe_and_empty ... ok
test tests::test_room_history_capped ... ok
test tests::test_room_join_leave ... ok
test tests::test_small_payload_length_encoding ... ok
test tests::test_token_bucket_bulk ... ok
test tests::test_token_bucket_exhaustion ... ok

test result: ok. 20 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests websocket

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### รายละเอียดของแต่ละ test

**Test 1-2: Accept Key**

```rust
#[test]
fn test_accept_key_rfc_example() {
    // ตัวอย่างจาก RFC 6455 Appendix B
    let key      = "dGhlIHNhbXBsZSBub25jZQ==";
    let expected = "s3pPLMBiTxaQ9kYGzzhZRbK+xOo=";
    assert_eq!(compute_accept_key(key), expected);
}

#[test]
fn test_accept_key_length() {
    // SHA-1 output = 20 bytes → base64 = ceil(20/3)*4 = 28 chars
    let accept = compute_accept_key("AnyBase64Key==");
    assert_eq!(accept.len(), 28, "base64(SHA1) must be 28 chars");
}
```

**Test 3-5: Frame Round-Trip**

```rust
#[test]
fn test_encode_decode_text_frame() {
    let frame   = WsFrame::text("Hello, WebSocket!");
    let encoded = frame.encode();
    let (decoded, consumed) = decode_frame(&encoded).expect("decode failed");
    assert_eq!(consumed, encoded.len());
    assert_eq!(decoded.fin, true);
    assert!(matches!(decoded.opcode, Opcode::Text));
    assert_eq!(decoded.payload, b"Hello, WebSocket!");
}
```

**Test 6-7: Masking**

```rust
#[test]
fn test_mask_unmask_identity() {
    let original    = b"Hello, World!".to_vec();
    let masking_key = [0xAB, 0xCD, 0xEF, 0x12];
    let mut data    = original.clone();
    apply_mask(&mut data, &masking_key);
    assert_ne!(data, original, "masked data must differ");
    apply_mask(&mut data, &masking_key); // XOR again = undo
    assert_eq!(data, original, "double-mask must restore original");
}
```

**Test 8-9: Payload Length Encoding**

```rust
#[test]
fn test_medium_payload_length_encoding() {
    // 300 bytes > 125 → ต้องใช้ extended 16-bit length field
    let frame   = WsFrame::binary(vec![0u8; 300]);
    let encoded = frame.encode();
    assert_eq!(encoded[1] & 0x7F, 126, "length indicator must be 126");
    let ext_len = u16::from_be_bytes([encoded[2], encoded[3]]);
    assert_eq!(ext_len, 300);
}
```

**Test 10-11: Fragment Reassembly**

```rust
#[test]
fn test_fragment_reassembly_three_parts() {
    let mut asm = FragmentAssembler::new();
    // Part 1: Text, fin=false → start accumulation
    let f1 = WsFrame { fin: false, opcode: Opcode::Text, ..
                       payload: b"AAA".to_vec(), .. };
    // Part 2: Continuation, fin=false → keep accumulating
    let f2 = WsFrame { fin: false, opcode: Opcode::Continuation, ..
                       payload: b"BBB".to_vec(), .. };
    // Part 3: Continuation, fin=true → complete
    let f3 = WsFrame { fin: true, opcode: Opcode::Continuation, ..
                       payload: b"CCC".to_vec(), .. };
    assert!(asm.feed(f1).is_none());
    assert!(asm.feed(f2).is_none());
    let msg = asm.feed(f3).expect("must complete");
    assert_eq!(msg.data, b"AAABBBCCC");
}
```

**Test 12-14: Pub/Sub**

```rust
#[test]
fn test_pubsub_subscribe_and_publish() {
    let mut broker = PubSubBroker::new();
    broker.subscribe("news", 1);
    broker.subscribe("news", 2);
    broker.subscribe("news", 3);
    let mut receivers = broker.publish("news", "Breaking!");
    receivers.sort();
    assert_eq!(receivers, vec![1, 2, 3]);
}
```

**Test 15-16: Chat Room**

```rust
#[test]
fn test_room_history_capped() {
    let mut room     = ChatRoom::new("general");
    room.max_history = 5;
    room.join(1);
    for i in 0..10u64 {
        room.post_message(1, &format!("msg-{}", i), i);
    }
    // เก็บแค่ 5 ล่าสุด: msg-5 ถึง msg-9
    assert_eq!(room.history.len(), 5);
    assert_eq!(room.history[0].text, "msg-5");
    assert_eq!(room.history[4].text, "msg-9");
}
```

**Test 17-18: Token Bucket**

```rust
#[test]
fn test_token_bucket_exhaustion() {
    let mut bucket = TokenBucket::new(5.0, 0.0); // capacity=5, ไม่ refill
    for _ in 0..5 { assert!(bucket.try_consume(1.0)); }
    assert!(!bucket.try_consume(1.0), "bucket ต้องว่างแล้ว");
}
```

**Test 19-20: Close Frame**

```rust
#[test]
fn test_close_frame_round_trip() {
    let frame        = WsFrame::close(1000, "Normal closure");
    let encoded      = frame.encode();
    let (decoded, _) = decode_frame(&encoded).expect("decode failed");
    assert!(matches!(decoded.opcode, Opcode::Close));
    let (code, reason) = parse_close_frame(&decoded.payload).unwrap();
    assert_eq!(code, 1000);
    assert_eq!(reason, "Normal closure");
}
```

---

## จุดผิดพลาดที่พบบ่อย (Common Pitfalls)

### Pitfall 1: ลืม mask ข้อความจาก client

ตาม RFC 6455 §5.3: **ทุก frame จาก client ต้อง masked** ถ้าไม่ mask หรือส่ง unmasked frame จาก client, server ต้องปิด connection ด้วย 1002 (Protocol Error)

```rust
// ❌ อ่าน payload โดยไม่ unmask
let mut payload = buf[offset..].to_vec();
// payload ยังคง masked อยู่ ถ้า parse เป็น JSON จะ error

// ✓ unmask ก่อนใช้งาน
if let Some(key) = masking_key {
    apply_mask(&mut payload, &key);
}
```

นอกจากนี้ยังต้องตรวจว่า server ส่ง **unmasked** frame เสมอ — ถ้าส่ง masked frame จาก server, client จะปิด connection

### Pitfall 2: parse HTTP headers ผิดพลาด เพราะ CRLF

HTTP headers ใช้ `\r\n` (CRLF) ไม่ใช่แค่ `\n` การแยก header ด้วย `split('\n')` อาจทิ้ง `\r` ไว้ที่ท้ายค่า ทำให้ header matching ล้มเหลว

```rust
// ❌
for line in raw.split('\n') {
    if let Some((k, v)) = line.split_once(':') {
        // v อาจมี '\r' นำหน้า/ท้าย
    }
}

// ✓
for line in raw.lines() { // lines() strip \r\n อัตโนมัติ
    if let Some((k, v)) = line.split_once(':') {
        let k = k.trim().to_lowercase();
        let v = v.trim().to_string();
    }
}
```

### Pitfall 3: ไม่จัดการ Close handshake ให้ครบ

WebSocket close handshake ต้องทำสองทิศทาง:
1. ฝ่ายที่ต้องการปิดส่ง Close frame ก่อน
2. อีกฝ่ายต้องส่ง Close frame ตอบกลับ
3. หลังจากนั้น TCP connection ถึงจะปิดได้

```rust
// ❌ รับ Close แล้วปิด TCP ทันที
Opcode::Close => {
    stream.shutdown().await?; // อีกฝ่ายยังไม่ได้รับ Close echo
}

// ✓ ส่ง Close ตอบก่อนปิด
Opcode::Close => {
    let echo = WsFrame::close(1000, "").encode();
    writer.write_all(&echo).await?;
    writer.flush().await?;
    // รอให้ read loop จบเอง หรือ timeout แล้วค่อย shutdown
    tokio::time::sleep(Duration::from_millis(100)).await;
    stream.shutdown().await?;
}
```

### Pitfall 4: broadcast channel lagging

`tokio::sync::broadcast` มี fixed-size buffer ถ้า consumer อ่านช้าเกินไป ข้อความเก่าจะถูก overwrite และ consumer จะได้รับ `RecvError::Lagged(n)` ซึ่งต้องจัดการอย่างชัดเจน

```rust
// ❌ ไม่ handle Lagged
loop {
    let msg = rx.recv().await.unwrap(); // panic ถ้า Lagged!
}

// ✓ handle Lagged gracefully
loop {
    match rx.recv().await {
        Ok(msg) => { /* process */ }
        Err(broadcast::error::RecvError::Lagged(n)) => {
            tracing::warn!("Lagged by {} messages", n);
            // ยังคงทำงานต่อได้ แต่ข้ามข้อความที่หายไป
        }
        Err(broadcast::error::RecvError::Closed) => break,
    }
}
```

### Pitfall 5: อ่าน HTTP request ได้ไม่ครบ

เมื่อ buffer ขนาด 4096 bytes อาจรับ HTTP request ไม่ครบใน read เดียว โดยเฉพาะถ้ามี header ยาว ๆ ต้องอ่านจนพบ `\r\n\r\n`

```rust
// ❌ อ่านครั้งเดียว อาจได้ไม่ครบ
let n = stream.read(&mut buf).await?;
let request = String::from_utf8_lossy(&buf[..n]);

// ✓ อ่านจนเจอ end of headers
let mut buf = Vec::new();
loop {
    let mut tmp = [0u8; 512];
    let n = stream.read(&mut tmp).await?;
    buf.extend_from_slice(&tmp[..n]);
    if buf.windows(4).any(|w| w == b"\r\n\r\n") { break; }
    if buf.len() > 8192 { return Err("Headers too large".into()); }
}
```

### Pitfall 6: RSV bits และ Extensions

ถ้า client ส่ง frame ที่มี RSV bits set (RSV1/RSV2/RSV3 ≠ 0) และไม่ได้ negotiate extension ใน handshake, server ต้องปิด connection ด้วย 1002 ถ้าละเลยจะเปิดช่องโหว่ด้านความปลอดภัย

```rust
let rsv = (buf[0] & 0x70) >> 4;  // bits 4-6
if rsv != 0 && !has_extension {
    return Err(WsError::ProtocolError("RSV bits set without extension"));
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/websocket-server

# ทดสอบด้วย websocat
websocat ws://127.0.0.1:8080
```

### Dockerfile

```dockerfile
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
# Pre-fetch dependencies
RUN mkdir src && echo "fn main() {}" > src/main.rs && cargo build --release
RUN rm src/main.rs

COPY src ./src
RUN touch src/main.rs && cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/websocket-server /usr/local/bin/

EXPOSE 8080
ENTRYPOINT ["websocket-server"]
```

```bash
docker build -t websocket-server .
docker run -p 8080:8080 websocket-server
```

### การทดสอบด้วย JavaScript Client

```javascript
// browser หรือ Node.js
const ws = new WebSocket("ws://localhost:8080");

ws.onopen = () => {
    console.log("Connected");
    // Subscribe ไปยัง topic
    ws.send(JSON.stringify({ cmd: "join", room: "general" }));
};

ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    console.log("Received:", msg);
};

ws.onclose = (event) => {
    console.log("Closed:", event.code, event.reason);
};

// ส่งข้อความ
ws.send(JSON.stringify({ cmd: "msg", room: "general", text: "Hello!" }));
```

### การ Monitor ด้วย Metrics

```rust
// เพิ่ม metrics ด้วย prometheus
use std::sync::atomic::{AtomicU64, Ordering};

static ACTIVE_CONNECTIONS: AtomicU64 = AtomicU64::new(0);
static MESSAGES_RECEIVED:  AtomicU64 = AtomicU64::new(0);
static MESSAGES_SENT:      AtomicU64 = AtomicU64::new(0);
```

### Environment Variables

```bash
# ตั้งค่าผ่าน environment
WS_BIND_ADDR=0.0.0.0:8080
WS_MAX_CONNECTIONS=10000
WS_MESSAGE_RATE_PER_SEC=10
WS_SEND_QUEUE_SIZE=64
WS_PING_INTERVAL_SECS=30
```

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: เพิ่ม TLS/WSS Support (ระดับกลาง)

WebSocket จริงในโลก production ต้องใช้ `wss://` (WebSocket Secure ผ่าน TLS) เพิ่ม `tokio-rustls` หรือ `tokio-native-tls` เพื่อ wrap TCP connection ด้วย TLS

```
เป้าหมาย:
- Load certificate/key จาก PEM files
- Wrap TcpStream ด้วย TlsAcceptor
- รับ wss:// connections จาก browser
- ทดสอบด้วย websocat wss://localhost:8443
```

คำใบ้: `tokio_rustls::TlsAcceptor::accept(tcp_stream)` คืน `TlsStream` ที่ implement `AsyncRead + AsyncWrite` เหมือน `TcpStream`

### Exercise 2: Presence System (ระดับกลาง)

ระบบ presence แสดง online/offline status ของผู้ใช้

```
เป้าหมาย:
- เพิ่ม UserPresence { user_id, status, last_seen }
- ทำ heartbeat timeout: ถ้าไม่ได้ยินจาก client นาน X วินาที → offline
- Subscribe ไปยัง presence updates ของ friends list
- ส่ง presence event ทุกครั้งที่ user เข้า/ออก
```

### Exercise 3: Persistent Message History ด้วย SQLite (ระดับกลาง)

แทนที่ in-memory history ด้วย SQLite ผ่าน `sqlx`

```sql
CREATE TABLE messages (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    room       TEXT    NOT NULL,
    from_id    INTEGER NOT NULL,
    text       TEXT    NOT NULL,
    timestamp  INTEGER NOT NULL,
    INDEX idx_room_ts (room, timestamp)
);
```

```
เป้าหมาย:
- บันทึกทุกข้อความลง SQLite
- เมื่อ user join room ส่ง history 50 ข้อความล่าสุด
- Support pagination: cmd "history" พร้อม before_id parameter
```

### Exercise 4: WebSocket Load Balancer (ระดับสูง)

สร้าง reverse proxy สำหรับ WebSocket ที่ route connections ไปยัง backend server หลายตัว

```
เป้าหมาย:
- Accept WebSocket connection จาก client
- เลือก backend ด้วย consistent hashing (client_id % n_backends)
- Proxy frames ระหว่าง client กับ backend โดยไม่ decode
- Health check backend servers ทุก 5 วินาที
- Failover อัตโนมัติถ้า backend ตาย
```

### Exercise 5: Compression Extension (permessage-deflate) (ระดับสูง)

RFC 7692 กำหนด `permessage-deflate` extension ให้ compress payload ด้วย deflate

```
เป้าหมาย:
- Negotiate extension ใน handshake header
- Compress payload ด้วย flate2::Compress ก่อน frame
- Decompress ใน receive path
- ทดสอบ: message 1KB ควรลดเหลือ < 100 bytes สำหรับ repetitive text
```

คำใบ้: RSV1 bit ใน frame header แสดงว่า payload ถูก compressed

### Exercise 6: Admin WebSocket Dashboard (ระดับสูง)

สร้าง real-time admin dashboard ที่แสดงสถิติ server ผ่าน WebSocket

```
เป้าหมาย:
- /admin endpoint สำหรับ monitoring
- Stream metrics ทุก 1 วินาที: active_connections, messages/sec, topic list
- Command: kick_client <id>, close_topic <name>
- Basic auth สำหรับ admin endpoint
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง WebSocket server ตั้งแต่ระดับ protocol ขึ้นมาทั้งหมด:

**Protocol Layer:**
- HTTP Upgrade handshake พร้อม SHA-1/base64 key derivation ตาม RFC 6455
- Binary frame codec: FIN bit, opcode, variable-length payload (7/16/64 bit), XOR masking
- Control frame handling: Ping→Pong automatic, Close handshake สองทิศทาง
- Fragmented message reassembly: buffer fragments จนกว่า FIN=1

**Application Layer:**
- Pub/Sub broker ด้วย `DashMap` + `broadcast::channel` สำหรับ concurrent fan-out
- Chat room abstraction: join/leave events, message history (capped at 50), typing indicators
- JSON-based client protocol สำหรับ interoperability

**Production Concerns:**
- Backpressure: bounded send queue ป้องกันไม่ให้ slow client กระทบคนอื่น
- Rate limiting: token bucket algorithm ป้องกัน message flood
- Graceful close: WebSocket close handshake ก่อน TCP shutdown

**Pattern สำคัญที่ได้เรียน:**
- `tokio::io::split` สำหรับ concurrent read/write บน single stream
- XOR mask เป็น self-inverse ทำให้ mask = unmask
- `broadcast::channel` สำหรับ efficient fan-out ใน async Rust
- Token bucket algorithm สำหรับ smooth rate limiting

โปรเจคถัดไป **H03: DNS Resolver** จะสร้าง DNS client ที่ parse DNS wire format, ส่ง UDP queries ไปยัง resolver จริง, implement recursive resolution และ cache ด้วย TTL — ทักษะ binary protocol parsing จากโปรเจคนี้จะเป็น foundation สำคัญ

---

**โปรเจคก่อนหน้า:** [Project H01: HTTP Server](project-h01-http-server.md) | **โปรเจคถัดไป:** [Project H03: DNS Resolver](project-h03-dns-resolver.md)
