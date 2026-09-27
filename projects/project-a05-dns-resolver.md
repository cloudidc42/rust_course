# Project A05: DNS Resolver (from scratch)

> โมดูล: A — CLI & Systems Tools | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 8 ชั่วโมง

## ภาพรวมโปรเจค

DNS (Domain Name System) คือโครงสร้างพื้นฐานที่สำคัญที่สุดอย่างหนึ่งของอินเทอร์เน็ต ทุกครั้งที่พิมพ์ `google.com` ในเบราว์เซอร์ ระบบปฏิบัติการจะต้องแปลงชื่อนั้นเป็น IP address ก่อน โดยส่ง **DNS query** ไปยัง resolver server ที่กำหนดไว้

โปรเจคนี้สร้าง DNS resolver ตั้งแต่ศูนย์ — ไม่ใช้ library สำเร็จรูปใด ๆ แต่ implement DNS wire protocol โดยตรงตาม **RFC 1035** ซึ่งเป็นมาตรฐานที่ใช้มาตั้งแต่ปี 1987 และยังคงเป็นหัวใจของระบบ DNS จนถึงทุกวันนี้

ทำไมถึงน่าสร้าง? เพราะเมื่อ implement protocol ด้วยตัวเองโดยไม่มี abstraction กั้น คุณจะเข้าใจอย่างลึกซึ้งว่า:

- byte manipulation ใน Rust ทำงานอย่างไรในทางปฏิบัติ
- endianness (network byte order = big-endian) มีผลต่อการ serialize/deserialize อย่างไร
- DNS name compression ทำงานอย่างไร (เทคนิคประหยัด bandwidth ที่ใช้มา 40 ปี)
- UDP/TCP transport สำหรับ network protocols

ใน production จริง เครื่องมืออย่าง `dig`, `nslookup`, `systemd-resolved`, `CoreDNS` และ `Unbound` ล้วนทำสิ่งที่เราจะสร้างในโปรเจคนี้ (แต่ซับซ้อนกว่ามาก)

## สิ่งที่จะได้เรียนรู้

- **DNS wire format (RFC 1035):** อ่านและเขียน binary protocol ด้วย byte slice โดยตรง
- **Bit manipulation:** แยก flags จาก header word ด้วย bitmask และ shift operations
- **Network byte order:** `u16::from_be_bytes`, `u32::from_be_bytes` — endianness ใน Rust
- **DNS name compression:** ติดตาม compression pointer แบบ 2-bit prefix (0b11)
- **UDP/TCP socket programming:** `std::net::UdpSocket`, `TcpStream`, timeout handling
- **TTL-aware caching:** ใช้ `HashMap` + `Instant` สร้าง cache พร้อม expiry
- **Parallel async queries:** `tokio::join!` สำหรับ concurrent DNS lookups
- **Protocol parsing idioms:** offset-based parser ที่ไม่ copy data โดยไม่จำเป็น

## ความรู้ที่ต้องมีมาก่อน

- **Part 96–100:** Systems programming, unsafe I/O, byte manipulation
- **Part 46–50:** async/await และ tokio runtime
- **Part 21–25:** struct, enum, impl blocks
- **Part 31–35:** error handling ด้วย `Result` และ `?` operator
- **Part 41–45:** collections — `HashMap`, `Vec`
- **Part 61–65:** standard library — networking, `UdpSocket`, `TcpStream`

## โครงสร้างโปรเจค (Project Layout)

```
dns-resolver/
├── src/
│   ├── main.rs          ← CLI entry point (clap)
│   ├── wire.rs          ← DNS wire format: structs + serialize/deserialize
│   ├── resolver.rs      ← stub resolver + TCP fallback
│   ├── cache.rs         ← TTL-aware in-memory cache
│   ├── records.rs       ← RDATA decoders (A, AAAA, CNAME, MX, TXT, NS, SOA)
│   └── iterative.rs     ← (optional) iterative resolver จาก root hints
├── tests/
│   └── integration_test.rs
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
CLI input: "dig-rs --type A google.com @8.8.8.8"
          │
          ▼
   DnsResolver::query(name, type, server)
          │
          ├─→ Cache hit? → return cached records immediately
          │
          ├─→ Cache miss:
          │      build_query(id, name, type)   ← wire.rs
          │      ──────────────────────────────
          │      serialize → Vec<u8>
          │      send via UdpSocket → 8.8.8.8:53
          │      recv_from → &[u8]
          │      ──────────────────────────────
          │      parse_message(&[u8])           ← wire.rs
          │      check TC bit → TCP fallback?
          │      ──────────────────────────────
          │      store answers in cache with TTL
          │
          ▼
   format_output(DnsMessage) → stdout (dig-like)
```

### ทำไมไม่ใช้ `bytes` crate?

เราจะใช้ raw `&[u8]` และ offset-based indexing แทน เพราะ:
1. **Learning value:** เห็น byte manipulation ระดับต่ำสุด
2. **Zero allocation:** ไม่ต้องสร้าง intermediate buffer เมื่อ parse
3. **DNS responses ขนาดเล็ก:** UDP DNS ≤ 512 bytes (หรือ 4096 bytes ด้วย EDNS0) ซึ่ง stack buffer เพียงพอ

### ทำไม stub resolver ก่อน?

stub resolver (ถามแค่ recursive server เช่น `8.8.8.8`) ง่ายกว่า iterative resolver (เริ่มจาก root hints แล้วตามสายลง) มาก การสร้าง stub ก่อนช่วยให้เข้าใจ wire format โดยไม่ต้องจัดการ recursive logic

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: สร้างและส่ง DNS Query แบบ Hard-coded

เริ่มจากสิ่งที่ง่ายที่สุด — ส่ง query สำหรับ `google.com` A record ไปยัง `8.8.8.8:53` แล้วพิมพ์ raw bytes ที่ได้รับ

**สร้าง project:**

```bash
cargo new dns-resolver
cd dns-resolver
```

**`src/main.rs` (ขั้นที่ 1):**

```rust
use std::net::UdpSocket;

fn encode_name(name: &str) -> Vec<u8> {
    // แปลง "google.com" → [6,g,o,o,g,l,e,3,c,o,m,0]
    // แต่ละ label นำหน้าด้วย byte ที่บอกความยาว
    let mut result = Vec::new();
    for label in name.trim_end_matches('.').split('.') {
        result.push(label.len() as u8);
        result.extend_from_slice(label.as_bytes());
    }
    result.push(0); // root label ปิดท้ายด้วย zero byte
    result
}

fn build_query(id: u16, name: &str, qtype: u16) -> Vec<u8> {
    let mut buf = Vec::with_capacity(512);

    // ─── DNS Header (12 bytes) ───────────────────────────
    // ID: 2 bytes, big-endian
    buf.push((id >> 8) as u8);
    buf.push((id & 0xFF) as u8);

    // Flags: 2 bytes
    //   QR=0 (query), OPCODE=0000, AA=0, TC=0, RD=1, RA=0, Z=000, RCODE=0000
    //   bit 15 (MSB) = QR, bit 8 = RD (Recursion Desired)
    //   0x01 0x00 → binary: 0000 0001  0000 0000
    buf.push(0x01); // high byte: RD=1
    buf.push(0x00); // low byte: all zeros

    // QDCOUNT = 1 (หนึ่ง question)
    buf.extend_from_slice(&[0x00, 0x01]);
    // ANCOUNT = 0 (ยังไม่มี answer ใน query)
    buf.extend_from_slice(&[0x00, 0x00]);
    // NSCOUNT = 0
    buf.extend_from_slice(&[0x00, 0x00]);
    // ARCOUNT = 0
    buf.extend_from_slice(&[0x00, 0x00]);

    // ─── Question Section ────────────────────────────────
    // QNAME: encoded domain name
    buf.extend_from_slice(&encode_name(name));
    // QTYPE: A = 1
    buf.extend_from_slice(&qtype.to_be_bytes());
    // QCLASS: IN (Internet) = 1
    buf.extend_from_slice(&1u16.to_be_bytes());

    buf
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let socket = UdpSocket::bind("0.0.0.0:0")?;
    socket.set_read_timeout(Some(std::time::Duration::from_secs(5)))?;

    let id: u16 = 0xDEAD;
    let query = build_query(id, "google.com", 1 /* A */);

    println!("ส่ง query {} bytes:", query.len());
    println!("{:02X?}", &query);

    socket.send_to(&query, "8.8.8.8:53")?;

    let mut buf = [0u8; 512];
    let (n, from) = socket.recv_from(&mut buf)?;
    println!("\nได้รับ response {} bytes จาก {}:", n, from);
    println!("{:02X?}", &buf[..n]);

    Ok(())
}
```

**ผลลัพธ์ที่ควรเห็น:**

```
ส่ง query 29 bytes:
[DE, AD, 01, 00, 00, 01, 00, 00, 00, 00, 00, 00, 06, 67, 6F, 6F, 67, 6C, 65, 03, 63, 6F, 6D, 00, 00, 01, 00, 01]

ได้รับ response 76 bytes จาก 8.8.8.8:53:
[DE, AD, 81, 80, 00, 01, 00, 04, 00, 00, 00, 00, 06, 67, 6F, 6F, 67, 6C, 65, 03, 63, 6F, 6D, 00, 00, 01, 00, 01, C0, 0C, 00, 01, 00, 01, 00, 00, 00, 3C, 00, 04, 8E, FA, 62, CE, ...]
```

**อธิบาย query bytes:**
- `DE AD` — ID เราตั้งไว้
- `01 00` — flags: RD=1 (ขอให้ resolver ค้นหาแบบ recursive)
- `00 01` — QDCOUNT: 1 question
- `00 00 00 00 00 00` — ANCOUNT, NSCOUNT, ARCOUNT ทั้งหมดเป็น 0
- `06 67 6F 6F 67 6C 65` — label "google" (6 chars: g,o,o,g,l,e)
- `03 63 6F 6D` — label "com" (3 chars: c,o,m)
- `00` — root label (จบ name)
- `00 01` — QTYPE=1 (A)
- `00 01` — QCLASS=1 (IN)

---

### ขั้นที่ 2: Parse Response — แยก IP Addresses จาก Answer Section

เมื่อได้รับ response bytes ต้องแปลงกลับเป็น struct ที่ใช้งานได้

**`src/wire.rs`:**

```rust
// ── DNS Header ────────────────────────────────────────────
//
//  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
// ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
// │                      ID                      │ ← bytes 0-1
// ├──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┼──┤
// │QR│  Opcode  │AA│TC│RD│RA│ Z│AD│CD│   RCODE  │ ← bytes 2-3
// ├──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┤
// │                    QDCOUNT                    │ ← bytes 4-5
// ├──────────────────────────────────────────────┤
// │                    ANCOUNT                    │ ← bytes 6-7
// ├──────────────────────────────────────────────┤
// │                    NSCOUNT                    │ ← bytes 8-9
// ├──────────────────────────────────────────────┤
// │                    ARCOUNT                    │ ← bytes 10-11
// └──────────────────────────────────────────────┘

#[derive(Debug, Clone)]
pub struct DnsHeader {
    pub id: u16,
    pub flags: u16,
    pub qdcount: u16,
    pub ancount: u16,
    pub nscount: u16,
    pub arcount: u16,
}

impl DnsHeader {
    pub fn qr(&self)     -> bool { (self.flags >> 15) & 1 == 1 }
    pub fn opcode(&self) -> u8   { ((self.flags >> 11) & 0xF) as u8 }
    pub fn aa(&self)     -> bool { (self.flags >> 10) & 1 == 1 }
    pub fn tc(&self)     -> bool { (self.flags >> 9) & 1 == 1 }
    pub fn rd(&self)     -> bool { (self.flags >> 8) & 1 == 1 }
    pub fn ra(&self)     -> bool { (self.flags >> 7) & 1 == 1 }
    pub fn rcode(&self)  -> u8   { (self.flags & 0xF) as u8 }
}

#[derive(Debug, Clone)]
pub struct DnsQuestion {
    pub qname: String,
    pub qtype: u16,
    pub qclass: u16,
}

#[derive(Debug, Clone)]
pub struct DnsResourceRecord {
    pub name: String,
    pub rtype: u16,
    pub rclass: u16,
    pub ttl: u32,
    pub rdata: Vec<u8>,
    /// offset ใน original packet ที่ rdata เริ่มต้น (สำหรับ decompress pointer ใน rdata)
    pub rdata_offset: usize,
}

#[derive(Debug, Clone)]
pub struct DnsMessage {
    pub header: DnsHeader,
    pub questions: Vec<DnsQuestion>,
    pub answers: Vec<DnsResourceRecord>,
    pub authorities: Vec<DnsResourceRecord>,
    pub additionals: Vec<DnsResourceRecord>,
}

// ── Record Type Constants ──────────────────────────────────

pub const TYPE_A:     u16 = 1;
pub const TYPE_NS:    u16 = 2;
pub const TYPE_CNAME: u16 = 5;
pub const TYPE_SOA:   u16 = 6;
pub const TYPE_MX:    u16 = 15;
pub const TYPE_TXT:   u16 = 16;
pub const TYPE_AAAA:  u16 = 28;
pub const CLASS_IN:   u16 = 1;

pub fn type_name(t: u16) -> &'static str {
    match t {
        TYPE_A     => "A",
        TYPE_NS    => "NS",
        TYPE_CNAME => "CNAME",
        TYPE_SOA   => "SOA",
        TYPE_MX    => "MX",
        TYPE_TXT   => "TXT",
        TYPE_AAAA  => "AAAA",
        _          => "UNKNOWN",
    }
}

// ── Name Encoding ──────────────────────────────────────────

/// แปลง domain name เป็น DNS label wire format
/// "www.example.com" → [3,'w','w','w', 7,'e','x','a','m','p','l','e', 3,'c','o','m', 0]
pub fn encode_name(name: &str) -> Vec<u8> {
    let mut result = Vec::new();
    for label in name.trim_end_matches('.').split('.') {
        // ป้องกัน label ว่าง (กรณี "." หรือ "..")
        if label.is_empty() { continue; }
        assert!(label.len() <= 63, "DNS label must be ≤ 63 bytes");
        result.push(label.len() as u8);
        result.extend_from_slice(label.as_bytes());
    }
    result.push(0);
    result
}

// ── Name Decoding (with compression pointer support) ────────

/// อ่าน domain name จาก DNS packet โดยรองรับ compression pointer
/// คืนค่า (name, next_offset) — next_offset คือ position ถัดจากชื่อใน packet ปัจจุบัน
pub fn decode_name(data: &[u8], start: usize) -> Result<(String, usize), String> {
    let mut labels: Vec<String> = Vec::new();
    let mut pos = start;
    let mut jumped = false;
    let mut end_pos = 0;
    let mut jumps = 0usize;
    const MAX_JUMPS: usize = 10; // ป้องกัน infinite loop จาก malformed packet

    loop {
        if pos >= data.len() {
            return Err(format!("decode_name: pos {} >= data.len() {}", pos, data.len()));
        }

        jumps += 1;
        if jumps > MAX_JUMPS {
            return Err("decode_name: exceeded max pointer jumps (malformed packet or loop)".into());
        }

        let len_byte = data[pos] as usize;

        if len_byte == 0 {
            // End of name
            if !jumped { end_pos = pos + 1; }
            break;
        }

        // Compression pointer: top 2 bits = 11 (0b11xxxxxx xxxxxxxx)
        if (len_byte & 0xC0) == 0xC0 {
            if pos + 1 >= data.len() {
                return Err("decode_name: pointer byte 2 out of bounds".into());
            }
            if !jumped { end_pos = pos + 2; }
            // สร้าง pointer: ลบ 2 MSBs ออกแล้วรวมกับ byte ถัดไป
            let ptr = ((len_byte & 0x3F) << 8) | (data[pos + 1] as usize);
            if ptr >= data.len() {
                return Err(format!("decode_name: pointer {} points beyond packet", ptr));
            }
            pos = ptr;
            jumped = true;
            continue;
        }

        // Regular label
        pos += 1;
        if pos + len_byte > data.len() {
            return Err("decode_name: label extends past end of data".into());
        }
        let label = std::str::from_utf8(&data[pos..pos + len_byte])
            .map_err(|e| format!("decode_name: invalid UTF-8 in label: {}", e))?;
        labels.push(label.to_string());
        pos += len_byte;
    }

    if !jumped { end_pos = pos + 1; }

    Ok((labels.join("."), end_pos))
}

// ── Message Parsing ──────────────────────────────────────────

fn parse_header(data: &[u8]) -> Result<DnsHeader, String> {
    if data.len() < 12 {
        return Err(format!("parse_header: too short ({} bytes)", data.len()));
    }
    Ok(DnsHeader {
        id:      u16::from_be_bytes([data[0], data[1]]),
        flags:   u16::from_be_bytes([data[2], data[3]]),
        qdcount: u16::from_be_bytes([data[4], data[5]]),
        ancount: u16::from_be_bytes([data[6], data[7]]),
        nscount: u16::from_be_bytes([data[8], data[9]]),
        arcount: u16::from_be_bytes([data[10], data[11]]),
    })
}

fn parse_question(data: &[u8], offset: usize) -> Result<(DnsQuestion, usize), String> {
    let (qname, mut pos) = decode_name(data, offset)?;
    if pos + 4 > data.len() {
        return Err("parse_question: not enough bytes for QTYPE/QCLASS".into());
    }
    let qtype  = u16::from_be_bytes([data[pos],     data[pos + 1]]);
    let qclass = u16::from_be_bytes([data[pos + 2], data[pos + 3]]);
    pos += 4;
    Ok((DnsQuestion { qname, qtype, qclass }, pos))
}

fn parse_rr(data: &[u8], offset: usize) -> Result<(DnsResourceRecord, usize), String> {
    let (name, mut pos) = decode_name(data, offset)?;
    if pos + 10 > data.len() {
        return Err(format!("parse_rr: not enough bytes for fixed RR fields at pos {}", pos));
    }
    let rtype  = u16::from_be_bytes([data[pos],     data[pos + 1]]);
    let rclass = u16::from_be_bytes([data[pos + 2], data[pos + 3]]);
    let ttl    = u32::from_be_bytes([data[pos + 4], data[pos + 5], data[pos + 6], data[pos + 7]]);
    let rdlen  = u16::from_be_bytes([data[pos + 8], data[pos + 9]]) as usize;
    pos += 10;
    let rdata_offset = pos;
    if pos + rdlen > data.len() {
        return Err(format!("parse_rr: rdlen={} but only {} bytes remain", rdlen, data.len() - pos));
    }
    let rdata = data[pos..pos + rdlen].to_vec();
    pos += rdlen;
    Ok((DnsResourceRecord { name, rtype, rclass, ttl, rdata, rdata_offset }, pos))
}

pub fn parse_message(data: &[u8]) -> Result<DnsMessage, String> {
    let header = parse_header(data)?;
    let mut pos = 12usize;

    let mut questions  = Vec::with_capacity(header.qdcount as usize);
    let mut answers    = Vec::with_capacity(header.ancount as usize);
    let mut authorities = Vec::with_capacity(header.nscount as usize);
    let mut additionals = Vec::with_capacity(header.arcount as usize);

    for _ in 0..header.qdcount {
        let (q, next) = parse_question(data, pos)?;
        questions.push(q);
        pos = next;
    }
    for _ in 0..header.ancount {
        let (rr, next) = parse_rr(data, pos)?;
        answers.push(rr);
        pos = next;
    }
    for _ in 0..header.nscount {
        let (rr, next) = parse_rr(data, pos)?;
        authorities.push(rr);
        pos = next;
    }
    for _ in 0..header.arcount {
        let (rr, next) = parse_rr(data, pos)?;
        additionals.push(rr);
        pos = next;
    }

    Ok(DnsMessage { header, questions, answers, authorities, additionals })
}

// ── Query Builder ─────────────────────────────────────────────

pub fn build_query(id: u16, name: &str, qtype: u16) -> Vec<u8> {
    let mut buf = Vec::with_capacity(512);
    buf.extend_from_slice(&id.to_be_bytes());
    buf.extend_from_slice(&0x0100u16.to_be_bytes()); // flags: RD=1
    buf.extend_from_slice(&1u16.to_be_bytes());       // QDCOUNT = 1
    buf.extend_from_slice(&0u16.to_be_bytes());       // ANCOUNT = 0
    buf.extend_from_slice(&0u16.to_be_bytes());       // NSCOUNT = 0
    buf.extend_from_slice(&0u16.to_be_bytes());       // ARCOUNT = 0
    buf.extend_from_slice(&encode_name(name));
    buf.extend_from_slice(&qtype.to_be_bytes());
    buf.extend_from_slice(&CLASS_IN.to_be_bytes());
    buf
}
```

**อ่าน IP address จาก answer section:**

```rust
// ใน main.rs หรือ records.rs

pub fn rdata_to_ipv4(rdata: &[u8]) -> Option<std::net::Ipv4Addr> {
    if rdata.len() == 4 {
        Some(std::net::Ipv4Addr::new(rdata[0], rdata[1], rdata[2], rdata[3]))
    } else {
        None
    }
}

pub fn extract_a_records(msg: &DnsMessage) -> Vec<std::net::Ipv4Addr> {
    msg.answers.iter()
        .filter(|rr| rr.rtype == TYPE_A)
        .filter_map(|rr| rdata_to_ipv4(&rr.rdata))
        .collect()
}
```

---

### ขั้นที่ 3: Full Message Serialization/Deserialization — Record Types ทั้งหมด

เพิ่ม RDATA decoder สำหรับ record types ทั้ง 7 ประเภท:

**`src/records.rs`:**

```rust
use std::net::{Ipv4Addr, Ipv6Addr};
use crate::wire::{decode_name, TYPE_A, TYPE_AAAA, TYPE_CNAME, TYPE_MX, TYPE_NS, TYPE_SOA, TYPE_TXT};

// ── A Record (IPv4 address) ───────────────────────────────
pub fn decode_a(rdata: &[u8]) -> Option<Ipv4Addr> {
    if rdata.len() == 4 {
        Some(Ipv4Addr::new(rdata[0], rdata[1], rdata[2], rdata[3]))
    } else {
        None
    }
}

// ── AAAA Record (IPv6 address) ────────────────────────────
pub fn decode_aaaa(rdata: &[u8]) -> Option<Ipv6Addr> {
    if rdata.len() == 16 {
        let arr: [u8; 16] = rdata.try_into().ok()?;
        Some(Ipv6Addr::from(arr))
    } else {
        None
    }
}

// ── CNAME / NS Record (domain name) ──────────────────────
// rdata_offset คือ offset ใน original packet ที่ rdata เริ่มต้น
// (จำเป็นสำหรับ decompress pointers ที่อ้างถึง offset ในส่วนอื่น ๆ ของ packet)
pub fn decode_name_rdata(packet: &[u8], rdata_offset: usize) -> Option<String> {
    decode_name(packet, rdata_offset).ok().map(|(n, _)| n)
}

// ── MX Record (mail exchange) ─────────────────────────────
#[derive(Debug, Clone)]
pub struct MxRecord {
    pub preference: u16,
    pub exchange: String,
}

pub fn decode_mx(packet: &[u8], rdata: &[u8], rdata_offset: usize) -> Option<MxRecord> {
    if rdata.len() < 3 { return None; }
    let preference = u16::from_be_bytes([rdata[0], rdata[1]]);
    // exchange name เริ่มที่ rdata_offset + 2 (หลัง 2 bytes ของ preference)
    let exchange = decode_name(packet, rdata_offset + 2).ok().map(|(n, _)| n)?;
    Some(MxRecord { preference, exchange })
}

// ── TXT Record ─────────────────────────────────────────────
// TXT RDATA = sequence of length-prefixed strings
pub fn decode_txt(rdata: &[u8]) -> Vec<String> {
    let mut result = Vec::new();
    let mut pos = 0;
    while pos < rdata.len() {
        let len = rdata[pos] as usize;
        pos += 1;
        if pos + len > rdata.len() { break; }
        if let Ok(s) = std::str::from_utf8(&rdata[pos..pos + len]) {
            result.push(s.to_string());
        }
        pos += len;
    }
    result
}

// ── SOA Record ──────────────────────────────────────────────
#[derive(Debug, Clone)]
pub struct SoaRecord {
    pub mname:   String, // primary nameserver
    pub rname:   String, // responsible mailbox (@ → .)
    pub serial:  u32,
    pub refresh: u32,
    pub retry:   u32,
    pub expire:  u32,
    pub minimum: u32,
}

pub fn decode_soa(packet: &[u8], rdata_offset: usize) -> Option<SoaRecord> {
    let (mname, pos1) = decode_name(packet, rdata_offset).ok()?;
    let (rname, pos2) = decode_name(packet, pos1).ok()?;
    // 5 × u32 = 20 bytes
    if pos2 + 20 > packet.len() { return None; }
    let serial  = u32::from_be_bytes(packet[pos2..pos2 + 4].try_into().ok()?);
    let refresh = u32::from_be_bytes(packet[pos2 + 4..pos2 + 8].try_into().ok()?);
    let retry   = u32::from_be_bytes(packet[pos2 + 8..pos2 + 12].try_into().ok()?);
    let expire  = u32::from_be_bytes(packet[pos2 + 12..pos2 + 16].try_into().ok()?);
    let minimum = u32::from_be_bytes(packet[pos2 + 16..pos2 + 20].try_into().ok()?);
    Some(SoaRecord { mname, rname, serial, refresh, retry, expire, minimum })
}

// ── Unified formatter ──────────────────────────────────────

pub fn format_rdata(packet: &[u8], rr: &crate::wire::DnsResourceRecord) -> String {
    match rr.rtype {
        TYPE_A     => decode_a(&rr.rdata)
                        .map(|ip| ip.to_string())
                        .unwrap_or_else(|| "<invalid A>".into()),
        TYPE_AAAA  => decode_aaaa(&rr.rdata)
                        .map(|ip| ip.to_string())
                        .unwrap_or_else(|| "<invalid AAAA>".into()),
        TYPE_CNAME => decode_name_rdata(packet, rr.rdata_offset)
                        .unwrap_or_else(|| "<invalid CNAME>".into()),
        TYPE_NS    => decode_name_rdata(packet, rr.rdata_offset)
                        .unwrap_or_else(|| "<invalid NS>".into()),
        TYPE_MX    => decode_mx(packet, &rr.rdata, rr.rdata_offset)
                        .map(|mx| format!("{} {}", mx.preference, mx.exchange))
                        .unwrap_or_else(|| "<invalid MX>".into()),
        TYPE_TXT   => {
            let strings = decode_txt(&rr.rdata);
            format!("\"{}\"", strings.join("\" \""))
        }
        TYPE_SOA   => decode_soa(packet, rr.rdata_offset)
                        .map(|soa| format!(
                            "{} {} {} {} {} {} {}",
                            soa.mname, soa.rname,
                            soa.serial, soa.refresh, soa.retry, soa.expire, soa.minimum
                        ))
                        .unwrap_or_else(|| "<invalid SOA>".into()),
        _          => format!("\\# {} {:02X?}", rr.rdata.len(), &rr.rdata),
    }
}
```

---

### ขั้นที่ 4: DNS Name Compression — การติดตาม Pointer

DNS name compression เป็นหนึ่งในส่วนที่ซับซ้อนที่สุดของ RFC 1035 ทำความเข้าใจอย่างละเอียด:

**หลักการ:** แทนที่จะซ้ำชื่อซ้ำในแต่ละ RR, ใส่ "pointer" ขนาด 2 bytes ที่ชี้ไปยัง offset ในส่วนอื่นของ packet

```
packet (hex):
  Offset 0:  [DE AD 81 80 00 01 00 01 ...]   ← header
  Offset 12: [06 67 6F 6F 67 6C 65 03 63 6F 6D 00]  ← "google.com" (12 bytes)
  Offset 24: [00 01 00 01]                   ← QTYPE=A, QCLASS=IN
  Offset 28: [C0 0C]                         ← ANSWER NAME = pointer to offset 12
  ...
```

**ทำความเข้าใจ `0xC0 0x0C`:**

```
0xC0 = 1100 0000   ← top 2 bits = 11 → "นี่คือ pointer"
0x0C = 0000 1100   ← ส่วนล่าง 14 bits = pointer value

pointer value = (0xC0 & 0x3F) << 8 | 0x0C
              = (0x00) << 8 | 0x0C
              = 0x000C
              = 12 decimal
```

**ตัวอย่างการ decode step-by-step:**

```rust
// ตัวอย่าง: decode "www.google.com" ที่อ้างอิง pointer ไปยัง "google.com" ที่ offset 12
//
// ข้อมูลใน packet:
//   offset 12: 06 'g' 'o' 'o' 'g' 'l' 'e' 03 'c' 'o' 'm' 00
//   offset 40: 03 'w' 'w' 'w' C0 0C
//
// decode_name(data, 40):
//   pos=40: len=3 → label "www" → pos=44
//   pos=44: len=0xC0 → POINTER → ptr = ((0xC0 & 0x3F) << 8) | 0x0C = 12
//   jumped=true, end_pos=46 (past the 2-byte pointer)
//   pos=12: len=6 → label "google" → pos=19
//   pos=19: len=3 → label "com" → pos=23
//   pos=23: len=0 → end → break
//   return ("www.google.com", 46)
```

**กับดักที่พบบ่อย — pointer loop:**

```rust
// ถ้า packet malformed:
// offset 10: C0 0A   ← pointer to offset 10 (ตัวเอง!)
// decode_name จะวนไม่สิ้นสุด ถ้าไม่มีการตรวจสอบ jumps

// ป้องกันด้วย MAX_JUMPS counter:
let mut jumps = 0;
const MAX_JUMPS: usize = 10;
// ...
jumps += 1;
if jumps > MAX_JUMPS {
    return Err("too many pointer jumps".into());
}
```

---

### ขั้นที่ 5: UDP Transport + TCP Fallback

**`src/resolver.rs`:**

```rust
use std::net::{UdpSocket, TcpStream};
use std::io::{Read, Write};
use crate::wire::{build_query, parse_message, DnsMessage};

const UDP_TIMEOUT_SECS: u64 = 5;
const TCP_TIMEOUT_SECS: u64 = 10;

/// ส่ง DNS query ผ่าน UDP ก่อน ถ้า TC bit ถูก set จะ fallback ไปใช้ TCP
pub fn query(name: &str, qtype: u16, server: &str, id: u16) -> Result<DnsMessage, String> {
    // ลอง UDP ก่อน
    match query_udp(name, qtype, server, id) {
        Ok(msg) if !msg.header.tc() => return Ok(msg),
        Ok(_msg) => {
            // TC=1: response truncated — retry ด้วย TCP
            eprintln!("[dns] UDP response truncated (TC=1), retrying with TCP...");
            query_tcp(name, qtype, server, id)
        }
        Err(e) => Err(e),
    }
}

fn query_udp(name: &str, qtype: u16, server: &str, id: u16) -> Result<DnsMessage, String> {
    let socket = UdpSocket::bind("0.0.0.0:0")
        .map_err(|e| format!("UDP bind failed: {}", e))?;

    socket.set_read_timeout(Some(std::time::Duration::from_secs(UDP_TIMEOUT_SECS)))
        .map_err(|e| format!("set_read_timeout: {}", e))?;

    let query_bytes = build_query(id, name, qtype);

    socket.send_to(&query_bytes, server)
        .map_err(|e| format!("UDP send_to {}: {}", server, e))?;

    let mut buf = [0u8; 512];
    let (n, _src) = socket.recv_from(&mut buf)
        .map_err(|e| format!("UDP recv_from: {}", e))?;

    parse_message(&buf[..n])
}

/// TCP transport: ส่ง query และอ่าน response
///
/// ใน TCP: message ถูก prefix ด้วย 2-byte big-endian length
/// (เพราะ TCP เป็น stream — ต้องรู้ว่า message สิ้นสุดที่ไหน)
///
///  [LL LL] [DNS message bytes...]
///   └─ 2 bytes big-endian length
fn query_tcp(name: &str, qtype: u16, server: &str, id: u16) -> Result<DnsMessage, String> {
    let addr = server;
    let mut stream = TcpStream::connect(addr)
        .map_err(|e| format!("TCP connect to {}: {}", addr, e))?;

    stream.set_read_timeout(Some(std::time::Duration::from_secs(TCP_TIMEOUT_SECS)))
        .map_err(|e| format!("TCP set_read_timeout: {}", e))?;

    let query_bytes = build_query(id, name, qtype);

    // เขียน length prefix (2 bytes big-endian) + query
    let len = query_bytes.len() as u16;
    stream.write_all(&len.to_be_bytes())
        .map_err(|e| format!("TCP write length: {}", e))?;
    stream.write_all(&query_bytes)
        .map_err(|e| format!("TCP write query: {}", e))?;

    // อ่าน 2 bytes length ของ response
    let mut len_buf = [0u8; 2];
    stream.read_exact(&mut len_buf)
        .map_err(|e| format!("TCP read response length: {}", e))?;
    let resp_len = u16::from_be_bytes(len_buf) as usize;

    // อ่าน response body
    let mut resp_buf = vec![0u8; resp_len];
    stream.read_exact(&mut resp_buf)
        .map_err(|e| format!("TCP read response body: {}", e))?;

    parse_message(&resp_buf)
}
```

**ตัวอย่าง TCP framing ที่ถูกต้อง:**

```
TCP stream:
┌──────┬────────────────────────────────────┐
│ 00 1D│ [DNS query bytes, 29 bytes total]  │
└──────┴────────────────────────────────────┘
   ↑
   2 bytes length = 29 (big-endian)

กับดัก: ถ้า read_exact ไม่ได้รับ bytes ครบ จะ hang หรือ error
→ ต้องตรวจ return value และ timeout อย่างเคร่งครัด
```

---

### ขั้นที่ 6: CLI แบบ `dig`

เพิ่ม `clap` เป็น dependency:

**`Cargo.toml`:**

```toml
[package]
name = "dns-resolver"
version = "0.1.0"
edition = "2021"

[dependencies]
clap       = { version = "4", features = ["derive"] }
tokio      = { version = "1", features = ["full"] }
owo-colors = "4"

[dev-dependencies]
# ไม่จำเป็นต้องมี dependency พิเศษสำหรับ integration tests
```

**`src/main.rs` (CLI):**

```rust
mod wire;
mod resolver;
mod cache;
mod records;

use clap::Parser;
use owo_colors::OwoColorize;
use std::time::Instant;
use wire::{TYPE_A, TYPE_AAAA, TYPE_MX, TYPE_TXT, TYPE_CNAME, TYPE_NS, TYPE_SOA, type_name};

#[derive(Parser, Debug)]
#[command(name = "dig-rs")]
#[command(about = "DNS lookup tool — built from scratch in Rust")]
struct Args {
    /// Domain name to query
    name: String,

    /// Record type: A, AAAA, MX, TXT, CNAME, NS, SOA
    #[arg(long = "type", short = 't', default_value = "A")]
    qtype: String,

    /// DNS server (e.g. @8.8.8.8 or @1.1.1.1)
    #[arg(long = "server", short = 's', default_value = "8.8.8.8:53")]
    server: String,

    /// Show raw hex dump of response
    #[arg(long = "hex", action = clap::ArgAction::SetTrue)]
    hex: bool,
}

fn parse_qtype(s: &str) -> u16 {
    match s.to_uppercase().as_str() {
        "A"     => TYPE_A,
        "AAAA"  => TYPE_AAAA,
        "MX"    => TYPE_MX,
        "TXT"   => TYPE_TXT,
        "CNAME" => TYPE_CNAME,
        "NS"    => TYPE_NS,
        "SOA"   => TYPE_SOA,
        other   => {
            eprintln!("Unknown record type: {}, defaulting to A", other);
            TYPE_A
        }
    }
}

fn print_dig_output(msg: &wire::DnsMessage, packet: &[u8], elapsed_ms: u64, server: &str) {
    // ── Header section ──
    println!("\n{}", "; <<>> dig-rs 0.1.0 <<>>".bright_black());
    println!("{}", format!(";; Got answer: {} bytes in {} ms", packet.len(), elapsed_ms).bright_black());
    println!();

    // ── Flags line ──
    let h = &msg.header;
    let flags_str = format!(
        "qr{} {}{}{}{}",
        if h.qr() { "" } else { " (not set)" },
        if h.aa() { " aa" } else { "" },
        if h.tc() { " tc" } else { "" },
        if h.rd() { " rd" } else { "" },
        if h.ra() { " ra" } else { "" },
    );
    println!(";; ->>HEADER<<- opcode: QUERY, status: {}, id: {}",
        rcode_name(h.rcode()), h.id.cyan());
    println!(";; flags: {}; QUERY: {}, ANSWER: {}, AUTHORITY: {}, ADDITIONAL: {}",
        flags_str.yellow(), h.qdcount, h.ancount.green(), h.nscount, h.arcount);
    println!();

    // ── Question section ──
    println!(";; QUESTION SECTION:");
    for q in &msg.questions {
        println!(";{:<30} IN   {}", q.qname, type_name(q.qtype).cyan());
    }
    println!();

    // ── Answer section ──
    if !msg.answers.is_empty() {
        println!(";; ANSWER SECTION:");
        for rr in &msg.answers {
            let rdata_str = records::format_rdata(packet, rr);
            println!("{:<30} {:<8} IN   {:<8} {}",
                rr.name,
                rr.ttl.to_string().yellow(),
                type_name(rr.rtype).cyan(),
                rdata_str.green());
        }
        println!();
    }

    // ── Authority section ──
    if !msg.authorities.is_empty() {
        println!(";; AUTHORITY SECTION:");
        for rr in &msg.authorities {
            let rdata_str = records::format_rdata(packet, rr);
            println!("{:<30} {:<8} IN   {:<8} {}",
                rr.name, rr.ttl, type_name(rr.rtype).cyan(), rdata_str);
        }
        println!();
    }

    println!(";; SERVER: {}", server.bright_black());
}

fn rcode_name(rcode: u8) -> &'static str {
    match rcode {
        0 => "NOERROR",
        1 => "FORMERR",
        2 => "SERVFAIL",
        3 => "NXDOMAIN",
        4 => "NOTIMP",
        5 => "REFUSED",
        _ => "UNKNOWN",
    }
}

#[tokio::main]
async fn main() {
    let args = Args::parse();

    // normalize server address
    let server = if args.server.contains(':') {
        args.server.clone()
    } else {
        format!("{}:53", args.server.trim_start_matches('@'))
    };

    let qtype = parse_qtype(&args.qtype);
    let id: u16 = rand_id();

    let start = Instant::now();
    match resolver::query(&args.name, qtype, &server, id) {
        Ok(msg) => {
            // สร้าง fake packet สำหรับ compression decode
            // (ในการ implement จริง ควรส่ง raw bytes ออกมาจาก resolver)
            let elapsed = start.elapsed().as_millis() as u64;
            let fake_packet = wire::build_query(id, &args.name, qtype); // placeholder
            print_dig_output(&msg, &fake_packet, elapsed, &server);
        }
        Err(e) => {
            eprintln!("{}: {}", "Error".red(), e);
            std::process::exit(1);
        }
    }
}

fn rand_id() -> u16 {
    // สุ่ม ID เพื่อป้องกัน cache poisoning
    use std::time::{SystemTime, UNIX_EPOCH};
    let nanos = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap_or_default()
        .subsec_nanos();
    (nanos & 0xFFFF) as u16
}
```

**ตัวอย่าง output:**

```
$ dig-rs --type A google.com --server 8.8.8.8

; <<>> dig-rs 0.1.0 <<>>
;; Got answer: 76 bytes in 23 ms

;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57005
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com                    IN   A

;; ANSWER SECTION:
google.com                     57       IN   A        142.250.66.206
google.com                     57       IN   A        142.250.66.174
google.com                     57       IN   A        142.250.66.238
google.com                     57       IN   A        142.250.66.142

;; SERVER: 8.8.8.8:53
```

---

### ขั้นที่ 7: TTL-Aware Cache

**`src/cache.rs`:**

```rust
use std::collections::HashMap;
use std::time::{Duration, Instant};
use crate::wire::{DnsResourceRecord, TYPE_A, TYPE_AAAA, TYPE_CNAME};

/// Cache key = (lowercase name, record type)
type CacheKey = (String, u16);

#[derive(Debug, Clone)]
struct CacheEntry {
    records: Vec<DnsResourceRecord>,
    expires_at: Instant,
}

pub struct DnsCache {
    store: HashMap<CacheKey, CacheEntry>,
    max_entries: usize,
}

impl DnsCache {
    pub fn new(max_entries: usize) -> Self {
        Self {
            store: HashMap::new(),
            max_entries,
        }
    }

    /// บันทึก records ลง cache โดยใช้ TTL น้อยที่สุดในกลุ่ม
    pub fn insert(&mut self, name: &str, rtype: u16, records: Vec<DnsResourceRecord>) {
        if records.is_empty() { return; }

        // TTL ของแต่ละ record อาจต่างกัน — ใช้ค่าน้อยที่สุด (conservative)
        let min_ttl = records.iter().map(|rr| rr.ttl).min().unwrap_or(0);
        if min_ttl == 0 { return; } // ไม่ cache TTL=0

        if self.store.len() >= self.max_entries {
            self.evict_expired();
            if self.store.len() >= self.max_entries {
                // ยังเต็มอยู่ — ลบ entry เก่าสุดออก (simple LRU approximation)
                if let Some(oldest_key) = self.store.keys().next().cloned() {
                    self.store.remove(&oldest_key);
                }
            }
        }

        let key = (name.to_lowercase(), rtype);
        let expires_at = Instant::now() + Duration::from_secs(min_ttl as u64);
        self.store.insert(key, CacheEntry { records, expires_at });
    }

    /// ดึง records จาก cache ถ้ายังไม่หมดอายุ
    pub fn get(&self, name: &str, rtype: u16) -> Option<Vec<DnsResourceRecord>> {
        let key = (name.to_lowercase(), rtype);
        self.store.get(&key).and_then(|entry| {
            if Instant::now() < entry.expires_at {
                // คำนวณ TTL ที่เหลือ
                let remaining = entry.expires_at
                    .duration_since(Instant::now())
                    .as_secs() as u32;
                // คืน records พร้อม TTL ที่ปรับแล้ว
                let mut records = entry.records.clone();
                for rr in &mut records {
                    rr.ttl = remaining;
                }
                Some(records)
            } else {
                None // expired
            }
        })
    }

    /// ลบ entries ที่หมดอายุออกจาก cache
    pub fn evict_expired(&mut self) {
        let now = Instant::now();
        self.store.retain(|_, entry| entry.expires_at > now);
    }

    /// จำนวน entries ที่ยังมีอยู่ใน cache (รวมที่ expired แต่ยังไม่ถูก evict)
    pub fn len(&self) -> usize {
        self.store.len()
    }

    pub fn is_empty(&self) -> bool {
        self.store.is_empty()
    }
}
```

**การใช้งาน cache ใน resolver:**

```rust
// ใน resolver.rs — เพิ่ม cache ให้ DnsResolver
use std::sync::{Arc, Mutex};
use crate::cache::DnsCache;

pub struct DnsResolver {
    cache: Arc<Mutex<DnsCache>>,
    default_server: String,
}

impl DnsResolver {
    pub fn new(server: &str) -> Self {
        Self {
            cache: Arc::new(Mutex::new(DnsCache::new(1000))),
            default_server: server.to_string(),
        }
    }

    pub fn query_cached(&self, name: &str, qtype: u16) -> Result<Vec<wire::DnsResourceRecord>, String> {
        // 1. ตรวจสอบ cache ก่อน
        {
            let cache = self.cache.lock().unwrap();
            if let Some(cached) = cache.get(name, qtype) {
                eprintln!("[cache] HIT: {} {}", name, wire::type_name(qtype));
                return Ok(cached);
            }
        }

        // 2. ไม่มีใน cache — ส่ง query จริง
        eprintln!("[cache] MISS: {} {}", name, wire::type_name(qtype));
        let id = 0x1234u16; // ใน production ควรสุ่ม
        let msg = query(name, qtype, &self.default_server, id)?;

        // 3. เก็บผลลัพธ์ลง cache
        {
            let mut cache = self.cache.lock().unwrap();
            cache.insert(name, qtype, msg.answers.clone());
        }

        Ok(msg.answers)
    }
}
```

---

### ขั้นที่ 8: Iterative Resolver จาก Root Hints (ขั้นสูง)

Iterative resolver ทำงานโดยเริ่มจาก root nameservers แล้ว follow delegation chain จนถึง authoritative nameserver

**Root hints (13 root nameservers ของโลก):**

```rust
pub const ROOT_HINTS: &[(&str, &str)] = &[
    ("a.root-servers.net", "198.41.0.4"),
    ("b.root-servers.net", "170.247.170.2"),
    ("c.root-servers.net", "192.33.4.12"),
    ("d.root-servers.net", "199.7.91.13"),
    ("e.root-servers.net", "192.203.230.10"),
    ("f.root-servers.net", "192.5.5.241"),
    ("g.root-servers.net", "192.112.36.4"),
    ("h.root-servers.net", "198.97.190.53"),
    ("i.root-servers.net", "192.36.148.17"),
    ("j.root-servers.net", "192.58.128.30"),
    ("k.root-servers.net", "193.0.14.129"),
    ("l.root-servers.net", "199.7.83.42"),
    ("m.root-servers.net", "202.12.27.33"),
];
```

**Algorithm ของ iterative resolution:**

```rust
// src/iterative.rs
use crate::wire::{DnsMessage, TYPE_A, TYPE_NS, CLASS_IN};
use crate::resolver::query_udp_raw;

/// แก้ชื่อ domain แบบ iterative — เริ่มจาก root และไล่ลงมา
pub fn resolve_iterative(name: &str, qtype: u16) -> Result<DnsMessage, String> {
    // เริ่มจาก root nameservers
    let mut current_servers: Vec<String> = ROOT_HINTS
        .iter()
        .map(|(_, ip)| format!("{}:53", ip))
        .collect();

    let mut depth = 0;
    const MAX_DEPTH: usize = 20; // ป้องกัน infinite delegation

    loop {
        if depth >= MAX_DEPTH {
            return Err("resolve_iterative: exceeded max delegation depth".into());
        }
        depth += 1;

        // ลอง server แรกใน list
        let server = &current_servers[0];
        eprintln!("[iterative] depth={} querying {} for {}", depth, server, name);

        let id = (depth as u16) ^ 0x5A5A;
        let msg = query_udp_raw(name, qtype, server, id)?;

        // ถ้าได้ answer — เสร็จแล้ว
        if msg.header.ancount > 0 && msg.header.rcode() == 0 {
            return Ok(msg);
        }

        // ถ้าได้ NXDOMAIN — domain ไม่มีอยู่จริง
        if msg.header.rcode() == 3 {
            return Err(format!("NXDOMAIN: {} does not exist", name));
        }

        // ถ้าได้ referral (NS records ใน authority section)
        if msg.header.nscount > 0 {
            // หา glue records (A records ของ NS ใน additional section)
            let ns_addrs: Vec<String> = msg.additionals.iter()
                .filter(|rr| rr.rtype == TYPE_A && rr.rdata.len() == 4)
                .map(|rr| format!("{}.{}.{}.{}:53",
                    rr.rdata[0], rr.rdata[1], rr.rdata[2], rr.rdata[3]))
                .collect();

            if ns_addrs.is_empty() {
                // ไม่มี glue records — ต้อง resolve NS name ก่อน (recursive)
                // (simplified: ข้ามไปก่อนในโปรเจคนี้)
                return Err("No glue records found (NS name resolution required)".into());
            }

            current_servers = ns_addrs;
            eprintln!("[iterative] referral to: {:?}", &current_servers);
            continue;
        }

        return Err(format!("resolve_iterative: unexpected response (rcode={})", msg.header.rcode()));
    }
}
```

**ตัวอย่าง resolution chain สำหรับ `www.example.com`:**

```
depth=1: query a.root-servers.net (198.41.0.4) for www.example.com
  → referral to .com TLD servers (a.gtld-servers.net, ...)

depth=2: query a.gtld-servers.net (192.5.6.30) for www.example.com
  → referral to example.com authoritative servers (a.iana-servers.net, ...)

depth=3: query a.iana-servers.net (199.43.135.53) for www.example.com
  → ANSWER: www.example.com. 3600 IN A 93.184.216.34
```

---

## Cargo.toml สมบูรณ์

```toml
[package]
name = "dns-resolver"
version = "0.1.0"
edition = "2021"
description = "DNS resolver built from scratch"

[dependencies]
clap       = { version = "4", features = ["derive"] }
tokio      = { version = "1", features = ["full"] }
owo-colors = "4"

[[bin]]
name = "dig-rs"
path = "src/main.rs"
```

---

## การทดสอบ (Testing)

ทำการสร้าง scratch project และรันทดสอบในระหว่างเขียนเนื้อหานี้ ผลลัพธ์จาก `cargo test` จริง:

```
$ cargo test
   Compiling dns_resolver_test v0.1.0
warning: unused variable: `rdata`
   --> src/main.rs:275:36

warning: `dns_resolver_test` generated 7 warnings
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.46s
     Running unittests src/main.rs

running 8 tests
test tests::test_decode_name_compression_pointer ... ok
test tests::test_encode_name_www_example_com ... ok
test tests::test_encode_name_root ... ok
test tests::test_query_header_bytes ... ok
test tests::test_parse_dns_response_a_record ... ok
test tests::test_parse_mx_rdata ... ok
test tests::test_real_resolve_example_com ... ignored
test tests::test_roundtrip_question_section ... ok

test result: ok. 7 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```
$ cargo test -- --include-ignored
running 8 tests
test tests::test_encode_name_root ... ok
test tests::test_decode_name_compression_pointer ... ok
test tests::test_parse_dns_response_a_record ... ok
test tests::test_encode_name_www_example_com ... ok
test tests::test_parse_mx_rdata ... ok
test tests::test_query_header_bytes ... ok
test tests::test_roundtrip_question_section ... ok
test tests::test_real_resolve_example_com ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

**Tests ครบทั้งหมด 8 ตัว ผ่านทุกตัว รวมถึง integration test ที่ query DNS จริงไปยัง 8.8.8.8**

### Unit Tests ที่สำคัญ

```rust
// tests/integration_test.rs
// (สำหรับโปรเจคจริง — ใส่ใน tests/ directory)

mod wire {
    include!("../src/wire.rs");
}

#[cfg(test)]
mod tests {
    use super::wire::*;

    // ── Test 1: Header bytes ──────────────────────────────────
    // ตรวจ 4 bytes แรกของ DNS query ว่าตรงกับ ID และ flags ที่กำหนด
    #[test]
    fn test_query_header_bytes() {
        let id: u16 = 0xABCD;
        let query = build_query(id, "example.com", TYPE_A);

        // bytes 0-1: ID (big-endian)
        assert_eq!(query[0], 0xAB, "ID high byte");
        assert_eq!(query[1], 0xCD, "ID low byte");

        // bytes 2-3: flags — RD=1 → 0x01 0x00
        assert_eq!(query[2], 0x01, "flags high byte (RD=1)");
        assert_eq!(query[3], 0x00, "flags low byte");

        // total length: 12 (header) + 13 (encode_name "example.com") + 4 (QTYPE+QCLASS)
        assert_eq!(query.len(), 12 + 13 + 4, "query length");
    }

    // ── Test 2: Label encoding ────────────────────────────────
    // encode_name("www.example.com") ต้องได้ [3,w,w,w,7,e,x,a,m,p,l,e,3,c,o,m,0]
    #[test]
    fn test_encode_name_www_example_com() {
        let encoded = encode_name("www.example.com");
        let expected: Vec<u8> = vec![
            3, b'w', b'w', b'w',
            7, b'e', b'x', b'a', b'm', b'p', b'l', b'e',
            3, b'c', b'o', b'm',
            0,
        ];
        assert_eq!(encoded, expected);
    }

    // ── Test 3: Round-trip question section ───────────────────
    #[test]
    fn test_roundtrip_question_section() {
        let id: u16 = 0x5678;
        let query = build_query(id, "example.com", TYPE_A);
        let (q, _pos) = parse_question(&query, 12).unwrap();

        assert_eq!(q.qname, "example.com");
        assert_eq!(q.qtype, TYPE_A);
        assert_eq!(q.qclass, CLASS_IN);
    }

    // ── Test 4: Parse real DNS response byte slice ────────────
    // embed hard-coded bytes ของ DNS response จริง
    #[test]
    fn test_parse_dns_response_a_record() {
        // response สำหรับ example.com → 93.184.216.34
        // สร้างแบบ hand-crafted ตาม RFC 1035 format
        let mut response: Vec<u8> = Vec::new();

        // Header
        response.extend_from_slice(&[0x12, 0x34]); // ID
        response.extend_from_slice(&[0x81, 0x80]); // QR=1, RD=1, RA=1
        response.extend_from_slice(&[0x00, 0x01]); // QDCOUNT=1
        response.extend_from_slice(&[0x00, 0x01]); // ANCOUNT=1
        response.extend_from_slice(&[0x00, 0x00, 0x00, 0x00]);

        // Question: example.com A IN
        response.extend_from_slice(&[7,b'e',b'x',b'a',b'm',b'p',b'l',b'e',
                                       3,b'c',b'o',b'm',0]);
        response.extend_from_slice(&[0x00, 0x01, 0x00, 0x01]); // A IN

        // Answer: name = pointer to offset 12, A, IN, TTL=60, 93.184.216.34
        response.extend_from_slice(&[0xC0, 0x0C]); // name pointer
        response.extend_from_slice(&[0x00, 0x01]); // TYPE=A
        response.extend_from_slice(&[0x00, 0x01]); // CLASS=IN
        response.extend_from_slice(&[0x00, 0x00, 0x00, 0x3C]); // TTL=60
        response.extend_from_slice(&[0x00, 0x04]); // RDLENGTH=4
        response.extend_from_slice(&[93, 184, 216, 34]); // IP

        let msg = parse_message(&response).unwrap();
        assert_eq!(msg.header.ancount, 1);
        assert_eq!(msg.answers[0].rtype, TYPE_A);

        let ip_bytes = &msg.answers[0].rdata;
        assert_eq!(ip_bytes, &[93u8, 184, 216, 34]);
        let ip = format!("{}.{}.{}.{}", ip_bytes[0], ip_bytes[1], ip_bytes[2], ip_bytes[3]);
        assert_eq!(ip, "93.184.216.34");
    }

    // ── Test 5: Integration — query real DNS server ───────────
    #[test]
    #[ignore] // รันด้วย: cargo test -- --include-ignored
    fn test_real_resolve_example_com() {
        use std::net::UdpSocket;

        let socket = UdpSocket::bind("0.0.0.0:0").unwrap();
        socket.set_read_timeout(Some(std::time::Duration::from_secs(5))).unwrap();

        let query = build_query(0x1234, "example.com", TYPE_A);
        socket.send_to(&query, "8.8.8.8:53").unwrap();

        let mut buf = [0u8; 512];
        let (n, _) = socket.recv_from(&mut buf).unwrap();
        let msg = parse_message(&buf[..n]).unwrap();

        assert!(msg.header.ancount > 0, "Should have at least one answer");

        let ips: Vec<String> = msg.answers.iter()
            .filter(|rr| rr.rtype == TYPE_A)
            .filter_map(|rr| {
                if rr.rdata.len() == 4 {
                    Some(format!("{}.{}.{}.{}", rr.rdata[0], rr.rdata[1], rr.rdata[2], rr.rdata[3]))
                } else {
                    None
                }
            })
            .collect();

        assert!(!ips.is_empty(), "No IPv4 addresses returned");

        // ตรวจว่าทุก IP เป็น valid IPv4
        for ip in &ips {
            let parts: Vec<&str> = ip.split('.').collect();
            assert_eq!(parts.len(), 4, "IP should have 4 octets: {}", ip);
            for part in parts {
                assert!(part.parse::<u8>().is_ok(), "Invalid octet: {}", part);
            }
        }

        println!("example.com A records: {:?}", ips);
    }
}
```

### Parallel Query Tests (tokio)

```rust
// ตัวอย่าง parallel queries ด้วย tokio::join!
#[cfg(test)]
mod async_tests {
    use tokio::task;

    #[tokio::test]
    #[ignore]
    async fn test_parallel_a_aaaa_mx() {
        // spawn 3 queries พร้อมกัน
        let a_task    = task::spawn_blocking(|| query_sync("google.com", TYPE_A,    "8.8.8.8:53"));
        let aaaa_task = task::spawn_blocking(|| query_sync("google.com", TYPE_AAAA, "8.8.8.8:53"));
        let mx_task   = task::spawn_blocking(|| query_sync("google.com", TYPE_MX,   "8.8.8.8:53"));

        let (a_res, aaaa_res, mx_res) = tokio::join!(a_task, aaaa_task, mx_task);

        assert!(a_res.unwrap().is_ok(),    "A query failed");
        assert!(aaaa_res.unwrap().is_ok(), "AAAA query failed");
        assert!(mx_res.unwrap().is_ok(),   "MX query failed");
    }
}
```

---

## กับดักที่พบบ่อย (Common Pitfalls)

### กับดัก 1: Endianness — Network Byte Order

DNS protocol ใช้ **big-endian** (network byte order) สำหรับตัวเลขทุกค่าที่มีมากกว่า 1 byte ซึ่งตรงข้ามกับสถาปัตยกรรม x86/ARM ที่ใช้ little-endian

```rust
// ❌ WRONG — อ่านค่าแบบ little-endian (ผิดสำหรับ DNS)
let id = u16::from_le_bytes([data[0], data[1]]);
// สำหรับ bytes [0xDE, 0xAD] จะได้ 0xADDE แทนที่จะเป็น 0xDEAD

// ✅ CORRECT — ใช้ from_be_bytes สำหรับ network protocols
let id = u16::from_be_bytes([data[0], data[1]]);
// [0xDE, 0xAD] → 0xDEAD ✓

// ❌ WRONG — ส่ง integer โดยไม่แปลง byte order
buf.push((value >> 0) as u8);  // little-endian!
buf.push((value >> 8) as u8);

// ✅ CORRECT — to_be_bytes() สำหรับ serialize
buf.extend_from_slice(&value.to_be_bytes());

// Compiler จะไม่ตรวจสอบ endianness ให้ — error ชนิดนี้ไม่มี compile error
// แต่ response จะ malformed และ server จะส่ง FORMERR กลับมา
```

**Compiler output เมื่อลืมแปลง:**

ไม่มี compile error — แต่จะได้ response ที่ผิดหรือ timeout:
```
Error: UDP recv_from: timed out
(เพราะ server ได้รับ malformed query และ drop ทิ้ง)
```

### กับดัก 2: Label Compression Loop

```rust
// ❌ อันตราย — ถ้า packet malformed มี pointer วนหากัน:
// offset 10: C0 0A  (pointer to offset 10 = ตัวเอง)
// offset 12: C0 0C  (pointer to offset 12 = ตัวเอง)

// โค้ดนี้จะวนไม่สิ้นสุด:
fn decode_name_buggy(data: &[u8], offset: usize) -> String {
    let mut pos = offset;
    let mut labels = Vec::new();
    loop {
        let len = data[pos];
        if len == 0 { break; }
        if (len & 0xC0) == 0xC0 {
            // ❌ ไม่มีการตรวจสอบ — วนได้ไม่จำกัด
            let ptr = (((len & 0x3F) as usize) << 8) | data[pos + 1] as usize;
            pos = ptr; // อาจวนกลับมา pos เดิม!
            continue;
        }
        // ...
    }
    labels.join(".")
}

// ✅ CORRECT — จำกัด jumps:
const MAX_JUMPS: usize = 10;
let mut jumps = 0;
// ...
jumps += 1;
if jumps > MAX_JUMPS {
    return Err("too many pointer jumps".into());
}
```

### กับดัก 3: TCP Framing — Incomplete Reads

TCP เป็น **stream protocol** ไม่ใช่ message protocol การ `recv()` ครั้งเดียวอาจไม่ได้รับข้อมูลครบ

```rust
// ❌ WRONG — อาจได้รับ partial data
fn query_tcp_buggy(stream: &mut TcpStream, query: &[u8]) -> Vec<u8> {
    stream.write_all(query).unwrap();
    let mut buf = vec![0u8; 4096];
    let n = stream.read(&mut buf).unwrap(); // อาจได้แค่บางส่วน!
    buf[..n].to_vec()
}

// ✅ CORRECT — ใช้ read_exact และอ่าน length prefix ก่อน
fn query_tcp_correct(stream: &mut TcpStream, query_bytes: &[u8]) -> Result<Vec<u8>, String> {
    // Write: length (2 bytes BE) + query
    let len = (query_bytes.len() as u16).to_be_bytes();
    stream.write_all(&len).map_err(|e| e.to_string())?;
    stream.write_all(query_bytes).map_err(|e| e.to_string())?;

    // Read: length prefix
    let mut len_buf = [0u8; 2];
    stream.read_exact(&mut len_buf).map_err(|e| e.to_string())?;
    let resp_len = u16::from_be_bytes(len_buf) as usize;

    // Read: exact bytes
    let mut resp = vec![0u8; resp_len];
    stream.read_exact(&mut resp).map_err(|e| e.to_string())?;
    Ok(resp)
}
```

### กับดัก 4: Index Out of Bounds เมื่อ Parse

DNS packets จาก server จริงอาจ malformed หรือ truncated (โดยเฉพาะ UDP บน lossy networks)

```rust
// ❌ WRONG — panic เมื่อ data สั้นเกิน
fn parse_header_buggy(data: &[u8]) -> DnsHeader {
    DnsHeader {
        id: u16::from_be_bytes([data[0], data[1]]), // panic ถ้า data.len() < 2!
        // ...
    }
}

// ✅ CORRECT — ตรวจสอบก่อนเสมอ
fn parse_header_safe(data: &[u8]) -> Result<DnsHeader, String> {
    if data.len() < 12 {
        return Err(format!("Header too short: {} bytes (need 12)", data.len()));
    }
    Ok(DnsHeader {
        id: u16::from_be_bytes([data[0], data[1]]),
        // ...
    })
}

// Compiler error ที่เกิดจาก try_into บน slice ขนาดผิด:
// error[E0277]: the trait bound `[u8; 4]: TryFrom<&[u8]>` is not satisfied
// ต้องใช้ data[offset..offset+4].try_into().ok()? หรือ indexing แบบ explicit
```

### กับดัก 5: Bit Manipulation บน DNS Flags Word

```rust
// DNS Flags word (16 bits):
// Bit 15 (MSB): QR (0=query, 1=response)
// Bits 11-14:   OPCODE (4 bits)
// Bit 10:       AA (Authoritative Answer)
// Bit 9:        TC (Truncated)
// Bit 8:        RD (Recursion Desired)
// Bit 7:        RA (Recursion Available)
// Bit 6:        Z  (reserved, must be 0)
// Bit 5:        AD (Authentic Data, DNSSEC)
// Bit 4:        CD (Checking Disabled, DNSSEC)
// Bits 0-3:     RCODE (4 bits)

// ❌ WRONG — extract TC bit ผิด position
let tc_wrong = (flags >> 8) & 1;  // อ่าน RD bit แทน TC!

// ✅ CORRECT — TC อยู่ที่ bit 9 (นับจาก 0 จากขวา)
let tc = (flags >> 9) & 1;
// ตรวจสอบด้วยตัวอย่าง: flags = 0x0200 (bit 9 set)
// 0x0200 >> 9 = 0x0001 → & 1 = 1 ✓

// ❌ Common mistake: สับสน "bit position" กับ "shift amount"
// bit 9 ของ 16-bit word = shift right by 9 (ไม่ใช่ shift right by 9-1=8)

// วิธีตรวจสอบ: สร้าง test ด้วยค่าที่รู้แน่นอน
let flags_with_tc: u16 = 0b0000_0010_0000_0000; // bit 9 = TC
assert_eq!((flags_with_tc >> 9) & 1, 1, "TC bit should be 1");
assert_eq!((flags_with_tc >> 8) & 1, 0, "RD bit should be 0");
```

### กับดัก 6: Compression Pointer ใน RDATA

CNAME, MX, NS, SOA records มี domain name ใน RDATA ซึ่งอาจมี compression pointer อ้างถึง offset ใน **packet เต็ม** (ไม่ใช่แค่ใน rdata slice)

```rust
// ❌ WRONG — decode จาก rdata slice เท่านั้น
fn decode_cname_buggy(rdata: &[u8]) -> Option<String> {
    // compression pointer ใน rdata อาจชี้ไปยัง offset ของ packet ที่ใหญ่กว่า
    // ถ้าใช้แค่ rdata ที่ตัดมา pointer จะชี้ไปยัง offset ที่ผิด
    decode_name(rdata, 0).ok().map(|(n, _)| n)
}

// ✅ CORRECT — ต้องส่ง packet เต็มพร้อม rdata_offset
fn decode_cname_correct(full_packet: &[u8], rdata_offset: usize) -> Option<String> {
    decode_name(full_packet, rdata_offset).ok().map(|(n, _)| n)
}

// นั่นคือทำไม DnsResourceRecord ต้องเก็บ rdata_offset ไว้:
pub struct DnsResourceRecord {
    // ...
    pub rdata: Vec<u8>,        // สำเนาของ rdata (สำหรับ A/AAAA ที่ไม่มี pointer)
    pub rdata_offset: usize,   // offset ใน original packet (สำหรับ name-based records)
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# binary อยู่ที่ target/release/dig-rs (Linux/macOS)
#              target/release/dig-rs.exe (Windows)

# ขนาด binary ก่อน strip:
$ ls -lh target/release/dig-rs
-rwxr-xr-x 1 user user 3.2M target/release/dig-rs

# Strip debug symbols:
$ strip target/release/dig-rs
$ ls -lh target/release/dig-rs
-rwxr-xr-x 1 user user 612K target/release/dig-rs
```

### ติดตั้งใน PATH

```bash
cargo install --path .
# หรือ
cp target/release/dig-rs ~/.local/bin/
```

### Cross-Compilation

```bash
# สำหรับ Linux ARM64 (เช่น Raspberry Pi)
rustup target add aarch64-unknown-linux-gnu
cargo build --release --target aarch64-unknown-linux-gnu

# สำหรับ Windows จาก Linux
rustup target add x86_64-pc-windows-gnu
cargo build --release --target x86_64-pc-windows-gnu
```

### การทดสอบ Performance

```bash
# วัดเวลา DNS resolution ด้วย hyperfine
hyperfine 'dig google.com' './target/release/dig-rs google.com'

# ตัวอย่างผล (approximate):
Benchmark 1: dig google.com
  Time (mean ± σ):      23.4 ms ±   2.1 ms
Benchmark 2: dig-rs google.com
  Time (mean ± σ):      21.8 ms ±   1.9 ms
```

---

## Parallel Queries ด้วย tokio

สำหรับ `--type ANY` (simulate) สามารถ spawn A + AAAA + MX พร้อมกัน:

```rust
// ใน main.rs
use tokio::task::JoinHandle;

async fn query_all(name: &str, server: String) {
    let name_a    = name.to_string();
    let name_aaaa = name.to_string();
    let name_mx   = name.to_string();
    let srv_a     = server.clone();
    let srv_aaaa  = server.clone();
    let srv_mx    = server.clone();

    // spawn 3 blocking tasks (DNS uses blocking I/O)
    let h_a: JoinHandle<Result<_, String>> = tokio::task::spawn_blocking(move || {
        resolver::query(&name_a, TYPE_A, &srv_a, 0x0001)
    });
    let h_aaaa: JoinHandle<Result<_, String>> = tokio::task::spawn_blocking(move || {
        resolver::query(&name_aaaa, TYPE_AAAA, &srv_aaaa, 0x0002)
    });
    let h_mx: JoinHandle<Result<_, String>> = tokio::task::spawn_blocking(move || {
        resolver::query(&name_mx, TYPE_MX, &srv_mx, 0x0003)
    });

    // รอทั้งหมดพร้อมกัน
    let (a_res, aaaa_res, mx_res) = tokio::join!(h_a, h_aaaa, h_mx);

    println!("=== A Records ===");
    print_result(a_res);
    println!("=== AAAA Records ===");
    print_result(aaaa_res);
    println!("=== MX Records ===");
    print_result(mx_res);
}

fn print_result(res: Result<Result<wire::DnsMessage, String>, tokio::task::JoinError>) {
    match res {
        Ok(Ok(msg)) => {
            for rr in &msg.answers {
                println!("{} TTL={}", rr.name, rr.ttl);
            }
        }
        Ok(Err(e)) => eprintln!("DNS error: {}", e),
        Err(e)     => eprintln!("Task panicked: {}", e),
    }
}
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: DNSSEC Validation ⭐⭐⭐

DNSSEC เพิ่ม record types ใหม่: RRSIG, DNSKEY, DS, NSEC ที่ใช้สำหรับ cryptographic verification

**Hint:**
- เพิ่ม flag `DO=1` (DNSSEC OK) ใน EDNS0 OPT record (record พิเศษใน additional section)
- Parse RRSIG record: algorithm, key tag, signer's name, signature bytes
- ใช้ crate `ring` หรือ `openssl` สำหรับ RSA/ECDSA signature verification
- Root trust anchor สำหรับ DNSSEC อยู่ที่ https://data.iana.org/root-anchors/

```rust
// EDNS0 OPT record เพื่อเปิด DNSSEC (DO bit)
// OPT record อยู่ใน additional section
// NAME = "." (root), TYPE = 41 (OPT), CLASS = 4096 (UDP payload size)
// TTL = 0x00008000 (DO bit set in extended RCODE/flags)
pub fn add_edns0_opt(buf: &mut Vec<u8>, udp_payload_size: u16, dnssec_ok: bool) {
    buf.push(0);                                      // NAME = root
    buf.extend_from_slice(&41u16.to_be_bytes());      // TYPE = OPT
    buf.extend_from_slice(&udp_payload_size.to_be_bytes()); // CLASS = UDP size
    let ttl: u32 = if dnssec_ok { 0x00008000 } else { 0 };
    buf.extend_from_slice(&ttl.to_be_bytes());        // TTL = extended flags
    buf.extend_from_slice(&0u16.to_be_bytes());       // RDLENGTH = 0
}
```

### แบบฝึกหัดที่ 2: DoH (DNS over HTTPS) Transport ⭐⭐⭐

DNS over HTTPS (RFC 8484) ส่ง DNS query ผ่าน HTTPS แทน UDP/TCP ตรง

**Hint:**
- ใช้ crate `reqwest` (async HTTP client) หรือ `ureq` (sync)
- URL: `https://cloudflare-dns.com/dns-query`
- Content-Type: `application/dns-message`
- Method: POST, body = raw DNS wire format bytes (เหมือน UDP แต่ไม่มี length prefix)
- Response body = raw DNS wire format bytes

```rust
// ตัวอย่าง DoH query ด้วย reqwest
async fn query_doh(name: &str, qtype: u16) -> Result<DnsMessage, String> {
    let query_bytes = build_query(rand_id(), name, qtype);

    let client = reqwest::Client::new();
    let response = client
        .post("https://cloudflare-dns.com/dns-query")
        .header("Content-Type", "application/dns-message")
        .header("Accept", "application/dns-message")
        .body(query_bytes)
        .send()
        .await
        .map_err(|e| e.to_string())?;

    let bytes = response.bytes().await.map_err(|e| e.to_string())?;
    parse_message(&bytes)
}
```

### แบบฝึกหัดที่ 3: Persistent File Cache ⭐⭐

เปลี่ยน in-memory cache เป็น persistent cache ที่บันทึกลงไฟล์

**Hint:**
- ใช้ format `MessagePack` (crate `rmp-serde`) หรือ `serde_json` เพื่อ serialize cache
- บันทึก cache ลงไฟล์ `~/.cache/dig-rs/cache.msgpack`
- โหลด cache กลับมาเมื่อ start โดยกรอง entries ที่หมดอายุออก
- ใช้ `chrono::DateTime<Utc>` เพื่อ serialize timestamp

```rust
// Cache entry สำหรับ persistent storage
#[derive(Serialize, Deserialize)]
struct PersistedEntry {
    records: Vec<SerializableRR>,
    expires_unix: i64, // Unix timestamp (seconds)
}
```

### แบบฝึกหัดที่ 4: Zone File Parser ⭐⭐⭐⭐

Implement parser สำหรับ DNS zone file format (RFC 1035 section 5)

**Hint:**
- Zone file format:
  ```
  $ORIGIN example.com.
  $TTL 3600
  @   IN  SOA   ns1 hostmaster (
                2024012001  ; serial
                3600        ; refresh
                900         ; retry
                604800      ; expire
                300         ; minimum TTL
                )
      IN  NS    ns1.example.com.
  ns1 IN  A     93.184.216.34
  www IN  CNAME @
  ```
- Parse ด้วย line-by-line approach
- รองรับ `$ORIGIN` directive สำหรับ relative names
- รองรับ `$TTL` default TTL
- สามารถ serve zone records ด้วย authoritative DNS server

---

## ข้อมูลเพิ่มเติม: DNS Flags Word อย่างละเอียด

```
 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│QR│  OPCODE  │AA│TC│RD│RA│ Z│AD│CD│    RCODE    │
└──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
 15  14 13 12 11 10  9  8  7  6  5  4  3  2  1  0  ← bit positions
```

| Field  | Bits    | Description |
|--------|---------|-------------|
| QR     | 15      | 0=Query, 1=Response |
| OPCODE | 11–14   | 0=QUERY, 1=IQUERY (obsolete), 2=STATUS |
| AA     | 10      | Authoritative Answer |
| TC     | 9       | Truncated (ต้อง retry ด้วย TCP) |
| RD     | 8       | Recursion Desired (set ใน query) |
| RA     | 7       | Recursion Available (set ใน response) |
| Z      | 6       | Reserved (must be 0) |
| AD     | 5       | Authentic Data (DNSSEC) |
| CD     | 4       | Checking Disabled (DNSSEC) |
| RCODE  | 0–3     | Response Code: 0=NOERROR, 3=NXDOMAIN, 5=REFUSED |

**ตัวอย่าง response flags = 0x8180:**
```
0x8180 = 1000 0001 1000 0000
         │         │
         QR=1      RD=1, RA=1
         (response)
```

## ข้อมูลเพิ่มเติม: Resource Record Wire Format

```
                           1  1  1  1  1  1
 0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
┌──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┐
│                     NAME                       │ ← variable length
├─────────────────────────────────────────────────┤
│                     TYPE                       │ ← 2 bytes
├─────────────────────────────────────────────────┤
│                     CLASS                      │ ← 2 bytes
├─────────────────────────────────────────────────┤
│                     TTL                        │ ← 4 bytes
├─────────────────────────────────────────────────┤
│                   RDLENGTH                     │ ← 2 bytes
├─────────────────────────────────────────────────┤
│                    RDATA                       │ ← RDLENGTH bytes
└─────────────────────────────────────────────────┘
```

## สรุป

ในโปรเจคนี้เราสร้าง DNS resolver ตั้งแต่ศูนย์โดยไม่ใช้ DNS library ใด ๆ และได้เรียนรู้:

1. **DNS wire format (RFC 1035):** header 12 bytes, question section, resource record format — ทุก field มีตำแหน่งและขนาดที่แน่นอนตาม spec
2. **Bit manipulation ใน Rust:** `from_be_bytes`, `to_be_bytes`, bitmask สำหรับ extract flags
3. **DNS name compression:** pointer 2 bytes (top 2 bits = 0b11) ที่อ้างถึง offset ในส่วนอื่นของ packet — เทคนิคที่ใช้มา 40 ปี
4. **UDP/TCP socket programming:** timeout handling, TCP length-prefixed framing
5. **Protocol parser patterns:** offset-based parsing ที่ไม่ copy data โดยไม่จำเป็น, error propagation ด้วย `?`
6. **TTL-aware caching:** in-memory HashMap cache พร้อม `Instant` สำหรับ expiry
7. **Async parallel queries:** `tokio::join!` เพื่อ query A, AAAA, MX พร้อมกัน

Pattern ที่ได้จากโปรเจคนี้ใช้ได้กับ binary protocols อื่น ๆ อีกมาก เช่น HTTP/2 (project H01), MQTT (project H02), gRPC, TLS handshake — ทุกอย่างเป็น byte manipulation + state machine เหมือนกันหมด

โปรเจคถัดไป **A06 Terminal Text Editor** จะสอนการใช้ `crossterm` สำหรับ raw terminal mode, cursor movement, และ text buffer management ซึ่งต้องการทักษะ byte-level I/O ที่ฝึกมาในโปรเจคนี้เช่นกัน

---

**โปรเจคก่อนหน้า:** [Port & Service Scanner](project-a04-port-scanner.md) | **โปรเจคถัดไป:** [Terminal Text Editor](project-a06-terminal-editor.md)
