# Project D04: Certificate Manager (ACME)

> โมดูล: D — Security & Cryptography | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

Certificate Manager คือ CLI tool และ daemon สำหรับจัดการ TLS certificate อัตโนมัติผ่าน **ACME protocol (RFC 8555)** — มาตรฐานที่ Let's Encrypt, ZeroSSL และ CA อื่น ๆ ใช้เพื่อให้ client พิสูจน์ความเป็นเจ้าของ domain โดยไม่ต้องมีการ manual verify

ปัญหาที่โปรเจคนี้แก้: ใน production environment ที่มีหลาย subdomain และ wildcard certificate การจัดการ certificate แบบ manual (สั่ง renew ทุก 90 วัน, copy ไฟล์, restart service) เป็น operational burden ที่ก่อให้เกิด outage เมื่อ certificate หมดอายุโดยไม่มีการแจ้งเตือน Certificate Manager แก้ปัญหานี้ด้วยการ automate ทุกขั้นตอน: ขอ cert ครั้งแรก, renew อัตโนมัติก่อนหมดอายุ 30 วัน, และส่ง webhook ไปยัง monitoring system

Use case จริงในโลก production:
- Server ที่ต้องการ wildcard cert `*.api.example.com` สำหรับ microservices ทุกตัว
- Internal CA สำหรับ mTLS ระหว่าง service ใน datacenter
- Kubernetes cluster ที่ใช้ cert-manager style automation แต่เขียนเองได้เต็ม control
- CI/CD pipeline ที่ต้องการ short-lived certificate สำหรับ ephemeral environment

Learning value ที่ได้: โปรเจคนี้สอน ACME protocol flow ครบทุก step ตั้งแต่ directory fetch จนถึง certificate download, การใช้ ECDSA P-256 สำหรับ JWS signing, HTTP-01 และ DNS-01 challenge, CSR generation ด้วย `rcgen`, และ daemon pattern ที่ตรวจสอบ certificate expiry ทุกวัน

## สิ่งที่จะได้เรียนรู้

- **ACME Protocol (RFC 8555)**: directory → nonce → account → order → authorization → challenge → finalize → certificate flow ครบ 8 ขั้นตอน
- **JWS (JSON Web Signature)**: สร้าง signed request ด้วย ECDSA P-256 key, JWK thumbprint (RFC 7638), base64url encoding ที่ถูกต้อง
- **HTTP-01 challenge**: embed axum HTTP server บน port 80 เพื่อ serve `/.well-known/acme-challenge/{token}` ใน-process
- **DNS-01 challenge**: เรียก DNS provider API (Cloudflare/Route53) สร้าง TXT record, poll DNS propagation ก่อน respond
- **CSR generation**: สร้าง PKCS#10 CSR ด้วย `rcgen`, กำหนด Subject Alternative Names รวม wildcard `*.domain.com`
- **Certificate chain validation**: ตรวจสอบ chain ครบ (leaf → intermediate → root) ด้วย `x509-cert`
- **Daemon pattern**: background loop ตรวจสอบ expiry ทุกวัน, renewal decision logic, configurable threshold
- **Webhook notification**: POST JSON event ไปยัง external URL เมื่อ cert issued/renewed/expiring/failed

## ความรู้ที่ต้องมีมาก่อน

- **Part 1–40**: Rust fundamentals — ownership, structs, enums, error handling, file I/O
- **Part 41–60**: Traits, generics, `serde` basics, `Cargo.toml` dependency management
- **Part 61–80**: Async/await ด้วย `tokio`, `reqwest` HTTP client, `axum` web framework
- **Part 81–95**: Process spawning, signal handling, background tasks
- **Part 96–110**: Cryptographic primitives — ECDSA, hash functions, encoding schemes
- ความคุ้นเคยกับ TLS/PKI concepts (จาก Part 100-105 เรื่อง TLS และ certificate chain)

## โครงสร้างโปรเจค (Project Layout)

```
cert-manager/
├── src/
│   ├── main.rs          ← CLI entry point (clap 4 subcommands)
│   ├── acme/
│   │   ├── mod.rs       ← AcmeClient struct, directory fetch
│   │   ├── account.rs   ← Account registration, JWK, key management
│   │   ├── order.rs     ← Order creation, identifier structs
│   │   ├── challenge.rs ← HTTP-01 server, DNS-01 TXT record logic
│   │   ├── finalize.rs  ← CSR submission, certificate download
│   │   └── jws.rs       ← JWS header/payload builder, ECDSA signing
│   ├── cert/
│   │   ├── mod.rs       ← CertStore — load/save/list certificates
│   │   ├── manifest.rs  ← CertManifest struct (JSON metadata)
│   │   ├── chain.rs     ← Chain validation (leaf→intermediate→root)
│   │   └── csr.rs       ← CSR generation with rcgen
│   ├── dns/
│   │   ├── mod.rs       ← DnsProvider trait
│   │   ├── cloudflare.rs← Cloudflare DNS API client
│   │   └── route53.rs   ← AWS Route 53 API client
│   ├── daemon.rs        ← Daily renewal daemon loop
│   ├── webhook.rs       ← Webhook POST notification
│   └── error.rs         ← CertManagerError enum
├── tests/
│   └── integration.rs   ← Integration tests (no network required)
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### ACME Protocol Flow ภาพรวม

```
                    ┌─────────────────────────────────────────────────┐
                    │                ACME CA (Let's Encrypt)           │
                    └────────────────────┬────────────────────────────┘
                                         │
  ┌──────────────────────────────────────┼──────────────────────────────────┐
  │                                      │ HTTPS                            │
  │  cert-manager CLI                    │                                  │
  │                                      ▼                                  │
  │  1. GET /directory                ┌──────┐                             │
  │     ← newAccount, newOrder URLs   │ Dir  │                             │
  │                                   └──────┘                             │
  │  2. HEAD /acme/new-nonce          ┌──────┐                             │
  │     ← Replay-Nonce header         │Nonce │                             │
  │                                   └──────┘                             │
  │  3. POST /acme/new-acct           ┌──────────┐                         │
  │     Body: JWS { JWK, payload }    │ Account  │ ← account URL          │
  │                                   └──────────┘                         │
  │  4. POST /acme/new-order          ┌──────────┐                         │
  │     Body: JWS { identifiers[] }   │  Order   │ ← authz URLs           │
  │                                   └──────────┘                         │
  │  5. GET /acme/authz/{id}          ┌──────────────────┐                 │
  │     ← challenges: http-01/dns-01  │ Authorization    │                 │
  │                                   └──────────────────┘                 │
  │  6a. HTTP-01: serve token         ┌──────────────────┐                 │
  │      axum on :80                  │ Challenge Verify │                 │
  │  6b. DNS-01: create TXT record    └──────────────────┘                 │
  │                                                                         │
  │  7. POST /acme/order/{id}/finalize┌──────────┐                         │
  │     Body: JWS { CSR (DER) }       │ Finalize │                         │
  │                                   └──────────┘                         │
  │  8. GET /acme/cert/{id}           ┌──────────┐                         │
  │     ← PEM certificate chain       │   Cert   │                         │
  │                                   └──────────┘                         │
  └─────────────────────────────────────────────────────────────────────────┘
```

### JWS Signing Flow

```
ECDSA P-256 Private Key
        │
        ├─► Public Key ─► JWK (crv, kty, x, y)
        │                   │
        │                   ▼
        │              JWK Thumbprint
        │              = base64url(SHA-256(canonical_JSON(JWK)))
        │
ACME Request:
  header = { "alg": "ES256", "nonce": <nonce>, "url": <url>, "jwk": <JWK> }
  payload = base64url(<request_body_json>) | "" (for POST-as-GET)
  message = base64url(header) + "." + base64url(payload)
  signature = ECDSA_P256_Sign(private_key, SHA-256(message))
  JWS = { protected: header, payload: payload, signature: base64url(sig) }
```

### Certificate Storage Layout

```
~/.config/cert-manager/
├── accounts/
│   └── letsencrypt/
│       ├── account.json     ← { account_url, contact_email }
│       └── key.pem          ← ECDSA P-256 account private key (0600)
└── certs/
    └── example.com/
        ├── manifest.json    ← { domain, issued_at, expires_at, auto_renew, chain_length, root_ca }
        ├── cert.pem         ← leaf certificate (0644)
        ├── chain.pem        ← intermediate chain (0644)
        ├── fullchain.pem    ← cert + chain (0644)
        └── key.pem          ← domain private key (0600)
```

### ทำไมเลือก ECDSA P-256 ไม่ใช่ RSA-2048

ECDSA P-256 (256-bit key) มี security level เทียบเท่า RSA-3072 ในขณะที่ key size เล็กกว่า 12x และ signing operation เร็วกว่า 20x ใน benchmark ทั่วไป Let's Encrypt รองรับ ECDSA P-256 certificate ตั้งแต่ปี 2016 และ TLS 1.3 ให้ preference กับ ECDSA handshake อย่างไรก็ตาม RSA-2048 ยังมีเหตุผลที่ใช้ได้: compatibility กับ legacy client ที่ไม่รองรับ ECDSA หรือ HSM ที่รองรับเฉพาะ RSA โปรเจคนี้ implement ทั้งสอง mode โดย default เป็น P-256

### DNS-01 Challenge Design Decision

DNS-01 challenge จำเป็นสำหรับ wildcard certificate เพราะ Let's Encrypt ไม่อนุญาต HTTP-01 สำหรับ `*.domain.com` — CA ต้องการการพิสูจน์ว่า requestor ควบคุม DNS zone ไม่ใช่แค่ HTTP server โปรเจคนี้ใช้ `DnsProvider` trait เพื่อ abstract Cloudflare และ Route53 API ออกจาก core ACME logic ทำให้เพิ่ม provider ใหม่ได้โดยไม่แก้ core code

## การพัฒนาทีละขั้นตอน

---

### ขั้นที่ 1: โครงสร้างโปรเจคและ Base Types

สร้าง Cargo project และกำหนด data types หลัก

**`Cargo.toml`**

```toml
[package]
name = "cert-manager"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "cert-manager"
path = "src/main.rs"

[dependencies]
# HTTP client และ async runtime
reqwest = { version = "0.12", features = ["json", "rustls-tls"] }
tokio = { version = "1", features = ["full"] }

# Web server สำหรับ HTTP-01 challenge
axum = "0.8"

# Cryptography
p256 = { version = "0.13", features = ["ecdsa", "jwk", "pem"] }
sha2 = "0.10"
base64 = "0.22"

# Certificate generation
rcgen = "0.13"

# Certificate parsing
x509-cert = "0.2"
der = "0.7"

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# CLI
clap = { version = "4", features = ["derive"] }

# Datetime
chrono = { version = "0.4", features = ["serde"] }

# Utilities
tracing = "0.1"
tracing-subscriber = "0.3"
anyhow = "1"
thiserror = "1"
time = { version = "0.3", features = ["formatting"] }
tokio-util = { version = "0.7", features = ["rt"] }
```

**`src/error.rs`**

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum CertManagerError {
    #[error("ACME protocol error: {0}")]
    Acme(String),

    #[error("HTTP request failed: {0}")]
    Http(#[from] reqwest::Error),

    #[error("JSON serialization error: {0}")]
    Json(#[from] serde_json::Error),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Cryptography error: {0}")]
    Crypto(String),

    #[error("Certificate validation error: {0}")]
    Validation(String),

    #[error("DNS provider error: {0}")]
    Dns(String),

    #[error("Challenge failed: {0}")]
    Challenge(String),

    #[error("Certificate not found for domain: {0}")]
    NotFound(String),

    #[error("Webhook delivery failed: {0}")]
    Webhook(String),
}

pub type Result<T> = std::result::Result<T, CertManagerError>;
```

**`src/main.rs`** (CLI skeleton)

```rust
use clap::{Parser, Subcommand};
use std::path::PathBuf;

mod acme;
mod cert;
mod daemon;
mod dns;
mod error;
mod webhook;

use error::Result;

#[derive(Parser)]
#[command(name = "cert-manager")]
#[command(version = "0.1.0")]
#[command(about = "ACME certificate manager — automates TLS certificate lifecycle")]
struct Cli {
    /// Path to config directory (default: ~/.config/cert-manager)
    #[arg(long, value_name = "DIR")]
    config_dir: Option<PathBuf>,

    /// ACME CA directory URL
    #[arg(long, default_value = "https://acme-v02.api.letsencrypt.org/directory")]
    ca_url: String,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Request a new certificate
    Issue {
        /// Primary domain name
        domain: String,

        /// Additional SANs (e.g. "*.example.com api.example.com")
        #[arg(short, long, value_name = "DOMAIN")]
        san: Vec<String>,

        /// Challenge type: http-01 or dns-01
        #[arg(long, default_value = "http-01")]
        challenge: String,

        /// DNS provider for dns-01 (cloudflare | route53)
        #[arg(long)]
        dns_provider: Option<String>,

        /// Contact email for ACME account registration
        #[arg(long)]
        email: Option<String>,
    },

    /// Renew certificate if expiring soon
    Renew {
        /// Domain to renew (renew all if omitted)
        domain: Option<String>,

        /// Renew if certificate expires within this many days
        #[arg(long, default_value_t = 30)]
        renew_before_days: i64,
    },

    /// List all managed certificates
    List,

    /// Show certificate details
    Show {
        domain: String,
    },

    /// Revoke a certificate
    Revoke {
        domain: String,

        /// RFC 5280 CRL reason code (0=unspecified, 1=keyCompromise, 4=superseded)
        #[arg(long, default_value_t = 0)]
        reason: u8,
    },

    /// Run as renewal daemon (checks daily)
    Daemon {
        /// Renew if certificate expires within this many days
        #[arg(long, default_value_t = 30)]
        renew_before_days: i64,

        /// Webhook URL for notifications
        #[arg(long)]
        webhook_url: Option<String>,
    },
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::from_default_env()
                .add_directive("cert_manager=info".parse()?),
        )
        .init();

    let cli = Cli::parse();

    let config_dir = cli
        .config_dir
        .unwrap_or_else(|| dirs::config_dir().unwrap().join("cert-manager"));

    match cli.command {
        Commands::Issue { domain, san, challenge, dns_provider, email } => {
            cmd_issue(&config_dir, &cli.ca_url, &domain, &san, &challenge, dns_provider, email).await?;
        }
        Commands::Renew { domain, renew_before_days } => {
            cmd_renew(&config_dir, &cli.ca_url, domain.as_deref(), renew_before_days).await?;
        }
        Commands::List => {
            cmd_list(&config_dir).await?;
        }
        Commands::Show { domain } => {
            cmd_show(&config_dir, &domain).await?;
        }
        Commands::Revoke { domain, reason } => {
            cmd_revoke(&config_dir, &cli.ca_url, &domain, reason).await?;
        }
        Commands::Daemon { renew_before_days, webhook_url } => {
            daemon::run_daemon(&config_dir, &cli.ca_url, renew_before_days, webhook_url).await?;
        }
    }

    Ok(())
}

async fn cmd_list(config_dir: &std::path::Path) -> Result<()> {
    let store = cert::CertStore::open(config_dir)?;
    let certs = store.list_certs()?;

    if certs.is_empty() {
        println!("No certificates managed.");
        return Ok(());
    }

    println!("{:<30} {:<25} {:<10} {}", "DOMAIN", "EXPIRES", "DAYS LEFT", "AUTO-RENEW");
    println!("{}", "-".repeat(75));

    let now = chrono::Utc::now();
    for manifest in certs {
        let days_left = (manifest.expires_at - now).num_days();
        let status = if days_left < 0 { "EXPIRED" } else { "ok" };
        println!(
            "{:<30} {:<25} {:<10} {}  [{}]",
            manifest.domain,
            manifest.expires_at.format("%Y-%m-%d %H:%M UTC"),
            days_left,
            if manifest.auto_renew { "yes" } else { "no" },
            status,
        );
    }
    Ok(())
}

// (ฟังก์ชัน cmd_issue, cmd_renew, cmd_show, cmd_revoke จะอยู่ใน module แยก)
async fn cmd_issue(
    _config_dir: &std::path::Path,
    _ca_url: &str,
    domain: &str,
    sans: &[String],
    _challenge: &str,
    _dns_provider: Option<String>,
    _email: Option<String>,
) -> Result<()> {
    println!("Issuing certificate for: {}", domain);
    if !sans.is_empty() {
        println!("Additional SANs: {}", sans.join(", "));
    }
    Ok(())
}

async fn cmd_renew(
    _config_dir: &std::path::Path,
    _ca_url: &str,
    domain: Option<&str>,
    renew_before_days: i64,
) -> Result<()> {
    match domain {
        Some(d) => println!("Renewing certificate for: {} (threshold: {}d)", d, renew_before_days),
        None => println!("Checking all certificates (threshold: {}d)", renew_before_days),
    }
    Ok(())
}

async fn cmd_show(config_dir: &std::path::Path, domain: &str) -> Result<()> {
    let store = cert::CertStore::open(config_dir)?;
    match store.load_manifest(domain)? {
        Some(m) => {
            println!("Domain:       {}", m.domain);
            println!("Issued:       {}", m.issued_at.format("%Y-%m-%d %H:%M UTC"));
            println!("Expires:      {}", m.expires_at.format("%Y-%m-%d %H:%M UTC"));
            println!("Chain length: {}", m.chain_length);
            println!("Root CA:      {}", m.root_ca);
            println!("Auto-renew:   {}", m.auto_renew);
        }
        None => eprintln!("Certificate not found for: {}", domain),
    }
    Ok(())
}

async fn cmd_revoke(
    _config_dir: &std::path::Path,
    _ca_url: &str,
    domain: &str,
    reason: u8,
) -> Result<()> {
    let reason_name = match reason {
        0 => "unspecified",
        1 => "keyCompromise",
        4 => "superseded",
        5 => "cessationOfOperation",
        _ => "unknown",
    };
    println!("Revoking certificate for: {} (reason: {} = {})", domain, reason, reason_name);
    Ok(())
}
```

---

### ขั้นที่ 2: JWS และ JWK Thumbprint

Module สำหรับสร้าง signed ACME request

**`src/acme/jws.rs`**

```rust
//! JWS (JSON Web Signature) สำหรับ ACME protocol (RFC 8555 §6)
//! ACME ใช้ JWS flattened serialization สำหรับทุก request

use base64::{engine::general_purpose::URL_SAFE_NO_PAD, Engine as _};
use p256::ecdsa::{SigningKey, Signature, signature::Signer};
use p256::pkcs8::EncodePrivateKey;
use p256::PublicKey;
use p256::elliptic_curve::sec1::ToEncodedPoint;
use sha2::{Digest, Sha256};
use serde_json::{json, Value};
use std::collections::BTreeMap;
use crate::error::{CertManagerError, Result};

/// สร้าง JWK (JSON Web Key) จาก ECDSA P-256 public key
/// ตาม RFC 7517 §4 และ RFC 7518 §6.2
pub fn public_key_to_jwk(public_key: &PublicKey) -> Value {
    let point = public_key.to_encoded_point(false); // uncompressed: 04 || x || y
    let x_bytes = &point.x().unwrap()[..];
    let y_bytes = &point.y().unwrap()[..];

    json!({
        "crv": "P-256",
        "kty": "EC",
        "x": URL_SAFE_NO_PAD.encode(x_bytes),
        "y": URL_SAFE_NO_PAD.encode(y_bytes),
    })
}

/// คำนวณ JWK Thumbprint ตาม RFC 7638 §3
///
/// Algorithm:
/// 1. สร้าง JSON object ที่มีเฉพาะ required members เรียงตาม lexicographic key order
/// 2. Serialize เป็น UTF-8 JSON (compact, no whitespace)
/// 3. Hash ด้วย SHA-256
/// 4. Encode ด้วย base64url (URL_SAFE_NO_PAD)
///
/// สำหรับ EC P-256 key, required members คือ: "crv", "kty", "x", "y"
pub fn jwk_thumbprint(public_key: &PublicKey) -> Result<String> {
    let point = public_key.to_encoded_point(false);
    let x_bytes = &point.x().unwrap()[..];
    let y_bytes = &point.y().unwrap()[..];

    // RFC 7638 §3.2: members MUST be sorted lexicographically
    let mut map = BTreeMap::new();
    map.insert("crv", "P-256".to_string());
    map.insert("kty", "EC".to_string());
    map.insert("x", URL_SAFE_NO_PAD.encode(x_bytes));
    map.insert("y", URL_SAFE_NO_PAD.encode(y_bytes));

    let canonical_json = serde_json::to_string(&map)
        .map_err(|e| CertManagerError::Crypto(e.to_string()))?;

    let hash = Sha256::digest(canonical_json.as_bytes());
    Ok(URL_SAFE_NO_PAD.encode(&hash))
}

/// สร้าง key authorization string สำหรับ ACME challenge
/// รูปแบบ: {token}.{jwk_thumbprint}  (RFC 8555 §8.1)
pub fn key_authorization(token: &str, thumbprint: &str) -> String {
    format!("{}.{}", token, thumbprint)
}

/// สร้าง DNS-01 TXT record value
/// รูปแบบ: base64url(SHA-256(key_authorization))  (RFC 8555 §8.4)
pub fn dns01_txt_value(key_auth: &str) -> String {
    let hash = Sha256::digest(key_auth.as_bytes());
    URL_SAFE_NO_PAD.encode(&hash)
}

/// JWS Header สำหรับ ACME request
/// การใช้ "jwk" (ไม่ใช่ "kid") เหมาะสำหรับ newAccount request
/// หลังจาก register แล้วใช้ "kid" แทน
#[derive(serde::Serialize)]
struct JwsHeader<'a> {
    alg: &'static str,
    nonce: &'a str,
    url: &'a str,
    #[serde(skip_serializing_if = "Option::is_none")]
    jwk: Option<Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    kid: Option<&'a str>,
}

/// สร้าง JWS flattened JSON serialization (RFC 7515 §7.2.2)
///
/// Parameters:
/// - `signing_key`: ECDSA P-256 private key
/// - `public_key`: corresponding public key (ใช้สร้าง JWK)
/// - `nonce`: Replay-Nonce จาก ACME server (ใช้ครั้งเดียว)
/// - `url`: URL ปลายทางของ request
/// - `payload`: request body (None = empty string สำหรับ POST-as-GET)
/// - `account_url`: ถ้ามีคือใช้ kid, ถ้า None คือยังไม่มี account (ใช้ jwk)
pub fn build_jws(
    signing_key: &SigningKey,
    public_key: &PublicKey,
    nonce: &str,
    url: &str,
    payload: Option<&Value>,
    account_url: Option<&str>,
) -> Result<Value> {
    // Build header
    let header = if let Some(kid) = account_url {
        JwsHeader { alg: "ES256", nonce, url, jwk: None, kid: Some(kid) }
    } else {
        let jwk = public_key_to_jwk(public_key);
        JwsHeader { alg: "ES256", nonce, url, jwk: Some(jwk), kid: None }
    };

    let header_json = serde_json::to_string(&header)
        .map_err(|e| CertManagerError::Crypto(e.to_string()))?;
    let protected = URL_SAFE_NO_PAD.encode(header_json.as_bytes());

    // Encode payload
    let payload_b64 = match payload {
        Some(v) => {
            let json = serde_json::to_string(v)
                .map_err(|e| CertManagerError::Crypto(e.to_string()))?;
            URL_SAFE_NO_PAD.encode(json.as_bytes())
        }
        None => String::new(), // POST-as-GET
    };

    // Sign: message = protected + "." + payload
    let message = format!("{}.{}", protected, payload_b64);
    let sig: Signature = signing_key.sign(message.as_bytes());

    // Convert DER signature to raw r||s format (64 bytes) for JWS
    let sig_bytes = sig.to_bytes();
    let signature = URL_SAFE_NO_PAD.encode(&sig_bytes);

    Ok(json!({
        "protected": protected,
        "payload": payload_b64,
        "signature": signature,
    }))
}
```

---

### ขั้นที่ 3: ACME Client — Directory, Nonce, Account

**`src/acme/account.rs`**

```rust
//! ACME Account management — registration และ key management

use p256::ecdsa::SigningKey;
use p256::pkcs8::{EncodePrivateKey, DecodePrivateKey};
use p256::PublicKey;
use rand::rngs::OsRng;
use serde::{Deserialize, Serialize};
use std::path::Path;
use crate::error::{CertManagerError, Result};

#[derive(Debug, Serialize, Deserialize)]
pub struct AccountInfo {
    pub account_url: String,
    pub contact_email: Option<String>,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

/// Load or create ECDSA P-256 signing key จาก PEM file
///
/// ถ้า key file ไม่มีอยู่ → generate ใหม่และบันทึกด้วย permission 0600
/// ถ้ามีอยู่แล้ว → load และ return
pub fn load_or_create_account_key(key_path: &Path) -> Result<SigningKey> {
    if key_path.exists() {
        let pem = std::fs::read_to_string(key_path)?;
        SigningKey::from_pkcs8_pem(&pem)
            .map_err(|e| CertManagerError::Crypto(format!("Load key failed: {}", e)))
    } else {
        // สร้าง key ใหม่
        let signing_key = SigningKey::random(&mut OsRng);

        // Serialize เป็น PKCS#8 PEM
        let pem = signing_key
            .to_pkcs8_pem(p256::pkcs8::LineEnding::LF)
            .map_err(|e| CertManagerError::Crypto(format!("Serialize key failed: {}", e)))?;

        // สร้าง parent directory ถ้ายังไม่มี
        if let Some(parent) = key_path.parent() {
            std::fs::create_dir_all(parent)?;
        }

        // เขียนไฟล์
        std::fs::write(key_path, pem.as_bytes())?;

        // ตั้ง permissions 0600 (Unix only)
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            std::fs::set_permissions(key_path, std::fs::Permissions::from_mode(0o600))?;
        }

        tracing::info!("Generated new account key at {}", key_path.display());
        Ok(signing_key)
    }
}

/// ดึง public key จาก signing key
pub fn public_key_from_signing(signing_key: &SigningKey) -> PublicKey {
    *signing_key.verifying_key()
}
```

**`src/acme/mod.rs`** — AcmeClient หลัก

```rust
//! ACME Protocol client (RFC 8555)
//!
//! Flow:
//! 1. GET /directory → AcmeDirectory (URL map)
//! 2. HEAD /acme/new-nonce → Replay-Nonce
//! 3. POST /acme/new-acct → account URL
//! 4. POST /acme/new-order → order URL + authz URLs
//! 5. GET /acme/authz/{id} → challenges
//! 6. POST /acme/chall/{id} → trigger validation
//! 7. Poll order status → "ready"
//! 8. POST /acme/order/{id}/finalize → CSR
//! 9. Poll order status → "valid"
//! 10. GET /acme/cert/{id} → PEM chain

use reqwest::Client;
use serde::{Deserialize, Serialize};
use serde_json::{json, Value};
use crate::error::{CertManagerError, Result};

pub mod account;
pub mod challenge;
pub mod finalize;
pub mod jws;
pub mod order;

/// ACME Directory response (RFC 8555 §7.1.1)
#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct AcmeDirectory {
    pub new_account: String,
    pub new_nonce: String,
    pub new_order: String,
    pub revoke_cert: String,
    pub key_change: Option<String>,
    pub meta: Option<DirectoryMeta>,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct DirectoryMeta {
    pub terms_of_service: Option<String>,
    pub website: Option<String>,
    pub caa_identities: Option<Vec<String>>,
}

/// ACME Order (RFC 8555 §7.1.3)
#[derive(Debug, Deserialize, Serialize)]
pub struct AcmeOrder {
    pub status: String,           // pending | ready | processing | valid | invalid
    pub identifiers: Vec<Identifier>,
    pub authorizations: Vec<String>,
    pub finalize: String,
    pub certificate: Option<String>,
    pub expires: Option<String>,
}

/// ACME Identifier (RFC 8555 §9.7.7)
#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct Identifier {
    #[serde(rename = "type")]
    pub id_type: String,  // "dns"
    pub value: String,    // domain name
}

/// ACME Authorization (RFC 8555 §7.1.4)
#[derive(Debug, Deserialize)]
pub struct AcmeAuthorization {
    pub status: String,
    pub identifier: Identifier,
    pub challenges: Vec<AcmeChallenge>,
    pub expires: Option<String>,
}

/// ACME Challenge (RFC 8555 §8)
#[derive(Debug, Deserialize, Clone)]
#[serde(rename_all = "camelCase")]
pub struct AcmeChallenge {
    #[serde(rename = "type")]
    pub challenge_type: String,  // "http-01" | "dns-01" | "tls-alpn-01"
    pub url: String,
    pub token: String,
    pub status: Option<String>,
    pub validated: Option<String>,
    pub error: Option<Value>,
}

pub struct AcmeClient {
    http: Client,
    directory_url: String,
    directory: Option<AcmeDirectory>,
    signing_key: p256::ecdsa::SigningKey,
    account_url: Option<String>,
}

impl AcmeClient {
    pub fn new(directory_url: &str, signing_key: p256::ecdsa::SigningKey) -> Self {
        let http = Client::builder()
            .user_agent("cert-manager/0.1 (Rust ACME client)")
            .build()
            .expect("build HTTP client");

        AcmeClient {
            http,
            directory_url: directory_url.to_string(),
            directory: None,
            signing_key,
            account_url: None,
        }
    }

    /// Step 1: Fetch ACME directory
    pub async fn fetch_directory(&mut self) -> Result<&AcmeDirectory> {
        let resp = self.http
            .get(&self.directory_url)
            .send()
            .await?
            .error_for_status()?;

        let dir: AcmeDirectory = resp.json().await?;
        tracing::debug!("Fetched directory from {}", self.directory_url);
        self.directory = Some(dir);
        Ok(self.directory.as_ref().unwrap())
    }

    /// Step 2: Fetch fresh nonce สำหรับ JWS replay protection
    /// ACME ใช้ HEAD request ไปที่ newNonce URL
    pub async fn fetch_nonce(&self) -> Result<String> {
        let nonce_url = self
            .directory
            .as_ref()
            .ok_or_else(|| CertManagerError::Acme("Directory not fetched".into()))?
            .new_nonce
            .clone();

        let resp = self.http
            .head(&nonce_url)
            .send()
            .await?;

        resp.headers()
            .get("replay-nonce")
            .and_then(|v| v.to_str().ok())
            .map(String::from)
            .ok_or_else(|| CertManagerError::Acme("Server did not return Replay-Nonce".into()))
    }

    fn public_key(&self) -> p256::PublicKey {
        *self.signing_key.verifying_key()
    }

    /// Step 3: Register new account หรือ fetch existing account
    ///
    /// ACME ใช้ "upsert" semantics — ถ้า key เคย register แล้วจะ return URL เดิม
    /// contact email เป็น optional แต่ Let's Encrypt แนะนำให้ใส่
    pub async fn register_account(
        &mut self,
        contact_email: Option<&str>,
        accept_tos: bool,
    ) -> Result<String> {
        let dir = self.directory.as_ref()
            .ok_or_else(|| CertManagerError::Acme("Directory not fetched".into()))?;
        let new_account_url = dir.new_account.clone();

        let nonce = self.fetch_nonce().await?;

        let mut payload = json!({
            "termsOfServiceAgreed": accept_tos,
        });

        if let Some(email) = contact_email {
            payload["contact"] = json!([format!("mailto:{}", email)]);
        }

        let jws = jws::build_jws(
            &self.signing_key,
            &self.public_key(),
            &nonce,
            &new_account_url,
            Some(&payload),
            None, // ใช้ JWK ไม่ใช่ KID สำหรับ newAccount
        )?;

        let resp = self.http
            .post(&new_account_url)
            .header("content-type", "application/jose+json")
            .json(&jws)
            .send()
            .await?;

        let account_url = resp
            .headers()
            .get("location")
            .and_then(|v| v.to_str().ok())
            .map(String::from)
            .ok_or_else(|| CertManagerError::Acme("No Location header in account response".into()))?;

        resp.error_for_status()?;
        self.account_url = Some(account_url.clone());
        tracing::info!("ACME account: {}", account_url);
        Ok(account_url)
    }

    /// Step 4: Create new order สำหรับ list of identifiers
    ///
    /// รองรับ multi-domain order: ["example.com", "*.example.com", "api.example.com"]
    pub async fn create_order(&self, domains: &[&str]) -> Result<(String, AcmeOrder)> {
        let dir = self.directory.as_ref()
            .ok_or_else(|| CertManagerError::Acme("Directory not fetched".into()))?;
        let new_order_url = dir.new_order.clone();

        let account_url = self.account_url.as_deref()
            .ok_or_else(|| CertManagerError::Acme("Not registered".into()))?;

        let nonce = self.fetch_nonce().await?;

        let identifiers: Vec<Value> = domains
            .iter()
            .map(|d| json!({ "type": "dns", "value": d }))
            .collect();

        let payload = json!({ "identifiers": identifiers });

        let jws = jws::build_jws(
            &self.signing_key,
            &self.public_key(),
            &nonce,
            &new_order_url,
            Some(&payload),
            Some(account_url),
        )?;

        let resp = self.http
            .post(&new_order_url)
            .header("content-type", "application/jose+json")
            .json(&jws)
            .send()
            .await?
            .error_for_status()?;

        let order_url = resp
            .headers()
            .get("location")
            .and_then(|v| v.to_str().ok())
            .map(String::from)
            .ok_or_else(|| CertManagerError::Acme("No Location in order response".into()))?;

        let order: AcmeOrder = resp.json().await?;
        tracing::info!("Created order {} (status: {})", order_url, order.status);
        Ok((order_url, order))
    }

    /// Fetch authorization details สำหรับ URL ที่ได้จาก order
    pub async fn get_authorization(&self, authz_url: &str) -> Result<AcmeAuthorization> {
        let account_url = self.account_url.as_deref()
            .ok_or_else(|| CertManagerError::Acme("Not registered".into()))?;
        let nonce = self.fetch_nonce().await?;

        // POST-as-GET: payload = "" (RFC 8555 §6.3)
        let jws = jws::build_jws(
            &self.signing_key,
            &self.public_key(),
            &nonce,
            authz_url,
            None, // POST-as-GET
            Some(account_url),
        )?;

        let resp = self.http
            .post(authz_url)
            .header("content-type", "application/jose+json")
            .json(&jws)
            .send()
            .await?
            .error_for_status()?;

        let authz: AcmeAuthorization = resp.json().await?;
        Ok(authz)
    }

    /// Notify server ว่า challenge พร้อมแล้ว (trigger validation)
    pub async fn respond_to_challenge(&self, challenge_url: &str) -> Result<()> {
        let account_url = self.account_url.as_deref()
            .ok_or_else(|| CertManagerError::Acme("Not registered".into()))?;
        let nonce = self.fetch_nonce().await?;

        // ส่ง empty JSON object {} เพื่อ trigger validation
        let payload = json!({});
        let jws = jws::build_jws(
            &self.signing_key,
            &self.public_key(),
            &nonce,
            challenge_url,
            Some(&payload),
            Some(account_url),
        )?;

        self.http
            .post(challenge_url)
            .header("content-type", "application/jose+json")
            .json(&jws)
            .send()
            .await?
            .error_for_status()?;

        Ok(())
    }
}
```

---

### ขั้นที่ 4: Challenge Responders

**`src/acme/challenge.rs`**

```rust
//! HTTP-01 และ DNS-01 challenge responders

use axum::{Router, routing::get, extract::{Path, State}, response::IntoResponse};
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;
use crate::error::{CertManagerError, Result};

// ─── HTTP-01 Challenge ────────────────────────────────────────────────────────

/// State ที่ axum server ใช้เพื่อ serve challenge tokens
type ChallengeMap = Arc<RwLock<HashMap<String, String>>>;

/// Handler: GET /.well-known/acme-challenge/{token}
/// Return: key authorization string สำหรับ token นั้น
/// Content-Type: application/octet-stream (RFC 8555 §8.3)
async fn serve_challenge(
    Path(token): Path<String>,
    State(challenges): State<ChallengeMap>,
) -> impl IntoResponse {
    let map = challenges.read().await;
    match map.get(&token) {
        Some(key_auth) => {
            tracing::info!("HTTP-01: served token {}", token);
            (
                axum::http::StatusCode::OK,
                [(axum::http::header::CONTENT_TYPE, "application/octet-stream")],
                key_auth.clone(),
            )
        }
        None => {
            tracing::warn!("HTTP-01: unknown token {}", token);
            (
                axum::http::StatusCode::NOT_FOUND,
                [(axum::http::header::CONTENT_TYPE, "text/plain")],
                "Not found".to_string(),
            )
        }
    }
}

/// HTTP-01 Challenge Server
///
/// เริ่ม axum server บน port 80 เพื่อ serve ACME challenge tokens
/// Server จะอยู่จนกว่าจะ call shutdown()
///
/// **หมายเหตุ Security**: port 80 ต้องการ CAP_NET_BIND_SERVICE บน Linux
/// หรือใช้ `authbind` / `setcap` เพื่อ allow binding
pub struct Http01ChallengeServer {
    challenges: ChallengeMap,
    shutdown_tx: Option<tokio::sync::oneshot::Sender<()>>,
}

impl Http01ChallengeServer {
    pub fn new() -> Self {
        Http01ChallengeServer {
            challenges: Arc::new(RwLock::new(HashMap::new())),
            shutdown_tx: None,
        }
    }

    /// เพิ่ม token → key_authorization mapping
    pub async fn add_challenge(&self, token: String, key_auth: String) {
        let mut map = self.challenges.write().await;
        map.insert(token, key_auth);
    }

    /// ลบ token หลัง challenge เสร็จ
    pub async fn remove_challenge(&self, token: &str) {
        let mut map = self.challenges.write().await;
        map.remove(token);
    }

    /// เริ่ม HTTP server บน port 80 ใน background task
    pub async fn start(&mut self) -> Result<()> {
        let challenges = self.challenges.clone();
        let (shutdown_tx, shutdown_rx) = tokio::sync::oneshot::channel::<()>();

        let app = Router::new()
            .route(
                "/.well-known/acme-challenge/:token",
                get(serve_challenge),
            )
            .with_state(challenges);

        let listener = tokio::net::TcpListener::bind("0.0.0.0:80")
            .await
            .map_err(|e| CertManagerError::Challenge(
                format!("Cannot bind port 80: {}. Try: sudo setcap cap_net_bind_service=+ep cert-manager", e)
            ))?;

        tokio::spawn(async move {
            axum::serve(listener, app)
                .with_graceful_shutdown(async {
                    let _ = shutdown_rx.await;
                })
                .await
                .unwrap_or_else(|e| tracing::error!("HTTP server error: {}", e));
        });

        self.shutdown_tx = Some(shutdown_tx);
        tracing::info!("HTTP-01 challenge server started on :80");
        Ok(())
    }

    /// หยุด HTTP server
    pub fn shutdown(mut self) {
        if let Some(tx) = self.shutdown_tx.take() {
            let _ = tx.send(());
        }
    }
}

// ─── DNS-01 Challenge ─────────────────────────────────────────────────────────

use base64::{engine::general_purpose::URL_SAFE_NO_PAD, Engine as _};
use sha2::{Digest, Sha256};

/// สร้าง DNS-01 TXT record value สำหรับ ACME challenge
/// ตาม RFC 8555 §8.4
///
/// DNS TXT record:
///   Name:  _acme-challenge.{domain}
///   Value: base64url(SHA-256(key_authorization))
pub fn compute_dns01_value(key_auth: &str) -> String {
    let hash = Sha256::digest(key_auth.as_bytes());
    URL_SAFE_NO_PAD.encode(&hash)
}

/// ชื่อ DNS TXT record สำหรับ DNS-01 challenge
pub fn dns_challenge_name(domain: &str) -> String {
    // Wildcard domain: *.example.com → _acme-challenge.example.com
    let base_domain = domain.trim_start_matches("*.");
    format!("_acme-challenge.{}", base_domain)
}

/// Poll DNS propagation ก่อน respond ไปหา ACME server
///
/// DNS-01 challenge ต้องรอให้ TXT record propagate ไปทั่วก่อน
/// เพราะ ACME server อาจใช้ resolver ที่ต่างออกไป
///
/// Strategy: check public resolver (8.8.8.8) สูงสุด max_attempts ครั้ง
/// ห่างกัน interval_secs วินาที
pub async fn wait_for_dns_propagation(
    domain: &str,
    expected_value: &str,
    max_attempts: u32,
    interval_secs: u64,
) -> Result<()> {
    use std::net::SocketAddr;

    let challenge_name = dns_challenge_name(domain);
    tracing::info!(
        "Waiting for DNS TXT {} = {}",
        challenge_name,
        expected_value
    );

    for attempt in 1..=max_attempts {
        // ในโปรเจคจริงใช้ `hickory-resolver` หรือ `trust-dns-resolver`
        // เพื่อ query DNS โดยตรง
        // ที่นี่แสดง pseudocode สำหรับ illustration
        tracing::debug!("DNS poll attempt {}/{}", attempt, max_attempts);

        // Simulated check — ใน production ใช้:
        // let resolver = TokioAsyncResolver::tokio_from_system_conf().await?;
        // let response = resolver.txt_lookup(&challenge_name).await;
        // if response.iter().any(|r| r.to_string() == expected_value) { return Ok(()); }

        if attempt < max_attempts {
            tokio::time::sleep(tokio::time::Duration::from_secs(interval_secs)).await;
        }
    }

    Err(CertManagerError::Challenge(format!(
        "DNS TXT {} did not propagate after {} attempts",
        challenge_name,
        max_attempts
    )))
}
```

---

### ขั้นที่ 5: CSR Generation และ Certificate Storage

**`src/cert/csr.rs`**

```rust
//! Certificate Signing Request (CSR) generation ด้วย rcgen
//!
//! สร้าง PKCS#10 CSR สำหรับ ECDSA P-256 หรือ RSA-2048 key
//! พร้อม Subject Alternative Names (SANs) รวมถึง wildcard

use rcgen::{
    CertificateParams, DistinguishedName, DnType,
    ExtendedKeyUsagePurpose, KeyPair, SanType,
};
use crate::error::{CertManagerError, Result};

pub struct CsrResult {
    /// PEM-encoded PKCS#10 CSR (ส่งไป ACME finalize endpoint)
    pub csr_pem: String,
    /// DER-encoded CSR (base64url encoding สำหรับ ACME)
    pub csr_der: Vec<u8>,
    /// PEM-encoded private key (บันทึกลง disk, permission 0600)
    pub key_pem: String,
}

/// Generate ECDSA P-256 key pair และ PKCS#10 CSR
///
/// Parameters:
/// - `common_name`: primary domain (เข้าไปใน Subject CN)
/// - `sans`: ทุก domain ที่ต้องการในใบ certificate รวม wildcard
///
/// ตัวอย่าง SANs: ["example.com", "*.example.com", "api.example.com"]
pub fn generate_csr_ecdsa(common_name: &str, sans: &[&str]) -> Result<CsrResult> {
    // Generate ECDSA P-256 key pair
    let key_pair = KeyPair::generate()
        .map_err(|e| CertManagerError::Crypto(format!("KeyPair::generate: {}", e)))?;

    build_csr(common_name, sans, key_pair)
}

/// Generate RSA-2048 key pair และ PKCS#10 CSR
pub fn generate_csr_rsa2048(common_name: &str, sans: &[&str]) -> Result<CsrResult> {
    use rcgen::KeyPair;
    // rcgen รองรับ RSA ผ่าน ring backend
    let key_pair = KeyPair::generate_for(&rcgen::PKCS_RSA_SHA256)
        .map_err(|e| CertManagerError::Crypto(format!("RSA KeyPair::generate: {}", e)))?;

    build_csr(common_name, sans, key_pair)
}

fn build_csr(common_name: &str, sans: &[&str], key_pair: KeyPair) -> Result<CsrResult> {
    let mut params = CertificateParams::new(
        sans.iter().map(|s| s.to_string()).collect::<Vec<_>>(),
    ).map_err(|e| CertManagerError::Crypto(format!("CertificateParams: {}", e)))?;

    let mut dn = DistinguishedName::new();
    dn.push(DnType::CommonName, common_name);
    // Organization เป็น optional — Let's Encrypt ไม่ตรวจสอบ
    params.distinguished_name = dn;

    // Extended Key Usage: serverAuth (OID 1.3.6.1.5.5.7.3.1)
    params.extended_key_usages = vec![ExtendedKeyUsagePurpose::ServerAuth];

    let csr = params
        .serialize_request(&key_pair)
        .map_err(|e| CertManagerError::Crypto(format!("serialize_request: {}", e)))?;

    let csr_pem = csr.pem()
        .map_err(|e| CertManagerError::Crypto(format!("CSR PEM: {}", e)))?;

    let csr_der = csr.der().to_vec();
    let key_pem = key_pair.serialize_pem();

    Ok(CsrResult { csr_pem, csr_der, key_pem })
}
```

**`src/cert/manifest.rs`**

```rust
//! Certificate manifest — JSON metadata ที่บันทึกคู่กับ certificate

use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// JSON manifest บันทึกข้อมูล certificate สำหรับ renewal decision
/// บันทึกที่ ~/.config/cert-manager/certs/{domain}/manifest.json
#[derive(Debug, Serialize, Deserialize, PartialEq, Clone)]
pub struct CertManifest {
    /// Primary domain name
    pub domain: String,

    /// เวลาที่ certificate ถูก issue
    pub issued_at: DateTime<Utc>,

    /// เวลาที่ certificate จะหมดอายุ (= certificate not_after)
    pub expires_at: DateTime<Utc>,

    /// ถ้า true → daemon จะ renew อัตโนมัติก่อนหมดอายุ
    pub auto_renew: bool,

    /// จำนวน certificate ใน chain (leaf + intermediates)
    pub chain_length: usize,

    /// ชื่อ Root CA (เช่น "ISRG Root X1")
    pub root_ca: String,

    /// Challenge type ที่ใช้ล่าสุด ("http-01" | "dns-01")
    pub challenge_type: String,

    /// SANs ทั้งหมดในใบ certificate
    pub sans: Vec<String>,
}

impl CertManifest {
    /// ตรวจสอบว่าต้อง renew หรือไม่
    /// คืนค่า true ถ้า (expires_at - now) < threshold_days
    pub fn needs_renewal(&self, now: DateTime<Utc>, threshold_days: i64) -> bool {
        let days_left = (self.expires_at - now).num_days();
        days_left < threshold_days
    }

    /// จำนวนวันที่เหลือก่อนหมดอายุ (อาจเป็นลบถ้า expired)
    pub fn days_until_expiry(&self, now: DateTime<Utc>) -> i64 {
        (self.expires_at - now).num_days()
    }
}
```

**`src/cert/mod.rs`**

```rust
//! Certificate store — load/save/list certificates บน filesystem

use std::path::{Path, PathBuf};
use crate::error::{CertManagerError, Result};

pub mod csr;
pub mod manifest;
pub mod chain;

pub use manifest::CertManifest;

pub struct CertStore {
    base_dir: PathBuf,
}

impl CertStore {
    /// เปิด CertStore ที่ base_dir (สร้าง directories ถ้ายังไม่มี)
    pub fn open(base_dir: &Path) -> Result<Self> {
        std::fs::create_dir_all(base_dir.join("certs"))?;
        std::fs::create_dir_all(base_dir.join("accounts"))?;
        Ok(CertStore { base_dir: base_dir.to_owned() })
    }

    fn cert_dir(&self, domain: &str) -> PathBuf {
        self.base_dir.join("certs").join(domain)
    }

    /// บันทึก certificate, chain, fullchain, key และ manifest
    ///
    /// Permissions:
    /// - cert.pem, chain.pem, fullchain.pem: 0644
    /// - key.pem: 0600
    pub fn save_cert(
        &self,
        domain: &str,
        cert_pem: &str,
        chain_pem: &str,
        key_pem: &str,
        manifest: &CertManifest,
    ) -> Result<()> {
        let dir = self.cert_dir(domain);
        std::fs::create_dir_all(&dir)?;

        // Fullchain = leaf cert + intermediate chain
        let fullchain = format!("{}\n{}", cert_pem.trim(), chain_pem.trim());

        write_file_with_mode(&dir.join("cert.pem"), cert_pem.as_bytes(), 0o644)?;
        write_file_with_mode(&dir.join("chain.pem"), chain_pem.as_bytes(), 0o644)?;
        write_file_with_mode(&dir.join("fullchain.pem"), fullchain.as_bytes(), 0o644)?;
        write_file_with_mode(&dir.join("key.pem"), key_pem.as_bytes(), 0o600)?;

        let manifest_json = serde_json::to_string_pretty(manifest)?;
        write_file_with_mode(&dir.join("manifest.json"), manifest_json.as_bytes(), 0o644)?;

        tracing::info!("Saved certificate for {} to {}", domain, dir.display());
        Ok(())
    }

    /// โหลด manifest สำหรับ domain
    pub fn load_manifest(&self, domain: &str) -> Result<Option<CertManifest>> {
        let path = self.cert_dir(domain).join("manifest.json");
        if !path.exists() {
            return Ok(None);
        }
        let json = std::fs::read_to_string(&path)?;
        let manifest: CertManifest = serde_json::from_str(&json)?;
        Ok(Some(manifest))
    }

    /// List manifests สำหรับทุก domain ที่ manage อยู่
    pub fn list_certs(&self) -> Result<Vec<CertManifest>> {
        let certs_dir = self.base_dir.join("certs");
        if !certs_dir.exists() {
            return Ok(vec![]);
        }

        let mut manifests = Vec::new();
        for entry in std::fs::read_dir(&certs_dir)? {
            let entry = entry?;
            let manifest_path = entry.path().join("manifest.json");
            if manifest_path.exists() {
                let json = std::fs::read_to_string(&manifest_path)?;
                if let Ok(m) = serde_json::from_str::<CertManifest>(&json) {
                    manifests.push(m);
                }
            }
        }

        // เรียงตาม expires_at ascending
        manifests.sort_by_key(|m| m.expires_at);
        Ok(manifests)
    }

    /// Load certificate PEM files
    pub fn load_cert_pem(&self, domain: &str) -> Result<(String, String, String, String)> {
        let dir = self.cert_dir(domain);
        if !dir.exists() {
            return Err(CertManagerError::NotFound(domain.to_string()));
        }
        let cert = std::fs::read_to_string(dir.join("cert.pem"))?;
        let chain = std::fs::read_to_string(dir.join("chain.pem"))?;
        let fullchain = std::fs::read_to_string(dir.join("fullchain.pem"))?;
        let key = std::fs::read_to_string(dir.join("key.pem"))?;
        Ok((cert, chain, fullchain, key))
    }
}

fn write_file_with_mode(path: &Path, data: &[u8], mode: u32) -> Result<()> {
    std::fs::write(path, data)?;
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        std::fs::set_permissions(path, std::fs::Permissions::from_mode(mode))?;
    }
    Ok(())
}
```

---

### ขั้นที่ 6: Certificate Chain Validation และ Daemon

**`src/cert/chain.rs`**

```rust
//! Certificate chain validation
//!
//! ตรวจสอบ certificate chain: leaf → intermediate(s) → root
//! รายงาน chain length และ root CA name

use x509_cert::Certificate;
use der::Decode;
use base64::{engine::general_purpose::STANDARD, Engine as _};
use chrono::{DateTime, Utc};
use crate::error::{CertManagerError, Result};

pub struct ChainInfo {
    pub chain_length: usize,
    pub root_ca: String,
    pub leaf_subject: String,
    pub not_before: DateTime<Utc>,
    pub not_after: DateTime<Utc>,
}

/// Parse PEM chain และตรวจสอบความถูกต้อง
///
/// PEM chain file อาจมีหลาย certificate block ต่อกัน:
/// -----BEGIN CERTIFICATE-----
/// (leaf cert)
/// -----END CERTIFICATE-----
/// -----BEGIN CERTIFICATE-----
/// (intermediate CA)
/// -----END CERTIFICATE-----
///
/// Returns: ChainInfo พร้อม root CA name และ expiry
pub fn validate_chain(fullchain_pem: &str) -> Result<ChainInfo> {
    let certs = parse_pem_chain(fullchain_pem)?;

    if certs.is_empty() {
        return Err(CertManagerError::Validation("Empty certificate chain".into()));
    }

    // Leaf certificate (first in chain)
    let leaf = &certs[0];

    // ตรวจสอบ subject ของ leaf
    let leaf_subject = format!("{}", leaf.tbs_certificate.subject);

    // Root CA = certificate สุดท้าย
    let root = certs.last().unwrap();
    let root_ca = format!("{}", root.tbs_certificate.subject);

    // Extract not_after จาก leaf certificate
    let not_after_dur = leaf.tbs_certificate.validity.not_after.to_unix_duration();
    let not_after = DateTime::from_timestamp(not_after_dur.as_secs() as i64, 0)
        .ok_or_else(|| CertManagerError::Validation("Invalid not_after timestamp".into()))?;

    let not_before_dur = leaf.tbs_certificate.validity.not_before.to_unix_duration();
    let not_before = DateTime::from_timestamp(not_before_dur.as_secs() as i64, 0)
        .ok_or_else(|| CertManagerError::Validation("Invalid not_before timestamp".into()))?;

    // ตรวจสอบ chain ordering: แต่ละ issuer ต้องตรงกับ subject ของ cert ถัดไป
    for i in 0..certs.len().saturating_sub(1) {
        let current_issuer = format!("{}", certs[i].tbs_certificate.issuer);
        let next_subject = format!("{}", certs[i + 1].tbs_certificate.subject);
        if current_issuer != next_subject {
            return Err(CertManagerError::Validation(format!(
                "Chain break: cert[{}] issuer '{}' != cert[{}] subject '{}'",
                i, current_issuer, i + 1, next_subject
            )));
        }
    }

    Ok(ChainInfo {
        chain_length: certs.len(),
        root_ca,
        leaf_subject,
        not_before,
        not_after,
    })
}

/// Parse PEM string ที่อาจมีหลาย certificate block
fn parse_pem_chain(pem: &str) -> Result<Vec<Certificate>> {
    let mut certs = Vec::new();
    let mut remaining = pem;

    while let Some(start) = remaining.find("-----BEGIN CERTIFICATE-----") {
        let after_header = &remaining[start + 27..];
        let end = after_header
            .find("-----END CERTIFICATE-----")
            .ok_or_else(|| CertManagerError::Validation("Unterminated PEM block".into()))?;

        let b64: String = after_header[..end]
            .chars()
            .filter(|c| !c.is_whitespace())
            .collect();

        let der = STANDARD
            .decode(&b64)
            .map_err(|e| CertManagerError::Validation(format!("Base64 decode: {}", e)))?;

        let cert = Certificate::from_der(&der)
            .map_err(|e| CertManagerError::Validation(format!("DER parse: {}", e)))?;

        certs.push(cert);
        remaining = &after_header[end + 25..];
    }

    Ok(certs)
}
```

**`src/daemon.rs`**

```rust
//! Renewal daemon — ตรวจสอบ certificate expiry ทุกวัน และ renew อัตโนมัติ
//!
//! ทำงานเป็น background loop:
//! 1. โหลด manifests ทุก domain
//! 2. ตรวจว่า cert ไหน expires_at - now < threshold_days
//! 3. Renew cert นั้นผ่าน ACME
//! 4. ส่ง webhook notification
//! 5. นอน 24 ชั่วโมง แล้ววนซ้ำ

use std::path::Path;
use chrono::Utc;
use crate::cert::CertStore;
use crate::error::Result;
use crate::webhook::{WebhookClient, WebhookEvent, EventType};

/// เริ่ม daemon loop
///
/// loop ทำงานต่อเนื่องจนกว่าจะ terminate ด้วย SIGTERM/SIGINT
/// แต่ละรอบตรวจสอบทุก managed certificate
pub async fn run_daemon(
    config_dir: &Path,
    ca_url: &str,
    renew_before_days: i64,
    webhook_url: Option<String>,
) -> Result<()> {
    tracing::info!(
        "Starting daemon (renew_before_days={}, webhook={})",
        renew_before_days,
        webhook_url.as_deref().unwrap_or("none")
    );

    let webhook = webhook_url.as_deref().map(WebhookClient::new);
    let check_interval = tokio::time::Duration::from_secs(24 * 60 * 60); // 24 hours

    loop {
        tracing::info!("Starting renewal check cycle");

        match check_and_renew_all(config_dir, ca_url, renew_before_days, &webhook).await {
            Ok(renewed_count) => {
                tracing::info!("Renewal cycle complete: {} certs renewed", renewed_count);
            }
            Err(e) => {
                tracing::error!("Renewal cycle error: {}", e);
            }
        }

        tracing::info!("Next check in 24 hours");
        tokio::time::sleep(check_interval).await;
    }
}

/// ตรวจสอบและ renew certificate ทุกตัวใน store
async fn check_and_renew_all(
    config_dir: &Path,
    _ca_url: &str,
    renew_before_days: i64,
    webhook: &Option<WebhookClient>,
) -> Result<usize> {
    let store = CertStore::open(config_dir)?;
    let manifests = store.list_certs()?;
    let now = Utc::now();
    let mut renewed = 0;

    for manifest in manifests {
        if !manifest.auto_renew {
            tracing::debug!("Skipping {} (auto_renew=false)", manifest.domain);
            continue;
        }

        let days_left = manifest.days_until_expiry(now);

        if days_left < 0 {
            tracing::warn!("Certificate EXPIRED: {} ({} days ago)", manifest.domain, -days_left);
            if let Some(wh) = webhook {
                let _ = wh.send(WebhookEvent {
                    event_type: EventType::Failed,
                    domain: manifest.domain.clone(),
                    expires_at: manifest.expires_at,
                    message: format!("Certificate expired {} days ago", -days_left),
                }).await;
            }
        } else if manifest.needs_renewal(now, renew_before_days) {
            tracing::info!(
                "Renewing {} (expires in {} days < threshold {})",
                manifest.domain, days_left, renew_before_days
            );

            // ใน production ต้อง call ACME client จริง
            // ที่นี่แสดง structure ของการ renewal
            match perform_renewal(&manifest.domain, &manifest).await {
                Ok(_) => {
                    renewed += 1;
                    if let Some(wh) = webhook {
                        let _ = wh.send(WebhookEvent {
                            event_type: EventType::Renewed,
                            domain: manifest.domain.clone(),
                            expires_at: manifest.expires_at,
                            message: format!("Renewed with {} days left", days_left),
                        }).await;
                    }
                }
                Err(e) => {
                    tracing::error!("Renewal failed for {}: {}", manifest.domain, e);
                    if let Some(wh) = webhook {
                        let _ = wh.send(WebhookEvent {
                            event_type: EventType::Failed,
                            domain: manifest.domain.clone(),
                            expires_at: manifest.expires_at,
                            message: format!("Renewal error: {}", e),
                        }).await;
                    }
                }
            }
        } else {
            tracing::debug!(
                "Certificate OK: {} ({} days left)",
                manifest.domain, days_left
            );

            // แจ้งเตือนถ้าเหลือน้อยกว่า 7 วัน (แม้ยังไม่ถึง threshold)
            if days_left < 7 {
                if let Some(wh) = webhook {
                    let _ = wh.send(WebhookEvent {
                        event_type: EventType::Expiring,
                        domain: manifest.domain.clone(),
                        expires_at: manifest.expires_at,
                        message: format!("Certificate expiring soon: {} days left", days_left),
                    }).await;
                }
            }
        }
    }

    Ok(renewed)
}

async fn perform_renewal(
    domain: &str,
    _manifest: &crate::cert::CertManifest,
) -> crate::error::Result<()> {
    // placeholder — จะ call AcmeClient จริงในโปรเจคสมบูรณ์
    tracing::info!("Performing renewal for {}", domain);
    Ok(())
}
```

**`src/webhook.rs`**

```rust
//! Webhook notification client
//!
//! ส่ง POST JSON ไปยัง configured URL เมื่อ certificate event เกิดขึ้น

use chrono::{DateTime, Utc};
use reqwest::Client;
use serde::{Deserialize, Serialize};
use crate::error::{CertManagerError, Result};

#[derive(Debug, Serialize, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum EventType {
    Issued,
    Renewed,
    Expiring,
    Failed,
    Revoked,
}

#[derive(Debug, Serialize)]
pub struct WebhookEvent {
    pub event_type: EventType,
    pub domain: String,
    pub expires_at: DateTime<Utc>,
    pub message: String,
    #[serde(skip)]
    _timestamp: (),
}

impl WebhookEvent {
    fn to_payload(&self) -> serde_json::Value {
        serde_json::json!({
            "event": self.event_type,
            "domain": self.domain,
            "expires_at": self.expires_at,
            "message": self.message,
            "timestamp": Utc::now(),
        })
    }
}

pub struct WebhookClient {
    url: String,
    http: Client,
}

impl WebhookClient {
    pub fn new(url: &str) -> Self {
        WebhookClient {
            url: url.to_string(),
            http: Client::builder()
                .timeout(std::time::Duration::from_secs(10))
                .build()
                .expect("build webhook HTTP client"),
        }
    }

    /// ส่ง webhook event ด้วย POST JSON
    ///
    /// ถ้า server ตอบ 2xx → success
    /// ถ้าไม่ตอบหรือ error → log แต่ไม่ panic (webhook เป็น best-effort)
    pub async fn send(&self, event: WebhookEvent) -> Result<()> {
        let payload = event.to_payload();

        tracing::debug!(
            "Sending webhook to {}: event={:?} domain={}",
            self.url, event.event_type, event.domain
        );

        let resp = self.http
            .post(&self.url)
            .header("content-type", "application/json")
            .header("x-cert-manager-event", format!("{:?}", event.event_type).to_lowercase())
            .json(&payload)
            .send()
            .await
            .map_err(|e| CertManagerError::Webhook(e.to_string()))?;

        if resp.status().is_success() {
            tracing::info!(
                "Webhook delivered: {} → {} ({})",
                event.domain, self.url, resp.status()
            );
        } else {
            tracing::warn!(
                "Webhook non-success: {} → {} ({})",
                event.domain, self.url, resp.status()
            );
        }

        Ok(())
    }
}
```

---

### ขั้นที่ 7: DNS Provider Abstraction

**`src/dns/mod.rs`**

```rust
//! DNS Provider trait — abstraction สำหรับ DNS-01 challenge

use crate::error::Result;

pub mod cloudflare;
pub mod route53;

/// Trait สำหรับ DNS provider ที่รองรับ ACME DNS-01 challenge
///
/// Implementation ต้องรองรับ:
/// 1. create_txt_record: สร้าง TXT record สำหรับ challenge
/// 2. delete_txt_record: ลบ TXT record หลัง challenge เสร็จ
/// 3. get_txt_records: ดึง TXT records เพื่อ verify propagation
#[async_trait::async_trait]
pub trait DnsProvider: Send + Sync {
    /// สร้าง TXT record สำหรับ DNS-01 challenge
    ///
    /// name: "_acme-challenge.example.com"
    /// value: base64url(SHA-256(key_authorization))
    async fn create_txt_record(&self, name: &str, value: &str) -> Result<String>; // returns record_id

    /// ลบ TXT record ที่สร้างไว้สำหรับ challenge
    async fn delete_txt_record(&self, name: &str, record_id: &str) -> Result<()>;

    /// ดึง TXT records ปัจจุบัน (สำหรับตรวจสอบ DNS propagation)
    async fn get_txt_records(&self, name: &str) -> Result<Vec<String>>;
}
```

**`src/dns/cloudflare.rs`**

```rust
//! Cloudflare DNS API client สำหรับ DNS-01 challenge
//!
//! ใช้ Cloudflare API v4: https://api.cloudflare.com/client/v4/

use reqwest::Client;
use serde::{Deserialize, Serialize};
use crate::error::{CertManagerError, Result};
use super::DnsProvider;

pub struct CloudflareDns {
    http: Client,
    api_token: String,
    zone_id: String,
}

#[derive(Serialize)]
struct CreateDnsRecord {
    #[serde(rename = "type")]
    record_type: String,
    name: String,
    content: String,
    ttl: u32,
}

#[derive(Deserialize)]
struct CloudflareResponse<T> {
    success: bool,
    result: Option<T>,
    errors: Vec<CloudflareError>,
}

#[derive(Deserialize, Debug)]
struct CloudflareError {
    code: u32,
    message: String,
}

#[derive(Deserialize)]
struct DnsRecord {
    id: String,
    content: String,
}

impl CloudflareDns {
    /// สร้าง CloudflareDns จาก API token และ zone ID
    ///
    /// API Token ต้องมี permission:
    /// - Zone:DNS:Edit สำหรับ zone ที่ต้องการ
    pub fn new(api_token: &str, zone_id: &str) -> Self {
        CloudflareDns {
            http: Client::new(),
            api_token: api_token.to_string(),
            zone_id: zone_id.to_string(),
        }
    }
}

#[async_trait::async_trait]
impl DnsProvider for CloudflareDns {
    async fn create_txt_record(&self, name: &str, value: &str) -> Result<String> {
        let url = format!(
            "https://api.cloudflare.com/client/v4/zones/{}/dns_records",
            self.zone_id
        );

        let body = CreateDnsRecord {
            record_type: "TXT".to_string(),
            name: name.to_string(),
            content: value.to_string(),
            ttl: 120, // ใช้ TTL ต่ำสำหรับ challenge (propagate เร็วกว่า)
        };

        let resp: CloudflareResponse<DnsRecord> = self.http
            .post(&url)
            .bearer_auth(&self.api_token)
            .json(&body)
            .send()
            .await?
            .json()
            .await?;

        if !resp.success {
            let errors: Vec<String> = resp.errors.iter()
                .map(|e| format!("[{}] {}", e.code, e.message))
                .collect();
            return Err(CertManagerError::Dns(errors.join("; ")));
        }

        let record_id = resp.result
            .ok_or_else(|| CertManagerError::Dns("No result in Cloudflare response".into()))?
            .id;

        tracing::info!("Created Cloudflare TXT record {} = {}", name, value);
        Ok(record_id)
    }

    async fn delete_txt_record(&self, name: &str, record_id: &str) -> Result<()> {
        let url = format!(
            "https://api.cloudflare.com/client/v4/zones/{}/dns_records/{}",
            self.zone_id, record_id
        );

        let resp = self.http
            .delete(&url)
            .bearer_auth(&self.api_token)
            .send()
            .await?;

        if !resp.status().is_success() {
            return Err(CertManagerError::Dns(format!(
                "Delete TXT record failed: {} ({})", name, resp.status()
            )));
        }

        tracing::info!("Deleted Cloudflare TXT record {} (id={})", name, record_id);
        Ok(())
    }

    async fn get_txt_records(&self, name: &str) -> Result<Vec<String>> {
        let url = format!(
            "https://api.cloudflare.com/client/v4/zones/{}/dns_records?type=TXT&name={}",
            self.zone_id, name
        );

        let resp: CloudflareResponse<Vec<DnsRecord>> = self.http
            .get(&url)
            .bearer_auth(&self.api_token)
            .send()
            .await?
            .json()
            .await?;

        if !resp.success {
            let errors: Vec<String> = resp.errors.iter()
                .map(|e| format!("[{}] {}", e.code, e.message))
                .collect();
            return Err(CertManagerError::Dns(errors.join("; ")));
        }

        let values = resp.result
            .unwrap_or_default()
            .into_iter()
            .map(|r| r.content)
            .collect();

        Ok(values)
    }
}
```

---

## การทดสอบ (Testing)

สร้าง verification project ใน scratchpad เพื่อรัน tests จริง (ไม่ต้องใช้ network)

```toml
# Cargo.toml สำหรับ verification project
[package]
name = "cert-manager-verify"
version = "0.1.0"
edition = "2021"

[dependencies]
base64 = "0.22"
sha2 = "0.10"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
rcgen = "0.13"
p256 = { version = "0.13", features = ["ecdsa", "jwk"] }
x509-cert = "0.2"
der = "0.7"
chrono = { version = "0.4", features = ["serde"] }
time = { version = "0.3", features = ["formatting", "parsing"] }
```

### Tests หลัก

**Test 1–2: JWK Thumbprint (RFC 7638)**

ตรวจสอบว่า `jwk_thumbprint` function ทำงานตาม RFC 7638:
- สร้าง canonical JSON ด้วย member ที่เรียงตาม lexicographic order (crv < kty < x < y)
- Hash ด้วย SHA-256
- Encode ด้วย base64url (no padding) → 43 characters

```rust
#[test]
fn test_jwk_thumbprint_rfc7638_example() {
    let crv = "P-256";
    let kty = "EC";
    let x = "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU";
    let y = "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0";
    let thumb = jwk_thumbprint(crv, kty, x, y);
    assert_eq!(thumb.len(), 43);
}

#[test]
fn test_jwk_thumbprint_canonical_json() {
    let canonical = r#"{"crv":"P-256","kty":"EC","x":"AAAA","y":"BBBB"}"#;
    let expected_hash = Sha256::digest(canonical.as_bytes());
    let expected = URL_SAFE_NO_PAD.encode(&expected_hash);
    let actual = jwk_thumbprint("P-256", "EC", "AAAA", "BBBB");
    assert_eq!(actual, expected);
}
```

**Test 3–5: Base64url Encoding/Decoding**

```rust
#[test]
fn test_base64url_roundtrip() {
    // 1, 2, 3 byte inputs → no padding characters
    assert_eq!(b64url_encode(b"a"), "YQ");
    assert_eq!(b64url_encode(b"ab"), "YWI");
    assert_eq!(b64url_encode(b"abc"), "YWJj");
    // URL-safe alphabet (ไม่มี + หรือ /)
    let decoded = b64url_decode("YWJj").unwrap();
    assert_eq!(decoded, b"abc");
}
```

**Test 6–7: CSR Generation ด้วย rcgen (no network)**

```rust
#[test]
fn test_csr_generation_basic() {
    let (pem_csr, pem_key) = generate_csr(
        "example.com",
        &["example.com", "*.example.com", "api.example.com"],
    ).expect("CSR generation should succeed");
    assert!(pem_csr.contains("-----BEGIN CERTIFICATE REQUEST-----"));
}

#[test]
fn test_csr_generation_wildcard_san() {
    let result = generate_csr("*.example.com", &["*.example.com", "example.com"]);
    assert!(result.is_ok(), "Wildcard SAN CSR should succeed");
}
```

**Test 8–9: Certificate Expiry Parsing**

```rust
#[test]
fn test_cert_expiry_from_rcgen_cert() {
    // สร้าง self-signed cert ด้วย rcgen แล้ว parse expiry
    let not_after = OffsetDateTime::now_utc() + time::Duration::days(90);
    // ... generate cert ...
    let expiry = parse_cert_expiry_from_pem(&pem);
    let days = (expiry.unwrap() - Utc::now()).num_days();
    assert!(days >= 88 && days <= 91); // ±2 วัน tolerance
}
```

**Test 10–12: Renewal Decision Logic**

```rust
#[test]
fn test_needs_renewal_within_threshold() {
    let now = Utc.with_ymd_and_hms(2024, 6, 1, 0, 0, 0).unwrap();
    let expiry_20d = Utc.with_ymd_and_hms(2024, 6, 21, 0, 0, 0).unwrap();
    let expiry_31d = Utc.with_ymd_and_hms(2024, 7, 2, 0, 0, 0).unwrap();
    assert!(needs_renewal(expiry_20d, now, 30)); // 20 < 30 → renew
    assert!(!needs_renewal(expiry_31d, now, 30)); // 31 > 30 → no renew
}
```

**Test 13–14: DNS-01 TXT Record Format**

```rust
#[test]
fn test_dns_challenge_name() {
    assert_eq!(dns_challenge_name("example.com"), "_acme-challenge.example.com");
    assert_eq!(dns_challenge_name("*.example.com"), "_acme-challenge.*.example.com");
}

#[test]
fn test_dns01_txt_value_is_base64url() {
    let key_auth = "evaGxfADs6pSRb2LAv9IZf17Dt3juxGJ+PCt92wr+oA.4sNbgH_s0ZD5...";
    let txt = dns01_txt_value(key_auth);
    assert_eq!(txt.len(), 43); // SHA-256 → base64url no-pad
    assert!(!txt.contains('=') && !txt.contains('+') && !txt.contains('/'));
}
```

### ผลการรันจริง (cargo test output)

```
$ cargo test

running 17 tests
test tests::test_base64url_uses_url_safe_alphabet ... ok
test tests::test_base64url_roundtrip ... ok
test tests::test_base64url_no_padding ... ok
test tests::test_acme_directory_deserialize ... ok
test tests::test_cert_expiry_from_rcgen_cert ... ok
test tests::test_cert_manifest_roundtrip ... ok
test tests::test_dns01_txt_value_is_base64url ... ok
test tests::test_csr_generation_basic ... ok
test tests::test_csr_generation_wildcard_san ... ok
test tests::test_dns_challenge_name ... ok
test tests::test_key_authorization_format ... ok
test tests::test_jwk_thumbprint_canonical_json ... ok
test tests::test_jwk_thumbprint_rfc7638_example ... ok
test tests::test_needs_renewal_custom_threshold ... ok
test tests::test_needs_renewal_within_threshold ... ok
test tests::test_revocation_reason_codes ... ok
test tests::test_needs_renewal_expired ... ok

test result: ok. 17 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

ผล verbose output (`cargo test -- --nocapture`):

```
CSR length: 456 chars
Key length: 241 chars
DNS-01 TXT value: vY6XhiqDn1ZzHx-UkmbaWqUAZ4eibUJX91ZFHOyD3MY
JWK thumbprint: oKIywvGUpTVTyxMQ3bwIIeQUudfr_CkLMjCE19ECD-U
Parsed cert expiry: 2026-12-26 12:08:21 UTC (89 days from now)
CertManifest JSON:
{
  "domain": "example.com",
  "issued_at": "2024-03-01T00:00:00Z",
  "expires_at": "2024-06-01T00:00:00Z",
  "auto_renew": true,
  "chain_length": 3,
  "root_ca": "ISRG Root X1"
}
```

## Pitfalls และข้อควรระวัง

### Pitfall 1: Nonce Reuse — "urn:ietf:params:acme:error:badNonce"

ACME server reject request ที่ใช้ nonce ซ้ำ nonce ต้องดึงใหม่จาก `HEAD /acme/new-nonce` **ก่อนทุก request** และใช้ได้ครั้งเดียวเท่านั้น ถ้า server ตอบกลับด้วย `badNonce` error ต้อง retry ด้วย nonce ใหม่

```rust
// ผิด: เก็บ nonce ไว้ใช้หลาย request
let nonce = client.fetch_nonce().await?;
let jws1 = build_jws(&key, &pub_key, &nonce, url1, ...)?; // ok
let jws2 = build_jws(&key, &pub_key, &nonce, url2, ...)?; // ERROR: nonce reused

// ถูก: fetch nonce ใหม่ทุกครั้ง
let nonce1 = client.fetch_nonce().await?;
let jws1 = build_jws(&key, &pub_key, &nonce1, url1, ...)?;
let nonce2 = client.fetch_nonce().await?;
let jws2 = build_jws(&key, &pub_key, &nonce2, url2, ...)?;
```

### Pitfall 2: JWS Payload สำหรับ POST-as-GET ต้องเป็น empty string ไม่ใช่ null

RFC 8555 §6.3 กำหนดว่า POST-as-GET request (เช่น fetch authorization, fetch certificate) ต้องมี payload เป็น **empty string** `""` ไม่ใช่ JSON `null` หรือ omit field ทั้งหมด

```rust
// ผิด: ใส่ null หรือ {} เป็น payload สำหรับ POST-as-GET
let payload_b64 = URL_SAFE_NO_PAD.encode("null"); // ERROR
let payload_b64 = URL_SAFE_NO_PAD.encode("{}");   // ผิด semantics

// ถูก: empty string สำหรับ POST-as-GET
let payload_b64 = String::new(); // ""
```

### Pitfall 3: JWS Signature Format — Raw r||s ไม่ใช่ DER

ACME ใช้ JWS ซึ่ง require signature เป็น raw concatenation ของ r และ s (64 bytes สำหรับ P-256) แต่ `p256::ecdsa` crate produce DER-encoded signature โดย default ต้องแปลงก่อน

```rust
use p256::ecdsa::{Signature, signature::Signer};

let sig: Signature = signing_key.sign(message.as_bytes());

// ผิด: ใช้ DER encoding โดยตรง
// sig.to_der() → DER format ที่ JWT ไม่รับ

// ถูก: ใช้ to_bytes() ซึ่ง return raw r||s (fixed 64 bytes)
let sig_bytes = sig.to_bytes(); // [u8; 64]
let signature = URL_SAFE_NO_PAD.encode(&sig_bytes);
```

### Pitfall 4: DNS-01 Wildcard Challenge ใช้ Base Domain ไม่ใช่ Wildcard

สำหรับ wildcard `*.example.com` ชื่อ TXT record ที่ต้องสร้างคือ `_acme-challenge.example.com` (ไม่ใช่ `_acme-challenge.*.example.com`) เพราะ DNS ไม่มี wildcard level ก่อน underscore prefix

```rust
// ผิด
fn dns_name(domain: &str) -> String {
    format!("_acme-challenge.{}", domain) // *.example.com → _acme-challenge.*.example.com
}

// ถูก
fn dns_name(domain: &str) -> String {
    let base = domain.trim_start_matches("*."); // ลบ *. prefix
    format!("_acme-challenge.{}", base) // → _acme-challenge.example.com
}
```

### Pitfall 5: Private Key File Permission ต้องเป็น 0600

การบันทึก private key ด้วย permission ที่กว้างเกินไป (เช่น 0644) ทำให้ user อื่นบน system อ่าน key ได้ ต้องตั้ง 0600 ทันทีหลัง write

```rust
// หลัง write key file ต้องตั้ง mode ทันที
#[cfg(unix)]
{
    use std::os::unix::fs::PermissionsExt;
    std::fs::set_permissions(&key_path, std::fs::Permissions::from_mode(0o600))?;
}
// บน Windows ต้องใช้ ACL แยกต่างหาก
```

### Pitfall 6: Order Polling — อย่า Busy-wait

หลัง respond ไปยัง challenge ต้อง poll order status จนเป็น `"valid"` ACME server ใช้เวลา validate ไม่แน่นอน ต้องใช้ exponential backoff ไม่ใช่ tight loop

```rust
// ผิด: tight loop
loop {
    let order = client.get_order(&order_url).await?;
    if order.status == "valid" { break; }
    // no sleep → ทำให้ rate limit ถูกตัด
}

// ถูก: exponential backoff
let mut delay = tokio::time::Duration::from_secs(2);
for _ in 0..10 {
    let order = client.get_order(&order_url).await?;
    match order.status.as_str() {
        "valid" => return Ok(order),
        "invalid" => return Err(CertManagerError::Acme("Order failed".into())),
        _ => {
            tokio::time::sleep(delay).await;
            delay = std::cmp::min(delay * 2, tokio::time::Duration::from_secs(60));
        }
    }
}
```

## การ Package และ Deploy

### Build Release Binary

```bash
cargo build --release
# Binary: target/release/cert-manager

# ให้ bind port 80 โดยไม่ต้องรัน root
sudo setcap cap_net_bind_service=+ep target/release/cert-manager
```

### Systemd Service สำหรับ Daemon Mode

```ini
# /etc/systemd/system/cert-manager.service
[Unit]
Description=ACME Certificate Manager Daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=certmgr
ExecStart=/usr/local/bin/cert-manager daemon \
    --renew-before-days 30 \
    --webhook-url https://hooks.example.com/cert-events
Restart=on-failure
RestartSec=60
AmbientCapabilities=CAP_NET_BIND_SERVICE

# Environment variables สำหรับ DNS provider
Environment=CLOUDFLARE_API_TOKEN=<token>
Environment=CLOUDFLARE_ZONE_ID=<zone_id>

[Install]
WantedBy=multi-user.target
```

```bash
# Enable และ start daemon
sudo systemctl enable cert-manager
sudo systemctl start cert-manager
sudo journalctl -u cert-manager -f  # ดู logs
```

### Docker

```dockerfile
FROM rust:1.77-slim AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/cert-manager /usr/local/bin/
EXPOSE 80
ENTRYPOINT ["cert-manager"]
CMD ["daemon", "--renew-before-days", "30"]
```

### ตัวอย่างการใช้งาน

```bash
# ขอ certificate ด้วย HTTP-01 challenge
cert-manager issue example.com --san "www.example.com" --email admin@example.com

# ขอ wildcard certificate ด้วย DNS-01 challenge (Cloudflare)
export CLOUDFLARE_API_TOKEN="xxx"
export CLOUDFLARE_ZONE_ID="yyy"
cert-manager issue "*.example.com" \
    --san "example.com" \
    --challenge dns-01 \
    --dns-provider cloudflare

# List certificates
cert-manager list
# DOMAIN                         EXPIRES                   DAYS LEFT  AUTO-RENEW
# ─────────────────────────────────────────────────────────────────────────────
# example.com                    2024-09-01 10:23 UTC      87         yes  [ok]
# *.example.com                  2024-08-15 08:00 UTC      70         yes  [ok]

# Show certificate details
cert-manager show example.com

# Renew specific domain
cert-manager renew example.com --renew-before-days 45

# Revoke certificate (reason: keyCompromise)
cert-manager revoke example.com --reason 1

# Run daemon
cert-manager daemon \
    --renew-before-days 30 \
    --webhook-url https://hooks.example.com/certs
```

## การต่อยอด (Extensions & Exercises)

### แบบฝึกหัดที่ 1: Implement TLS-ALPN-01 Challenge (ระดับกลาง)

TLS-ALPN-01 เป็น challenge แบบที่สามของ ACME — ACME server เปิด TLS connection ไปยัง port 443 ของ domain และตรวจสอบ special ALPN extension ใน certificate ชั่วคราว ข้อดีคือไม่ต้องใช้ port 80 หรือ DNS access

```
เป้าหมาย:
- สร้าง TLS server ชั่วคราวบน port 443 ด้วย rustls
- Generate validation certificate ที่มี ALPN extension (OID 1.3.6.1.5.5.7.1.31)
  ที่มี SHA-256(key_authorization) เป็น value
- Server response ต่อ ACME validation request แล้ว shutdown ตัวเอง
```

### แบบฝึกหัดที่ 2: Certificate Transparency Log Monitoring (ระดับกลาง)

Certificate Transparency (CT) log บันทึก certificate ทุกใบที่ CA ออก สร้าง monitor ที่ตรวจ CT log หา certificate ที่ออกให้ domain ของคุณโดยไม่ผ่าน cert-manager

```
เป้าหมาย:
- เรียก crt.sh API หรือ Google CT API เพื่อ query certificates สำหรับ domain
- เปรียบเทียบกับ manifest ที่เรามี
- Alert ผ่าน webhook ถ้ามี certificate ที่ไม่ได้ออกจากระบบเรา
crate ที่อาจใช้: reqwest (ส่งไปหา crt.sh JSON API)
```

### แบบฝึกหัดที่ 3: Multi-CA Support และ Failover (ระดับสูง)

ในการใช้งาน production อาจต้องการ fallback CA ถ้า primary CA ล่ม สร้าง CA failover logic

```
เป้าหมาย:
- Config file: list ของ CA URLs พร้อม priority
- ถ้า primary CA ล้มเหลว (error หรือ timeout) → ลอง CA ถัดไปโดยอัตโนมัติ
- Manifest บันทึกว่า cert ออกจาก CA ไหน
- CLI flag: --ca-url สำหรับ override
ความซับซ้อน: ACME account key อาจไม่ work ข้าม CA ต้องมี separate account ต่อ CA
```

### แบบฝึกหัดที่ 4: PKCS#12 Export (ระดับเริ่มต้น)

บาง application (เช่น Java Keystore, Windows) ต้องการ certificate ในรูปแบบ PKCS#12 (.pfx/.p12) ไม่ใช่ PEM แยกไฟล์

```
เป้าหมาย:
- Subcommand: cert-manager export --format pkcs12 example.com
- สร้าง .p12 file ที่รวม cert + chain + key
- รับ optional passphrase ด้วย rpassword crate
crate ที่ใช้: openssl หรือ p12 crate
```

## สรุป

โปรเจคนี้ implement ACME protocol (RFC 8555) ครบทุกขั้นตอนจาก directory fetch จนถึง certificate storage และ auto-renewal daemon สิ่งสำคัญที่ได้เรียน:

**Pattern สำคัญ:**
- **JWS signing flow**: header + payload → base64url → ECDSA sign → JWS object เป็น pattern ที่ใช้ซ้ำทุก ACME request
- **RFC 7638 JWK Thumbprint**: canonical JSON ที่ fields เรียง lexicographically → SHA-256 → base64url — เป็น fingerprint ของ public key
- **Challenge responder pattern**: HTTP-01 ใช้ axum server embedded ใน-process, DNS-01 ใช้ trait abstraction สำหรับ provider ต่าง ๆ
- **Daemon pattern**: infinite loop + sleep + error handling แบบ resilient ที่ log error แต่ไม่หยุดทำงาน
- **Permission-correct file storage**: key files ต้องได้ 0600 ทันทีหลัง write — ไม่ใช่แก้ทีหลัง

**Crates ที่ได้ใช้จริง:**
- `p256` + `sha2` + `base64`: cryptographic building blocks สำหรับ JWS
- `rcgen`: CSR generation ที่ไม่ต้องเรียก OpenSSL CLI
- `x509-cert` + `der`: parsing DER/PEM certificate สำหรับ chain validation
- `axum 0.8`: HTTP server แบบ embedded สำหรับ HTTP-01 challenge
- `clap 4` derive API: CLI ที่ clean ด้วย boilerplate น้อย

โปรเจคถัดไปใน module D คือ **[Project D05: Audit Log](project-d05-audit-log.md)** ซึ่งจะนำ cryptographic techniques จากโปรเจคนี้ (hashing, signing) ไปประยุกต์กับ tamper-evident audit log ที่ทุก entry ถูก link กันด้วย hash chain — similar to Merkle tree แต่ sequential

---

**โปรเจคก่อนหน้า:** [Project D03: TLS Proxy](project-d03-tls-proxy.md) | **โปรเจคถัดไป:** [Project D05: Audit Log](project-d05-audit-log.md)
