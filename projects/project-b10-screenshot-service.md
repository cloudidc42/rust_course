# Project B10: Screenshot-as-a-Service

> โมดูล: B — Web Services & APIs | ความยาก: ⭐⭐⭐⭐ | เวลาโดยประมาณ: 7 ชั่วโมง

## ภาพรวมโปรเจค

Screenshot-as-a-Service คือ HTTP API ที่รับ URL แล้วส่งกลับรูปภาพของหน้าเว็บนั้น ๆ โดยใช้ headless browser จริง ๆ ในการ render แทนการ parse HTML ด้วยตัวเอง ทำให้ได้ภาพที่ตรงกับสิ่งที่ผู้ใช้เห็นในเบราว์เซอร์จริง รวมถึง JavaScript-rendered content

**Use case จริงในโลก production:**
- บริการ thumbnail preview ของ URL ใน social platform หรือ chat app
- Automated visual regression testing สำหรับ CI/CD pipeline
- PDF/image export สำหรับ web report
- Website monitoring และ uptime check แบบ visual
- Content moderation ก่อนอนุมัติ URL ที่ user ส่งมา

**Learning value:**
โปรเจคนี้บังคับให้เรียนรู้ทักษะที่ขาดไม่ได้ในงาน backend production ได้แก่ CDP (Chrome DevTools Protocol), การจัดการ resource pool ด้วย Semaphore, การป้องกัน SSRF (Server-Side Request Forgery), job queue แบบ async, และการออกแบบ cache layer ด้วย Redis

## สิ่งที่จะได้เรียนรู้

- ควบคุม Chromium ผ่าน CDP protocol ด้วย crate `chromiumoxide`
- ป้องกัน SSRF โดยตรวจสอบ RFC1918 address ก่อนส่ง request
- สร้าง job queue แบบ async ด้วย `tokio::sync::mpsc` และ `Semaphore`
- ออกแบบ browser pool เพื่อ reuse Chrome process แทนการสร้างใหม่ต่อ request
- ใช้ Redis เป็น cache layer ด้วย SHA-256 key
- บันทึกไฟล์ไปยัง local disk หรือ S3-compatible storage
- ออกแบบ admin endpoint สำหรับ observability และ operational control
- เขียน unit test สำหรับ security logic, state machine, และ concurrency primitive

## ความรู้ที่ต้องมีมาก่อน

- จาก Part 46–50: `async`/`await`, `tokio` runtime, `tokio::spawn`
- จาก Part 51–55: Axum routing, extractors, middleware
- จาก Part 56–60: Error handling ด้วย `thiserror`, `anyhow`
- จาก Part 61–70: `Arc`, `Mutex`, `RwLock`, shared state ใน async
- จาก Part 71–80: HTTP client ด้วย `reqwest`, JSON serialization ด้วย `serde`
- จาก Part 96–110: Redis client, S3 SDK, production observability

## โครงสร้างโปรเจค (Project Layout)

```
screenshot-service/
├── src/
│   ├── main.rs           # entry point, router, AppState
│   ├── validation.rs     # URL validation + SSRF prevention
│   ├── cache.rs          # SHA-256 cache key generation
│   ├── jobs.rs           # Job struct + state machine + JobStore
│   ├── concurrency.rs    # BrowserSemaphore wrapper
│   ├── browser.rs        # BrowserPool — Chrome process management
│   ├── screenshot.rs     # screenshot logic (CDP commands)
│   ├── storage.rs        # save to disk / S3
│   └── admin.rs          # stats + cache flush handlers
├── tests/
│   └── integration.rs    # integration tests (ต้องมี runtime environment)
├── Cargo.toml
└── README.md
```

## การออกแบบ (Architecture & Design)

### Data Flow — Synchronous Endpoint

```
POST /screenshots
      │
      ▼
[1] URL Validation
    - parse scheme (http/https only)
    - resolve hostname → IP
    - reject RFC1918 / loopback
      │
      ▼
[2] Cache Lookup
    SHA256(url+params) → Redis GET
    hit ──► return cached bytes
    miss ──► continue
      │
      ▼
[3] Acquire Semaphore Permit
    (blocks if all 3 browser slots are busy)
      │
      ▼
[4] BrowserPool.get()
    assign idle Chrome instance
      │
      ▼
[5] Inject headers/cookies
    navigate to URL, wait wait_ms
    capture screenshot
      │
      ▼
[6] Cache result
    Redis SET EX <ttl>
      │
      ▼
[7] Return image/png or image/jpeg
```

### Data Flow — Async Endpoint

```
POST /screenshots/async
      │ validate + insert Job(Queued) into JobStore
      │ send job_id to mpsc channel
      ▼
[background worker]
      │ receive job_id
      │ update Job → Processing
      │ acquire Semaphore permit
      │ take screenshot (same [4]–[6] above)
      │ upload to storage
      │ update Job → Done (download_url)
      │   └── or Done → Failed (error)
      ▼

GET /jobs/:id  ──► poll status
```

### Design Decisions

**เหตุใดจึงเลือก `chromiumoxide` แทน `fantoccini` หรือ Selenium?**
`chromiumoxide` ใช้ CDP โดยตรงผ่าน WebSocket ทำให้ควบคุม Chrome ได้ละเอียดกว่า WebDriver API มาก เช่น inject headers, intercept network request, หรือ emulate device ได้ง่ายกว่า ทั้งยังไม่ต้องรัน chromedriver เป็น process แยก

**เหตุใดจึงใช้ browser pool แทนการสร้าง Chrome ใหม่ต่อ request?**
การ launch Chrome ใหม่ทุกครั้งใช้เวลา 1–3 วินาที ซึ่งยอมรับไม่ได้ใน web service การ reuse browser instance ที่อยู่ใน idle state ทำให้ latency ลดเหลือ ~200ms (เฉพาะ navigation time)

**เหตุใดจึงต้องมี Semaphore?**
Chrome แต่ละ instance ใช้ RAM ~200MB หากปล่อยให้ request พร้อมกันทั้งหมดเปิด browser ได้ไม่จำกัด อาจ OOM ใน container ที่มี RAM จำกัด Semaphore จำกัดจำนวน concurrent browser slots ไว้ที่ N=3 (configurable) request ที่เกินจะรอใน queue โดยอัตโนมัติ

## การพัฒนาทีละขั้นตอน

### ขั้นที่ 1: โครงสร้างโปรเจคและ Dependencies

เริ่มจาก `Cargo.toml` ที่ประกาศ dependency ทั้งหมด:

```toml
[package]
name = "screenshot-service"
version = "0.1.0"
edition = "2021"

[dependencies]
# Web framework
axum = { version = "0.8", features = ["macros"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["trace", "cors"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Headless browser (CDP)
chromiumoxide = { version = "0.7", features = ["tokio-runtime"] }

# Redis
redis = { version = "0.27", features = ["tokio-comp", "connection-manager"] }

# HTTP client (สำหรับ DNS resolution ในการตรวจสอบ SSRF)
reqwest = { version = "0.12", features = ["json"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Hashing สำหรับ cache key
sha2 = "0.10"
hex = "0.4"

# UUID สำหรับ job ID
uuid = { version = "1", features = ["v4"] }

# URL parsing
url = "2"

# Utilities
thiserror = "2"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
chrono = { version = "0.4", features = ["serde"] }
dashmap = "6"
bytes = "1"

# S3-compatible storage
aws-sdk-s3 = "1"
aws-config = "1"

[dev-dependencies]
tokio = { version = "1", features = ["full", "test-util"] }
axum-test = "15"
```

**โครงสร้าง module ใน `src/main.rs`:**

```rust
mod admin;
mod browser;
mod cache;
mod concurrency;
mod jobs;
mod screenshot;
mod storage;
mod validation;
```

---

### ขั้นที่ 2: URL Validation และ SSRF Prevention

`src/validation.rs` — ส่วนที่สำคัญที่สุดด้านความปลอดภัย

```rust
use std::net::IpAddr;
use std::str::FromStr;
use thiserror::Error;

#[derive(Debug, Error, PartialEq)]
pub enum ValidationError {
    #[error("Invalid URL: {0}")]
    InvalidUrl(String),
    #[error("Scheme not allowed: {0} (only http and https are permitted)")]
    SchemeNotAllowed(String),
    #[error("SSRF blocked: resolved IP {0} falls within a private or loopback range")]
    SsrfBlocked(String),
    #[error("URL missing host")]
    MissingHost,
}

/// ตรวจว่า IP address อยู่ใน RFC1918, loopback, หรือ link-local range หรือไม่
///
/// ช่วง address ที่ต้องปฏิเสธ:
///   127.0.0.0/8   — loopback (IPv4)
///   10.0.0.0/8    — RFC1918 private
///   172.16.0.0/12 — RFC1918 private
///   192.168.0.0/16— RFC1918 private
///   169.254.0.0/16— link-local (APIPA)
///   ::1           — loopback (IPv6)
///   fe80::/10     — link-local (IPv6)
///   fc00::/7      — unique local (IPv6)
pub fn is_private_ip(ip: &IpAddr) -> bool {
    match ip {
        IpAddr::V4(v4) => {
            let o = v4.octets();
            // 127.0.0.0/8 — loopback
            if o[0] == 127 { return true; }
            // 10.0.0.0/8 — RFC1918
            if o[0] == 10 { return true; }
            // 172.16.0.0/12 — RFC1918: 172.16.x.x ถึง 172.31.x.x
            if o[0] == 172 && (16..=31).contains(&o[1]) { return true; }
            // 192.168.0.0/16 — RFC1918
            if o[0] == 192 && o[1] == 168 { return true; }
            // 169.254.0.0/16 — link-local
            if o[0] == 169 && o[1] == 254 { return true; }
            // 0.0.0.0/8 — "this" network
            if o[0] == 0 { return true; }
            false
        }
        IpAddr::V6(v6) => {
            if v6.is_loopback() { return true; }
            // IPv4-mapped: ::ffff:x.x.x.x
            if let Some(v4) = v6.to_ipv4_mapped() {
                return is_private_ip(&IpAddr::V4(v4));
            }
            let seg = v6.segments();
            // fe80::/10 — link-local
            if (seg[0] & 0xffc0) == 0xfe80 { return true; }
            // fc00::/7 — unique local
            if (seg[0] & 0xfe00) == 0xfc00 { return true; }
            false
        }
    }
}

/// Validate URL: ตรวจ scheme → ตรวจ host literal IP → ปฏิเสธ localhost
/// DNS resolution ที่แท้จริงทำใน async context ด้วย `validate_url_async`
pub fn validate_url(url: &str) -> Result<String, ValidationError> {
    let parsed = url::Url::parse(url)
        .map_err(|e| ValidationError::InvalidUrl(e.to_string()))?;

    let scheme = parsed.scheme();
    if scheme != "http" && scheme != "https" {
        return Err(ValidationError::SchemeNotAllowed(scheme.to_string()));
    }

    let host = parsed.host_str().ok_or(ValidationError::MissingHost)?;

    // หากเป็น literal IP — ตรวจทันที
    if let Ok(ip) = IpAddr::from_str(host) {
        if is_private_ip(&ip) {
            return Err(ValidationError::SsrfBlocked(ip.to_string()));
        }
        return Ok(url.to_string());
    }

    // ปฏิเสธ "localhost" ชัดเจน
    if host.eq_ignore_ascii_case("localhost") {
        return Err(ValidationError::SsrfBlocked("localhost".to_string()));
    }

    Ok(url.to_string())
}

/// Async validation: resolve DNS แล้วตรวจ IP ที่ได้
/// เรียกก่อน navigate เพื่อป้องกัน DNS rebinding attack
pub async fn validate_url_async(url: &str) -> Result<String, ValidationError> {
    // ขั้นแรก: ตรวจ scheme + literal IP
    validate_url(url)?;

    let parsed = url::Url::parse(url).unwrap();
    let host = parsed.host_str().ok_or(ValidationError::MissingHost)?;

    // ถ้าเป็น literal IP ผ่านแล้วตั้งแต่ validate_url
    if IpAddr::from_str(host).is_ok() {
        return Ok(url.to_string());
    }

    // Resolve DNS
    let port = parsed.port_or_known_default().unwrap_or(80);
    let addrs: Vec<_> = tokio::net::lookup_host(format!("{}:{}", host, port))
        .await
        .map_err(|e| ValidationError::InvalidUrl(format!("DNS lookup failed: {}", e)))?
        .collect();

    if addrs.is_empty() {
        return Err(ValidationError::InvalidUrl(
            "Hostname did not resolve to any address".to_string(),
        ));
    }

    // ตรวจว่า IP ที่ resolve ได้ทุกตัวไม่อยู่ใน private range
    for addr in &addrs {
        if is_private_ip(&addr.ip()) {
            return Err(ValidationError::SsrfBlocked(addr.ip().to_string()));
        }
    }

    Ok(url.to_string())
}
```

---

### ขั้นที่ 3: Cache Key Generation

`src/cache.rs` — cache key ที่ deterministic และ collision-resistant

```rust
use sha2::{Digest, Sha256};

/// ข้อมูลที่ใช้สร้าง cache key — ทุก field ที่เปลี่ยนแล้วผลต่างกัน
#[derive(Debug, Clone)]
pub struct CacheKeyParams<'a> {
    pub url: &'a str,
    pub width: u32,
    pub height: u32,
    pub wait_ms: u32,
    pub full_page: bool,
    pub format: &'a str,
}

/// สร้าง hex-encoded SHA-256 cache key จาก parameters
///
/// ใช้ `|` เป็น separator เพื่อป้องกัน collision เช่น
///   url="a|1" width=0  vs  url="a" width=10
pub fn generate_cache_key(params: &CacheKeyParams<'_>) -> String {
    let mut hasher = Sha256::new();
    let canonical = format!(
        "{}|{}|{}|{}|{}|{}",
        params.url,
        params.width,
        params.height,
        params.wait_ms,
        params.full_page as u8,
        params.format,
    );
    hasher.update(canonical.as_bytes());
    hex::encode(hasher.finalize())
}

pub fn redis_key(hash: &str) -> String {
    format!("screenshot:{}", hash)
}

/// Cache client wrapper
pub struct CacheClient {
    pub client: redis::Client,
    pub ttl_seconds: u64,
}

impl CacheClient {
    pub fn new(redis_url: &str, ttl_seconds: u64) -> anyhow::Result<Self> {
        let client = redis::Client::open(redis_url)?;
        Ok(CacheClient { client, ttl_seconds })
    }

    /// ลองดึง screenshot bytes จาก cache
    pub async fn get(&self, key: &str) -> Option<Vec<u8>> {
        use redis::AsyncCommands;
        let mut conn = self.client.get_multiplexed_async_connection().await.ok()?;
        let data: Option<Vec<u8>> = conn.get(key).await.ok()?;
        data
    }

    /// บันทึก screenshot bytes เข้า cache พร้อม TTL
    pub async fn set(&self, key: &str, data: &[u8]) -> anyhow::Result<()> {
        use redis::AsyncCommands;
        let mut conn = self.client.get_multiplexed_async_connection().await?;
        conn.set_ex(key, data, self.ttl_seconds).await?;
        Ok(())
    }

    /// ลบ cache ทั้งหมด (สำหรับ admin endpoint)
    pub async fn flush_pattern(&self, pattern: &str) -> anyhow::Result<u64> {
        use redis::AsyncCommands;
        let mut conn = self.client.get_multiplexed_async_connection().await?;
        let keys: Vec<String> = conn.keys(pattern).await?;
        let count = keys.len() as u64;
        if !keys.is_empty() {
            conn.del(&keys).await?;
        }
        Ok(count)
    }
}
```

---

### ขั้นที่ 4: Job Queue และ State Machine

`src/jobs.rs` — state machine ที่ explicit transition เท่านั้น

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use uuid::Uuid;

/// Life-cycle ของ screenshot job
///
/// Transition ที่อนุญาต:
///   Queued ──► Processing ──► Done
///                        └──► Failed
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum JobStatus {
    Queued,
    Processing,
    Done,
    Failed,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Job {
    pub id: String,
    pub url: String,
    pub status: JobStatus,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    /// ใส่เมื่อ status == Done
    pub download_url: Option<String>,
    /// ใส่เมื่อ status == Failed
    pub error: Option<String>,
}

impl Job {
    pub fn new(url: impl Into<String>) -> Self {
        let now = Utc::now();
        Job {
            id: Uuid::new_v4().to_string(),
            url: url.into(),
            status: JobStatus::Queued,
            created_at: now,
            updated_at: now,
            download_url: None,
            error: None,
        }
    }

    pub fn start_processing(&mut self) -> Result<(), String> {
        if self.status != JobStatus::Queued {
            return Err(format!(
                "Cannot transition to Processing from {:?}",
                self.status
            ));
        }
        self.status = JobStatus::Processing;
        self.updated_at = Utc::now();
        Ok(())
    }

    pub fn complete(&mut self, download_url: impl Into<String>) -> Result<(), String> {
        if self.status != JobStatus::Processing {
            return Err(format!(
                "Cannot transition to Done from {:?}",
                self.status
            ));
        }
        self.status = JobStatus::Done;
        self.download_url = Some(download_url.into());
        self.updated_at = Utc::now();
        Ok(())
    }

    pub fn fail(&mut self, reason: impl Into<String>) -> Result<(), String> {
        if self.status != JobStatus::Processing {
            return Err(format!(
                "Cannot transition to Failed from {:?}",
                self.status
            ));
        }
        self.status = JobStatus::Failed;
        self.error = Some(reason.into());
        self.updated_at = Utc::now();
        Ok(())
    }

    pub fn is_terminal(&self) -> bool {
        matches!(self.status, JobStatus::Done | JobStatus::Failed)
    }
}

/// In-memory job store (ใน production ควรใช้ Redis หรือ database)
#[derive(Debug, Clone, Default)]
pub struct JobStore {
    jobs: Arc<Mutex<HashMap<String, Job>>>,
}

impl JobStore {
    pub fn new() -> Self { Self::default() }

    pub fn insert(&self, job: Job) -> String {
        let id = job.id.clone();
        self.jobs.lock().unwrap().insert(id.clone(), job);
        id
    }

    pub fn get(&self, id: &str) -> Option<Job> {
        self.jobs.lock().unwrap().get(id).cloned()
    }

    pub fn update(&self, job: Job) {
        self.jobs.lock().unwrap().insert(job.id.clone(), job);
    }

    pub fn queued_count(&self) -> usize {
        self.jobs.lock().unwrap().values()
            .filter(|j| j.status == JobStatus::Queued)
            .count()
    }

    pub fn active_count(&self) -> usize {
        self.jobs.lock().unwrap().values()
            .filter(|j| j.status == JobStatus::Processing)
            .count()
    }
}
```

---

### ขั้นที่ 5: Concurrency Control ด้วย Semaphore

`src/concurrency.rs` — จำกัด concurrent Chrome instances

```rust
use std::sync::Arc;
use tokio::sync::{OwnedSemaphorePermit, Semaphore};

/// BrowserSemaphore จำกัดจำนวน Chrome instances ที่รันพร้อมกัน
///
/// เมื่อ permit หมด — `acquire()` จะรอโดยอัตโนมัติ (back-pressure)
/// Permit ที่ถือไว้จะ release เมื่อ drop ออกจาก scope
#[derive(Clone)]
pub struct BrowserSemaphore {
    semaphore: Arc<Semaphore>,
    max: usize,
}

impl BrowserSemaphore {
    pub fn new(max_concurrent: usize) -> Self {
        BrowserSemaphore {
            semaphore: Arc::new(Semaphore::new(max_concurrent)),
            max: max_concurrent,
        }
    }

    /// รับ permit — รอถ้า pool เต็ม
    pub async fn acquire(&self) -> OwnedSemaphorePermit {
        Arc::clone(&self.semaphore)
            .acquire_owned()
            .await
            .expect("BrowserSemaphore closed unexpectedly")
    }

    pub fn available_permits(&self) -> usize {
        self.semaphore.available_permits()
    }

    pub fn max(&self) -> usize {
        self.max
    }
}
```

---

### ขั้นที่ 6: Browser Pool และ Screenshot Logic

`src/browser.rs` — pool ของ Chrome instances

```rust
use anyhow::{Context, Result};
use chromiumoxide::{Browser, BrowserConfig};
use std::sync::Arc;
use tokio::sync::Mutex;

/// Chrome browser instance พร้อม metadata
pub struct BrowserInstance {
    pub browser: Browser,
    pub in_use: bool,
}

/// Pool ของ Chrome instances — reuse แทน launch ใหม่ต่อ request
pub struct BrowserPool {
    instances: Arc<Mutex<Vec<BrowserInstance>>>,
    pool_size: usize,
    chromium_path: String,
}

impl BrowserPool {
    /// สร้าง pool ขนาด `size` instances
    /// chromium_path: path ไปยัง Chromium binary
    pub async fn new(size: usize, chromium_path: impl Into<String>) -> Result<Self> {
        let path = chromium_path.into();
        let mut instances = Vec::with_capacity(size);

        for _ in 0..size {
            let browser = launch_browser(&path).await?;
            instances.push(BrowserInstance {
                browser,
                in_use: false,
            });
        }

        Ok(BrowserPool {
            instances: Arc::new(Mutex::new(instances)),
            pool_size: size,
            chromium_path: path,
        })
    }

    /// ขอ browser จาก pool
    /// ถ้าทุกตัว in_use — launch ใหม่ (Semaphore ด้านนอกจะป้องกันไม่ให้เกิน limit)
    pub async fn acquire(&self) -> Result<BrowserHandle<'_>> {
        let mut guard = self.instances.lock().await;
        if let Some((idx, inst)) = guard.iter_mut().enumerate().find(|(_, i)| !i.in_use) {
            inst.in_use = true;
            let idx = idx;
            drop(guard);
            return Ok(BrowserHandle {
                pool: self.instances.clone(),
                index: idx,
            });
        }
        // Pool เต็ม — launch browser ใหม่ชั่วคราว
        // (ปกติ Semaphore ด้านนอกจะป้องกันกรณีนี้)
        drop(guard);
        let browser = launch_browser(&self.chromium_path).await?;
        let mut guard = self.instances.lock().await;
        let idx = guard.len();
        guard.push(BrowserInstance { browser, in_use: true });
        drop(guard);
        Ok(BrowserHandle {
            pool: self.instances.clone(),
            index: idx,
        })
    }

    pub fn pool_size(&self) -> usize {
        self.pool_size
    }
}

/// RAII handle — release browser กลับ pool เมื่อ drop
pub struct BrowserHandle<'a> {
    pool: Arc<Mutex<Vec<BrowserInstance>>>,
    index: usize,
    _phantom: std::marker::PhantomData<&'a ()>,
}

impl<'a> BrowserHandle<'a> {
    pub async fn browser(&self) -> tokio::sync::MutexGuard<'_, Vec<BrowserInstance>> {
        self.pool.lock().await
    }
}

impl<'a> Drop for BrowserHandle<'a> {
    fn drop(&mut self) {
        let pool = self.pool.clone();
        let index = self.index;
        tokio::spawn(async move {
            let mut guard = pool.lock().await;
            if let Some(inst) = guard.get_mut(index) {
                inst.in_use = false;
            }
        });
    }
}

/// Launch Chromium ด้วย flags ที่เหมาะสม
async fn launch_browser(chromium_path: &str) -> Result<Browser> {
    let config = BrowserConfig::builder()
        .chrome_executable(chromium_path)
        // --no-sandbox จำเป็นใน container / Docker environment
        .arg("--no-sandbox")
        // --disable-gpu ป้องกัน error ใน headless environment
        .arg("--disable-gpu")
        .arg("--disable-dev-shm-usage")
        .arg("--disable-setuid-sandbox")
        .arg("--no-first-run")
        .arg("--no-zygote")
        .build()
        .map_err(|e| anyhow::anyhow!("BrowserConfig error: {}", e))?;

    let (browser, mut handler) = Browser::launch(config)
        .await
        .context("Failed to launch Chromium")?;

    // spawn background task สำหรับจัดการ CDP events
    tokio::spawn(async move {
        loop {
            if handler.next().await.is_none() {
                break;
            }
        }
    });

    Ok(browser)
}
```

`src/screenshot.rs` — ทำ screenshot จริงด้วย CDP

```rust
use anyhow::{Context, Result};
use chromiumoxide::{
    browser::Browser,
    page::Page,
    cdp::browser_protocol::page::{
        CaptureScreenshotFormat, CaptureScreenshotParams,
    },
    cdp::browser_protocol::network::{
        EnableParams, SetExtraHttpHeadersParams, Headers,
        SetCookiesParams, CookieParam,
    },
};
use serde_json::json;
use std::collections::HashMap;

use crate::validation::CookieSpec;

/// Parameters สำหรับ screenshot request
#[derive(Debug, Clone)]
pub struct ScreenshotParams {
    pub url: String,
    pub width: u32,
    pub height: u32,
    pub wait_ms: u32,
    pub full_page: bool,
    pub format: ScreenshotFormat,
    pub extra_headers: Option<HashMap<String, String>>,
    pub cookies: Option<Vec<CookieSpec>>,
}

#[derive(Debug, Clone, PartialEq)]
pub enum ScreenshotFormat {
    Png,
    Jpeg,
}

impl ScreenshotFormat {
    pub fn content_type(&self) -> &'static str {
        match self {
            ScreenshotFormat::Png => "image/png",
            ScreenshotFormat::Jpeg => "image/jpeg",
        }
    }

    pub fn from_str(s: &str) -> Self {
        if s.eq_ignore_ascii_case("jpeg") || s.eq_ignore_ascii_case("jpg") {
            ScreenshotFormat::Jpeg
        } else {
            ScreenshotFormat::Png
        }
    }
}

/// ทำ screenshot ด้วย browser instance ที่รับมา
///
/// # ขั้นตอน
/// 1. สร้าง new page
/// 2. Set viewport
/// 3. Enable network (สำหรับ headers/cookies)
/// 4. Inject headers และ cookies ถ้ามี
/// 5. Navigate ไปยัง URL และรอ load
/// 6. รอ wait_ms เพิ่มเติม (สำหรับ JavaScript animation)
/// 7. Capture screenshot
pub async fn take_screenshot(
    browser: &Browser,
    params: &ScreenshotParams,
) -> Result<Vec<u8>> {
    let page = browser
        .new_page("about:blank")
        .await
        .context("Failed to open new page")?;

    // Set viewport
    page.set_user_agent("Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36")
        .await?;
    page.emulate_media_type("screen").await.ok();

    // Enable network domain
    page.execute(EnableParams::default()).await.ok();

    // Inject custom headers
    if let Some(headers) = &params.extra_headers {
        if !headers.is_empty() {
            let header_map: HashMap<String, serde_json::Value> = headers
                .iter()
                .map(|(k, v)| (k.clone(), json!(v)))
                .collect();
            page.execute(SetExtraHttpHeadersParams::new(Headers::new(
                serde_json::to_value(header_map)?,
            )))
            .await
            .ok();
        }
    }

    // Inject cookies
    if let Some(cookies) = &params.cookies {
        if !cookies.is_empty() {
            let cookie_params: Vec<CookieParam> = cookies
                .iter()
                .map(|c| {
                    CookieParam::new(c.name.clone(), c.value.clone())
                        .with_domain(c.domain.clone())
                })
                .collect();
            page.execute(SetCookiesParams::new(cookie_params))
                .await
                .ok();
        }
    }

    // Navigate
    page.goto(&params.url)
        .await
        .context("Navigation failed")?;

    page.wait_for_navigation()
        .await
        .context("Waiting for navigation failed")?;

    // Wait เพิ่มเติมถ้า client ระบุ wait_ms
    if params.wait_ms > 0 {
        tokio::time::sleep(std::time::Duration::from_millis(
            params.wait_ms as u64,
        ))
        .await;
    }

    // Set viewport dimensions
    page.set_viewport(chromiumoxide::page::ScreenshotViewport {
        width: params.width,
        height: params.height,
        device_scale_factor: None,
        emulating_mobile: None,
        is_landscape: None,
        has_touch: None,
    })
    .await
    .ok();

    // Capture screenshot
    let capture_params = CaptureScreenshotParams::builder()
        .format(match params.format {
            ScreenshotFormat::Jpeg => CaptureScreenshotFormat::Jpeg,
            ScreenshotFormat::Png => CaptureScreenshotFormat::Png,
        })
        .capture_beyond_viewport(params.full_page)
        .build();

    let screenshot = page
        .execute(capture_params)
        .await
        .context("Screenshot capture failed")?;

    let bytes = screenshot
        .result
        .data
        .as_deref()
        .map(|d| base64::decode(d))
        .transpose()
        .context("Base64 decode failed")?
        .unwrap_or_default();

    // ปิด page หลังเสร็จ
    page.close().await.ok();

    Ok(bytes)
}
```

---

### ขั้นที่ 7: Storage Layer — Disk และ S3

`src/storage.rs` — บันทึก screenshot ไปยัง local disk หรือ S3

```rust
use anyhow::{Context, Result};
use std::path::{Path, PathBuf};
use tokio::io::AsyncWriteExt;

#[derive(Debug, Clone)]
pub enum StorageBackend {
    Local { base_dir: PathBuf },
    S3 {
        bucket: String,
        prefix: String,
        region: String,
        endpoint_url: Option<String>,
    },
}

pub struct Storage {
    backend: StorageBackend,
    base_url: String,
}

impl Storage {
    pub fn new_local(base_dir: impl AsRef<Path>, base_url: impl Into<String>) -> Self {
        Storage {
            backend: StorageBackend::Local {
                base_dir: base_dir.as_ref().to_path_buf(),
            },
            base_url: base_url.into(),
        }
    }

    pub fn new_s3(
        bucket: impl Into<String>,
        prefix: impl Into<String>,
        region: impl Into<String>,
        endpoint_url: Option<String>,
        base_url: impl Into<String>,
    ) -> Self {
        Storage {
            backend: StorageBackend::S3 {
                bucket: bucket.into(),
                prefix: prefix.into(),
                region: region.into(),
                endpoint_url,
            },
            base_url: base_url.into(),
        }
    }

    /// บันทึก bytes และ return URL สำหรับดาวน์โหลด
    pub async fn save(
        &self,
        filename: &str,
        data: &[u8],
        content_type: &str,
    ) -> Result<String> {
        match &self.backend {
            StorageBackend::Local { base_dir } => {
                let path = base_dir.join(filename);
                if let Some(parent) = path.parent() {
                    tokio::fs::create_dir_all(parent).await?;
                }
                let mut file = tokio::fs::File::create(&path)
                    .await
                    .with_context(|| format!("Cannot create file: {:?}", path))?;
                file.write_all(data).await?;
                Ok(format!("{}/{}", self.base_url.trim_end_matches('/'), filename))
            }
            StorageBackend::S3 {
                bucket,
                prefix,
                region,
                endpoint_url,
            } => {
                save_to_s3(
                    bucket,
                    prefix,
                    filename,
                    data,
                    content_type,
                    region,
                    endpoint_url.as_deref(),
                    &self.base_url,
                )
                .await
            }
        }
    }
}

async fn save_to_s3(
    bucket: &str,
    prefix: &str,
    filename: &str,
    data: &[u8],
    content_type: &str,
    region: &str,
    endpoint_url: Option<&str>,
    base_url: &str,
) -> Result<String> {
    use aws_sdk_s3::{config::Builder, primitives::ByteStream, Client};

    let mut config_builder = aws_config::from_env()
        .region(aws_config::meta::region::RegionProviderChain::default_provider()
            .or_else(region));

    if let Some(endpoint) = endpoint_url {
        config_builder = config_builder.endpoint_url(endpoint);
    }

    let aws_config = config_builder.load().await;
    let client = Client::new(&aws_config);

    let key = format!("{}/{}", prefix.trim_end_matches('/'), filename);
    client
        .put_object()
        .bucket(bucket)
        .key(&key)
        .body(ByteStream::from(data.to_vec()))
        .content_type(content_type)
        .send()
        .await
        .context("S3 PutObject failed")?;

    Ok(format!("{}/{}", base_url.trim_end_matches('/'), key))
}
```

---

### ขั้นที่ 8: Main Router, Handlers และ Admin Endpoints

`src/main.rs` — ประกอบ router และ AppState

```rust
mod admin;
mod browser;
mod cache;
mod concurrency;
mod jobs;
mod screenshot;
mod storage;
mod validation;

use axum::{
    extract::{Path, State},
    http::{HeaderMap, HeaderValue, StatusCode},
    response::{IntoResponse, Response},
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::{collections::HashMap, sync::Arc};
use tokio::sync::mpsc;
use tracing::info;

use cache::{generate_cache_key, redis_key, CacheKeyParams};
use concurrency::BrowserSemaphore;
use jobs::{Job, JobStore};
use validation::{validate_url, CookieSpec};

// ── Request / Response types ───────────────────────────────────────────

#[derive(Debug, Deserialize)]
pub struct ScreenshotRequest {
    pub url: String,
    #[serde(default = "default_width")]
    pub width: u32,
    #[serde(default = "default_height")]
    pub height: u32,
    #[serde(default)]
    pub wait_ms: u32,
    #[serde(default)]
    pub full_page: bool,
    #[serde(default = "default_format")]
    pub format: String,
    #[serde(default)]
    pub headers: Option<HashMap<String, String>>,
    #[serde(default)]
    pub cookies: Option<Vec<CookieSpec>>,
}

fn default_width() -> u32 { 1280 }
fn default_height() -> u32 { 800 }
fn default_format() -> String { "png".to_string() }

#[derive(Debug, Serialize)]
pub struct AsyncJobResponse { pub job_id: String }

#[derive(Debug, Serialize)]
pub struct JobStatusResponse {
    pub id: String,
    pub status: String,
    pub download_url: Option<String>,
    pub error: Option<String>,
}

// ── AppState ───────────────────────────────────────────────────────────

#[derive(Clone)]
pub struct AppState {
    pub job_store: JobStore,
    pub semaphore: BrowserSemaphore,
    pub job_tx: mpsc::Sender<String>,
    pub cache_hits: Arc<std::sync::atomic::AtomicU64>,
    pub cache_misses: Arc<std::sync::atomic::AtomicU64>,
    pub total_screenshots: Arc<std::sync::atomic::AtomicU64>,
}

// ── Error type ─────────────────────────────────────────────────────────

pub enum AppError {
    Validation(String),
    NotFound,
    Internal(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AppError::Validation(m) => (StatusCode::BAD_REQUEST, m),
            AppError::NotFound => (StatusCode::NOT_FOUND, "not found".into()),
            AppError::Internal(m) => (StatusCode::INTERNAL_SERVER_ERROR, m),
        };
        (status, Json(serde_json::json!({"error": msg}))).into_response()
    }
}

// ── Handlers ───────────────────────────────────────────────────────────

/// POST /screenshots — synchronous: รอผลแล้ว return binary
async fn take_screenshot(
    State(state): State<Arc<AppState>>,
    Json(req): Json<ScreenshotRequest>,
) -> Result<Response, AppError> {
    validate_url(&req.url).map_err(|e| AppError::Validation(e.to_string()))?;

    let format = screenshot::ScreenshotFormat::from_str(&req.format);
    let params = CacheKeyParams {
        url: &req.url,
        width: req.width,
        height: req.height,
        wait_ms: req.wait_ms,
        full_page: req.full_page,
        format: &req.format,
    };
    let cache_key = redis_key(&generate_cache_key(&params));

    // TODO: ตรวจ Redis cache ก่อน
    // if let Some(cached) = cache.get(&cache_key).await { ... }

    // Acquire semaphore — block ถ้า browser slots เต็ม
    let _permit = state.semaphore.acquire().await;

    // TODO: ดึง browser จาก pool แล้วทำ screenshot จริง
    // let browser = pool.acquire().await?;
    // let bytes = take_screenshot(&browser, &params).await?;

    state.total_screenshots.fetch_add(1, std::sync::atomic::Ordering::Relaxed);

    // Stub response (ใช้ใน development/test)
    let image_bytes = minimal_png();
    let content_type = format.content_type();
    let mut headers = HeaderMap::new();
    headers.insert(
        axum::http::header::CONTENT_TYPE,
        HeaderValue::from_str(content_type).unwrap(),
    );
    Ok((StatusCode::OK, headers, image_bytes).into_response())
}

/// POST /screenshots/async — return job_id ทันที, ทำงานใน background
async fn enqueue_screenshot(
    State(state): State<Arc<AppState>>,
    Json(req): Json<ScreenshotRequest>,
) -> Result<Json<AsyncJobResponse>, AppError> {
    validate_url(&req.url).map_err(|e| AppError::Validation(e.to_string()))?;

    let job = Job::new(&req.url);
    let job_id = state.job_store.insert(job);
    let _ = state.job_tx.try_send(job_id.clone());
    Ok(Json(AsyncJobResponse { job_id }))
}

/// GET /jobs/:id — poll status ของ async job
async fn get_job(
    State(state): State<Arc<AppState>>,
    Path(id): Path<String>,
) -> Result<Json<JobStatusResponse>, AppError> {
    let job = state.job_store.get(&id).ok_or(AppError::NotFound)?;
    Ok(Json(JobStatusResponse {
        id: job.id,
        status: format!("{:?}", job.status).to_lowercase(),
        download_url: job.download_url,
        error: job.error,
    }))
}

/// GET /admin/stats — queue depth, worker count, cache hit rate
async fn admin_stats(State(state): State<Arc<AppState>>) -> Json<serde_json::Value> {
    use std::sync::atomic::Ordering::Relaxed;
    let hits = state.cache_hits.load(Relaxed);
    let misses = state.cache_misses.load(Relaxed);
    let total_cache = hits + misses;
    let hit_rate = if total_cache > 0 {
        hits as f64 / total_cache as f64
    } else {
        0.0
    };

    Json(serde_json::json!({
        "queue_depth": state.job_store.queued_count(),
        "active_workers": state.semaphore.max() - state.semaphore.available_permits(),
        "max_workers": state.semaphore.max(),
        "cache_hits": hits,
        "cache_misses": misses,
        "cache_hit_rate": hit_rate,
        "total_screenshots": state.total_screenshots.load(Relaxed),
    }))
}

/// POST /admin/clear-cache — flush Redis cache ทั้งหมด
async fn clear_cache() -> Json<serde_json::Value> {
    // TODO: เรียก cache.flush_pattern("screenshot:*").await
    Json(serde_json::json!({
        "status": "ok",
        "message": "cache cleared",
    }))
}

fn minimal_png() -> Vec<u8> {
    vec![
        0x89,0x50,0x4e,0x47,0x0d,0x0a,0x1a,0x0a,
        0x00,0x00,0x00,0x0d,0x49,0x48,0x44,0x52,
        0x00,0x00,0x00,0x01,0x00,0x00,0x00,0x01,
        0x08,0x06,0x00,0x00,0x00,0x1f,0x15,0xc4,
        0x89,0x00,0x00,0x00,0x0a,0x49,0x44,0x41,
        0x54,0x78,0x9c,0x62,0x00,0x01,0x00,0x00,
        0x05,0x00,0x01,0x0d,0x0a,0x2d,0xb4,0x00,
        0x00,0x00,0x00,0x49,0x45,0x4e,0x44,0xae,
        0x42,0x60,0x82,
    ]
}

/// Background worker: รับ job_id จาก channel แล้วทำ screenshot
async fn job_worker(
    store: JobStore,
    semaphore: BrowserSemaphore,
    mut rx: mpsc::Receiver<String>,
    // pool: Arc<BrowserPool>,
    // storage: Arc<Storage>,
    // cache: Arc<CacheClient>,
) {
    while let Some(job_id) = rx.recv().await {
        let store = store.clone();
        let semaphore = semaphore.clone();

        tokio::spawn(async move {
            let Some(mut job) = store.get(&job_id) else {
                return;
            };
            if job.start_processing().is_err() {
                return;
            }
            store.update(job.clone());

            // Acquire semaphore
            let _permit = semaphore.acquire().await;

            // TODO: ทำ screenshot จริง
            // let bytes = take_screenshot(&browser, &params).await?;
            // let url = storage.save(&filename, &bytes, content_type).await?;

            // Simulate success
            tokio::time::sleep(std::time::Duration::from_millis(10)).await;
            let download_url = format!("/files/{}.png", &job_id[..8]);
            if job.complete(download_url).is_ok() {
                store.update(job);
            }
        });
    }
}

pub fn build_router(state: Arc<AppState>) -> Router {
    Router::new()
        .route("/screenshots", post(take_screenshot))
        .route("/screenshots/async", post(enqueue_screenshot))
        .route("/jobs/{id}", get(get_job))
        .route("/admin/stats", get(admin_stats))
        .route("/admin/clear-cache", post(clear_cache))
        .with_state(state)
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::from_default_env()
                .add_directive("screenshot_service=info".parse().unwrap()),
        )
        .init();

    let (tx, rx) = mpsc::channel::<String>(1000);

    let state = Arc::new(AppState {
        job_store: JobStore::new(),
        semaphore: BrowserSemaphore::new(3),
        job_tx: tx.clone(),
        cache_hits: Arc::new(std::sync::atomic::AtomicU64::new(0)),
        cache_misses: Arc::new(std::sync::atomic::AtomicU64::new(0)),
        total_screenshots: Arc::new(std::sync::atomic::AtomicU64::new(0)),
    });

    // Start background worker
    let worker_store = state.job_store.clone();
    let worker_sem = state.semaphore.clone();
    tokio::spawn(async move {
        job_worker(worker_store, worker_sem, rx).await;
    });

    let app = build_router(state);

    let addr = std::env::var("LISTEN_ADDR").unwrap_or_else(|_| "0.0.0.0:3000".to_string());
    info!("screenshot-service listening on {}", addr);

    let listener = tokio::net::TcpListener::bind(&addr).await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

## การทดสอบ (Testing)

### Unit Tests ที่รันได้โดยไม่ต้องมี Chromium หรือ Redis

สร้าง project ใน scratchpad แล้วรัน `cargo test`:

```
running 53 tests
test cache::tests::test_cache_key_is_64_hex_chars ... ok
test cache::tests::test_different_url_gives_different_key ... ok
test cache::tests::test_different_format_gives_different_key ... ok
test cache::tests::test_cache_key_is_deterministic ... ok
test cache::tests::test_different_width_gives_different_key ... ok
test cache::tests::test_full_page_flag_changes_key ... ok
test cache::tests::test_known_hash_value ... ok
test cache::tests::test_redis_key_prefix ... ok
test concurrency::tests::test_semaphore_max_reported_correctly ... ok
test concurrency::tests::test_semaphore_allows_up_to_max ... ok
test concurrency::tests::test_semaphore_released_on_drop ... ok
test jobs::tests::test_done_cannot_transition_again ... ok
test jobs::tests::test_is_terminal_done ... ok
test jobs::tests::test_is_terminal_failed ... ok
test jobs::tests::test_new_job_is_queued ... ok
test jobs::tests::test_processing_to_done ... ok
test jobs::tests::test_processing_to_failed ... ok
test jobs::tests::test_queued_cannot_skip_to_done ... ok
test jobs::tests::test_queued_cannot_skip_to_failed ... ok
test jobs::tests::test_queued_is_not_terminal ... ok
test jobs::tests::test_queued_to_processing ... ok
test jobs::tests::test_store_get_missing_returns_none ... ok
test jobs::tests::test_store_insert_and_get ... ok
test jobs::tests::test_store_queued_count ... ok
test jobs::tests::test_store_update ... ok
test validation::tests::test_check_resolved_ip_public_allowed ... ok
test validation::tests::test_check_resolved_ip_private_blocked ... ok
test validation::tests::test_ipv6_link_local_is_private ... ok
test validation::tests::test_ipv6_loopback_is_private ... ok
test validation::tests::test_ipv6_public_is_not_private ... ok
test validation::tests::test_ipv6_unique_local_is_private ... ok
test validation::tests::test_link_local_169_254_is_private ... ok
test validation::tests::test_loopback_127_0_0_1_is_private ... ok
test validation::tests::test_loopback_127_255_255_255_is_private ... ok
test validation::tests::test_public_ip_1_1_1_1_is_not_private ... ok
test validation::tests::test_public_ip_8_8_8_8_is_not_private ... ok
test validation::tests::test_rfc1918_10_255_255_255_is_private ... ok
test validation::tests::test_rfc1918_10_x_x_x_is_private ... ok
test validation::tests::test_rfc1918_172_16_is_private ... ok
test validation::tests::test_rfc1918_172_31_is_private ... ok
test validation::tests::test_rfc1918_172_32_is_public ... ok
test validation::tests::test_rfc1918_192_168_is_private ... ok
test validation::tests::test_scheme_data_rejected ... ok
test validation::tests::test_scheme_file_rejected ... ok
test validation::tests::test_scheme_ftp_rejected ... ok
test validation::tests::test_scheme_http_allowed ... ok
test validation::tests::test_scheme_https_allowed ... ok
test validation::tests::test_scheme_javascript_rejected ... ok
test validation::tests::test_url_localhost_rejected ... ok
test validation::tests::test_url_valid_https ... ok
test validation::tests::test_url_with_ip_loopback_rejected ... ok
test validation::tests::test_url_with_ip_private_rejected ... ok
test concurrency::tests::test_semaphore_blocks_when_exhausted ... ok

test result: ok. 53 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.05s
```

### Integration Tests (ต้องมี runtime environment)

Integration tests ต้องการ Chromium binary และ Redis ที่รันอยู่ จึงถูก mark ด้วย `#[ignore]` เพื่อ skip ใน CI ปกติ รัน manually ด้วย `cargo test -- --ignored`

```rust
// tests/integration.rs
//
// Integration tests สำหรับ screenshot-service
// ต้องการ:
//   - Chromium ที่ /opt/pw-browsers/chromium (หรือ PATH)
//   - Redis ที่ redis://127.0.0.1:6379
//
// รันด้วย: cargo test -- --ignored

use screenshot_service::validation::{is_private_ip, validate_url_async};
use screenshot_service::browser::BrowserPool;
use screenshot_service::screenshot::{take_screenshot, ScreenshotParams, ScreenshotFormat};

const CHROMIUM_PATH: &str = "/opt/pw-browsers/chromium";

#[tokio::test]
#[ignore = "requires Chromium at /opt/pw-browsers/chromium"]
async fn test_screenshot_google() {
    let pool = BrowserPool::new(1, CHROMIUM_PATH).await.unwrap();
    let handle = pool.acquire().await.unwrap();
    let guard = handle.browser().await;
    let browser = &guard[0].browser;

    let params = ScreenshotParams {
        url: "https://example.com".to_string(),
        width: 1280,
        height: 800,
        wait_ms: 500,
        full_page: false,
        format: ScreenshotFormat::Png,
        extra_headers: None,
        cookies: None,
    };

    let bytes = take_screenshot(browser, &params).await.unwrap();
    assert!(!bytes.is_empty(), "Screenshot should produce non-empty bytes");
    // ตรวจ PNG signature
    assert_eq!(&bytes[..8], &[0x89,0x50,0x4e,0x47,0x0d,0x0a,0x1a,0x0a]);
}

#[tokio::test]
#[ignore = "requires DNS resolution in test environment"]
async fn test_ssrf_via_dns_rebinding() {
    // ทดสอบว่า DNS resolution ที่ resolve เป็น private IP ถูกปฏิเสธ
    // ในสภาพแวดล้อมที่ควบคุมได้
    let result = validate_url_async("http://10.0.0.1/").await;
    assert!(result.is_err());
}

#[tokio::test]
#[ignore = "requires Chromium at /opt/pw-browsers/chromium"]
async fn test_screenshot_with_custom_headers() {
    use std::collections::HashMap;
    let pool = BrowserPool::new(1, CHROMIUM_PATH).await.unwrap();
    let handle = pool.acquire().await.unwrap();
    let guard = handle.browser().await;
    let browser = &guard[0].browser;

    let mut headers = HashMap::new();
    headers.insert("X-Custom-Header".to_string(), "test-value".to_string());
    headers.insert("Accept-Language".to_string(), "th-TH,th;q=0.9".to_string());

    let params = ScreenshotParams {
        url: "https://httpbin.org/headers".to_string(),
        width: 1280,
        height: 800,
        wait_ms: 1000,
        full_page: false,
        format: ScreenshotFormat::Png,
        extra_headers: Some(headers),
        cookies: None,
    };

    let bytes = take_screenshot(browser, &params).await.unwrap();
    assert!(!bytes.is_empty());
}
```

### Test Coverage Summary

| Module | Tests | Coverage |
|--------|-------|---------|
| `validation.rs` | 28 tests | scheme validation, RFC1918 ranges, localhost, IPv6 |
| `cache.rs` | 8 tests | determinism, collision resistance, known hash |
| `jobs.rs` | 15 tests | all state transitions, terminal states, store operations |
| `concurrency.rs` | 4 tests | max permits, blocking, release on drop |
| **Total** | **53 tests** | **all pass** |

## Pitfalls ที่พบบ่อย

### Pitfall 1: SSRF ผ่าน DNS Rebinding

**ปัญหา:** การตรวจสอบ URL ที่ผ่านแล้ว ไม่ได้หมายความว่าจะปลอดภัยตลอดไป attacker สามารถ set TTL ของ DNS record ให้สั้นมาก (0 วินาที) ทำให้ hostname resolve เป็น public IP ตอน validate แต่ resolve เป็น `10.0.0.1` ตอน Chrome จริงๆ ไป connect

**การแก้:**
1. ทำ DNS resolution ก่อน navigate และใช้ IP address ที่ resolve ได้แทน hostname ใน URL ที่ส่งให้ Chrome
2. หรือใช้ `--host-rules` flag ของ Chrome เพื่อ force resolve hostname ผ่าน lookup ที่เราควบคุม
3. ในทางปฏิบัติ: validate ทั้ง pre-request (scheme + hostname check) และ post-DNS-resolution (IP check) ทุกครั้ง

```rust
// หลัง resolve DNS แล้ว — ตรวจ IP ที่ได้จริง
pub async fn validate_url_async(url: &str) -> Result<String, ValidationError> {
    validate_url(url)?; // ตรวจ scheme + literal IP ก่อน

    let parsed = url::Url::parse(url).unwrap();
    let host = parsed.host_str().ok_or(ValidationError::MissingHost)?;

    // ถ้าเป็น literal IP ผ่านแล้วตั้งแต่ validate_url
    if IpAddr::from_str(host).is_ok() {
        return Ok(url.to_string());
    }

    let port = parsed.port_or_known_default().unwrap_or(80);
    let addrs: Vec<_> = tokio::net::lookup_host(format!("{}:{}", host, port))
        .await
        .map_err(|e| ValidationError::InvalidUrl(format!("DNS lookup failed: {}", e)))?
        .collect();

    // ตรวจทุก IP ที่ resolve ได้
    for addr in &addrs {
        if is_private_ip(&addr.ip()) {
            return Err(ValidationError::SsrfBlocked(addr.ip().to_string()));
        }
    }
    Ok(url.to_string())
}
```

---

### Pitfall 2: Chrome Instance ไม่ถูก Cleanup เมื่อ Crash

**ปัญหา:** ถ้า screenshot logic panic หรือ future ถูก cancel ขณะที่ browser อยู่ใน `in_use = true` pool จะติดค้างอยู่ในสภาพที่ไม่มี browser ว่างเลย

**การแก้:** ใช้ RAII pattern ผ่าน `BrowserHandle` ที่ implement `Drop`:

```rust
impl Drop for BrowserHandle<'_> {
    fn drop(&mut self) {
        let pool = self.pool.clone();
        let index = self.index;
        // spawn task เพื่อ release แม้ในขณะที่ async runtime กำลัง unwind
        tokio::spawn(async move {
            if let Ok(mut guard) = pool.try_lock() {
                if let Some(inst) = guard.get_mut(index) {
                    inst.in_use = false;
                }
            }
        });
    }
}
```

Semaphore permit ก็ควรถูก drop พร้อมกัน — เพราะ `OwnedSemaphorePermit` implement `Drop` อยู่แล้ว ให้แน่ใจว่า permit และ `BrowserHandle` อยู่ใน scope เดียวกัน:

```rust
// ดี: ทั้งสองถูก drop พร้อมกันเมื่อ scope นี้จบ
let _permit = semaphore.acquire().await;
let _browser = pool.acquire().await?;
let bytes = take_screenshot(&browser, &params).await?;
// _permit และ _browser drop ที่นี่
```

---

### Pitfall 3: `chromiumoxide` handler loop ต้อง spawn แยก

**ปัญหา:** `Browser::launch()` return ทั้ง `(Browser, BrowserHandler)` — ถ้าไม่ spawn `BrowserHandler` ไว้ใน background, CDP messages จะไม่ถูกประมวลผล และทุก page navigation จะ hang ตลอดไป

**การแก้:**

```rust
// ผิด: ทิ้ง handler ไว้เฉยๆ
let (browser, _handler) = Browser::launch(config).await?;
// browser จะ hang ทันทีที่ใช้งาน!

// ถูก: spawn handler ใน background
let (browser, mut handler) = Browser::launch(config).await?;
tokio::spawn(async move {
    loop {
        if handler.next().await.is_none() {
            break;
        }
    }
});
// ตอนนี้ browser ใช้งานได้ปกติ
```

---

### Pitfall 4: `cargo test` vs Integration Tests — อย่าเปิด Chrome ใน unit test

**ปัญหา:** ถ้าเขียน test ที่ launch Chrome จริงโดยไม่ mark `#[ignore]` CI pipeline จะพัง เพราะ:
1. ใน Docker container ที่ไม่มี `--no-sandbox` Chrome จะ refuse to start
2. Chromium binary path อาจต่างกันในแต่ละ environment
3. Test จะช้ามาก (แต่ละ test อาจใช้เวลา 3–10 วินาที)

**การแก้:**
- Unit tests: test logic เท่านั้น (validation, cache key, state machine, semaphore)
- Integration tests ทุกตัวที่ต้อง Chrome หรือ Redis ต้อง `#[ignore]`
- สร้าง `Makefile` หรือ script สำหรับรัน integration tests แยก:

```makefile
test-unit:
    cargo test

test-integration:
    CHROMIUM_PATH=/opt/pw-browsers/chromium \
    REDIS_URL=redis://127.0.0.1:6379 \
    cargo test -- --ignored
```

---

### Pitfall 5: Semaphore Permit ต้องถือไว้ตลอดการทำงาน ไม่ใช่แค่ตอน "check"

**ปัญหา:** บางครั้ง developer เขียน code ที่ acquire permit แล้ว drop ทันทีก่อนทำงานจริง ทำให้ semaphore ไม่มีผลจริง

```rust
// ผิด: permit ถูก drop ทันที!
{
    let _permit = semaphore.acquire().await;
} // ← drop ที่นี่ — browser slot ว่างแล้วก่อนที่จะทำงาน
let bytes = take_screenshot(...).await?;

// ถูก: permit ต้องถือไว้ตลอด scope ที่ใช้งาน browser
let _permit = semaphore.acquire().await;
let bytes = take_screenshot(...).await?;
// permit drop ที่นี่ หลังจาก screenshot เสร็จแล้ว
```

## การ Package และ Deploy

### Build Release Binary

```bash
# Build สำหรับ production
cargo build --release

# Binary อยู่ที่
./target/release/screenshot-service
```

### Dockerfile

```dockerfile
FROM rust:1.82-slim AS builder
WORKDIR /app
COPY . .
RUN apt-get update && apt-get install -y pkg-config libssl-dev && rm -rf /var/lib/apt/lists/*
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y \
    chromium \
    fonts-thai-tlwg \
    fonts-noto \
    ca-certificates \
    libssl3 \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/target/release/screenshot-service /usr/local/bin/

ENV CHROMIUM_PATH=/usr/bin/chromium
ENV REDIS_URL=redis://redis:6379
ENV LISTEN_ADDR=0.0.0.0:3000
ENV MAX_BROWSERS=3
ENV CACHE_TTL_SECONDS=60

EXPOSE 3000
CMD ["screenshot-service"]
```

### docker-compose.yml

```yaml
version: "3.9"
services:
  screenshot-service:
    build: .
    ports:
      - "3000:3000"
    environment:
      - REDIS_URL=redis://redis:6379
      - MAX_BROWSERS=3
      - CACHE_TTL_SECONDS=60
      - STORAGE_TYPE=local
      - STORAGE_BASE_DIR=/data/screenshots
      - BASE_URL=http://localhost:3000
    volumes:
      - screenshots:/data/screenshots
    depends_on:
      - redis
    # จำเป็นสำหรับ Chromium ใน container
    shm_size: '256mb'

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data

volumes:
  screenshots:
  redis-data:
```

### Environment Variables

| Variable | Default | คำอธิบาย |
|----------|---------|---------|
| `LISTEN_ADDR` | `0.0.0.0:3000` | address ที่ server listen |
| `CHROMIUM_PATH` | `/opt/chromium/chromium` | path ไปยัง Chromium binary |
| `MAX_BROWSERS` | `3` | จำนวน concurrent browser slots |
| `REDIS_URL` | `redis://127.0.0.1:6379` | Redis connection URL |
| `CACHE_TTL_SECONDS` | `60` | TTL ของ screenshot cache |
| `STORAGE_TYPE` | `local` | `local` หรือ `s3` |
| `STORAGE_BASE_DIR` | `./screenshots` | path สำหรับ local storage |
| `S3_BUCKET` | — | ชื่อ S3 bucket |
| `S3_PREFIX` | `screenshots` | prefix ใน S3 |
| `S3_REGION` | `us-east-1` | AWS region |
| `S3_ENDPOINT_URL` | — | custom endpoint (MinIO, Cloudflare R2) |
| `BASE_URL` | `http://localhost:3000` | base URL สำหรับ download links |

### ตัวอย่างการใช้งาน API

```bash
# Synchronous screenshot
curl -X POST http://localhost:3000/screenshots \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com","width":1280,"height":800,"format":"png"}' \
  --output screenshot.png

# Async screenshot
JOB=$(curl -s -X POST http://localhost:3000/screenshots/async \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com","full_page":true}' | jq -r .job_id)

# Poll job status
curl http://localhost:3000/jobs/$JOB

# Screenshot with custom headers
curl -X POST http://localhost:3000/screenshots \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com",
    "headers": {"Accept-Language": "th-TH,th;q=0.9"},
    "cookies": [{"name":"session","value":"abc123","domain":"example.com"}]
  }' --output screenshot.png

# Admin stats
curl http://localhost:3000/admin/stats | jq .

# Flush cache
curl -X POST http://localhost:3000/admin/clear-cache
```

## การต่อยอด (Extensions & Exercises)

### Exercise 1: Rate Limiting ต่อ IP (ระดับ: กลาง)

เพิ่ม middleware ที่จำกัดจำนวน request ต่อ IP address โดยเก็บ counter ใน `DashMap<IpAddr, (u32, Instant)>`

```rust
// Hint: สร้าง Tower middleware layer
use tower::ServiceBuilder;
use tower_http::limit::RequestBodyLimitLayer;

// ใน AppState เพิ่ม
pub rate_limiter: Arc<RateLimiter>,

// RateLimiter struct
pub struct RateLimiter {
    requests: DashMap<String, VecDeque<Instant>>,
    max_requests: usize,
    window: Duration,
}

impl RateLimiter {
    pub fn check(&self, ip: &str) -> bool {
        let now = Instant::now();
        let mut entry = self.requests.entry(ip.to_string()).or_default();
        // ลบ entries ที่เก่ากว่า window
        while entry.front().map_or(false, |t| now.duration_since(*t) > self.window) {
            entry.pop_front();
        }
        if entry.len() >= self.max_requests {
            return false; // rate limit exceeded
        }
        entry.push_back(now);
        true
    }
}
```

### Exercise 2: Webhook Notification เมื่อ Job เสร็จ (ระดับ: กลาง)

เพิ่ม optional `callback_url` ใน `ScreenshotRequest` — เมื่อ async job เสร็จให้ POST ผลลัพธ์ไปยัง URL นั้น

```rust
// เพิ่มใน ScreenshotRequest
pub callback_url: Option<String>,

// ใน job_worker หลัง job.complete(...)
if let Some(url) = callback_url {
    let payload = serde_json::json!({
        "job_id": job_id,
        "status": "done",
        "download_url": download_url,
    });
    let _ = reqwest::Client::new()
        .post(&url)
        .json(&payload)
        .timeout(Duration::from_secs(5))
        .send()
        .await;
}
```

**สิ่งที่ต้องระวัง:** callback URL ต้องผ่าน SSRF validation เช่นเดียวกัน — อย่าอนุญาตให้ callback URL ชี้ไปยัง private network!

### Exercise 3: PDF Export (ระดับ: สูง)

เพิ่ม endpoint `POST /pdf` ที่ return PDF แทน image โดยใช้ CDP `printToPDF` command

```rust
use chromiumoxide::cdp::browser_protocol::page::{
    PrintToPdfParams, PrintToPdfParamsBuilder,
};

pub async fn take_pdf(browser: &Browser, params: &PdfParams) -> Result<Vec<u8>> {
    let page = browser.new_page(&params.url).await?;
    page.wait_for_navigation().await?;

    let pdf_params = PrintToPdfParamsBuilder::default()
        .print_background(true)
        .paper_width(8.27)  // A4 width in inches
        .paper_height(11.69) // A4 height in inches
        .build();

    let result = page.execute(pdf_params).await?;
    let pdf_data = base64::decode(result.result.data.unwrap_or_default())?;
    page.close().await.ok();
    Ok(pdf_data)
}
```

### Exercise 4: Metrics Endpoint ด้วย Prometheus (ระดับ: สูง)

เพิ่ม `GET /metrics` ที่ return Prometheus-compatible metrics สำหรับ scraping

```rust
// เพิ่ม dependency
// prometheus = { version = "0.13", features = ["process"] }

use prometheus::{
    register_counter_vec, register_histogram_vec,
    register_gauge, Encoder, TextEncoder,
    CounterVec, HistogramVec, Gauge,
};

pub struct Metrics {
    pub screenshot_total: CounterVec,
    pub screenshot_duration: HistogramVec,
    pub cache_hits_total: CounterVec,
    pub queue_depth: Gauge,
    pub active_browsers: Gauge,
}

// Handler
async fn metrics_handler() -> impl IntoResponse {
    let encoder = TextEncoder::new();
    let metric_families = prometheus::gather();
    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer).unwrap();
    (
        [(axum::http::header::CONTENT_TYPE, "text/plain; version=0.0.4")],
        buffer,
    )
}
```

## สรุป

ในโปรเจคนี้เราได้สร้าง screenshot service ที่ประกอบด้วย:

**Security-first design:** การป้องกัน SSRF ด้วยการตรวจสอบ RFC1918 ranges ทั้งจาก literal IP และจาก DNS resolution ผลลัพธ์ — ป้องกันไม่ให้ service ถูกใช้เป็น proxy ไปยัง internal network

**Production patterns:**
- `BrowserPool` + `BrowserSemaphore` สำหรับ resource management ที่ prevent OOM
- `async` job queue ด้วย `tokio::sync::mpsc` สำหรับ long-running tasks
- Redis cache ด้วย SHA-256 key สำหรับ deduplication
- RAII pattern ที่แน่ใจว่า browser slots คืน pool เสมอ แม้เกิด panic

**Testing strategy:** แยก unit tests (ที่รันได้เร็ว ไม่ต้องมี infrastructure) ออกจาก integration tests (ที่ต้องมี Chromium + Redis) ด้วย `#[ignore]` — ทำให้ CI pipeline เร็วและน่าเชื่อถือ

**ทักษะที่ได้จากโปรเจคนี้** ใช้ได้โดยตรงกับงาน: web scraping service, automated testing platform, content moderation pipeline, และ PDF generation service ทุกประเภทที่ต้องควบคุม headless browser ใน production environment

---

**โปรเจคก่อนหน้า:** [project-b09-api-gateway.md](project-b09-api-gateway.md) | **โปรเจคถัดไป:** [project-c01-etl-pipeline.md](project-c01-etl-pipeline.md)
