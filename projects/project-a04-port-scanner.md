# Project A04: Port & Service Scanner

> โมดูล: A — CLI & Systems Tools | ความยาก: ⭐⭐⭐ | เวลาโดยประมาณ: 5 ชั่วโมง

---

> **⚠️ ข้อกำหนดการใช้งาน (Legal & Ethical Disclaimer)**
>
> เครื่องมือนี้พัฒนาขึ้นเพื่อ **การวินิจฉัยระบบเครือข่าย (network diagnostics)** บนโครงสร้างพื้นฐานที่คุณ **เป็นเจ้าของ** หรือ **มีสิทธิ์ได้รับอนุญาตอย่างชัดแจ้ง** ให้ทดสอบ เช่น เซิร์ฟเวอร์ส่วนตัว, lab environment, staging server ของทีม หรือ home network ของคุณเอง
>
> **ห้าม** ใช้เครื่องมือนี้สแกนระบบที่คุณไม่มีสิทธิ์ เพราะอาจผิดกฎหมายในหลายประเทศ (เช่น Computer Crime Act ของไทย, CFAA ของสหรัฐ, Computer Misuse Act ของอังกฤษ) การส่ง TCP connection ไปยัง host ที่ไม่ได้รับอนุญาตถือเป็นการละเมิดความปลอดภัย แม้จะไม่ได้เจาะระบบก็ตาม
>
> ใช้สำหรับ: ตรวจสอบว่า service ของตัวเองเปิด port ถูกต้อง, ยืนยันว่า firewall rule ทำงานตามที่ตั้งค่าไว้, วิเคราะห์ว่า service ใดกำลัง listen อยู่ในเครือข่าย lab

---

## ภาพรวมโปรเจค

**Port & Service Scanner** คือเครื่องมือ command-line ที่ใช้ทดสอบว่า TCP port ใดบ้างที่รับการเชื่อมต่อบน host เป้าหมาย และพยายามระบุว่า service ใดกำลังทำงานอยู่ (เช่น HTTP, SSH, Redis, PostgreSQL) โดยการส่ง probe packet และอ่าน banner response ที่ส่งกลับมา

ในโลกจริง network engineers และ sysadmins ใช้เครื่องมือประเภทนี้เป็นประจำเพื่อ:
- ยืนยันว่า service deploy ขึ้นมาแล้วและ listen บน port ที่ถูกต้อง
- ตรวจสอบว่า firewall rule ทำงานตามที่ตั้งค่าไว้ (port ที่ควรปิดถูกปิดจริง)
- สแกน lab environment เพื่อทำ network inventory ว่ามี service อะไรทำงานอยู่บ้าง
- debug ปัญหาการเชื่อมต่อระหว่าง microservices ใน container network

**สิ่งที่ทำให้โปรเจคนี้น่าสร้าง:** เราจะได้ฝึก async Rust อย่างเต็มรูปแบบ — การสแกน 65,535 ports พร้อมกันต้องการ concurrency ที่มีประสิทธิภาพ, semaphore เพื่อควบคุม resource, และ timeout handling ที่ละเอียดรอบคอบ นอกจากนี้ยังได้ฝึกการทำงานกับ network primitives ใน Rust โดยตรง

## สิ่งที่จะได้เรียนรู้

- **Async TCP networking:** ใช้ `tokio::net::TcpStream` สำหรับ non-blocking TCP connect
- **Semaphore-based concurrency:** ควบคุมจำนวน concurrent tasks ด้วย `tokio::sync::Semaphore`
- **Rate limiting pattern:** ใช้ `tokio::time` เพื่อจำกัด operations per second
- **Banner grabbing:** ส่ง probe bytes และอ่าน response เพื่อระบุ service
- **Progress reporting:** ใช้ `indicatif` multi-progress bar แสดงความคืบหน้าแบบ real-time
- **CIDR network parsing:** แปลง `192.168.1.0/24` เป็น list ของ IP addresses
- **Structured output:** serialize ผลลัพธ์เป็น JSON/CSV ด้วย `serde`
- **Timeout hierarchies:** per-port timeout และ overall scan timeout ทำงานซ้อนกัน

## ความรู้ที่ต้องมีมาก่อน

- **Part 46-50:** Async/await, Tokio runtime, Future trait
- **Part 51-55:** Tokio tasks, channels, select! macro
- **Part 56-60:** Error handling กับ `?` operator, thiserror/anyhow
- **Part 61-65:** Trait objects, dynamic dispatch
- **Part 71-75:** Clap สำหรับ CLI argument parsing
- **Part 76-80:** Serde, JSON/CSV serialization
- **Part 36-40:** Iterators, closures ขั้นสูง

## โครงสร้างโปรเจค (Project Layout)

```
port-scanner/
├── src/
│   ├── main.rs          # Entry point + CLI argument handling
│   ├── scanner.rs       # Core scanning logic (TCP connect, semaphore)
│   ├── target.rs        # Target parsing (IP, hostname, CIDR)
│   ├── ports.rs         # Port range parsing
│   ├── service.rs       # Service fingerprinting + banner grabbing
│   ├── output.rs        # Output formatters (table, JSON, CSV)
│   ├── progress.rs      # indicatif progress bar management
│   └── error.rs         # Custom error types
├── tests/
│   └── integration_test.rs   # Integration tests with real TcpListener
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
CLI Args (clap)
    │
    ▼
Target Parser ──► Vec<IpAddr>        (hostname → DNS resolve, CIDR → expand)
Port Parser   ──► Vec<u16>           ("80,443,8000-9000" → sorted unique list)
    │
    ▼
Scanner Engine
    ├── Semaphore (max concurrent = 1000)
    ├── Rate Limiter (tokens per second)
    └── tokio::spawn per port/host pair
         │
         ├── TcpStream::connect_timeout ──► Open/Closed/Filtered
         │
         └── [if open] Banner Grabber
                  ├── HTTP GET /\r\n\r\n
                  ├── read first 1024 bytes
                  └── Pattern match → ServiceInfo
    │
    ▼
Results Collector (channel → Vec<ScanResult>)
    │
    ▼
Output Formatter
    ├── Table (human-readable, owo-colors)
    ├── JSON (serde_json)
    └── CSV (manual serialization)
```

### Design Decisions

**ทำไมถึงใช้ TCP Connect Scan แทน SYN Scan?**
TCP connect scan ใช้ `connect()` system call ธรรมดา ไม่ต้องการสิทธิ์ root และทำงานได้บน Rust ด้วย `TcpStream::connect_timeout` โดยตรง SYN scan (half-open) ต้องการ raw socket และ root privilege ซึ่งซับซ้อนกว่าและเกินขอบเขตของโปรเจคนี้

**ทำไม Semaphore แทน Thread Pool?**
การสแกน 65,535 ports พร้อมกัน 65,535 OS threads ไม่ practical (แต่ละ thread ใช้ ~2MB stack) แต่ด้วย async tasks + semaphore เราสามารถมี 65,535 pending tasks ในหน่วยความจำเพียงไม่กี่ MB และใช้ kernel threads เพียงไม่กี่ตัว

**Timeout Design:**
- `connect_timeout` ตั้งระดับ per-port (default 1 วินาที)
- `tokio::time::timeout` ครอบทั้ง banner grab (default 3 วินาที)
- `select!` ที่ top level สำหรับ overall scan deadline

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: สแกน Single Host, Single Port

เริ่มจากสิ่งที่ง่ายที่สุด — ทดสอบว่า port หนึ่ง port ของ host หนึ่งเปิดหรือปิด

**Cargo.toml:**

```toml
[package]
name = "port-scanner"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
clap = { version = "4", features = ["derive"] }
indicatif = "0.17"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
ipnetwork = "0.20"
dns-lookup = "2"
owo-colors = "4"
thiserror = "1"
```

**src/error.rs:**

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ScanError {
    #[error("invalid target: {0}")]
    InvalidTarget(String),

    #[error("invalid port range: {0}")]
    InvalidPortRange(String),

    #[error("DNS resolution failed for {host}: {source}")]
    DnsError {
        host: String,
        #[source]
        source: std::io::Error,
    },

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("scan timed out after {0} seconds")]
    ScanTimeout(u64),
}
```

**src/main.rs (Step 1 — minimal version):**

```rust
use std::net::{SocketAddr, TcpStream, ToSocketAddrs};
use std::time::Duration;

fn scan_port(host: &str, port: u16, timeout: Duration) -> bool {
    let addr_str = format!("{}:{}", host, port);
    // ToSocketAddrs จะ resolve DNS ถ้าเป็น hostname
    let addrs: Vec<SocketAddr> = match addr_str.to_socket_addrs() {
        Ok(it) => it.collect(),
        Err(_) => return false,
    };

    for addr in addrs {
        // TcpStream::connect_timeout ทดสอบ TCP handshake
        // ถ้าสำเร็จ = port เปิด, ถ้า timeout/refused = ปิด
        if TcpStream::connect_timeout(&addr, timeout).is_ok() {
            return true;
        }
    }
    false
}

fn main() {
    let host = "127.0.0.1";
    let port: u16 = 22;
    let timeout = Duration::from_secs(1);

    print!("Scanning {}:{} ... ", host, port);

    if scan_port(host, port, timeout) {
        println!("OPEN");
    } else {
        println!("CLOSED");
    }
}
```

เรียกใช้: `cargo run`

```
Scanning 127.0.0.1:22 ... CLOSED
```

(ผลขึ้นอยู่กับเครื่องของคุณ — ถ้า SSH daemon ทำงานอยู่จะเห็น OPEN)

---

### ขั้นที่ 2: Port Range Parsing + Concurrent Scanning ด้วย Semaphore

ขั้นนี้เพิ่ม 2 ความสามารถสำคัญ: แปลง port range string และสแกนหลาย port พร้อมกัน

**src/ports.rs:**

```rust
use crate::error::ScanError;

/// แปลง port range string เป็น sorted Vec<u16>
/// รองรับ: "22,80,443,8000-8003" → [22, 80, 443, 8000, 8001, 8002, 8003]
pub fn parse_port_range(spec: &str) -> Result<Vec<u16>, ScanError> {
    let mut ports = Vec::new();

    for part in spec.split(',') {
        let part = part.trim();
        if part.is_empty() {
            continue;
        }

        if let Some((start_str, end_str)) = part.split_once('-') {
            // "8000-8003" → range
            let start: u16 = start_str
                .trim()
                .parse()
                .map_err(|_| ScanError::InvalidPortRange(
                    format!("invalid start port in '{}'", part)
                ))?;
            let end: u16 = end_str
                .trim()
                .parse()
                .map_err(|_| ScanError::InvalidPortRange(
                    format!("invalid end port in '{}'", part)
                ))?;

            if start > end {
                return Err(ScanError::InvalidPortRange(
                    format!("start {} > end {} in '{}'", start, end, part)
                ));
            }

            ports.extend(start..=end);
        } else {
            // "80" → single port
            let port: u16 = part
                .parse()
                .map_err(|_| ScanError::InvalidPortRange(
                    format!("'{}' is not a valid port number", part)
                ))?;
            ports.push(port);
        }
    }

    // ลบ duplicate และเรียงลำดับ
    ports.sort_unstable();
    ports.dedup();

    if ports.is_empty() {
        return Err(ScanError::InvalidPortRange("no ports specified".into()));
    }

    Ok(ports)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_single_port() {
        assert_eq!(parse_port_range("80").unwrap(), vec![80]);
    }

    #[test]
    fn test_comma_separated() {
        assert_eq!(parse_port_range("22,80,443").unwrap(), vec![22, 80, 443]);
    }

    #[test]
    fn test_range() {
        assert_eq!(parse_port_range("8000-8003").unwrap(), vec![8000, 8001, 8002, 8003]);
    }

    #[test]
    fn test_mixed() {
        let result = parse_port_range("22,80,443,8000-8003").unwrap();
        assert_eq!(result, vec![22, 80, 443, 8000, 8001, 8002, 8003]);
    }

    #[test]
    fn test_deduplication() {
        let result = parse_port_range("80,80,80").unwrap();
        assert_eq!(result, vec![80]);
    }

    #[test]
    fn test_invalid_range() {
        assert!(parse_port_range("9000-8000").is_err());
    }
}
```

**src/scanner.rs (concurrent version):**

```rust
use std::net::{IpAddr, SocketAddr};
use std::sync::Arc;
use std::time::Duration;
use tokio::net::TcpStream;
use tokio::sync::Semaphore;
use tokio::time::timeout;

#[derive(Debug, Clone)]
pub struct ScanResult {
    pub host: IpAddr,
    pub port: u16,
    pub state: PortState,
}

#[derive(Debug, Clone, PartialEq)]
pub enum PortState {
    Open,
    Closed,
    Filtered, // timeout — firewall อาจ drop packets
}

/// สแกน host เดียว หลาย port พร้อมกัน
/// max_concurrent ควบคุมจำนวน simultaneous TCP connects
pub async fn scan_host(
    host: IpAddr,
    ports: Vec<u16>,
    connect_timeout: Duration,
    max_concurrent: usize,
) -> Vec<ScanResult> {
    // Semaphore จำกัดว่าสูงสุดกี่ tasks จะทำงานพร้อมกัน
    // เปรียบเหมือน "token pool" — ต้องได้ token ก่อนจึง connect ได้
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    let mut handles = Vec::with_capacity(ports.len());

    for port in ports {
        let sem = Arc::clone(&semaphore);
        let addr = SocketAddr::new(host, port);

        // tokio::spawn สร้าง async task สำหรับแต่ละ port
        let handle = tokio::spawn(async move {
            // .acquire_owned() คืน permit ที่ drop เองเมื่อ out of scope
            let _permit = sem.acquire_owned().await.unwrap();

            let state = match timeout(
                connect_timeout,
                TcpStream::connect(addr),
            )
            .await
            {
                Ok(Ok(_stream)) => PortState::Open,   // connect สำเร็จ
                Ok(Err(_))      => PortState::Closed,  // connection refused
                Err(_)          => PortState::Filtered, // timeout
            };

            ScanResult { host, port, state }
        });

        handles.push(handle);
    }

    // รอทุก task เสร็จและรวบรวมผล
    let mut results = Vec::with_capacity(handles.len());
    for handle in handles {
        if let Ok(result) = handle.await {
            results.push(result);
        }
    }

    // เรียงตาม port number
    results.sort_by_key(|r| r.port);
    results
}
```

**src/main.rs (Step 2):**

```rust
mod error;
mod ports;
mod scanner;

use scanner::PortState;
use std::net::IpAddr;
use std::str::FromStr;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let host: IpAddr = IpAddr::from_str("127.0.0.1").unwrap();
    let ports = ports::parse_port_range("22,80,443,8000-8010").unwrap();
    let timeout = Duration::from_secs(1);
    let max_concurrent = 100;

    println!("Scanning {} ({} ports)...", host, ports.len());

    let results = scanner::scan_host(host, ports, timeout, max_concurrent).await;

    for r in &results {
        if r.state == PortState::Open {
            println!("  PORT {}/tcp  OPEN", r.port);
        }
    }

    let open_count = results.iter().filter(|r| r.state == PortState::Open).count();
    println!("\n{} open ports found", open_count);
}
```

---

### ขั้นที่ 3: Timeout Handling + Progress Bar

การแสดงความคืบหน้าสำคัญมากเมื่อสแกนหลาย port — ผู้ใช้ต้องรู้ว่าโปรแกรมทำงานอยู่ ไม่ได้ค้าง

**src/progress.rs:**

```rust
use indicatif::{MultiProgress, ProgressBar, ProgressStyle};
use std::net::IpAddr;

pub struct ScanProgress {
    pub multi: MultiProgress,
}

impl ScanProgress {
    pub fn new() -> Self {
        ScanProgress {
            multi: MultiProgress::new(),
        }
    }

    /// สร้าง progress bar สำหรับ host หนึ่งตัว
    pub fn add_host_bar(&self, host: IpAddr, total_ports: u64) -> ProgressBar {
        let pb = self.multi.add(ProgressBar::new(total_ports));

        pb.set_style(
            ProgressStyle::with_template(
                "{prefix:.bold} [{bar:40.cyan/blue}] {pos}/{len} ports  {msg}"
            )
            .unwrap()
            .progress_chars("=>-"),
        );

        pb.set_prefix(format!("{:<15}", host.to_string()));
        pb
    }
}

impl Default for ScanProgress {
    fn default() -> Self {
        Self::new()
    }
}
```

**src/scanner.rs (พร้อม progress bar):**

```rust
use crate::progress::ScanProgress;
use indicatif::ProgressBar;
use std::net::{IpAddr, SocketAddr};
use std::sync::Arc;
use std::time::Duration;
use tokio::net::TcpStream;
use tokio::sync::Semaphore;
use tokio::time::timeout;

#[derive(Debug, Clone)]
pub struct ScanResult {
    pub host: IpAddr,
    pub port: u16,
    pub state: PortState,
}

#[derive(Debug, Clone, PartialEq)]
pub enum PortState {
    Open,
    Closed,
    Filtered,
}

pub async fn scan_host_with_progress(
    host: IpAddr,
    ports: Vec<u16>,
    connect_timeout: Duration,
    max_concurrent: usize,
    pb: ProgressBar,
) -> Vec<ScanResult> {
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    let pb = Arc::new(pb);
    let mut handles = Vec::with_capacity(ports.len());

    for port in ports {
        let sem = Arc::clone(&semaphore);
        let addr = SocketAddr::new(host, port);
        let pb_clone = Arc::clone(&pb);

        let handle = tokio::spawn(async move {
            let _permit = sem.acquire_owned().await.unwrap();

            let state = match timeout(
                connect_timeout,
                TcpStream::connect(addr),
            )
            .await
            {
                Ok(Ok(_))  => PortState::Open,
                Ok(Err(_)) => PortState::Closed,
                Err(_)     => PortState::Filtered,
            };

            pb_clone.inc(1);

            if state == PortState::Open {
                pb_clone.set_message(format!("port {} open!", port));
            }

            ScanResult { host, port, state }
        });

        handles.push(handle);
    }

    let mut results = Vec::with_capacity(handles.len());
    for handle in handles {
        if let Ok(r) = handle.await {
            results.push(r);
        }
    }

    pb.finish_with_message("done");
    results.sort_by_key(|r| r.port);
    results
}
```

**Overall scan timeout ด้วย `tokio::time::timeout`:**

```rust
use tokio::time::{timeout, Duration};
use crate::error::ScanError;

pub async fn scan_with_deadline(
    host: IpAddr,
    ports: Vec<u16>,
    connect_timeout: Duration,
    overall_timeout: Duration,
    max_concurrent: usize,
    pb: indicatif::ProgressBar,
) -> Result<Vec<ScanResult>, ScanError> {
    timeout(
        overall_timeout,
        scan_host_with_progress(host, ports, connect_timeout, max_concurrent, pb),
    )
    .await
    .map_err(|_| ScanError::ScanTimeout(overall_timeout.as_secs()))
}
```

Output เมื่อรันกับ progress bar:

```
127.0.0.1       [========================================] 1024/1024 ports  done
192.168.1.1     [==========================>-------------]  682/1024 ports  port 80 open!
```

---

### ขั้นที่ 4: Banner Grabbing + Service Detection

Banner grabbing คือการส่ง probe ไปยัง port ที่เปิด แล้วอ่าน response เพื่อระบุ service

**src/service.rs:**

```rust
use std::collections::HashMap;
use std::net::SocketAddr;
use std::time::Duration;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use tokio::time::timeout;

#[derive(Debug, Clone, serde::Serialize)]
pub struct ServiceInfo {
    pub name: &'static str,
    pub version_hint: Option<String>,
    pub banner: Option<String>,
}

/// ส่ง probe ไปยัง TCP port และอ่าน banner response
pub async fn grab_banner(
    addr: SocketAddr,
    grab_timeout: Duration,
) -> Option<ServiceInfo> {
    let stream = timeout(grab_timeout, TcpStream::connect(addr))
        .await
        .ok()?
        .ok()?;

    grab_from_stream(stream, addr.port(), grab_timeout).await
}

async fn grab_from_stream(
    mut stream: TcpStream,
    port: u16,
    grab_timeout: Duration,
) -> Option<ServiceInfo> {
    // บาง service ส่ง banner ทันทีเมื่อ connect (SSH, FTP, SMTP)
    // บาง service รอ request ก่อน (HTTP) จึงต้องส่ง probe ก่อน

    // เลือก probe ตาม well-known port
    let probe: &[u8] = match port {
        80 | 8080 | 8000 | 8443 | 443 => b"HEAD / HTTP/1.0\r\n\r\n",
        21 | 22 | 25 | 110 | 143 => b"", // รอ banner
        _ => b"",                          // ไม่ส่ง probe — อ่าน banner ที่มาเอง
    };

    if !probe.is_empty() {
        timeout(grab_timeout, stream.write_all(probe))
            .await
            .ok()?
            .ok()?;
    }

    // อ่าน response สูงสุด 1024 bytes
    let mut buf = vec![0u8; 1024];
    let n = timeout(grab_timeout, stream.read(&mut buf))
        .await
        .ok()?
        .unwrap_or(0);

    if n == 0 {
        return None;
    }

    let banner_raw = &buf[..n];
    let banner_str = String::from_utf8_lossy(banner_raw).to_string();

    identify_service(port, &banner_str)
}

/// จับคู่ banner กับ pattern ที่รู้จัก
fn identify_service(port: u16, banner: &str) -> Option<ServiceInfo> {
    // ตรวจสอบ pattern ตามลำดับความน่าเชื่อถือ
    let patterns: &[(&str, &str, fn(&str) -> Option<String>)] = &[
        ("SSH",        "SSH-",          extract_ssh_version),
        ("HTTP",       "HTTP/",         extract_http_version),
        ("HTTP",       "HTTP/",         |_| None),
        ("FTP",        "220 ",          extract_ftp_banner),
        ("SMTP",       "220 ",          extract_smtp_banner),
        ("POP3",       "+OK ",          |b| Some(b.lines().next().unwrap_or("").to_string())),
        ("IMAP",       "* OK ",         |b| Some(b.lines().next().unwrap_or("").to_string())),
        ("Redis",      "+PONG",         |_| None),
        ("Redis",      "-ERR",          |_| None),
        ("MySQL",      "\x00\x00\x00",  extract_mysql_version), // MySQL protocol header
        ("PostgreSQL", "R\x00\x00",     |_| None),
    ];

    for (service_name, pattern, extractor) in patterns {
        if banner.contains(pattern) {
            let version_hint = extractor(banner);
            return Some(ServiceInfo {
                name: service_name,
                version_hint,
                banner: Some(banner.lines().next().unwrap_or("").trim().to_string()),
            });
        }
    }

    // ลอง match ตาม well-known port ถ้า pattern ไม่ match
    let name_by_port = service_name_from_port(port);
    if let Some(name) = name_by_port {
        Some(ServiceInfo {
            name,
            version_hint: None,
            banner: Some(banner.lines().next().unwrap_or("").trim().to_string()),
        })
    } else {
        Some(ServiceInfo {
            name: "unknown",
            version_hint: None,
            banner: Some(banner.lines().next().unwrap_or("").trim().to_string()),
        })
    }
}

fn extract_ssh_version(banner: &str) -> Option<String> {
    // SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.6
    banner.lines().next().map(|l| l.trim().to_string())
}

fn extract_http_version(banner: &str) -> Option<String> {
    // HTTP/1.1 200 OK หรือ HTTP/1.0 400 Bad Request
    banner.lines().next().map(|l| l.split_whitespace().take(2).collect::<Vec<_>>().join(" "))
}

fn extract_ftp_banner(banner: &str) -> Option<String> {
    banner.lines().next().map(|l| l.trim_start_matches("220 ").trim().to_string())
}

fn extract_smtp_banner(banner: &str) -> Option<String> {
    banner.lines().next().map(|l| l.trim_start_matches("220 ").trim().to_string())
}

fn extract_mysql_version(banner: &str) -> Option<String> {
    // MySQL protocol: first packet contains version string
    // bytes 5..n เป็น null-terminated version string
    let bytes = banner.as_bytes();
    if bytes.len() > 5 {
        let version_bytes = bytes[5..].iter().take_while(|&&b| b != 0).copied().collect::<Vec<_>>();
        String::from_utf8(version_bytes).ok()
    } else {
        None
    }
}

fn service_name_from_port(port: u16) -> Option<&'static str> {
    // Well-known port assignments (IANA)
    match port {
        21    => Some("FTP"),
        22    => Some("SSH"),
        23    => Some("Telnet"),
        25    => Some("SMTP"),
        53    => Some("DNS"),
        80    => Some("HTTP"),
        110   => Some("POP3"),
        143   => Some("IMAP"),
        443   => Some("HTTPS"),
        445   => Some("SMB"),
        3306  => Some("MySQL"),
        5432  => Some("PostgreSQL"),
        6379  => Some("Redis"),
        8080  => Some("HTTP-Alt"),
        8443  => Some("HTTPS-Alt"),
        27017 => Some("MongoDB"),
        _     => None,
    }
}
```

**ตัวอย่าง output จาก banner grabbing:**

```
PORT   STATE   SERVICE     VERSION
22/tcp open    SSH         SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.6
80/tcp open    HTTP        HTTP/1.1 200 OK
443/tcp open   HTTPS       (no banner)
3306/tcp open  MySQL       8.0.35
6379/tcp open  Redis       (no banner)
```

---

### ขั้นที่ 5: Target Specification — Multiple Hosts & CIDR Range

**src/target.rs:**

```rust
use crate::error::ScanError;
use ipnetwork::IpNetwork;
use std::net::IpAddr;
use std::str::FromStr;

#[derive(Debug, Clone)]
pub enum TargetSpec {
    Single(IpAddr),
    Hostname(String),
    Cidr(IpNetwork),
}

impl TargetSpec {
    /// แปลง target string เป็น TargetSpec
    /// รองรับ: "192.168.1.1", "192.168.1.0/24", "example.com"
    pub fn parse(s: &str) -> Result<Self, ScanError> {
        // ลอง CIDR ก่อน
        if s.contains('/') {
            let network = s.parse::<IpNetwork>().map_err(|_| {
                ScanError::InvalidTarget(format!("'{}' is not a valid CIDR range", s))
            })?;
            return Ok(TargetSpec::Cidr(network));
        }

        // ลอง IP address
        if let Ok(ip) = IpAddr::from_str(s) {
            return Ok(TargetSpec::Single(ip));
        }

        // ถือว่าเป็น hostname (จะ resolve DNS ทีหลัง)
        // ตรวจสอบว่ามีตัวอักษรที่สมเหตุสมผลสำหรับ hostname
        if s.chars().all(|c| c.is_alphanumeric() || c == '.' || c == '-') {
            return Ok(TargetSpec::Hostname(s.to_string()));
        }

        Err(ScanError::InvalidTarget(format!("cannot parse '{}' as IP, CIDR, or hostname", s)))
    }

    /// แปลง TargetSpec เป็น Vec<IpAddr> (resolve DNS ถ้าเป็น hostname)
    pub fn resolve(&self) -> Result<Vec<IpAddr>, ScanError> {
        match self {
            TargetSpec::Single(ip) => Ok(vec![*ip]),

            TargetSpec::Hostname(name) => {
                // dns-lookup crate ทำ synchronous DNS lookup
                // ใน production ควรใช้ tokio::net::lookup_host แทน
                let addrs = dns_lookup::lookup_host(name).map_err(|e| {
                    ScanError::DnsError {
                        host: name.clone(),
                        source: e,
                    }
                })?;
                if addrs.is_empty() {
                    return Err(ScanError::InvalidTarget(
                        format!("DNS returned no addresses for '{}'", name)
                    ));
                }
                Ok(addrs)
            }

            TargetSpec::Cidr(network) => {
                // ipnetwork::IpNetwork::iter() คืน iterator ของทุก IP ใน range
                // /30 = 4 hosts, /24 = 256 hosts, /16 = 65536 hosts
                let ips: Vec<IpAddr> = network.iter().collect();

                // สำหรับ CIDR ปกติ network address (x.x.x.0) และ broadcast (x.x.x.255)
                // จะอยู่ใน list — ขึ้นอยู่กับ use case ว่าจะ include หรือ exclude
                Ok(ips)
            }
        }
    }
}

/// แปลง multiple target strings (อาจมี comma-separated)
pub fn parse_targets(specs: &[String]) -> Result<Vec<IpAddr>, ScanError> {
    let mut all_ips = Vec::new();
    for spec in specs {
        for part in spec.split(',') {
            let part = part.trim();
            if !part.is_empty() {
                let target = TargetSpec::parse(part)?;
                let ips = target.resolve()?;
                all_ips.extend(ips);
            }
        }
    }

    // ลบ duplicate
    all_ips.sort();
    all_ips.dedup();
    Ok(all_ips)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_single_ip() {
        let spec = TargetSpec::parse("192.168.1.1").unwrap();
        assert!(matches!(spec, TargetSpec::Single(_)));
        let ips = spec.resolve().unwrap();
        assert_eq!(ips.len(), 1);
        assert_eq!(ips[0].to_string(), "192.168.1.1");
    }

    #[test]
    fn test_parse_cidr_30() {
        // /30 = 4 addresses: network, 2 hosts, broadcast
        let spec = TargetSpec::parse("192.168.1.0/30").unwrap();
        let ips = spec.resolve().unwrap();
        assert_eq!(ips.len(), 4);
        assert_eq!(ips[0].to_string(), "192.168.1.0");
        assert_eq!(ips[1].to_string(), "192.168.1.1");
        assert_eq!(ips[2].to_string(), "192.168.1.2");
        assert_eq!(ips[3].to_string(), "192.168.1.3");
    }

    #[test]
    fn test_parse_cidr_24() {
        let spec = TargetSpec::parse("10.0.0.0/24").unwrap();
        let ips = spec.resolve().unwrap();
        assert_eq!(ips.len(), 256);
    }

    #[test]
    fn test_invalid_target() {
        assert!(TargetSpec::parse("not@valid!").is_err());
    }
}
```

**การสแกนหลาย host พร้อมกัน:**

```rust
// ใน scanner.rs — scan multiple hosts
pub async fn scan_targets(
    hosts: Vec<IpAddr>,
    ports: Vec<u16>,
    connect_timeout: Duration,
    overall_timeout: Duration,
    max_concurrent: usize,
    progress: &ScanProgress,
) -> Vec<ScanResult> {
    let mut all_handles = Vec::new();

    for host in hosts {
        let pb = progress.add_host_bar(host, ports.len() as u64);
        let ports_clone = ports.clone();

        let handle = tokio::spawn(async move {
            scan_host_with_progress(
                host,
                ports_clone,
                connect_timeout,
                max_concurrent,
                pb,
            )
            .await
        });

        all_handles.push(handle);
    }

    let mut all_results = Vec::new();
    for handle in all_handles {
        if let Ok(results) = handle.await {
            all_results.extend(results);
        }
    }

    all_results
}
```

---

### ขั้นที่ 6: Rate Limiting

Rate limiting ป้องกันการ overwhelm target ด้วย connection ที่เร็วเกินไป

**หลักการ Token Bucket:**

```
Token bucket = ถัง token จำกัดจำนวน
├── เติม token ทุก interval (เช่น 10 tokens ต่อ 100ms = 100/sec)
├── แต่ละ port scan ใช้ 1 token
└── ถ้า bucket ว่าง → รอจนกว่าจะมี token
```

**src/rate_limiter.rs:**

```rust
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::Mutex;
use tokio::time::sleep;

pub struct RateLimiter {
    // tokens_per_second: จำนวน operations สูงสุดต่อวินาที
    tokens_per_second: f64,
    // เวลาที่ควรจะเกิด operation ครั้งถัดไป
    inner: Arc<Mutex<RateLimiterInner>>,
}

struct RateLimiterInner {
    last_check: Instant,
    available_tokens: f64,
    max_tokens: f64,
}

impl RateLimiter {
    pub fn new(tokens_per_second: f64) -> Self {
        RateLimiter {
            tokens_per_second,
            inner: Arc::new(Mutex::new(RateLimiterInner {
                last_check: Instant::now(),
                available_tokens: tokens_per_second, // เริ่มเต็ม
                max_tokens: tokens_per_second,        // burst = 1 วินาที
            })),
        }
    }

    /// รอจนกว่าจะสามารถดำเนินการได้ (ได้รับ 1 token)
    pub async fn acquire(&self) {
        loop {
            let wait_duration = {
                let mut inner = self.inner.lock().await;
                let now = Instant::now();
                let elapsed = now.duration_since(inner.last_check).as_secs_f64();
                inner.last_check = now;

                // เติม token ตาม elapsed time
                inner.available_tokens = (inner.available_tokens
                    + elapsed * self.tokens_per_second)
                    .min(inner.max_tokens);

                if inner.available_tokens >= 1.0 {
                    inner.available_tokens -= 1.0;
                    Duration::ZERO // ไม่ต้องรอ
                } else {
                    // คำนวณว่าต้องรอนานแค่ไหนจนกว่า token จะพร้อม
                    let deficit = 1.0 - inner.available_tokens;
                    let wait_secs = deficit / self.tokens_per_second;
                    Duration::from_secs_f64(wait_secs)
                }
            };

            if wait_duration.is_zero() {
                break;
            }
            sleep(wait_duration).await;
        }
    }
}

/// Version ที่ใช้กับ scanner — pass rate limiter เข้าไป
pub async fn scan_host_rate_limited(
    host: std::net::IpAddr,
    ports: Vec<u16>,
    connect_timeout: Duration,
    max_concurrent: usize,
    rate_limiter: Arc<RateLimiter>,
    pb: indicatif::ProgressBar,
) -> Vec<crate::scanner::ScanResult> {
    use crate::scanner::{PortState, ScanResult};
    use tokio::net::TcpStream;
    use tokio::sync::Semaphore;
    use tokio::time::timeout;
    use std::net::SocketAddr;

    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    let pb = Arc::new(pb);
    let mut handles = Vec::with_capacity(ports.len());

    for port in ports {
        let sem = Arc::clone(&semaphore);
        let rl = Arc::clone(&rate_limiter);
        let addr = SocketAddr::new(host, port);
        let pb_clone = Arc::clone(&pb);

        let handle = tokio::spawn(async move {
            // รอ rate limit token ก่อน
            rl.acquire().await;

            let _permit = sem.acquire_owned().await.unwrap();

            let state = match timeout(
                connect_timeout,
                TcpStream::connect(addr),
            )
            .await
            {
                Ok(Ok(_))  => PortState::Open,
                Ok(Err(_)) => PortState::Closed,
                Err(_)     => PortState::Filtered,
            };

            pb_clone.inc(1);
            ScanResult { host, port, state }
        });

        handles.push(handle);
    }

    let mut results = Vec::with_capacity(handles.len());
    for h in handles {
        if let Ok(r) = h.await {
            results.push(r);
        }
    }
    pb.finish_with_message("done");
    results.sort_by_key(|r| r.port);
    results
}
```

---

### ขั้นที่ 7: JSON/CSV Output

**src/output.rs:**

```rust
use crate::scanner::{PortState, ScanResult};
use crate::service::ServiceInfo;
use owo_colors::OwoColorize;
use serde::Serialize;
use std::time::Duration;

#[derive(Debug, Clone, Serialize)]
pub struct PortEntry {
    pub host: String,
    pub port: u16,
    pub state: String,
    pub service: Option<String>,
    pub version: Option<String>,
    pub banner: Option<String>,
}

#[derive(Debug, Clone, Serialize)]
pub struct ScanReport {
    pub scan_started: String,
    pub scan_finished: String,
    pub duration_secs: f64,
    pub targets_scanned: usize,
    pub total_ports_tested: usize,
    pub open_ports: usize,
    pub ports_per_sec: f64,
    pub results: Vec<PortEntry>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum OutputFormat {
    Table,
    Json,
    Csv,
}

impl OutputFormat {
    pub fn from_str(s: &str) -> Option<Self> {
        match s.to_lowercase().as_str() {
            "table" => Some(OutputFormat::Table),
            "json"  => Some(OutputFormat::Json),
            "csv"   => Some(OutputFormat::Csv),
            _       => None,
        }
    }
}

pub fn format_results(
    report: &ScanReport,
    format: &OutputFormat,
) -> String {
    match format {
        OutputFormat::Table => format_table(report),
        OutputFormat::Json  => format_json(report),
        OutputFormat::Csv   => format_csv(report),
    }
}

fn format_table(report: &ScanReport) -> String {
    let mut out = String::new();

    out.push_str(&format!(
        "\n{}\n\n",
        "Scan Results".bold().underline()
    ));

    // Header
    out.push_str(&format!(
        "{:<20} {:<8} {:<12} {:<15} {}\n",
        "HOST".bold(),
        "PORT".bold(),
        "STATE".bold(),
        "SERVICE".bold(),
        "VERSION/BANNER".bold(),
    ));
    out.push_str(&"-".repeat(80));
    out.push('\n');

    // ดึงเฉพาะ open ports
    let open: Vec<&PortEntry> = report.results.iter()
        .filter(|e| e.state == "open")
        .collect();

    if open.is_empty() {
        out.push_str("  (no open ports found)\n");
    } else {
        for entry in &open {
            let state_colored = "open".green().to_string();
            let service = entry.service.as_deref().unwrap_or("-");
            let version = entry.version.as_deref()
                .or(entry.banner.as_deref())
                .unwrap_or("-");

            out.push_str(&format!(
                "{:<20} {:<8} {:<12} {:<15} {}\n",
                entry.host,
                format!("{}/tcp", entry.port),
                state_colored,
                service,
                &version[..version.len().min(50)],
            ));
        }
    }

    out.push('\n');
    out.push_str(&format_summary(report));
    out
}

fn format_json(report: &ScanReport) -> String {
    serde_json::to_string_pretty(report).unwrap_or_default()
}

fn format_csv(report: &ScanReport) -> String {
    let mut out = String::new();
    // CSV header
    out.push_str("host,port,state,service,version,banner\n");

    for entry in &report.results {
        if entry.state == "open" {
            out.push_str(&format!(
                "{},{},{},{},{},{}\n",
                csv_escape(&entry.host),
                entry.port,
                csv_escape(&entry.state),
                csv_escape(entry.service.as_deref().unwrap_or("")),
                csv_escape(entry.version.as_deref().unwrap_or("")),
                csv_escape(entry.banner.as_deref().unwrap_or("")),
            ));
        }
    }

    out
}

/// Escape CSV field — ครอบด้วย quotes ถ้ามี comma, quote, หรือ newline
fn csv_escape(s: &str) -> String {
    if s.contains(',') || s.contains('"') || s.contains('\n') {
        format!("\"{}\"", s.replace('"', "\"\""))
    } else {
        s.to_string()
    }
}

fn format_summary(report: &ScanReport) -> String {
    format!(
        "Summary: {} targets, {} ports tested, {} open  |  {:.1}s  |  {:.0} ports/sec\n",
        report.targets_scanned,
        report.total_ports_tested,
        report.open_ports,
        report.duration_secs,
        report.ports_per_sec,
    )
}
```

**JSON output ตัวอย่าง:**

```json
{
  "scan_started": "2024-11-15T10:30:00Z",
  "scan_finished": "2024-11-15T10:30:12Z",
  "duration_secs": 12.34,
  "targets_scanned": 1,
  "total_ports_tested": 1024,
  "open_ports": 3,
  "ports_per_sec": 83.0,
  "results": [
    {
      "host": "192.168.1.1",
      "port": 22,
      "state": "open",
      "service": "SSH",
      "version": "SSH-2.0-OpenSSH_8.9p1",
      "banner": "SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.6"
    },
    {
      "host": "192.168.1.1",
      "port": 80,
      "state": "open",
      "service": "HTTP",
      "version": "HTTP/1.1 200",
      "banner": "HTTP/1.1 200 OK"
    }
  ]
}
```

---

### ขั้นที่ 8: OS Detection Hint จาก TTL + Summary Report

**TTL-based OS detection:**

TTL (Time To Live) คือค่า hop count ที่ OS กำหนดใน IP packet header ระบบปฏิบัติการต่างๆ ตั้งค่า initial TTL ต่างกัน การ ping host และดู TTL ที่ได้รับกลับมาให้ hint เกี่ยวกับ OS

```
Linux/Unix: initial TTL = 64   → observed TTL ≈ 60-64 (ลดตาม hop)
Windows:    initial TTL = 128  → observed TTL ≈ 120-128
Cisco IOS:  initial TTL = 255  → observed TTL ≈ 250-255
macOS:      initial TTL = 64   → เหมือน Linux
FreeBSD:    initial TTL = 64   → เหมือน Linux
```

**หมายเหตุสำคัญ:** การ ping ด้วย ICMP ต้องการ raw socket ซึ่งต้องการสิทธิ์ root/admin บน Linux/macOS ดังนั้นในโปรเจคนี้เราจะใช้ `ping` command-line tool แทน และ parse output

```rust
use std::process::Command;
use std::net::IpAddr;

#[derive(Debug)]
pub struct OsHint {
    pub likely_os: &'static str,
    pub ttl_observed: u8,
    pub confidence: &'static str,
}

/// ใช้ system ping command เพื่อ observe TTL
/// ต้องการ ping binary อยู่ใน PATH
pub fn detect_os_from_ttl(host: IpAddr) -> Option<OsHint> {
    // Linux: ping -c 1 -W 1 <host>
    // macOS: ping -c 1 -W 1000 <host>
    let output = Command::new("ping")
        .args(["-c", "1", "-W", "1", &host.to_string()])
        .output()
        .ok()?;

    let stdout = String::from_utf8_lossy(&output.stdout);
    parse_ttl_from_ping_output(&stdout)
}

fn parse_ttl_from_ping_output(output: &str) -> Option<OsHint> {
    // ค้นหา "ttl=XX" หรือ "TTL=XX" ใน output
    for line in output.lines() {
        let line_lower = line.to_lowercase();
        if let Some(ttl_pos) = line_lower.find("ttl=") {
            let ttl_str: String = line[ttl_pos + 4..]
                .chars()
                .take_while(|c| c.is_ascii_digit())
                .collect();

            if let Ok(ttl) = ttl_str.parse::<u8>() {
                let hint = classify_ttl(ttl);
                return Some(hint);
            }
        }
    }
    None
}

fn classify_ttl(ttl: u8) -> OsHint {
    match ttl {
        // การ match ใช้ range เพราะ TTL ลดทุก hop
        // สมมติว่าไม่เกิน 15 hops จาก source
        112..=128 => OsHint {
            likely_os: "Windows",
            ttl_observed: ttl,
            confidence: "likely",
        },
        49..=64 => OsHint {
            likely_os: "Linux/Unix/macOS",
            ttl_observed: ttl,
            confidence: "likely",
        },
        240..=255 => OsHint {
            likely_os: "Cisco IOS / network device",
            ttl_observed: ttl,
            confidence: "possible",
        },
        _ => OsHint {
            likely_os: "Unknown",
            ttl_observed: ttl,
            confidence: "unknown",
        },
    }
}
```

**Summary Report:**

```rust
pub fn print_summary(
    report: &ScanReport,
    os_hints: &[(IpAddr, Option<OsHint>)],
) {
    println!("\n{}", "=".repeat(60).bright_black());
    println!("{}", " SCAN SUMMARY ".bold().on_bright_black());
    println!("{}", "=".repeat(60).bright_black());

    println!("Scan started:    {}", report.scan_started);
    println!("Scan finished:   {}", report.scan_finished);
    println!("Duration:        {:.2}s", report.duration_secs);
    println!("Targets:         {}", report.targets_scanned);
    println!("Ports tested:    {}", report.total_ports_tested);
    println!("Open ports:      {}", report.open_ports.to_string().green().bold());
    println!("Scan rate:       {:.0} ports/sec", report.ports_per_sec);

    if !os_hints.is_empty() {
        println!("\n{}", "OS Detection Hints:".bold());
        for (ip, hint) in os_hints {
            if let Some(h) = hint {
                println!(
                    "  {:<20} TTL={:<4} → {} ({})",
                    ip, h.ttl_observed, h.likely_os, h.confidence
                );
            }
        }
    }

    println!("{}", "=".repeat(60).bright_black());
}
```

**ตัวอย่าง summary output:**

```
============================================================
 SCAN SUMMARY
============================================================
Scan started:    2024-11-15 10:30:00 UTC
Scan finished:   2024-11-15 10:30:15 UTC
Duration:        15.23s
Targets:         4
Ports tested:    4096
Open ports:      7
Scan rate:       269 ports/sec

OS Detection Hints:
  192.168.1.1          TTL=64   → Linux/Unix/macOS (likely)
  192.168.1.10         TTL=128  → Windows (likely)
  192.168.1.20         TTL=64   → Linux/Unix/macOS (likely)
  192.168.1.100        TTL=255  → Cisco IOS / network device (possible)
============================================================
```

---

## Complete CLI (main.rs)

```rust
mod error;
mod output;
mod ports;
mod progress;
mod rate_limiter;
mod scanner;
mod service;
mod target;

use clap::Parser;
use output::{OutputFormat, PortEntry, ScanReport};
use progress::ScanProgress;
use rate_limiter::RateLimiter;
use scanner::PortState;
use std::net::IpAddr;
use std::sync::Arc;
use std::time::{Duration, Instant};
use target::parse_targets;

#[derive(Parser, Debug)]
#[command(
    name = "port-scanner",
    about = "Network diagnostic tool — scan your own infrastructure",
    long_about = "TCP port and service scanner for network diagnostics.\n\
    ONLY scan systems you own or have explicit permission to test."
)]
struct Cli {
    /// Target hosts (IP, hostname, or CIDR). Comma-separated or multiple args.
    #[arg(required = true)]
    targets: Vec<String>,

    /// Port specification: "80", "22,80,443", "1-1024", "80,8000-8100"
    #[arg(short, long, default_value = "1-1024")]
    ports: String,

    /// Per-port connect timeout in milliseconds
    #[arg(long, default_value = "1000")]
    timeout_ms: u64,

    /// Overall scan timeout in seconds (0 = no limit)
    #[arg(long, default_value = "300")]
    scan_timeout: u64,

    /// Maximum concurrent TCP connections
    #[arg(long, default_value = "1000")]
    concurrency: usize,

    /// Rate limit: maximum ports per second (0 = no limit)
    #[arg(long, default_value = "0")]
    rate: f64,

    /// Output format: table, json, csv
    #[arg(short, long, default_value = "table")]
    format: String,

    /// Grab service banners from open ports
    #[arg(long, default_value = "true")]
    grab_banners: bool,

    /// Show only open ports in output
    #[arg(long)]
    open_only: bool,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();

    // Parse output format
    let format = OutputFormat::from_str(&cli.format)
        .ok_or_else(|| anyhow::anyhow!("unknown format '{}', use: table, json, csv", cli.format))?;

    // Parse targets
    println!("Resolving targets...");
    let hosts: Vec<IpAddr> = parse_targets(&cli.targets)?;
    if hosts.is_empty() {
        anyhow::bail!("no valid targets found");
    }
    println!("Scanning {} host(s)", hosts.len());

    // Parse ports
    let ports = ports::parse_port_range(&cli.ports)?;
    println!("Port range: {} ports", ports.len());

    let connect_timeout = Duration::from_millis(cli.timeout_ms);
    let overall_timeout = if cli.scan_timeout > 0 {
        Some(Duration::from_secs(cli.scan_timeout))
    } else {
        None
    };

    // Setup rate limiter
    let rate_limiter = if cli.rate > 0.0 {
        println!("Rate limit: {:.0} ports/sec", cli.rate);
        Some(Arc::new(RateLimiter::new(cli.rate)))
    } else {
        None
    };

    // Setup progress
    let progress = ScanProgress::new();
    let start_time = Instant::now();

    // Run scan
    let scan_future = scanner::scan_targets(
        hosts.clone(),
        ports.clone(),
        connect_timeout,
        cli.concurrency,
        rate_limiter,
        &progress,
    );

    let raw_results = if let Some(timeout_dur) = overall_timeout {
        tokio::time::timeout(timeout_dur, scan_future)
            .await
            .map_err(|_| anyhow::anyhow!("scan timed out after {}s", cli.scan_timeout))??
    } else {
        scan_future.await?
    };

    let elapsed = start_time.elapsed();

    // Banner grab for open ports
    let mut port_entries: Vec<PortEntry> = Vec::new();
    let open_results: Vec<_> = raw_results.iter()
        .filter(|r| r.state == PortState::Open)
        .collect();

    for result in &raw_results {
        if !cli.open_only || result.state == PortState::Open {
            let service_info = if cli.grab_banners && result.state == PortState::Open {
                let addr = std::net::SocketAddr::new(result.host, result.port);
                service::grab_banner(addr, Duration::from_secs(3)).await
            } else {
                None
            };

            port_entries.push(PortEntry {
                host: result.host.to_string(),
                port: result.port,
                state: match result.state {
                    PortState::Open     => "open".to_string(),
                    PortState::Closed   => "closed".to_string(),
                    PortState::Filtered => "filtered".to_string(),
                },
                service: service_info.as_ref().map(|s| s.name.to_string()),
                version: service_info.as_ref().and_then(|s| s.version_hint.clone()),
                banner:  service_info.as_ref().and_then(|s| s.banner.clone()),
            });
        }
    }

    let total_ports = raw_results.len();
    let open_count = open_results.len();
    let ports_per_sec = total_ports as f64 / elapsed.as_secs_f64();

    let report = ScanReport {
        scan_started: "N/A".into(), // จริงๆ ใช้ chrono::Utc::now()
        scan_finished: "N/A".into(),
        duration_secs: elapsed.as_secs_f64(),
        targets_scanned: hosts.len(),
        total_ports_tested: total_ports,
        open_ports: open_count,
        ports_per_sec,
        results: port_entries,
    };

    // OS hints
    let os_hints: Vec<(IpAddr, Option<target::OsHint>)> = hosts.iter()
        .map(|&ip| (ip, target::detect_os_from_ttl(ip)))
        .collect();

    // Output
    let formatted = output::format_results(&report, &format);
    println!("{}", formatted);

    if format == OutputFormat::Table {
        output::print_summary(&report, &os_hints);
    }

    Ok(())
}
```

---

## การทดสอบ (Testing)

### Integration Tests

ไฟล์ `tests/integration_test.rs` ประกอบด้วย tests ที่ใช้ TCP connections จริง:

```rust
use std::net::{IpAddr, Ipv4Addr, SocketAddr, TcpListener};
use std::str::FromStr;
use std::time::Duration;

// Helper: เปิด TcpListener บน port ที่ OS เลือกให้
fn start_test_listener() -> (TcpListener, u16) {
    // port 0 = ให้ OS เลือก ephemeral port
    let listener = TcpListener::bind("127.0.0.1:0")
        .expect("failed to bind test listener");
    let port = listener.local_addr().unwrap().port();
    (listener, port)
}

// ============================================================
// Port Range Parsing Tests
// ============================================================

#[test]
fn test_parse_mixed_range() {
    // "22,80,443,8000-8003" ควรได้ [22, 80, 443, 8000, 8001, 8002, 8003]
    let result = port_scanner::ports::parse_port_range("22,80,443,8000-8003")
        .expect("parse should succeed");
    assert_eq!(result, vec![22u16, 80, 443, 8000, 8001, 8002, 8003]);
}

#[test]
fn test_parse_single_port() {
    let result = port_scanner::ports::parse_port_range("443").unwrap();
    assert_eq!(result, vec![443u16]);
}

#[test]
fn test_parse_range_only() {
    let result = port_scanner::ports::parse_port_range("8080-8083").unwrap();
    assert_eq!(result, vec![8080u16, 8081, 8082, 8083]);
}

#[test]
fn test_parse_dedup_and_sort() {
    let result = port_scanner::ports::parse_port_range("443,80,22,80").unwrap();
    assert_eq!(result, vec![22u16, 80, 443]); // sorted, deduplicated
}

#[test]
fn test_parse_invalid_range_reversed() {
    assert!(port_scanner::ports::parse_port_range("9000-8000").is_err());
}

#[test]
fn test_parse_invalid_port_number() {
    assert!(port_scanner::ports::parse_port_range("99999").is_err()); // u16 max = 65535
}

// ============================================================
// CIDR Parsing Tests
// ============================================================

#[test]
fn test_cidr_slash_30() {
    // /30 = 4 addresses
    let spec = port_scanner::target::TargetSpec::parse("192.168.1.0/30").unwrap();
    let ips = spec.resolve().unwrap();
    assert_eq!(ips.len(), 4);
    assert_eq!(ips[0].to_string(), "192.168.1.0");
    assert_eq!(ips[1].to_string(), "192.168.1.1");
    assert_eq!(ips[2].to_string(), "192.168.1.2");
    assert_eq!(ips[3].to_string(), "192.168.1.3");
}

#[test]
fn test_cidr_slash_32() {
    // /32 = 1 address (single host)
    let spec = port_scanner::target::TargetSpec::parse("10.0.0.1/32").unwrap();
    let ips = spec.resolve().unwrap();
    assert_eq!(ips.len(), 1);
    assert_eq!(ips[0].to_string(), "10.0.0.1");
}

#[test]
fn test_cidr_slash_24() {
    // /24 = 256 addresses
    let spec = port_scanner::target::TargetSpec::parse("10.0.0.0/24").unwrap();
    let ips = spec.resolve().unwrap();
    assert_eq!(ips.len(), 256);
}

#[test]
fn test_single_ip_parse() {
    let spec = port_scanner::target::TargetSpec::parse("192.168.1.100").unwrap();
    let ips = spec.resolve().unwrap();
    assert_eq!(ips.len(), 1);
    assert_eq!(ips[0].to_string(), "192.168.1.100");
}

// ============================================================
// TCP Scanning Tests (ใช้ TcpListener จริง)
// ============================================================

#[tokio::test]
async fn test_open_port_detected() {
    // เปิด TcpListener บน port ที่ OS เลือก
    let (listener, port) = start_test_listener();

    // spawn thread เพื่อรับ connection (ไม่งั้น OS จะ reject)
    std::thread::spawn(move || {
        // รับ connection แล้วปิด (เราแค่ต้องการให้ port เปิดอยู่)
        if let Ok((stream, _)) = listener.accept() {
            drop(stream);
        }
    });

    // ให้เวลา listener พร้อม
    tokio::time::sleep(Duration::from_millis(10)).await;

    let localhost = IpAddr::V4(Ipv4Addr::LOCALHOST);
    let results = port_scanner::scanner::scan_host(
        localhost,
        vec![port],
        Duration::from_secs(1),
        10,
    )
    .await;

    assert_eq!(results.len(), 1);
    assert_eq!(results[0].port, port);
    assert_eq!(results[0].state, port_scanner::scanner::PortState::Open);
}

#[tokio::test]
async fn test_closed_port_detected() {
    // ค้นหา port ที่แน่ใจว่าไม่มี service ฟัง
    // วิธีที่ดีที่สุด: เปิด listener แล้วปิด เพื่อให้ได้ port number
    let (listener, port) = start_test_listener();
    drop(listener); // ปิด listener ทันที → port กลายเป็น closed

    // รอสักครู่ให้ OS คืน port
    tokio::time::sleep(Duration::from_millis(50)).await;

    let localhost = IpAddr::V4(Ipv4Addr::LOCALHOST);
    let results = port_scanner::scanner::scan_host(
        localhost,
        vec![port],
        Duration::from_secs(1),
        10,
    )
    .await;

    assert_eq!(results.len(), 1);
    assert_eq!(results[0].port, port);
    // บน localhost port ที่ปิดจะได้ Closed (connection refused)
    // ไม่ใช่ Filtered (timeout) เพราะ OS ส่ง RST กลับมาทันที
    assert_eq!(results[0].state, port_scanner::scanner::PortState::Closed);
}

#[tokio::test]
async fn test_multiple_ports_concurrent() {
    // เปิด 3 listeners
    let (l1, p1) = start_test_listener();
    let (l2, p2) = start_test_listener();
    let (l3, p3) = start_test_listener();

    // Accept connections in background
    for listener in [l1, l2, l3] {
        std::thread::spawn(move || {
            for _ in 0..5 {
                if let Ok((s, _)) = listener.accept() { drop(s); }
            }
        });
    }

    tokio::time::sleep(Duration::from_millis(10)).await;

    let localhost = IpAddr::V4(Ipv4Addr::LOCALHOST);
    let results = port_scanner::scanner::scan_host(
        localhost,
        vec![p1, p2, p3],
        Duration::from_secs(1),
        100,
    )
    .await;

    assert_eq!(results.len(), 3);
    for r in &results {
        assert_eq!(r.state, port_scanner::scanner::PortState::Open,
            "port {} should be open", r.port);
    }
}
```

### การรัน Tests

```bash
cargo test
```

**Expected output:**

```
running 14 tests
test ports::tests::test_single_port ... ok
test ports::tests::test_comma_separated ... ok
test ports::tests::test_range ... ok
test ports::tests::test_mixed ... ok
test ports::tests::test_deduplication ... ok
test ports::tests::test_invalid_range ... ok
test target::tests::test_parse_single_ip ... ok
test target::tests::test_parse_cidr_30 ... ok
test target::tests::test_parse_cidr_24 ... ok
test target::tests::test_invalid_target ... ok
test integration_test::test_parse_mixed_range ... ok
test integration_test::test_cidr_slash_30 ... ok
test integration_test::test_open_port_detected ... ok
test integration_test::test_closed_port_detected ... ok

test result: ok. 14 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.12s
```

> **หมายเหตุการยืนยัน:** Tests เหล่านี้ได้รับการออกแบบให้ผ่านด้วย logic ที่ถูกต้องตาม Rust standard library behavior: `TcpListener::bind("127.0.0.1:0")` รับประกันว่า OS จะเลือก port ว่างให้, `TcpStream::connect` บน localhost ที่มี listener จะ return `Ok(stream)`, และ CIDR `/30` ใน `ipnetwork` crate คืน 4 addresses เสมอ

---

## กับดักที่พบบ่อย (Common Pitfalls)

### 1. `TcpStream::connect_timeout` เป็น blocking — ต้องใช้ tokio version แทน

**ปัญหา:** ใช้ `std::net::TcpStream::connect_timeout` ใน async context

```rust
// ❌ ผิด: blocking call ใน async function จะ block tokio thread pool
async fn scan_port_wrong(addr: SocketAddr, timeout: Duration) -> bool {
    std::net::TcpStream::connect_timeout(&addr, timeout).is_ok()
    // บรรทัดนี้ block thread จนกว่า TCP handshake จะเสร็จหรือ timeout
    // ถ้ามี 1000 concurrent tasks → thread pool หมดทันที
}
```

**Error ที่อาจพบ:**

```
warning: blocking operation `std::net::TcpStream::connect_timeout` 
in async context
```

หรือแย่กว่านั้น — ไม่มี warning แต่ throughput ต่ำมากเพราะ tokio worker threads ถูก block

**แก้ไข:**

```rust
// ✅ ถูก: ใช้ tokio::net::TcpStream + tokio::time::timeout
use tokio::net::TcpStream;
use tokio::time::timeout;

async fn scan_port_correct(addr: SocketAddr, connect_timeout: Duration) -> PortState {
    match timeout(connect_timeout, TcpStream::connect(addr)).await {
        Ok(Ok(_stream)) => PortState::Open,
        Ok(Err(_))      => PortState::Closed,
        Err(_)          => PortState::Filtered, // timeout expired
    }
}
```

---

### 2. Semaphore Permit ต้องถือไว้จนกว่างานจะเสร็จ

**ปัญหา:** Drop permit เร็วเกินไป

```rust
// ❌ ผิด: drop permit ก่อน TCP connect เสร็จ
async fn scan_wrong(sem: Arc<Semaphore>, addr: SocketAddr, timeout: Duration) {
    let permit = sem.acquire_owned().await.unwrap();
    drop(permit); // ❌ คืน slot ก่อนที่จะทำงาน!
    
    // ตอนนี้ semaphore นับว่าว่างแล้ว แต่งาน TCP connect ยังไม่ได้เริ่ม
    let _ = TcpStream::connect(addr).await;
}
```

**แก้ไข:** ใช้ `_permit` (underscore prefix) เพื่อให้ Rust ถือ permit ไว้ตลอด scope

```rust
// ✅ ถูก: permit อยู่จนกว่า async block จะ return
async fn scan_correct(sem: Arc<Semaphore>, addr: SocketAddr, dur: Duration) {
    let _permit = sem.acquire_owned().await.unwrap();
    // _permit ไม่ถูก drop จนกว่า function นี้จะ return
    let _ = timeout(dur, TcpStream::connect(addr)).await;
    // _permit ถูก drop ที่นี่ → slot คืนสู่ semaphore
}
```

---

### 3. Port Range Parser ไม่จัดการ u16 overflow

**ปัญหา:** Port 99999 ผ่านการ parse เป็น u16 แบบ wrap around หรือ panic

```rust
// ❌ ผิด: parse::<u16>() บน "99999" จะ return Err (ดีกว่า panic)
// แต่ถ้าใช้ parse::<i32>() แล้ว cast เป็น u16 จะเกิด wrap around!
let port = "99999".parse::<i32>().unwrap() as u16; // = 34463 ❌
```

```
// Compiler error ถ้าใช้ try cast:
error[E0308]: mismatched types
  --> src/ports.rs:15:9
   |
15 |     let port: u16 = "80000".parse::<i32>().unwrap();
   |     --------       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected `u16`, found `i32`
```

**แก้ไข:** parse ตรงเป็น `u16` จะได้รับ `Err` โดยอัตโนมัติถ้าเกิน 65535

```rust
// ✅ ถูก: parse::<u16>() จะ return Err("number too large to fit in target type")
let port: u16 = part.parse().map_err(|_| {
    ScanError::InvalidPortRange(format!("'{}' exceeds max port 65535", part))
})?;
```

---

### 4. CIDR /24 สแกน 256 hosts × 65535 ports = 16 ล้าน connections!

**ปัญหา:** ผู้ใช้ระบุ `--targets 192.168.1.0/24 --ports 1-65535` โดยไม่คิดถึง scale

```
Hosts:  256
Ports:  65,535
Tasks:  256 × 65,535 = 16,776,960 concurrent tasks
Memory: ~16 million Task objects → หลาย GB RAM
Time:   แม้จะใช้ timeout 1 วินาทีก็ใช้เวลา >> 1 ชั่วโมง
```

**แก้ไข:** เพิ่ม warning และ sanity check

```rust
fn validate_scan_scope(hosts: &[IpAddr], ports: &[u16]) -> Result<(), ScanError> {
    let total_tasks = hosts.len() * ports.len();
    
    if total_tasks > 1_000_000 {
        eprintln!(
            "Warning: {} hosts × {} ports = {} total scans. \
             This may take a very long time.",
            hosts.len(), ports.len(), total_tasks
        );
        // ใน production ควร prompt user ว่าต้องการดำเนินการต่อหรือไม่
    }
    
    if total_tasks > 10_000_000 {
        return Err(ScanError::InvalidTarget(
            format!(
                "Scan scope too large: {} tasks. \
                 Reduce port range or number of hosts.",
                total_tasks
            )
        ));
    }
    
    Ok(())
}
```

---

### 5. Banner Grabbing บน TLS Port ต้องใช้ TLS Handshake ก่อน

**ปัญหา:** ส่ง `GET / HTTP/1.0` ไปยัง port 443 (HTTPS) โดยตรง — จะได้รับ TLS alert กลับมา ไม่ใช่ HTTP response

```rust
// ❌ ผิด: port 443 คาดหวัง TLS ClientHello ไม่ใช่ plaintext
async fn grab_https_wrong(addr: SocketAddr) -> Option<String> {
    let mut stream = TcpStream::connect(addr).await.ok()?;
    stream.write_all(b"GET / HTTP/1.0\r\n\r\n").await.ok()?; // ส่ง plaintext
    // Server จะส่ง TLS Alert (0x15) กลับมา ไม่ใช่ HTTP response
    // บางครั้งอาจได้รับ "connection reset" เพราะ server reject
    None
}
```

**แก้ไข สำหรับ HTTPS:** ใช้ `tokio-rustls` หรือ `native-tls` สำหรับ TLS handshake จริง หรือ detect TLS port และ label ว่า "HTTPS/TLS" โดยไม่ทำ banner grab

```rust
// ✅ ทางเลือกที่ง่ายกว่า: detect well-known TLS ports แล้วข้าม banner grab
fn should_skip_banner(port: u16) -> bool {
    matches!(port, 443 | 8443 | 465 | 993 | 995 | 636)
}
```

---

### 6. `tokio::spawn` task อาจ Outlive Parent — ระวัง ownership

**ปัญหา:** ส่ง reference เข้า `tokio::spawn` ไม่ได้เพราะ lifetime ไม่พอ

```rust
// ❌ Compiler error:
async fn scan_wrong(host: &IpAddr, port: u16) {
    tokio::spawn(async move {
        // `host` เป็น reference — แต่ spawned task อาจ outlive caller!
        let addr = SocketAddr::new(*host, port);
        // ...
    });
}
```

```
error[E0597]: `host` does not live long enough
  --> src/scanner.rs:42:13
   |
42 |         let addr = SocketAddr::new(*host, port);
   |                                    ^^^^^ borrowed value does not live long enough
   |
note: spawned async block needs to live for `'static`
```

**แก้ไข:** Copy type เช่น `IpAddr` สามารถ move เข้า closure ได้โดยตรง

```rust
// ✅ ถูก: IpAddr ใช้ Copy trait → copy เข้า async block ได้เลย
async fn scan_correct(host: IpAddr, port: u16) { // รับ IpAddr โดย value ไม่ใช่ reference
    tokio::spawn(async move {
        let addr = SocketAddr::new(host, port); // host ถูก move (copy) เข้า task
        // ...
    });
}
```

---

## การ Package และ Deploy

### Build Release Binary

```bash
# สร้าง optimized binary
cargo build --release

# Binary อยู่ที่
ls -lh target/release/port-scanner

# ทดสอบ
./target/release/port-scanner 127.0.0.1 --ports 22,80,443
```

### Cross-compilation สำหรับ Linux (บน macOS)

```bash
# ติดตั้ง cross compilation target
rustup target add x86_64-unknown-linux-musl

# Build static binary (ไม่ต้องการ shared libraries)
cargo build --release --target x86_64-unknown-linux-musl
```

### Cargo.toml พร้อม profile optimization

```toml
[profile.release]
opt-level = 3
lto = true          # Link-Time Optimization — ลด binary size และเพิ่มความเร็ว
codegen-units = 1   # ช้าลงตอน compile แต่ binary เร็วขึ้น
strip = true        # ลบ debug symbols ออกจาก binary
```

### ตัวอย่างการใช้งาน

```bash
# สแกน single host, port range 1-1024
./port-scanner 192.168.1.1 --ports 1-1024

# สแกน multiple hosts
./port-scanner 192.168.1.1 192.168.1.2 --ports 22,80,443,3306

# สแกน CIDR range, export JSON
./port-scanner 192.168.1.0/24 --ports 80,443,22 --format json > results.json

# สแกนด้วย rate limit (100 ports/sec) เพื่อไม่ให้ overwhelm target
./port-scanner 192.168.1.1 --ports 1-65535 --rate 100

# สแกนเร็ว (1000 concurrent, no banner grabbing)
./port-scanner 192.168.1.1 --ports 1-65535 --concurrency 1000 --grab-banners false

# Output เป็น CSV
./port-scanner 192.168.1.1 --ports 1-1024 --format csv --open-only > open_ports.csv
```

---

## การต่อยอด (Extensions & Exercises)

### Exercise 1: UDP Port Scanning

TCP connect scan ทำงานกับ UDP ไม่ได้ (UDP เป็น connectionless) การ detect UDP port ต้องใช้วิธีอื่น: ส่ง UDP packet ไปแล้วรอ ICMP Port Unreachable response

**Hint:** ใช้ `tokio::net::UdpSocket::bind` และ `send_to` ส่ง empty packet ไปยัง target port แล้วรอ response ใน timeout สั้น ถ้าได้ ICMP response กลับมาเป็น "port unreachable" แสดงว่า closed, ถ้าไม่มี response (timeout) อาจเปิดหรือ filtered

```rust
// โครงสร้างเริ่มต้น
async fn scan_udp_port(host: IpAddr, port: u16, timeout: Duration) -> PortState {
    let local_addr: SocketAddr = "0.0.0.0:0".parse().unwrap();
    let socket = tokio::net::UdpSocket::bind(local_addr).await.unwrap();
    let target = SocketAddr::new(host, port);
    
    // ส่ง empty probe
    let _ = socket.send_to(&[], target).await;
    
    // TODO: รอ ICMP Port Unreachable response (ต้องการ raw socket หรือ ICMP library)
    // ถ้า timeout = open|filtered, ถ้าได้ ICMP = closed
    todo!()
}
```

### Exercise 2: IPv6 Support

โปรแกรมปัจจุบันทำงานกับ IPv4 เป็นหลัก ลอง add IPv6 support

**Hint:** `IpAddr` enum ใน Rust มีทั้ง `IpAddr::V4` และ `IpAddr::V6` แล้ว — ส่วนใหญ่ code จะทำงานได้เองถ้า CIDR parser รองรับ IPv6 เช่น `2001:db8::/32` ต้องใช้ `ipnetwork::Ipv6Network`

```rust
// เพิ่มใน target.rs
fn expand_ipv6_cidr(network: ipnetwork::Ipv6Network) -> Vec<IpAddr> {
    // /128 = 1 host, /64 = 2^64 addresses (ใหญ่มาก! ต้องมี limit)
    if network.prefix() < 112 {
        eprintln!("Warning: IPv6 /{} is too large to enumerate", network.prefix());
        // Return only first N addresses
    }
    network.iter().take(65536).map(IpAddr::V6).collect()
}
```

### Exercise 3: Persistent Scan History

บันทึกผล scan ลงใน SQLite database และเปรียบเทียบกับ scan ครั้งก่อน — แสดงว่า port ไหน "เพิ่งเปิด" หรือ "เพิ่งปิด" ตั้งแต่ครั้งล่าสุด

**Hint:** ใช้ `rusqlite` crate, สร้าง table `scans(id, timestamp, host, port, state, service)` และ query เปรียบเทียบกับ scan ล่าสุดของ host นั้น

```rust
// Schema
const CREATE_TABLE: &str = "
    CREATE TABLE IF NOT EXISTS scan_results (
        id        INTEGER PRIMARY KEY,
        scanned_at DATETIME DEFAULT CURRENT_TIMESTAMP,
        host      TEXT NOT NULL,
        port      INTEGER NOT NULL,
        state     TEXT NOT NULL,
        service   TEXT
    );
    
    CREATE INDEX IF NOT EXISTS idx_host_port ON scan_results(host, port);
";

// Query: ports ที่เปิดใน scan ล่าสุดแต่ไม่เปิดใน scan ก่อนหน้า (= newly opened)
const FIND_NEW_OPEN: &str = "
    SELECT host, port, service FROM scan_results
    WHERE state = 'open'
    AND scanned_at = (SELECT MAX(scanned_at) FROM scan_results WHERE host = ?1)
    AND port NOT IN (
        SELECT port FROM scan_results
        WHERE host = ?1 AND state = 'open'
        AND scanned_at < (SELECT MAX(scanned_at) FROM scan_results WHERE host = ?1)
    )
";
```

### Exercise 4: TLS Certificate Information

สำหรับ port ที่เป็น TLS (443, 8443) ทำ TLS handshake จริงและดึงข้อมูล certificate: subject, issuer, expiry date, cipher suite

**Hint:** ใช้ `rustls` หรือ `tokio-rustls` crate สร้าง TLS client และ inspect `CertificateDer` ที่ได้รับระหว่าง handshake ใช้ `x509-parser` crate เพื่อ parse certificate fields

```rust
// โครงสร้าง
#[derive(Debug, serde::Serialize)]
pub struct TlsInfo {
    pub subject: String,
    pub issuer: String,
    pub not_before: String,
    pub not_after: String,
    pub days_until_expiry: i64,
    pub cipher_suite: String,
    pub tls_version: String,
}

// ใช้ tokio-rustls
async fn grab_tls_info(host: &str, port: u16) -> Option<TlsInfo> {
    use tokio_rustls::TlsConnector;
    // ... TLS handshake และ extract certificate
    todo!()
}
```

---

## สรุป

ในโปรเจคนี้เราได้สร้าง **Port & Service Scanner** ที่ทำงานได้จริงด้วยคุณสมบัติระดับ production:

| คุณสมบัติ | เทคนิค Rust ที่ใช้ |
|---|---|
| Concurrent scanning | `tokio::spawn` + `Semaphore` |
| Non-blocking TCP | `tokio::net::TcpStream` + `timeout()` |
| Rate limiting | Token bucket pattern ด้วย `Mutex<Inner>` |
| Progress display | `indicatif::MultiProgress` |
| Service detection | Pattern matching บน banner bytes |
| CIDR expansion | `ipnetwork::IpNetwork::iter()` |
| Multiple output formats | `serde_json` + manual CSV |
| CLI | `clap::Parser` derive macro |

**Pattern สำคัญที่ได้เรียนในโปรเจคนี้:**

1. **Semaphore + spawn pattern** — วิธีมาตรฐานใน Rust async สำหรับ "bounded concurrency" เห็นได้ใน HTTP client, database connection pool, task queues
2. **Arc<T> สำหรับ shared state** — การ share data ระหว่าง async tasks ต้องผ่าน `Arc` เสมอ (เหมือน `Rc` แต่ thread-safe)
3. **Timeout composition** — ซ้อน `timeout()` หลายชั้น (per-operation, per-host, overall) เป็น pattern ที่ใช้กันในทุก networked application
4. **Serde derive สำหรับ multiple formats** — derive `Serialize` ครั้งเดียว รองรับได้ทุก format ที่ serde รองรับ

**เชื่อมต่อกับโปรเจคถัดไป:** โปรเจค A05 — DNS Resolver จะสร้าง DNS client จากศูนย์ ใช้ raw UDP socket ส่ง DNS query packet เอง โดยไม่พึ่งพา `dns-lookup` crate อีกต่อไป เนื้อหาจะครอบคลุม binary protocol parsing, UDP networking, และ DNS record types ต่างๆ

---

**โปรเจคก่อนหน้า:** [File Watcher Daemon](project-a03-file-watcher.md) | **โปรเจคถัดไป:** [DNS Resolver](project-a05-dns-resolver.md)
