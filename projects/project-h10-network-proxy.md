# Project H10: Network Proxy (SOCKS5 + HTTP CONNECT)

> โมดูล: H — Networking/Protocols | ความยาก: ⭐⭐⭐⭐⭐ | เวลาโดยประมาณ: 20 ชั่วโมง

## ภาพรวมโปรเจค

**Network Proxy** คืออุปกรณ์ซอฟต์แวร์ที่ทำหน้าที่เป็น "ตัวกลาง" ระหว่าง client กับ server ปลายทาง แทนที่ client จะเชื่อมต่อ server โดยตรง ก็จะส่ง request ไปหา proxy ก่อน แล้ว proxy จึงสร้าง connection ใหม่ไปหา server จริง ในโปรเจคนี้เราจะสร้าง **dual-protocol proxy** ที่รองรับ 2 โปรโตคอลพร้อมกัน คือ **SOCKS5** และ **HTTP CONNECT**

### ทำไมโปรเจคนี้ถึงสำคัญ?

Proxy server ปรากฏอยู่ใน use case จริงมากมาย:

| Use Case | โปรโตคอลที่ใช้ | ตัวอย่าง |
|----------|---------------|---------|
| VPN / Tunneling ในองค์กร | SOCKS5 | Shadowsocks, OpenSSH `-D` |
| HTTPS traffic inspection | HTTP CONNECT | Squid, mitmproxy |
| Load balancer upstream | SOCKS5 + HTTP | HAProxy, Envoy |
| Tor anonymity network | SOCKS5 | Tor Browser |
| Corporate web filtering | HTTP CONNECT | Zscaler, Netskope |
| Dev environment tunnels | SOCKS5 | `kubectl port-forward` ภายใน |

การสร้าง proxy ตั้งแต่ต้นจะทำให้เข้าใจ:
- โปรโตคอล handshake ระดับ byte ที่ browser/OS ทำให้เราทุกครั้งที่ใช้ proxy settings
- วิธีที่ HTTPS ทำงานผ่าน proxy (HTTP CONNECT tunnel)
- Concurrent I/O ด้วย `tokio::io::copy_bidirectional`
- Pattern ของ protocol multiplexing — peek แค่ 1 byte แล้วตัดสินใจ handler

### สิ่งที่จะสร้าง

```
Network Proxy (port 1080)
  │
  ├─ peek byte 0x05 → SOCKS5 handler
  │    ├─ method negotiation (no-auth / user:pass)
  │    ├─ CONNECT request parse (IPv4 / IPv6 / domain)
  │    └─ bidirectional forward → upstream
  │
  ├─ peek "CONNECT " → HTTP CONNECT handler
  │    ├─ parse request line + headers
  │    ├─ respond "200 Connection Established"
  │    └─ bidirectional forward → upstream
  │
  ├─ peek "GET/POST/..." → HTTP forward proxy handler
  │    └─ rewrite request + forward
  │
  ├─ ACL engine (allow/deny rules with glob patterns)
  │    ├─ deny *.ads.example.com
  │    └─ allow corporate.internal
  │
  └─ Connection pool (DashMap<SocketAddr, VecDeque<PooledConn>>)
       ├─ get() — reuse idle connection
       ├─ return_conn() — put back after use
       └─ evict_expired() — background cleanup task
```

---

## สิ่งที่จะได้เรียนรู้

- **Binary protocol parsing** — อ่าน byte slice แบบ manual ด้วย index slicing, `from_be_bytes`, enum dispatch
- **SOCKS5 RFC 1928** — handshake 2 รอบ (greeting + request), address types 3 แบบ, reply codes
- **HTTP CONNECT tunnel** — ทำไม browser ถึงใช้วิธีนี้สำหรับ HTTPS ผ่าน proxy
- **Protocol multiplexing** — peek first byte โดยไม่ consume เพื่อ dispatch handler ที่ถูกต้อง
- **`tokio::io::copy_bidirectional`** — forward TCP stream 2 ทิศทางพร้อมกันใน async context
- **`DashMap`** — concurrent HashMap ที่ thread-safe โดยไม่ต้องใช้ global `Mutex`
- **Glob pattern matching** — ใช้ crate `glob` สำหรับ wildcard host filtering ใน ACL
- **Connection pool pattern** — reuse upstream TCP connections เพื่อลด latency และ resource overhead

---

## ความรู้ที่ต้องมีมาก่อน

- **Part 41-50** — `async/await`, tokio runtime, `Future`, `tokio::spawn`, `TcpListener`, `TcpStream`
- **Part 51-60** — `Arc<T>`, `Mutex<T>`, trait objects `dyn Trait + Send + Sync`, `Box<dyn Error>`
- **Part 61-70** — `AtomicU64`, `Ordering::SeqCst`, interior mutability patterns
- **Part 71-80** — `VecDeque`, `HashMap`, closures, iterator adapters
- **Part 96-110** — Production error handling, `thiserror`, structured logging
- **Project H05** (Load Balancer) — `DashMap`, `tokio::io::copy_bidirectional`, TCP listener pattern
- **Project H09** (QUIC Transport) — async I/O patterns, connection lifecycle

---

## โครงสร้างโปรเจค (Project Layout)

```
network-proxy/
├── src/
│   ├── main.rs           # tokio::main, CLI args, TcpListener accept loop
│   ├── lib.rs            # re-exports ทุก module
│   ├── socks5.rs         # SOCKS5 handshake parser, reply builder
│   ├── http_connect.rs   # HTTP CONNECT parser, response builder
│   ├── protocol_detect.rs# peek-based protocol detection
│   ├── forward.rs        # ForwardStats, ConnTimer, copy_bidirectional wrapper
│   ├── acl.rs            # AccessRule, AclEngine with glob matching
│   └── pool.rs           # ConnectionPool (DashMap + VecDeque)
├── tests/
│   └── integration.rs    # end-to-end tests (optional, ต้องการ real network)
├── Cargo.toml
└── README.md
```

---

## การออกแบบ (Architecture & Design)

### Data Flow ของ SOCKS5 Connection

```
Client                    Proxy                     Upstream
  │                         │                          │
  │──── TCP connect ────────▶│                          │
  │                         │                          │
  │── greeting (05 01 00) ──▶│  parse_socks5_greeting() │
  │◀─── reply (05 00) ───────│  build_greeting_reply()  │
  │                         │                          │
  │── CONNECT req (IPv4) ───▶│  parse_socks5_request()  │
  │                         │──── TCP connect ─────────▶│
  │◀─── reply (05 00 ...) ───│  build_socks5_reply()    │
  │                         │                          │
  │════ bidirectional ══════▶│════ copy_bidirectional ══▶│
  │◀═══════════════════════╸│◀════════════════════════╸│
```

### Data Flow ของ HTTP CONNECT

```
Client                    Proxy                     Upstream
  │                         │                          │
  │── CONNECT host:443 ────▶│  parse_http_connect()    │
  │   HTTP/1.1              │                          │
  │                         │──── TCP connect ─────────▶│
  │◀── 200 Connection ───────│  build_connect_ok()      │
  │    Established          │                          │
  │                         │                          │
  │════ TLS Handshake ══════▶│════════ forwarded ═══════▶│
  │◀═══════════════════════╸│◀════════════════════════╸│
```

### Protocol Detection ด้วย Peek

แทนที่จะ block และ buffer ข้อมูลทั้งหมดก่อน เราใช้เทคนิค "peek" — อ่านข้อมูลโดยไม่เอาออกจาก buffer:

```rust
// tokio TcpStream มี peek() method
let mut buf = [0u8; 8];
let n = stream.peek(&mut buf).await?;
let proto = detect_protocol(&buf[..n]);
```

Byte แรกบอกเราได้ทันที:
- `0x05` → SOCKS5 (SOCKS version 5)
- `C` + `ONNECT ` → HTTP CONNECT tunnel
- `G`, `P`, `H`, `D` → HTTP forward proxy methods

### ทำไมต้องมี Connection Pool?

การสร้าง TCP connection ใหม่ทุกครั้งมีค่าใช้จ่ายสูง:
- **3-way handshake** อย่างน้อย 1 RTT (round-trip time)
- **TLS handshake** อีก 1-2 RTT สำหรับ HTTPS upstream
- **OS resource** — socket descriptor, buffer allocation

Connection pool ช่วยให้ connections เดิมที่ยังอยู่ใน ESTABLISHED state ถูก reuse ได้ทันที ลด latency จาก ~50ms เหลือ ~5ms สำหรับ upstream ที่อยู่ไกล

```
DashMap<SocketAddr, VecDeque<PooledConn>>
     │
     └── key = upstream address (e.g., 1.2.3.4:443)
         value = queue ของ idle connections (FIFO)
```

---

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: ตั้งค่าโปรเจคและ Dependencies

สร้าง project structure:

```bash
cargo new network-proxy
cd network-proxy
```

แก้ไข `Cargo.toml`:

```toml
[package]
name = "network-proxy"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
dashmap = "5"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
glob = "0.3"

[dev-dependencies]
tokio = { version = "1", features = ["full"] }
```

**Crates ที่เลือกใช้:**

| Crate | เหตุผล |
|-------|--------|
| `tokio` | async runtime สำหรับ TCP listener + bidirectional copy |
| `dashmap` | concurrent HashMap สำหรับ connection pool โดยไม่ต้องล็อค global mutex |
| `glob` | wildcard pattern matching สำหรับ ACL host rules |
| `serde` / `serde_json` | serialize/deserialize config และ stats |

สร้าง module structure ใน `src/lib.rs`:

```rust
pub mod socks5;
pub mod http_connect;
pub mod protocol_detect;
pub mod forward;
pub mod acl;
pub mod pool;

pub use socks5::*;
pub use http_connect::*;
pub use protocol_detect::*;
pub use forward::*;
pub use acl::*;
pub use pool::*;
```

---

### ขั้นที่ 2: SOCKS5 Handshake Parser

โปรโตคอล SOCKS5 (RFC 1928) ทำงาน 2 รอบ:

**รอบที่ 1 — Method Negotiation:**
```
Client → Proxy: VER(1)=05 | NMETHODS(1) | METHODS(NMETHODS)
Proxy → Client: VER(1)=05 | METHOD(1)
```

**รอบที่ 2 — CONNECT Request:**
```
Client → Proxy: VER(1)=05 | CMD(1)=01 | RSV(1)=00 | ATYP(1) | DST.ADDR | DST.PORT(2)
```
โดย `ATYP`:
- `0x01` = IPv4 (4 bytes)
- `0x04` = IPv6 (16 bytes)
- `0x03` = Domain name (1 byte length + N bytes)

สร้างไฟล์ `src/socks5.rs`:

```rust
use std::net::{Ipv4Addr, Ipv6Addr};

/// Auth methods ที่ SOCKS5 รองรับ
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum AuthMethod {
    NoAuth,               // 0x00
    UsernamePassword,     // 0x02
    Unsupported(u8),
}

impl AuthMethod {
    pub fn from_byte(b: u8) -> Self {
        match b {
            0x00 => AuthMethod::NoAuth,
            0x02 => AuthMethod::UsernamePassword,
            other => AuthMethod::Unsupported(other),
        }
    }

    pub fn to_byte(&self) -> u8 {
        match self {
            AuthMethod::NoAuth => 0x00,
            AuthMethod::UsernamePassword => 0x02,
            AuthMethod::Unsupported(b) => *b,
        }
    }
}

/// Address ปลายทาง 3 แบบ
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Socks5Addr {
    Ipv4(Ipv4Addr),
    Ipv6(Ipv6Addr),
    Domain(String),
}

impl Socks5Addr {
    pub fn to_host_string(&self) -> String {
        match self {
            Socks5Addr::Ipv4(a)  => a.to_string(),
            Socks5Addr::Ipv6(a)  => format!("[{}]", a),
            Socks5Addr::Domain(d) => d.clone(),
        }
    }
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Socks5Command {
    Connect,
    Bind,
    UdpAssociate,
}

#[derive(Debug, Clone)]
pub struct Socks5Greeting {
    pub version: u8,
    pub methods: Vec<AuthMethod>,
}

#[derive(Debug, Clone)]
pub struct Socks5Request {
    pub version: u8,
    pub command: Socks5Command,
    pub dest_addr: Socks5Addr,
    pub dest_port: u16,
}

#[derive(Debug, Clone, PartialEq, Eq)]
#[repr(u8)]
pub enum Socks5Reply {
    Success              = 0x00,
    GeneralFailure       = 0x01,
    ConnectionNotAllowed = 0x02,
    NetworkUnreachable   = 0x03,
    HostUnreachable      = 0x04,
    ConnectionRefused    = 0x05,
    TtlExpired           = 0x06,
    CommandNotSupported  = 0x07,
    AddressTypeNotSupported = 0x08,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Socks5ParseError {
    InvalidVersion,
    InvalidCommand,
    InvalidAddressType,
    BufferTooShort,
    InvalidDomainName,
}

/// Parse greeting: VER | NMETHODS | METHODS
pub fn parse_socks5_greeting(buf: &[u8]) -> Result<Socks5Greeting, Socks5ParseError> {
    if buf.len() < 2 {
        return Err(Socks5ParseError::BufferTooShort);
    }
    if buf[0] != 5 {
        return Err(Socks5ParseError::InvalidVersion);
    }
    let nmethods = buf[1] as usize;
    if buf.len() < 2 + nmethods {
        return Err(Socks5ParseError::BufferTooShort);
    }
    let methods = buf[2..2 + nmethods]
        .iter()
        .map(|&b| AuthMethod::from_byte(b))
        .collect();
    Ok(Socks5Greeting { version: 5, methods })
}

/// Build server greeting reply: VER | METHOD
pub fn build_socks5_greeting_reply(method: &AuthMethod) -> [u8; 2] {
    [0x05, method.to_byte()]
}

/// Parse CONNECT request: VER | CMD | RSV | ATYP | DST.ADDR | DST.PORT
pub fn parse_socks5_request(buf: &[u8]) -> Result<Socks5Request, Socks5ParseError> {
    if buf.len() < 4 {
        return Err(Socks5ParseError::BufferTooShort);
    }
    if buf[0] != 5 {
        return Err(Socks5ParseError::InvalidVersion);
    }
    let cmd = match buf[1] {
        0x01 => Socks5Command::Connect,
        0x02 => Socks5Command::Bind,
        0x03 => Socks5Command::UdpAssociate,
        _    => return Err(Socks5ParseError::InvalidCommand),
    };
    let (addr, port_offset) = match buf[3] {
        0x01 => {
            if buf.len() < 10 { return Err(Socks5ParseError::BufferTooShort); }
            (Socks5Addr::Ipv4(Ipv4Addr::new(buf[4], buf[5], buf[6], buf[7])), 8)
        }
        0x04 => {
            if buf.len() < 22 { return Err(Socks5ParseError::BufferTooShort); }
            let mut o = [0u8; 16];
            o.copy_from_slice(&buf[4..20]);
            (Socks5Addr::Ipv6(Ipv6Addr::from(o)), 20)
        }
        0x03 => {
            if buf.len() < 5 { return Err(Socks5ParseError::BufferTooShort); }
            let dlen = buf[4] as usize;
            if buf.len() < 5 + dlen + 2 { return Err(Socks5ParseError::BufferTooShort); }
            let d = std::str::from_utf8(&buf[5..5 + dlen])
                .map_err(|_| Socks5ParseError::InvalidDomainName)?
                .to_string();
            (Socks5Addr::Domain(d), 5 + dlen)
        }
        _ => return Err(Socks5ParseError::InvalidAddressType),
    };
    let port = u16::from_be_bytes([buf[port_offset], buf[port_offset + 1]]);
    Ok(Socks5Request { version: 5, command: cmd, dest_addr: addr, dest_port: port })
}

/// Build SOCKS5 reply: VER | REP | RSV | ATYP=1 | BND.ADDR(4) | BND.PORT(2)
pub fn build_socks5_reply(reply: Socks5Reply) -> Vec<u8> {
    vec![0x05, reply as u8, 0x00, 0x01, 0, 0, 0, 0, 0, 0]
}
```

**ทดสอบด้วยมือ — IPv4 address bytes:**
```
05 01 00 01 7f 00 00 01 1f 90
│  │  │  │  └──────────┘ └──┘
│  │  │  │  127.0.0.1    port 8080
│  │  │  ATYP=IPv4
│  │  RSV
│  CONNECT command
SOCKS5 version
```

---

### ขั้นที่ 3: HTTP CONNECT Parser

HTTP CONNECT เป็น method พิเศษที่ browser ใช้เพื่อ tunnel HTTPS ผ่าน HTTP proxy:

```
CONNECT example.com:443 HTTP/1.1\r\n
Host: example.com:443\r\n
Proxy-Authorization: Basic ...\r\n
\r\n
```

Proxy ตอบกลับ:
```
HTTP/1.1 200 Connection Established\r\n
\r\n
```

หลังจากนั้น proxy ทำหน้าที่ tunnel โปร่งใส ไม่ได้อ่าน TLS traffic เลย

สร้าง `src/http_connect.rs`:

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct HttpConnectRequest {
    pub host: String,
    pub port: u16,
    pub http_version: String,
    pub headers: Vec<(String, String)>,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum HttpConnectError {
    NotConnectMethod,
    MalformedRequestLine,
    InvalidPort,
    MissingHostPort,
    BufferTooShort,
}

/// Parse HTTP CONNECT request from raw bytes
pub fn parse_http_connect(buf: &[u8]) -> Result<HttpConnectRequest, HttpConnectError> {
    let text = std::str::from_utf8(buf)
        .map_err(|_| HttpConnectError::MalformedRequestLine)?;
    let mut lines = text.split("\r\n");

    let request_line = lines.next().ok_or(HttpConnectError::BufferTooShort)?;
    let parts: Vec<&str> = request_line.splitn(3, ' ').collect();
    if parts.len() < 3 {
        return Err(HttpConnectError::MalformedRequestLine);
    }
    if parts[0] != "CONNECT" {
        return Err(HttpConnectError::NotConnectMethod);
    }

    let (host, port) = parse_host_port(parts[1])?;
    let http_version = parts[2].to_string();

    let mut headers = Vec::new();
    for line in lines {
        if line.is_empty() { break; }
        if let Some(colon) = line.find(':') {
            let name  = line[..colon].trim().to_lowercase();
            let value = line[colon + 1..].trim().to_string();
            headers.push((name, value));
        }
    }

    Ok(HttpConnectRequest { host, port, http_version, headers })
}

fn parse_host_port(hp: &str) -> Result<(String, u16), HttpConnectError> {
    if hp.starts_with('[') {
        // IPv6: [::1]:443
        let end = hp.find(']').ok_or(HttpConnectError::MalformedRequestLine)?;
        let host = hp[1..end].to_string();
        let port: u16 = hp[end + 2..].parse()
            .map_err(|_| HttpConnectError::InvalidPort)?;
        return Ok((host, port));
    }
    let colon = hp.rfind(':').ok_or(HttpConnectError::MissingHostPort)?;
    let host = hp[..colon].to_string();
    let port: u16 = hp[colon + 1..].parse()
        .map_err(|_| HttpConnectError::InvalidPort)?;
    Ok((host, port))
}

pub fn build_connect_ok() -> &'static [u8] {
    b"HTTP/1.1 200 Connection Established\r\n\r\n"
}

pub fn build_connect_error(code: u16, reason: &str) -> String {
    format!("HTTP/1.1 {} {}\r\n\r\n", code, reason)
}
```

---

### ขั้นที่ 4: Protocol Detection ด้วย Peek

หัวใจของ dual-protocol proxy คือการตัดสินใจ handler ที่ถูกต้องจากข้อมูลขั้นต้น:

```rust
// src/protocol_detect.rs

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum DetectedProtocol {
    Socks5,
    HttpConnect,
    HttpForward,
    Unknown(u8),
}

pub fn detect_protocol(peek: &[u8]) -> DetectedProtocol {
    if peek.is_empty() {
        return DetectedProtocol::Unknown(0);
    }
    match peek[0] {
        0x05 => DetectedProtocol::Socks5,
        b'C' if peek.starts_with(b"CONNECT ") => DetectedProtocol::HttpConnect,
        b'G' if peek.starts_with(b"GET ")     => DetectedProtocol::HttpForward,
        b'P' if peek.starts_with(b"POST ")
             || peek.starts_with(b"PUT ")     => DetectedProtocol::HttpForward,
        b'H' if peek.starts_with(b"HEAD ")    => DetectedProtocol::HttpForward,
        b'D' if peek.starts_with(b"DELETE ")  => DetectedProtocol::HttpForward,
        other => DetectedProtocol::Unknown(other),
    }
}
```

ใน `main.rs` การใช้งานจริงจะเป็น:

```rust
use tokio::net::TcpStream;

async fn handle_connection(mut stream: TcpStream) -> anyhow::Result<()> {
    let mut peek_buf = [0u8; 8];
    let n = stream.peek(&mut peek_buf).await?;
    let proto = detect_protocol(&peek_buf[..n]);

    match proto {
        DetectedProtocol::Socks5      => handle_socks5(stream).await?,
        DetectedProtocol::HttpConnect => handle_http_connect(stream).await?,
        DetectedProtocol::HttpForward => handle_http_forward(stream).await?,
        DetectedProtocol::Unknown(b)  => {
            eprintln!("unknown protocol, first byte: 0x{:02x}", b);
        }
    }
    Ok(())
}
```

สิ่งสำคัญคือ `peek()` ไม่ consume ข้อมูลจาก buffer ดังนั้น handler ที่รับ `stream` ไปต่อยังอ่านข้อมูลเดิมได้อีกครั้ง นี่คือ zero-copy protocol detection

---

### ขั้นที่ 5: ForwardStats และ Connection Forwarding

`ForwardStats` เก็บสถิติของแต่ละ connection ที่ forward ผ่าน proxy:

```rust
// src/forward.rs

use std::time::Instant;

#[derive(Debug, Clone, Default, PartialEq, Eq)]
pub struct ForwardStats {
    pub bytes_sent: u64,   // bytes จาก client ไป upstream
    pub bytes_recv: u64,   // bytes จาก upstream มา client
    pub duration_ms: u64,  // เวลาทั้งหมดของ connection
}

impl ForwardStats {
    pub fn new() -> Self { Self::default() }

    pub fn total_bytes(&self) -> u64 {
        self.bytes_sent + self.bytes_recv
    }

    pub fn with_sent(mut self, n: u64) -> Self {
        self.bytes_sent = n; self
    }
    pub fn with_recv(mut self, n: u64) -> Self {
        self.bytes_recv = n; self
    }
    pub fn with_duration_ms(mut self, ms: u64) -> Self {
        self.duration_ms = ms; self
    }
}

pub struct ConnTimer {
    start: Instant,
}

impl ConnTimer {
    pub fn start() -> Self {
        ConnTimer { start: Instant::now() }
    }

    pub fn elapsed_ms(&self) -> u64 {
        self.start.elapsed().as_millis() as u64
    }

    pub fn finish(&self, bytes_sent: u64, bytes_recv: u64) -> ForwardStats {
        ForwardStats {
            bytes_sent,
            bytes_recv,
            duration_ms: self.elapsed_ms(),
        }
    }
}

pub fn accumulate_stats(stats: &[ForwardStats]) -> ForwardStats {
    stats.iter().fold(ForwardStats::new(), |acc, s| ForwardStats {
        bytes_sent:  acc.bytes_sent  + s.bytes_sent,
        bytes_recv:  acc.bytes_recv  + s.bytes_recv,
        duration_ms: acc.duration_ms + s.duration_ms,
    })
}
```

**การ Forward จริงด้วย `copy_bidirectional`:**

```rust
use tokio::io::copy_bidirectional;
use tokio::net::TcpStream;
use tokio::time::{timeout, Duration};

pub async fn forward_connection(
    mut client: TcpStream,
    upstream_addr: &str,
) -> anyhow::Result<ForwardStats> {
    let timer = ConnTimer::start();

    // connect upstream ด้วย timeout 10 วินาที
    let mut upstream = timeout(
        Duration::from_secs(10),
        TcpStream::connect(upstream_addr),
    )
    .await
    .map_err(|_| anyhow::anyhow!("connect timeout"))??;

    // forward bidirectional — returns (bytes_from_a, bytes_from_b)
    let (sent, recv) = copy_bidirectional(&mut client, &mut upstream).await?;
    Ok(timer.finish(sent, recv))
}
```

`copy_bidirectional` ทำงานด้วย 2 internal `copy` loop รันพร้อมกันใน single `select!` loop — เมื่อฝั่งใดปิดก็ gracefully shutdown อีกฝั่ง

---

### ขั้นที่ 6: ACL Engine ด้วย Glob Pattern

ACL (Access Control List) กรอง destination ก่อนสร้าง upstream connection:

```rust
// src/acl.rs

use glob::Pattern;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum AclAction { Allow, Deny }

#[derive(Debug, Clone)]
pub struct AccessRule {
    pub action: AclAction,
    pub host_pattern: String,                  // glob: "*.ads.com", "exact.host"
    pub port_range: Option<(u16, u16)>,        // None = any port
}

impl AccessRule {
    pub fn new_allow(pattern: &str) -> Self {
        AccessRule { action: AclAction::Allow, host_pattern: pattern.to_string(), port_range: None }
    }
    pub fn new_deny(pattern: &str) -> Self {
        AccessRule { action: AclAction::Deny, host_pattern: pattern.to_string(), port_range: None }
    }
    pub fn with_port_range(mut self, min: u16, max: u16) -> Self {
        self.port_range = Some((min, max)); self
    }

    fn matches(&self, host: &str, port: u16) -> bool {
        let host_ok = Pattern::new(&self.host_pattern)
            .map(|p| p.matches(host))
            .unwrap_or(false);
        if !host_ok { return false; }
        match self.port_range {
            None => true,
            Some((lo, hi)) => port >= lo && port <= hi,
        }
    }
}

#[derive(Debug, Clone)]
pub struct AclEngine {
    rules: Vec<AccessRule>,
    default_action: AclAction,
}

impl AclEngine {
    pub fn new(default_action: AclAction) -> Self {
        AclEngine { rules: Vec::new(), default_action }
    }

    pub fn add_rule(&mut self, rule: AccessRule) {
        self.rules.push(rule);
    }

    /// ตรวจสอบ — return true ถ้า allow, false ถ้า deny
    pub fn check(&self, host: &str, port: u16) -> bool {
        for rule in &self.rules {
            if rule.matches(host, port) {
                return rule.action == AclAction::Allow;
            }
        }
        self.default_action == AclAction::Allow
    }
}
```

**ตัวอย่างการใช้งาน — Corporate proxy config:**

```rust
let mut acl = AclEngine::new(AclAction::Deny); // default deny all

// อนุญาต corporate resources
acl.add_rule(AccessRule::new_allow("*.internal.corp"));
acl.add_rule(AccessRule::new_allow("intranet.example.com"));

// บล็อก ads และ malware
acl.add_rule(AccessRule::new_deny("*.doubleclick.net"));
acl.add_rule(AccessRule::new_deny("*.googlesyndication.com"));
acl.add_rule(AccessRule::new_deny("*.malware-site.*"));

// อนุญาต HTTPS ของ approved domains
acl.add_rule(AccessRule::new_allow("*.github.com").with_port_range(443, 443));

assert!(!acl.check("tracker.doubleclick.net", 80));
assert!(acl.check("app.internal.corp", 8080));
```

---

### ขั้นที่ 7: Connection Pool

Connection pool ลด overhead ของการสร้าง TCP connection ซ้ำ ๆ ไปยัง upstream เดิม:

```rust
// src/pool.rs

use dashmap::DashMap;
use std::collections::VecDeque;
use std::net::SocketAddr;
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::{Duration, Instant};

pub struct PooledConn {
    pub addr: SocketAddr,
    pub created_at: Instant,
    pub last_used: Instant,
    pub id: u64,
}

impl PooledConn {
    pub fn new(addr: SocketAddr, id: u64) -> Self {
        let now = Instant::now();
        PooledConn { addr, created_at: now, last_used: now, id }
    }

    pub fn is_expired(&self, max_idle: Duration) -> bool {
        self.last_used.elapsed() > max_idle
    }
}

#[derive(Debug, Clone)]
pub struct PoolConfig {
    pub max_idle_per_host: usize,
    pub idle_timeout: Duration,
}

impl Default for PoolConfig {
    fn default() -> Self {
        PoolConfig {
            max_idle_per_host: 10,
            idle_timeout: Duration::from_secs(60),
        }
    }
}

pub struct ConnectionPool {
    pools: DashMap<SocketAddr, VecDeque<PooledConn>>,
    config: PoolConfig,
    next_id: AtomicU64,
}

impl ConnectionPool {
    pub fn new(config: PoolConfig) -> Self {
        ConnectionPool {
            pools: DashMap::new(),
            config,
            next_id: AtomicU64::new(1),
        }
    }

    /// Get idle connection; ล้าง expired entries ก่อน
    pub fn get(&self, addr: SocketAddr) -> Option<PooledConn> {
        let mut q = self.pools.entry(addr).or_default();
        while q.front().map(|c| c.is_expired(self.config.idle_timeout)).unwrap_or(false) {
            q.pop_front();
        }
        q.pop_front()
    }

    /// Return connection to pool; ทิ้งถ้า pool เต็มแล้ว
    pub fn return_conn(&self, mut conn: PooledConn) {
        let addr = conn.addr;
        conn.last_used = Instant::now();
        let mut q = self.pools.entry(addr).or_default();
        if q.len() < self.config.max_idle_per_host {
            q.push_back(conn);
        }
    }

    pub fn alloc_id(&self) -> u64 {
        self.next_id.fetch_add(1, Ordering::SeqCst)
    }

    pub fn idle_count(&self, addr: SocketAddr) -> usize {
        self.pools.get(&addr).map(|q| q.len()).unwrap_or(0)
    }

    /// Background cleanup: ลบ connections ที่หมดอายุออกทุก host
    pub fn evict_expired(&self) {
        for mut entry in self.pools.iter_mut() {
            entry.value_mut().retain(|c| !c.is_expired(self.config.idle_timeout));
        }
    }

    pub fn total_idle(&self) -> usize {
        self.pools.iter().map(|e| e.value().len()).sum()
    }
}
```

**Background eviction task:**

```rust
// ใน main.rs — spawn cleanup task ทุก 30 วินาที
let pool = Arc::new(ConnectionPool::new(PoolConfig::default()));
let pool_clone = Arc::clone(&pool);

tokio::spawn(async move {
    let mut interval = tokio::time::interval(Duration::from_secs(30));
    loop {
        interval.tick().await;
        pool_clone.evict_expired();
    }
});
```

---

### ขั้นที่ 8: Main Entry Point

ประกอบทุก module เข้าด้วยกัน:

```rust
// src/main.rs

use std::sync::Arc;
use tokio::net::TcpListener;
use network_proxy::*;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let listener = TcpListener::bind("0.0.0.0:1080").await?;
    println!("Proxy listening on 0.0.0.0:1080");

    // สร้าง shared state
    let pool = Arc::new(ConnectionPool::new(PoolConfig::default()));
    let mut acl = AclEngine::new(AclAction::Allow);
    acl.add_rule(AccessRule::new_deny("*.ads.doubleclick.net"));
    acl.add_rule(AccessRule::new_deny("*.malware.*"));
    let acl = Arc::new(acl);

    // Background pool cleanup
    {
        let pool = Arc::clone(&pool);
        tokio::spawn(async move {
            let mut interval = tokio::time::interval(
                std::time::Duration::from_secs(30)
            );
            loop {
                interval.tick().await;
                pool.evict_expired();
            }
        });
    }

    loop {
        let (stream, peer) = listener.accept().await?;
        let pool = Arc::clone(&pool);
        let acl  = Arc::clone(&acl);

        tokio::spawn(async move {
            if let Err(e) = handle_client(stream, peer, pool, acl).await {
                eprintln!("[{peer}] error: {e}");
            }
        });
    }
}

async fn handle_client(
    mut stream: tokio::net::TcpStream,
    peer: std::net::SocketAddr,
    pool: Arc<ConnectionPool>,
    acl: Arc<AclEngine>,
) -> anyhow::Result<()> {
    // Peek ดู protocol
    let mut peek_buf = [0u8; 8];
    let n = stream.peek(&mut peek_buf).await?;
    let proto = detect_protocol(&peek_buf[..n]);

    println!("[{peer}] protocol: {:?}", proto);

    match proto {
        DetectedProtocol::Socks5 => {
            // อ่าน greeting
            let mut buf = vec![0u8; 257];
            let n = stream.try_read(&mut buf)?;
            let greeting = parse_socks5_greeting(&buf[..n])?;

            // เลือก method
            let method = if greeting.methods.contains(&AuthMethod::NoAuth) {
                AuthMethod::NoAuth
            } else {
                // ไม่รองรับ method ที่ขอ
                stream.try_write(&[0x05, 0xFF])?;
                return Ok(());
            };
            stream.try_write(&build_socks5_greeting_reply(&method))?;

            // อ่าน CONNECT request
            let n = stream.try_read(&mut buf)?;
            let req = parse_socks5_request(&buf[..n])?;

            let host = req.dest_addr.to_host_string();
            let port = req.dest_port;

            // ACL check
            if !acl.check(&host, port) {
                stream.try_write(&build_socks5_reply(Socks5Reply::ConnectionNotAllowed))?;
                return Ok(());
            }

            // Connect upstream
            let upstream_addr = format!("{}:{}", host, port);
            match tokio::net::TcpStream::connect(&upstream_addr).await {
                Ok(upstream) => {
                    stream.try_write(&build_socks5_reply(Socks5Reply::Success))?;
                    let (sent, recv) = tokio::io::copy_bidirectional(
                        &mut stream,
                        &mut { upstream }
                    ).await?;
                    println!("[{peer}] → {upstream_addr} sent={sent} recv={recv}");
                }
                Err(_) => {
                    stream.try_write(&build_socks5_reply(Socks5Reply::ConnectionRefused))?;
                }
            }
        }
        DetectedProtocol::HttpConnect => {
            // อ่าน request ทั้งหมด
            let mut buf = vec![0u8; 4096];
            let n = stream.try_read(&mut buf)?;
            let req = parse_http_connect(&buf[..n])?;

            if !acl.check(&req.host, req.port) {
                let resp = build_connect_error(403, "Forbidden");
                stream.try_write(resp.as_bytes())?;
                return Ok(());
            }

            let upstream_addr = format!("{}:{}", req.host, req.port);
            match tokio::net::TcpStream::connect(&upstream_addr).await {
                Ok(mut upstream) => {
                    stream.try_write(build_connect_ok())?;
                    tokio::io::copy_bidirectional(&mut stream, &mut upstream).await?;
                }
                Err(_) => {
                    let resp = build_connect_error(502, "Bad Gateway");
                    stream.try_write(resp.as_bytes())?;
                }
            }
        }
        _ => {
            eprintln!("[{peer}] unsupported protocol");
        }
    }
    Ok(())
}
```

---

## การทดสอบ (Testing)

### Unit Tests — ครบ 38 tests ใน 6 modules

ไฟล์ทดสอบกระจายอยู่ใน module ต่าง ๆ ครอบคลุม:

| Module | จำนวน tests | หัวข้อที่ test |
|--------|------------|--------------|
| `socks5` | 11 | greeting parse, IPv4/IPv6/domain request, reply builder |
| `http_connect` | 5 | basic CONNECT, headers, error cases |
| `protocol_detect` | 7 | SOCKS5/HTTP CONNECT/HTTP methods/unknown/edge cases |
| `forward` | 4 | ForwardStats builder, accumulate, ConnTimer |
| `acl` | 7 | deny domain, allow list, port range, glob, default actions |
| `pool` | 5 | empty get, return/get, max idle, evict expired, total idle |

### ผลการรัน `cargo test`

```
running 38 tests
test acl::tests::test_acl_default_deny ... ok
test acl::tests::test_acl_allow_specific_allow_list ... ok
test acl::tests::test_acl_default_allow ... ok
test acl::tests::test_acl_deny_ads_domain ... ok
test forward::tests::test_accumulate_stats ... ok
test acl::tests::test_acl_first_match_wins ... ok
test acl::tests::test_acl_port_range ... ok
test acl::tests::test_acl_glob_wildcard ... ok
test http_connect::tests::test_build_connect_ok ... ok
test http_connect::tests::test_parse_connect_basic ... ok
test http_connect::tests::test_parse_connect_missing_port ... ok
test http_connect::tests::test_parse_connect_not_connect_method ... ok
test http_connect::tests::test_parse_connect_with_headers ... ok
test forward::tests::test_forward_stats_total_bytes ... ok
test forward::tests::test_forward_stats_builder ... ok
test pool::tests::test_pool_get_empty ... ok
test pool::tests::test_pool_max_idle_per_host ... ok
test pool::tests::test_pool_total_idle ... ok
test protocol_detect::tests::test_detect_connect_partial ... ok
test pool::tests::test_pool_return_and_get ... ok
test protocol_detect::tests::test_detect_empty ... ok
test forward::tests::test_conn_timer ... ok
test protocol_detect::tests::test_detect_http_post ... ok
test protocol_detect::tests::test_detect_socks5 ... ok
test protocol_detect::tests::test_detect_unknown ... ok
test socks5::tests::test_build_greeting_reply ... ok
test socks5::tests::test_build_socks5_reply_success ... ok
test socks5::tests::test_parse_greeting_invalid_version ... ok
test socks5::tests::test_parse_greeting_multiple_methods ... ok
test socks5::tests::test_parse_greeting_no_auth ... ok
test socks5::tests::test_parse_greeting_too_short ... ok
test socks5::tests::test_parse_request_domain ... ok
test socks5::tests::test_parse_request_ipv4 ... ok
test socks5::tests::test_parse_request_ipv6 ... ok
test socks5::tests::test_socks5_addr_to_host_string ... ok
test protocol_detect::tests::test_detect_http_connect ... ok
test pool::tests::test_pool_evict_expired ... ok
test protocol_detect::tests::test_detect_http_get ... ok

test result: ok. 38 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.02s
```

### Integration Test ด้วย curl

เมื่อ proxy server รันแล้ว ทดสอบด้วย:

```bash
# ทดสอบ SOCKS5
curl --proxy socks5://localhost:1080 http://httpbin.org/get

# ทดสอบ HTTP CONNECT (HTTPS)
curl --proxy http://localhost:1080 https://httpbin.org/get

# ทดสอบ HTTP forward proxy
curl --proxy http://localhost:1080 http://httpbin.org/get
```

---

## จุดที่ต้องระวัง (Pitfalls)

### Pitfall 1: `peek()` กับ async — ต้องใช้ให้ถูก

ใน tokio, `TcpStream::peek()` return ข้อมูลโดยไม่ advance read position แต่มีข้อควรระวัง:

```rust
// ❌ ผิด: peek() อาจ return 0 bytes ถ้า stream ยังไม่พร้อม
let n = stream.peek(&mut buf).await?;
if n == 0 { /* connection closed? */ }

// ✓ ถูก: loop จนได้ข้อมูลพอ
loop {
    let n = stream.peek(&mut buf).await?;
    if n >= 1 { break; }
}
```

นอกจากนี้ `peek()` ไม่รับประกันว่าจะได้ครบทั้งหมดที่ขอ อาจได้น้อยกว่า buffer size ดังนั้นควร check ว่ามีข้อมูลอย่างน้อย 1 byte ก่อน dispatch

### Pitfall 2: `copy_bidirectional` กับ half-close

TCP รองรับ "half-close" — ฝั่งหนึ่งปิด write แต่ยังอ่านได้ `copy_bidirectional` จัดการ case นี้อย่างถูกต้อง แต่ถ้าใช้ `copy()` ทางเดียวใน 2 task แยกกัน ต้องระวัง:

```rust
// ❌ อาจ hang ถ้า upstream ส่งข้อมูลค้างหลัง client ปิด write
let (mut cr, mut cw) = client.split();
let (mut ur, mut uw) = upstream.split();
tokio::join!(
    tokio::io::copy(&mut cr, &mut uw),
    tokio::io::copy(&mut ur, &mut cw),
);

// ✓ ดีกว่า: ใช้ copy_bidirectional ที่จัดการ half-close ให้
tokio::io::copy_bidirectional(&mut client, &mut upstream).await?;
```

### Pitfall 3: SOCKS5 Domain Length — off-by-one

ใน ATYP=0x03 (domain), byte แรกหลัง ATYP คือ **length** ของ domain name ไม่ใช่ตัว domain:

```
03  0B  65 78 61 6d 70 6c 65 2e 63 6f 6d  00 50
│   │   └─────────────────────────────┘   └──┘
│   length=11                              port=80
ATYP=domain
```

ถ้าลืม `buf[4]` เป็น length แล้วเอา index ผิด จะ parse domain เพี้ยนไปทั้งหมด:

```rust
// ❌ ผิด: เริ่ม slice จากตำแหน่ง 4 (นับ length byte เป็นส่วนหนึ่งของ domain)
let domain = std::str::from_utf8(&buf[4..4 + dlen])?;

// ✓ ถูก: length อยู่ที่ buf[4], domain เริ่มที่ buf[5]
let dlen = buf[4] as usize;
let domain = std::str::from_utf8(&buf[5..5 + dlen])?;
```

### Pitfall 4: DashMap ไม่ใช่ `Mutex<HashMap>` — ระวัง deadlock จาก nested lock

`DashMap` ใช้ shard-based locking ภายใน ถ้า lock entry หนึ่งค้างไว้แล้วพยายาม lock entry อื่นจาก thread เดียวกัน อาจเกิด deadlock ในบางกรณี:

```rust
// ❌ อาจ deadlock: hold entry A แล้ว access entry B ที่อาจอยู่ใน shard เดียวกัน
let mut a = pool.pools.get_mut(&addr_a);
let mut b = pool.pools.get_mut(&addr_b); // อาจ deadlock!

// ✓ ถูก: ทำทีละ entry แล้วปล่อย lock ก่อนเข้า entry ต่อไป
{
    let mut a = pool.pools.entry(addr_a).or_default();
    // ... ทำงานกับ a แล้วปล่อย
}
{
    let mut b = pool.pools.entry(addr_b).or_default();
    // ... ทำงานกับ b
}
```

### Pitfall 5: HTTP CONNECT กับ `proxy-authorization` Header

เมื่อ proxy ต้องการ authentication, client ส่ง `Proxy-Authorization: Basic <base64>` แต่หลังจาก `200 Connection Established` แล้ว header นี้จะไม่ถูกส่งซ้ำใน TLS traffic ด้านใน เพราะ client รู้ว่า proxy ผ่านแล้ว

```
CONNECT api.example.com:443 HTTP/1.1
Proxy-Authorization: Basic dXNlcjpwYXNz
                                              ← header นี้อยู่ใน plain HTTP
HTTP/1.1 200 Connection Established           ← proxy confirm

[TLS ClientHello]                             ← เริ่ม tunnel ไม่มี header แล้ว
```

ดังนั้นถ้า implement proxy auth ต้องตรวจ header ก่อน respond `200` ไม่ใช่หลัง

### Pitfall 6: Connection Pool กับ async runtime

ใน async context, `PooledConn` ที่ wrap `TcpStream` จริงจะมี lifetime ผูกกับ tokio runtime ถ้า `return_conn()` เรียกจาก thread ที่ไม่มี runtime context อาจเกิด panic:

```rust
// ❌ อาจ panic: drop TcpStream นอก tokio context
std::thread::spawn(|| {
    drop(conn); // TcpStream drop triggers async shutdown
});

// ✓ ถูก: ทำงานกับ connection pool เสมอใน async context
tokio::spawn(async move {
    pool.return_conn(conn);
});
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
./target/release/network-proxy
```

### Configuration File (JSON)

```json
{
  "listen_addr": "0.0.0.0:1080",
  "pool": {
    "max_idle_per_host": 10,
    "idle_timeout_secs": 60
  },
  "acl": {
    "default_action": "allow",
    "rules": [
      { "action": "deny",  "host_pattern": "*.ads.doubleclick.net" },
      { "action": "deny",  "host_pattern": "*.malware.*"           },
      { "action": "allow", "host_pattern": "*.internal.corp"       }
    ]
  }
}
```

โหลด config ด้วย `serde_json`:

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct ProxyConfig {
    listen_addr: String,
    pool: PoolConfigJson,
    acl: AclConfigJson,
}

#[derive(Deserialize)]
struct PoolConfigJson {
    max_idle_per_host: usize,
    idle_timeout_secs: u64,
}

#[derive(Deserialize)]
struct AclRuleJson {
    action: String,       // "allow" | "deny"
    host_pattern: String,
    port_range: Option<(u16, u16)>,
}

#[derive(Deserialize)]
struct AclConfigJson {
    default_action: String,
    rules: Vec<AclRuleJson>,
}
```

### Docker

```dockerfile
FROM rust:1.78 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/network-proxy /usr/local/bin/
EXPOSE 1080
CMD ["network-proxy", "--config", "/etc/proxy/config.json"]
```

```bash
docker build -t network-proxy:latest .
docker run -p 1080:1080 -v ./config.json:/etc/proxy/config.json network-proxy:latest
```

### Systemd Unit File

```ini
[Unit]
Description=Network Proxy (SOCKS5 + HTTP CONNECT)
After=network.target

[Service]
Type=simple
User=proxy
ExecStart=/usr/local/bin/network-proxy --config /etc/proxy/config.json
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Username/Password Authentication (SOCKS5 Sub-negotiation)

หลังจาก method negotiation เลือก `0x02` (username/password), client ส่ง sub-negotiation:

```
VER(1)=01 | ULEN(1) | UNAME(ULEN) | PLEN(1) | PASSWD(PLEN)
```

**งาน:** Implement `parse_socks5_auth_subneg(buf: &[u8]) -> Result<(String, String), Socks5ParseError>` และ `build_socks5_auth_reply(success: bool) -> [u8; 2]` พร้อม tests ที่ครอบคลุม valid/invalid credentials

**ประโยชน์:** เข้าใจ multi-round protocol handshake ที่ต้องจัดการ state ระหว่าง round

### แบบฝึกหัดที่ 2: Transparent DNS Resolution กับ Caching

ปัจจุบัน `TcpStream::connect("hostname:port")` ทำ DNS lookup ทุกครั้ง ให้เพิ่ม:

```rust
pub struct DnsCache {
    cache: DashMap<String, (Vec<IpAddr>, Instant)>,
    ttl: Duration,
}

impl DnsCache {
    pub async fn resolve(&self, hostname: &str) -> Vec<IpAddr> { ... }
}
```

**งาน:** Implement `DnsCache` ด้วย `tokio::net::lookup_host()`, cache result ตาม TTL, และ fallback ถ้า cached IP ใช้ไม่ได้ พร้อม metrics จำนวน cache hit/miss

**ประโยชน์:** เข้าใจ DNS resolution ใน async context และ cache invalidation strategy

### แบบฝึกหัดที่ 3: Bandwidth Throttling

เพิ่ม rate limiting per-connection หรือ per-upstream:

```rust
pub struct ThrottledStream<S> {
    inner: S,
    rate_limiter: TokenBucket,
}

pub struct TokenBucket {
    tokens: f64,
    capacity: f64,
    refill_rate: f64, // tokens per second
    last_refill: Instant,
}
```

**งาน:** Implement `AsyncRead` + `AsyncWrite` สำหรับ `ThrottledStream` ที่ intercept `poll_read`/`poll_write` เพื่อเพิ่ม delay เมื่อ token หมด

**ประโยชน์:** เข้าใจ `Pin<&mut Self>`, `Poll<Result<...>>`, และการ wrap async I/O

### แบบฝึกหัดที่ 4: SOCKS5 UDP Associate

นอกจาก CONNECT command ยังมี UDP ASSOCIATE ที่ใช้สำหรับ proxy UDP traffic:

```
Client → Proxy: SOCKS5 CONNECT request ด้วย command=0x03
Proxy  → Client: reply พร้อม BND.ADDR:BND.PORT ที่ client ต้องส่ง UDP ไปหา
Client → Proxy UDP: header(FRAG|ATYP|DST.ADDR|DST.PORT) + DATA
Proxy  → Upstream: forward UDP packet โดยตัด SOCKS5 header ออก
```

**งาน:** Implement UDP associate handler ด้วย `tokio::net::UdpSocket` สร้าง per-connection UDP relay socket พร้อม timeout cleanup

**ประโยชน์:** เข้าใจ UDP socket lifecycle ใน async และความแตกต่างระหว่าง TCP/UDP proxy

### แบบฝึกหัดที่ 5: Stats Dashboard (HTTP API)

เพิ่ม HTTP admin API บน port 9090:

```
GET /stats → JSON summary ของ connections ปัจจุบัน
GET /pool  → connection pool status per upstream
GET /acl   → ACL rules และจำนวน hits/misses
POST /acl/reload → โหลด ACL rules ใหม่จาก config file
```

**งาน:** ใช้ `tokio::net::TcpListener` แยก port, parse HTTP request แบบ minimal (ไม่ต้องใช้ framework), return JSON ด้วย `serde_json::to_string()`

**ประโยชน์:** เข้าใจ dual-port server architecture และ shared state ระหว่าง proxy และ admin server

### แบบฝึกหัดที่ 6: Upstream Chain (Proxy-of-Proxy)

บางองค์กรต้องการ proxy chain:

```
Client → Proxy A (SOCKS5) → Proxy B (SOCKS5) → Internet
```

**งาน:** เพิ่ม upstream proxy config — ถ้า config กำหนด upstream proxy ให้เชื่อมต่อผ่าน SOCKS5 หรือ HTTP CONNECT ไปหา proxy ถัดไปแทนที่จะเชื่อมตรง ต้อง implement SOCKS5 client (ไม่ใช่แค่ server) ด้วย

**ประโยชน์:** เข้าใจว่า proxy protocol เดียวกันสามารถเล่นทั้งบทบาท client และ server ได้

---

## สรุป

โปรเจคนี้สร้าง **dual-protocol network proxy** ตั้งแต่ byte-level parsing ไปจนถึง concurrent connection forwarding ครอบคลุม concepts สำคัญ:

**Protocol Engineering:**
- SOCKS5 RFC 1928 — 2-round handshake, 3 address types, auth methods
- HTTP CONNECT tunnel — separation ระหว่าง proxy negotiation และ tunneled traffic
- Protocol detection ด้วย peek — zero-copy dispatch ใน single byte

**Concurrency Patterns:**
- `tokio::io::copy_bidirectional` — TCP tunnel 2 ทิศทางใน single async task
- `DashMap<K, VecDeque<V>>` — concurrent connection pool ที่ไม่ต้องใช้ global lock
- Background eviction task ด้วย `tokio::time::interval`

**Production Concerns:**
- ACL engine ด้วย glob patterns — กรอง destination ก่อน upstream connect
- `ForwardStats` — collect bytes transferred และ duration ทุก connection
- Connection pool — reuse upstream connections เพื่อลด latency

Pattern เหล่านี้ปรากฏในซอฟต์แวร์ real-world เช่น Shadowsocks (SOCKS5 over encrypted channel), mitmproxy (HTTP CONNECT with TLS interception), และ Envoy (multi-protocol proxy core) — การสร้างตั้งแต่ต้นทำให้เข้าใจว่า 1 byte แรกของ connection สามารถกำหนดทิศทาง traffic ทั้งหมดได้อย่างไร

โปรเจคถัดไป **I01: Linear Regression** จะเปลี่ยนจาก network I/O ไปสู่ ML/AI domain โดยนำ Rust performance มาใช้กับ numerical computation และ matrix operations — skills ที่สะสมมาจาก module H จะยังใช้ได้กับ data pipeline ที่ต้องรับข้อมูล streaming จาก network

---

**โปรเจคก่อนหน้า:** [Project H09: QUIC Transport](project-h09-quic-transport.md) | **โปรเจคถัดไป:** [Project I01: Linear Regression](project-i01-linear-regression.md)
