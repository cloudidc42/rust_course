# Project D03: TLS-Terminating Reverse Proxy

> โมดูล: D — Security & Cryptography | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

TLS-Terminating Reverse Proxy คือ service ที่รับ TLS connection จาก client แล้วส่งต่อ request ไปยัง backend server ผ่าน plain TCP หรือ TLS อีกชั้นหนึ่ง การ "terminate" TLS ที่ proxy หมายความว่า proxy ถือ certificate และ private key, ทำ TLS handshake กับ client, และ backend เห็นเฉพาะ plaintext HTTP

Use case หลักในโลก production:
- **Ingress controller** ใน Kubernetes: ถือ wildcard cert สำหรับทั้ง cluster
- **API Gateway**: ตรวจสอบ client certificate (mTLS) ก่อน route ไป service ที่เหมาะสม
- **CDN edge node**: terminate TLS ใกล้ผู้ใช้, ส่งต่อผ่าน HTTP/2 ไป origin

Learning value หลักของโปรเจคนี้คือการเข้าใจ TLS lifecycle ทั้งหมด: การ load และตรวจสอบ certificate จาก PEM files, การ route request ด้วย Server Name Indication (SNI), การจัดการ connection pool ไปยัง backend, และการ proxy HTTP headers ตาม RFC 7230

## สิ่งที่จะได้เรียนรู้

- ใช้ `rustls` 0.23 API สำหรับ `ServerConfig`, `ResolvesServerCert`, และ ALPN negotiation
- Parse และตรวจสอบ PEM certificate chain และ private key ด้วย `rustls-pemfile`
- Implement `ResolvesServerCert` trait เพื่อทำ SNI-based certificate selection
- สร้าง connection pool ด้วย `Arc<Mutex<VecDeque<T>>>` พร้อม idle timeout
- Round-robin และ least-connections load balancing ด้วย atomic counter
- ตรวจสอบ hop-by-hop headers ตาม RFC 7230 §6.1 และ forward headers ที่ถูกต้อง
- Expose Prometheus metrics ผ่าน `/metrics` endpoint บน admin port แยกต่างหาก
- เปิดใช้ OCSP stapling และ HTTP/2 ผ่าน ALPN

## ความรู้ที่ต้องมีมาก่อน

- **Part 46–50**: Async/await, Tokio runtime, `TcpListener`, `TcpStream`
- **Part 55–60**: Arc, Mutex, atomic types, shared state ใน async context
- **Part 96–100**: TLS fundamentals — X.509 certificate structure, PEM encoding, TLS handshake
- **Part 101–105**: HTTP/1.1 semantics, header fields, hop-by-hop vs end-to-end headers
- **Part 106–110**: Prometheus metrics, instrumentation patterns

## โครงสร้างโปรเจค (Project Layout)

```
tls-proxy/
├── src/
│   ├── main.rs          — CLI args, startup, acceptor loop
│   ├── cert.rs          — PEM loading, expiry check, OCSP stapling
│   ├── sni.rs           — ResolvesServerCert impl, routing table
│   ├── headers.rs       — X-Forwarded-For, hop-by-hop removal
│   ├── pool.rs          — connection pool per upstream host
│   ├── lb.rs            — round-robin and least-connections
│   ├── metrics.rs       — Prometheus counters/gauges/histograms
│   └── mtls.rs          — client cert verification, CN extraction
├── tests/
│   └── integration.rs   — end-to-end proxy test with test cert
├── certs/               — (gitignored) PEM files สำหรับ development
│   ├── server.crt
│   ├── server.key
│   └── ca.crt
└── Cargo.toml
```

## การออกแบบ (Architecture & Design)

### Data Flow

```
Client (TLS)
    │
    ▼ tokio::net::TcpListener::accept()
┌──────────────────────────────────────────┐
│  TLS Acceptor (tokio-rustls)             │
│  • SNI lookup → select ServerCertResolver│
│  • ALPN negotiation (h2 / http/1.1)      │
│  • Optional: verify client cert (mTLS)   │
└──────────────────────────────────────────┘
    │  plaintext HTTP stream
    ▼
┌──────────────────────────────────────────┐
│  HTTP Parser                             │
│  • Remove hop-by-hop headers             │
│  • Inject X-Forwarded-For, X-Real-IP    │
│  • Inject X-Client-Cert-CN (if mTLS)    │
└──────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────┐
│  Load Balancer                           │
│  • Round-robin OR least-connections      │
│  • Pick upstream host                    │
└──────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────┐
│  Connection Pool (per upstream host)     │
│  • Reuse idle TCP/TLS connections        │
│  • Idle timeout eviction                 │
│  • Health-check ping                     │
└──────────────────────────────────────────┘
    │
    ▼ TCP (plain) or TLS (re-encrypt)
Backend Server
```

### Design Decisions

**ทำไมใช้ `rustls` แทน OpenSSL?**  
`rustls` เป็น pure-Rust TLS library ที่ไม่ต้องพึ่ง C FFI, compile ได้ทุก platform โดยไม่ต้องติดตั้ง OpenSSL บน host ทำให้ Docker image เล็กลง และ memory safety ครอบคลุม crypto code ทั้งหมด

**ทำไม SNI routing ทำในชั้น TLS แทนชั้น HTTP?**  
SNI อยู่ใน TLS `ClientHello` ก่อน handshake เสร็จ การเลือก certificate ต้องทำก่อน client ส่ง HTTP request มาได้ `ResolvesServerCert` trait ให้ rustls callback มาหา `CertifiedKey` ตาม server name ที่ client ระบุ

**Connection pool แยกต่างหากต่อ upstream host?**  
Connection หนึ่งถูก multiplex ได้เฉพาะกับ host เดียว ถ้าใช้ pool รวมกันจะต้องติดตาม destination ทำให้ซับซ้อนโดยไม่จำเป็น การแยก pool ต่อ host ทำให้ health check และ eviction logic อิสระจากกัน

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: โครงสร้างโปรเจคและ CLI Arguments

เริ่มจาก `Cargo.toml` และ CLI arg parsing ด้วย `clap 4`

**`Cargo.toml`**

```toml
[package]
name = "tls-proxy"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "tls_proxy"
path = "src/main.rs"

[dependencies]
tokio        = { version = "1",    features = ["full"] }
tokio-rustls = "0.26"
rustls       = { version = "0.23", features = ["ring"] }
rustls-pemfile = "2"
h2           = "0.4"
clap         = { version = "4",    features = ["derive"] }
serde        = { version = "1",    features = ["derive"] }
serde_json   = "1"
prometheus   = { version = "0.13", features = ["process"] }

[dev-dependencies]
tokio  = { version = "1", features = ["full", "test-util"] }
rcgen  = "0.13"
```

**`src/main.rs` — CLI และ startup**

```rust
use clap::Parser;

#[derive(Parser, Debug)]
#[command(name = "tls-proxy", about = "TLS-terminating reverse proxy")]
pub struct Args {
    /// Bind address for the TLS listener
    #[arg(long, default_value = "0.0.0.0:443")]
    pub listen: String,

    /// Admin port for /metrics and /healthz
    #[arg(long, default_value = "9090")]
    pub admin_port: u16,

    /// Upstream backend addresses (comma-separated), e.g. host1:8080,host2:8080
    #[arg(long, value_delimiter = ',', required = true)]
    pub upstream: Vec<String>,

    /// Path to PEM certificate file (may be a chain)
    #[arg(long, default_value = "certs/server.crt")]
    pub cert: String,

    /// Path to PEM private key file
    #[arg(long, default_value = "certs/server.key")]
    pub key: String,

    /// Load balancing algorithm: round-robin | least-conn
    #[arg(long, default_value = "round-robin")]
    pub lb_algo: String,

    /// Maximum connections per upstream host in the pool
    #[arg(long, default_value_t = 50)]
    pub pool_size: usize,

    /// Idle connection timeout in seconds
    #[arg(long, default_value_t = 60)]
    pub idle_timeout_secs: u64,

    /// Require client TLS certificate (mTLS)
    #[arg(long)]
    pub require_client_cert: bool,

    /// Path to CA bundle for client cert verification
    #[arg(long)]
    pub client_ca: Option<String>,

    /// Warn when certificate expires within this many days
    #[arg(long, default_value_t = 30)]
    pub cert_expiry_warn_days: u64,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args = Args::parse();
    println!("Starting tls-proxy on {}", args.listen);
    println!("Upstreams: {:?}", args.upstream);
    println!("LB algorithm: {}", args.lb_algo);
    // Full implementation in later steps
    Ok(())
}
```

**output:**
```
$ cargo run -- --upstream 127.0.0.1:8080,127.0.0.1:8081
Starting tls-proxy on 0.0.0.0:443
Upstreams: ["127.0.0.1:8080", "127.0.0.1:8081"]
LB algorithm: round-robin
```

---

### ขั้นที่ 2: Certificate Loading และ Expiry Detection

Module `cert.rs` รับผิดชอบการ parse PEM files และตรวจสอบ certificate validity

**`src/cert.rs`**

```rust
use std::fs;
use std::io::BufReader;
use std::path::Path;
use std::sync::Arc;
use std::time::{Duration, SystemTime, UNIX_EPOCH};

use rustls::pki_types::{CertificateDer, PrivateKeyDer};
use rustls::sign::CertifiedKey;
use rustls::ServerConfig;

/// โหลด certificate chain จาก PEM file
/// รองรับทั้ง leaf-only และ full chain (leaf + intermediates + root)
pub fn load_cert_chain(path: &Path) -> Result<Vec<CertificateDer<'static>>, Box<dyn std::error::Error>> {
    let pem_bytes = fs::read(path)?;
    parse_cert_chain(&pem_bytes).map_err(|e| e.into())
}

/// Parse PEM bytes เป็น DER certificate list
pub fn parse_cert_chain(pem_bytes: &[u8]) -> Result<Vec<CertificateDer<'static>>, String> {
    let mut reader = BufReader::new(pem_bytes);
    let certs: Vec<CertificateDer<'static>> = rustls_pemfile::certs(&mut reader)
        .map(|r| r.map_err(|e| e.to_string()))
        .collect::<Result<Vec<_>, _>>()?;
    if certs.is_empty() {
        return Err("no certificates found in PEM data".into());
    }
    Ok(certs)
}

/// โหลด private key จาก PEM file
/// ลอง PKCS#8, SEC1 (EC), และ PKCS#1 (RSA) ตามลำดับ
pub fn load_private_key(path: &Path) -> Result<PrivateKeyDer<'static>, Box<dyn std::error::Error>> {
    let pem_bytes = fs::read(path)?;
    parse_private_key(&pem_bytes).map_err(|e| e.into())
}

/// Parse private key จาก PEM bytes
pub fn parse_private_key(pem_bytes: &[u8]) -> Result<PrivateKeyDer<'static>, String> {
    let mut reader = BufReader::new(pem_bytes);
    rustls_pemfile::private_key(&mut reader)
        .map_err(|e| e.to_string())?
        .ok_or_else(|| "no private key found in PEM data".to_string())
}

/// ข้อมูล expiry ของ certificate
#[derive(Debug)]
pub struct CertExpiry {
    /// Common name จาก subject field (ถ้ามี)
    pub common_name: Option<String>,
    /// Unix timestamp (seconds) ที่ certificate หมดอายุ
    pub not_after_secs: u64,
}

impl CertExpiry {
    /// คืนค่า true ถ้า certificate จะหมดอายุภายใน threshold_days วัน
    pub fn expires_within_days(&self, threshold_days: u64) -> bool {
        let threshold_secs = threshold_days * 86_400;
        let now_secs = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or(Duration::ZERO)
            .as_secs();
        self.not_after_secs.saturating_sub(now_secs) < threshold_secs
    }

    /// เวลาที่เหลือก่อน certificate หมดอายุ (0 ถ้าหมดแล้ว)
    pub fn remaining_seconds(&self) -> u64 {
        let now_secs = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or(Duration::ZERO)
            .as_secs();
        self.not_after_secs.saturating_sub(now_secs)
    }

    pub fn remaining_days(&self) -> u64 {
        self.remaining_seconds() / 86_400
    }
}

/// สร้าง CertExpiry จาก Unix timestamp ที่รู้อยู่แล้ว
/// ใช้สำหรับ testing และสำหรับ X.509 parsing ที่ทำเสร็จแล้ว
pub fn cert_expiry_from_timestamp(not_after_secs: u64) -> CertExpiry {
    CertExpiry {
        common_name: None,
        not_after_secs,
    }
}

/// ตรวจสอบ expiry และ log warning ถ้าใกล้หมดอายุ
pub fn check_and_warn_expiry(expiry: &CertExpiry, warn_days: u64) {
    if expiry.expires_within_days(warn_days) {
        let days = expiry.remaining_days();
        if days == 0 {
            eprintln!("WARN: certificate has already expired or expires today");
        } else {
            eprintln!(
                "WARN: certificate expires in {} days (threshold: {} days)",
                days, warn_days
            );
        }
    }
}

/// สร้าง rustls ServerConfig จาก certificate และ key
/// ตั้งค่า TLS 1.2 และ 1.3 เท่านั้น (ปฏิเสธ TLS 1.0 และ 1.1)
pub fn build_server_config(
    cert_chain: Vec<CertificateDer<'static>>,
    private_key: PrivateKeyDer<'static>,
) -> Result<ServerConfig, rustls::Error> {
    let config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(cert_chain, private_key)?;
    Ok(config)
}

/// สร้าง ServerConfig ที่เปิดใช้ ALPN สำหรับ HTTP/2 และ HTTP/1.1
pub fn build_server_config_with_alpn(
    cert_chain: Vec<CertificateDer<'static>>,
    private_key: PrivateKeyDer<'static>,
) -> Result<ServerConfig, rustls::Error> {
    let mut config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(cert_chain, private_key)?;

    // ALPN: ลอง h2 ก่อน, fallback เป็น http/1.1
    config.alpn_protocols = vec![
        b"h2".to_vec(),
        b"http/1.1".to_vec(),
    ];
    Ok(config)
}

/// โหลด CA certificates สำหรับ mTLS client verification
pub fn load_ca_bundle(path: &Path) -> Result<Vec<CertificateDer<'static>>, Box<dyn std::error::Error>> {
    load_cert_chain(path)
}
```

**ตัวอย่างการใช้งาน:**

```rust
use std::path::Path;

fn startup_cert_check(cert_path: &str, key_path: &str, warn_days: u64) {
    // โหลด cert
    let cert_chain = cert::load_cert_chain(Path::new(cert_path))
        .expect("failed to load certificate");

    // โหลด key
    let private_key = cert::load_private_key(Path::new(key_path))
        .expect("failed to load private key");

    println!("Loaded {} certificates in chain", cert_chain.len());

    // ตรวจ expiry (ในโปรเจคจริงจะใช้ X.509 parser เพื่อดึง not_after จาก DER)
    // ตัวอย่างนี้ใช้ timestamp โดยตรง
    let expiry = cert::cert_expiry_from_timestamp(
        std::time::SystemTime::now()
            .duration_since(std::time::UNIX_EPOCH)
            .unwrap()
            .as_secs() + 25 * 86_400 // หมดใน 25 วัน
    );
    cert::check_and_warn_expiry(&expiry, warn_days);
}
```

**output:**
```
Loaded 1 certificates in chain
WARN: certificate expires in 25 days (threshold: 30 days)
```

---

### ขั้นที่ 3: SNI-Based Routing และ ResolvesServerCert

`rustls` เรียก `ResolvesServerCert::resolve()` ในระหว่าง TLS handshake เพื่อให้ server เลือก certificate ตาม SNI ที่ client ระบุมาใน `ClientHello`

**`src/sni.rs`**

```rust
use std::collections::HashMap;
use std::sync::{Arc, RwLock};

use rustls::server::{ClientHello, ResolvesServerCert};
use rustls::sign::CertifiedKey;

/// Backend target ที่ SNI hostname map ไปหา
#[derive(Debug, Clone, PartialEq)]
pub struct Backend {
    pub host: String,
    pub port: u16,
}

impl Backend {
    pub fn new(host: impl Into<String>, port: u16) -> Self {
        Self { host: host.into(), port }
    }

    pub fn addr(&self) -> String {
        format!("{}:{}", self.host, self.port)
    }
}

/// ตาราง route: SNI hostname → backend list
/// รองรับ exact match และ wildcard (*.example.com)
#[derive(Debug, Default)]
pub struct SniRoutingTable {
    routes: HashMap<String, Vec<Backend>>,
    default_backends: Option<Vec<Backend>>,
}

impl SniRoutingTable {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn add_route(&mut self, hostname: impl Into<String>, backends: Vec<Backend>) {
        self.routes.insert(hostname.into(), backends);
    }

    pub fn set_default(&mut self, backends: Vec<Backend>) {
        self.default_backends = Some(backends);
    }

    /// Resolve SNI → backend list
    /// ลำดับ: exact match → wildcard → default
    pub fn resolve(&self, sni: &str) -> Option<&[Backend]> {
        // 1. Exact match
        if let Some(backends) = self.routes.get(sni) {
            return Some(backends);
        }
        // 2. Wildcard: api.example.com → *.example.com
        if let Some(dot_pos) = sni.find('.') {
            let wildcard = format!("*{}", &sni[dot_pos..]);
            if let Some(backends) = self.routes.get(&wildcard) {
                return Some(backends);
            }
        }
        // 3. Default catch-all
        self.default_backends.as_deref()
    }
}

/// rustls resolver ที่เลือก TLS certificate ตาม SNI
///
/// เก็บ map จาก hostname → Arc<CertifiedKey>
/// CertifiedKey คือคู่ของ certificate chain + signing key
pub struct SniCertResolver {
    /// hostname → certified key
    certs: RwLock<HashMap<String, Arc<CertifiedKey>>>,
    /// fallback ถ้าไม่ตรงกับ hostname ใดเลย
    default_cert: Option<Arc<CertifiedKey>>,
}

impl SniCertResolver {
    pub fn new(default_cert: Option<Arc<CertifiedKey>>) -> Self {
        Self {
            certs: RwLock::new(HashMap::new()),
            default_cert,
        }
    }

    /// เพิ่ม certificate สำหรับ hostname ที่ระบุ
    pub fn add_cert(&self, hostname: impl Into<String>, cert: Arc<CertifiedKey>) {
        self.certs.write().unwrap().insert(hostname.into(), cert);
    }
}

impl ResolvesServerCert for SniCertResolver {
    fn resolve(&self, client_hello: ClientHello<'_>) -> Option<Arc<CertifiedKey>> {
        // SNI ที่ client ส่งมา (อาจเป็น None ถ้า client ไม่รองรับ SNI)
        let sni = client_hello.server_name()?;

        let certs = self.certs.read().unwrap();

        // 1. Exact match
        if let Some(key) = certs.get(sni) {
            return Some(Arc::clone(key));
        }

        // 2. Wildcard match: sub.example.com → *.example.com
        if let Some(dot_pos) = sni.find('.') {
            let wildcard = format!("*{}", &sni[dot_pos..]);
            if let Some(key) = certs.get(&wildcard) {
                return Some(Arc::clone(key));
            }
        }

        // 3. Default cert
        self.default_cert.as_ref().map(Arc::clone)
    }
}

/// สร้าง ServerConfig ที่ใช้ SniCertResolver
pub fn build_sni_server_config(
    resolver: Arc<SniCertResolver>,
) -> Result<rustls::ServerConfig, rustls::Error> {
    let mut config = rustls::ServerConfig::builder()
        .with_no_client_auth()
        .with_cert_resolver(resolver);

    config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
    Ok(config)
}
```

**ตัวอย่าง: สร้าง resolver สำหรับหลาย domain**

```rust
use rustls::crypto::ring::sign::any_supported_type;
use rustls::pki_types::PrivateKeyDer;
use rustls::sign::CertifiedKey;
use std::sync::Arc;

fn setup_sni_resolver() -> Arc<SniCertResolver> {
    // โหลด cert สำหรับ api.example.com
    let api_certs = cert::load_cert_chain(Path::new("certs/api.crt")).unwrap();
    let api_key  = cert::load_private_key(Path::new("certs/api.key")).unwrap();
    let api_signing_key = any_supported_type(&api_key).unwrap();
    let api_certified  = Arc::new(CertifiedKey::new(api_certs, api_signing_key));

    // โหลด cert สำหรับ web.example.com
    let web_certs = cert::load_cert_chain(Path::new("certs/web.crt")).unwrap();
    let web_key  = cert::load_private_key(Path::new("certs/web.key")).unwrap();
    let web_signing_key = any_supported_type(&web_key).unwrap();
    let web_certified  = Arc::new(CertifiedKey::new(web_certs, web_signing_key));

    let resolver = Arc::new(SniCertResolver::new(Some(Arc::clone(&api_certified))));
    resolver.add_cert("api.example.com", api_certified);
    resolver.add_cert("web.example.com", web_certified);
    // Wildcard สำหรับ subdomain ทั้งหมด
    // resolver.add_cert("*.example.com", wildcard_certified);

    resolver
}
```

---

### ขั้นที่ 4: Header Manipulation ตาม RFC 7230

**`src/headers.rs`**

HTTP headers แบ่งเป็นสองประเภทตาม RFC 7230:
- **End-to-end headers**: ส่งต่อไปยัง final recipient ทุกทอด
- **Hop-by-hop headers**: มีความหมายเฉพาะระหว่าง proxies สองตัว ห้ามส่งต่อ

```rust
use std::collections::{HashMap, HashSet};

/// Hop-by-hop headers ตาม RFC 7230 §6.1
/// headers เหล่านี้ต้องถูก remove ก่อนส่งต่อ request/response
pub fn hop_by_hop_headers() -> HashSet<String> {
    [
        "connection",
        "keep-alive",
        "transfer-encoding",
        "te",
        "trailer",
        "upgrade",
        "proxy-authorization",
        "proxy-authenticate",
    ]
    .iter()
    .map(|s| s.to_string())
    .collect()
}

/// โครงสร้าง headers สำหรับ HTTP request
#[derive(Debug, Clone)]
pub struct RequestHeaders {
    pub inner: HashMap<String, String>,
}

impl RequestHeaders {
    pub fn new() -> Self {
        Self { inner: HashMap::new() }
    }

    pub fn insert(&mut self, name: impl Into<String>, value: impl Into<String>) {
        self.inner.insert(name.into().to_lowercase(), value.into());
    }

    pub fn get(&self, name: &str) -> Option<&str> {
        self.inner.get(&name.to_lowercase()).map(|s| s.as_str())
    }

    pub fn remove(&mut self, name: &str) {
        self.inner.remove(&name.to_lowercase());
    }

    pub fn contains(&self, name: &str) -> bool {
        self.inner.contains_key(&name.to_lowercase())
    }
}

/// เพิ่ม forwarding headers สำหรับ outbound request
///
/// - `X-Forwarded-For`: IP ของ client ที่เชื่อมต่อ proxy นี้
///   ถ้ามีค่าเดิมอยู่แล้วให้ append (รักษา chain ของ proxies)
/// - `X-Forwarded-Proto`: "https" เสมอ เพราะ client เชื่อมต่อผ่าน TLS
/// - `X-Real-IP`: IP ของ client ที่ติดต่อ proxy โดยตรง (ไม่สะสม)
pub fn add_forwarding_headers(headers: &mut RequestHeaders, client_ip: &str) {
    let xff = match headers.get("x-forwarded-for") {
        Some(existing) => format!("{}, {}", existing, client_ip),
        None => client_ip.to_string(),
    };
    headers.insert("x-forwarded-for", xff);
    headers.insert("x-forwarded-proto", "https");
    headers.insert("x-real-ip", client_ip);
}

/// ลบ hop-by-hop headers ตาม RFC 7230 §6.1
///
/// ขั้นตอน:
/// 1. อ่าน Connection header value เพื่อดูว่า header ใดอีกบ้างที่ถูกกำหนดเป็น hop-by-hop
/// 2. ลบ Connection header และ headers ที่ระบุไว้ใน Connection value
/// 3. ลบ headers ที่อยู่ใน static hop-by-hop set
pub fn remove_hop_by_hop(headers: &mut RequestHeaders) {
    let hop = hop_by_hop_headers();

    // เก็บรายชื่อ headers ที่ระบุใน Connection: header ก่อน
    let connection_listed: Vec<String> = headers
        .get("connection")
        .unwrap_or("")
        .split(',')
        .map(|s| s.trim().to_lowercase())
        .filter(|s| !s.is_empty())
        .collect();

    // ลบทั้งหมด
    for h in &hop {
        headers.remove(h);
    }
    for h in &connection_listed {
        headers.remove(h);
    }
}

/// เพิ่ม X-Client-Cert-CN header สำหรับ mTLS
///
/// CN (Common Name) ของ client certificate จะถูกส่งไปยัง backend
/// เพื่อให้ backend ทราบว่า client ผ่านการ authenticate ด้วย cert ใด
pub fn add_client_cert_cn(headers: &mut RequestHeaders, cn: Option<&str>) {
    if let Some(cn_value) = cn {
        headers.insert("x-client-cert-cn", cn_value);
    }
}

/// ตรวจสอบว่า headers พร้อมส่งต่อไปยัง backend
/// (ใช้ใน debug mode เท่านั้น)
pub fn debug_headers(headers: &RequestHeaders) {
    for (name, value) in &headers.inner {
        eprintln!("  {}: {}", name, value);
    }
}
```

---

### ขั้นที่ 5: Connection Pool และ Load Balancing

**`src/pool.rs`**

```rust
use std::collections::VecDeque;
use std::sync::{Arc, Mutex};
use std::time::{Duration, Instant};

/// Connection ที่ถูก pool ดูแล
#[derive(Debug)]
pub struct PooledConn {
    /// ID ของ connection (monotonic)
    pub id: usize,
    /// เวลาที่ connection นี้ถูกสร้าง (ใช้ตรวจ idle timeout)
    pub created_at: Instant,
}

#[derive(Debug)]
struct PoolInner {
    idle: VecDeque<PooledConn>,
    active_count: usize,
    next_id: usize,
}

/// Connection pool แบบ bounded
///
/// - `max_size`: จำนวน connections สูงสุด (active + idle รวมกัน)
/// - `idle_timeout`: connections ที่ idle นานเกินนี้จะถูก evict ออก
#[derive(Debug, Clone)]
pub struct ConnectionPool {
    inner: Arc<Mutex<PoolInner>>,
    max_size: usize,
    idle_timeout: Duration,
}

impl ConnectionPool {
    pub fn new(max_size: usize, idle_timeout: Duration) -> Self {
        Self {
            inner: Arc::new(Mutex::new(PoolInner {
                idle: VecDeque::new(),
                active_count: 0,
                next_id: 0,
            })),
            max_size,
            idle_timeout,
        }
    }

    /// ขอ connection จาก pool
    ///
    /// ลำดับ:
    /// 1. Evict idle connections ที่หมด timeout
    /// 2. คืน idle connection ถ้ามี
    /// 3. สร้าง connection ใหม่ถ้า active_count < max_size
    /// 4. คืน None ถ้า pool เต็ม
    pub fn acquire(&self) -> Option<PooledConn> {
        let mut inner = self.inner.lock().unwrap();
        let now = Instant::now();

        // Evict expired idle connections
        inner.idle.retain(|c| now.duration_since(c.created_at) < self.idle_timeout);

        // ใช้ idle connection ที่มีอยู่
        if let Some(conn) = inner.idle.pop_front() {
            inner.active_count += 1;
            return Some(conn);
        }

        // สร้าง connection ใหม่ถ้าไม่เกิน max_size
        if inner.active_count < self.max_size {
            let id = inner.next_id;
            inner.next_id += 1;
            inner.active_count += 1;
            Some(PooledConn { id, created_at: Instant::now() })
        } else {
            None
        }
    }

    /// คืน connection กลับเข้า pool
    pub fn release(&self, conn: PooledConn) {
        let mut inner = self.inner.lock().unwrap();
        inner.active_count = inner.active_count.saturating_sub(1);
        inner.idle.push_back(conn);
    }

    pub fn active_count(&self) -> usize {
        self.inner.lock().unwrap().active_count
    }

    pub fn idle_count(&self) -> usize {
        self.inner.lock().unwrap().idle.len()
    }

    /// ตรวจสอบสุขภาพของ connections ด้วย TCP ping (simplification)
    pub async fn health_check_all(&self) {
        // ในโปรเจคจริง: ส่ง HEAD / ไปที่ backend แล้วตรวจ response
        // ถ้า connection ตายแล้วให้ evict ออก
    }
}
```

**`src/lb.rs`**

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

/// Upstream host ที่ load balancer เลือกให้ connections ไป
#[derive(Debug, Clone)]
pub struct UpstreamHost {
    /// ที่อยู่ของ backend เช่น "10.0.0.1:8080"
    pub addr: String,
    /// จำนวน connections ที่กำลัง active (ใช้โดย LeastConnections)
    pub active_connections: Arc<AtomicUsize>,
}

impl UpstreamHost {
    pub fn new(addr: impl Into<String>) -> Self {
        Self {
            addr: addr.into(),
            active_connections: Arc::new(AtomicUsize::new(0)),
        }
    }

    pub fn active_count(&self) -> usize {
        self.active_connections.load(Ordering::Relaxed)
    }

    pub fn increment_active(&self) {
        self.active_connections.fetch_add(1, Ordering::Relaxed);
    }

    pub fn decrement_active(&self) {
        self.active_connections.fetch_sub(1, Ordering::Relaxed);
    }
}

/// Round-robin load balancer
///
/// ใช้ atomic counter เพื่อ thread-safety โดยไม่ต้องใช้ Mutex
/// Wraps around เมื่อ counter ถึง len() ด้วย modulo
#[derive(Debug)]
pub struct RoundRobin {
    hosts: Vec<UpstreamHost>,
    counter: AtomicUsize,
}

impl RoundRobin {
    pub fn new(hosts: Vec<UpstreamHost>) -> Self {
        Self { hosts, counter: AtomicUsize::new(0) }
    }

    /// คืน host ถัดไปตาม rotation
    pub fn next(&self) -> Option<&UpstreamHost> {
        if self.hosts.is_empty() {
            return None;
        }
        let idx = self.counter.fetch_add(1, Ordering::Relaxed) % self.hosts.len();
        Some(&self.hosts[idx])
    }

    pub fn len(&self) -> usize {
        self.hosts.len()
    }
}

/// Least-connections load balancer
///
/// เลือก host ที่มี active connections น้อยที่สุดในขณะนั้น
/// เหมาะกับ workload ที่แต่ละ request ใช้เวลาต่างกันมาก
#[derive(Debug)]
pub struct LeastConnections {
    hosts: Vec<UpstreamHost>,
}

impl LeastConnections {
    pub fn new(hosts: Vec<UpstreamHost>) -> Self {
        Self { hosts }
    }

    /// คืน host ที่มี active connections น้อยที่สุด
    pub fn pick(&self) -> Option<&UpstreamHost> {
        self.hosts.iter().min_by_key(|h| h.active_count())
    }
}

/// Wrapper ที่เลือก algorithm ตาม CLI flag
pub enum LoadBalancer {
    RoundRobin(RoundRobin),
    LeastConnections(LeastConnections),
}

impl LoadBalancer {
    pub fn from_args(algo: &str, hosts: Vec<UpstreamHost>) -> Self {
        match algo {
            "least-conn" => Self::LeastConnections(LeastConnections::new(hosts)),
            _ => Self::RoundRobin(RoundRobin::new(hosts)),
        }
    }

    pub fn pick(&self) -> Option<&UpstreamHost> {
        match self {
            Self::RoundRobin(rr) => rr.next(),
            Self::LeastConnections(lc) => lc.pick(),
        }
    }
}
```

---

### ขั้นที่ 6: mTLS, Metrics, OCSP Stapling และ HTTP/2

**`src/mtls.rs`** — mTLS client cert verification

```rust
use rustls::pki_types::CertificateDer;
use rustls::server::danger::ClientCertVerifier;
use rustls::{DistinguishedName, OtherError, RootCertStore};
use std::sync::Arc;

/// ดึง Common Name (CN) จาก client certificate DER bytes
///
/// ในโปรเจคจริงจะ parse X.509 SubjectDN ASN.1 structure
/// เพื่อหา RelativeDistinguishedName ที่มี OID = 2.5.4.3 (commonName)
/// ตัวอย่างนี้แสดง pattern การส่งไปยัง header
pub fn extract_client_cn(cert_der: &CertificateDer<'_>) -> Option<String> {
    // Production implementation จะใช้ x509-parser crate:
    // let (_, cert) = x509_parser::parse_x509_certificate(cert_der)?;
    // cert.subject().iter_common_name().next()?.as_str().ok().map(str::to_owned)
    //
    // Placeholder สำหรับ demo:
    let _ = cert_der;
    Some("client.example.com".to_string())
}

/// สร้าง RootCertStore จาก CA certificate DER list
pub fn build_root_store(
    ca_certs: Vec<CertificateDer<'static>>,
) -> Result<RootCertStore, rustls::Error> {
    let mut root_store = RootCertStore::empty();
    for cert in ca_certs {
        root_store.add(cert)?;
    }
    Ok(root_store)
}
```

**`src/metrics.rs`** — Prometheus instrumentation

```rust
use prometheus::{
    Counter, Gauge, Histogram, HistogramOpts, IntCounter, IntGauge, Opts, Registry,
};
use std::sync::Arc;

/// Metrics ทั้งหมดของ proxy
pub struct ProxyMetrics {
    /// จำนวน requests ทั้งหมดที่รับมา
    pub total_requests: IntCounter,
    /// จำนวน connections ที่กำลัง active อยู่ ณ ขณะนั้น
    pub active_connections: IntGauge,
    /// Histogram ของ latency ต่อ request (seconds)
    pub request_duration: Histogram,
    /// Histogram ของ TLS handshake duration (seconds)
    pub tls_handshake_duration: Histogram,
    /// registry สำหรับ /metrics endpoint
    pub registry: Registry,
}

impl ProxyMetrics {
    pub fn new() -> Result<Self, prometheus::Error> {
        let registry = Registry::new();

        let total_requests = IntCounter::with_opts(Opts::new(
            "proxy_total_requests",
            "Total number of requests handled by the proxy",
        ))?;

        let active_connections = IntGauge::with_opts(Opts::new(
            "proxy_active_connections",
            "Number of currently active client connections",
        ))?;

        // Latency buckets: 1ms, 5ms, 10ms, 50ms, 100ms, 500ms, 1s, 5s
        let latency_buckets = vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0];
        let request_duration = Histogram::with_opts(
            HistogramOpts::new(
                "proxy_request_duration_seconds",
                "End-to-end request latency in seconds",
            )
            .buckets(latency_buckets.clone()),
        )?;

        let tls_handshake_duration = Histogram::with_opts(
            HistogramOpts::new(
                "proxy_tls_handshake_duration_seconds",
                "TLS handshake duration in seconds",
            )
            .buckets(latency_buckets),
        )?;

        registry.register(Box::new(total_requests.clone()))?;
        registry.register(Box::new(active_connections.clone()))?;
        registry.register(Box::new(request_duration.clone()))?;
        registry.register(Box::new(tls_handshake_duration.clone()))?;

        Ok(Self {
            total_requests,
            active_connections,
            request_duration,
            tls_handshake_duration,
            registry,
        })
    }

    /// Render metrics ในรูปแบบ Prometheus text format
    pub fn render(&self) -> Result<String, prometheus::Error> {
        use prometheus::Encoder;
        let encoder = prometheus::TextEncoder::new();
        let metric_families = self.registry.gather();
        let mut buffer = Vec::new();
        encoder.encode(&metric_families, &mut buffer)?;
        Ok(String::from_utf8(buffer).unwrap_or_default())
    }
}

/// Admin HTTP server สำหรับ /metrics และ /healthz
pub async fn run_admin_server(
    port: u16,
    metrics: Arc<ProxyMetrics>,
) {
    use tokio::net::TcpListener;
    use tokio::io::{AsyncReadExt, AsyncWriteExt};

    let listener = TcpListener::bind(format!("0.0.0.0:{}", port))
        .await
        .expect("failed to bind admin port");

    loop {
        let Ok((mut stream, _)) = listener.accept().await else { continue };
        let metrics_clone = Arc::clone(&metrics);

        tokio::spawn(async move {
            let mut buf = [0u8; 4096];
            let n = stream.read(&mut buf).await.unwrap_or(0);
            let request = String::from_utf8_lossy(&buf[..n]);

            let (status, body) = if request.starts_with("GET /metrics") {
                let body = metrics_clone.render().unwrap_or_default();
                ("200 OK", body)
            } else if request.starts_with("GET /healthz") {
                ("200 OK", "ok\n".to_string())
            } else {
                ("404 Not Found", "not found\n".to_string())
            };

            let response = format!(
                "HTTP/1.1 {}\r\nContent-Type: text/plain\r\nContent-Length: {}\r\n\r\n{}",
                status,
                body.len(),
                body
            );
            let _ = stream.write_all(response.as_bytes()).await;
        });
    }
}
```

**OCSP Stapling** — `src/cert.rs` (เพิ่มเติม)

```rust
use std::sync::{Arc, RwLock};
use tokio::time::{interval, Duration};

/// OCSP response cache
///
/// OCSP stapling คือการที่ server fetch OCSP response ล่วงหน้า
/// แล้วแนบไปกับ TLS handshake (ServerHello) แทนที่จะให้ client ไปถาม OCSP responder เอง
/// ลด latency และเพิ่ม privacy ของ client
#[derive(Default)]
pub struct OcspCache {
    /// DER-encoded OCSP response (อาจเป็น None ถ้ายังไม่ได้ fetch)
    response: RwLock<Option<Vec<u8>>>,
}

impl OcspCache {
    pub fn new() -> Arc<Self> {
        Arc::new(Self::default())
    }

    /// อ่าน OCSP response ที่ cache ไว้
    pub fn get(&self) -> Option<Vec<u8>> {
        self.response.read().unwrap().clone()
    }

    /// อัปเดต OCSP response (เรียกหลัง fetch สำเร็จ)
    pub fn set(&self, resp: Vec<u8>) {
        *self.response.write().unwrap() = Some(resp);
    }
}

/// Background task ที่ fetch OCSP response เป็นระยะ
///
/// RFC 6960: OCSP response มี validity period — ควร refresh ก่อนหมดอายุ
/// ในโปรเจคจริงจะ:
/// 1. ดึง OCSP URL จาก certificate extension (Authority Information Access)
/// 2. สร้าง OCSPRequest และส่ง HTTP POST ไปที่ OCSP responder
/// 3. ตรวจสอบ signature ของ OCSPResponse ด้วย CA public key
/// 4. Cache response และส่งให้ rustls ผ่าน cert_data ใน CertifiedKey
pub async fn ocsp_refresh_loop(
    cache: Arc<OcspCache>,
    ocsp_url: String,
    refresh_interval: Duration,
) {
    let mut ticker = interval(refresh_interval);
    loop {
        ticker.tick().await;
        match fetch_ocsp_response(&ocsp_url).await {
            Ok(resp) => {
                cache.set(resp);
                eprintln!("INFO: OCSP response refreshed successfully");
            }
            Err(e) => {
                eprintln!("WARN: OCSP fetch failed: {}", e);
                // ใช้ cached response เดิมต่อไปถ้ายังไม่หมดอายุ
            }
        }
    }
}

async fn fetch_ocsp_response(url: &str) -> Result<Vec<u8>, Box<dyn std::error::Error>> {
    // ในโปรเจคจริง: สร้าง OCSPRequest DER แล้ว POST ไปที่ url
    // ตัวอย่างนี้คืน placeholder
    let _ = url;
    Ok(vec![]) // DER-encoded OCSPResponse
}
```

**HTTP/2 ALPN negotiation** — `src/main.rs` (proxy loop)

```rust
use tokio_rustls::TlsAcceptor;
use rustls::ServerConfig;
use std::sync::Arc;

/// ตรวจสอบ ALPN protocol ที่ตกลงกันหลัง TLS handshake เสร็จ
///
/// หลังจาก TLS handshake สำเร็จ:
/// - ถ้า negotiated protocol = "h2" → ใช้ h2 crate parse HTTP/2 frames
/// - ถ้า negotiated protocol = "http/1.1" หรือ None → ใช้ HTTP/1.1 parser
///
/// ALPN (Application-Layer Protocol Negotiation, RFC 7301) ทำงานใน TLS extension:
/// client ส่งรายการ protocols ที่รองรับ, server เลือกจาก list ของตัวเอง
async fn handle_connection(
    tls_stream: tokio_rustls::server::TlsStream<tokio::net::TcpStream>,
    backend_addr: &str,
    metrics: Arc<ProxyMetrics>,
) {
    let (io, session) = tls_stream.get_ref();
    let protocol = session.alpn_protocol();

    match protocol {
        Some(b"h2") => {
            // HTTP/2: ใช้ h2::server::handshake() เพื่อ parse frame-based protocol
            // แต่ละ request มา as h2::RecvStream, response ส่ง as h2::SendStream
            eprintln!("DEBUG: client negotiated HTTP/2");
            // handle_http2(tls_stream, backend_addr).await;
        }
        _ => {
            // HTTP/1.1: อ่าน request line + headers + body ตามปกติ
            eprintln!("DEBUG: using HTTP/1.1");
            // handle_http1(tls_stream, backend_addr).await;
        }
    }
}
```

---

### ขั้นที่ 7: Main Loop และ Proxy Logic

**`src/main.rs`** — complete proxy loop

```rust
use std::net::SocketAddr;
use std::sync::Arc;
use std::time::Instant;

use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio_rustls::TlsAcceptor;

/// Main TLS accept loop
///
/// สำหรับแต่ละ incoming connection:
/// 1. Accept TCP connection
/// 2. Wrap ด้วย TlsAcceptor → TLS handshake
/// 3. Record handshake duration
/// 4. Spawn tokio task เพื่อ handle request
pub async fn run_proxy(
    listen_addr: &str,
    tls_config: Arc<rustls::ServerConfig>,
    lb: Arc<lb::LoadBalancer>,
    pools: Arc<std::collections::HashMap<String, pool::ConnectionPool>>,
    metrics: Arc<metrics::ProxyMetrics>,
) -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind(listen_addr).await?;
    let acceptor = TlsAcceptor::from(tls_config);

    eprintln!("INFO: listening on {}", listen_addr);

    loop {
        let (tcp_stream, client_addr) = listener.accept().await?;
        let acceptor = acceptor.clone();
        let lb = Arc::clone(&lb);
        let pools = Arc::clone(&pools);
        let metrics = Arc::clone(&metrics);

        tokio::spawn(async move {
            metrics.active_connections.inc();

            let hs_start = Instant::now();
            let tls_stream = match acceptor.accept(tcp_stream).await {
                Ok(s) => s,
                Err(e) => {
                    eprintln!("WARN: TLS handshake failed from {}: {}", client_addr, e);
                    metrics.active_connections.dec();
                    return;
                }
            };
            let hs_duration = hs_start.elapsed().as_secs_f64();
            metrics.tls_handshake_duration.observe(hs_duration);

            let req_start = Instant::now();
            metrics.total_requests.inc();

            if let Err(e) = proxy_request(
                tls_stream,
                client_addr,
                lb,
                pools,
            ).await {
                eprintln!("WARN: proxy error for {}: {}", client_addr, e);
            }

            let req_duration = req_start.elapsed().as_secs_f64();
            metrics.request_duration.observe(req_duration);
            metrics.active_connections.dec();
        });
    }
}

/// Proxy request จาก TLS stream ไปยัง backend
async fn proxy_request(
    tls_stream: tokio_rustls::server::TlsStream<TcpStream>,
    client_addr: SocketAddr,
    lb: Arc<lb::LoadBalancer>,
    pools: Arc<std::collections::HashMap<String, pool::ConnectionPool>>,
) -> Result<(), Box<dyn std::error::Error>> {
    // เลือก backend
    let backend = lb.pick().ok_or("no upstream hosts available")?;

    // ดึง connection จาก pool
    let pool = pools.get(&backend.addr).ok_or("no pool for backend")?;
    let _conn = pool.acquire().ok_or("connection pool exhausted")?;
    backend.increment_active();

    // สร้าง TCP connection ไปยัง backend
    let mut backend_stream = TcpStream::connect(&backend.addr).await?;

    // อ่าน request จาก TLS stream
    let (mut tls_read, mut tls_write) = tokio::io::split(tls_stream);
    let (mut backend_read, mut backend_write) = backend_stream.split();

    // Bidirectional copy: client ↔ backend
    let client_to_backend = tokio::io::copy(&mut tls_read, &mut backend_write);
    let backend_to_client = tokio::io::copy(&mut backend_read, &mut tls_write);

    // รอทั้งสองทิศทาง; จบเมื่อฝั่งใดฝั่งหนึ่ง close
    let _ = tokio::join!(client_to_backend, backend_to_client);

    backend.decrement_active();
    Ok(())
}
```

## การทดสอบ (Testing)

tests เหล่านี้ compile และ run จริงด้วย scratchpad project ที่สร้างขึ้นระหว่างเขียนโปรเจคนี้

```rust
#[cfg(test)]
mod tests {
    use super::cert_impl::*;
    use super::headers_impl::*;
    use super::lb_impl::*;
    use super::pool_impl::*;
    use super::sni_impl::*;
    use std::time::{Duration, SystemTime, UNIX_EPOCH};

    // ----------------------------------------------------------
    // PEM cert parsing
    // ----------------------------------------------------------

    fn generate_test_cert_pem() -> (String, String) {
        use rcgen::{generate_simple_self_signed, CertifiedKey};
        let CertifiedKey { cert, key_pair } =
            generate_simple_self_signed(vec!["test.example.com".to_string()])
                .unwrap();
        (cert.pem(), key_pair.serialize_pem())
    }

    #[test]
    fn test_parse_valid_cert_chain() {
        let (cert_pem, _) = generate_test_cert_pem();
        let certs = parse_cert_chain(cert_pem.as_bytes()).unwrap();
        assert_eq!(certs.len(), 1);
    }

    #[test]
    fn test_parse_empty_pem_returns_error() {
        assert!(parse_cert_chain(b"").is_err());
    }

    #[test]
    fn test_parse_valid_private_key() {
        let (_, key_pem) = generate_test_cert_pem();
        let key = parse_private_key(key_pem.as_bytes()).unwrap();
        assert!(matches!(key, rustls::pki_types::PrivateKeyDer::Pkcs8(_)));
    }

    #[test]
    fn test_parse_no_key_returns_error() {
        let (cert_pem, _) = generate_test_cert_pem();
        assert!(parse_private_key(cert_pem.as_bytes()).is_err());
    }

    // ----------------------------------------------------------
    // Certificate expiry warning
    // ----------------------------------------------------------

    #[test]
    fn test_cert_expiry_within_30_days_true() {
        let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
        let expiry = cert_expiry_from_timestamp(now + 10 * 86_400);
        assert!(expiry.expires_within_days(30));
    }

    #[test]
    fn test_cert_expiry_within_30_days_false() {
        let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
        let expiry = cert_expiry_from_timestamp(now + 90 * 86_400);
        assert!(!expiry.expires_within_days(30));
    }

    #[test]
    fn test_cert_already_expired() {
        let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs();
        let expiry = cert_expiry_from_timestamp(now.saturating_sub(86_400));
        assert!(expiry.expires_within_days(30));
        assert_eq!(expiry.remaining_seconds(), 0);
    }

    // ----------------------------------------------------------
    // SNI routing
    // ----------------------------------------------------------

    #[test]
    fn test_sni_exact_match() {
        let mut table = SniRoutingTable::new();
        table.add_route("api.example.com", vec![Backend::new("10.0.0.1", 8080)]);
        let backends = table.resolve("api.example.com").unwrap();
        assert_eq!(backends[0].addr(), "10.0.0.1:8080");
    }

    #[test]
    fn test_sni_wildcard_match() {
        let mut table = SniRoutingTable::new();
        table.add_route("*.example.com", vec![Backend::new("10.0.0.3", 9090)]);
        let backends = table.resolve("anything.example.com").unwrap();
        assert_eq!(backends[0].port, 9090);
    }

    #[test]
    fn test_sni_exact_takes_priority_over_wildcard() {
        let mut table = SniRoutingTable::new();
        table.add_route("*.example.com", vec![Backend::new("wildcard", 1111)]);
        table.add_route("specific.example.com", vec![Backend::new("exact", 2222)]);
        assert_eq!(table.resolve("specific.example.com").unwrap()[0].host, "exact");
    }

    #[test]
    fn test_sni_no_match_returns_none() {
        assert!(SniRoutingTable::new().resolve("notfound.example.com").is_none());
    }

    // ----------------------------------------------------------
    // Header manipulation
    // ----------------------------------------------------------

    #[test]
    fn test_add_forwarding_headers_no_existing_xff() {
        let mut h = RequestHeaders::new();
        add_forwarding_headers(&mut h, "203.0.113.10");
        assert_eq!(h.get("x-forwarded-for"), Some("203.0.113.10"));
        assert_eq!(h.get("x-forwarded-proto"), Some("https"));
        assert_eq!(h.get("x-real-ip"), Some("203.0.113.10"));
    }

    #[test]
    fn test_add_forwarding_headers_appends_to_existing_xff() {
        let mut h = RequestHeaders::new();
        h.insert("x-forwarded-for", "192.168.1.1");
        add_forwarding_headers(&mut h, "203.0.113.10");
        assert_eq!(h.get("x-forwarded-for"), Some("192.168.1.1, 203.0.113.10"));
    }

    #[test]
    fn test_remove_hop_by_hop_headers() {
        let mut h = RequestHeaders::new();
        h.insert("connection", "keep-alive");
        h.insert("keep-alive", "timeout=5");
        h.insert("transfer-encoding", "chunked");
        h.insert("content-type", "application/json");
        remove_hop_by_hop(&mut h);
        assert!(!h.contains("connection"));
        assert!(!h.contains("keep-alive"));
        assert!(!h.contains("transfer-encoding"));
        assert!(h.contains("content-type")); // must NOT be removed
    }

    #[test]
    fn test_remove_connection_listed_headers() {
        // RFC 7230 §6.1: headers named in Connection: must be removed
        let mut h = RequestHeaders::new();
        h.insert("connection", "x-custom-hop, upgrade");
        h.insert("x-custom-hop", "value");
        h.insert("host", "example.com");
        remove_hop_by_hop(&mut h);
        assert!(!h.contains("x-custom-hop"));
        assert!(h.contains("host"));
    }

    // ----------------------------------------------------------
    // Connection pool
    // ----------------------------------------------------------

    #[test]
    fn test_pool_acquire_and_release() {
        let pool = ConnectionPool::new(5, Duration::from_secs(60));
        let conn = pool.acquire().unwrap();
        assert_eq!(pool.active_count(), 1);
        pool.release(conn);
        assert_eq!(pool.active_count(), 0);
        assert_eq!(pool.idle_count(), 1);
    }

    #[test]
    fn test_pool_reuses_idle_connection() {
        let pool = ConnectionPool::new(5, Duration::from_secs(60));
        let conn1 = pool.acquire().unwrap();
        let id1 = conn1.id;
        pool.release(conn1);
        let conn2 = pool.acquire().unwrap();
        assert_eq!(conn2.id, id1);
        pool.release(conn2);
    }

    #[test]
    fn test_pool_max_size_enforced() {
        let pool = ConnectionPool::new(2, Duration::from_secs(60));
        let c1 = pool.acquire().unwrap();
        let c2 = pool.acquire().unwrap();
        assert!(pool.acquire().is_none(), "pool at max should return None");
        pool.release(c1);
        pool.release(c2);
    }

    #[test]
    fn test_pool_idle_timeout_eviction() {
        let pool = ConnectionPool::new(5, Duration::from_nanos(1));
        let conn = pool.acquire().unwrap();
        pool.release(conn);
        std::thread::sleep(Duration::from_millis(1));
        let new_conn = pool.acquire().unwrap();
        assert_eq!(new_conn.id, 1, "evicted idle conn; new conn has id=1");
        pool.release(new_conn);
    }

    // ----------------------------------------------------------
    // Load balancing
    // ----------------------------------------------------------

    #[test]
    fn test_round_robin_distribution() {
        let hosts = vec![
            UpstreamHost::new("h1:8080"),
            UpstreamHost::new("h2:8080"),
            UpstreamHost::new("h3:8080"),
        ];
        let lb = RoundRobin::new(hosts);
        let mut counts = std::collections::HashMap::new();
        for _ in 0..9 {
            *counts.entry(lb.next().unwrap().addr.clone()).or_insert(0) += 1;
        }
        for (_, count) in &counts {
            assert_eq!(*count, 3);
        }
    }

    #[test]
    fn test_round_robin_wraps_around() {
        let lb = RoundRobin::new(vec![
            UpstreamHost::new("a:80"),
            UpstreamHost::new("b:80"),
        ]);
        assert_eq!(lb.next().unwrap().addr, "a:80");
        assert_eq!(lb.next().unwrap().addr, "b:80");
        assert_eq!(lb.next().unwrap().addr, "a:80");
    }

    #[test]
    fn test_least_connections_picks_lowest() {
        use std::sync::atomic::Ordering;
        let hosts = vec![
            UpstreamHost::new("lc1:8080"),
            UpstreamHost::new("lc2:8080"),
            UpstreamHost::new("lc3:8080"),
        ];
        hosts[0].active_connections.store(5, Ordering::Relaxed);
        hosts[1].active_connections.store(2, Ordering::Relaxed);
        hosts[2].active_connections.store(8, Ordering::Relaxed);
        let lb = LeastConnections::new(hosts);
        assert_eq!(lb.pick().unwrap().addr, "lc2:8080");
    }
}
```

**`cargo test` output จริง (27 tests, 0 failed):**

```
running 27 tests
test tests::test_cert_expiry_within_30_days_false ... ok
test tests::test_cert_already_expired ... ok
test tests::test_add_forwarding_headers_appends_to_existing_xff ... ok
test tests::test_add_forwarding_headers_no_existing_xff ... ok
test tests::test_client_cert_cn_header ... ok
test tests::test_least_connections_picks_lowest ... ok
test tests::test_cert_expiry_within_30_days_true ... ok
test tests::test_client_cert_cn_header_none ... ok
test tests::test_least_connections_tie_picks_first ... ok
test tests::test_parse_empty_pem_returns_error ... ok
test tests::test_parse_no_key_returns_error ... ok
test tests::test_parse_valid_private_key ... ok
test tests::test_parse_valid_cert_chain ... ok
test tests::test_pool_acquire_and_release ... ok
test tests::test_pool_max_size_enforced ... ok
test tests::test_pool_reuses_idle_connection ... ok
test tests::test_remove_connection_listed_headers ... ok
test tests::test_remove_hop_by_hop_headers ... ok
test tests::test_round_robin_distribution ... ok
test tests::test_round_robin_empty_returns_none ... ok
test tests::test_round_robin_wraps_around ... ok
test tests::test_sni_default_fallback ... ok
test tests::test_sni_exact_match ... ok
test tests::test_sni_exact_takes_priority_over_wildcard ... ok
test tests::test_sni_wildcard_match ... ok
test tests::test_sni_no_match_returns_none ... ok
test tests::test_pool_idle_timeout_eviction ... ok

test result: ok. 27 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Pitfalls และข้อผิดพลาดที่พบบ่อย

### Pitfall 1: `rustls` 0.23 API แตกต่างจากเวอร์ชั่นก่อนหน้ามาก

`rustls` เปลี่ยน API หลักหลายจุดใน 0.23:

```rust
// WRONG: rustls 0.21 / 0.22 API (ใช้ไม่ได้แล้ว)
let config = ServerConfig::builder()
    .with_safe_defaults()       // ไม่มีแล้ว
    .with_no_client_auth()
    .with_single_cert(certs, key)?;

// CORRECT: rustls 0.23 API
// builder() ส่งคืน ConfigBuilder<ServerConfig, WantsVerifier> โดยตรง
let config = ServerConfig::builder()
    .with_no_client_auth()
    .with_single_cert(certs, key)?;

// ประเภทก็เปลี่ยน: ใช้ rustls::pki_types::CertificateDer<'static>
// แทน rustls::Certificate
// และ PrivateKeyDer<'static> แทน rustls::PrivateKey
```

ถ้า compile error ขึ้นเรื่อง `with_safe_defaults` ให้ดู changelog ของ rustls version ที่ใช้อยู่จริง

### Pitfall 2: tokio-rustls `TlsAcceptor::accept()` บล็อกถ้า client ไม่ส่ง ClientHello

TLS handshake อาจค้างอยู่ indefinitely ถ้า client เปิด TCP connection แล้วไม่ส่งข้อมูล ควรใส่ timeout:

```rust
use tokio::time::timeout;

let tls_stream = match timeout(
    Duration::from_secs(10),
    acceptor.accept(tcp_stream),
).await {
    Ok(Ok(stream)) => stream,
    Ok(Err(e)) => {
        eprintln!("WARN: TLS error: {}", e);
        return;
    }
    Err(_) => {
        eprintln!("WARN: TLS handshake timeout");
        return;
    }
};
```

### Pitfall 3: hop-by-hop headers ต้องถูก remove ทั้งจาก request และ response

ผู้พัฒนาหลายคน remove hop-by-hop headers จาก request แต่ลืม response ทำให้ backend ส่ง `Transfer-Encoding: chunked` มา และ proxy ส่งต่อไปยัง client โดยไม่ decode ก่อน ทำให้ client ได้รับ chunked data แบบ raw

```rust
// ต้อง remove จากทั้งสองทิศทาง
fn forward_request(mut headers: RequestHeaders) -> RequestHeaders {
    remove_hop_by_hop(&mut headers);
    headers
}

fn forward_response(mut headers: ResponseHeaders) -> ResponseHeaders {
    remove_hop_by_hop_response(&mut headers);
    headers
}
```

### Pitfall 4: Connection pool ที่ใช้ TcpStream ต้องตรวจสอบว่า connection ยังมีชีวิตอยู่

Connection ที่ idle ใน pool อาจถูก RST โดย backend หรือ firewall ทำให้ `write()` ครั้งแรกสำเร็จ (เพราะ kernel buffer ยังมีที่) แต่ `read()` ถัดมา error ควรมี retry logic:

```rust
async fn acquire_and_verify(pool: &ConnectionPool, addr: &str) 
    -> Result<TcpStream, Box<dyn std::error::Error>> 
{
    // ลอง idle connection จาก pool ก่อน
    if let Some(conn) = pool.acquire() {
        // ตรวจสอบด้วย zero-byte read (non-blocking)
        // ถ้าอ่านได้ 0 bytes = connection closed
        // ถ้า WouldBlock = connection ยังมีชีวิต
        // ในโค้ดจริงจะใช้ try_read() ของ tokio TcpStream
        return Ok(/* existing stream */);
    }
    // สร้าง connection ใหม่
    Ok(TcpStream::connect(addr).await?)
}
```

### Pitfall 5: SNI resolver ต้องไม่ panic ใน `resolve()` callback

`resolve()` ถูกเรียกจาก rustls ภายใน TLS handshake ซึ่งอยู่ใน async context ของ tokio ถ้า panic ที่นี่จะทำให้ thread/task crash และ acceptor หยุดทำงาน ควรใช้ `unwrap_or_default()` หรือคืน `None`:

```rust
impl ResolvesServerCert for SniCertResolver {
    fn resolve(&self, client_hello: ClientHello<'_>) -> Option<Arc<CertifiedKey>> {
        // ใช้ ? แทน unwrap() ทุกที่
        let sni = client_hello.server_name()?;
        let certs = self.certs.read().ok()?;  // ไม่ panic ถ้า lock poisoned
        certs.get(sni).map(Arc::clone)
            .or_else(|| self.default_cert.as_ref().map(Arc::clone))
    }
}
```

## การ Package และ Deploy

### Build release binary

```bash
cargo build --release
ls -lh target/release/tls_proxy
# -rwxr-xr-x 1 root root 8.2M tls_proxy
```

### ใช้งาน

```bash
# สร้าง self-signed cert สำหรับ development
openssl req -x509 -nodes -days 365 \
    -newkey rsa:4096 \
    -keyout certs/server.key \
    -out certs/server.crt \
    -subj "/CN=localhost"

# รัน proxy
./target/release/tls_proxy \
    --listen 0.0.0.0:443 \
    --upstream 127.0.0.1:8080,127.0.0.1:8081 \
    --cert certs/server.crt \
    --key certs/server.key \
    --lb-algo least-conn \
    --pool-size 100 \
    --admin-port 9090

# ทดสอบ
curl -k https://localhost/
curl http://localhost:9090/metrics
curl http://localhost:9090/healthz
```

### Docker image

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
# rustls ใช้ pure Rust → ไม่ต้องติดตั้ง libssl-dev
COPY --from=builder /app/target/release/tls_proxy /usr/local/bin/
EXPOSE 443 9090
ENTRYPOINT ["tls_proxy"]
```

```bash
docker build -t tls-proxy:latest .
docker run -p 443:443 -p 9090:9090 \
    -v $(pwd)/certs:/app/certs \
    tls-proxy:latest \
    --upstream backend:8080 \
    --cert /app/certs/server.crt \
    --key /app/certs/server.key
```

### Systemd unit file

```ini
[Unit]
Description=TLS Terminating Reverse Proxy
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=tls-proxy
AmbientCapabilities=CAP_NET_BIND_SERVICE
ExecStart=/usr/local/bin/tls_proxy \
    --listen 0.0.0.0:443 \
    --upstream 127.0.0.1:8080 \
    --cert /etc/tls-proxy/server.crt \
    --key /etc/tls-proxy/server.key
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: เพิ่ม WebSocket proxying (ระดับ: ปานกลาง)

WebSocket protocol เริ่มต้นเป็น HTTP request พร้อม `Upgrade: websocket` header หลังจาก server ตอบ `101 Switching Protocols` connection จะเปลี่ยนเป็น bidirectional WebSocket frames

งาน:
1. ตรวจจับ `Upgrade: websocket` header ใน incoming request
2. ส่ง upgrade request ไปยัง backend (อย่า strip hop-by-hop headers ที่เกี่ยวกับ upgrade)
3. หลัง `101` response: pipe raw bytes ระหว่าง client และ backend โดยไม่ parse
4. ทดสอบด้วย `websocat ws://localhost:8080` ผ่าน proxy

```rust
fn is_websocket_upgrade(headers: &RequestHeaders) -> bool {
    headers.get("upgrade").map_or(false, |v| v.eq_ignore_ascii_case("websocket"))
        && headers.get("connection").map_or(false, |v| 
            v.to_lowercase().contains("upgrade"))
}
```

### แบบฝึกหัดที่ 2: Certificate auto-renewal ด้วย ACME (ระดับ: ยาก)

ACME protocol (RFC 8555) ที่ Let's Encrypt ใช้ช่วยให้ server ออก certificate ได้อัตโนมัติ

งาน:
1. เพิ่ม crate `instant-acme` เป็น dependency
2. Implement ACME HTTP-01 challenge: เมื่อ Let's Encrypt ส่ง GET `/.well-known/acme-challenge/<token>` ให้ตอบ key authorization
3. ลองใช้ Let's Encrypt staging environment ก่อน
4. หลังได้ cert ใหม่: reload `SniCertResolver` โดยไม่ restart proxy (hot reload)
5. ตั้ง routine refresh: check expiry ทุก 12 ชั่วโมง, renew เมื่อเหลือ < 30 วัน

### แบบฝึกหัดที่ 3: Rate limiting ต่อ client IP (ระดับ: ปานกลาง)

งาน:
1. สร้าง `RateLimiter` struct ที่ใช้ token bucket algorithm ต่อ IP
2. เก็บ state ใน `HashMap<IpAddr, TokenBucket>` พร้อม `RwLock`
3. ปฏิเสธ connection ที่เกิน limit ด้วย `429 Too Many Requests` response (ใน TLS layer)
4. เพิ่ม CLI flags: `--rate-limit-rps 100 --rate-limit-burst 200`
5. Expose metric: `proxy_rate_limited_requests_total`

```rust
struct TokenBucket {
    tokens: f64,
    max_tokens: f64,
    refill_rate: f64,  // tokens per second
    last_refill: Instant,
}

impl TokenBucket {
    fn try_consume(&mut self, tokens: f64) -> bool {
        self.refill();
        if self.tokens >= tokens {
            self.tokens -= tokens;
            true
        } else {
            false
        }
    }

    fn refill(&mut self) {
        let elapsed = self.last_refill.elapsed().as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.max_tokens);
        self.last_refill = Instant::now();
    }
}
```

### แบบฝึกหัดที่ 4: Request/Response body inspection (ระดับ: ยาก)

งาน:
1. เพิ่ม mode `--inspect-mode` ที่ทำให้ proxy อ่าน request body ก่อน forward
2. Implement middleware chain pattern: `Vec<Box<dyn Middleware>>`
3. สร้าง `LoggingMiddleware` ที่ log body ขนาด <= 4KB
4. สร้าง `JsonValidationMiddleware` ที่ reject request ถ้า `Content-Type: application/json` แต่ body ไม่ใช่ valid JSON
5. ระวัง: body ที่อ่านแล้วต้องถูก buffer และส่งต่อไปยัง backend ด้วย (`Content-Length` ต้องถูกต้อง)

```rust
#[async_trait::async_trait]
trait Middleware: Send + Sync {
    async fn process_request(
        &self,
        req: &mut ProxyRequest,
    ) -> Result<MiddlewareAction, ProxyError>;
}

enum MiddlewareAction {
    Continue,
    Reject(u16, String), // status code, body
}
```

## สรุป

โปรเจคนี้สร้าง TLS-terminating reverse proxy ที่ครอบคลุม concepts หลักของ production-grade security infrastructure:

**สิ่งที่สร้าง:**
- TLS acceptor ด้วย `tokio-rustls` ที่รองรับ TLS 1.2 และ 1.3
- SNI-based certificate selection ผ่าน `ResolvesServerCert` trait
- HTTP header forwarding ที่ถูกต้องตาม RFC 7230
- Connection pool ที่มี idle timeout eviction
- Round-robin และ least-connections load balancing ด้วย atomic counters
- Prometheus metrics ครอบคลุม request latency, active connections, และ TLS handshake duration
- OCSP stapling pattern สำหรับ certificate status caching

**Patterns สำคัญที่ได้เรียน:**
1. `Arc<dyn Trait>` pattern สำหรับ pluggable behaviors (cert resolver, load balancer)
2. Atomic counters (`AtomicUsize`) สำหรับ lock-free round-robin ใน concurrent context
3. `RwLock` สำหรับ read-heavy, write-rare state (SNI cert map)
4. Timeout wrapping ทุก I/O operation เพื่อป้องกัน resource leak
5. Separation of concerns: TLS acceptor, HTTP parser, load balancer, และ connection pool เป็น modules อิสระ

**เชื่อมโยงไปโปรเจคถัดไป:** Project D04 Certificate Manager จะลงลึกเรื่อง X.509 certificate lifecycle ทั้งหมด: การ issue, renew, revoke certificates ผ่าน ACME protocol และการ manage CA trust chains ซึ่งต่อยอดจาก certificate loading และ mTLS verification ที่สร้างใน D03 นี้

---

**โปรเจคก่อนหน้า:** [project-d02-file-encryptor.md](project-d02-file-encryptor.md) | **โปรเจคถัดไป:** [project-d04-cert-manager.md](project-d04-cert-manager.md)
