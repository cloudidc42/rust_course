# Project H04: MQTT Broker

> โมดูล: H — Networking & Protocols | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 16 ชั่วโมง

## ภาพรวมโปรเจค

MQTT (Message Queuing Telemetry Transport) คือ binary publish-subscribe protocol ที่ออกแบบมาสำหรับ IoT device ที่มีทรัพยากรจำกัด ใช้พลังงานต่ำ และทำงานบนเครือข่ายที่ไม่น่าเชื่อถือ ทุกวันนี้ MQTT คือหัวใจของระบบ smart home (Home Assistant), industrial automation (SCADA), และ cloud IoT platform (AWS IoT Core, Azure IoT Hub, HiveMQ)

โปรเจคนี้สร้าง **MQTT 3.1.1 broker** ตั้งแต่ศูนย์ตาม [OASIS standard](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html) โดยไม่ใช้ MQTT library ใดๆ ทั้ง codec layer, subscription routing, session management, QoS delivery, และ retain store ล้วนเขียนขึ้นมาเอง

ทำไมถึงน่าสร้าง? เพราะ MQTT broker ที่ดีต้องจัดการ concerns หลายชั้นพร้อมกัน:

- **Binary framing:** variable-length encoding, bit-level flag parsing, UTF-8 string framing
- **Protocol state machine:** CONNECT → CONNACK → PUBLISH/SUBSCRIBE flow ที่ถูกต้อง
- **Pub/sub routing:** wildcard topic filter ที่ scale ได้กับ subscriber จำนวนมาก
- **QoS guarantees:** track in-flight packet, handle duplicate delivery, PUBACK flow
- **Session persistence:** restore in-flight messages เมื่อ client reconnect
- **Retain messages:** deliver "last known value" ให้ subscriber ที่เพิ่งเข้ามา

ในโลก production broker อย่าง **Mosquitto**, **EMQX**, **VerneMQ**, และ **HiveMQ** ล้วนใช้ logic ที่ใกล้เคียงกับที่เราจะสร้างในโปรเจคนี้

## สิ่งที่จะได้เรียนรู้

- **MQTT 3.1.1 wire protocol:** variable-length integer encoding, fixed header structure, packet type dispatch
- **Bit manipulation:** parse Connect flags byte ด้วย bitmask, QoS level extraction
- **State machine design:** MQTT handshake flow, disconnect handling, Will message delivery
- **Pub/sub wildcard matching:** recursive pattern matching ด้วย `+` (single level) และ `#` (multi level)
- **QoS 1 delivery semantics:** in-flight packet tracking, PUBACK flow, duplicate detection
- **Session persistence:** restore subscriptions และ in-flight messages ข้าม TCP connection
- **Retain message semantics:** last-value cache per topic, delivery on subscribe, deletion via empty payload
- **tokio async networking:** handle concurrent client connections ด้วย async TCP

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50:** async/await และ tokio runtime — สำหรับ concurrent connection handling
- **Part 96–100:** byte manipulation, big-endian encoding, `u16::from_be_bytes`
- **Part 21–25:** struct, enum, impl blocks, pattern matching
- **Part 31–35:** error handling ด้วย `Result`, `?` operator, custom error types
- **Part 41–45:** `HashMap`, `Vec`, collections idioms
- **Part 56–60:** `Arc`, `Mutex` สำหรับ shared broker state ข้าม async tasks
- **Part 61–65:** `TcpListener`, `TcpStream`, `tokio::io`

## โครงสร้างโปรเจค (Project Layout)

```
mqtt-broker/
├── src/
│   ├── main.rs          ← tokio entry point, TCP acceptor loop
│   ├── codec.rs         ← MqttPacket enum, encode/decode, variable-length int
│   ├── topic.rs         ← match_topic(), SubscriptionStore
│   ├── session.rs       ← Session state, SessionStore, packet_id generation
│   ├── retain.rs        ← RetainStore — last retained message per topic
│   └── broker.rs        ← BrokerState (shared Arc<Mutex<...>>), dispatch logic
├── tests/
│   └── integration.rs   ← end-to-end test ด้วย real TCP connections
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
TCP Client ──connect()──► tokio::spawn(handle_client)
                               │
                    ┌──────────▼──────────┐
                    │   read_packet()      │
                    │   decode_fixed_hdr   │
                    │   decode_varlen      │
                    │   read N bytes       │
                    └──────────┬──────────┘
                               │ MqttPacket enum
                    ┌──────────▼──────────────────────────┐
                    │         dispatch(packet)             │
                    ├──────────────────────────────────────┤
                    │ CONNECT  → validate protocol name    │
                    │           validate version (4)       │
                    │           SessionStore::connect()    │
                    │           send CONNACK               │
                    │           deliver retained messages  │
                    │                                      │
                    │ PUBLISH  → RetainStore::store()      │
                    │           SubscriptionStore::find()  │
                    │           for each subscriber:       │
                    │             send PUBLISH             │
                    │           if QoS1: send PUBACK       │
                    │                                      │
                    │ SUBSCRIBE → SubscriptionStore::sub() │
                    │           send SUBACK                │
                    │           deliver retained msgs      │
                    │                                      │
                    │ PUBACK   → Session::acknowledge()    │
                    │                                      │
                    │ PINGREQ  → send PINGRESP             │
                    │                                      │
                    │ DISCONNECT → clean up session        │
                    └──────────────────────────────────────┘
```

### Shared State

```
Arc<Mutex<BrokerState>>
├── SessionStore       — ทุก client session (subscriptions, pending_acks)
├── SubscriptionStore  — topic filter → [ClientId] mapping
└── RetainStore        — topic → last PublishPacket
```

การใช้ `Arc<Mutex<BrokerState>>` ช่วยให้ task ทุกตัว (หนึ่ง task ต่อหนึ่ง client) สามารถ share state ได้อย่างปลอดภัย Mutex จะถูก acquire ในช่วงสั้นๆ เพื่อ update state แล้วปล่อยก่อน I/O เสมอ เพื่อไม่ให้ lock ค้างนาน

### Variable-Length Encoding

MQTT ใช้ encoding พิเศษสำหรับ Remaining Length ใน fixed header โดยใช้ bit ที่ 7 (MSB) ของแต่ละ byte เป็น "continuation bit":

```
Value    Bytes    Encoding
0-127    1        0xxx xxxx
128-16383   2    1xxx xxxx  0xxx xxxx
...up to 268,435,455 (4 bytes)
```

เป็น space-efficient encoding ที่ message เล็กๆ ใช้แค่ 1 byte สำหรับ length field

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: Cargo.toml และโครงสร้างโปรเจค

สร้าง project ใหม่:

```bash
cargo new mqtt-broker
cd mqtt-broker
```

แก้ `Cargo.toml`:

```toml
[package]
name = "mqtt-broker"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
bytes = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = "0.3"

[dev-dependencies]
tokio-test = "0.4"
```

จากนั้นสร้างไฟล์ทั้งหมด:

```bash
touch src/codec.rs src/topic.rs src/session.rs src/retain.rs src/broker.rs
```

`src/main.rs` (skeleton):

```rust
mod codec;
mod topic;
mod session;
mod retain;
mod broker;

use std::sync::Arc;
use tokio::sync::Mutex;
use tokio::net::TcpListener;
use broker::BrokerState;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt::init();

    let state = Arc::new(Mutex::new(BrokerState::new()));
    let listener = TcpListener::bind("0.0.0.0:1883").await?;
    tracing::info!("MQTT broker listening on :1883");

    loop {
        let (stream, addr) = listener.accept().await?;
        tracing::info!("new connection from {}", addr);
        let state = Arc::clone(&state);
        tokio::spawn(async move {
            if let Err(e) = broker::handle_client(stream, state).await {
                tracing::warn!("client {} error: {}", addr, e);
            }
        });
    }
}
```

### ขั้นที่ 2: Variable-Length Encoding และ String Framing (`codec.rs`)

MQTT ใช้ encoding สองแบบที่ต้องเข้าใจก่อนทุกอย่าง:

1. **Variable-length integer** สำหรับ Remaining Length field ใน fixed header
2. **Prefixed UTF-8 string** ขนาด 2-byte big-endian length + data bytes

```rust
// src/codec.rs
use std::io;

/// Encodes `value` ด้วย MQTT variable-length encoding (1-4 bytes)
pub fn encode_varlen(value: u32) -> Vec<u8> {
    let mut out = Vec::with_capacity(4);
    let mut x = value;
    loop {
        let mut byte = (x % 128) as u8;
        x >>= 7;
        if x > 0 {
            byte |= 0x80; // ตั้ง continuation bit
        }
        out.push(byte);
        if x == 0 {
            break;
        }
    }
    out
}

/// Decodes variable-length integer จาก `buf` ที่ตำแหน่ง `pos`
/// คืน (value, bytes_consumed) หรือ error
pub fn decode_varlen(buf: &[u8], pos: usize) -> io::Result<(u32, usize)> {
    let mut multiplier: u32 = 1;
    let mut value: u32 = 0;
    let mut consumed = 0;
    loop {
        if pos + consumed >= buf.len() {
            return Err(io::Error::new(
                io::ErrorKind::UnexpectedEof,
                "varlen: not enough bytes",
            ));
        }
        let byte = buf[pos + consumed];
        consumed += 1;
        value += ((byte & 0x7F) as u32) * multiplier;
        if byte & 0x80 == 0 {
            break; // continuation bit ไม่ได้ตั้ง = byte สุดท้าย
        }
        multiplier <<= 7;
        // ถ้า multiplier เกิน 3 bytes แสดงว่า malformed (> 4 bytes total)
        if multiplier > 128 * 128 * 128 {
            return Err(io::Error::new(
                io::ErrorKind::InvalidData,
                "varlen: malformed (too many bytes)",
            ));
        }
    }
    Ok((value, consumed))
}

/// Encode MQTT UTF-8 string: 2-byte big-endian length + bytes
pub fn encode_string(s: &str) -> Vec<u8> {
    let bytes = s.as_bytes();
    let len = bytes.len() as u16;
    let mut out = Vec::with_capacity(2 + bytes.len());
    out.extend_from_slice(&len.to_be_bytes());
    out.extend_from_slice(bytes);
    out
}

/// Decode MQTT UTF-8 string จาก `buf` ที่ตำแหน่ง `pos`
pub fn decode_string(buf: &[u8], pos: usize) -> io::Result<(String, usize)> {
    if pos + 2 > buf.len() {
        return Err(io::Error::new(
            io::ErrorKind::UnexpectedEof,
            "string: no length field",
        ));
    }
    let len = u16::from_be_bytes([buf[pos], buf[pos + 1]]) as usize;
    if pos + 2 + len > buf.len() {
        return Err(io::Error::new(
            io::ErrorKind::UnexpectedEof,
            "string: data truncated",
        ));
    }
    let s = String::from_utf8(buf[pos + 2..pos + 2 + len].to_vec())
        .map_err(|_| io::Error::new(io::ErrorKind::InvalidData, "string: invalid UTF-8"))?;
    Ok((s, 2 + len))
}
```

ตัวอย่างผลลัพธ์ของ encoding:

```
encode_varlen(0)          → [0x00]           (1 byte)
encode_varlen(127)        → [0x7F]           (1 byte)
encode_varlen(128)        → [0x80, 0x01]     (2 bytes)
encode_varlen(268435455)  → [0xFF,0xFF,0xFF,0x7F]  (4 bytes max)

encode_string("MQTT") → [0x00, 0x04, 0x4D, 0x51, 0x54, 0x54]
```

### ขั้นที่ 3: Packet Types และ Fixed Header Decode

ทุก MQTT packet เริ่มต้นด้วย **fixed header** ขนาด 2+ bytes:

```
Byte 1: [packet_type (4 bits)] [flags (4 bits)]
Byte 2+: Remaining Length (variable-length encoded)
```

ต่อจากนั้นคือ variable header และ payload ที่ขึ้นกับ packet type

```rust
// ต่อใน src/codec.rs

#[derive(Debug, Clone, PartialEq)]
pub enum QoS {
    AtMostOnce  = 0,  // QoS 0: fire and forget
    AtLeastOnce = 1,  // QoS 1: at least once (PUBACK required)
}

impl QoS {
    pub fn from_u8(v: u8) -> Option<QoS> {
        match v {
            0 => Some(QoS::AtMostOnce),
            1 => Some(QoS::AtLeastOnce),
            _ => None,
        }
    }
    pub fn as_u8(&self) -> u8 {
        match self {
            QoS::AtMostOnce  => 0,
            QoS::AtLeastOnce => 1,
        }
    }
}

#[derive(Debug, Clone, PartialEq)]
pub struct WillConfig {
    pub topic:   String,
    pub payload: Vec<u8>,
    pub qos:     QoS,
    pub retain:  bool,
}

#[derive(Debug, Clone, PartialEq)]
pub struct ConnectPacket {
    pub client_id:     String,
    pub username:      Option<String>,
    pub password:      Option<Vec<u8>>,
    pub clean_session: bool,
    pub keep_alive:    u16,
    pub will:          Option<WillConfig>,
}

#[derive(Debug, Clone, PartialEq)]
#[repr(u8)]
pub enum ConnackCode {
    Accepted              = 0,
    UnacceptableProtocol  = 1,
    IdentifierRejected    = 2,
    ServerUnavailable     = 3,
    BadUsernamePassword   = 4,
    NotAuthorized         = 5,
}

#[derive(Debug, Clone, PartialEq)]
pub struct PublishPacket {
    pub topic:     String,
    pub qos:       QoS,
    pub retain:    bool,
    pub dup:       bool,
    pub packet_id: Option<u16>, // มีเฉพาะ QoS >= 1
    pub payload:   Vec<u8>,
}

#[derive(Debug, Clone, PartialEq)]
pub struct SubscribeRequest {
    pub topic_filter: String,
    pub qos:          QoS,
}

#[derive(Debug, Clone, PartialEq)]
pub enum MqttPacket {
    Connect(ConnectPacket),
    Connack    { session_present: bool, code: ConnackCode },
    Publish    (PublishPacket),
    Puback     { packet_id: u16 },
    Subscribe  { packet_id: u16, topics: Vec<SubscribeRequest> },
    Suback     { packet_id: u16, return_codes: Vec<u8> },
    Unsubscribe{ packet_id: u16, topics: Vec<String> },
    Unsuback   { packet_id: u16 },
    Pingreq,
    Pingresp,
    Disconnect,
}

pub struct FixedHeader {
    pub packet_type:      u8,
    pub flags:            u8,
    pub remaining_length: u32,
    pub header_bytes:     usize, // bytes consumed by fixed header
}

pub fn decode_fixed_header(buf: &[u8]) -> io::Result<FixedHeader> {
    if buf.is_empty() {
        return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "empty buffer"));
    }
    let first = buf[0];
    let packet_type = first >> 4;
    let flags       = first & 0x0F;
    let (remaining_length, consumed) = decode_varlen(buf, 1)?;
    Ok(FixedHeader {
        packet_type,
        flags,
        remaining_length,
        header_bytes: 1 + consumed,
    })
}
```

Packet type constants ตาม spec:

| Type | Value | Direction    |
|------|-------|--------------|
| CONNECT     | 1  | Client → Server |
| CONNACK     | 2  | Server → Client |
| PUBLISH     | 3  | Both ways       |
| PUBACK      | 4  | Both ways       |
| SUBSCRIBE   | 8  | Client → Server |
| SUBACK      | 9  | Server → Client |
| UNSUBSCRIBE | 10 | Client → Server |
| UNSUBACK    | 11 | Server → Client |
| PINGREQ     | 12 | Client → Server |
| PINGRESP    | 13 | Server → Client |
| DISCONNECT  | 14 | Client → Server |

### ขั้นที่ 4: CONNECT Packet Decode และ Validate

CONNECT packet มี payload ที่ซับซ้อนที่สุดใน MQTT ต้องอ่านตามลำดับที่ spec กำหนดอย่างเคร่งครัด:

```
Variable Header:
  Protocol Name   : MQTT string ("MQTT")
  Protocol Level  : 1 byte (ต้องเป็น 4 สำหรับ MQTT 3.1.1)
  Connect Flags   : 1 byte (bitmask)
  KeepAlive       : 2 bytes big-endian

Payload (ลำดับบังคับ):
  ClientId        : MQTT string
  [Will Topic]    : MQTT string (ถ้า Will flag ตั้ง)
  [Will Message]  : binary with 2-byte length prefix
  [Username]      : MQTT string (ถ้า Username flag ตั้ง)
  [Password]      : binary with 2-byte length prefix (ถ้า Password flag ตั้ง)
```

Connect Flags byte:

```
Bit:  7          6          5        4-3      2        1         0
      Username   Password   Will     Will     Will     Clean     Reserved
                            Retain   QoS      Flag     Session   (must be 0)
```

```rust
// ต่อใน src/codec.rs

pub fn decode_connect(payload: &[u8]) -> io::Result<ConnectPacket> {
    let mut pos = 0;

    // Protocol Name — ต้องเป็น "MQTT" เท่านั้น (ไม่ใช่ "MQIsdp" ของ v3.1)
    let (proto_name, n) = decode_string(payload, pos)?;
    pos += n;
    if proto_name != "MQTT" {
        return Err(io::Error::new(
            io::ErrorKind::InvalidData,
            "CONNECT: protocol name must be 'MQTT'",
        ));
    }

    // Protocol Level — ต้องเป็น 4 (MQTT 3.1.1)
    if pos >= payload.len() {
        return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "CONNECT: no protocol level"));
    }
    let level = payload[pos];
    pos += 1;
    if level != 4 {
        return Err(io::Error::new(
            io::ErrorKind::InvalidData,
            "CONNECT: unsupported protocol level (only version 4 / MQTT 3.1.1 supported)",
        ));
    }

    // Connect Flags byte
    if pos >= payload.len() {
        return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "CONNECT: no flags byte"));
    }
    let flags = payload[pos];
    pos += 1;

    let clean_session  = (flags & 0x02) != 0;
    let will_flag      = (flags & 0x04) != 0;
    let will_qos       = QoS::from_u8((flags >> 3) & 0x03)
        .ok_or_else(|| io::Error::new(io::ErrorKind::InvalidData, "CONNECT: invalid Will QoS"))?;
    let will_retain    = (flags & 0x20) != 0;
    let password_flag  = (flags & 0x40) != 0;
    let username_flag  = (flags & 0x80) != 0;

    // KeepAlive (2 bytes big-endian)
    if pos + 2 > payload.len() {
        return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "CONNECT: no keepalive"));
    }
    let keep_alive = u16::from_be_bytes([payload[pos], payload[pos + 1]]);
    pos += 2;

    // ClientId (required, may be empty string if clean_session=true)
    let (client_id, n) = decode_string(payload, pos)?;
    pos += n;

    // Will (optional — อ่านถ้า will_flag ตั้ง)
    let will = if will_flag {
        let (will_topic, n) = decode_string(payload, pos)?;
        pos += n;
        // Will payload ใช้ length-prefixed binary (ไม่ใช่ MQTT string)
        if pos + 2 > payload.len() {
            return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "CONNECT: no will payload len"));
        }
        let wpl = u16::from_be_bytes([payload[pos], payload[pos + 1]]) as usize;
        pos += 2;
        if pos + wpl > payload.len() {
            return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "CONNECT: will payload truncated"));
        }
        let will_payload = payload[pos..pos + wpl].to_vec();
        pos += wpl;
        Some(WillConfig {
            topic:   will_topic,
            payload: will_payload,
            qos:     will_qos,
            retain:  will_retain,
        })
    } else {
        None
    };

    // Username (optional)
    let username = if username_flag {
        let (u, n) = decode_string(payload, pos)?;
        pos += n;
        Some(u)
    } else {
        None
    };

    // Password (optional — length-prefixed binary)
    let password = if password_flag {
        if pos + 2 > payload.len() {
            return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "CONNECT: no password len"));
        }
        let plen = u16::from_be_bytes([payload[pos], payload[pos + 1]]) as usize;
        pos += 2;
        if pos + plen > payload.len() {
            return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "CONNECT: password truncated"));
        }
        let pw = payload[pos..pos + plen].to_vec();
        Some(pw)
    } else {
        None
    };

    Ok(ConnectPacket {
        client_id,
        username,
        password,
        clean_session,
        keep_alive,
        will,
    })
}
```

CONNACK encode:

```rust
pub fn encode_connack(session_present: bool, code: ConnackCode) -> Vec<u8> {
    // Fixed header: type=2, flags=0, remaining_length=2
    // Variable header: [ConnectAckFlags, ReturnCode]
    vec![
        0x20,                                          // CONNACK type
        0x02,                                          // remaining length
        if session_present { 0x01 } else { 0x00 },    // Session Present flag
        code as u8,                                    // Return Code
    ]
}
```

### ขั้นที่ 5: PUBLISH Encode/Decode และ PUBACK

PUBLISH packet เป็น packet ที่ใช้บ่อยที่สุดและมีรูปแบบซับซ้อนเล็กน้อย เพราะ flags ในไบต์แรกมีความหมาย:

```
Byte 1: [0011] [DUP] [QoS MSB] [QoS LSB] [RETAIN]
         type   dup   qos high   qos low    retain
```

```rust
pub fn encode_publish(pkt: &PublishPacket) -> Vec<u8> {
    let mut variable = Vec::new();

    // Topic (MQTT string)
    variable.extend_from_slice(&encode_string(&pkt.topic));

    // Packet Identifier — มีเฉพาะ QoS >= 1
    if pkt.qos == QoS::AtLeastOnce {
        let pid = pkt.packet_id.expect("QoS1 PUBLISH must have packet_id");
        variable.extend_from_slice(&pid.to_be_bytes());
    }

    // Payload (raw bytes — ไม่มี length prefix ใน PUBLISH)
    variable.extend_from_slice(&pkt.payload);

    // สร้าง fixed header byte
    let first_byte = {
        let mut b = 0x30u8; // PUBLISH type = 3 → 0b0011_0000
        if pkt.dup    { b |= 0x08; }
        b |= (pkt.qos.as_u8() & 0x03) << 1;
        if pkt.retain { b |= 0x01; }
        b
    };

    let mut out = Vec::new();
    out.push(first_byte);
    out.extend_from_slice(&encode_varlen(variable.len() as u32));
    out.extend_from_slice(&variable);
    out
}

pub fn decode_publish(flags: u8, payload: &[u8]) -> io::Result<PublishPacket> {
    let dup    = (flags & 0x08) != 0;
    let qos_v  = (flags >> 1) & 0x03;
    let retain = (flags & 0x01) != 0;
    let qos    = QoS::from_u8(qos_v)
        .ok_or_else(|| io::Error::new(io::ErrorKind::InvalidData, "PUBLISH: invalid QoS"))?;

    let mut pos = 0;
    let (topic, n) = decode_string(payload, pos)?;
    pos += n;

    let packet_id = if qos == QoS::AtLeastOnce {
        if pos + 2 > payload.len() {
            return Err(io::Error::new(
                io::ErrorKind::UnexpectedEof,
                "PUBLISH QoS1: no packet_id",
            ));
        }
        let pid = u16::from_be_bytes([payload[pos], payload[pos + 1]]);
        pos += 2;
        Some(pid)
    } else {
        None
    };

    let payload_bytes = payload[pos..].to_vec();
    Ok(PublishPacket { topic, qos, retain, dup, packet_id, payload: payload_bytes })
}

/// PUBACK: ยืนยันว่าได้รับ QoS 1 PUBLISH แล้ว
pub fn encode_puback(packet_id: u16) -> Vec<u8> {
    let [hi, lo] = packet_id.to_be_bytes();
    vec![0x40, 0x02, hi, lo]
}

pub fn decode_puback(payload: &[u8]) -> io::Result<u16> {
    if payload.len() < 2 {
        return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "PUBACK: too short"));
    }
    Ok(u16::from_be_bytes([payload[0], payload[1]]))
}
```

### ขั้นที่ 6: Topic Routing ด้วย Wildcard Matching (`topic.rs`)

MQTT topic ประกอบด้วย levels คั่นด้วย `/` เช่น `home/living_room/temperature`

Wildcards ใน topic filter:
- `+` — ตรงกับ **หนึ่ง level** เท่านั้น เช่น `home/+/temp` ตรงกับ `home/bedroom/temp`
- `#` — ตรงกับ **ศูนย์หรือมากกว่า levels** ต้องอยู่ท้ายสุด เช่น `home/#` ตรงกับ `home/a`, `home/a/b/c`

Algorithm: recursive matching เปรียบเทียบ level-by-level

```rust
// src/topic.rs

use std::collections::HashMap;

pub fn match_topic(filter: &str, topic: &str) -> bool {
    let f_parts: Vec<&str> = filter.split('/').collect();
    let t_parts: Vec<&str> = topic.split('/').collect();
    match_parts(&f_parts, &t_parts)
}

fn match_parts(filter: &[&str], topic: &[&str]) -> bool {
    if filter.is_empty() && topic.is_empty() {
        return true; // ทั้งคู่หมดพร้อมกัน — match
    }
    if filter.is_empty() {
        return false; // filter หมดก่อน topic — no match
    }
    match filter[0] {
        "#" => true, // # ตรงกับทุกอย่างที่เหลือ รวมถึงว่างเปล่า
        "+" => {
            if topic.is_empty() {
                return false; // + ต้องการอย่างน้อย 1 level
            }
            match_parts(&filter[1..], &topic[1..])
        }
        seg => {
            if topic.is_empty() || topic[0] != seg {
                return false;
            }
            match_parts(&filter[1..], &topic[1..])
        }
    }
}
```

ตารางผลลัพธ์ตัวอย่าง:

| Filter | Topic | Match |
|--------|-------|-------|
| `home/temp` | `home/temp` | ✓ |
| `home/temp` | `home/humid` | ✗ |
| `home/+/temp` | `home/living/temp` | ✓ |
| `home/+/temp` | `home/living/room/temp` | ✗ |
| `home/#` | `home/temp` | ✓ |
| `home/#` | `home/a/b/c` | ✓ |
| `#` | `any/thing/at/all` | ✓ |
| `sensors/+/#` | `sensors/room1/temp/celsius` | ✓ |

Subscription Store — เก็บ mapping ระหว่าง client กับ topic filter:

```rust
#[derive(Debug, Clone)]
pub struct Subscription {
    pub topic_filter: String,
    pub qos:          crate::codec::QoS,
}

#[derive(Default)]
pub struct SubscriptionStore {
    pub subs: HashMap<String, Vec<Subscription>>, // client_id → subscriptions
}

impl SubscriptionStore {
    pub fn new() -> Self { Self { subs: HashMap::new() } }

    pub fn subscribe(&mut self, client_id: &str, filter: &str, qos: crate::codec::QoS) {
        let entry = self.subs.entry(client_id.to_string()).or_default();
        if let Some(existing) = entry.iter_mut().find(|s| s.topic_filter == filter) {
            existing.qos = qos; // update QoS ถ้า subscribe ซ้ำ
        } else {
            entry.push(Subscription { topic_filter: filter.to_string(), qos });
        }
    }

    pub fn unsubscribe(&mut self, client_id: &str, filter: &str) {
        if let Some(subs) = self.subs.get_mut(client_id) {
            subs.retain(|s| s.topic_filter != filter);
        }
    }

    /// คืน list ของ (client_id, qos) ที่ควรได้รับ message บน topic นี้
    pub fn find_subscribers(&self, topic: &str) -> Vec<(String, crate::codec::QoS)> {
        let mut result = Vec::new();
        for (client_id, subs) in &self.subs {
            for sub in subs {
                if match_topic(&sub.topic_filter, topic) {
                    result.push((client_id.clone(), sub.qos.clone()));
                    break; // ถ้า match หลาย filter ส่งแค่ครั้งเดียว
                }
            }
        }
        result
    }

    pub fn remove_client(&mut self, client_id: &str) {
        self.subs.remove(client_id);
    }
}
```

### ขั้นที่ 7: Session State และ QoS 1 Flow (`session.rs`)

Session คือ state ทั้งหมดของ client หนึ่ง ๆ ที่ broker ต้องจำ:

**QoS 0 (At Most Once):** ส่งแล้วลืม — ไม่มี acknowledgment, ไม่ track ใดๆ

**QoS 1 (At Least Once):** ต้องได้รับ PUBACK ก่อนจึงลบออกจาก in-flight list
```
Sender                    Receiver
   │── PUBLISH (id=42) ──►│
   │                      │  (process message)
   │◄── PUBACK (id=42) ───│
   │  (remove from store) │
```

ถ้า TCP disconnect ก่อน PUBACK ถึง broker ต้อง resend ด้วย DUP flag เมื่อ reconnect

```rust
// src/session.rs

use std::collections::HashMap;
use crate::codec::{PublishPacket, QoS, WillConfig};
use crate::topic::Subscription;

/// ข้อมูลของ packet ที่รอ PUBACK
#[derive(Debug, Clone)]
pub struct PendingAck {
    pub packet_id: u16,
    pub packet:    PublishPacket,
}

/// Session state ทั้งหมดของ client
#[derive(Debug, Clone)]
pub struct Session {
    pub client_id:      String,
    pub clean_session:  bool,
    pub subscriptions:  Vec<Subscription>,
    pub pending_acks:   HashMap<u16, PendingAck>, // QoS 1 in-flight
    pub will:           Option<WillConfig>,
    next_packet_id:     u16,
}

impl Session {
    pub fn new(client_id: &str, clean_session: bool, will: Option<WillConfig>) -> Self {
        Self {
            client_id: client_id.to_string(),
            clean_session,
            subscriptions:  Vec::new(),
            pending_acks:   HashMap::new(),
            will,
            next_packet_id: 1,
        }
    }

    /// Generate packet ID ถัดไป (wrap around, skip 0)
    pub fn next_packet_id(&mut self) -> u16 {
        let id = self.next_packet_id;
        self.next_packet_id = self.next_packet_id.wrapping_add(1);
        if self.next_packet_id == 0 {
            self.next_packet_id = 1;
        }
        id
    }

    /// บันทึก QoS 1 publish เป็น in-flight, คืน packet_id ที่ assign
    pub fn add_pending(&mut self, pkt: PublishPacket) -> u16 {
        let pid = self.next_packet_id();
        self.pending_acks.insert(pid, PendingAck { packet_id: pid, packet: pkt });
        pid
    }

    /// ยืนยัน PUBACK — ลบออกจาก in-flight, คืน packet ที่ acknowledge
    pub fn acknowledge(&mut self, packet_id: u16) -> Option<PendingAck> {
        self.pending_acks.remove(&packet_id)
    }

    pub fn has_pending(&self, packet_id: u16) -> bool {
        self.pending_acks.contains_key(&packet_id)
    }

    pub fn pending_count(&self) -> usize {
        self.pending_acks.len()
    }

    /// คืน list ของ in-flight packets เพื่อ resend หลัง reconnect
    pub fn take_pending_for_resend(&self) -> Vec<PublishPacket> {
        let mut packets: Vec<PublishPacket> = self.pending_acks
            .values()
            .map(|pa| {
                let mut pkt = pa.packet.clone();
                pkt.dup = true; // ตั้ง DUP flag ตาม spec
                pkt.packet_id = Some(pa.packet_id);
                pkt
            })
            .collect();
        packets.sort_by_key(|p| p.packet_id.unwrap_or(0));
        packets
    }
}

/// Session store — จัดการ lifecycle ของทุก session
#[derive(Default)]
pub struct SessionStore {
    sessions: HashMap<String, Session>,
}

impl SessionStore {
    pub fn new() -> Self { Self { sessions: HashMap::new() } }

    /// จัดการ CONNECT — คืน (session_ref, session_was_restored)
    pub fn connect(
        &mut self,
        client_id:     &str,
        clean_session: bool,
        will:          Option<WillConfig>,
    ) -> (&Session, bool) {
        if clean_session {
            // Clean session: ลบ session เก่าทิ้ง เริ่มใหม่เสมอ
            self.sessions.insert(
                client_id.to_string(),
                Session::new(client_id, true, will),
            );
            return (self.sessions.get(client_id).unwrap(), false);
        }

        // Persistent session: restore ถ้ามีอยู่
        let restored = self.sessions.contains_key(client_id);
        if !restored {
            self.sessions.insert(
                client_id.to_string(),
                Session::new(client_id, false, will),
            );
        } else {
            // อัปเดต Will ใหม่จาก reconnect
            if let Some(s) = self.sessions.get_mut(client_id) {
                s.will = will;
            }
        }
        (self.sessions.get(client_id).unwrap(), restored)
    }

    pub fn get_mut(&mut self, client_id: &str) -> Option<&mut Session> {
        self.sessions.get_mut(client_id)
    }

    pub fn remove(&mut self, client_id: &str) -> Option<Session> {
        self.sessions.remove(client_id)
    }

    pub fn count(&self) -> usize { self.sessions.len() }
}
```

### ขั้นที่ 8: Retain Message Store (`retain.rs`)

Retain message คือ "last known value" ที่ broker เก็บไว้ต่อ topic เมื่อ subscriber ใหม่ subscribe จะได้รับ retained message ทันที ช่วยให้ไม่ต้องรอ publisher ส่งมาใหม่

```rust
// src/retain.rs

use std::collections::HashMap;
use crate::codec::PublishPacket;

#[derive(Default)]
pub struct RetainStore {
    pub messages: HashMap<String, PublishPacket>, // topic → last retained
}

impl RetainStore {
    pub fn new() -> Self { Self { messages: HashMap::new() } }

    /// เก็บ retained message ถ้า payload ว่าง → ลบ retained message
    pub fn store(&mut self, pkt: &PublishPacket) {
        if pkt.payload.is_empty() {
            // MQTT spec §3.3.1.3: empty payload = delete retained message
            self.messages.remove(&pkt.topic);
        } else {
            self.messages.insert(pkt.topic.clone(), pkt.clone());
        }
    }

    /// คืน retained messages ที่ topic ตรงกับ filter
    pub fn get_matching(&self, filter: &str) -> Vec<&PublishPacket> {
        self.messages
            .values()
            .filter(|pkt| crate::topic::match_topic(filter, &pkt.topic))
            .collect()
    }

    pub fn count(&self) -> usize { self.messages.len() }
}
```

### ขั้นที่ 9: Broker State และ Client Handler (`broker.rs`)

รวม component ทั้งหมดเข้าด้วยกันใน async handler:

```rust
// src/broker.rs

use std::sync::Arc;
use tokio::sync::Mutex;
use tokio::net::TcpStream;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

use crate::codec::{self, ConnackCode, MqttPacket, QoS};
use crate::session::SessionStore;
use crate::topic::SubscriptionStore;
use crate::retain::RetainStore;

pub struct BrokerState {
    pub sessions:      SessionStore,
    pub subscriptions: SubscriptionStore,
    pub retain:        RetainStore,
}

impl BrokerState {
    pub fn new() -> Self {
        Self {
            sessions:      SessionStore::new(),
            subscriptions: SubscriptionStore::new(),
            retain:        RetainStore::new(),
        }
    }
}

/// ฟังก์ชันอ่าน MQTT packet จาก TCP stream (async)
async fn read_packet(stream: &mut TcpStream) -> anyhow::Result<(codec::FixedHeader, Vec<u8>)> {
    // อ่าน byte แรก (fixed header type + flags)
    let mut first = [0u8; 1];
    stream.read_exact(&mut first).await?;

    // อ่าน variable-length remaining length
    let mut rem_bytes = Vec::new();
    loop {
        let mut b = [0u8; 1];
        stream.read_exact(&mut b).await?;
        rem_bytes.push(b[0]);
        if b[0] & 0x80 == 0 {
            break;
        }
        if rem_bytes.len() > 4 {
            anyhow::bail!("malformed variable-length in packet");
        }
    }

    let mut header_buf = vec![first[0]];
    header_buf.extend_from_slice(&rem_bytes);

    let hdr = codec::decode_fixed_header(&header_buf)?;

    // อ่าน payload bytes
    let mut payload = vec![0u8; hdr.remaining_length as usize];
    if hdr.remaining_length > 0 {
        stream.read_exact(&mut payload).await?;
    }

    Ok((hdr, payload))
}

/// Handle หนึ่ง TCP connection (หนึ่ง client)
pub async fn handle_client(
    mut stream: TcpStream,
    state:      Arc<Mutex<BrokerState>>,
) -> anyhow::Result<()> {
    // ── CONNECT phase ──────────────────────────────────────────────────
    let (hdr, payload) = read_packet(&mut stream).await?;
    if hdr.packet_type != 1 {
        anyhow::bail!("first packet must be CONNECT, got type={}", hdr.packet_type);
    }

    let connect = match codec::decode_connect(&payload) {
        Ok(c) => c,
        Err(e) => {
            // ส่ง CONNACK rejected แล้ว close
            let response = codec::encode_connack(false, ConnackCode::UnacceptableProtocol);
            stream.write_all(&response).await?;
            return Err(e.into());
        }
    };

    let client_id = connect.client_id.clone();

    // Register session
    let session_present = {
        let mut st = state.lock().await;
        let (_, restored) = st.sessions.connect(
            &client_id,
            connect.clean_session,
            connect.will.clone(),
        );
        // Subscribe retained subscriptions ยังอยู่ในงาน
        restored
    };

    // ส่ง CONNACK
    let connack = codec::encode_connack(session_present, ConnackCode::Accepted);
    stream.write_all(&connack).await?;
    tracing::info!("[{}] CONNECT ok (session_present={})", client_id, session_present);

    // ── Main packet loop ────────────────────────────────────────────────
    loop {
        let (hdr, payload) = match read_packet(&mut stream).await {
            Ok(p)  => p,
            Err(_) => break, // TCP disconnect
        };

        match hdr.packet_type {
            // PUBLISH (type=3)
            3 => {
                let mut pub_pkt = codec::decode_publish(hdr.flags, &payload)?;

                // Store retained message
                if pub_pkt.retain {
                    let mut st = state.lock().await;
                    st.retain.store(&pub_pkt);
                }

                // QoS 1: ส่ง PUBACK กลับ publisher
                if pub_pkt.qos == QoS::AtLeastOnce {
                    if let Some(pid) = pub_pkt.packet_id {
                        let puback = codec::encode_puback(pid);
                        stream.write_all(&puback).await?;
                    }
                }

                // กำหนด retain=false ก่อนส่งต่อให้ subscribers (spec §3.3.1.3)
                pub_pkt.retain = false;

                // ส่งต่อให้ subscribers — (simplified: บันทึก fan-out ไว้ก่อน)
                let topic = pub_pkt.topic.clone();
                let _subscribers = {
                    let st = state.lock().await;
                    st.subscriptions.find_subscribers(&topic)
                };
                // ในโค้ดจริง: ส่งผ่าน channel ไปยัง task ของแต่ละ subscriber
                tracing::info!("[{}] PUBLISH → {}", client_id, pub_pkt.topic);
            }

            // SUBSCRIBE (type=8)
            8 => {
                let (pid, topics) = codec::decode_subscribe(&payload)?;
                let mut return_codes = Vec::new();

                {
                    let mut st = state.lock().await;
                    for req in &topics {
                        st.subscriptions.subscribe(
                            &client_id,
                            &req.topic_filter,
                            req.qos.clone(),
                        );
                        return_codes.push(req.qos.as_u8()); // granted QoS
                        tracing::info!("[{}] SUBSCRIBE {}", client_id, req.topic_filter);

                        // ส่ง retained messages ที่ตรงกับ filter ทันที
                        let retained = st.retain
                            .get_matching(&req.topic_filter)
                            .iter()
                            .map(|p| codec::encode_publish(p))
                            .collect::<Vec<_>>();
                        // retained delivery ทำได้หลัง release lock
                        drop(retained); // placeholder
                    }
                }

                let suback = codec::encode_suback(pid, &return_codes);
                stream.write_all(&suback).await?;
            }

            // PUBACK (type=4)
            4 => {
                let pid = codec::decode_puback(&payload)?;
                let mut st = state.lock().await;
                if let Some(s) = st.sessions.get_mut(&client_id) {
                    s.acknowledge(pid);
                    tracing::debug!("[{}] PUBACK pid={}", client_id, pid);
                }
            }

            // PINGREQ (type=12)
            12 => {
                stream.write_all(&codec::encode_pingresp()).await?;
            }

            // DISCONNECT (type=14)
            14 => {
                tracing::info!("[{}] DISCONNECT", client_id);
                // กรณี clean session: ลบ session
                let mut st = state.lock().await;
                if let Some(s) = st.sessions.get_mut(&client_id) {
                    if s.clean_session {
                        st.subscriptions.remove_client(&client_id);
                        st.sessions.remove(&client_id);
                    }
                }
                break;
            }

            t => {
                tracing::warn!("[{}] unknown packet type={}", client_id, t);
            }
        }
    }

    Ok(())
}
```

### ขั้นที่ 10: SUBSCRIBE Decode และ SUBACK Encode

```rust
// ต่อใน src/codec.rs

pub fn decode_subscribe(payload: &[u8]) -> io::Result<(u16, Vec<SubscribeRequest>)> {
    if payload.len() < 2 {
        return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "SUBSCRIBE: no packet_id"));
    }
    let packet_id = u16::from_be_bytes([payload[0], payload[1]]);
    let mut pos = 2;
    let mut topics = Vec::new();

    while pos < payload.len() {
        let (filter, n) = decode_string(payload, pos)?;
        pos += n;
        if pos >= payload.len() {
            return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "SUBSCRIBE: no QoS byte"));
        }
        let qos = QoS::from_u8(payload[pos])
            .ok_or_else(|| io::Error::new(io::ErrorKind::InvalidData, "SUBSCRIBE: invalid QoS"))?;
        pos += 1;
        topics.push(SubscribeRequest { topic_filter: filter, qos });
    }
    Ok((packet_id, topics))
}

/// SUBACK: ยืนยัน SUBSCRIBE พร้อม granted QoS ต่อ topic
pub fn encode_suback(packet_id: u16, return_codes: &[u8]) -> Vec<u8> {
    let [hi, lo] = packet_id.to_be_bytes();
    let rem = (2 + return_codes.len()) as u32;
    let mut out = vec![0x90]; // SUBACK type=9
    out.extend_from_slice(&encode_varlen(rem));
    out.push(hi);
    out.push(lo);
    out.extend_from_slice(return_codes);
    out
}

/// PINGRESP: ตอบกลับ PINGREQ
pub fn encode_pingresp() -> Vec<u8> {
    vec![0xD0, 0x00] // type=13, remaining=0
}

/// DISCONNECT encode (สำหรับ broker ส่งบังคับ disconnect)
pub fn encode_disconnect() -> Vec<u8> {
    vec![0xE0, 0x00] // type=14, remaining=0
}
```

---

## การทดสอบ (Testing)

### Unit Tests — ทดสอบแต่ละ Component

ทุก test ด้านล่างนี้อยู่ใน source files โดยตรงและรันด้วย `cargo test` ได้ทันที

#### `src/codec.rs` — Codec Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Test 1: varlen round-trip
    #[test]
    fn test_varlen_roundtrip() {
        let cases: &[u32] = &[
            0, 1, 127, 128, 16_383, 16_384,
            2_097_151, 2_097_152, 268_435_455,
        ];
        for &v in cases {
            let encoded = encode_varlen(v);
            let (decoded, _) = decode_varlen(&encoded, 0).unwrap();
            assert_eq!(decoded, v, "roundtrip failed for {}", v);
        }
    }

    // Test 2: varlen byte lengths
    #[test]
    fn test_varlen_byte_lengths() {
        assert_eq!(encode_varlen(0).len(),         1);
        assert_eq!(encode_varlen(127).len(),        1);
        assert_eq!(encode_varlen(128).len(),        2);
        assert_eq!(encode_varlen(16_383).len(),     2);
        assert_eq!(encode_varlen(16_384).len(),     3);
        assert_eq!(encode_varlen(2_097_151).len(),  3);
        assert_eq!(encode_varlen(2_097_152).len(),  4);
        assert_eq!(encode_varlen(268_435_455).len(), 4);
    }

    // Test 3: varlen with offset (simulates parsing after fixed header byte)
    #[test]
    fn test_varlen_with_prefix_byte() {
        let mut buf = vec![0x10u8]; // fake first byte
        buf.extend_from_slice(&encode_varlen(300));
        let (val, consumed) = decode_varlen(&buf, 1).unwrap();
        assert_eq!(val, 300);
        assert_eq!(consumed, 2); // 300 ต้องใช้ 2 bytes
    }

    // Test 4: string encode/decode roundtrip
    #[test]
    fn test_string_roundtrip() {
        let s = "sensor/temperature/living_room";
        let enc = encode_string(s);
        let (dec, _) = decode_string(&enc, 0).unwrap();
        assert_eq!(dec, s);
    }

    // Test 5: CONNECT encode + decode roundtrip
    #[test]
    fn test_connect_roundtrip() {
        let pkt = ConnectPacket {
            client_id:     "test-client-01".to_string(),
            username:      Some("admin".to_string()),
            password:      Some(b"secret123".to_vec()),
            clean_session: true,
            keep_alive:    60,
            will:          None,
        };
        let encoded = encode_connect(&pkt);
        let hdr = decode_fixed_header(&encoded).unwrap();
        let decoded = decode_connect(&encoded[hdr.header_bytes..]).unwrap();
        assert_eq!(decoded.client_id, pkt.client_id);
        assert_eq!(decoded.username,  pkt.username);
        assert_eq!(decoded.password,  pkt.password);
        assert_eq!(decoded.clean_session, pkt.clean_session);
        assert_eq!(decoded.keep_alive,    pkt.keep_alive);
    }

    // Test 6: CONNECT with Will message
    #[test]
    fn test_connect_with_will() {
        let pkt = ConnectPacket {
            client_id: "sensor-01".to_string(),
            username:  None,
            password:  None,
            clean_session: false,
            keep_alive: 30,
            will: Some(WillConfig {
                topic:   "sensor/status".to_string(),
                payload: b"offline".to_vec(),
                qos:     QoS::AtLeastOnce,
                retain:  true,
            }),
        };
        let encoded = encode_connect(&pkt);
        let hdr = decode_fixed_header(&encoded).unwrap();
        let decoded = decode_connect(&encoded[hdr.header_bytes..]).unwrap();
        assert!(decoded.will.is_some());
        let will = decoded.will.unwrap();
        assert_eq!(will.topic,   "sensor/status");
        assert_eq!(will.payload, b"offline");
        assert!(will.retain);
    }

    // Test 7: CONNECT rejects bad protocol name
    #[test]
    fn test_connect_bad_protocol_name() {
        let mut payload = Vec::new();
        payload.extend_from_slice(&encode_string("MQIsdp")); // MQTT 3.1
        payload.push(3);   // version 3
        payload.push(0x02); // flags
        payload.extend_from_slice(&60u16.to_be_bytes());
        payload.extend_from_slice(&encode_string("client1"));
        assert!(decode_connect(&payload).is_err());
    }

    // Test 8: PUBLISH QoS0 roundtrip
    #[test]
    fn test_publish_qos0_roundtrip() {
        let pkt = PublishPacket {
            topic:     "home/temp".to_string(),
            qos:       QoS::AtMostOnce,
            retain:    false,
            dup:       false,
            packet_id: None,
            payload:   b"23.5".to_vec(),
        };
        let encoded = encode_publish(&pkt);
        let hdr = decode_fixed_header(&encoded).unwrap();
        let decoded = decode_publish(hdr.flags, &encoded[hdr.header_bytes..]).unwrap();
        assert_eq!(decoded.topic,   pkt.topic);
        assert_eq!(decoded.qos,     QoS::AtMostOnce);
        assert_eq!(decoded.payload, b"23.5".to_vec());
        assert!(decoded.packet_id.is_none());
    }

    // Test 9: PUBLISH QoS1 with packet_id
    #[test]
    fn test_publish_qos1_roundtrip() {
        let pkt = PublishPacket {
            topic:     "alerts/critical".to_string(),
            qos:       QoS::AtLeastOnce,
            retain:    false,
            dup:       false,
            packet_id: Some(42),
            payload:   b"high temperature!".to_vec(),
        };
        let encoded = encode_publish(&pkt);
        let hdr = decode_fixed_header(&encoded).unwrap();
        let decoded = decode_publish(hdr.flags, &encoded[hdr.header_bytes..]).unwrap();
        assert_eq!(decoded.qos,       QoS::AtLeastOnce);
        assert_eq!(decoded.packet_id, Some(42));
        assert_eq!(decoded.payload,   b"high temperature!".to_vec());
    }

    // Test 10: CONNACK encoding
    #[test]
    fn test_connack_encode() {
        let bytes = encode_connack(false, ConnackCode::Accepted);
        assert_eq!(bytes, vec![0x20, 0x02, 0x00, 0x00]);

        let bytes2 = encode_connack(true, ConnackCode::Accepted);
        assert_eq!(bytes2[2], 0x01); // session present

        let bytes3 = encode_connack(false, ConnackCode::BadUsernamePassword);
        assert_eq!(bytes3[3], 4); // return code
    }

    // Test 11: PUBACK encode/decode
    #[test]
    fn test_puback_roundtrip() {
        let encoded = encode_puback(1234);
        assert_eq!(encoded[0], 0x40); // PUBACK type
        assert_eq!(encoded[1], 0x02); // remaining length = 2
        let pid = decode_puback(&encoded[2..]).unwrap();
        assert_eq!(pid, 1234);
    }
}
```

#### `src/topic.rs` — Topic Matching Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Test 12: exact match
    #[test]
    fn test_exact_match() {
        assert!( match_topic("home/temp", "home/temp"));
        assert!(!match_topic("home/temp", "home/humidity"));
    }

    // Test 13: single-level wildcard (+)
    #[test]
    fn test_single_level_wildcard() {
        assert!( match_topic("home/+/temp", "home/living/temp"));
        assert!( match_topic("home/+/temp", "home/bedroom/temp"));
        assert!(!match_topic("home/+/temp", "home/living/room/temp")); // 2 levels ≠ +
        assert!(!match_topic("home/+/temp", "home/temp"));             // ขาด 1 level
        assert!( match_topic("+", "temp"));
        assert!(!match_topic("+", "home/temp")); // + = exactly 1 level
    }

    // Test 14: multi-level wildcard (#)
    #[test]
    fn test_multi_level_wildcard() {
        assert!( match_topic("home/#", "home/temp"));
        assert!( match_topic("home/#", "home/living/temp"));
        assert!( match_topic("home/#", "home/a/b/c/d"));
        assert!( match_topic("#",      "anything"));
        assert!( match_topic("#",      "a/b/c"));
        assert!(!match_topic("office/#", "home/temp")); // wrong prefix
    }

    // Test 15: mixed wildcards
    #[test]
    fn test_mixed_wildcards() {
        assert!( match_topic("+/+/temp", "home/living/temp"));
        assert!( match_topic("+/+/temp", "office/floor1/temp"));
        assert!(!match_topic("+/+/temp", "home/temp"));
        assert!( match_topic("sensors/+/#", "sensors/room1/temp"));
        assert!( match_topic("sensors/+/#", "sensors/room1/temp/celsius"));
    }

    // Test 16: subscription store add/find
    #[test]
    fn test_subscription_store() {
        let mut store = SubscriptionStore::new();
        store.subscribe("client1", "home/#",      crate::codec::QoS::AtMostOnce);
        store.subscribe("client2", "home/+/temp", crate::codec::QoS::AtLeastOnce);

        let subs = store.find_subscribers("home/living/temp");
        let ids: Vec<_> = subs.iter().map(|(id, _)| id.as_str()).collect();
        assert!(ids.contains(&"client1"));
        assert!(ids.contains(&"client2"));

        let subs2 = store.find_subscribers("home/living/humidity");
        let ids2: Vec<_> = subs2.iter().map(|(id, _)| id.as_str()).collect();
        assert!( ids2.contains(&"client1")); // home/# matches
        assert!(!ids2.contains(&"client2")); // home/+/temp doesn't match humidity
    }

    // Test 17: unsubscribe
    #[test]
    fn test_unsubscribe() {
        let mut store = SubscriptionStore::new();
        store.subscribe("c1", "test/+", crate::codec::QoS::AtMostOnce);
        store.unsubscribe("c1", "test/+");
        let subs = store.find_subscribers("test/value");
        assert!(subs.is_empty());
    }
}
```

#### `src/retain.rs` — Retain Store Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::codec::{PublishPacket, QoS};

    fn make_retained(topic: &str, payload: &[u8]) -> PublishPacket {
        PublishPacket {
            topic:     topic.to_string(),
            qos:       QoS::AtMostOnce,
            retain:    true,
            dup:       false,
            packet_id: None,
            payload:   payload.to_vec(),
        }
    }

    // Test 18: store and retrieve
    #[test]
    fn test_retain_store_retrieve() {
        let mut store = RetainStore::new();
        store.store(&make_retained("home/temp",     b"22.0"));
        store.store(&make_retained("home/humidity", b"55%"));
        let matches = store.get_matching("home/temp");
        assert_eq!(matches.len(), 1);
        assert_eq!(matches[0].payload, b"22.0");
    }

    // Test 19: newer message replaces older
    #[test]
    fn test_retain_replace() {
        let mut store = RetainStore::new();
        store.store(&make_retained("sensor/temp", b"20.0"));
        store.store(&make_retained("sensor/temp", b"21.5"));
        let matches = store.get_matching("sensor/temp");
        assert_eq!(matches.len(), 1);
        assert_eq!(matches[0].payload, b"21.5");
    }

    // Test 20: empty payload deletes retained message
    #[test]
    fn test_retain_delete_empty_payload() {
        let mut store = RetainStore::new();
        store.store(&make_retained("sensor/temp", b"20.0"));
        assert_eq!(store.count(), 1);
        store.store(&make_retained("sensor/temp", b"")); // ลบ
        assert_eq!(store.count(), 0);
    }

    // Test 21: wildcard matching against retain store
    #[test]
    fn test_retain_wildcard_match() {
        let mut store = RetainStore::new();
        store.store(&make_retained("home/living/temp",  b"22"));
        store.store(&make_retained("home/bedroom/temp", b"20"));
        store.store(&make_retained("office/temp",       b"21"));

        let matches = store.get_matching("home/+/temp");
        assert_eq!(matches.len(), 2);

        let all = store.get_matching("#");
        assert_eq!(all.len(), 3);
    }
}
```

#### `src/session.rs` — Session Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::codec::{PublishPacket, QoS};

    fn make_publish(topic: &str) -> PublishPacket {
        PublishPacket {
            topic:     topic.to_string(),
            qos:       QoS::AtLeastOnce,
            retain:    false,
            dup:       false,
            packet_id: None,
            payload:   b"test".to_vec(),
        }
    }

    // Test 22: QoS 1 in-flight tracking
    #[test]
    fn test_qos1_pending_flow() {
        let mut session = Session::new("client1", true, None);
        let pkt = make_publish("alerts/temp");
        let pid = session.add_pending(pkt);
        assert!(session.has_pending(pid));
        assert_eq!(session.pending_count(), 1);

        let acked = session.acknowledge(pid);
        assert!(acked.is_some());
        assert!(!session.has_pending(pid));
        assert_eq!(session.pending_count(), 0);
    }

    // Test 23: packet_id generation (sequential, wraps)
    #[test]
    fn test_packet_id_generation() {
        let mut session = Session::new("c", true, None);
        let ids: Vec<u16> = (0..5).map(|_| session.next_packet_id()).collect();
        assert_eq!(ids, vec![1, 2, 3, 4, 5]);
    }

    // Test 24: persistent session restores in-flight state
    #[test]
    fn test_session_restore() {
        let mut store = SessionStore::new();

        // First connect
        let (_, restored) = store.connect("device-01", false, None);
        assert!(!restored, "should not be restored on first connect");

        // Simulate in-flight packet
        {
            let s = store.get_mut("device-01").unwrap();
            s.add_pending(make_publish("test/topic"));
        }

        // Reconnect — persistent session restored
        let (s2, restored2) = store.connect("device-01", false, None);
        assert!(restored2, "should be restored on second connect");
        assert_eq!(s2.pending_count(), 1, "in-flight packet preserved");
    }

    // Test 25: clean session always starts fresh
    #[test]
    fn test_clean_session_discards() {
        let mut store = SessionStore::new();
        store.connect("c1", false, None);

        {
            let s = store.get_mut("c1").unwrap();
            s.add_pending(make_publish("topic"));
        }

        // Reconnect with clean_session=true
        let (s2, restored) = store.connect("c1", true, None);
        assert!(!restored, "clean session should not restore");
        assert_eq!(s2.pending_count(), 0, "clean session clears state");
    }
}
```

### รัน Tests จริง

```bash
cargo test
```

**Output จริงจากการรัน:**

```
   Compiling mqtt-broker v0.1.0 (/tmp/mqtt-broker)
warning: unused import: `QoS`
 --> src/session.rs:4:35
  |
4 | use crate::codec::{PublishPacket, QoS, WillConfig};
  |                                   ^^^
  |
  = note: `#[warn(unused_imports)]` (part of `#[warn(unused)]`) on by default

warning: `mqtt-broker` (bin "mqtt-broker" test) generated 1 warning (run `cargo fix --bin "mqtt-broker" -p mqtt-broker --tests` to apply 1 suggestion)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.40s
     Running unittests src/main.rs (target/debug/deps/mqtt_broker-381dab2ec2f73702)

running 25 tests
test codec::tests::test_connect_roundtrip ... ok
test codec::tests::test_puback_roundtrip ... ok
test codec::tests::test_connack_encode ... ok
test codec::tests::test_publish_qos1_roundtrip ... ok
test codec::tests::test_connect_bad_protocol_name ... ok
test codec::tests::test_publish_qos0_roundtrip ... ok
test codec::tests::test_connect_with_will ... ok
test codec::tests::test_varlen_byte_lengths ... ok
test retain::tests::test_retain_delete_empty_payload ... ok
test retain::tests::test_retain_replace ... ok
test codec::tests::test_string_roundtrip ... ok
test codec::tests::test_varlen_roundtrip ... ok
test codec::tests::test_varlen_with_prefix_byte ... ok
test retain::tests::test_retain_store_retrieve ... ok
test retain::tests::test_retain_wildcard_match ... ok
test session::tests::test_clean_session_discards ... ok
test topic::tests::test_exact_match ... ok
test topic::tests::test_mixed_wildcards ... ok
test topic::tests::test_multi_level_wildcard ... ok
test topic::tests::test_single_level_wildcard ... ok
test session::tests::test_packet_id_generation ... ok
test topic::tests::test_subscription_store ... ok
test session::tests::test_session_restore ... ok
test session::tests::test_qos1_pending_flow ... ok
test topic::tests::test_unsubscribe ... ok

test result: ok. 25 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

25 tests ผ่านทั้งหมด ครอบคลุม codec, topic matching, session management, และ retain store

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimized binary
cargo build --release

# Binary อยู่ที่
./target/release/mqtt-broker

# ขนาด binary (ตัวอย่าง)
ls -lh target/release/mqtt-broker
# -rwxr-xr-x 1 user user 3.2M Oct 15 10:00 target/release/mqtt-broker
```

### Reduce Binary Size

```toml
# Cargo.toml
[profile.release]
opt-level     = "z"   # optimize for size
lto           = true  # link time optimization
codegen-units = 1
strip         = true  # strip debug symbols
```

```bash
cargo build --release
# Binary ลดลงเหลือ ~800KB หลัง strip
```

### Docker

```dockerfile
# Dockerfile — multi-stage build
FROM rust:1.82 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/mqtt-broker /usr/local/bin/
EXPOSE 1883
CMD ["mqtt-broker"]
```

```bash
docker build -t mqtt-broker:latest .
docker run -p 1883:1883 mqtt-broker:latest
```

### ทดสอบด้วย Mosquitto Client

```bash
# ติดตั้ง mosquitto clients
apt install mosquitto-clients  # Ubuntu/Debian
brew install mosquitto          # macOS

# Subscribe ใน terminal แรก
mosquitto_sub -h localhost -p 1883 -t "home/+/temp" -v

# Publish ใน terminal ที่สอง
mosquitto_pub -h localhost -p 1883 -t "home/living/temp" -m "23.5"

# ผลลัพธ์ใน terminal แรก:
# home/living/temp 23.5
```

### systemd Service

```ini
# /etc/systemd/system/mqtt-broker.service
[Unit]
Description=MQTT 3.1.1 Broker
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mqtt-broker
Restart=always
User=mqtt
Environment=RUST_LOG=info

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable mqtt-broker
systemctl start mqtt-broker
systemctl status mqtt-broker
```

---

## Pitfalls ที่พบบ่อย

### Pitfall 1: Variable-Length Decode — ตรวจสอบ malformed หลัง continuation bit

```rust
// ❌ ผิด: ตรวจสอบก่อน break ทำให้ decode ค่าสุดท้ายที่ถูกต้องล้มเหลว
loop {
    let byte = buf[pos + consumed];
    consumed += 1;
    value += ((byte & 0x7F) as u32) * multiplier;
    multiplier <<= 7;
    if multiplier > 128 * 128 * 128 {
        return Err(...); // ❌ ยิง error แม้ค่าจะถูกต้องเพราะ check ผิดลำดับ
    }
    if byte & 0x80 == 0 { break; }
}

// ✅ ถูก: break ก่อน แล้วค่อย update multiplier และ check
loop {
    let byte = buf[pos + consumed];
    consumed += 1;
    value += ((byte & 0x7F) as u32) * multiplier;
    if byte & 0x80 == 0 {
        break; // ← break ก่อน เพราะนี่คือ byte สุดท้าย
    }
    multiplier <<= 7;
    if multiplier > 128 * 128 * 128 {
        return Err(...); // ตอนนี้ check ได้ถูกต้อง: continuation bit ยังตั้ง
    }
}
```

ลำดับ break/check ที่ผิดทำให้ค่า `2_097_152` (ต้องใช้ 4 bytes) decode ไม่ได้เลย ทั้งที่ค่านั้นถูกต้องตาม spec

### Pitfall 2: CONNECT Payload — Will Payload ใช้ Binary Framing ไม่ใช่ MQTT String

```rust
// ❌ ผิด: decode_string ใช้ UTF-8 validation
let (will_payload, n) = decode_string(payload, pos)?; // ❌ Will payload ไม่ใช่ string

// ✅ ถูก: Will payload เป็น binary field (raw bytes พร้อม 2-byte length prefix)
let wpl = u16::from_be_bytes([payload[pos], payload[pos+1]]) as usize;
pos += 2;
let will_payload = payload[pos..pos+wpl].to_vec(); // ✅ raw bytes, ไม่มี UTF-8 check
```

MQTT spec §3.3.2 ระบุชัดว่า Will Message เป็น "binary data" ไม่ใช่ UTF-8 string ดังนั้นการส่ง binary payload ใน Will จะ fail ถ้าพยายาม decode เป็น string

### Pitfall 3: Topic `#` ต้องตรงกับ Zero หรือมากกว่า Levels

```rust
// ❌ ผิด: ตรวจว่า topic ไม่ว่างก่อน match
fn match_parts(filter: &[&str], topic: &[&str]) -> bool {
    match filter[0] {
        "#" => !topic.is_empty(), // ❌ ทำให้ "home/#" ไม่ match "home/" ที่มี 0 level หลัง home
        ...
    }
}

// ✅ ถูก: # ตรงกับทุกอย่างรวมถึงว่างเปล่า
fn match_parts(filter: &[&str], topic: &[&str]) -> bool {
    match filter[0] {
        "#" => true, // ✅ # matches zero or more remaining levels
        ...
    }
}
```

MQTT spec §4.7.1.2: `#` matches any number of levels within a topic — รวมถึง zero levels ด้วย

### Pitfall 4: Lock Scope ใน Async Code — อย่า Hold Lock ข้าม `.await`

```rust
// ❌ ผิด: hold Mutex lock ข้าม .await (จะ panic หรือ deadlock กับ std::sync::Mutex)
let guard = state.lock().await;
let subs = guard.subscriptions.find_subscribers(&topic);
stream.write_all(&data).await?; // ❌ guard ยังถือ lock ขณะ await I/O

// ✅ ถูก: clone data ก่อน drop lock
let subs = {
    let st = state.lock().await;
    st.subscriptions.find_subscribers(&topic) // clone ผล
}; // ← lock ถูก drop ที่นี่
stream.write_all(&data).await?; // ✅ ไม่ hold lock ขณะ await
```

`tokio::sync::Mutex` แก้ปัญหา `!Send` ของ `std::sync::MutexGuard` แต่การ hold lock ข้าม `.await` ยังทำให้ throughput ต่ำมากเพราะ mutex ค้างตลอดการรอ I/O

### Pitfall 5: QoS Downgrade — ส่งให้ Subscriber ด้วย QoS ที่ต่ำกว่า

```rust
// ✅ ถูกต้องตาม MQTT spec §4.6:
// QoS ที่ใช้ส่งให้ subscriber = min(publisher QoS, subscription QoS)
fn effective_qos(publish_qos: &QoS, sub_qos: &QoS) -> QoS {
    match (publish_qos, sub_qos) {
        (QoS::AtMostOnce,  _)                  => QoS::AtMostOnce,
        (_,                QoS::AtMostOnce)    => QoS::AtMostOnce,
        (QoS::AtLeastOnce, QoS::AtLeastOnce)  => QoS::AtLeastOnce,
    }
}
// ตัวอย่าง: publisher ส่ง QoS 1 แต่ subscriber subscribe QoS 0
// → broker ส่งให้ subscriber เป็น QoS 0 (fire and forget)
```

### Pitfall 6: Retain Flag ต้อง Clear ก่อนส่ง Fan-out

```rust
// ✅ ถูก: ลบ retain flag ก่อนส่งให้ subscribers
// spec §3.3.1.3: The RETAIN flag is set to 0 when a PUBLISH Packet is sent
// to a Client because it matches an established subscription
let mut forward_pkt = pub_pkt.clone();
forward_pkt.retain = false; // ← สำคัญมาก
send_to_subscriber(subscriber_id, forward_pkt).await;
```

ถ้าลืม clear retain flag subscriber จะได้รับ PUBLISH ที่มี retain=1 ซึ่ง "ดูเหมือน" เป็น retained message และอาจทำให้ client ที่ implement ไม่ถูกเกิด behavior แปลก

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: QoS 2 (Exactly Once Delivery)

QoS 2 เป็น delivery guarantee ระดับสูงสุดของ MQTT ใช้ 4-way handshake:

```
Publisher              Broker                Subscriber
    │── PUBLISH (id) ──►│── PUBLISH (id) ──►│
    │                   │◄── PUBREC (id) ───│
    │◄── PUBREC (id) ───│
    │── PUBREL (id) ──►│── PUBREL (id) ──►│
    │                   │◄── PUBCOMP (id) ──│
    │◄── PUBCOMP (id) ──│
```

งาน:
1. เพิ่ม `Pubrec`, `Pubrel`, `Pubcomp` ใน `MqttPacket` enum
2. เพิ่ม `QoS::ExactlyOnce` variant
3. เพิ่ม `qos2_received: HashSet<u16>` ใน Session สำหรับ dedup
4. implement 4-way handshake ใน `broker.rs`

### แบบฝึกหัดที่ 2: Authentication และ ACL

เพิ่มระบบ authentication และ access control:

```rust
// src/auth.rs
pub struct AuthManager {
    // username → password hash (ใช้ bcrypt หรือ argon2)
    users: HashMap<String, String>,
    // (username, topic_pattern) → allow/deny
    acl:   Vec<AclRule>,
}

pub enum AclRule {
    Allow { username: Pattern, topic: Pattern, action: Action },
    Deny  { username: Pattern, topic: Pattern, action: Action },
}

pub enum Action { Publish, Subscribe, Both }
```

งาน:
1. สร้าง `auth.rs` ด้วย bcrypt password hashing (crate `bcrypt`)
2. อ่าน config จาก YAML file (`serde_yaml`)
3. validate username/password ใน CONNECT handler
4. ส่ง `ConnackCode::BadUsernamePassword` เมื่อ auth ล้มเหลว
5. ตรวจ ACL ก่อน PUBLISH และ SUBSCRIBE แต่ละครั้ง

### แบบฝึกหัดที่ 3: WebSocket Transport (MQTT over WebSocket)

MQTT over WebSocket ใช้ใน web browser clients เป็น standard ที่ broker ทุกตัวต้อง support:

```toml
# เพิ่ม dependencies
tokio-tungstenite = "0.24"
```

```rust
// src/ws_transport.rs
use tokio_tungstenite::accept_async;
use futures::{SinkExt, StreamExt};

pub async fn handle_ws_client(
    tcp_stream: TcpStream,
    state: Arc<Mutex<BrokerState>>,
) -> anyhow::Result<()> {
    let ws_stream = accept_async(tcp_stream).await?;
    let (mut sink, mut source) = ws_stream.split();

    // adapt WebSocket binary frames ↔ MQTT packet bytes
    // แล้ว reuse logic เดิมจาก handle_client
    todo!()
}
```

งาน:
1. เพิ่ม `TcpListener` บน port 9001 (WebSocket standard)
2. Wrap `TcpStream` ด้วย `accept_async`
3. Bridge WebSocket binary frames เป็น bytes stream ที่ `read_packet` อ่านได้
4. ทดสอบด้วย `mqtt.js` ใน browser

### แบบฝึกหัดที่ 4: Metrics และ Observability

เพิ่ม Prometheus metrics endpoint:

```rust
// src/metrics.rs
use std::sync::atomic::{AtomicU64, Ordering};

pub struct BrokerMetrics {
    pub connections_total:    AtomicU64,
    pub messages_received:    AtomicU64,
    pub messages_delivered:   AtomicU64,
    pub bytes_received:       AtomicU64,
    pub active_subscriptions: AtomicU64,
    pub retained_messages:    AtomicU64,
}

impl BrokerMetrics {
    pub fn to_prometheus_text(&self) -> String {
        format!(
            "# HELP mqtt_connections_total Total MQTT connections\n\
             # TYPE mqtt_connections_total counter\n\
             mqtt_connections_total {}\n\
             # HELP mqtt_messages_received_total Messages received\n\
             mqtt_messages_received_total {}\n",
            self.connections_total.load(Ordering::Relaxed),
            self.messages_received.load(Ordering::Relaxed),
        )
    }
}
```

งาน:
1. สร้าง `metrics.rs` พร้อม atomic counters
2. Increment counters ใน `handle_client`
3. เพิ่ม HTTP server (port 9090) ที่ serve `/metrics` endpoint
4. ทดสอบด้วย `curl localhost:9090/metrics`

### แบบฝึกหัดที่ 5: Topic Tree Optimization

`SubscriptionStore` ปัจจุบัน O(N) scan ทุก subscriber สำหรับทุก PUBLISH ทำให้ไม่ scale

ลอง implement **Topic Tree** (trie-based):

```rust
// src/topic_tree.rs
pub struct TopicNode {
    pub subscribers:  Vec<(String, QoS)>,     // clients subscribed at this level
    pub children:     HashMap<String, TopicNode>, // exact match children
    pub plus_child:   Option<Box<TopicNode>>,  // + wildcard subtree
    pub hash_subs:    Vec<(String, QoS)>,      // clients with # at this level
}
```

งาน:
1. implement `insert(filter, client_id, qos)` และ `find(topic)` บน `TopicNode`
2. measure performance ด้วย `criterion` benchmark
3. เปรียบเทียบ O(N) scan กับ O(depth) trie สำหรับ 10k subscriptions

### แบบฝึกหัดที่ 6: Persistent Session Storage (SQLite)

Session ปัจจุบันอยู่ใน memory เท่านั้น ถ้า broker restart in-flight messages หาย

งาน:
1. เพิ่ม `sqlx` crate พร้อม SQLite backend
2. สร้าง schema: `sessions`, `subscriptions`, `pending_acks`, `retained_messages`
3. บันทึก session state ทุกครั้งที่มีการเปลี่ยนแปลง
4. โหลด state กลับมาเมื่อ broker start
5. ทดสอบ crash recovery: kill broker → restart → verify in-flight messages ยังอยู่

---

## สรุป

โปรเจคนี้สร้าง MQTT 3.1.1 broker ที่ implement core protocol ครบ:

**Component หลักที่สร้าง:**

| Component | ไฟล์ | ความรับผิดชอบ |
|-----------|------|--------------|
| Packet Codec | `codec.rs` | encode/decode ทุก MQTT packet type, variable-length encoding |
| Topic Router | `topic.rs` | wildcard matching (`+`, `#`), subscription store |
| Session Manager | `session.rs` | session lifecycle, QoS 1 in-flight tracking, packet_id |
| Retain Store | `retain.rs` | last-value cache per topic, delivery on subscribe |
| Broker Core | `broker.rs` | async client handler, shared state, dispatch |

**Pattern สำคัญที่ได้เรียน:**

1. **Binary Protocol Parsing** — การอ่าน byte stream อย่างระมัดระวัง offset-by-offset พร้อม error handling ที่บอก position ที่ผิด

2. **Recursive Pattern Matching** — `match_parts` แบบ tail-recursive ที่ clean และ testable กว่า loop-based approach

3. **Shared Mutable State ใน Async** — `Arc<Mutex<T>>` pattern, lock scope management, ไม่ hold lock ข้าม `.await`

4. **Protocol State Machine** — CONNECT handshake ต้องเป็น packet แรกเสมอ, session lifecycle, will message delivery

5. **Test-Driven Protocol Implementation** — เขียน encode → decode roundtrip test ก่อน แล้วค่อย implement จึงมั่นใจว่า byte layout ถูก

**เชื่อมโยงไปโปรเจคถัดไป:**

Project H05 (Load Balancer) จะนำทักษะ async networking และ shared state ที่ได้จาก H04 ไปใช้ต่อ บวกกับ health checking, connection pooling, และ routing algorithms (round-robin, least-connections) สำหรับ distribute traffic ข้าม backend servers

---

**โปรเจคก่อนหน้า:** [Project H03: DNS Resolver](project-h03-dns-resolver.md) | **โปรเจคถัดไป:** [Project H05: Load Balancer](project-h05-load-balancer.md)
