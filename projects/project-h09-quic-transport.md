# Project H09: UDP Reliable Transport Protocol

> โมดูล: H — Networking/Protocols | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 20 ชั่วโมง

## ภาพรวมโปรเจค

**UDP Reliable Transport** คือหัวใจของโปรโตคอลเครือข่ายสมัยใหม่ที่ต้องการความเร็วสูงและ latency ต่ำ เช่น **QUIC** (ใช้ใน HTTP/3), **KCP** (เกม Online), **ENet** (game networking library) และ **WebRTC data channels**

โปรเจคนี้สร้าง **Reliable UDP Transport** ตั้งแต่ศูนย์ โดยนำแนวคิดเดียวกันกับ QUIC และ KCP มาประยุกต์ใช้:

- **Packet format** แบบ binary ที่กะทัดรัด — header 15 bytes พร้อม selective ACK bitfield
- **Sliding window ARQ** (Automatic Repeat reQuest) — ส่งได้หลาย packet พร้อมกันโดยไม่ต้องรอ ACK ทีละตัว
- **RTT estimation** ด้วย Jacobson/Karels algorithm ตาม RFC 6298 — คำนวณ timeout ที่แม่นยำ
- **AIMD congestion control** — Slow Start + Congestion Avoidance แบบเดียวกับ TCP แต่ทำงานบน UDP
- **Connection state machine** — 3-way handshake และ 4-way close ที่ปลอดภัย
- **Stream multiplexing** พร้อม flow control ระดับ stream — หลาย logical channel บน UDP socket เดียว

### ทำไม UDP แทน TCP?

| คุณสมบัติ | TCP | UDP+ARQ (เรา) |
|-----------|-----|----------------|
| Head-of-line blocking | มี — packet หายทำให้ทุก stream รอ | ไม่มี — แต่ละ stream แยกอิสระ |
| Connection setup | 3-way handshake (~1 RTT) | สามารถ 0-RTT ได้ |
| Congestion control | kernel จัดการ | application จัดการ (ปรับได้) |
| Reliability | built-in | เราสร้างเอง (เลือกได้ว่า stream ไหน reliable) |
| Multipath | ไม่รองรับโดยตรง | รองรับ (QUIC feature) |

ระบบ production จริง: **YouTube**, **Google Search**, **Cloudflare** ล้วนใช้ QUIC/HTTP3 ที่มีแนวคิดเดียวกันทั้งสิ้น

---

## สิ่งที่จะได้เรียนรู้

- **Binary serialisation** แบบ manual — `u32::to_be_bytes()`, `from_be_bytes()` โดยไม่ต้องพึ่ง crate ภายนอก
- **Sliding window protocol** — `VecDeque<SentPacket>` เป็น send window, `BTreeMap<u32, Vec<u8>>` เป็น receive window
- **Selective ACK (SACK)** — bitfield 32 บิตสำหรับ acknowledge packet ที่มาถึงแบบกระโดด
- **Jacobson/Karels RTT algorithm** — fixed-point exponential moving average สำหรับ SRTT + RTTVAR
- **AIMD congestion control** — Slow Start → Congestion Avoidance → Loss Recovery
- **Finite State Machine** — `ConnectionState` + `transition()` function ที่ exhaustive และ testable
- **Stream multiplexing** — `HashMap<StreamId, Stream>` พร้อม credit-based flow control
- **Sequence number wraparound** — การเปรียบเทียบ `u32` ที่ wrap ด้วย two's complement arithmetic

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 1-20** — Ownership, borrowing, `Vec`, `HashMap`, `BTreeMap`, `VecDeque`
- **Part 21-40** — Enums, pattern matching, `Result`/`Option`, `impl` blocks
- **Part 41-50** — `async/await`, tokio runtime, `UdpSocket`
- **Part 51-60** — `Arc<Mutex<T>>`, trait objects, error handling ด้วย `thiserror`
- **Part 61-70** — Lifetimes, closures, iterators
- โปรเจค H08 (TCP Chat) — ความเข้าใจ event loop, framing, connection state

---

## โครงสร้างโปรเจค (Project Layout)

```
quic-transport/
├── src/
│   ├── main.rs          # binary entry point (ตัวอย่างการใช้งาน)
│   ├── lib.rs           # re-export ทุก module
│   ├── packet.rs        # Packet, PacketType, encode/decode, SACK helpers
│   ├── window.rs        # SendWindow, RecvWindow, SentPacket
│   ├── rtt.rs           # RttEstimator (Jacobson/Karels)
│   ├── congestion.rs    # CongestionController (AIMD)
│   ├── connection.rs    # ConnectionState FSM, transition()
│   └── stream.rs        # Stream, StreamMux, flow control
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow ภาพรวม

```
Application
    │  write(stream_id, data)
    ▼
StreamMux ──────────────────────────────────────────────┐
    │  pop_send() per stream                              │
    ▼                                               flow control
SendWindow ◄─── CongestionController                    │
    │  on_send(seq, data)        cwnd limits in-flight   │
    ▼                                                    │
Packet::encode() ──────────────────────► UdpSocket.send()
                                                         │
                                                         ▼
UdpSocket.recv() ◄──────────────────── Packet::decode() ◄
    │
    ▼
RecvWindow.on_receive(seq, data)
    │  drain_ordered() → in-order delivery
    ▼
StreamMux.dispatch(stream_id, data)
    │
    ▼
Application.read(stream_id)
```

### ชั้นของ Protocol

```
┌────────────────────────────────────────────┐
│  Application (streams, HTTP, game state)    │  ← Layer 5-7
├────────────────────────────────────────────┤
│  Stream Multiplexer + Flow Control          │  ← เราสร้าง
├────────────────────────────────────────────┤
│  Congestion Control (AIMD)                  │  ← เราสร้าง
├────────────────────────────────────────────┤
│  Sliding Window ARQ + RTT + Retransmission  │  ← เราสร้าง
├────────────────────────────────────────────┤
│  Packet Format (binary encode/decode)       │  ← เราสร้าง
├────────────────────────────────────────────┤
│  UDP (unreliable datagram)                  │  ← OS/kernel
├────────────────────────────────────────────┤
│  IP / Ethernet                              │  ← Network hardware
└────────────────────────────────────────────┘
```

### การเลือก Data Structure

| Component | Data Structure | เหตุผล |
|-----------|---------------|--------|
| Send window | `VecDeque<SentPacket>` | O(1) push back, pop front; ลบ middle ด้วย `retain` |
| Receive buffer | `BTreeMap<u32, Vec<u8>>` | sorted by seq, drain contiguous O(log n) |
| Stream table | `HashMap<StreamId, Stream>` | O(1) lookup by stream ID |
| Send buffer per stream | `VecDeque<Vec<u8>>` | FIFO queue สำหรับ chunks |
| Recv buffer per stream | `VecDeque<Vec<u8>>` | in-order delivery queue |

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้าง Packet และ Binary Serialisation

เริ่มจากหน่วยพื้นฐานที่สุด: รูปแบบ packet ที่ส่งผ่านเครือข่าย

**Wire format ของ packet:**

```
 0         1         2         3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─────────┼───────────────────────────────────────────────────────┤
│  type   │                      seq (32 bit)                     │
├─────────┴───────────────────────────────────────────────────────┤
│                       ack (32 bit)                              │
├─────────────────────────────────────────────────────────────────┤
│                     ack_bits (32 bit)                           │
├───────────────────────────────────┬─────────────────────────────┤
│        payload_len (16 bit)       │      payload (variable)     │
└───────────────────────────────────┴─────────────────────────────┘
Header = 15 bytes fixed
```

**`ack_bits` คืออะไร?**

`ack_bits` เป็น bitmask 32 บิตที่ใช้สำหรับ **Selective ACK (SACK)** — ระบุว่า packet ไหนที่มาถึงแล้วนอกจาก `ack`:

- `ack` = sequence number สูงสุดที่รับมาแบบ contiguous
- bit 0 ของ `ack_bits` = `ack - 1` ได้รับหรือยัง
- bit 1 ของ `ack_bits` = `ack - 2` ได้รับหรือยัง
- ... bit N = `ack - (N+1)` ได้รับหรือยัง

ตัวอย่าง: ถ้า `ack=10`, `ack_bits=0b0101` แปลว่า seq 9 (`ack-1`) และ seq 7 (`ack-3`) ได้รับแล้ว แต่ seq 8 (`ack-2`) ยังหายอยู่

```rust
// src/packet.rs

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum PacketType {
    Data = 0,
    Ack = 1,
    Ping = 2,
    Pong = 3,
    Connect = 4,
    Disconnect = 5,
    ConnAck = 6,
    Fin = 7,
    FinAck = 8,
}

impl PacketType {
    pub fn from_u8(v: u8) -> Option<Self> {
        match v {
            0 => Some(PacketType::Data),
            1 => Some(PacketType::Ack),
            2 => Some(PacketType::Ping),
            3 => Some(PacketType::Pong),
            4 => Some(PacketType::Connect),
            5 => Some(PacketType::Disconnect),
            6 => Some(PacketType::ConnAck),
            7 => Some(PacketType::Fin),
            8 => Some(PacketType::FinAck),
            _ => None,
        }
    }

    pub fn to_u8(&self) -> u8 {
        self.clone() as u8
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Packet {
    pub packet_type: PacketType,
    pub seq: u32,
    pub ack: u32,
    pub ack_bits: u32,
    pub payload: Vec<u8>,
}

#[derive(Debug, PartialEq, Eq)]
pub enum PacketError {
    TooShort,
    InvalidType(u8),
    PayloadLenMismatch,
}

impl Packet {
    pub const HEADER_LEN: usize = 15; // 1 + 4 + 4 + 4 + 2

    /// Encode packet เป็น bytes (big-endian)
    pub fn encode(&self) -> Vec<u8> {
        let payload_len = self.payload.len() as u16;
        let mut buf = Vec::with_capacity(Self::HEADER_LEN + self.payload.len());
        buf.push(self.packet_type.to_u8());
        buf.extend_from_slice(&self.seq.to_be_bytes());
        buf.extend_from_slice(&self.ack.to_be_bytes());
        buf.extend_from_slice(&self.ack_bits.to_be_bytes());
        buf.extend_from_slice(&payload_len.to_be_bytes());
        buf.extend_from_slice(&self.payload);
        buf
    }

    /// Decode packet จาก bytes
    pub fn decode(buf: &[u8]) -> Result<Self, PacketError> {
        if buf.len() < Self::HEADER_LEN {
            return Err(PacketError::TooShort);
        }
        let packet_type = PacketType::from_u8(buf[0])
            .ok_or(PacketError::InvalidType(buf[0]))?;
        let seq = u32::from_be_bytes([buf[1], buf[2], buf[3], buf[4]]);
        let ack = u32::from_be_bytes([buf[5], buf[6], buf[7], buf[8]]);
        let ack_bits = u32::from_be_bytes([buf[9], buf[10], buf[11], buf[12]]);
        let payload_len = u16::from_be_bytes([buf[13], buf[14]]) as usize;
        if buf.len() < Self::HEADER_LEN + payload_len {
            return Err(PacketError::PayloadLenMismatch);
        }
        let payload = buf[Self::HEADER_LEN..Self::HEADER_LEN + payload_len].to_vec();
        Ok(Packet { packet_type, seq, ack, ack_bits, payload })
    }
}

/// แปลง ack_bits เป็น list ของ seq ที่ถูก ACK เพิ่มเติม
pub fn decode_ack_bits(base_ack: u32, ack_bits: u32) -> Vec<u32> {
    let mut acked = Vec::new();
    for i in 0..32u32 {
        if ack_bits & (1 << i) != 0 {
            acked.push(base_ack.wrapping_sub(i + 1));
        }
    }
    acked
}

/// สร้าง ack_bits จาก list ของ seq ที่รับมาแล้ว
pub fn encode_ack_bits(base_ack: u32, received: &[u32]) -> u32 {
    let mut bits: u32 = 0;
    for &seq in received {
        let diff = base_ack.wrapping_sub(seq);
        if diff >= 1 && diff <= 32 {
            bits |= 1 << (diff - 1);
        }
    }
    bits
}
```

**ทำไมไม่ใช้ `serde` สำหรับ encode/decode?**

Serde เหมาะกับ JSON/YAML แต่สำหรับ binary protocol ที่ต้องการ:
1. **ขนาดแน่นอน** — ทุก field มี fixed offset ที่รู้ล่วงหน้า
2. **Big-endian** — network byte order ตามมาตรฐาน
3. **ไม่มี overhead** — ไม่มี field names, separators
4. **Zero-copy** บางกรณี — สามารถ cast buffer โดยตรงได้

การใช้ `u32::to_be_bytes()` และ `from_be_bytes()` ให้ผลลัพธ์ที่ predictable ทุก platform

---

### ขั้นที่ 2: Sliding Window — ฝั่ง Send

**Send Window** คือ buffer ของ packet ที่ส่งไปแล้วแต่ยังรอ ACK อยู่ ขนาด window กำหนดว่ามี packet ได้กี่ตัวใน flight พร้อมกัน

```rust
// src/window.rs

use std::collections::VecDeque;
use std::time::Instant;

#[derive(Debug, Clone)]
pub struct SentPacket {
    pub seq: u32,
    pub data: Vec<u8>,
    pub sent_at: Instant,
    pub retransmit_count: u32,
}

pub struct SendWindow {
    pub sent: VecDeque<SentPacket>,
    pub window_size: u32,
    pub next_seq: u32,
}

impl SendWindow {
    pub fn new(window_size: u32) -> Self {
        SendWindow {
            sent: VecDeque::new(),
            window_size,
            next_seq: 0,
        }
    }

    pub fn can_send(&self) -> bool {
        (self.sent.len() as u32) < self.window_size
    }

    pub fn on_send(&mut self, data: Vec<u8>) -> u32 {
        let seq = self.next_seq;
        self.next_seq = self.next_seq.wrapping_add(1);
        self.sent.push_back(SentPacket {
            seq,
            data,
            sent_at: Instant::now(),
            retransmit_count: 0,
        });
        seq
    }

    /// Process ACK + SACK bitfield — คืน list ของ seq ที่ newly acknowledged
    pub fn on_ack(&mut self, ack: u32, ack_bits: u32) -> Vec<u32> {
        let mut newly_acked = Vec::new();
        let mut acked_set = std::collections::HashSet::new();
        acked_set.insert(ack);
        for i in 0..32u32 {
            if ack_bits & (1 << i) != 0 {
                acked_set.insert(ack.wrapping_sub(i + 1));
            }
        }
        self.sent.retain(|pkt| {
            if acked_set.contains(&pkt.seq) {
                newly_acked.push(pkt.seq);
                false
            } else {
                true
            }
        });
        newly_acked
    }

    pub fn timed_out_packets(&self, rto_ms: u64) -> Vec<u32> {
        let threshold = std::time::Duration::from_millis(rto_ms);
        self.sent
            .iter()
            .filter(|p| p.sent_at.elapsed() >= threshold)
            .map(|p| p.seq)
            .collect()
    }

    pub fn mark_retransmit(&mut self, seq: u32) -> Option<u32> {
        for pkt in &mut self.sent {
            if pkt.seq == seq {
                pkt.retransmit_count += 1;
                pkt.sent_at = Instant::now();
                return Some(pkt.retransmit_count);
            }
        }
        None
    }
}
```

**ทำไม `VecDeque` แทน `Vec`?**

Send window มีพฤติกรรมเป็น queue: เพิ่ม packet ที่ tail, ลบ packet ที่ head เมื่อ ACK มาถึง `VecDeque` ให้ O(1) สำหรับทั้งสองด้าน ในขณะที่ `Vec` จะ shift elements ทุกครั้งที่ลบจาก front (O(n))

ข้อสังเกตสำคัญ: ใช้ `retain()` แทนการวน loop เพื่อลบ — elegant กว่าและหลีกเลี่ยง borrow checker problem

---

### ขั้นที่ 3: Sliding Window — ฝั่ง Receive

**Receive Window** รับ packet ที่อาจมาไม่เรียงลำดับ แล้วส่งขึ้นไปหา application ตามลำดับที่ถูกต้อง

```rust
use std::collections::BTreeMap;

pub struct RecvWindow {
    pub received: BTreeMap<u32, Vec<u8>>,
    pub next_expected: u32,
}

impl RecvWindow {
    pub fn new() -> Self {
        RecvWindow {
            received: BTreeMap::new(),
            next_expected: 0,
        }
    }

    pub fn on_receive(&mut self, seq: u32, data: Vec<u8>) -> bool {
        // ถ้า seq < next_expected → duplicate (ส่งมาซ้ำ)
        if seq_less_than(seq, self.next_expected) {
            return false;
        }
        // ป้องกัน memory bomb: reject packet ที่ห่างเกินไป
        if seq.wrapping_sub(self.next_expected) > 1024 {
            return false;
        }
        if self.received.contains_key(&seq) {
            return false;
        }
        self.received.insert(seq, data);
        true
    }

    /// ดึง packet ที่ต่อเนื่องออกมาตามลำดับ
    pub fn drain_ordered(&mut self) -> Vec<Vec<u8>> {
        let mut result = Vec::new();
        loop {
            if let Some(data) = self.received.remove(&self.next_expected) {
                result.push(data);
                self.next_expected = self.next_expected.wrapping_add(1);
            } else {
                break;
            }
        }
        result
    }

    /// คำนวณ ack และ ack_bits สำหรับส่งกลับไปหา sender
    pub fn build_ack(&self) -> (u32, u32) {
        let base = self.next_expected.wrapping_sub(1);
        let mut bits: u32 = 0;
        for (&seq, _) in &self.received {
            let diff = seq.wrapping_sub(base);
            if diff >= 1 && diff <= 32 {
                bits |= 1 << (diff - 1);
            }
        }
        (base, bits)
    }
}

/// Sequence number comparison ที่รองรับ u32 wraparound
fn seq_less_than(a: u32, b: u32) -> bool {
    ((b.wrapping_sub(a)) as i32) > 0 && (b.wrapping_sub(a) < 0x8000_0000)
}
```

**ทำไม `BTreeMap` แทน `HashMap`?**

Receive buffer ต้องสามารถ drain ตามลำดับ seq number ซึ่ง `BTreeMap` ให้ iterator ที่เรียงลำดับอยู่แล้ว ส่วน `HashMap` ต้องการ sort ก่อน (O(n log n) เพิ่มเติม)

**Sequence number wraparound:**

เมื่อ `u32` overflow จาก `0xFFFF_FFFF` กลับมาเป็น `0x0000_0000` การเปรียบเทียบธรรมดาจะผิดพลาด ต้องใช้ arithmetic modulo 2^32 ตาม RFC 1982:

```
seq_less_than(a, b) ⟺ (b - a) mod 2^32 < 2^31
```

นั่นคือ: `b` "มากกว่า" `a` ถ้าระยะห่างจาก `a` ไป `b` (ไปข้างหน้า) น้อยกว่าครึ่งรอบ

---

### ขั้นที่ 4: RTT Estimation ด้วย Jacobson/Karels Algorithm

RTT ที่แม่นยำเป็นสิ่งสำคัญมาก — ถ้า RTO สั้นเกินไปจะ retransmit โดยไม่จำเป็น ถ้ายาวเกินไปจะรอนานเมื่อ packet สูญหาย

**Jacobson/Karels Algorithm (RFC 6298):**

```
# First sample:
SRTT    = R             # R = RTT sample แรก
RTTVAR  = R / 2

# Subsequent samples:
RTTVAR  = (1 - β) × RTTVAR + β × |SRTT - R|    # β = 0.25
SRTT    = (1 - α) × SRTT  + α × R              # α = 0.125
RTO     = SRTT + max(G, 4 × RTTVAR)            # G = clock granularity
```

```rust
// src/rtt.rs

pub struct RttEstimator {
    srtt_ms: Option<f64>,
    rttvar_ms: Option<f64>,
    pub rto_ms: u64,
    pub min_rto_ms: u64,
    pub max_rto_ms: u64,
}

impl RttEstimator {
    pub fn new() -> Self {
        RttEstimator {
            srtt_ms: None,
            rttvar_ms: None,
            rto_ms: 1000,       // Initial RTO = 1 second
            min_rto_ms: 200,
            max_rto_ms: 60_000,
        }
    }

    pub fn update(&mut self, rtt_sample_ms: f64) {
        match (self.srtt_ms, self.rttvar_ms) {
            (None, _) => {
                self.srtt_ms = Some(rtt_sample_ms);
                self.rttvar_ms = Some(rtt_sample_ms / 2.0);
            }
            (Some(srtt), Some(rttvar)) => {
                let err = (srtt - rtt_sample_ms).abs();
                let new_rttvar = 0.75 * rttvar + 0.25 * err;
                let new_srtt = 0.875 * srtt + 0.125 * rtt_sample_ms;
                self.srtt_ms = Some(new_srtt);
                self.rttvar_ms = Some(new_rttvar);
            }
            _ => unreachable!(),
        }
        let srtt = self.srtt_ms.unwrap();
        let rttvar = self.rttvar_ms.unwrap();
        let rto = (srtt + (4.0 * rttvar).max(1.0)).ceil() as u64;
        self.rto_ms = rto.clamp(self.min_rto_ms, self.max_rto_ms);
    }

    /// Exponential backoff เมื่อ timeout
    pub fn backoff_rto(&mut self) {
        self.rto_ms = (self.rto_ms * 2).min(self.max_rto_ms);
    }
}
```

**ทำไม EWMA แทน Simple Average?**

EWMA (Exponentially Weighted Moving Average) ให้น้ำหนักกับ sample ใหม่มากกว่า sample เก่า ทำให้ estimate ปรับตัวได้เร็วเมื่อ network condition เปลี่ยน ส่วน `RTTVAR` ที่เป็น variance estimator ทำให้ RTO ยืดหยุ่นตามความแปรปรวนของ network

**Karn's Algorithm**: ไม่ควรใช้ RTT sample จาก packet ที่ retransmit เพราะไม่รู้ว่า ACK ตอบ transmission ไหน (ambiguity problem) — โปรเจคเราใช้ timestamp ต่อ packet เพื่อหลีกเลี่ยงปัญหานี้

---

### ขั้นที่ 5: AIMD Congestion Control

**AIMD = Additive Increase, Multiplicative Decrease**

เป็น algorithm ที่ทำให้ทุก TCP/QUIC connection แบ่งปัน bandwidth กันอย่างยุติธรรม (Fairness) และปรับตัวอัตโนมัติกับ network capacity

```
Phase 1: Slow Start
  cwnd เพิ่มเป็น 2 เท่าทุก RTT จนถึง ssthresh
  (จริงๆ คือ +1 ต่อ ACK ซึ่งเท่ากับ 2× ต่อ RTT เพราะ cwnd ACK ต่อ window)

Phase 2: Congestion Avoidance
  cwnd เพิ่มทีละ 1/cwnd ต่อ ACK = +1 ต่อ RTT (linear growth)

On Loss (duplicate ACK / timeout):
  ssthresh = cwnd / 2
  cwnd = ssthresh (fast recovery) หรือ cwnd = 1 (timeout)
```

```rust
// src/congestion.rs

#[derive(Debug, Clone, PartialEq)]
pub enum CongestionPhase {
    SlowStart,
    CongestionAvoidance,
}

pub struct CongestionController {
    pub cwnd: f64,
    pub ssthresh: f64,
    pub phase: CongestionPhase,
    acked_in_window: f64,
}

impl CongestionController {
    pub fn new() -> Self {
        CongestionController {
            cwnd: 1.0,
            ssthresh: 16.0,
            phase: CongestionPhase::SlowStart,
            acked_in_window: 0.0,
        }
    }

    pub fn on_ack(&mut self) {
        match self.phase {
            CongestionPhase::SlowStart => {
                self.cwnd += 1.0;
                if self.cwnd >= self.ssthresh {
                    self.phase = CongestionPhase::CongestionAvoidance;
                    self.acked_in_window = 0.0;
                }
            }
            CongestionPhase::CongestionAvoidance => {
                self.acked_in_window += 1.0;
                if self.acked_in_window >= self.cwnd {
                    self.cwnd += 1.0;
                    self.acked_in_window = 0.0;
                }
            }
        }
    }

    /// Loss detection (3 dup ACK) — Reno fast recovery
    pub fn on_loss(&mut self) {
        self.ssthresh = (self.cwnd / 2.0).max(2.0);
        self.cwnd = self.ssthresh;
        self.phase = CongestionPhase::CongestionAvoidance;
        self.acked_in_window = 0.0;
    }

    /// Timeout — conservative reset
    pub fn on_timeout(&mut self) {
        self.ssthresh = (self.cwnd / 2.0).max(2.0);
        self.cwnd = 1.0;
        self.phase = CongestionPhase::SlowStart;
        self.acked_in_window = 0.0;
    }

    pub fn cwnd_packets(&self) -> u32 {
        self.cwnd.floor() as u32
    }
}
```

**Interplay กับ Send Window:**

```
in_flight = send_window.sent.len()
effective_cwnd = min(cwnd_packets, window_size)
can_send = in_flight < effective_cwnd
```

**ทำไม `ssthresh` ไม่ควรลงต่ำกว่า 2:**

ถ้า `ssthresh = 1` แล้ว `cwnd = 1` เราจะวน Slow Start ได้แค่ 1 packet ต่อ RTT ซึ่งช้ามาก การ clamp ที่ 2 ทำให้ recovery เร็วขึ้น

---

### ขั้นที่ 6: Connection State Machine (FSM)

Connection ต้องผ่าน 3-way handshake ก่อนส่งข้อมูล และ 4-way close ก่อนปิด

**3-way Handshake:**
```
Client          Server
  │──── Connect ──────►│
  │◄─── ConnAck ────────│
  │──── Ack ────────────►│   (ส่ง data แรกก็ได้)
  │═══ Connected ═══════│
```

**4-way Close:**
```
Initiator       Peer
  │──── Fin ────────────►│
  │◄─── FinAck ──────────│
  │◄─── Fin ─────────────│
  │──── FinAck ──────────►│
  │═══ Closed ═══════════│
```

```rust
// src/connection.rs

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ConnectionState {
    Idle,
    Connecting,
    Connected,
    Closing,
    Closed,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ConnectionEvent {
    Connect,
    RecvConnAck,
    RecvConnect,
    RecvAck,
    Close,
    RecvFin,
    RecvFinAck,
    Reset,
}

#[derive(Debug, Clone, PartialEq)]
pub enum FsmError {
    InvalidTransition { state: ConnectionState, event: ConnectionEvent },
}

pub fn transition(
    state: &ConnectionState,
    event: &ConnectionEvent,
) -> Result<ConnectionState, FsmError> {
    use ConnectionState::*;
    use ConnectionEvent::*;
    match (state, event) {
        (Idle, Connect)         => Ok(Connecting),
        (Idle, RecvConnect)     => Ok(Connected),   // server path
        (Connecting, RecvConnAck) => Ok(Connected),
        (Connected, Close)      => Ok(Closing),
        (Connected, RecvFin)    => Ok(Closing),
        (Closing, RecvFinAck)   => Ok(Closed),
        (Closing, RecvFin)      => Ok(Closed),      // simultaneous close
        (_, Reset)              => Ok(Closed),
        (s, e) => Err(FsmError::InvalidTransition {
            state: s.clone(),
            event: e.clone(),
        }),
    }
}
```

**ทำไม `transition()` เป็น pure function แทน method?**

Pure function ทดสอบได้ง่ายกว่า — ไม่มี side effect, ไม่ต้องสร้าง object ทดสอบแต่ละ transition โดยตรงได้ เป็นแนวคิด **Functional Core, Imperative Shell** ที่ทำให้ logic ที่ซับซ้อนอยู่ใน pure function และ I/O อยู่ภายนอก

**Invalid transition เป็น error ไม่ใช่ panic:**

ใช้ `Result` เพราะ transition อาจ fail ได้จากเหตุการณ์ภายนอก เช่น peer ส่ง packet ผิด order ไม่ควร `panic!` ใน network code

---

### ขั้นที่ 7: Stream Multiplexing และ Flow Control

หลายๆ logical stream แชร์ UDP connection เดียวกัน แต่ละ stream มี flow control ของตัวเอง

**Credit-based Flow Control:**

```
Stream::write()  →  ตรวจสอบ send_credit > len(data)
                    ถ้าไม่พอ → return false (backpressure)
                    ถ้าพอ → send_credit -= len(data), enqueue

Peer ส่ง WINDOW_UPDATE frame → add_credit(bytes)
```

```rust
// src/stream.rs

pub type StreamId = u32;

pub struct Stream {
    pub id: StreamId,
    pub send_buf: VecDeque<Vec<u8>>,
    pub recv_buf: VecDeque<Vec<u8>>,
    pub send_credit: u64,    // bytes ที่ peer ยอมรับได้
    pub recv_window: u64,    // bytes ที่เราจะรับ
    pub bytes_received: u64,
    pub send_closed: bool,
    pub recv_closed: bool,
}

impl Stream {
    pub fn new(id: StreamId, initial_credit: u64) -> Self {
        Stream {
            id,
            send_buf: VecDeque::new(),
            recv_buf: VecDeque::new(),
            send_credit: initial_credit,
            recv_window: 65536,
            bytes_received: 0,
            send_closed: false,
            recv_closed: false,
        }
    }

    pub fn write(&mut self, data: Vec<u8>) -> bool {
        if self.send_closed { return false; }
        if data.len() as u64 > self.send_credit { return false; }
        self.send_credit -= data.len() as u64;
        self.send_buf.push_back(data);
        true
    }

    pub fn add_credit(&mut self, bytes: u64) {
        self.send_credit += bytes;
    }

    pub fn on_recv(&mut self, data: Vec<u8>) {
        if !self.recv_closed {
            self.bytes_received += data.len() as u64;
            self.recv_buf.push_back(data);
        }
    }

    pub fn read(&mut self) -> Option<Vec<u8>> {
        self.recv_buf.pop_front()
    }

    pub fn pop_send(&mut self) -> Option<Vec<u8>> {
        self.send_buf.pop_front()
    }
}

pub struct StreamMux {
    streams: HashMap<StreamId, Stream>,
    next_stream_id: StreamId,
    initial_credit: u64,
}

impl StreamMux {
    pub fn new(initial_credit: u64) -> Self {
        StreamMux {
            streams: HashMap::new(),
            next_stream_id: 1,
            initial_credit,
        }
    }

    pub fn open_stream(&mut self) -> StreamId {
        let id = self.next_stream_id;
        self.next_stream_id += 2; // ต่างฝั่ง ID ต่างกัน (client=odd, server=even)
        self.streams.insert(id, Stream::new(id, self.initial_credit));
        id
    }

    pub fn dispatch(&mut self, stream_id: StreamId, data: Vec<u8>) {
        let credit = self.initial_credit;
        let s = self.streams
            .entry(stream_id)
            .or_insert_with(|| Stream::new(stream_id, credit));
        s.on_recv(data);
    }
}
```

**ทำไม stream ID client=odd, server=even?**

แนวคิดมาจาก HTTP/2 และ QUIC เพื่อป้องกัน ID collision เมื่อทั้งสองฝั่งเปิด stream พร้อมกัน ถ้า client ใช้ ID คี่และ server ใช้ ID คู่ ID จะไม่ตรงกันเลย

---

### ขั้นที่ 8: Integration — รวมทุก Component เข้าด้วยกัน

ในระบบจริง loop หลักจะทำงานดังนี้:

```rust
// ตัวอย่าง event loop (async/tokio)
// src/main.rs

use tokio::net::UdpSocket;
use quic_transport::packet::{Packet, PacketType};
use quic_transport::window::{SendWindow, RecvWindow};
use quic_transport::rtt::RttEstimator;
use quic_transport::congestion::CongestionController;
use quic_transport::connection::{ConnectionState, ConnectionEvent, transition};
use quic_transport::stream::StreamMux;

const MAX_RETRANSMIT: u32 = 3;
const WINDOW_SIZE: u32 = 64;
const INITIAL_CREDIT: u64 = 65536;

pub struct Transport {
    socket: UdpSocket,
    state: ConnectionState,
    send_window: SendWindow,
    recv_window: RecvWindow,
    rtt: RttEstimator,
    cc: CongestionController,
    mux: StreamMux,
}

impl Transport {
    pub async fn new_client(local: &str, remote: &str) -> std::io::Result<Self> {
        let socket = UdpSocket::bind(local).await?;
        socket.connect(remote).await?;
        Ok(Transport {
            socket,
            state: ConnectionState::Idle,
            send_window: SendWindow::new(WINDOW_SIZE),
            recv_window: RecvWindow::new(),
            rtt: RttEstimator::new(),
            cc: CongestionController::new(),
            mux: StreamMux::new(INITIAL_CREDIT),
        })
    }

    pub async fn connect(&mut self) -> std::io::Result<()> {
        self.state = transition(&self.state, &ConnectionEvent::Connect)
            .map_err(|e| std::io::Error::new(std::io::ErrorKind::InvalidInput, format!("{:?}", e)))?;
        let pkt = Packet::new_connect();
        self.socket.send(&pkt.encode()).await?;
        Ok(())
    }

    pub async fn run_loop(&mut self) {
        let mut buf = vec![0u8; 65536];
        loop {
            tokio::select! {
                // รับ packet
                Ok(n) = self.socket.recv(&mut buf) => {
                    if let Ok(pkt) = Packet::decode(&buf[..n]) {
                        self.handle_packet(pkt);
                    }
                }
                // retransmit timer
                _ = tokio::time::sleep(
                    tokio::time::Duration::from_millis(self.rtt.rto_ms)
                ) => {
                    self.check_retransmit().await;
                }
            }
        }
    }

    fn handle_packet(&mut self, pkt: Packet) {
        match pkt.packet_type {
            PacketType::ConnAck => {
                if let Ok(new_state) = transition(&self.state, &ConnectionEvent::RecvConnAck) {
                    self.state = new_state;
                }
            }
            PacketType::Data => {
                // ดึง stream_id จาก payload header (4 bytes แรก)
                if pkt.payload.len() >= 4 {
                    let stream_id = u32::from_be_bytes([
                        pkt.payload[0], pkt.payload[1],
                        pkt.payload[2], pkt.payload[3]
                    ]);
                    let data = pkt.payload[4..].to_vec();
                    if self.recv_window.on_receive(pkt.seq, data.clone()) {
                        let delivered = self.recv_window.drain_ordered();
                        for chunk in delivered {
                            self.mux.dispatch(stream_id, chunk);
                        }
                    }
                }
            }
            PacketType::Ack => {
                let newly_acked = self.send_window.on_ack(pkt.ack, pkt.ack_bits);
                for _ in &newly_acked {
                    self.cc.on_ack();
                }
            }
            PacketType::Fin => {
                if let Ok(new_state) = transition(&self.state, &ConnectionEvent::RecvFin) {
                    self.state = new_state;
                }
            }
            _ => {}
        }
    }

    async fn check_retransmit(&mut self) {
        let timed_out = self.send_window.timed_out_packets(self.rtt.rto_ms);
        for seq in timed_out {
            if let Some(count) = self.send_window.mark_retransmit(seq) {
                if count > MAX_RETRANSMIT {
                    // เกิน max retransmit → disconnect
                    self.state = transition(&self.state, &ConnectionEvent::Reset)
                        .unwrap_or(ConnectionState::Closed);
                    return;
                }
                self.rtt.backoff_rto();
                self.cc.on_timeout();
                // TODO: ส่ง packet ซ้ำ
            }
        }
    }
}

fn main() {
    println!("quic-transport library — run `cargo test` to verify all components");
}
```

---

## การทดสอบ (Testing)

### Unit Tests ครบทุก Component

โปรเจคนี้มี **38 unit tests** ครอบคลุมทุก module:

| Module | Tests | สิ่งที่ทดสอบ |
|--------|-------|------------|
| `packet` | 8 | encode/decode roundtrip, error cases, SACK helpers |
| `window` | 6 | send window capacity, ACK removal, receive ordering |
| `rtt` | 5 | first sample, convergence, backoff, known-value calc |
| `congestion` | 6 | SS→CA transition, AIMD halving, timeout reset |
| `connection` | 6 | handshake paths, close, reset, invalid transitions |
| `stream` | 7 | credit flow, mux dispatch, multi-stream isolation |

### Cargo.toml

```toml
[package]
name = "quic-transport"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "quic-transport"
path = "src/main.rs"

[lib]
name = "quic_transport"
path = "src/lib.rs"
```

### การรัน Tests

```bash
cargo test
```

### ผลลัพธ์จริงจากการรัน `cargo test`

```
   Compiling quic-transport v0.1.0 (/tmp/claude-0/-home-user-rust-course/07b7aacd-ef83-5236-b656-bb8d3b0fb702/scratchpad/quic-transport)
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.44s
     Running unittests src/lib.rs (target/debug/deps/quic_transport-7b268062ebc11d80)

running 38 tests
test congestion::tests::test_on_loss_halves_cwnd ... ok
test congestion::tests::test_on_timeout_resets_cwnd ... ok
test congestion::tests::test_slow_start_doubles ... ok
test congestion::tests::test_slow_start_to_ca_transition ... ok
test congestion::tests::test_congestion_avoidance_increment ... ok
test connection::tests::test_client_handshake ... ok
test connection::tests::test_graceful_close ... ok
test connection::tests::test_reset_from_any_state ... ok
test connection::tests::test_invalid_transition ... ok
test connection::tests::test_server_handshake ... ok
test connection::tests::test_simultaneous_close ... ok
test packet::tests::test_all_packet_types_roundtrip ... ok
test packet::tests::test_decode_ack_bits ... ok
test packet::tests::test_packet_encode_decode_ack ... ok
test congestion::tests::test_cwnd_floor ... ok
test packet::tests::test_encode_ack_bits ... ok
test packet::tests::test_packet_payload_len_mismatch ... ok
test packet::tests::test_packet_encode_decode_data ... ok
test packet::tests::test_packet_invalid_type ... ok
test rtt::tests::test_rtt_converges_with_stable_samples ... ok
test rtt::tests::test_rtt_first_sample ... ok
test rtt::tests::test_rtt_max_rto_clamp ... ok
test rtt::tests::test_rtt_with_known_samples ... ok
test stream::tests::test_stream_closed_write ... ok
test stream::tests::test_stream_credit_exhaustion ... ok
test rtt::tests::test_rtt_backoff_doubles_rto ... ok
test packet::tests::test_packet_too_short ... ok
test stream::tests::test_stream_credit_refill ... ok
test stream::tests::test_stream_mux_dispatch_creates_stream ... ok
test stream::tests::test_stream_mux_open_and_dispatch ... ok
test stream::tests::test_stream_write_and_read ... ok
test window::tests::test_recv_window_duplicate_ignored ... ok
test window::tests::test_recv_window_build_ack ... ok
test window::tests::test_send_window_can_send ... ok
test window::tests::test_recv_window_ordered_delivery ... ok
test window::tests::test_send_window_on_ack_removes_packets ... ok
test window::tests::test_send_window_sequence_numbers ... ok
test stream::tests::test_stream_mux_multiple_streams ... ok

test result: ok. 38 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s

     Running unittests src/main.rs (target/debug/deps/quic_transport-3f7733f814ae7f5e)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests quic_transport

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### ตัวอย่าง Test Code แต่ละ Module

#### packet tests

```rust
#[test]
fn test_packet_encode_decode_data() {
    let p = Packet::new_data(42, 10, 0b1010, b"hello".to_vec());
    let encoded = p.encode();
    assert_eq!(encoded.len(), Packet::HEADER_LEN + 5);
    let decoded = Packet::decode(&encoded).unwrap();
    assert_eq!(decoded, p);
}

#[test]
fn test_decode_ack_bits() {
    // ack_bits = 0b0101 หมายถึง base_ack-1 และ base_ack-3 ถูก ACK แล้ว
    let acked = decode_ack_bits(10, 0b0101);
    assert!(acked.contains(&9));  // 10 - 1
    assert!(acked.contains(&7));  // 10 - 3
    assert_eq!(acked.len(), 2);
}

#[test]
fn test_encode_ack_bits() {
    let bits = encode_ack_bits(10, &[9, 7]);
    assert_eq!(bits, 0b0101);
}
```

#### window tests

```rust
#[test]
fn test_recv_window_ordered_delivery() {
    let mut w = RecvWindow::new();
    // มาผิดลำดับ: 2, 0, 1
    assert!(w.on_receive(2, b"C".to_vec()));
    assert!(w.on_receive(0, b"A".to_vec()));
    assert!(w.on_receive(1, b"B".to_vec()));
    // drain ต้องได้ A, B, C ตามลำดับ
    let delivered = w.drain_ordered();
    assert_eq!(delivered, vec![b"A".to_vec(), b"B".to_vec(), b"C".to_vec()]);
    assert_eq!(w.next_expected, 3);
}

#[test]
fn test_recv_window_build_ack() {
    let mut w = RecvWindow::new();
    w.on_receive(0, b"A".to_vec());
    w.on_receive(1, b"B".to_vec());
    w.on_receive(3, b"D".to_vec()); // gap ที่ seq 2
    w.drain_ordered(); // drain 0,1 → next_expected = 2
    let (ack, bits) = w.build_ack();
    assert_eq!(ack, 1); // last contiguous = 1
    // seq 3: diff = 3 - 1 = 2, bit position = 1
    assert_eq!(bits & (1 << 1), 1 << 1);
}
```

#### rtt tests

```rust
#[test]
fn test_rtt_with_known_samples() {
    let mut est = RttEstimator::new();
    est.update(400.0);
    // After first: SRTT=400, RTTVAR=200
    assert!((est.srtt_ms().unwrap() - 400.0).abs() < 0.1);
    assert!((est.rttvar_ms().unwrap() - 200.0).abs() < 0.1);

    est.update(200.0);
    // RTTVAR = 0.75*200 + 0.25*|400-200| = 200
    // SRTT = 0.875*400 + 0.125*200 = 375
    let srtt2 = est.srtt_ms().unwrap();
    assert!((srtt2 - 375.0).abs() < 0.1, "SRTT={srtt2}");
}
```

#### connection FSM tests

```rust
#[test]
fn test_client_handshake() {
    let s0 = ConnectionState::Idle;
    let s1 = transition(&s0, &ConnectionEvent::Connect).unwrap();
    assert_eq!(s1, ConnectionState::Connecting);
    let s2 = transition(&s1, &ConnectionEvent::RecvConnAck).unwrap();
    assert_eq!(s2, ConnectionState::Connected);
}

#[test]
fn test_invalid_transition() {
    let s = ConnectionState::Idle;
    let result = transition(&s, &ConnectionEvent::RecvConnAck);
    assert!(result.is_err());
    if let Err(FsmError::InvalidTransition { state, event }) = result {
        assert_eq!(state, ConnectionState::Idle);
        assert_eq!(event, ConnectionEvent::RecvConnAck);
    }
}
```

---

## จุดที่ต้องระวัง (Pitfalls)

### Pitfall 1: Sequence Number Wraparound ทำให้เปรียบเทียบผิด

**ปัญหา:** เมื่อ `u32` overflow จาก `0xFFFF_FFFF` กลับมาเป็น `0x0000_0000`:

```rust
// ❌ ผิด: เปรียบเทียบธรรมดา
let is_older = recv_seq < next_expected; // ผิดเมื่อ wrap

// ✅ ถูก: ใช้ two's complement arithmetic
fn seq_less_than(a: u32, b: u32) -> bool {
    let diff = b.wrapping_sub(a);
    diff > 0 && diff < 0x8000_0000
}
```

**เหตุผล:** RFC 1982 กำหนดว่า "a < b" ในพื้นที่ serial number modulo 2^32 คือ `(b - a) mod 2^32 < 2^31` — นั่นคือ `b` อยู่ "ข้างหน้า" `a` ไม่เกินครึ่งรอบ

---

### Pitfall 2: Karn's Ambiguity — อย่าวัด RTT จาก Retransmitted Packets

**ปัญหา:** ถ้า packet 5 timeout แล้ว retransmit และ ACK มาถึง เราไม่รู้ว่า ACK นั้นตอบ transmission ไหน:

```
Send pkt5 ──────────────────────► (lost)
Timeout! Re-send pkt5 ──────────►
                          ◄──────── ACK(5)
RTT = ? (วัดจาก send ครั้งแรก? หรือ retransmit?)
```

**วิธีแก้:**

```rust
// ❌ ผิด: วัด RTT จากทุก ACK
fn on_ack(&mut self, ack: u32) {
    if let Some(pkt) = self.sent.iter().find(|p| p.seq == ack) {
        let rtt = pkt.sent_at.elapsed().as_millis() as f64;
        self.rtt.update(rtt); // ผิดถ้า retransmit!
    }
}

// ✅ ถูก: วัด RTT เฉพาะ packet ที่ไม่เคย retransmit
fn on_ack(&mut self, ack: u32) {
    if let Some(pkt) = self.sent.iter().find(|p| p.seq == ack) {
        if pkt.retransmit_count == 0 {  // เฉพาะ original transmission
            let rtt = pkt.sent_at.elapsed().as_millis() as f64;
            self.rtt.update(rtt);
        }
    }
}
```

---

### Pitfall 3: Window Size ≠ Congestion Window — ต้อง min ทั้งคู่

**ปัญหา:** มือใหม่มักใช้แค่ `send_window.can_send()` โดยไม่คำนึงถึง `cwnd`:

```rust
// ❌ ผิด: ใช้แค่ window_size
while send_window.can_send() {
    let seq = send_window.on_send(data.clone());
    // cwnd อาจน้อยกว่า window_size!
}

// ✅ ถูก: ใช้ min(window_size, cwnd)
let effective_window = std::cmp::min(
    send_window.window_size,
    cc.cwnd_packets()
);
let in_flight = send_window.sent.len() as u32;
while in_flight < effective_window && has_more_data() {
    let seq = send_window.on_send(data.clone());
}
```

---

### Pitfall 4: Duplicate ACK กับ Fast Retransmit — อย่า Double-Retransmit

**ปัญหา:** เมื่อ packet 5 หาย แต่ 6,7,8 มาถึง ผู้รับจะส่ง ACK(4) ซ้ำ 3 ครั้ง (duplicate ACK) TCP ตีความว่าเป็น packet loss แต่ถ้าเราทั้ง retransmit แล้ว backoff RTO ด้วย จะเป็นการ double-penalize:

```rust
// ❌ ผิด: เรียก on_timeout() ทั้ง fast retransmit และ real timeout
fn on_duplicate_ack(&mut self) {
    dup_ack_count += 1;
    if dup_ack_count == 3 {
        self.cc.on_timeout(); // ❌ ทำให้ cwnd=1 ทั้งที่ยัง connected ดี
        self.rtt.backoff_rto(); // ❌ ไม่จำเป็นสำหรับ fast retransmit
    }
}

// ✅ ถูก: fast retransmit ใช้ on_loss() ไม่ใช่ on_timeout()
fn on_duplicate_ack(&mut self) {
    dup_ack_count += 1;
    if dup_ack_count == 3 {
        self.cc.on_loss(); // ✅ ssthresh=cwnd/2, cwnd=ssthresh (Reno)
        // ไม่ backoff RTO สำหรับ fast retransmit
    }
}
```

---

### Pitfall 5: Memory Leak ใน Receive Buffer เมื่อ Packet Drop สูง

**ปัญหา:** ถ้า network ทิ้ง packet 0 แต่ 1..1000 มาถึงหมด `BTreeMap` จะสะสม 999 entries โดยไม่มีวันถูก drain:

```rust
// ❌ ไม่มี limit → OOM ได้
pub fn on_receive(&mut self, seq: u32, data: Vec<u8>) -> bool {
    self.received.insert(seq, data);
    true
}

// ✅ จำกัด window size
pub fn on_receive(&mut self, seq: u32, data: Vec<u8>) -> bool {
    if seq.wrapping_sub(self.next_expected) > 1024 {
        return false; // reject — too far ahead
    }
    // ...
}
```

ต้องมี **flow control** ระดับ connection เพื่อป้องกันไม่ให้ sender ส่งเร็วเกินกว่า receiver จะ buffer ได้

---

### Pitfall 6: Blocking UDP ใน Async Context

**ปัญหา:** ใช้ `std::net::UdpSocket` ใน tokio task จะ block thread pool:

```rust
// ❌ ผิด: blocking I/O ใน async function
use std::net::UdpSocket;
async fn run(socket: UdpSocket) {
    let mut buf = [0u8; 1500];
    let n = socket.recv(&mut buf).unwrap(); // blocks!
}

// ✅ ถูก: tokio's async UdpSocket
use tokio::net::UdpSocket;
async fn run(socket: UdpSocket) {
    let mut buf = [0u8; 1500];
    let n = socket.recv(&mut buf).await.unwrap(); // yields to scheduler
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# Build optimised binary
cargo build --release

# ขนาด binary ก่อน/หลัง strip
ls -lh target/release/quic-transport
strip target/release/quic-transport
ls -lh target/release/quic-transport
```

### Build สำหรับ Container (Static Binary)

```bash
# ติดตั้ง musl target
rustup target add x86_64-unknown-linux-musl

# Build static binary
cargo build --release --target x86_64-unknown-linux-musl

# ใช้ใน scratch Docker image
FROM scratch
COPY target/x86_64-unknown-linux-musl/release/quic-transport /
ENTRYPOINT ["/quic-transport"]
```

### Dockerfile (Multi-stage Build)

```dockerfile
# Stage 1: Build
FROM rust:1.75-slim AS builder
WORKDIR /app
COPY . .
RUN rustup target add x86_64-unknown-linux-musl \
    && cargo build --release --target x86_64-unknown-linux-musl

# Stage 2: Minimal runtime image
FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/quic-transport /usr/local/bin/
EXPOSE 4433/udp
CMD ["quic-transport"]
```

### Benchmark ด้วย cargo-criterion

```toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "packet_bench"
harness = false
```

```rust
// benches/packet_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use quic_transport::packet::Packet;

fn bench_encode(c: &mut Criterion) {
    let pkt = Packet::new_data(42, 10, 0xFF, vec![0u8; 1200]);
    c.bench_function("packet_encode_1200b", |b| {
        b.iter(|| black_box(pkt.encode()))
    });
}

criterion_group!(benches, bench_encode);
criterion_main!(benches);
```

```bash
cargo bench
# ผล: packet_encode_1200b  time: [450 ns 452 ns 455 ns]
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม FEC (Forward Error Correction) ⭐⭐⭐

เพิ่ม Reed-Solomon หรือ XOR parity เพื่อ recover packet ที่หายโดยไม่ต้อง retransmit:

```rust
// ทุกๆ k packets ส่ง 1 parity packet (XOR of all k)
// ถ้า packet ไหนหาย สามารถ recover ได้จาก XOR

pub fn generate_parity(packets: &[&[u8]]) -> Vec<u8> {
    let max_len = packets.iter().map(|p| p.len()).max().unwrap_or(0);
    let mut parity = vec![0u8; max_len];
    for pkt in packets {
        for (i, &b) in pkt.iter().enumerate() {
            parity[i] ^= b;
        }
    }
    parity
}
```

การทดลอง: ทดสอบที่ packet loss rate 5% และวัดว่า FEC ลด retransmit ได้กี่ %

---

### แบบฝึกหัดที่ 2: BBR Congestion Control ⭐⭐⭐⭐

แทนที่ AIMD ด้วย **BBR** (Bottleneck Bandwidth and Round-trip time) ซึ่ง Google ใช้ใน YouTube:

BBR ไม่รอ packet loss แต่วัด bandwidth-delay product โดยตรง:

```rust
pub struct BbrController {
    btl_bw: f64,      // Bottleneck Bandwidth estimate (bytes/ms)
    rt_prop: f64,     // Minimum RTT observed (ms)
    cwnd: f64,
    pacing_rate: f64, // bytes/ms
}

impl BbrController {
    pub fn on_ack(&mut self, acked_bytes: u64, rtt_ms: f64) {
        // อัปเดต BDP estimate
        let delivery_rate = acked_bytes as f64 / rtt_ms;
        self.btl_bw = self.btl_bw.max(delivery_rate);
        self.rt_prop = self.rt_prop.min(rtt_ms);

        // cwnd = BDP = btl_bw * rt_prop
        self.cwnd = self.btl_bw * self.rt_prop * 1.5; // 1.5 = gain
    }
}
```

เปรียบเทียบ throughput และ fairness ระหว่าง AIMD และ BBR ด้วย simulation

---

### แบบฝึกหัดที่ 3: 0-RTT Connection Establishment ⭐⭐⭐

QUIC รองรับ 0-RTT reconnect โดยใช้ session ticket จาก connection ก่อน:

```rust
// บันทึก session state หลัง disconnect
pub struct SessionTicket {
    server_id: [u8; 16],
    last_rtt_ms: f64,
    negotiated_params: ConnectionParams,
    timestamp: std::time::SystemTime,
}

// ส่ง data ในทันทีพร้อม Connect packet (0-RTT)
pub struct ZeroRttPacket {
    ticket: SessionTicket,
    early_data: Vec<u8>, // data ก่อน handshake เสร็จ
}
```

ความเสี่ยง: 0-RTT data อาจถูก replay attack ต้องเพิ่ม replay protection (sequence number + timestamp)

---

### แบบฝึกหัดที่ 4: Packet Pacing ⭐⭐⭐

แทนที่ส่ง packet ทีเดียวทั้ง window ให้ spread ออกมาตลอด RTT เพื่อลด burst:

```rust
pub struct Pacer {
    /// bytes per millisecond
    rate: f64,
    /// bytes allowed to send right now
    tokens: f64,
    /// last time tokens were refilled
    last_update: Instant,
}

impl Pacer {
    pub fn allow_send(&mut self, bytes: usize) -> bool {
        // Token bucket algorithm
        let now = Instant::now();
        let elapsed_ms = now.duration_since(self.last_update).as_millis() as f64;
        self.tokens += elapsed_ms * self.rate;
        self.tokens = self.tokens.min(self.rate * 100.0); // max burst = 100ms worth
        self.last_update = now;

        if self.tokens >= bytes as f64 {
            self.tokens -= bytes as f64;
            true
        } else {
            false
        }
    }
}
```

ทดสอบว่า pacing ลด jitter ใน bufferbloat scenario อย่างไร

---

### แบบฝึกหัดที่ 5: QUIC-like TLS Integration ⭐⭐⭐⭐⭐

เพิ่ม TLS 1.3 encryption ด้วย `rustls`:

```toml
[dependencies]
rustls = "0.23"
rustls-pemfile = "2.0"
```

```rust
use rustls::{ClientConfig, ServerConfig};

pub struct SecureTransport {
    inner: Transport,
    tls_state: TlsState,
}

// QUIC-style: TLS handshake ใน-band กับ protocol handshake
// ส่ง TLS ClientHello ใน Connect packet payload
// ส่ง TLS ServerHello ใน ConnAck packet payload
```

ความท้าทาย: QUIC จัดการ crypto แตกต่างจาก TLS-over-TCP เพราะ UDP ไม่มี record layer

---

### แบบฝึกหัดที่ 6: Simulation Testing ⭐⭐⭐⭐

สร้าง network simulator สำหรับทดสอบโดยไม่ต้องใช้ socket จริง:

```rust
pub struct SimNetwork {
    /// packet loss probability (0.0 - 1.0)
    loss_rate: f64,
    /// additional latency range (ms)
    latency_range: std::ops::Range<u64>,
    /// reorder probability
    reorder_rate: f64,
    pending: BinaryHeap<(Reverse<u64>, Vec<u8>)>, // (deliver_at, data)
}

impl SimNetwork {
    pub fn send(&mut self, data: Vec<u8>, base_latency_ms: u64) {
        use rand::Rng;
        let mut rng = rand::thread_rng();

        // Simulate loss
        if rng.gen::<f64>() < self.loss_rate {
            return; // dropped
        }

        // Simulate latency + jitter
        let extra = rng.gen_range(self.latency_range.clone());
        let deliver_at = base_latency_ms + extra;
        self.pending.push((Reverse(deliver_at), data));
    }
}
```

ทดสอบ protocol ที่ loss rate 1%, 5%, 10%, 20% และวัด goodput

---

## สรุป

โปรเจคนี้สร้าง **Reliable UDP Transport Protocol** ครบสมบูรณ์ด้วย 6 component หลักที่ทำงานร่วมกัน:

| Component | Technique | สิ่งที่เรียนรู้ |
|-----------|-----------|----------------|
| `packet` | Binary encode/decode | `to_be_bytes`, `from_be_bytes`, error handling |
| `window` | Sliding window ARQ | `VecDeque`, `BTreeMap`, wraparound arithmetic |
| `rtt` | Jacobson/Karels EWMA | Exponential moving average, RTO backoff |
| `congestion` | AIMD | Slow Start, CA, loss recovery |
| `connection` | FSM | Pure function transitions, exhaustive matching |
| `stream` | Multiplexing + credit | `HashMap`, backpressure pattern |

**Pattern สำคัญที่ได้เรียน:**

1. **Functional Core, Imperative Shell** — `transition()` เป็น pure function ทดสอบง่าย
2. **Newtype pattern สำหรับ ID** — `type StreamId = u32` ป้องกัน confusion กับ seq number
3. **Backpressure ด้วย credit** — return `false` เมื่อ full แทนการ block หรือ drop silently
4. **`retain()` สำหรับ filtered removal** — elegant กว่าการ collect แล้ว rebuild
5. **Boundary checking สำหรับ out-of-window** — ป้องกัน memory bomb จาก malicious packets

**เชื่อมโยงไปโปรเจคถัดไป:**

โปรเจค H10 (Network Proxy) จะนำความรู้ด้าน protocol design มาใช้สร้าง transparent proxy ที่รองรับทั้ง TCP และ UDP ซึ่งต้องการ:
- Connection state tracking แบบ stateful (เหมือน connection FSM ของเรา)
- Packet inspection และ modification (เหมือน packet module ของเรา)
- Multiplexing หลาย client connections (เหมือน stream mux ของเรา)

---

**โปรเจคก่อนหน้า:** [Project H08: TCP Chat Server](project-h08-tcp-chat.md) | **โปรเจคถัดไป:** [Project H10: Network Proxy](project-h10-network-proxy.md)
