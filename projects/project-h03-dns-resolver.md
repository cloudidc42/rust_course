# Project H03: DNS Resolver

> โมดูล: H — Networking & Protocols | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 14 ชั่วโมง

## ภาพรวมโปรเจค

DNS (Domain Name System) คือโครงสร้างพื้นฐานที่ขาดไม่ได้ของอินเทอร์เน็ต ทุกครั้งที่พิมพ์ชื่อเว็บไซต์ ระบบปฏิบัติการจะต้องแปลงชื่อนั้นเป็น IP address ผ่านโปรโตคอลที่ถูกกำหนดใน **RFC 1035** ตั้งแต่ปี 1987 และยังคงเป็นหัวใจของอินเทอร์เน็ตจนถึงทุกวันนี้

โปรเจคนี้สร้าง **Stub DNS Resolver** ที่สมบูรณ์แบบตั้งแต่ศูนย์ — implement wire protocol ทุกชั้นด้วยมือ ไม่ใช้ library สำเร็จรูปใด ๆ นอกจาก `tokio` สำหรับ async I/O ระดับความซับซ้อนอยู่ที่การผสมผสานระหว่าง binary protocol parsing, network programming แบบ async, caching พร้อม TTL expiry, และ recursive resolution algorithm

ใน production จริง ซอฟต์แวร์อย่าง **CoreDNS**, **Unbound**, **systemd-resolved**, และ **dnsmasq** ล้วนทำสิ่งที่เราจะ implement ในโปรเจคนี้ (แต่ซับซ้อนและ optimize กว่ามาก) การเขียน resolver เองทำให้เข้าใจว่าทำไม `dig google.com` บางครั้งถึงรอนานกว่าปกติ, ทำไม CNAME records ถึงทำให้เกิด latency, และ cache poisoning attack ทำงานอย่างไร

### Use Cases ใน Production

- **Self-hosted resolver** ภายใน data center เพื่อ split-horizon DNS
- **DNS proxy** ที่กรอง malicious domains (เช่น Pi-hole)
- **Testing tool** สำหรับ verify DNS propagation หลัง deploy
- **Service discovery** สำหรับ microservices ภายใน Kubernetes cluster
- **Security research** เพื่อ analyze DNS traffic และตรวจจับ DNS tunneling

## สิ่งที่จะได้เรียนรู้

- **RFC 1035 wire format:** อ่านและเขียน binary protocol ด้วย byte slice โดยตรง, endianness (big-endian / network byte order)
- **Bit manipulation:** แยก flags ด้วย bitmask และ shift operations, `(flags >> 9) & 1` pattern
- **DNS name compression:** decode pointer 0xC0 prefix ที่ชี้กลับไปยัง offset ก่อนหน้าในข้อความเดียวกัน
- **Async UDP/TCP socket:** `tokio::net::UdpSocket`, `TcpStream`, `tokio::time::timeout`
- **TCP fallback protocol:** DNS over TCP มี 2-byte length prefix — ทำไมถึงต้องมี
- **TTL-aware cache:** `HashMap` + `Instant` สร้าง in-memory cache พร้อม expiry และ negative caching
- **CNAME chain following:** recursive async fn ที่ต้อง `Box::pin` เพราะ Rust ไม่รู้ขนาด future ล่วงหน้า
- **Iterative resolution:** วิธีที่ resolver จริงทำงาน — จาก root → TLD → authoritative server

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50:** async/await, `tokio` runtime, `Future` trait
- **Part 51–55:** `tokio::net::UdpSocket`, `TcpStream`, async I/O
- **Part 21–25:** struct, enum, impl blocks, pattern matching
- **Part 31–35:** error handling ด้วย `Result`, `?` operator, custom error types
- **Part 41–45:** collections — `HashMap`, `Vec`
- **Part 96–100:** byte manipulation, `from_be_bytes`, `to_be_bytes`, bit operations
- **Part 61–65:** `std::time::Instant`, `Duration`, timing
- **Part 36–40:** lifetime basics สำหรับ `Box::pin` recursive async

## โครงสร้างโปรเจค (Project Layout)

```
dns-resolver/
├── src/
│   ├── main.rs          ← CLI entry point
│   ├── wire.rs          ← DNS wire format: structs + serialize/deserialize
│   ├── cache.rs         ← TTL-aware in-memory cache + statistics
│   ├── resolver.rs      ← Stub resolver + CNAME chain following
│   └── iterative.rs     ← Iterative resolution simulation + root hints
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow ของ DNS Query

```
CLI: dns-resolver example.com A @8.8.8.8
          │
          ▼
   Resolver::query("example.com", A)
          │
          ├── DnsCache::get("example.com", A)
          │       └── HIT → return cached records
          │
          └── MISS:
               build_query(id, "example.com", A)  [wire.rs]
               │   encode_name("example.com")
               │   → \x07example\x03com\x00
               │   → 12-byte header + question
               │
               UdpSocket::send(query_bytes) → 8.8.8.8:53
               UdpSocket::recv() → response_bytes
               │
               parse_message(response_bytes)  [wire.rs]
               │   decode_header()
               │   decode_name() × (qdcount + ancount)
               │   parse_rdata() per record
               │
               DnsFlags::from_u16(flags)
               │   TC bit set? → TCP fallback (2-byte length prefix)
               │
               CNAME in answers? → resolve_iterative(target, depth+1)
               │
               DnsCache::insert(name, type, records, min_ttl)
               │
               return Vec<DnsRecord>
```

### การออกแบบ DnsRecord และ RData

สิ่งที่ทำให้ parser ซับซ้อนคือ RDATA ของแต่ละ record type มีรูปแบบต่างกันมาก เราใช้ enum `RData` ที่มี typed variant แต่ละชนิด เพื่อให้ code ที่ใช้ข้อมูลทำงานได้ด้วย pattern matching โดยไม่ต้องใช้ raw bytes:

```
RData::A(Ipv4Addr)                       ← 4 bytes
RData::AAAA(Ipv6Addr)                    ← 16 bytes
RData::CName(String)                     ← encoded name
RData::Mx { preference: u16, exchange }  ← u16 + encoded name
RData::Ns(String)                        ← encoded name
RData::Txt(Vec<String>)                  ← [length_byte + string_bytes]*
RData::Soa { mname, rname, 5×u32 }       ← 2 names + 5 fixed u32
RData::Raw(Vec<u8>)                      ← fallback for unknown types
```

### Cache Design

Cache key คือ `(domain_name_lowercase, record_type_u16)` เก็บใน `HashMap` แต่ละ entry มี `expires_at: Instant` สำหรับ TTL check โดยไม่ต้องใช้ background task (lazy eviction — ตรวจสอบ ณ เวลาที่ lookup)

Negative caching (NXDOMAIN) เป็น feature สำคัญ: เมื่อ domain ไม่มีอยู่จริง เราเก็บ negative entry ไว้เพื่อไม่ต้อง query ซ้ำ ซึ่งช่วยลด latency อย่างมากสำหรับ typo หรือ domain ที่ถูกลบแล้ว

### ทำไมต้องใช้ `Box::pin` สำหรับ Recursive Async

```rust
// ไม่ compile:
async fn resolve(&mut self, name: &str, depth: usize) -> Result<...> {
    // ...
    return self.resolve(target, depth + 1).await; // ← recursive!
}
```

Rust ต้องรู้ขนาดของ `Future` ณ compile time แต่ recursive function สร้าง future ที่มีขนาดไม่จำกัด (F ประกอบด้วย F ซ้ำ ๆ) การใช้ `Box::pin(async move { ... })` แก้ปัญหาโดยย้ายไปอยู่บน heap แทน

```rust
fn resolve<'a>(&'a mut self, name: &'a str, depth: usize)
    -> Pin<Box<dyn Future<Output = Result<Vec<DnsRecord>, Error>> + Send + 'a>>
{
    Box::pin(async move {
        // สามารถ recursive ได้ เพราะ Box ทำให้ขนาด fixed (pointer size)
    })
}
```

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: DNS Header และ Flags — Binary Protocol พื้นฐาน

เริ่มจากสิ่งที่สำคัญที่สุด: เข้าใจว่า DNS header 12 bytes มีโครงสร้างอย่างไร

```
 0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F   (bit position)
┌──────────────────────────────────────────────────┐
│                    ID (16 bits)                   │ bytes 0-1
├──────────────────────────────────────────────────┤
│QR│ OPCODE  │AA│TC│RD│RA│ Z │AD│CD│   RCODE       │ bytes 2-3
├──────────────────────────────────────────────────┤
│                  QDCOUNT (16 bits)                │ bytes 4-5
├──────────────────────────────────────────────────┤
│                  ANCOUNT (16 bits)                │ bytes 6-7
├──────────────────────────────────────────────────┤
│                  NSCOUNT (16 bits)                │ bytes 8-9
├──────────────────────────────────────────────────┤
│                  ARCOUNT (16 bits)                │ bytes 10-11
└──────────────────────────────────────────────────┘
```

**Flags word (bytes 2-3)** แต่ละ bit มีความหมาย:
- **QR** (bit 15): 0 = query, 1 = response
- **OPCODE** (bits 14-11): 0 = QUERY, 1 = IQUERY, 2 = STATUS
- **AA** (bit 10): Authoritative Answer
- **TC** (bit 9): TrunCated — message ถูกตัดทอน ต้อง retry ด้วย TCP
- **RD** (bit 8): Recursion Desired — ขอให้ server หาคำตอบแทนเรา
- **RA** (bit 7): Recursion Available — server รองรับ recursion
- **RCODE** (bits 3-0): 0=NOERROR, 1=FORMERR, 2=SERVFAIL, 3=NXDOMAIN

```rust
// src/wire.rs — ขั้นที่ 1: Header structs

#[derive(Debug, Clone, PartialEq)]
pub struct DnsHeader {
    pub id: u16,
    pub flags: u16,
    pub qdcount: u16,
    pub ancount: u16,
    pub nscount: u16,
    pub arcount: u16,
}

#[derive(Debug, Clone, PartialEq)]
pub struct DnsFlags {
    pub qr: bool,
    pub opcode: u8,
    pub aa: bool,
    pub tc: bool,
    pub rd: bool,
    pub ra: bool,
    pub rcode: u8,
}

impl DnsFlags {
    pub fn from_u16(flags: u16) -> Self {
        DnsFlags {
            qr:     (flags >> 15) & 1 == 1,
            opcode: ((flags >> 11) & 0xF) as u8,
            aa:     (flags >> 10) & 1 == 1,
            tc:     (flags >> 9) & 1 == 1,
            rd:     (flags >> 8) & 1 == 1,
            ra:     (flags >> 7) & 1 == 1,
            rcode:  (flags & 0xF) as u8,
        }
    }

    pub fn to_u16(&self) -> u16 {
        let mut f: u16 = 0;
        if self.qr     { f |= 1 << 15; }
        f |= ((self.opcode as u16) & 0xF) << 11;
        if self.aa     { f |= 1 << 10; }
        if self.tc     { f |= 1 << 9; }
        if self.rd     { f |= 1 << 8; }
        if self.ra     { f |= 1 << 7; }
        f |= (self.rcode as u16) & 0xF;
        f
    }
}

pub fn encode_header(h: &DnsHeader) -> [u8; 12] {
    let mut out = [0u8; 12];
    out[0..2].copy_from_slice(&h.id.to_be_bytes());
    out[2..4].copy_from_slice(&h.flags.to_be_bytes());
    out[4..6].copy_from_slice(&h.qdcount.to_be_bytes());
    out[6..8].copy_from_slice(&h.ancount.to_be_bytes());
    out[8..10].copy_from_slice(&h.nscount.to_be_bytes());
    out[10..12].copy_from_slice(&h.arcount.to_be_bytes());
    out
}

pub fn decode_header(buf: &[u8]) -> Result<DnsHeader, String> {
    if buf.len() < 12 {
        return Err(format!("Buffer too short: {} bytes", buf.len()));
    }
    Ok(DnsHeader {
        id:      u16::from_be_bytes([buf[0], buf[1]]),
        flags:   u16::from_be_bytes([buf[2], buf[3]]),
        qdcount: u16::from_be_bytes([buf[4], buf[5]]),
        ancount: u16::from_be_bytes([buf[6], buf[7]]),
        nscount: u16::from_be_bytes([buf[8], buf[9]]),
        arcount: u16::from_be_bytes([buf[10], buf[11]]),
    })
}
```

**Key point:** `u16::from_be_bytes([buf[0], buf[1]])` แปลง big-endian network bytes เป็น Rust `u16` — ถ้าใช้ `from_le_bytes` จะได้ค่าผิดทั้งหมด

### ขั้นที่ 2: DNS Name Encoding — Wire Format

DNS ใช้ "label encoding" แทนที่จะเก็บ `example.com` เป็น ASCII string ตรง ๆ:

```
"example.com" → \x07 e x a m p l e \x03 c o m \x00
                  │                   │          └── null terminator
                  │                   └── length = 3
                  └── length = 7
```

กฎที่ต้องตรวจสอบตาม RFC 1035:
- แต่ละ label ต้องไม่เกิน **63 bytes** (2 MSB ของ length byte ต้องเป็น 0)
- ชื่อทั้งหมดต้องไม่เกิน **255 bytes** (รวม length bytes และ null terminator)
- Length byte `0xC0` หรือสูงกว่าเป็น **compression pointer** (ไม่ใช่ความยาว)

```rust
// src/wire.rs — ขั้นที่ 2: Name encoding

/// Encode domain name to DNS wire format
pub fn encode_name(name: &str) -> Result<Vec<u8>, String> {
    let mut out = Vec::new();
    let name = name.trim_end_matches('.');  // strip trailing dot
    if name.is_empty() {
        out.push(0u8);
        return Ok(out);
    }
    for label in name.split('.') {
        let bytes = label.as_bytes();
        if bytes.len() > 63 {
            return Err(format!("Label '{}' exceeds 63 bytes", label));
        }
        out.push(bytes.len() as u8);
        out.extend_from_slice(bytes);
    }
    out.push(0u8);  // null terminator
    if out.len() > 255 {
        return Err("Encoded name exceeds 255 bytes".to_string());
    }
    Ok(out)
}

/// Decode domain name with compression pointer support
pub fn decode_name(buf: &[u8], offset: usize) -> Result<(String, usize), String> {
    let mut labels: Vec<String> = Vec::new();
    let mut pos = offset;
    let mut jumped = false;
    let mut end_offset = offset;
    let mut jump_count = 0;

    loop {
        if pos >= buf.len() {
            return Err(format!("out of bounds at {}", pos));
        }
        let len = buf[pos];

        if len == 0 {
            if !jumped { end_offset = pos + 1; }
            break;
        }

        // Compression pointer: top 2 bits = 11
        if len & 0xC0 == 0xC0 {
            if pos + 1 >= buf.len() {
                return Err("pointer truncated".to_string());
            }
            let ptr = (((len & 0x3F) as usize) << 8) | buf[pos + 1] as usize;
            if !jumped { end_offset = pos + 2; }
            jumped = true;
            pos = ptr;
            jump_count += 1;
            if jump_count > 128 {
                return Err("too many pointer jumps (loop?)".to_string());
            }
            continue;
        }

        if len & 0xC0 != 0 {
            return Err(format!("invalid label length: 0x{:02X}", len));
        }

        let label_len = len as usize;
        pos += 1;
        if pos + label_len > buf.len() {
            return Err("label extends beyond buffer".to_string());
        }
        let label = std::str::from_utf8(&buf[pos..pos + label_len])
            .map_err(|e| format!("invalid UTF-8: {}", e))?;
        labels.push(label.to_string());
        pos += label_len;
    }

    Ok((labels.join("."), end_offset))
}
```

**DNS Name Compression** คือเทคนิคประหยัด bandwidth ที่ออกแบบมาตั้งแต่ปี 1987: แทนที่จะเขียน `example.com` ซ้ำในทุก record ข้อความ DNS จะเขียนครั้งแรกแบบ full encoding แล้ว record ต่อ ๆ ไปใช้ pointer 2 bytes ชี้กลับไปยัง offset นั้น

```
Byte 0xC0 หรือ 0b11xxxxxx = compression pointer
Byte 0xC0 0x0C = pointer ไปยัง offset 12 (ต้นของ question section)
```

Decoder ต้องติดตาม `jumped` flag เพื่อไม่ให้ `end_offset` ถูก advance ต่อจาก pointer (เราต้องการ offset หลัง pointer 2 bytes ไม่ใช่หลัง destination)

### ขั้นที่ 3: Record Types และ RDATA Parsing

Record type แต่ละชนิดมี RDATA format เฉพาะตัว:

```rust
// src/wire.rs — ขั้นที่ 3: RecordType enum และ RData

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash,
         serde::Serialize, serde::Deserialize)]
pub enum RecordType {
    A,
    NS,
    CNAME,
    SOA,
    MX,
    TXT,
    AAAA,
    ANY,
    Unknown(u16),
}

impl RecordType {
    pub fn from_u16(v: u16) -> Self {
        match v {
            1   => RecordType::A,
            2   => RecordType::NS,
            5   => RecordType::CNAME,
            6   => RecordType::SOA,
            15  => RecordType::MX,
            16  => RecordType::TXT,
            28  => RecordType::AAAA,
            255 => RecordType::ANY,
            x   => RecordType::Unknown(x),
        }
    }
    pub fn to_u16(self) -> u16 {
        match self {
            RecordType::A          => 1,
            RecordType::NS         => 2,
            RecordType::CNAME      => 5,
            RecordType::SOA        => 6,
            RecordType::MX         => 15,
            RecordType::TXT        => 16,
            RecordType::AAAA       => 28,
            RecordType::ANY        => 255,
            RecordType::Unknown(x) => x,
        }
    }
}

#[derive(Debug, Clone, PartialEq)]
pub enum RData {
    A(std::net::Ipv4Addr),
    AAAA(std::net::Ipv6Addr),
    CName(String),
    Mx { preference: u16, exchange: String },
    Ns(String),
    Txt(Vec<String>),
    Soa {
        mname:   String,
        rname:   String,
        serial:  u32,
        refresh: u32,
        retry:   u32,
        expire:  u32,
        minimum: u32,
    },
    Raw(Vec<u8>),  // fallback สำหรับ type ที่ยังไม่รองรับ
}

#[derive(Debug, Clone, PartialEq)]
pub struct DnsRecord {
    pub name:   String,
    pub rtype:  RecordType,
    pub rclass: u16,
    pub ttl:    u32,
    pub rdata:  RData,
}
```

**Note สำคัญเรื่อง `Unknown(u16)` variant:** เราไม่สามารถใช้ `#[repr(u16)]` กับ enum ที่มี tuple variant ได้ใน Rust — ต้องใช้ method `to_u16()` แทน

การ parse RDATA ทำผ่าน function `parse_rdata` ซึ่งต้องรับทั้ง `rdata_bytes` และ `full_msg` เพราะ CNAME/NS/MX ที่มีชื่อ domain อาจมี compression pointer ชี้กลับไปยัง offset ในข้อความหลัก:

```rust
pub fn parse_rdata(
    rtype: RecordType,
    rdata: &[u8],
    full_msg: &[u8],
) -> Result<RData, String> {
    match rtype {
        RecordType::A => {
            if rdata.len() != 4 {
                return Err(format!("A record rdata wrong length: {}", rdata.len()));
            }
            Ok(RData::A(std::net::Ipv4Addr::new(
                rdata[0], rdata[1], rdata[2], rdata[3]
            )))
        }
        RecordType::AAAA => {
            if rdata.len() != 16 {
                return Err(format!("AAAA wrong length: {}", rdata.len()));
            }
            let mut octets = [0u8; 16];
            octets.copy_from_slice(rdata);
            Ok(RData::AAAA(std::net::Ipv6Addr::from(octets)))
        }
        RecordType::MX => {
            if rdata.len() < 3 {
                return Err("MX rdata too short".to_string());
            }
            let preference = u16::from_be_bytes([rdata[0], rdata[1]]);
            let (exchange, _) = decode_name_from_rdata(&rdata[2..], full_msg)?;
            Ok(RData::Mx { preference, exchange })
        }
        RecordType::TXT => {
            let mut strings = Vec::new();
            let mut i = 0;
            while i < rdata.len() {
                let slen = rdata[i] as usize;
                i += 1;
                if i + slen > rdata.len() {
                    return Err("TXT string truncated".to_string());
                }
                let s = String::from_utf8_lossy(&rdata[i..i+slen]).to_string();
                strings.push(s);
                i += slen;
            }
            Ok(RData::Txt(strings))
        }
        RecordType::SOA => {
            let (mname, off1) = decode_name_from_rdata(rdata, full_msg)?;
            let (rname, off2) = decode_name_from_slice(&rdata[off1..], full_msg)?;
            let off2 = off1 + off2;
            if off2 + 20 > rdata.len() {
                return Err("SOA rdata too short".to_string());
            }
            Ok(RData::Soa {
                mname,
                rname,
                serial:  u32::from_be_bytes([rdata[off2],   rdata[off2+1],
                                             rdata[off2+2], rdata[off2+3]]),
                refresh: u32::from_be_bytes([rdata[off2+4], rdata[off2+5],
                                             rdata[off2+6], rdata[off2+7]]),
                retry:   u32::from_be_bytes([rdata[off2+8], rdata[off2+9],
                                             rdata[off2+10],rdata[off2+11]]),
                expire:  u32::from_be_bytes([rdata[off2+12],rdata[off2+13],
                                             rdata[off2+14],rdata[off2+15]]),
                minimum: u32::from_be_bytes([rdata[off2+16],rdata[off2+17],
                                             rdata[off2+18],rdata[off2+19]]),
            })
        }
        _ => Ok(RData::Raw(rdata.to_vec())),
    }
}
```

**TXT record** มีรูปแบบพิเศษ: RDATA ประกอบด้วย string หลาย ๆ ตัวต่อกัน แต่ละตัวนำหน้าด้วย length byte (ไม่ใช่ null-terminated) — SPF records มักมีเพียง 1 string แต่มาตรฐานอนุญาตให้มีหลาย string ต่อกัน

**SOA record** ประกอบด้วย 2 domain names (mname, rname) ตามด้วย 5 u32 values (serial, refresh, retry, expire, minimum) — zone `rname` คือ email ของ admin ที่เขียน `.` แทน `@` เช่น `admin.example.com` = `admin@example.com`

### ขั้นที่ 4: Build Query และ Parse Message — Full Wire Format

```rust
// src/wire.rs — ขั้นที่ 4: ประกอบและแยกข้อความทั้งหมด

pub struct DnsQuestion {
    pub qname:  String,
    pub qtype:  RecordType,
    pub qclass: u16,  // 1 = IN (Internet)
}

pub struct DnsMessage {
    pub header:     DnsHeader,
    pub questions:  Vec<DnsQuestion>,
    pub answers:    Vec<DnsRecord>,
    pub authority:  Vec<DnsRecord>,
    pub additional: Vec<DnsRecord>,
}

/// สร้าง DNS query message (wire bytes)
pub fn build_query(id: u16, name: &str, qtype: RecordType) -> Result<Vec<u8>, String> {
    let mut buf = Vec::with_capacity(512);

    // Header: ID + flags (RD=1) + counts
    buf.extend_from_slice(&id.to_be_bytes());
    let flags: u16 = 1 << 8;  // RD = 1, ทุก bit อื่นเป็น 0
    buf.extend_from_slice(&flags.to_be_bytes());
    buf.extend_from_slice(&1u16.to_be_bytes()); // qdcount = 1
    buf.extend_from_slice(&0u16.to_be_bytes()); // ancount
    buf.extend_from_slice(&0u16.to_be_bytes()); // nscount
    buf.extend_from_slice(&0u16.to_be_bytes()); // arcount

    // Question: QNAME + QTYPE + QCLASS
    buf.extend_from_slice(&encode_name(name)?);
    buf.extend_from_slice(&qtype.to_u16().to_be_bytes());
    buf.extend_from_slice(&1u16.to_be_bytes()); // QCLASS = IN

    Ok(buf)
}

/// Parse response bytes เป็น DnsMessage
pub fn parse_message(buf: &[u8]) -> Result<DnsMessage, String> {
    if buf.len() < 12 {
        return Err("DNS message too short".to_string());
    }
    let header = DnsHeader {
        id:      u16::from_be_bytes([buf[0], buf[1]]),
        flags:   u16::from_be_bytes([buf[2], buf[3]]),
        qdcount: u16::from_be_bytes([buf[4], buf[5]]),
        ancount: u16::from_be_bytes([buf[6], buf[7]]),
        nscount: u16::from_be_bytes([buf[8], buf[9]]),
        arcount: u16::from_be_bytes([buf[10], buf[11]]),
    };

    let mut pos = 12usize;

    // Parse questions
    let mut questions = Vec::new();
    for _ in 0..header.qdcount {
        let (qname, new_pos) = decode_name(buf, pos)?;
        pos = new_pos;
        if pos + 4 > buf.len() { return Err("question truncated".to_string()); }
        let qtype  = RecordType::from_u16(u16::from_be_bytes([buf[pos], buf[pos+1]]));
        let qclass = u16::from_be_bytes([buf[pos+2], buf[pos+3]]);
        pos += 4;
        questions.push(DnsQuestion { qname, qtype, qclass });
    }

    // Parse 3 sections: answers, authority, additional
    let mut answers    = Vec::new();
    let mut authority  = Vec::new();
    let mut additional = Vec::new();
    let section_counts = [
        (header.ancount, &mut answers),
        (header.nscount, &mut authority),
        (header.arcount, &mut additional),
    ];

    for (count, section) in section_counts {
        for _ in 0..count {
            let (name, new_pos) = decode_name(buf, pos)?;
            pos = new_pos;
            if pos + 10 > buf.len() { return Err("record truncated".to_string()); }
            let rtype  = RecordType::from_u16(u16::from_be_bytes([buf[pos],   buf[pos+1]]));
            let rclass = u16::from_be_bytes([buf[pos+2], buf[pos+3]]);
            let ttl    = u32::from_be_bytes([buf[pos+4], buf[pos+5],
                                             buf[pos+6], buf[pos+7]]);
            let rdlen  = u16::from_be_bytes([buf[pos+8], buf[pos+9]]) as usize;
            pos += 10;
            if pos + rdlen > buf.len() { return Err("rdata truncated".to_string()); }
            let rdata = parse_rdata(rtype, &buf[pos..pos + rdlen], buf)?;
            pos += rdlen;
            section.push(DnsRecord { name, rtype, rclass, ttl, rdata });
        }
    }

    Ok(DnsMessage { header, questions, answers, authority, additional })
}
```

Resource record format ใน wire:
```
┌─────────────────────────────────────┐
│  NAME (variable — encoded name)      │
├────────────┬────────────────────────┤
│ TYPE (2)   │ CLASS (2)              │
├────────────┴────────────────────────┤
│  TTL (4 bytes, unsigned)             │
├────────────┬────────────────────────┤
│ RDLENGTH(2)│ RDATA (RDLENGTH bytes) │
└────────────┴────────────────────────┘
```

### ขั้นที่ 5: DNS Cache พร้อม TTL Expiry

```rust
// src/cache.rs — ขั้นที่ 5: TTL-aware cache

use std::collections::HashMap;
use std::time::{Duration, Instant};
use crate::wire::{DnsRecord, RecordType};

#[derive(Debug, Clone)]
pub struct CacheEntry {
    pub records:    Vec<DnsRecord>,
    pub expires_at: Instant,
    pub negative:   bool,  // true = NXDOMAIN
}

impl CacheEntry {
    pub fn is_expired(&self) -> bool {
        Instant::now() > self.expires_at
    }
}

type CacheKey = (String, u16);  // (domain_lowercase, record_type)

#[derive(Debug, Default, Clone)]
pub struct CacheStats {
    pub hits:      u64,
    pub misses:    u64,
    pub inserts:   u64,
    pub evictions: u64,
}

impl CacheStats {
    pub fn hit_rate(&self) -> f64 {
        let total = self.hits + self.misses;
        if total == 0 { 0.0 } else { self.hits as f64 / total as f64 }
    }
}

pub struct DnsCache {
    entries: HashMap<CacheKey, CacheEntry>,
    pub stats: CacheStats,
}

impl DnsCache {
    pub fn new() -> Self {
        DnsCache { entries: HashMap::new(), stats: CacheStats::default() }
    }

    /// เก็บ records พร้อม TTL (positive cache)
    pub fn insert(&mut self, name: &str, rtype: RecordType,
                  records: Vec<DnsRecord>, ttl_secs: u32) {
        let key = (name.to_lowercase(), rtype.to_u16());
        self.entries.insert(key, CacheEntry {
            records,
            expires_at: Instant::now() + Duration::from_secs(ttl_secs as u64),
            negative: false,
        });
        self.stats.inserts += 1;
    }

    /// เก็บ negative cache entry (NXDOMAIN)
    pub fn insert_negative(&mut self, name: &str, rtype: RecordType, ttl_secs: u32) {
        let key = (name.to_lowercase(), rtype.to_u16());
        self.entries.insert(key, CacheEntry {
            records: vec![],
            expires_at: Instant::now() + Duration::from_secs(ttl_secs as u64),
            negative: true,
        });
        self.stats.inserts += 1;
    }

    /// Lookup — คืน None ถ้า miss หรือ expired (lazy eviction)
    pub fn get(&mut self, name: &str, rtype: RecordType) -> Option<&CacheEntry> {
        let key = (name.to_lowercase(), rtype.to_u16());
        if let Some(entry) = self.entries.get(&key) {
            if entry.is_expired() {
                self.entries.remove(&key);
                self.stats.evictions += 1;
                self.stats.misses += 1;
                return None;
            }
            self.stats.hits += 1;
            return self.entries.get(&(name.to_lowercase(), rtype.to_u16()));
        }
        self.stats.misses += 1;
        None
    }

    /// ลบ entries ที่หมดอายุทั้งหมด (periodic cleanup)
    pub fn evict_expired(&mut self) {
        let expired: Vec<CacheKey> = self.entries.iter()
            .filter(|(_, v)| v.is_expired())
            .map(|(k, _)| k.clone())
            .collect();
        self.stats.evictions += expired.len() as u64;
        for k in expired { self.entries.remove(&k); }
    }
}
```

**Lazy eviction** คือ strategy ที่ตรวจสอบ expiry ณ เวลาที่ `get()` ถูกเรียก แทนที่จะมี background task คอย scan — เหมาะกับ use case ที่ access pattern ไม่ dense มากนัก ข้อดีคือ simple, ไม่ต้องใช้ `Arc<Mutex<>>` หรือ background thread

ถ้า cache มีขนาดใหญ่และ entries หมดอายุแต่ไม่ถูก access อีก ให้เรียก `evict_expired()` เป็นระยะ (เช่นทุก 60 วินาที)

### ขั้นที่ 6: Stub Resolver — UDP Query พร้อม TCP Fallback

```rust
// src/resolver.rs — ขั้นที่ 6: Stub resolver

use tokio::net::UdpSocket;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use std::time::Duration;

/// ส่ง DNS query ผ่าน UDP, fallback TCP เมื่อ TC bit ถูก set
pub async fn query_server(
    server: &str,
    name: &str,
    qtype: RecordType,
    timeout_ms: u64,
) -> Result<DnsMessage, ResolverError> {
    let id: u16 = rand_id();
    let query_bytes = build_query(id, name, qtype)
        .map_err(|e| ResolverError::Parse(e))?;

    let server_addr = format!("{}:53", server);

    // 1. ลองส่ง UDP ก่อน
    let response = query_udp(&query_bytes, &server_addr, timeout_ms).await?;
    let msg = parse_message(&response).map_err(|e| ResolverError::Parse(e))?;

    let flags = DnsFlags::from_u16(msg.header.flags);
    if flags.tc {
        // TC=1 หมายความว่า response ถูกตัดให้เหลือ 512 bytes — retry ด้วย TCP
        let response_tcp = query_tcp(&query_bytes, &server_addr, timeout_ms).await?;
        return parse_message(&response_tcp).map_err(|e| ResolverError::Parse(e));
    }

    Ok(msg)
}

async fn query_udp(query: &[u8], server: &str, timeout_ms: u64)
    -> Result<Vec<u8>, ResolverError>
{
    let sock = UdpSocket::bind("0.0.0.0:0").await?;  // ephemeral port
    sock.connect(server).await?;
    sock.send(query).await?;

    let mut buf = vec![0u8; 4096];
    let result = tokio::time::timeout(
        Duration::from_millis(timeout_ms),
        sock.recv(&mut buf),
    ).await;

    match result {
        Ok(Ok(n))  => Ok(buf[..n].to_vec()),
        Ok(Err(e)) => Err(ResolverError::Io(e)),
        Err(_)     => Err(ResolverError::Timeout),
    }
}

async fn query_tcp(query: &[u8], server: &str, timeout_ms: u64)
    -> Result<Vec<u8>, ResolverError>
{
    let mut stream = tokio::time::timeout(
        Duration::from_millis(timeout_ms),
        TcpStream::connect(server),
    ).await
        .map_err(|_| ResolverError::Timeout)?
        .map_err(ResolverError::Io)?;

    // DNS over TCP: นำหน้าด้วย 2 bytes บอกขนาด query
    let len = query.len() as u16;
    stream.write_all(&len.to_be_bytes()).await?;
    stream.write_all(query).await?;

    // อ่าน response: 2 bytes ความยาว + response bytes
    let mut len_buf = [0u8; 2];
    stream.read_exact(&mut len_buf).await?;
    let resp_len = u16::from_be_bytes(len_buf) as usize;

    let mut resp = vec![0u8; resp_len];
    stream.read_exact(&mut resp).await?;
    Ok(resp)
}
```

**ทำไม DNS over TCP ต้องมี 2-byte length prefix?**

UDP คือ datagram protocol — แต่ละ packet ชัดเจนว่าจบที่ไหน แต่ TCP คือ stream protocol — ข้อมูลไหลต่อเนื่องไม่มีขอบเขต receiver ต้องรู้ว่า DNS message แต่ละอันมีกี่ bytes ก่อนถึงจะ parse ได้ถูกต้อง

ใน DNS protocol ดั้งเดิม (RFC 1035) UDP มีขีดจำกัด 512 bytes เมื่อ response ใหญ่กว่านั้น server จะ set TC=1 และส่ง 512 bytes แรกกลับมา client ต้องเปิด TCP connection ใหม่ แล้ว query อีกครั้งเพื่อรับ full response (EDNS0 ใน RFC 2671 ขยาย limit เป็นสูงสุด 65535 bytes ใน UDP แต่ยังต้องรองรับ TC fallback)

### ขั้นที่ 7: High-Level Resolver พร้อม CNAME Chain Following

```rust
// src/resolver.rs — ขั้นที่ 7: CNAME chain + cache integration

pub struct Resolver {
    pub cache:      DnsCache,
    pub servers:    Vec<String>,
    pub timeout_ms: u64,
}

impl Resolver {
    pub fn new(servers: Vec<String>) -> Self {
        Resolver { cache: DnsCache::new(), servers, timeout_ms: 3000 }
    }

    pub async fn query(&mut self, name: &str, qtype: RecordType)
        -> Result<Vec<DnsRecord>, ResolverError>
    {
        self.resolve_iterative(name, qtype, 0).await
    }

    // Box::pin จำเป็นสำหรับ recursive async fn
    fn resolve_iterative<'a>(
        &'a mut self,
        name: &'a str,
        qtype: RecordType,
        depth: usize,
    ) -> std::pin::Pin<Box<dyn std::future::Future<
        Output = Result<Vec<DnsRecord>, ResolverError>
    > + Send + 'a>> {
        Box::pin(async move {
            if depth > 8 {
                return Err(ResolverError::CnameLoop);
            }

            // ตรวจ cache ก่อน
            if let Some(entry) = self.cache.get(name, qtype) {
                if entry.negative {
                    return Err(ResolverError::NxDomain);
                }
                return Ok(entry.records.clone());
            }

            // Query แต่ละ server จนกว่าจะสำเร็จ
            let mut last_err = ResolverError::Timeout;
            for server in self.servers.clone() {
                match query_server(&server, name, qtype, self.timeout_ms).await {
                    Ok(msg) => {
                        let flags = DnsFlags::from_u16(msg.header.flags);
                        if flags.rcode == 3 {
                            self.cache.insert_negative(name, qtype, 60);
                            return Err(ResolverError::NxDomain);
                        }
                        if flags.rcode != 0 {
                            return Err(ResolverError::ServerError(flags.rcode));
                        }

                        let answers = msg.answers.clone();
                        if answers.is_empty() { continue; }

                        // ตรวจหา CNAME ใน answers
                        let cname_target = answers.iter().find_map(|r| {
                            if r.rtype == RecordType::CNAME {
                                if let RData::CName(ref t) = r.rdata {
                                    return Some(t.clone());
                                }
                            }
                            None
                        });

                        if let Some(target) = cname_target {
                            if qtype != RecordType::CNAME {
                                // ติดตาม CNAME chain แบบ recursive
                                let t = target.clone();
                                return self.resolve_iterative(&t, qtype, depth + 1).await;
                            }
                        }

                        let min_ttl = answers.iter().map(|r| r.ttl).min().unwrap_or(60);
                        self.cache.insert(name, qtype, answers.clone(), min_ttl);
                        return Ok(answers);
                    }
                    Err(e) => { last_err = e; continue; }
                }
            }
            Err(last_err)
        })
    }
}
```

**CNAME chain** เป็น feature ที่ทำให้ DNS ยืดหยุ่น เช่น `www.example.com` CNAME → `lb.example.com` CNAME → `203.0.113.5` เมื่อ query A record สำหรับ `www.example.com` resolver ต้องติดตาม CNAME chain จนถึง A record จริง โดยมี limit ที่ 8 hop เพื่อป้องกัน infinite loop

### ขั้นที่ 8: Iterative Resolution Simulation

Resolver จริงในโลกทำงานแบบ **iterative** — ไม่ใช่ recursive ไปยัง server เดียว แต่เริ่มจาก root server แล้วติดตาม delegation chain ทีละขั้น:

1. Query root (`.`) → ได้รับ NS records ของ `.com` TLD
2. Query TLD nameserver สำหรับ `example.com` → ได้รับ NS records ของ `example.com`
3. Query authoritative nameserver → ได้รับ A record สุดท้าย

```rust
// src/iterative.rs — ขั้นที่ 8: Iterative resolution

/// Root hints — IP addresses ของ root nameservers จริง
pub const ROOT_HINTS: &[&str] = &[
    "198.41.0.4",     // a.root-servers.net
    "170.247.170.2",  // b.root-servers.net
    "192.33.4.12",    // c.root-servers.net
    "199.7.91.13",    // d.root-servers.net
];

#[derive(Debug, Clone)]
pub struct DelegationStep {
    pub zone:       String,
    pub nameserver: String,
}

/// คำนวณ resolution path จาก root → full name
pub fn resolution_path(name: &str) -> Vec<String> {
    let name = name.trim_end_matches('.');
    let labels: Vec<&str> = name.split('.').collect();
    let n = labels.len();
    let mut path = Vec::new();
    for i in 0..=n {
        if i == 0 {
            path.push(".".to_string());
        } else {
            path.push(labels[n - i..].join("."));
        }
    }
    path
}

/// วางแผนลำดับ NS server ที่ต้อง query (simulation)
pub fn plan_iterative_resolution(name: &str) -> Vec<DelegationStep> {
    let path = resolution_path(name);
    let mut steps = Vec::new();
    for (i, zone) in path.iter().enumerate() {
        if zone == "." {
            steps.push(DelegationStep {
                zone: ".".to_string(),
                nameserver: ROOT_HINTS[0].to_string(),
            });
        } else if i < path.len() - 1 {
            steps.push(DelegationStep {
                zone: zone.clone(),
                nameserver: format!("ns.{}", zone),
            });
        }
    }
    steps
}
```

ตัวอย่าง resolution path สำหรับ `www.example.com`:
```
. → com → example.com → www.example.com
```
แต่ละขั้น: query NS server ของ zone ก่อนหน้า เพื่อหา NS server ของ zone ปัจจุบัน

### ขั้นที่ 9: CLI Entry Point

```rust
// src/main.rs

mod wire;
mod cache;
mod resolver;
mod iterative;

use resolver::Resolver;
use wire::RecordType;

#[tokio::main]
async fn main() {
    let args: Vec<String> = std::env::args().collect();
    if args.len() < 2 {
        eprintln!("Usage: dns-resolver <name> [type] [@server]");
        std::process::exit(1);
    }

    let name = &args[1];
    let qtype = args.get(2)
        .map(|t| match t.to_uppercase().as_str() {
            "A"     => RecordType::A,
            "AAAA"  => RecordType::AAAA,
            "MX"    => RecordType::MX,
            "TXT"   => RecordType::TXT,
            "CNAME" => RecordType::CNAME,
            "NS"    => RecordType::NS,
            _       => RecordType::A,
        })
        .unwrap_or(RecordType::A);

    let server = args.iter()
        .find(|a| a.starts_with('@'))
        .map(|a| a.trim_start_matches('@').to_string())
        .unwrap_or_else(|| "8.8.8.8".to_string());

    let mut r = Resolver::new(vec![server.clone()]);
    println!("Querying {} {:?} @{}", name, qtype, server);

    match r.query(name, qtype).await {
        Ok(records) => {
            println!("Got {} answer(s):", records.len());
            for rec in &records {
                println!("  {} TTL={} {:?}", rec.name, rec.ttl, rec.rdata);
            }
            println!("\nCache stats: {:?}", r.cache.stats);
        }
        Err(e) => {
            eprintln!("Error: {}", e);
            std::process::exit(1);
        }
    }
}
```

### ขั้นที่ 10: Cargo.toml

```toml
[package]
name = "dns-resolver"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "dns-resolver"
path = "src/main.rs"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

## การทดสอบ (Testing)

โปรเจคมี **34 unit tests** ที่ครอบคลุมทุก component หลัก:

### รายการ Tests

**wire.rs — DNS wire format:**
- `test_header_encode_decode_roundtrip` — encode header แล้ว decode กลับต้องได้ค่าเดิม
- `test_header_bytes_are_big_endian` — ตรวจสอบ byte order ใน encoded output
- `test_header_decode_too_short` — buffer น้อยกว่า 12 bytes ต้อง error
- `test_flags_decode_standard_response` — decode 0x8180 ต้องได้ QR=1, RD=1, RA=1
- `test_flags_truncated_bit` — ตรวจสอบ TC flag extraction
- `test_flags_roundtrip` — DnsFlags → u16 → DnsFlags ต้องได้ค่าเดิม
- `test_flags_nxdomain` — RCODE=3 ต้องถูก decode ถูกต้อง
- `test_encode_name_simple` — "example.com" → `\x07example\x03com\x00`
- `test_encode_name_single_label` — "localhost" → `\x09localhost\x00`
- `test_encode_name_trailing_dot` — "example.com." ≡ "example.com"
- `test_encode_name_label_too_long` — label 64 bytes ต้อง error
- `test_decode_name_simple` — decode wire bytes → "example.com"
- `test_decode_name_with_compression_pointer` — pointer 0xC0 0x00 ต้อง follow ได้
- `test_decode_name_root` — `\x00` → ""
- `test_parse_a_record` — 4 bytes → Ipv4Addr ถูกต้อง
- `test_parse_aaaa_record` — 16 bytes → Ipv6Addr
- `test_parse_mx_record` — preference u16 + encoded name
- `test_parse_txt_record` — length-prefixed string
- `test_parse_a_wrong_length` — 3 bytes ต้อง error
- `test_build_query_structure` — ตรวจ ID, flags, qdcount ใน output bytes

**cache.rs — DNS cache:**
- `test_cache_insert_and_hit` — insert แล้ว get ต้องได้ records
- `test_cache_miss_returns_none` — get ที่ไม่มีใน cache ต้อง None
- `test_cache_ttl_expiry` — entry ที่ expires_at อยู่ในอดีตต้อง return None
- `test_negative_cache_insert_and_get` — NXDOMAIN entry ต้องมี `negative=true`
- `test_cache_stats_hit_rate` — 1 hit + 1 miss = hit_rate 0.5
- `test_cache_case_insensitive_lookup` — "Example.COM" และ "example.com" เป็น key เดียวกัน
- `test_evict_expired_cleans_up` — expired entry ต้องถูกลบออก

**resolver.rs — Stub resolver:**
- `test_is_truncated_true` — flags ที่มี TC=1 ต้อง return true
- `test_is_truncated_false` — flags ที่ไม่มี TC ต้อง return false
- `test_is_nxdomain` — RCODE=3 ต้อง return true
- `test_cname_chain_detection` — CNAME rdata ต้อง extract target ถูกต้อง

**iterative.rs — Iterative resolver:**
- `test_resolution_path_three_labels` — www.example.com → [., com, example.com, www.example.com]
- `test_resolution_path_two_labels` — example.com → 3 steps
- `test_plan_iterative_starts_with_root` — ขั้นแรกต้องเป็น root server

### Real `cargo test` Output

```
running 34 tests
test cache::tests::test_cache_case_insensitive_lookup ... ok
test cache::tests::test_cache_insert_and_hit ... ok
test cache::tests::test_cache_stats_hit_rate ... ok
test cache::tests::test_cache_miss_returns_none ... ok
test cache::tests::test_cache_ttl_expiry ... ok
test cache::tests::test_evict_expired_cleans_up ... ok
test cache::tests::test_negative_cache_insert_and_get ... ok
test iterative::tests::test_plan_iterative_starts_with_root ... ok
test iterative::tests::test_resolution_path_three_labels ... ok
test resolver::tests::test_is_nxdomain ... ok
test resolver::tests::test_cname_chain_detection ... ok
test iterative::tests::test_resolution_path_two_labels ... ok
test resolver::tests::test_is_truncated_true ... ok
test wire::tests::test_build_query_structure ... ok
test resolver::tests::test_is_truncated_false ... ok
test wire::tests::test_decode_name_root ... ok
test wire::tests::test_decode_name_simple ... ok
test wire::tests::test_decode_name_with_compression_pointer ... ok
test wire::tests::test_encode_name_label_too_long ... ok
test wire::tests::test_encode_name_simple ... ok
test wire::tests::test_encode_name_single_label ... ok
test wire::tests::test_flags_roundtrip ... ok
test wire::tests::test_encode_name_trailing_dot ... ok
test wire::tests::test_flags_decode_standard_response ... ok
test wire::tests::test_flags_nxdomain ... ok
test wire::tests::test_flags_truncated_bit ... ok
test wire::tests::test_header_decode_too_short ... ok
test wire::tests::test_parse_a_record ... ok
test wire::tests::test_header_bytes_are_big_endian ... ok
test wire::tests::test_header_encode_decode_roundtrip ... ok
test wire::tests::test_parse_a_wrong_length ... ok
test wire::tests::test_parse_aaaa_record ... ok
test wire::tests::test_parse_mx_record ... ok
test wire::tests::test_parse_txt_record ... ok

test result: ok. 34 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### ทดสอบ CNAME Chain ด้วย Mock Data

สามารถทดสอบ CNAME chain ได้โดยสร้าง mock response bytes และ parse ด้วย `parse_message()`:

```rust
#[test]
fn test_cname_chain_parse() {
    // สร้าง response ที่มี CNAME record ใน answer section
    // header: QR=1, ancount=1
    let mut buf: Vec<u8> = vec![
        0x00, 0x01,  // ID
        0x81, 0x80,  // QR=1, RD=1, RA=1
        0x00, 0x01,  // qdcount=1
        0x00, 0x01,  // ancount=1
        0x00, 0x00,  // nscount=0
        0x00, 0x00,  // arcount=0
    ];
    // Question: www.example.com A IN
    buf.extend_from_slice(b"\x03www\x07example\x03com\x00");
    buf.extend_from_slice(&[0x00, 0x01, 0x00, 0x01]); // TYPE=A, CLASS=IN

    // Answer: www.example.com CNAME example.com TTL=300
    buf.extend_from_slice(b"\x03www\x07example\x03com\x00");
    buf.extend_from_slice(&[0x00, 0x05]);  // TYPE=CNAME
    buf.extend_from_slice(&[0x00, 0x01]);  // CLASS=IN
    buf.extend_from_slice(&[0x00, 0x00, 0x01, 0x2C]); // TTL=300
    let cname_rdata = b"\x07example\x03com\x00";
    buf.extend_from_slice(&(cname_rdata.len() as u16).to_be_bytes());
    buf.extend_from_slice(cname_rdata);

    let msg = parse_message(&buf).unwrap();
    assert_eq!(msg.header.ancount, 1);
    if let RData::CName(ref target) = msg.answers[0].rdata {
        assert_eq!(target, "example.com");
    }
}
```

### Integration Test สำหรับ Cache TTL

```rust
#[tokio::test]
async fn test_cache_negative_ttl_flow() {
    let mut cache = DnsCache::new();
    // Insert negative entry ที่จะหมดอายุทันที
    cache.insert_negative("gone.example", RecordType::A, 0);
    // Force expiry
    use std::time::{Duration, Instant};
    // ใน unit test เราจำลองโดย manipulate entry directly
    let key = ("gone.example".to_string(), RecordType::A.to_u16());
    if let Some(entry) = cache.entries.get_mut(&key) {
        entry.expires_at = Instant::now() - Duration::from_millis(1);
    }
    // ต้อง return None (expired)
    assert!(cache.get("gone.example", RecordType::A).is_none());
}
```

## Pitfalls และข้อผิดพลาดที่พบบ่อย

### Pitfall 1: Little-endian vs Big-endian

**ปัญหา:** DNS protocol ใช้ **network byte order (big-endian)** แต่ x86/ARM ส่วนใหญ่เป็น little-endian การ cast ตรง ๆ จะได้ค่าผิด

```rust
// ผิด! — ทำงานผิดบน little-endian systems
let id = u16::from_le_bytes([buf[0], buf[1]]); // 0x1234 → 0x3412

// ถูกต้อง
let id = u16::from_be_bytes([buf[0], buf[1]]); // 0x1234 → 0x1234
```

กฎทั่วไป: ทุก `u16`/`u32` ใน DNS wire format ต้องใช้ `from_be_bytes()` และ `to_be_bytes()` เสมอ ไม่มีข้อยกเว้น

### Pitfall 2: Name Compression Pointer อาจชี้ไปข้างหน้า (Forward Pointer)

RFC 1035 ไม่ได้ห้าม compression pointer ที่ชี้ไปยัง offset ที่ยังไม่ผ่านมา แต่ในทางปฏิบัติ pointer ที่ดีจะชี้กลับไปเท่านั้น (backward) เพราะชี้ไปข้างหน้าจะทำให้ decode ลำบาก

**ปัญหาลึกกว่า:** pointer loop — `A → B → A` ทำให้ infinite loop

```rust
// ต้องมี jump limit เสมอ
let mut jump_count = 0;
if len & 0xC0 == 0xC0 {
    jump_count += 1;
    if jump_count > 128 {
        return Err("too many pointer jumps (loop?)".to_string());
    }
    // ...
}
```

DNS server ที่ malicious หรือมี bug สามารถส่ง poison message ที่มี circular pointer มาได้ การมี limit ป้องกัน DoS

### Pitfall 3: end_offset ของ decode_name เมื่อมี Compression

เมื่อ `decode_name` ติดตาม pointer ไปยัง offset อื่น ค่า `end_offset` ที่ควร return คือ **หลัง 2-byte pointer** ไม่ใช่หลัง destination ของ pointer เพราะ parser ต้องเดินหน้าต่อจาก pointer ไม่ใช่จาก destination

```rust
if len & 0xC0 == 0xC0 {
    if !jumped {
        end_offset = pos + 2;  // <-- บันทึกก่อน jump เสมอ
    }
    jumped = true;
    pos = ptr;  // jump ไป destination
    // ไม่อัปเดต end_offset อีกแล้ว!
}
```

ถ้า return `pos + 1` (ตำแหน่งหลัง null terminator ที่ destination) parser จะข้ามข้อมูลใน message หลังจาก pointer 2 bytes ไป

### Pitfall 4: Recursive Async ใน Rust ต้องใช้ `Box::pin`

Rust ต้องรู้ขนาดของทุก type ณ compile time รวมถึง `Future` แต่ recursive async function สร้าง type ที่มีขนาดไม่สิ้นสุด:

```
type F = async fn() -> R containing F
       = ... F ... F ... F ...  (infinite)
```

**วิธีแก้:**

```rust
// ต้องเขียนแบบนี้ — ไม่ใช่ async fn ธรรมดา
fn my_async_fn<'a>(&'a mut self, depth: usize)
    -> Pin<Box<dyn Future<Output = Result<(), Error>> + Send + 'a>>
{
    Box::pin(async move {
        if depth < 5 {
            self.my_async_fn(depth + 1).await?;
        }
        Ok(())
    })
}
```

`Box<dyn Future>` มีขนาด fixed (pointer + vtable) จึงทำลาย infinite type recursion

### Pitfall 5: RDATA กับ Compression Pointer ต้องใช้ full_msg

CNAME, NS, MX records มี domain name ใน RDATA ซึ่งอาจมี compression pointer ชี้กลับไปที่ส่วนต้นของ **ข้อความทั้งหมด** ไม่ใช่แค่ rdata slice

```rust
// ผิด! — decode ใน rdata slice เท่านั้น pointer จะ out of bounds
let (name, _) = decode_name(rdata_slice, 0)?;

// ถูกต้อง — ต้องหา offset ของ rdata ใน full message
pub fn parse_rdata(rtype: RecordType, rdata: &[u8], full_msg: &[u8]) -> ...
// แล้วหา offset ของ rdata ใน full_msg สำหรับ decode_name
```

### Pitfall 6: DNS ID Collision และ Amplification

**ID collision:** ถ้าใช้ ID ที่คาดเดาได้ (เช่น `1` ทุก query) server ที่ malicious สามารถ spoof response ได้ ควรใช้ random หรือ time-based ID

**DNS Amplification Attack:** recursive resolver ที่ expose ต่อ internet อาจถูกใช้สำหรับ DDoS amplification เพราะ DNS response ใหญ่กว่า query มาก ควรจำกัดว่า IP ไหนสามารถ query ได้

### Pitfall 7: SOA rname ไม่ใช่ email ธรรมดา

SOA record มี field `rname` ซึ่งดูเหมือน domain name แต่จริง ๆ คือ email ที่แทน `@` ด้วย `.`:

```
rname "admin.example.com" = email "admin@example.com"
```

จุดเข้าใจผิดคือ `hostmaster.example.com` เป็น convention ที่พบบ่อย ไม่ใช่ literal nameserver

## การ Package และ Deploy

### Build Release Binary

```bash
# Optimized release build
cargo build --release

# Binary อยู่ที่
./target/release/dns-resolver
```

### ใช้งาน Binary

```bash
# Query A record
./target/release/dns-resolver example.com A @8.8.8.8

# Query MX records
./target/release/dns-resolver gmail.com MX @1.1.1.1

# Query TXT (SPF)
./target/release/dns-resolver example.com TXT

# ใช้ root server โดยตรง
./target/release/dns-resolver com NS @198.41.0.4
```

### Docker Image

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/dns-resolver /usr/local/bin/
ENTRYPOINT ["dns-resolver"]
```

```bash
docker build -t dns-resolver .
docker run dns-resolver example.com A @8.8.8.8
```

### Static Binary (musl)

```bash
# เพิ่ม target
rustup target add x86_64-unknown-linux-musl

# Build static binary ที่ไม่ต้องการ shared libraries
cargo build --release --target x86_64-unknown-linux-musl

# ขนาด binary ลดลงเพิ่มเติมด้วย strip
strip target/x86_64-unknown-linux-musl/release/dns-resolver
```

Static binary มีประโยชน์มากสำหรับ container-based deployment เพราะสามารถ copy เข้าไป container ได้เลยโดยไม่ต้องใส่ library

## การต่อยอด (Extensions & Exercises)

### 1. เพิ่ม EDNS0 Support (ง่าย)

RFC 2671/6891 กำหนด EDNS0 ซึ่งขยาย UDP payload size เกิน 512 bytes โดยเพิ่ม OPT pseudo-record ใน additional section:

```rust
/// เพิ่ม OPT record ใน additional section ของ query
fn add_edns0(buf: &mut Vec<u8>, max_payload: u16) {
    // NAME = root (0x00)
    buf.push(0x00);
    // TYPE = OPT (41)
    buf.extend_from_slice(&41u16.to_be_bytes());
    // CLASS = payload size (4096 bytes recommended)
    buf.extend_from_slice(&max_payload.to_be_bytes());
    // TTL = extended RCODE + flags (all zero)
    buf.extend_from_slice(&0u32.to_be_bytes());
    // RDLENGTH = 0 (no options)
    buf.extend_from_slice(&0u16.to_be_bytes());
}
```

เพิ่ม OPT record นี้ใน `build_query()` และ increment arcount เป็น 1 — server ส่วนใหญ่จะตอบด้วย EDNS0 response ขนาดใหญ่กว่า 512 bytes

### 2. DNSSEC Validation (ยาก)

DNSSEC เพิ่ม digital signature ให้กับ DNS records เพื่อป้องกัน cache poisoning:
- Parse RRSIG, DNSKEY, DS record types
- ตรวจสอบ chain of trust จาก root KSK
- Flag responses ว่า AD (Authenticated Data)

Crate ที่มีประโยชน์: `ring` (cryptography), `trust-dns-proto` (ถ้ายอมใช้ library)

### 3. DNS-over-HTTPS (DoH) Client (กลาง)

RFC 8484 กำหนดวิธีส่ง DNS query ผ่าน HTTPS โดยส่ง binary wire format ใน POST body:

```rust
// DoH endpoint: https://dns.google/dns-query
// Content-Type: application/dns-message
// Accept: application/dns-message

let response = reqwest::Client::new()
    .post("https://dns.google/dns-query")
    .header("content-type", "application/dns-message")
    .body(query_bytes)
    .send()
    .await?;
```

เพิ่ม crate `reqwest` และ feature `native-tls` เพื่อรองรับ DoH

### 4. Zone File Parser (กลาง)

ไฟล์ zone (เช่น `/etc/bind/db.example.com`) ใช้ text format ที่กำหนดใน RFC 1035:

```
; Zone file for example.com
$ORIGIN example.com.
$TTL 3600
@   IN SOA ns1.example.com. admin.example.com. (
            2024010101  ; Serial
            7200        ; Refresh
            3600        ; Retry
            1209600     ; Expire
            3600 )      ; Minimum
    IN NS ns1.example.com.
    IN A  203.0.113.1
www IN A  203.0.113.1
```

Implementation นี้จะทำให้ resolver ของเราสามารถ serve authoritative DNS responses จาก zone file ได้เอง

### 5. Parallel Multi-Server Query (ง่าย)

แทนที่จะ query server ทีละตัวตามลำดับ ส่ง query ไปทุก server พร้อมกันแล้วใช้ผล response แรกที่มาถึง:

```rust
use tokio::select;
use futures::future::select_all;

pub async fn query_parallel(
    servers: &[String],
    name: &str,
    qtype: RecordType,
    timeout_ms: u64,
) -> Result<DnsMessage, ResolverError> {
    let futures: Vec<_> = servers.iter()
        .map(|s| Box::pin(query_server(s, name, qtype, timeout_ms)))
        .collect();

    let (result, _, _) = select_all(futures).await;
    result
}
```

Approach นี้คือ "happy eyeballs" pattern ที่ browser ใช้สำหรับ IPv4/IPv6 connection

### 6. Prometheus Metrics Exporter (กลาง)

เพิ่ม HTTP endpoint `/metrics` ที่ expose cache statistics ในรูปแบบ Prometheus:

```
dns_cache_hits_total 150
dns_cache_misses_total 42
dns_cache_entries 89
dns_cache_hit_rate 0.781
dns_queries_total{type="A"} 130
dns_queries_total{type="MX"} 20
dns_query_duration_seconds_p99 0.042
```

Crate ที่ใช้: `prometheus` หรือ `metrics` + `metrics-exporter-prometheus`

## สรุป

ในโปรเจคนี้เราสร้าง DNS Resolver ที่สมบูรณ์ครอบคลุม:

**Binary Protocol Parsing:** เรียนรู้ว่า network protocols ทำงานกับ raw bytes อย่างไร — `u16::from_be_bytes`, bit masking/shifting, offset-based parsing ที่ต้องจัดการ compression pointer อย่างระมัดระวัง

**Async Network I/O:** ใช้ `tokio::net::UdpSocket` และ `TcpStream` สำหรับ network query พร้อม `tokio::time::timeout` ป้องกัน hanging และ implement TCP fallback ที่ DNS protocol กำหนด

**CNAME Chain Following:** เข้าใจ pattern recursive async ใน Rust ที่ต้องใช้ `Box::pin` — ปัญหา infinitely-sized future และวิธีแก้โดยการย้ายไปบน heap

**TTL-based Cache:** `HashMap` + `Instant` สำหรับ lazy eviction ทั้ง positive และ negative caching พร้อม statistics — pattern นี้นำไปใช้กับ caching use case อื่น ๆ ได้ทันที

**Iterative Resolution:** เข้าใจว่า DNS hierarchy ทำงานอย่างไรในโลกจริง — root servers → TLD → authoritative NS

Pattern เหล่านี้นำไปใช้ได้กับ network protocol implementation อื่น ๆ เช่น HTTP/1.1 parser, MQTT protocol, gRPC framing, หรือ Redis RESP protocol — โครงสร้างพื้นฐานคล้ายกันทั้งหมด: fixed header → variable-length data → parse recursively

---

**โปรเจคก่อนหน้า:** [WebSocket Server](project-h02-websocket.md) | **โปรเจคถัดไป:** [MQTT Broker](project-h04-mqtt-broker.md)
